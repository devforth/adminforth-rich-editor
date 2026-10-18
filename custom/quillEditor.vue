<template>
  <div
    v-if="wasQuillInitializedSuccessfully" 
    class="af-quill-editor block w-full whitespace-normal rounded-default border shadow-customLight transition-colors
      bg-lightInputBackground text-lightInputText border-lightInputBorder hover:border-lightInputBorderHover
      focus-within:border-lightInputFocusBorder focus-within:ring-[3px] focus-within:ring-lightInputFocusRing
      dark:bg-darkInputBackground dark:text-darkInputText dark:border-darkInputBorder dark:hover:border-darkInputBorderHover
      dark:focus-within:border-darkInputFocusBorder dark:focus-within:ring-darkInputFocusRing"
  >
    <div  
      ref="editor"
      @keydown.tab.prevent.stop="approveCompletion('all')"
      @keydown.ctrl.right.prevent.stop="approveCompletion('word')"
      @keydown.ctrl.down.prevent.stop="startCompletion()"
      @mousedown="textAreaClickHandler"
    >
      
    </div>
  </div>
  <Textarea
      v-else
      :modelValue="currentValue"
      @update:modelValue="emit('update:value', $event);"
      placeholder="Enter some text..."
      class="w-64"
   />


</template>

<script setup lang="ts">
import { onMounted, ref, onUnmounted, watch, h, render, type Ref, type FunctionalComponent } from "vue";
import { callAdminForthApi } from '@/utils';
import { AdminForthColumnCommon } from '@/types/Common';
import adminforth from '@/adminforth';

import AsyncQueue from './async-queue';
import Quill from "quill";
import "quill/dist/quill.snow.css";
import QuillTableBetter from 'quill-table-better-yaroslav8765';
import 'quill-table-better-yaroslav8765/dist/quill-table-better.css';
import { Textarea } from '@/afcl'
import {
  IconAlignCenter, IconAlignJustify, IconAlignLeft, IconAlignRight, IconBold, IconCode, IconHeading2, IconHeading3,
  IconImage, IconItalic, IconLink, IconList, IconListChecks, IconListOrdered, IconRemoveFormatting, IconStrikethrough,
  IconTable, IconTextQuote, IconUnderline,
} from '@iconify-prerendered/vue-lucide';

import { useI18n } from 'vue-i18n';
import { ok } from "assert";

const { t } = useI18n();
function dbg(title: string,...args: any[]) {
  // return; // comment for debug
  // console.log(title, ...args.map(a =>JSON.stringify(a, null, 1))); 
}

// blots/embed: Represents inline embed elements, like images or videos that can be inserted into the text flow.
const Embed = Quill.import('blots/embed');
const BlockEmbed = Quill.import('blots/block/embed');

// @ts-ignore
class CompleteBlot extends Embed {
  static blotName = 'complete';
  static tagName = 'span';
  // https://stackoverflow.com/a/78434756/27379293
  static className = "complete-blot";

  static create(value: { text: string }) {
    let node = super.create();
    // we should keep contenteditable=true for case when user clicks on empty area
    // node.setAttribute('contenteditable', 'false');
    node.setAttribute('completer', '');
    node.innerText = value.text;
    return node;
  }
  
  static value(node: HTMLElement) {
    return {
      text: node.innerText,
    };
  }
}

// @ts-ignore
class ImageBlot extends BlockEmbed {
  static blotName = 'image';
  static tagName = 'img';

  static create(value) {
    let node = super.create();
    node.setAttribute('alt', value.alt);
    node.setAttribute('src', value.url);
    node.setAttribute('data-s3path', value['s3Path']);
    return node;
  }
  
  static value(node) {
    return {
      alt: node.getAttribute('alt'),
      url: node.getAttribute('src'),
      s3Path: node.getAttribute('data-s3path'),
    };
  }
}

// @ts-ignore
class ImageBlotEmbed extends Embed {
  static blotName = 'image_embed';
  static tagName = 'img';

  static create(value) {
    let node = super.create();
    node.setAttribute('alt', value.alt);
    node.setAttribute('src', value.url);
    node.setAttribute('data-s3path', value['s3Path']);
    return node;
  }
  
  static value(node) {
    return {
      alt: node.getAttribute('alt'),
      url: node.getAttribute('src'),
      s3Path: node.getAttribute('data-s3path'),
    };
  }
}

