<svelte:options immutable={true} />

<script lang="ts">
  import { onDestroy, onMount } from 'svelte'
  import { keyComboFromEvent } from 'svelte-jsoneditor'
  import { noop } from 'lodash-es'
  import type { OnFind, OnPaste } from 'svelte-jsoneditor'
  import { UpdateSelectionAfterChange } from 'svelte-jsoneditor'
  import { EditorView } from '@codemirror/view'
  import { EditorState } from '@codemirror/state'
  import {
    CompletionContext,
    autocompletion,
    completionStatus,
    type CompletionResult
  } from '@codemirror/autocomplete'

  import { createDebug } from 'svelte-jsoneditor/utils/debug'

  const debug = createDebug('jsoneditor:EditableCodeMirror')

  export let value: string
  export let onChange: (newValue: string, updateSelection: UpdateSelectionAfterChange) => void
  export let onCancel: () => void
  export let onFind: OnFind
  export let onPaste: OnPaste = noop
  export let onValueClass: (value: string) => string = () => ''

  let codeMirrorRef: HTMLDivElement
  let codeMirrorView: EditorView

  let valueClass: string
  let is_doc_empty = false

  $: valueClass = onValueClass(value)
  let closed = false

  onMount(() => {
    try {
      codeMirrorView = createCodeMirrorView({
        target: codeMirrorRef,
        initialText: value
      })
    } catch (err) {
      // TODO: report error to the user
      console.error(err)
    }

    is_doc_empty = !value

    debug('onMount', { value })

    // focus
    if (codeMirrorView) {
      // setCursorToEnd(codeMirrorView.contentDOM)
      codeMirrorView.focus()
      const end = codeMirrorView.state.doc.length
      codeMirrorView.dispatch({
        selection: { anchor: end }
      })

      // The refresh method can be used to update the classnames for example
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      codeMirrorView.contentDOM.refresh = () => {
        debug('refresh')
      }

      // The cancel method can be used to cancel editing, without firing a change
      // when the contents did change in the meantime. It is the same as pressing ESC
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      codeMirrorView.contentDOM.cancel = handleCancel

      // svelte-jsoneeditor uses jse-editable-div class internally for certain functionality
      codeMirrorView.contentDOM.classList.add('jse-editable-div')
    }
  })

  function handleCancel() {
    debug('handleCancel')
    closed = true
    onCancel()
  }

  function handleValueKeyDown(event: KeyboardEvent) {
    event.stopPropagation()

    const combo = keyComboFromEvent(event)

    if (combo === 'Escape') {
      // cancel changes (needed to prevent triggering a change onDestroy)
      handleCancel()
    }

    if (combo === 'Enter' || combo === 'Tab') {
      const status = completionStatus(codeMirrorView.state)

      // only close the editable div when the user presses Enter or Tab and there is no autocompletion active
      if (status === null) {
        closed = true
        onChange(codeMirrorView.state.doc.toString(), UpdateSelectionAfterChange.nextInside)
      }
    }

    if (combo === 'Ctrl+F') {
      event.preventDefault()
      onFind(false)
    }

    if (combo === 'Ctrl+H') {
      event.preventDefault()
      onFind(true)
    }
  }

  function handleBlur() {
    debug('handleBlur')

    // we only want to close the editable div when the focus did go to another
    // element on the same page, but not when the user switches to another
    // application or browser tab to copy/paste something whilst still editing
    // the value, hence the check for document.hasFocus()
    if (document.hasFocus() && !closed) {
      closed = true
      onChange(codeMirrorView.state.doc.toString(), UpdateSelectionAfterChange.self)
    }
  }

  onDestroy(() => {
    debug('onDestroy')

    if (!closed) {
      onChange(codeMirrorView.state.doc.toString(), UpdateSelectionAfterChange.no)
    }
    codeMirrorView.destroy()
  })

  function createCodeMirrorView({
    target,
    initialText
  }: {
    target: HTMLDivElement
    initialText: string
  }): EditorView {
    const state = EditorState.create({
      doc: initialText,
      extensions: [
        EditorView.updateListener.of((update) => {
          if (update.docChanged) {
            const newValue = update.state.doc.toString()
            valueClass = onValueClass(newValue)

            is_doc_empty = !newValue
          }
        }),
        EditorView.domEventHandlers({
          mousedown: (event) => {
            // stop propagation to prevent selecting the whole line
            event.stopPropagation()
          },
          keydown: handleValueKeyDown,
          blur: handleBlur,
          paste: handleValuePaste
        }),
        EditorView.lineWrapping,
        autocompletion({ override: [myCompletions] })
      ]
    })

    codeMirrorView = new EditorView({
      state,
      parent: target
    })

    codeMirrorView.focus()

    return codeMirrorView
  }

  function handleValuePaste(event: ClipboardEvent) {
    event.stopPropagation()

    if (!onPaste || !event.clipboardData) {
      return
    }

    const clipboardText = event.clipboardData.getData('text/plain')

    debug('handleValuePaste', { clipboardText })
    onPaste(clipboardText)
  }

  onDestroy(() => {
    codeMirrorView.destroy()
  })

  function myCompletions(context: CompletionContext): CompletionResult | null {
    let before = context.matchBefore(/\w+/)
    const completions = [
      { label: 'form.data.firstname', type: 'keyword' },
      { label: 'form.data.lastname', type: 'keyword' },
      { label: 'form.data.email', type: 'keyword' },
      { label: 'form.data.phone', type: 'keyword' },
      { label: 'form.data.address', type: 'keyword' },
      { label: 'form.data.city', type: 'keyword' },
      { label: 'form.data.zip', type: 'keyword' },
      { label: 'form.data.country', type: 'keyword' }
    ]
    // If completion wasn't explicitly started and there
    // is no word before the cursor, don't open completions.
    if (!context.explicit && !before) return null
    return {
      from: before ? before.from : context.pos,
      options: completions,
      validFor: /^\w*$/
    }
  }
