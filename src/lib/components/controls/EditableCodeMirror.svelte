<svelte:options immutable={true} />

<script lang="ts">
  import { onDestroy, onMount } from 'svelte'
  import { addNewLineSuffix, removeNewLineSuffix, setCursorToEnd } from '$lib/utils/domUtils.js'
  import { keyComboFromEvent } from '$lib/utils/keyBindings.js'
  import { createDebug } from '$lib/utils/debug.js'
  import { noop } from 'lodash-es'
  import type { OnFind, OnPaste } from '$lib/types'
  import { UpdateSelectionAfterChange } from '$lib/types'
  import { classnames } from '$lib/utils/cssUtils.js'
  import { EditorView, keymap } from '@codemirror/view'
  import { EditorState } from '@codemirror/state'
  import type { ChangeEventHandler } from 'svelte/elements'

  const debug = createDebug('jsoneditor:EditableCodeMirror')

  export let value: string
  export let shortText = false
  export let onChange: (newValue: string, updateSelection: UpdateSelectionAfterChange) => void
  export let onCancel: () => void
  export let onFind: OnFind
  export let onPaste: OnPaste = noop
  export let onValueClass: (value: string) => string = () => ''

  let codeMirrorRef: HTMLDivElement
  let codeMirrorView: EditorView

  let valueClass: string
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
      closed = true
      onChange(codeMirrorView.state.doc.toString(), UpdateSelectionAfterChange.nextInside)
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
        EditorView.lineWrapping
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
</script>

<div class={classnames('jse-editable-cm', valueClass)} bind:this={codeMirrorRef} />

<style src="./EditableCodeMirror.scss"></style>
