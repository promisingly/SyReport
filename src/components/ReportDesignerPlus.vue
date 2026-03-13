<template>
  <div class="designer-shell" @keydown="onKeydown" tabindex="0">
    <!-- Top Toolbar -->
    <div class="top-toolbar">
      <div class="toolbar-left">
        <span class="brand">📊 SyReport 报表设计器</span>
      </div>
      <div class="toolbar-center">
        <el-button-group>
          <el-button size="small" :type="mode === 'design' ? 'primary' : ''" @click="mode = 'design'">
            设计
          </el-button>
          <el-button size="small" :type="mode === 'preview' ? 'primary' : ''" @click="mode = 'preview'">
            预览
          </el-button>
        </el-button-group>
        <el-divider direction="vertical" />
        <el-button size="small" @click="showGrid = !showGrid" :type="showGrid ? 'primary' : ''">
          网格
        </el-button>
        <el-button size="small" @click="zoomIn">放大</el-button>
        <el-button size="small" @click="zoomOut">缩小</el-button>
        <span class="zoom-label">{{ Math.round(zoom * 100) }}%</span>
        <el-divider direction="vertical" />
        <el-button size="small" type="success" @click="exportPDF">导出 PDF</el-button>
        <el-button size="small" @click="exportTemplate">导出模板</el-button>
        <el-button size="small" @click="importTemplateDialog = true">导入模板</el-button>
      </div>
      <div class="toolbar-right">
        <el-button size="small" type="danger" plain @click="clearTemplate">清空</el-button>
      </div>
    </div>

    <!-- Main Layout -->
    <div class="body-layout">
      <!-- Left Panel -->
      <div class="left-panel" v-show="mode === 'design'">
        <!-- Components -->
        <div class="panel-section">
          <div class="panel-section-title">组件</div>
          <div class="component-list">
            <div
              v-for="comp in componentTypes"
              :key="comp.type"
              class="comp-item"
              draggable="true"
              @dragstart="onCompDragStart($event, comp)"
            >
              <span class="comp-icon">{{ comp.icon }}</span>
              <span class="comp-label">{{ comp.label }}</span>
            </div>
          </div>
        </div>

        <!-- Data Fields -->
        <div class="panel-section">
          <div class="panel-section-title">
            数据字段
            <el-tooltip content="从示例数据中解析字段" placement="top">
              <el-button size="small" link @click="parseDataFields">刷新</el-button>
            </el-tooltip>
          </div>
          <div class="field-list">
            <div
              v-for="field in dataFields"
              :key="field.key"
              class="field-item"
              draggable="true"
              @dragstart="onFieldDragStart($event, field)"
            >
              <span class="field-icon">🔑</span>
              <span class="field-label">{{ field.label }}</span>
            </div>
            <div v-if="dataFields.length === 0" class="empty-tip">
              请先在下方填写示例数据
            </div>
          </div>
        </div>

        <!-- Sample Data -->
        <div class="panel-section panel-section-grow">
          <div class="panel-section-title">
            示例数据 JSON
            <el-button size="small" link @click="formatSampleData">格式化</el-button>
          </div>
          <div class="json-editor-container">
            <JsonEditor
              ref="sampleDataEditorRef"
              v-model="previewDataText"
              height="100%"
              @update:modelValue="parseDataFields"
            />
          </div>
          <div v-if="sampleDataError" class="json-error">{{ sampleDataError }}</div>
        </div>
      </div>

      <!-- Canvas Area -->
      <div class="canvas-area">
        <!-- Design Mode -->
        <div v-show="mode === 'design'" class="canvas-scroll">
          <div
            class="canvas-wrapper"
            :style="{ transform: `scale(${zoom})`, transformOrigin: 'top center' }"
          >
            <!-- Page Canvas -->
            <div
              ref="designCanvasRef"
              class="page-canvas"
              :class="{ 'show-grid': showGrid, 'dragging-field': !!draggingField }"
              :style="{
                width: template.page.width + 'px',
                minHeight: template.page.height + 'px',
                paddingLeft: template.page.paddingLeft + 'px',
                paddingRight: template.page.paddingRight + 'px',
                paddingTop: template.page.paddingTop + 'px',
                paddingBottom: template.page.paddingBottom + 'px'
              }"
              @mousedown.self="clearSelection"
              @dragover.prevent
              @drop="onCanvasDrop"
            >
              <!-- Bands -->
              <template v-for="band in template.bands" :key="band.key">
                <div
                  class="band"
                  :class="{ active: currentBand === band.key }"
                >
                  <div
                    class="band-label"
                    @click.stop="selectBand(band)"
                  >
                    <span>{{ band.title }}</span>
                    <span class="band-height-info">{{ band.height }}px</span>
                  </div>
                  <div
                    class="band-body"
                    :style="{ height: band.height + 'px', position: 'relative' }"
                    @mousedown.self="clearSelection"
                    @dragover.prevent
                    @drop="onBandDrop($event, band)"
                  >
                    <!-- Elements in this band -->
                    <div
                      v-for="el in band.elements"
                      :key="el.id"
                      class="design-element"
                      :class="{
                        selected: selectedElement && selectedElement.id === el.id,
                        [el.type]: true
                      }"
                      :style="getElementStyle(el)"
                      @mousedown.stop="startDragElement($event, el, band)"
                      @click.stop="selectElement(el)"
                    >
                      <!-- Text -->
                      <template v-if="el.type === 'text'">
                        <span :style="getTextStyle(el)">{{ el.content }}</span>
                      </template>
                      <!-- Field -->
                      <template v-else-if="el.type === 'field'">
                        <span :style="getTextStyle(el)">{{ getFieldPreview(el) }}</span>
                      </template>
                      <!-- Image -->
                      <template v-else-if="el.type === 'image'">
                        <img
                          v-if="el.src"
                          :src="el.src"
                          :style="{ width: '100%', height: '100%', objectFit: el.objectFit || 'contain' }"
                          draggable="false"
                        />
                        <div v-else class="image-placeholder">🖼 图片</div>
                      </template>
                      <!-- Line -->
                      <template v-else-if="el.type === 'line'">
                        <div class="line-el" :style="getLineStyle(el)" />
                      </template>
                      <!-- Table -->
                      <template v-else-if="el.type === 'table'">
                        <div class="table-placeholder">
                          <div class="table-header-row">
                            <div
                              v-for="col in el.columns"
                              :key="col.key"
                              class="table-th"
                              :style="{ width: col.width + 'px' }"
                            >{{ col.header }}</div>
                          </div>
                          <div class="table-data-row">
                            <div
                              v-for="col in el.columns"
                              :key="col.key"
                              class="table-td"
                              :style="{ width: col.width + 'px' }"
                            >{{ col.field }}</div>
                          </div>
                        </div>
                      </template>
                      <!-- Resize handle -->
                      <div
                        v-if="selectedElement && selectedElement.id === el.id"
                        class="resize-handle resize-se"
                        @mousedown.stop="startResize($event, el)"
                      />
                    </div>
                  </div>
                  <!-- Band resize handle -->
                  <div
                    class="band-resize-handle"
                    @mousedown.stop="startBandResize($event, band)"
                  />
                </div>
              </template>
            </div>
          </div>
        </div>

        <!-- Preview Mode -->
        <div v-show="mode === 'preview'" class="preview-scroll">
          <div
            ref="previewRootRef"
            class="print-root preview-page"
            :style="{
              width: template.page.width + 'px',
              minHeight: template.page.height + 'px',
              paddingLeft: template.page.paddingLeft + 'px',
              paddingRight: template.page.paddingRight + 'px',
              paddingTop: template.page.paddingTop + 'px',
              paddingBottom: template.page.paddingBottom + 'px'
            }"
          >
            <template v-for="band in template.bands" :key="band.key">
              <div :style="{ height: band.height + 'px', position: 'relative', width: '100%' }">
                <div
                  v-for="el in band.elements"
                  :key="el.id"
                  :style="getElementStyle(el)"
                >
                  <template v-if="el.type === 'text'">
                    <span :style="getTextStyle(el)">{{ el.content }}</span>
                  </template>
                  <template v-else-if="el.type === 'field'">
                    <span :style="getTextStyle(el)">{{ getFieldValue(el) }}</span>
                  </template>
                  <template v-else-if="el.type === 'image'">
                    <img
                      v-if="el.src"
                      :src="el.src"
                      :style="{ width: '100%', height: '100%', objectFit: el.objectFit || 'contain' }"
                    />
                  </template>
                  <template v-else-if="el.type === 'line'">
                    <div class="line-el" :style="getLineStyle(el)" />
                  </template>
                  <template v-else-if="el.type === 'table'">
                    <table class="preview-table" :style="{ width: '100%', borderCollapse: 'collapse' }">
                      <thead>
                        <tr>
                          <th
                            v-for="col in el.columns"
                            :key="col.key"
                            :style="{ width: col.width + 'px', border: '1px solid #ccc', padding: '4px', background: '#f5f5f5' }"
                          >{{ col.header }}</th>
                        </tr>
                      </thead>
                      <tbody>
                        <tr
                          v-for="(row, idx) in getTableData(el)"
                          :key="idx"
                        >
                          <td
                            v-for="col in el.columns"
                            :key="col.key"
                            :style="{ border: '1px solid #ccc', padding: '4px' }"
                          >{{ row[col.field] }}</td>
                        </tr>
                      </tbody>
                    </table>
                  </template>
                </div>
              </div>
            </template>
          </div>
        </div>
      </div>

      <!-- Right Panel -->
      <div class="right-panel" v-show="mode === 'design'">
        <!-- Page Properties -->
        <div class="panel-section">
          <div class="panel-section-title">页面属性</div>
          <el-form size="small" label-width="70px" label-position="left">
            <el-form-item label="宽度">
              <el-input-number v-model="template.page.width" :min="200" :max="2000" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="高度">
              <el-input-number v-model="template.page.height" :min="200" :max="5000" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="左边距">
              <el-input-number v-model="template.page.paddingLeft" :min="0" :max="200" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="右边距">
              <el-input-number v-model="template.page.paddingRight" :min="0" :max="200" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="上边距">
              <el-input-number v-model="template.page.paddingTop" :min="0" :max="200" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="下边距">
              <el-input-number v-model="template.page.paddingBottom" :min="0" :max="200" controls-position="right" style="width:100%" />
            </el-form-item>
          </el-form>
        </div>

        <!-- Element Properties -->
        <div v-if="selectedElement" class="panel-section">
          <div class="panel-section-title">
            元素属性
            <el-button size="small" link type="danger" @click="deleteSelected">删除</el-button>
          </div>
          <el-form size="small" label-width="70px" label-position="left">
            <el-form-item label="X">
              <el-input-number v-model="selectedElement.x" :min="0" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="Y">
              <el-input-number v-model="selectedElement.y" :min="0" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="宽度">
              <el-input-number v-model="selectedElement.width" :min="10" controls-position="right" style="width:100%" />
            </el-form-item>
            <el-form-item label="高度">
              <el-input-number v-model="selectedElement.height" :min="10" controls-position="right" style="width:100%" />
            </el-form-item>

            <!-- Text / Field specific -->
            <template v-if="selectedElement.type === 'text' || selectedElement.type === 'field'">
              <el-form-item label="内容" v-if="selectedElement.type === 'text'">
                <el-input v-model="selectedElement.content" type="textarea" :rows="2" />
              </el-form-item>
              <el-form-item label="字段" v-if="selectedElement.type === 'field'">
                <el-input v-model="selectedElement.field" />
              </el-form-item>
              <el-form-item label="字号">
                <el-input-number v-model="selectedElement.fontSize" :min="8" :max="72" controls-position="right" style="width:100%" />
              </el-form-item>
              <el-form-item label="颜色">
                <el-color-picker v-model="selectedElement.color" size="small" />
              </el-form-item>
              <el-form-item label="对齐">
                <el-radio-group v-model="selectedElement.textAlign" size="small">
                  <el-radio-button value="left">左</el-radio-button>
                  <el-radio-button value="center">中</el-radio-button>
                  <el-radio-button value="right">右</el-radio-button>
                </el-radio-group>
              </el-form-item>
              <el-form-item label="加粗">
                <el-switch v-model="selectedElement.bold" />
              </el-form-item>
              <el-form-item label="斜体">
                <el-switch v-model="selectedElement.italic" />
              </el-form-item>
              <el-form-item label="背景色">
                <el-color-picker v-model="selectedElement.backgroundColor" size="small" show-alpha />
              </el-form-item>
              <el-form-item label="边框">
                <el-switch v-model="selectedElement.border" />
              </el-form-item>
            </template>

            <!-- Image specific -->
            <template v-if="selectedElement.type === 'image'">
              <el-form-item label="图片URL">
                <el-input v-model="selectedElement.src" placeholder="输入图片URL" />
              </el-form-item>
              <el-form-item label="适应">
                <el-select v-model="selectedElement.objectFit">
                  <el-option label="contain" value="contain" />
                  <el-option label="cover" value="cover" />
                  <el-option label="fill" value="fill" />
                </el-select>
              </el-form-item>
            </template>

            <!-- Line specific -->
            <template v-if="selectedElement.type === 'line'">
              <el-form-item label="颜色">
                <el-color-picker v-model="selectedElement.color" size="small" />
              </el-form-item>
              <el-form-item label="粗细">
                <el-input-number v-model="selectedElement.lineWidth" :min="1" :max="20" controls-position="right" style="width:100%" />
              </el-form-item>
              <el-form-item label="方向">
                <el-radio-group v-model="selectedElement.direction" size="small">
                  <el-radio-button value="horizontal">水平</el-radio-button>
                  <el-radio-button value="vertical">垂直</el-radio-button>
                </el-radio-group>
              </el-form-item>
            </template>

            <!-- Table specific -->
            <template v-if="selectedElement.type === 'table'">
              <el-form-item label="数据字段">
                <el-input v-model="selectedElement.dataField" placeholder="数组字段名" />
              </el-form-item>
              <div class="table-columns-editor">
                <div class="panel-section-title" style="font-size:12px;margin-bottom:6px;">
                  列配置
                  <el-button size="small" link @click="addTableColumn(selectedElement)">+ 添加列</el-button>
                </div>
                <div v-for="(col, idx) in selectedElement.columns" :key="col.key" class="table-col-row">
                  <el-input v-model="col.header" placeholder="列标题" size="small" style="flex:1" />
                  <el-input v-model="col.field" placeholder="字段名" size="small" style="flex:1;margin:0 4px" />
                  <el-input-number v-model="col.width" :min="30" size="small" style="width:70px" controls-position="right" />
                  <el-button size="small" link type="danger" @click="removeTableColumn(selectedElement, idx)">×</el-button>
                </div>
              </div>
            </template>
          </el-form>
        </div>

        <!-- Band Properties -->
        <div v-if="currentBandObj && !selectedElement" class="panel-section">
          <div class="panel-section-title">区域属性</div>
          <el-form size="small" label-width="70px" label-position="left">
            <el-form-item label="标题">
              <el-input v-model="currentBandObj.title" />
            </el-form-item>
            <el-form-item label="高度">
              <el-input-number v-model="currentBandObj.height" :min="20" :max="2000" controls-position="right" style="width:100%" />
            </el-form-item>
          </el-form>
        </div>

        <!-- JSON Template -->
        <div class="panel-section panel-section-grow">
          <div class="panel-section-title">
            模板 JSON
            <el-button size="small" link @click="copyTemplate">复制</el-button>
          </div>
          <div class="json-editor-container">
            <JsonEditor
              :model-value="jsonTemplate"
              :read-only="true"
              height="100%"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- Import Template Dialog -->
    <el-dialog v-model="importTemplateDialog" title="导入模板 JSON" width="600px">
      <div style="height:300px">
        <JsonEditor v-model="importTemplateText" height="100%" />
      </div>
      <template #footer>
        <el-button @click="importTemplateDialog = false">取消</el-button>
        <el-button type="primary" @click="doImportTemplate">导入</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import JsonEditor from './JsonEditor.vue'

