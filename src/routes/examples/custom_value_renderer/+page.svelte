<script lang="ts">
  import {
    EditableValue,
    EnumValue,
    JSONEditor,
    renderValue,
    type RenderValueComponentDescription,
    type RenderValueProps
  } from 'svelte-jsoneditor'
  import ReadonlyPassword from '../../components/ReadonlyPassword.svelte'
  import { EvaluatorAction } from '../../components/EvaluatorAction'
  import EditableDiv from 'svelte-jsoneditor/components/controls/EditableDiv.svelte'
  import EditableCodeMirror from 'svelte-jsoneditor/components/controls/EditableCodeMirror.svelte'
  import { EditorState } from '@codemirror/state'
  import { EditorView } from '@codemirror/view'
  import { onMount } from 'svelte'
  import {
    autocompletion,
    type CompletionContext,
    type CompletionResult
  } from '@codemirror/autocomplete'

  let content = {
    text: undefined, // can be used to pass a stringified JSON document instead
    json: {
      username: 'John',
      password: 'secret...',
      gender: 'male',
      evaluate: '2 + 3',
      codemirror: 'test'
    }
  }

  const genderOptions = [
    { value: null, text: '-' },
    { value: 'male', text: 'Male' },
    { value: 'female', text: 'Female' },
    { value: 'other', text: 'Other' }
  ]

  function onRenderValue(props: RenderValueProps): RenderValueComponentDescription[] {
    const {
      path,
      value,
      readOnly,
      enforceString,
      isEditing,
      parser,
      normalization,
      selection,
      onPatch,
      onPasteJson,
      onSelect,
      onFind,
      findNextInside,
      focus
    } = props

    const key = props.path[props.path.length - 1]
    if (key === 'password' && !isEditing) {
      return [
        {
          component: ReadonlyPassword,
          props: {
            value,
            path,
            readOnly,
            onSelect
          }
        }
      ]
    }

    if (key === 'gender') {
      return [
        {
          component: EnumValue,
          props: {
            value,
            path,
            readOnly,
            parser,
            onPatch,
            selection,
            options: genderOptions
          }
        }
      ]
    }

    if (key === 'evaluate' && !isEditing) {
      return [
        {
          action: EvaluatorAction,
          props: {
            value,
            path,
            readOnly,
            onSelect
          }
        }
      ]
    }

    if (key === 'codemirror' && isEditing) {
      return [
        {
          component: EditableValue,
          props: {
            path,
            value,
            enforceString,
            parser,
            normalization,
            onPatch,
            onPasteJson,
            onSelect,
            onFind,
            findNextInside,
            focus,
            editableComponent: EditableCodeMirror
          }
        }
      ]
    }

    // fallback on the default render components
    return renderValue(props)
  }

  let codeMirrorRef: HTMLDivElement
  let codeMirrorView: EditorView
  let editorState: EditorState

  onMount(() => {
    try {
      codeMirrorView = createCodeMirrorView({
        target: codeMirrorRef,
        initialText: 'ola'
      })
    } catch (err) {
      // TODO: report error to the user
      console.error(err)
    }
  })

  function myCompletions(context: CompletionContext): CompletionResult | null {
    let before = context.matchBefore(/\w+/)

    const completions = [
      { label: 'panic', type: 'keyword' },
      { label: 'park', type: 'constant', info: 'Test completion' },
      { label: 'password', type: 'variable' }
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
          editorState = update.state

          if (update.docChanged) {
            console.log('update.state.doc.toString()', update.state.doc.toString())
          }
        }),
        autocompletion({ override: [myCompletions] })
      ]
    })

    codeMirrorView = new EditorView({
      state,
      parent: target
    })

    codeMirrorView.contentDOM.addEventListener('mousedown', (event) => {
      // stop propagation to prevent selecting the whole line
      console.log('mousedown std')
      event.stopPropagation()
    })

    return codeMirrorView
  }
</script>

<svelte:head>
  <title>Custom value renderer (password, enum, action) | svelte-jsoneditor</title>
</svelte:head>

<h1>Custom value renderer (password, enum, action)</h1>

<div bind:this={codeMirrorRef} />

<p>
  Provide a custom <code>onRenderValue</code> method, which demonstrates three things:
</p>
<ol>
  <li>
    It hides the value of all fields with the name "password" using a Svelte password component <code
      >ReadonlyPassword</code
    >
  </li>
  <li>
    It creates an enum component for the fields with name "gender" using a Svelte component <code
      >EnumValue</code
    >.
  </li>
  <li>
    The creates a custom component for the field named "evaluate" using a Svelte Action, which
    evaluates the value as an expression containing an addition of two or more values. This solution
    can be used when using svelte-jsoneditor in a Vanilla JS environment.
  </li>
</ol>

<div class="editor">
  <JSONEditor bind:content {onRenderValue} />
</div>

<style>
  .editor {
    width: 700px;
    height: 400px;
  }
</style>