// @ts-ignore
Quill.register(CompleteBlot);
// @ts-ignore
Quill.register(ImageBlot);
// @ts-ignore
Quill.register(ImageBlotEmbed);

Quill.register({
  'modules/table-better': QuillTableBetter
}, true);

// the toolbar draws Lucide icons like the rest of the UI; Quill takes icons as SVG strings,
// so they are set after the table module registered its own one
function iconSvg(icon: FunctionalComponent) {
  const holder = document.createElement('div');
  render(h(icon), holder);
  return holder.innerHTML;
}

const quillIcons = Quill.import('ui/icons') as Record<string, any>;
Object.assign(quillIcons, {
  bold: iconSvg(IconBold),
  italic: iconSvg(IconItalic),
  underline: iconSvg(IconUnderline),
  strike: iconSvg(IconStrikethrough),
  'table-better': iconSvg(IconTable),
  blockquote: iconSvg(IconTextQuote),
  'code-block': iconSvg(IconCode),
  link: iconSvg(IconLink),
  image: iconSvg(IconImage),
  clean: iconSvg(IconRemoveFormatting),
  header: { ...quillIcons.header, 2: iconSvg(IconHeading2), 3: iconSvg(IconHeading3) },
  list: { ...quillIcons.list, ordered: iconSvg(IconListOrdered), bullet: iconSvg(IconList), check: iconSvg(IconListChecks) },
  align: { ...quillIcons.align, '': iconSvg(IconAlignLeft), center: iconSvg(IconAlignCenter), right: iconSvg(IconAlignRight), justify: iconSvg(IconAlignJustify) },
});


const updaterQueue = new AsyncQueue();

const props = defineProps<{
  column: AdminForthColumn,
  record: any,
  meta: any,
}>();

const emit = defineEmits([
  'update:value',
]);

const currentValue: Ref<string> = ref('');

const editor = ref<HTMLElement>();
const completion = ref<string[] | null>(null);
let quill: any = null;
const editorFocused = ref(false);

let lastText: string | null = null;

const imageProgress = ref(0);
const wasQuillInitializedSuccessfully = ref(true);

const selectedSymbolsCount = ref(0);

async function saveToServer(file: File) {
  const fd = new FormData();
  fd.append('image', file);

  const originalFilename = file.name.split('.').slice(0, -1).join('.');
  const originalExtension = file.name.split('.').pop();
  // send fd to s3
  const { uploadUrl, tagline, previewUrl, filePath, error } = await callAdminForthApi({
      path: `/plugin/${props.meta.uploadPluginInstanceId}/get_file_upload_url`,
      method: 'POST',
      body: {
        originalFilename,
        contentType: file.type,
        size: file.size,
        originalExtension,
      },
  });

  if (error) {
    adminforth.alert({
      message: `${t('File was not uploaded because of error:')} ${error}`,
      variant: 'danger'
    });
    return;
  }

  const xhr = new XMLHttpRequest();
  const success = await new Promise((resolve) => {
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        imageProgress.value = Math.round((e.loaded / e.total) * 100);
      }
    };
    xhr.addEventListener('loadend', () => {
      const success = xhr.readyState === 4 && xhr.status === 200;
      // try to read response
      resolve(success);
    });
    xhr.open('PUT', uploadUrl, true);
    xhr.setRequestHeader('Content-Type', file.type);
    xhr.setRequestHeader('x-amz-tagging', tagline);
    xhr.send(file);
  });
  if (!success) {
    adminforth.alert({
      messageHtml: `<div>${t('Sorry but the file was not uploaded because of storage adapter Request Error:')} </div>
      <pre style="white-space: pre-wrap; word-wrap: break-word; overflow-wrap: break-word; max-width: 100%;">${
        xhr.responseText.replace(/</g, '&lt;').replace(/>/g, '&gt;')
      }</pre>`,
      variant: 'danger',
      timeout: 30,
    });
    imageProgress.value = 0;
    return;
  }

  // here we have s3Path, call createResource to save the image
  const range = quill.getSelection();
    const formats = quill.getFormat(range.index);

    // here we are checking if we are inside table cell
    if (formats['table-cell']) {
      // if so, we insert image_embed blot in order not to break table structure
      quill.insertEmbed(range.index, 'image_embed', { 
        url: previewUrl, 
        s3Path: filePath, 
        alt: file.name 
      }, 'user');
    } else {
      quill.insertEmbed(range.index, 'image', { 
        url: previewUrl, 
        s3Path: filePath, 
        alt: file.name 
      }, 'user');
    }
}