// ─── Constants ──────────────────────────────────────────────────────────────

const componentTypes = [
  { type: 'text', label: '文本', icon: '📝' },
  { type: 'field', label: '字段', icon: '🔤' },
  { type: 'image', label: '图片', icon: '🖼' },
  { type: 'line', label: '线条', icon: '➖' },
  { type: 'table', label: '表格', icon: '📋' }
]

// ─── State ───────────────────────────────────────────────────────────────────

const mode = ref('design')
const showGrid = ref(true)
const zoom = ref(1)
const currentBand = ref(null)
const selectedElement = ref(null)
const draggingField = ref(null)
const draggingComp = ref(null)
const importTemplateDialog = ref(false)
const importTemplateText = ref('')
const sampleDataError = ref('')

const sampleDataEditorRef = ref(null)
const designCanvasRef = ref(null)
const previewRootRef = ref(null)

let idCounter = 1
function nextId() { return `el_${idCounter++}` }

// ─── Template ────────────────────────────────────────────────────────────────

const template = reactive({
  page: {
    width: 794,
    height: 1123,
    paddingLeft: 30,
    paddingRight: 30,
    paddingTop: 20,
    paddingBottom: 20
  },
  bands: [
    {
      key: 'pageHeader',
      title: '页眉',
      height: 80,
      elements: []
    },
    {
      key: 'body',
      title: '主体',
      height: 600,
      elements: []
    },
    {
      key: 'pageFooter',
      title: '页脚',
      height: 60,
      elements: []
    }
  ]
})

