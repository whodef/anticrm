<!--
// Copyright © 2020, 2021 Anticrm Platform Contributors.
// Copyright © 2021 Hardcore Engineering Inc.
// 
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
// 
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// 
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { IntlString, translate } from '@hcengineering/platform'

  import { FocusPosition } from '@tiptap/core'
  import { AnyExtension, Editor, Extension, HTMLContent } from '@tiptap/core'

  import Placeholder from '@tiptap/extension-placeholder'

  import { createEventDispatcher, onDestroy, onMount } from 'svelte'
  import textEditorPlugin from '../plugin'
  import { defaultExtensions } from './extensions'
  import { Node as ProseMirrorNode } from '@tiptap/pm/model'
  import { themeStore } from '@hcengineering/ui'
  import TextEditorStyleToolbar from './TextEditorStyleToolbar.svelte'
  import { TextFormatCategory } from '../types'
  import { InlineStyleToolbar } from './extension/inlineStyleToolbar'

  export let content: string = ''
  export let placeholder: IntlString = textEditorPlugin.string.EditorPlaceholder
  export let extensions: AnyExtension[] = []
  export let textFormatCategories: TextFormatCategory[] = []
  export let supportSubmit = true
  export let isEmpty = true

  let element: HTMLElement
  let editor: Editor

  let placeHolderStr: string = ''

  $: ph = translate(placeholder, {}, $themeStore.language).then((r) => {
    placeHolderStr = r
  })

  const dispatch = createEventDispatcher()

  export function isEditable (): boolean {
    return editor.isEditable
  }
  export function setEditable (editable: boolean): void {
    if (editor) editor.setEditable(editable)
  }
  export function submit (): void {
    content = editor.getHTML()
    dispatch('content', content)
  }
  export function setContent (newContent: string): void {
    if (content !== newContent) {
      content = newContent
      editor.commands.setContent(content)
      isEmpty = editor.isEmpty
    }
  }
  export function clear (): void {
    content = ''
    editor.commands.clearContent(false)

    // editor.commands.clearContent false as argument prevent from onUpdate
    // so if we want to stay in sync with editor.isEmpty we need to do this manually
    isEmpty = true
  }
  export function insertText (text: string): void {
    editor.commands.insertContent(text as HTMLContent)
  }

  export function isEmptyContent (): boolean {
    return isEmpty
  }

  let needFocus = false
  let focused = false
  let posFocus: FocusPosition | undefined = undefined
  let showContextMenu = false
  let textEditorToolbar: HTMLElement

  export function focus (position?: FocusPosition): void {
    posFocus = position
    needFocus = true
  }

  $: if (editor && needFocus) {
    if (!focused) {
      editor.commands.focus(posFocus)
      posFocus = undefined
    }
    needFocus = false
  }

  const Handle = Extension.create({
    addKeyboardShortcuts () {
      return {
        'Ctrl-Enter': () => {
          const res = this.editor.commands.splitListItem('listItem')
          if (!res) {
            this.editor.commands.first(({ commands }) => [
              () => commands.newlineInCode(),
              () => commands.createParagraphNear(),
              () => commands.liftEmptyBlock(),
              () => commands.splitBlock()
            ])
          }
          return true
        },
        'Shift-Enter': () => {
          this.editor.commands.setHardBreak()
          return true
        },
        Enter: () => {
          submit()
          return true
        },
        Space: () => {
          if (editor.isActive('link')) {
            this.editor.commands.toggleMark('link')
          }
          return false
        }
      }
    }
  })

  onMount(() => {
    ph.then(() => {
      editor = new Editor({
        element,
        content,
        extensions: [
          ...defaultExtensions,
          ...(supportSubmit ? [Handle] : []), // order important
          Placeholder.configure({ placeholder: placeHolderStr }),
          ...extensions,
          InlineStyleToolbar.configure({
            element: textEditorToolbar,
            getEditorElement: () => element,
            isShown: () => showContextMenu
          })
        ],
        parseOptions: {
          preserveWhitespace: 'full'
        },
        onTransaction: () => {
          // force re-render so `editor.isActive` works as expected
          editor = editor
        },
        onBlur: ({ event }) => {
          focused = false
          dispatch('blur', event)
        },
        onFocus: () => {
          focused = true
          dispatch('focus', editor.getHTML())
        },
        onUpdate: () => {
          content = editor.getHTML()
          isEmpty = editor.isEmpty
          showContextMenu = false
          dispatch('value', content)
          dispatch('update', content)
        },
        onCreate: () => {
          isEmpty = editor.isEmpty
        },
        onSelectionUpdate: () => {
          showContextMenu = false
        }
      })
    })
  })

  onDestroy(() => {
    if (editor) {
      editor.destroy()
    }
  })

  function onEditorClick () {
    if (!editor.isEmpty) {
      showContextMenu = true
    }
  }

  /**
   * @public
   */
  export function removeNode (nde: ProseMirrorNode): void {
    const deleteOp = (n: ProseMirrorNode, pos: number) => {
      if (nde === n) {
        // const pos = editor.view.posAtDOM(nde, 0)
        editor.view.dispatch(editor.view.state.tr.delete(pos, pos + 1))
      }
      n.descendants(deleteOp)
    }
    editor.view.state.doc.descendants((n, pos) => {
      deleteOp(n, pos)
    })
  }
</script>

<div class="formatPanel buttons-group xsmall-gap mb-4" bind:this={textEditorToolbar}>
  <TextEditorStyleToolbar
    textEditor={editor}
    {textFormatCategories}
    on:focus={() => {
      needFocus = true
    }}
  />
</div>
<div class="select-text" style="width: 100%;" on:mousedown={onEditorClick} bind:this={element} />

<style lang="scss" global>
  .ProseMirror {
    overflow-y: auto;
    font: inherit;
    min-height: inherit !important;
    max-height: inherit !important;
    outline: none;
    line-height: 150%;
    color: var(--theme-caption-color);

    p:not(:last-child) {
      margin-block-end: 1em;
    }

    > * + * {
      margin-top: 0.75em;
    }

    /* Placeholder (at the top) */
    p.is-editor-empty:first-child::before {
      content: attr(data-placeholder);
      float: left;
      color: var(--theme-halfcontent-color);
      pointer-events: none;
      height: 0;
    }
    &:focus-within p.is-editor-empty:first-child::before {
      color: var(--theme-trans-color);
    }

    &::-webkit-scrollbar-thumb {
      background-color: var(--scrollbar-bar-color);
    }
    &::-webkit-scrollbar-thumb:hover {
      background-color: var(--scrollbar-bar-hover);
    }
    &::-webkit-scrollbar-corner {
      background-color: var(--scrollbar-bar-color);
    }
    &::-webkit-scrollbar-track {
      margin: 0;
    }
  }

  .formatPanel {
    margin: -0.5rem -0.25rem 0.5rem;
    padding: 0.375rem;
    background-color: var(--theme-comp-header-color);
    border-radius: 0.5rem;
    box-shadow: var(--theme-popup-shadow);
    z-index: 1;
  }
</style>