</script>

<div class="jse-editable-cm {valueClass}" bind:this={codeMirrorRef} />

<style>
  .jse-editable-cm {
    outline: 2px solid #656565;
    background-color: inherit !important;
    cursor: text !important;
    padding: 0 5px;
    z-index: 3;
    min-width: 2em;
  }

  .jse-editable-cm.jse-empty:not(:focus-within) {
    outline: 1px dotted rgba(0, 0, 0, 0.2);
    -moz-outline-radius: 2px;
  }

  .jse-editable-cm :global(.cm-scroller) {
    font-family: inherit;
    font-size: inherit;
    line-height: inherit;
    padding: 0;
  }

  .jse-editable-cm :global(.cm-focused) {
    outline: none !important;
  }

  .jse-editable-cm :global(.cm-line) {
    padding: 0;
  }

  .jse-editable-cm :global(.cm-content) {
    padding: 0;
  }

  .jse-editable-cm.jse-value.jse-string :global(.cm-content) {
    color: #008000;
  }

  .jse-editable-cm.jse-value.jse-number :global(.cm-content) {
    color: #ee422e;
  }

  .jse-editable-cm.jse-value.jse-boolean :global(.cm-content) {
    color: #ff8c00;
  }

  .jse-editable-cm.jse-value.jse-null :global(.cm-content) {
    color: #004ed0;
  }

  .jse-editable-cm.jse-value.jse-invalid :global(.cm-content) {
    color: black;
  }

  .jse-editable-cm.jse-value.jse-url :global(.cm-content) {
    color: #008000;
    text-decoration: underline;
  }

  .jse-editable-cm.jse-value.jse-object {
    min-width: 16px;
    color: rgba(0, 0, 0, 0.38);
  }

  .jse-editable-cm.jse-value.jse-array {
    min-width: 16px;
    color: rgba(0, 0, 0, 0.38);
  }
</style>