// ─── Computed ────────────────────────────────────────────────────────────────

const currentBandObj = computed(() =>
  template.bands.find(b => b.key === currentBand.value) || null
)

const jsonTemplate = computed(() =>
  JSON.stringify(template, null, 2)
)

// ─── Data Fields ─────────────────────────────────────────────────────────────

const previewDataText = ref(JSON.stringify({
  title: '季度销售报表',
  company: '示例科技有限公司',
  date: '2026-03-13',
  total: 128000,
  items: [
    { name: '产品A', qty: 100, price: 580 },
    { name: '产品B', qty: 80, price: 720 },
    { name: '产品C', qty: 60, price: 350 }
  ]
}, null, 2))

const dataFields = ref([])

function parseDataFields() {
  sampleDataError.value = ''
  try {
    const data = JSON.parse(previewDataText.value)
    const fields = []
    function extract(obj, prefix = '') {
      for (const key of Object.keys(obj)) {
        const val = obj[key]
        const fullKey = prefix ? `${prefix}.${key}` : key
        if (Array.isArray(val)) {
          fields.push({ key: fullKey, label: fullKey + ' (数组)', isArray: true })
          if (val.length > 0 && typeof val[0] === 'object') {
            for (const subKey of Object.keys(val[0])) {
              fields.push({ key: `${fullKey}[].${subKey}`, label: `${fullKey}[].${subKey}`, isArray: false })
            }
          }
        } else if (val !== null && typeof val === 'object') {
          extract(val, fullKey)
        } else {
          fields.push({ key: fullKey, label: fullKey, isArray: false })
        }
      }
    }
    extract(data)
    dataFields.value = fields
  } catch (e) {
    sampleDataError.value = '⚠ JSON 格式错误: ' + e.message
    dataFields.value = []
  }
}

