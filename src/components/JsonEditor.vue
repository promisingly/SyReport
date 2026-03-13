<template>
  <div class="json-editor-wrap" :style="{ height: height }">
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
import { computed, ref } from 'vue'
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
    default: '100%'
  }
})

const emit = defineEmits(['update:modelValue'])

const editorRef = ref(null)

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
  bracketPairColorization: { enabled: true }
}))

function handleMount(editor) {
  editorRef.value = editor
  try {
    editor.getAction('editor.action.formatDocument')?.run()
  } catch (e) {
    // ignore
  }
}

function format() {
  try {
    editorRef.value?.getAction('editor.action.formatDocument')?.run()
  } catch (e) {
    // ignore
  }
}

defineExpose({ format })
</script>

<style scoped>
.json-editor-wrap {
  border: 1px solid #dcdfe6;
  border-radius: 6px;
  overflow: hidden;
  background: #1e1e1e;
}

.json-editor-wrap :deep(.monaco-editor) {
  border-radius: 6px;
}
</style>