async function imageHandler() {
  const input = document.createElement('input');
  input.setAttribute('type', 'file');
  input.click();

  // Listen upload local image and save to server
  input.onchange = () => {
    const file = input.files[0];

    // file type is only image.
    if (/^image\//.test(file.type)) {
      saveToServer(file);
    } else {
      console.warn('You could only upload images.');
    }
  };
}

const quillOptions = {
    theme: "snow",
    readOnly:props.column?.editReadonly,
    placeholder: 'Type here...',
    // formats : ['complete'],
    modules: {
      toolbar: {
        container: props.meta.toolbar || [
            ['bold', 'italic', 'underline', 'strike'],        // toggled buttons
            ['table-better'],
            ['blockquote', 'code-block', 'link', ...props.meta.uploadPluginInstanceId ? ['image'] : []],
            // [
            //   // 'image', 
            //   // 'video', 
            //   // 'formula'
            // ],
            

            [{ 'header': 2 }, { 'header': 3 }],               // custom button values
            [{ 'list': 'ordered'}, { 'list': 'bullet' }, { 'list': 'check' }],
            // [{ 'script': 'sub'}, { 'script': 'super' }],      // superscript/subscript
            // [{ 'indent': '-1'}, { 'indent': '+1' }],          // outdent/indent
            // [{ 'direction': 'rtl' }],                         // text direction

            // [{ 'size': ['small', false, 'large', 'huge'] }],  // custom dropdown
            // [{ 'header': [1, 2, 3, 4, 5, 6, false] }],

            // [{ 'color': [] }, { 'background': [] }],          // dropdown with defaults from theme
            // [{ 'font': [] }],
            [{ 'align': [] }],

            ['clean']                                         // remove formatting button
        ],
        handlers: {
          image: imageHandler,
        },
      },
      'table-better': {
        language: 'en_US',
        menus: ['column', 'row', 'merge', 'table', 'cell', 'wrap', 'delete'],
        toolbarTable: true
      },
      keyboard: {
        bindings: {
          tab: {
            key: 9,
            handler: function (range: any, context: any) {
              if (completion.value !== null) {
                return true;
              }
            },
          },
        },
      }
    },
  };

  function initValue(quill: Quill) {
    const html = props.record[props.column.name] || '';
    const delta = quill.clipboard.convert({ html });
    const [range] = quill.selection.getRange();
    try {
      quill.updateContents(delta, Quill.sources.USER);
      quill.setSelection(
        delta.length() - (range?.length || 0),
        Quill.sources.SILENT
      );
      quill.scrollSelectionIntoView();
    } catch (e) {
      console.error('Error while setting initial quill content', e);
      return {ok: false, error: "Invalid initial content"};
    }
    return {ok: true};
  }

onMounted(() => {
  currentValue.value = props.record[props.column.name] || '';
  quill = new Quill(editor.value as HTMLElement, quillOptions);
  const initResult = initValue(quill);
  if (!initResult.ok) {
    adminforth.alert({
      message: `Error while initializing rich text editor: ${initResult.error}`,
      variant: 'danger',
    });
    wasQuillInitializedSuccessfully.value = false;
    return;
  }
  wasQuillInitializedSuccessfully.value = true;
  lastText = quill.getText();
  const linkButton = document.querySelector('.ql-link');
  if (linkButton) {
    linkButton.classList.add('ql-table-button-disabled');
  }

  quill.on(Quill.events.TEXT_CHANGE, async (delta: any, oldDelta: any, source: string) => {
    dbg('🪽 TEXT_CHANGE fired ', delta, oldDelta, source);
    updaterQueue.add(emitTextUpdate);
    startCompletion();
  });
  
  quill.on('selection-change', (range: any, oldRange: any, source: string) => {
    dbg('🪽 selection changed', range, oldRange, source);
    if (range === null) {
      // blur event
      removeCompletionOnBlur();
      editorFocused.value = false;
      return;
    } else {
      editorFocused.value = true;
      startCompletion();
    }
    if (range.length > 0) {
      selectedSymbolsCount.value = range.length;
      linkButton?.classList.remove('ql-table-button-disabled');
    } else {
      selectedSymbolsCount.value = 0;
      linkButton.classList.add('ql-table-button-disabled');
    }
    const text = quill.getText();
    // don't allow to select after completion
    // TODO
    // if (range?.index === text.length) {
    //   console.log('RANGE IDX', range.index, text.length, 'text', JSON.stringify(text, null, 1));
    //   dbg('✋ prevent selection after completion');
    //   quill.setSelection(text.length - 1, 0, 'silent');
    // }
  });


  // handle right swipe on mobile uding document/window, and console log if swiped in right direction
  if ('ontouchstart' in window) {
    document.addEventListener('touchstart', handleTouchStart, false);
    document.addEventListener('touchmove', handleTouchMove, false);
  }
});

