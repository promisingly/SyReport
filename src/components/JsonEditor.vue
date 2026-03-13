<template>
  <div class="json-editor-wrap">
    <VueMonacoEditor
      v-model:value="innerValue"
      language="json"
      theme="vs-dark"
      :options="editorOptions"
      @mount="handleMount"
    />
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { VueMonacoEditor } from '@guolao/vue-monaco-editor'

const props = defineProps({
  modelValue: {
    type: String,
    default: ''
  },
  readOnly: {
    type: Boolean,
    default: false
  },
  height: {
    type: String,
    default: '320px'
  }
})

const emit = defineEmits(['update:modelValue'])

const innerValue = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})

const editorOptions = computed(() => ({
  readOnly: props.readOnly,
  automaticLayout: true,
  minimap: { enabled: false },
  scrollBeyondLastLine: false,
  fontSize: 13,
  wordWrap: 'on',
  formatOnPaste: true,
  formatOnType: true,
  tabSize: 2,
  lineNumbers: 'on',
  folding: true,
  scrollbar: {
    verticalScrollbarSize: 8,
    horizontalScrollbarSize: 8
  }
}))

function handleMount(editor) {
  try {
    editor.getAction('editor.action.formatDocument')?.run()
  } catch (e) {
    console.debug('Format action not available:', e)
  }
}
</script>

<style scoped>
.json-editor-wrap {
  border: 1px solid #dcdfe6;
  border-radius: 8px;
  overflow: hidden;
  background: #1e1e1e;
}
</style>