function formatSampleData() {
  try {
    previewDataText.value = JSON.stringify(JSON.parse(previewDataText.value), null, 2)
    ElMessage.success('JSON 已格式化')
  } catch (e) {
    ElMessage.error('JSON 格式错误，无法格式化')
  }
}

// ─── Selection ───────────────────────────────────────────────────────────────

function clearSelection() {
  selectedElement.value = null
  currentBand.value = null
}

function selectElement(el) {
  selectedElement.value = el
}

function selectBand(band) {
  currentBand.value = band.key
  selectedElement.value = null
}

// ─── Keyboard Shortcuts ───────────────────────────────────────────────────────

function onKeydown(e) {
  if (e.key === 'Escape') {
    clearSelection()
  }
  if ((e.key === 'Delete' || e.key === 'Backspace') && selectedElement.value) {
    // only delete if not focused on an input
    if (!['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName)) {
      deleteSelected()
    }
  }
}

// ─── Drag: Components ────────────────────────────────────────────────────────

function onCompDragStart(e, comp) {
  draggingComp.value = comp
  draggingField.value = null
  e.dataTransfer.effectAllowed = 'copy'
}

function onFieldDragStart(e, field) {
  draggingField.value = field
  draggingComp.value = null
  e.dataTransfer.effectAllowed = 'copy'
}

function onCanvasDrop(e) {
  // fallback drop on canvas (no specific band)
  const band = template.bands.find(b => b.key === 'body')
  if (band) handleDrop(e, band, designCanvasRef.value)
}

function onBandDrop(e, band) {
  handleDrop(e, band, e.currentTarget)
}

function handleDrop(e, band, container) {
  e.stopPropagation()
  const rect = container.getBoundingClientRect()
  const x = Math.round((e.clientX - rect.left) / zoom.value)
  const y = Math.round((e.clientY - rect.top) / zoom.value)

  if (draggingComp.value) {
    const el = createDefaultElement(draggingComp.value.type, x, y)
    band.elements.push(el)
    selectedElement.value = el
    currentBand.value = band.key
    draggingComp.value = null
  } else if (draggingField.value) {
    const el = createDefaultElement('field', x, y)
    el.field = draggingField.value.key
    el.content = draggingField.value.label
    band.elements.push(el)
    selectedElement.value = el
    currentBand.value = band.key
    draggingField.value = null
  }
}

function createDefaultElement(type, x = 10, y = 10) {
  const base = {
    id: nextId(),
    type,
    x,
    y,
    width: 150,
    height: 30
  }
  if (type === 'text') {
    return { ...base, content: '文本', fontSize: 14, color: '#303133', bold: false, italic: false, textAlign: 'left', border: false, backgroundColor: '' }
  }
  if (type === 'field') {
    return { ...base, field: '', content: '', fontSize: 14, color: '#303133', bold: false, italic: false, textAlign: 'left', border: false, backgroundColor: '' }
  }
  if (type === 'image') {
    return { ...base, src: '', objectFit: 'contain', width: 100, height: 80 }
  }
  if (type === 'line') {
    return { ...base, color: '#303133', lineWidth: 1, direction: 'horizontal', height: 4 }
  }
  if (type === 'table') {
    return {
      ...base,
      dataField: 'items',
      width: 500,
      height: 120,
      columns: [
        { key: 'c1', header: '名称', field: 'name', width: 150 },
        { key: 'c2', header: '数量', field: 'qty', width: 80 },
        { key: 'c3', header: '单价', field: 'price', width: 80 }
      ]
    }
  }
  return base
}

// ─── Drag: Elements (move) ────────────────────────────────────────────────────

let dragState = null

function startDragElement(e, el, band) {
  selectedElement.value = el
  currentBand.value = band.key

  const startX = e.clientX
  const startY = e.clientY
  const origX = el.x
  const origY = el.y

  dragState = { el, startX, startY, origX, origY }

  window.addEventListener('mousemove', onDragElementMove)
  window.addEventListener('mouseup', onDragElementUp)
}

function onDragElementMove(e) {
  if (!dragState) return
  const { el, startX, startY, origX, origY } = dragState
  const dx = (e.clientX - startX) / zoom.value
  const dy = (e.clientY - startY) / zoom.value
  el.x = Math.max(0, Math.round(origX + dx))
  el.y = Math.max(0, Math.round(origY + dy))
}

function onDragElementUp() {
  dragState = null
  window.removeEventListener('mousemove', onDragElementMove)
  window.removeEventListener('mouseup', onDragElementUp)
}

// ─── Resize: Elements ─────────────────────────────────────────────────────────

let resizeState = null

function startResize(e, el) {
  const startX = e.clientX
  const startY = e.clientY
  const origW = el.width
  const origH = el.height

  resizeState = { el, startX, startY, origW, origH }
  window.addEventListener('mousemove', onResizeMove)
  window.addEventListener('mouseup', onResizeUp)
}

function onResizeMove(e) {
  if (!resizeState) return
  const { el, startX, startY, origW, origH } = resizeState
  const dx = (e.clientX - startX) / zoom.value
  const dy = (e.clientY - startY) / zoom.value
  el.width = Math.max(20, Math.round(origW + dx))
  el.height = Math.max(10, Math.round(origH + dy))
}

function onResizeUp() {
  resizeState = null
  window.removeEventListener('mousemove', onResizeMove)
  window.removeEventListener('mouseup', onResizeUp)
}

// ─── Band Resize ──────────────────────────────────────────────────────────────

let bandResizeState = null

function startBandResize(e, band) {
  bandResizeState = { band, startY: e.clientY, origH: band.height }
  window.addEventListener('mousemove', onBandResizeMove)
  window.addEventListener('mouseup', onBandResizeUp)
}

function onBandResizeMove(e) {
  if (!bandResizeState) return
  const { band, startY, origH } = bandResizeState
  const dy = (e.clientY - startY) / zoom.value
  band.height = Math.max(20, Math.round(origH + dy))
}

function onBandResizeUp() {
  bandResizeState = null
  window.removeEventListener('mousemove', onBandResizeMove)
  window.removeEventListener('mouseup', onBandResizeUp)
}

// ─── Zoom ─────────────────────────────────────────────────────────────────────

function zoomIn() { zoom.value = Math.min(3, parseFloat((zoom.value + 0.1).toFixed(1))) }
function zoomOut() { zoom.value = Math.max(0.2, parseFloat((zoom.value - 0.1).toFixed(1))) }

// ─── Element Styles ───────────────────────────────────────────────────────────

function getElementStyle(el) {
  return {
    position: 'absolute',
    left: el.x + 'px',
    top: el.y + 'px',
    width: el.width + 'px',
    height: el.height + 'px',
    overflow: 'hidden',
    cursor: 'move',
    userSelect: 'none',
    boxSizing: 'border-box',
    ...(el.border ? { border: '1px solid #dcdfe6' } : {}),
    ...(el.backgroundColor ? { background: el.backgroundColor } : {})
  }
}

function getTextStyle(el) {
  return {
    fontSize: (el.fontSize || 14) + 'px',
    color: el.color || '#303133',
    fontWeight: el.bold ? 'bold' : 'normal',
    fontStyle: el.italic ? 'italic' : 'normal',
    textAlign: el.textAlign || 'left',
    display: 'block',
    width: '100%',
    height: '100%',
    lineHeight: el.height + 'px',
    whiteSpace: 'nowrap',
    overflow: 'hidden',
    textOverflow: 'ellipsis'
  }
}

function getLineStyle(el) {
  if (el.direction === 'vertical') {
    return {
      position: 'absolute',
      left: '50%',
      top: 0,
      bottom: 0,
      width: (el.lineWidth || 1) + 'px',
      background: el.color || '#303133'
    }
  }
  return {
    position: 'absolute',
    left: 0,
    right: 0,
    top: '50%',
    height: (el.lineWidth || 1) + 'px',
    background: el.color || '#303133'
  }
}

// ─── Preview Data ─────────────────────────────────────────────────────────────

function getParsedData() {
  try {
    return JSON.parse(previewDataText.value)
  } catch (e) {
    return {}
  }
}

function getFieldPreview(el) {
  if (!el.field) return el.content || `[${el.id}]`
  const data = getParsedData()
  return getNestedValue(data, el.field) ?? `{${el.field}}`
}

function getFieldValue(el) {
  if (!el.field) return el.content || ''
  const data = getParsedData()
  return getNestedValue(data, el.field) ?? ''
}

function getTableData(el) {
  const data = getParsedData()
  const arr = getNestedValue(data, el.dataField)
  return Array.isArray(arr) ? arr : []
}

function getNestedValue(obj, path) {
  if (!path) return undefined
  const parts = path.split('.')
  let cur = obj
  for (const part of parts) {
    if (cur == null) return undefined
    cur = cur[part]
  }
  return cur
}

// ─── Delete Element ───────────────────────────────────────────────────────────

function deleteSelected() {
  if (!selectedElement.value) return
  for (const band of template.bands) {
    const idx = band.elements.findIndex(e => e.id === selectedElement.value.id)
    if (idx !== -1) {
      band.elements.splice(idx, 1)
      selectedElement.value = null
      return
    }
  }
}

// ─── Table Columns ────────────────────────────────────────────────────────────

let colCounter = 1
function addTableColumn(el) {
  el.columns.push({ key: `c_${colCounter++}`, header: '列', field: '', width: 100 })
}
function removeTableColumn(el, idx) {
  el.columns.splice(idx, 1)
}

// ─── Template Import / Export ─────────────────────────────────────────────────

function exportTemplate() {
  const blob = new Blob([JSON.stringify(template, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'report-template.json'
  a.click()
  URL.revokeObjectURL(url)
}

function doImportTemplate() {
  try {
    const parsed = JSON.parse(importTemplateText.value)
    if (!parsed.page || !parsed.bands) {
      ElMessage.error('模板格式不正确，需要 page 和 bands 字段')
      return
    }
    Object.assign(template, parsed)
    importTemplateDialog.value = false
    importTemplateText.value = ''
    ElMessage.success('模板导入成功')
  } catch (e) {
    ElMessage.error('JSON 解析错误: ' + e.message)
  }
}

function copyTemplate() {
  navigator.clipboard.writeText(jsonTemplate.value)
    .then(() => ElMessage.success('模板 JSON 已复制到剪贴板'))
    .catch(() => ElMessage.error('复制失败'))
}

function clearTemplate() {
  ElMessageBox.confirm('确定要清空所有设计内容吗？', '警告', {
    type: 'warning',
    confirmButtonText: '清空',
    cancelButtonText: '取消'
  }).then(() => {
    for (const band of template.bands) {
      band.elements = []
    }
    selectedElement.value = null
    ElMessage.success('已清空')
  }).catch(() => {})
}

// ─── Export PDF ───────────────────────────────────────────────────────────────

async function exportPDF() {
  const prevMode = mode.value
  mode.value = 'preview'
  await nextTick()

  try {
    const [{ default: html2canvas }, { default: jsPDF }] = await Promise.all([
      import('html2canvas'),
      import('jspdf')
    ])

    const el = previewRootRef.value
    const canvas = await html2canvas(el, { scale: 2, useCORS: true, backgroundColor: '#fff' })
    const imgData = canvas.toDataURL('image/png')

    const pdf = new jsPDF({
      orientation: template.page.width > template.page.height ? 'landscape' : 'portrait',
      unit: 'px',
      format: [template.page.width, template.page.height]
    })

    pdf.addImage(imgData, 'PNG', 0, 0, template.page.width, template.page.height)
    pdf.save('report.pdf')
    ElMessage.success('PDF 导出成功')
  } catch (e) {
    ElMessage.error('PDF 导出失败: ' + e.message)
  } finally {
    mode.value = prevMode
  }
}

// ─── Lifecycle ────────────────────────────────────────────────────────────────

onMounted(() => {
  parseDataFields()
})

onBeforeUnmount(() => {
  window.removeEventListener('mousemove', onDragElementMove)
  window.removeEventListener('mouseup', onDragElementUp)
  window.removeEventListener('mousemove', onResizeMove)
  window.removeEventListener('mouseup', onResizeUp)
  window.removeEventListener('mousemove', onBandResizeMove)
  window.removeEventListener('mouseup', onBandResizeUp)
})
</script>

<style scoped>
/* ── Shell ───────────────────────────────────────────────────────────────── */
.designer-shell {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  overflow: hidden;
  background: #f0f2f5;
  outline: none;
}

/* ── Top Toolbar ─────────────────────────────────────────────────────────── */
.top-toolbar {
  flex: 0 0 auto;
  height: 46px;
  display: flex;
  align-items: center;
  padding: 0 12px;
  background: #fff;
  border-bottom: 1px solid #ebeef5;
  gap: 8px;
  box-shadow: 0 1px 4px rgba(0,0,0,.08);
}

.toolbar-left { flex: 0 0 auto; }
.toolbar-center { flex: 1; display: flex; align-items: center; gap: 6px; }
.toolbar-right { flex: 0 0 auto; }

.brand {
  font-size: 15px;
  font-weight: 600;
  color: #409eff;
  letter-spacing: 0.5px;
}

.zoom-label {
  font-size: 12px;
  color: #909399;
  min-width: 40px;
  text-align: center;
}

/* ── Body Layout ─────────────────────────────────────────────────────────── */
.body-layout {
  flex: 1;
  display: flex;
  overflow: hidden;
  min-height: 0;
}

/* ── Left Panel ──────────────────────────────────────────────────────────── */
.left-panel {
  flex: 0 0 220px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #fff;
  border-right: 1px solid #ebeef5;
}

/* ── Right Panel ─────────────────────────────────────────────────────────── */
.right-panel {
  flex: 0 0 240px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #fff;
  border-left: 1px solid #ebeef5;
}

/* ── Panel Sections ──────────────────────────────────────────────────────── */
.panel-section {
  flex: 0 0 auto;
  padding: 10px 12px;
  border-bottom: 1px solid #f0f2f5;
  overflow: hidden;
}

.panel-section-grow {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
  overflow: hidden;
}

.panel-section-title {
  font-size: 12px;
  font-weight: 600;
  color: #606266;
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

/* ── Component List ──────────────────────────────────────────────────────── */
.component-list {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}

.comp-item {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 6px 8px;
  border: 1px dashed #dcdfe6;
  border-radius: 4px;
  cursor: grab;
  font-size: 12px;
  color: #606266;
  background: #fafafa;
  transition: all 0.15s;
}

.comp-item:hover {
  border-color: #409eff;
  color: #409eff;
  background: #ecf5ff;
}

.comp-icon { font-size: 14px; }
.comp-label { font-size: 11px; }

/* ── Field List ──────────────────────────────────────────────────────────── */
.field-list {
  max-height: 120px;
  overflow-y: auto;
  overflow-x: hidden;
}

.field-item {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 4px 6px;
  border-radius: 3px;
  cursor: grab;
  font-size: 11px;
  color: #606266;
  transition: background 0.15s;
}

.field-item:hover {
  background: #ecf5ff;
  color: #409eff;
}

.field-icon { font-size: 11px; }
.field-label { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

.empty-tip {
  font-size: 11px;
  color: #c0c4cc;
  text-align: center;
  padding: 8px 0;
}

/* ── JSON Editor Container ───────────────────────────────────────────────── */
.json-editor-container {
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.json-error {
  font-size: 11px;
  color: #f56c6c;
  margin-top: 4px;
}

/* ── Canvas Area ─────────────────────────────────────────────────────────── */
.canvas-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-width: 0;
}

.canvas-scroll,
.preview-scroll {
  flex: 1;
  overflow: auto;
  min-height: 0;
  min-width: 0;
  padding: 20px;
  background: #eef2f6;
}

.canvas-wrapper {
  display: inline-block;
  transform-origin: top center;
}

/* ── Page Canvas ─────────────────────────────────────────────────────────── */
.page-canvas {
  background: #fff;
  box-shadow: 0 2px 12px rgba(0,0,0,.15);
  position: relative;
  margin: 0 auto;
}

.page-canvas.show-grid {
  background-image:
    linear-gradient(to right, #e8edf3 1px, transparent 1px),
    linear-gradient(to bottom, #e8edf3 1px, transparent 1px);
  background-size: 10px 10px;
}

.page-canvas.dragging-field {
  cursor: crosshair;
}

/* ── Band ────────────────────────────────────────────────────────────────── */
.band {
  position: relative;
  border-bottom: 1px dashed #c0c4cc;
}

.band.active > .band-label {
  background: #ecf5ff;
  color: #409eff;
}

.band-label {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 2px 6px;
  font-size: 11px;
  color: #909399;
  background: #fafafa;
  border-bottom: 1px solid #e8edf3;
  cursor: pointer;
  user-select: none;
}

.band-height-info {
  font-size: 10px;
  color: #c0c4cc;
}

.band-body {
  overflow: visible;
}

.band-resize-handle {
  position: absolute;
  bottom: -4px;
  left: 0;
  right: 0;
  height: 8px;
  cursor: ns-resize;
  z-index: 10;
}

.band-resize-handle:hover {
  background: rgba(64, 158, 255, 0.2);
}

/* ── Design Element ──────────────────────────────────────────────────────── */
.design-element {
  border: 1px dashed transparent;
  transition: border-color 0.1s;
}

.design-element:hover {
  border-color: #a0cfff;
}

.design-element.selected {
  border-color: #409eff !important;
  border-style: solid !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

/* ── Resize Handle ───────────────────────────────────────────────────────── */
.resize-handle {
  position: absolute;
  width: 8px;
  height: 8px;
  background: #409eff;
  border: 1px solid #fff;
  border-radius: 2px;
}

.resize-se {
  bottom: -4px;
  right: -4px;
  cursor: se-resize;
}

/* ── Image Placeholder ───────────────────────────────────────────────────── */
.image-placeholder {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f7fa;
  color: #c0c4cc;
  font-size: 12px;
  border: 1px dashed #dcdfe6;
}

/* ── Line ────────────────────────────────────────────────────────────────── */
.line-el {
  position: absolute;
}

/* ── Table Placeholder ───────────────────────────────────────────────────── */
.table-placeholder {
  width: 100%;
  height: 100%;
  overflow: hidden;
  font-size: 11px;
}

.table-header-row,
.table-data-row {
  display: flex;
}

.table-th {
  padding: 2px 4px;
  background: #f5f7fa;
  border: 1px solid #e4e7ed;
  font-weight: 600;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.table-td {
  padding: 2px 4px;
  border: 1px solid #ebeef5;
  color: #909399;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* ── Table Columns Editor ────────────────────────────────────────────────── */
.table-columns-editor {
  margin-top: 6px;
}

.table-col-row {
  display: flex;
  align-items: center;
  margin-bottom: 4px;
}

/* ── Preview Page ────────────────────────────────────────────────────────── */
.preview-page {
  background: #fff;
  box-shadow: 0 2px 12px rgba(0,0,0,.15);
  margin: 0 auto;
}

/* ── Scrollbar Styling ───────────────────────────────────────────────────── */
.left-panel::-webkit-scrollbar,
.right-panel::-webkit-scrollbar,
.field-list::-webkit-scrollbar,
.canvas-scroll::-webkit-scrollbar,
.preview-scroll::-webkit-scrollbar {
  width: 4px;
}

.left-panel::-webkit-scrollbar-thumb,
.right-panel::-webkit-scrollbar-thumb,
.field-list::-webkit-scrollbar-thumb,
.canvas-scroll::-webkit-scrollbar-thumb,
.preview-scroll::-webkit-scrollbar-thumb {
  background: #c0c4cc;
  border-radius: 2px;
}
</style>