function textAreaClickHandler() {
  const linkButton = document.querySelector('.ql-link');
  if (linkButton && selectedSymbolsCount.value === 0) {
    linkButton.classList.add('ql-table-button-disabled');
  }
}

function isQuillContentEmpty(html: string): boolean {
  const div = document.createElement('div');
  div.innerHTML = html;

  div.querySelectorAll('br').forEach(br => br.remove());

  div.querySelectorAll('p').forEach(p => {
    if (!p.textContent?.trim()) {
      p.remove();
    }
  });

  const text = div.textContent?.trim();
  return !text;
}

async function emitTextUpdate() {
  const editorHtml = quill.root.innerHTML;
  // remove completion from html
  let html = editorHtml.replace(/<span[^>]*completer[^>]*>.*?<\/span>/g, '');
  
  // remove all style attributes
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, 'text/html');
  const elements = doc.querySelectorAll('*');
  elements.forEach((el) => {
    el.removeAttribute('style');
  });
  html = doc.body.innerHTML;

  if (lastText === html) {
    return;
  }

  lastText = html;

  await (new Promise((resolve) => setTimeout(resolve, 0)));
  const isEmpty = isQuillContentEmpty(html);
  dbg('⬆️ emit value suggestion-input', html);
  // console.log('⬆️ emit value suggestion-input', html);
  emit('update:value', html);
}

// Auto-Completion functions
let tmt: null | ReturnType<typeof setTimeout> = null;

let xDown: null | number = null;
let yDown: null | number = null;

function handleTouchStart(evt: TouchEvent) {
  xDown = evt.touches[0].clientX;
  yDown = evt.touches[0].clientY;
}

function handleTouchMove(evt: TouchEvent) {
  if (!xDown || !yDown) {
    return;
  }

  let xUp = evt.touches[0].clientX;
  let yUp = evt.touches[0].clientY;

  let xDiff = xDown - xUp;
  let yDiff = yDown - yUp;

  if (Math.abs(xDiff) > Math.abs(yDiff)) {
    if (xDiff < 0) {
      // complete word if completion and input is focused
      dbg('👇 swipe right', completion.value, editorFocused.value);
      if (completion.value !== null && editorFocused.value) {
        approveCompletion('word');
        // [Intervention] Unable to preventDefault inside passive event listener due to target being treated as passive. See https://www.chromestatus.com/feature/5093566007214080
        // evt.preventDefault();
        evt.stopPropagation();
      }
    }
  }

  xDown = null;
  yDown = null;
}

onUnmounted(() => {
  quill.off(Quill.events.TEXT_CHANGE);
  quill.off('selection-change');

  if ('ontouchstart' in window) {
    document.removeEventListener('touchstart', handleTouchStart);
    document.removeEventListener('touchmove', handleTouchMove);
  }
});


async function complete(textBeforeCursor: string) {
  const res = await callAdminForthApi({
      path: `/plugin/${props.meta.pluginInstanceId}/doComplete`,
      method: 'POST',
      body: {
        record: {...props.record, [props.column.name]: textBeforeCursor},
      },
  });

  return res.completion;
}

function updateCompleteEmbed(text: string) {
  const curCursorPos = quill.getSelection();
  const d = quill.getContents();
  const c = d.ops.find((op: any) => op.insert.complete);
  if (!c) {
    return;
  }
  c.insert.complete.text = text;
  quill.setContents(d.ops, 'silent');
  quill.setSelection(curCursorPos.index, curCursorPos.length, 'silent');
}

function deleteCompleteEmbed() {
  const completeNode = quill.root.querySelector('[completer]');
  const completeBlot = Quill.find(completeNode);
  const blotIdx: number | null = completeBlot ? quill.getIndex(completeBlot) : null;

  dbg('👇 complete blot idx', blotIdx);

  if (blotIdx !== null) {
    quill.deleteText(blotIdx, 1, 'silent');
  }
}

function approveCompletion(type: 'all' | 'word') { 
  if (!props.meta.shouldComplete) {
    return;
  }

  dbg('💨 approveCompletion')

  if (completion.value === null) {
    return;
  }

  const cursorPosition = quill.getSelection();

  let shouldComplete = false;
  if (type === 'all') {
    dbg(`👇 insert all at ${cursorPosition.index}, ${completion.value.join('')}`);
    deleteCompleteEmbed();
    quill.insertText(cursorPosition.index, completion.value.join(''), 'silent');
    shouldComplete = true;
  } else {
    const word = completion.value[0];
    quill.insertText(cursorPosition.index, word, 'silent');
    completion.value = completion.value.slice(1);
    if (completion.value.length === 0) {
      shouldComplete = true;
    } else {
      // update completion
      // TODO probably better way to update Embed?
      updateCompleteEmbed(completion.value.join(''));
    }
  }

  updaterQueue.add(emitTextUpdate);

  if (shouldComplete) {
    startCompletion();
  }

}

async function startCompletion() {
  if (!props.meta.shouldComplete || props.column?.editReadonly ) {
    return;
  }
  completion.value = null;
  deleteCompleteEmbed();

  if (tmt) {
    clearTimeout(tmt);
  }
  tmt = setTimeout(async () => {
    const currentTmt = tmt;
    const cursorPosition = quill.getSelection();
    dbg('👇 get pos', cursorPosition.index, cursorPosition.length)
    if (cursorPosition.length !== 0) {
      // we will not complete if text selected
      return;
    }

    const charAfterCursor = quill.getText(cursorPosition.index, 1);
    dbg('👇 charAfterCursor', charAfterCursor);
    if (charAfterCursor !== '\n') {
      // we will not complete if not at the end of the line
      return;
    }

    const textBeforeCursor = quill.getText(0, cursorPosition.index);

    const completionAnswer = await complete(textBeforeCursor);
    if (currentTmt !== tmt) {
      // while we were waiting for completion, new completion was started
      return;
    }

    quill.insertEmbed(cursorPosition.index, 'complete', { text: completionAnswer.join('') }, 'silent');

    //dbg('👇 set pos', cursorPosition.index, cursorPosition.length)
    //quill.setSelection(cursorPosition.index, cursorPosition.length, 'silent');

    completion.value = completionAnswer;

    dbg('👇 completion finished', quill.getContents());

  }, props.meta.debounceTime || 300);
}

function removeCompletionOnBlur() {
  if (lastText?.trim().length === 0) {
    completion.value = null;
    const d = quill.getContents();
    const i = d.ops.findIndex((op: any) => op.insert.complete);
    if (i !== -1) {
      d.ops.splice(i, 1);
      quill.setContents(d, 'silent');
      dbg('🧹 Cleaned completion from ops to make ph visible');
    }
  }
}

</script>

<style lang="scss">

.af-quill-editor {

  // the toolbar sits on a hairline over the text, its buttons are quiet like AFCL ghost icon buttons
  .ql-toolbar.ql-snow[class] {
    @apply flex flex-wrap items-center gap-0.5 border-0 border-b border-lightInputBorder px-1.5 py-1 dark:border-darkInputBorder;
    font-family: inherit;

    .ql-formats {
      @apply inline-flex items-center gap-0.5 mr-2;
    }

    button, .ql-picker-label {
      @apply w-7 h-7 p-1.5 rounded-default border-0 text-lightInputIconColor transition-colors dark:text-darkInputIconColor;
    }

    button svg, .ql-picker-label svg {
      @apply w-4 h-4;
    }

    button:hover, .ql-picker-label:hover {
      @apply bg-lightSecondaryDarken text-lightInputText dark:bg-darkSecondaryLighten dark:text-darkInputText;
    }

    button.ql-active, .ql-picker-label.ql-active, .ql-picker-item.ql-selected {
      @apply text-lightPrimary dark:text-darkPrimary;
    }

    .ql-picker.ql-icon-picker {
      @apply w-7 h-7;
    }

    .ql-picker-options {
      @apply p-1 rounded-lg border-0 bg-lightDropdownOptionsBackground shadow-popover dark:bg-darkDropdownOptionsBackground dark:shadow-darkPopover;
    }

    .ql-picker-item {
      @apply w-7 h-7 p-1.5 rounded-md text-lightInputIconColor hover:bg-lightDropdownOptionsHoverBackground hover:text-lightInputText dark:text-darkInputIconColor dark:hover:bg-darkDropdownOptionsHoverBackground dark:hover:text-darkInputText;
    }
  }

  .ql-container {
    border: 0;
    font-family: inherit;
    font-size: inherit;

    .ql-editor {
      position: relative;
      @apply px-3 py-2.5 text-sm;
      min-height: 100px;
      &.ql-blank::before {
        @apply left-3 right-3 not-italic text-lightInputPlaceholderText dark:text-darkInputPlaceholderText;
      }
    }
  }

  .ql-editor [completer] {
    @apply italic text-lightInputPlaceholderText dark:text-darkInputPlaceholderText;
  }

  // basic typography
  .ql-editor {
    line-height: 1.5;

    /* Headings */
    h1, h2, h3, h4, h5, h6 {
      margin-top: 0.5rem;
      margin-bottom: 0.25rem;
    }

    /* Paragraphs */
    p {
      margin-bottom: 0.4rem;
    }

    /* Lists */
    ul, ol {
      margin-top: 0.4rem;
      margin-bottom: 0.4rem;
      padding-left: 1rem;
    }

    /* List items */
    li {
      margin-bottom: 0.2rem;
    }

    /* Horizontal rule */
    hr {
      margin: 0.8rem 0;
      @apply border-0 border-t border-lightInputBorder dark:border-darkInputBorder;
    }

    /* Blockquotes */
    blockquote {
      margin: 0.8rem 0;
      padding-left: 0.8rem;
      @apply border-0 border-l-2 border-lightInputBorder text-lightInputIconColor dark:border-darkInputBorder dark:text-darkInputIconColor;
    }

    /* Preformatted text */
    pre {
      margin: 0.8rem 0;
      padding: 0.4rem;
      @apply rounded bg-lightSecondaryDarken text-lightInputText dark:bg-darkSecondaryLighten dark:text-darkInputText;
    }

    /* Tables */
    table {
      margin: 0.8rem 0;
      border-collapse: collapse;
    }

    th, td {
      padding: 0.3rem;
      @apply border border-lightInputBorder dark:border-darkInputBorder;
    }

    thead th {
      @apply border-b-2;
    }

    /* Images */
    img {
      margin: 0.4rem 0;
      max-width: 100%;
      height: auto;
    }
  }

  // Quill's own icons which are left (table menus) follow the muted icon color too
  .ql-snow .ql-stroke {
    @apply stroke-lightInputIconColor dark:stroke-darkInputIconColor;
  }

  .ql-snow .ql-fill {
    @apply fill-lightInputIconColor dark:fill-darkInputIconColor;
  }

  .ql-snow .ql-active .ql-stroke {
    @apply stroke-lightPrimary dark:stroke-darkPrimary;
  }

  .ql-snow .ql-active .ql-fill {
    @apply fill-lightPrimary dark:fill-darkPrimary;
  }

}

.ql-table-button-disabled {
  background-color: theme('colors.lightInputBackground') !important;
  svg path  {
    opacity: 0.5;
  } 
  svg .ql-stroke, svg .ql-fill {
    stroke: theme('colors.lightPrimary') !important;
    opacity: 0.5 !important;
  }
}

.dark .ql-table-button-disabled {
  background-color: theme('colors.darkInputBackground') !important;
  svg path  {
    opacity: 0.5;
  }
  svg .ql-stroke, svg .ql-fill {
    stroke: theme('colors.darkPrimary') !important;
    opacity: 0.5 !important;
  }
}

.ql-table-select-container {
  @apply dark:bg-darkForm;
  @apply bg-lightForm;
}

.ql-table-select-label {
  @apply dark:text-darkInputText;
  @apply text-lightInputText;
}

.ql-table-select-list span {
  @apply dark:border-darkInputBorder;
  @apply border-lightInputBorder;
  @apply bg-lightInputBackground;
  @apply dark:bg-darkInputBackground;
}

.ql-table-menus-container, .ql-table-dropdown-list, .ql-table-properties-form {

  h2 {
    @apply dark:text-darkInputText;
    @apply text-lightInputText;
  }

  @apply dark:bg-darkForm;
  @apply bg-lightForm;
  @apply dark:border-darkInputBorder;
  @apply border-lightInputBorder;
  
  .ql-table-dropdown svg path {
    @apply fill-lightPrimary;
    @apply dark:fill-darkPrimary;
    @apply stroke-lightInputText;
    @apply dark:stroke-darkInputText;
  }

  .ql-table-dropdown[data-category="delete"] svg path {
    @apply fill-lightForm;
    @apply dark:fill-darkForm;
    @apply stroke-lightPrimary;
    @apply dark:stroke-darkPrimary;
  }

  .ql-table-dropdown[data-category="delete"]:hover svg path {
    @apply fill-lightInputBackground;
    @apply dark:fill-darkInputBackground;
  }

  .ql-table-dropdown > span.ql-table-tooltip-hover svg:nth-of-type(2) path {
    @apply fill-lightForm stroke-lightPrimary;
    @apply dark:fill-darkForm dark:stroke-darkPrimary;
  }

  .ql-table-dropdown:hover > span.ql-table-tooltip-hover svg:nth-of-type(2) path {
    @apply fill-lightInputBackground stroke-lightPrimary;
    @apply dark:fill-darkInputBackground dark:stroke-darkPrimary;
  }

  .properties-form-action-row {
    button {
      @apply dark:bg-darkEditViewButtonBackground;
      @apply bg-lightEditViewButtonBackground;
      @apply dark:text-darkInputText;
      @apply text-lightInputText;

      &:hover {
        @apply dark:bg-darkEditViewButtonBackgroundHover;
        @apply bg-lightEditViewButtonBackgroundHover;
      }
    }
  }

  label {
    top: -75%;
    @apply dark:text-darkInputText;
    @apply text-lightInputText;
    @apply dark:bg-darkForm;
    @apply bg-lightForm;
  }
}

  .ql-table-properties-form .properties-form-row .ql-table-check-container .ql-table-btns-checked  {
    @apply dark:bg-darkEditViewButtonBackgroundHover;
    @apply bg-lightEditViewButtonBackgroundHover;
    svg path {
      @apply dark:stroke-darkPrimary;
      @apply stroke-lightPrimary;
    }
  }

  .ql-table-properties-form .properties-form-row .ql-table-check-container .ql-table-tooltip-hover:hover  {
    @apply dark:bg-darkEditViewButtonBackgroundHover;
    @apply bg-lightEditViewButtonBackgroundHover;
  }

  .ql-table-dropdown-properties .ql-table-dropdown-list li:hover {
    @apply dark:bg-darkInputBackground;
    @apply bg-lightInputBackground;
  }

  .ql-table-dropdown,.ql-table-dropdown li {
    &:hover {
    @apply dark:bg-darkInputBackground;
    @apply bg-lightInputBackground;
  }
}

.ql-table-switch-inner[aria-checked="true"] {
  @apply dark:bg-darkPrimary;
  @apply bg-lightPrimary;
}

.ql-table-triangle-down::before, .ql-table-triangle-down::after {
  border-bottom: theme('colors.lightForm') !important;
}

.dark .ql-table-triangle-down::before,.dark .ql-table-triangle-down::after {
  border-bottom: theme('colors.darkForm') !important;
}

.ql-table-color-container .color-picker .color-picker-select {
  @apply dark:bg-darkForm;
  @apply bg-lightForm;
  .erase-container:hover {
    @apply dark:bg-darkInputBackground;
    @apply bg-lightInputBackground;
  }
  svg path {
    @apply dark:stroke-darkPrimary;
    @apply stroke-lightPrimary;
  }
  .color-picker-palette {
    @apply dark:bg-darkForm;
    @apply bg-lightForm;
  }
}

.properties-form-row .ql-table-check-container {
  visibility: hidden;
}

</style>