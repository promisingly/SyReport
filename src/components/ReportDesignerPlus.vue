<template>
  <div class="designer-shell" @keydown.esc="clearSelection" tabindex="0">
    <!-- ─── Top Toolbar ─── -->
    <div class="top-toolbar">
      <div class="toolbar-left">
        <el-button size="small" :icon="Document" @click="newTemplate">新建</el-button>
        <el-button size="small" :icon="FolderOpened" @click="importTemplate">导入</el-button>
        <el-button size="small" :icon="Download" @click="exportTemplate">导出</el-button>
        <el-divider direction="vertical" />
        <el-button size="small" :icon="ZoomIn" @click="zoom = Math.min(zoom + 0.1, 2)">放大</el-button>
        <el-button size="small" @click="zoom = 1">{{ Math.round(zoom * 100) }}%</el-button>
        <el-button size="small" :icon="ZoomOut" @click="zoom = Math.max(zoom - 0.1, 0.3)">缩小</el-button>
        <el-divider direction="vertical" />
        <el-button size="small" :icon="Grid" @click="showGrid = !showGrid" :type="showGrid ? 'primary' : ''">网格</el-button>
      </div>
      <div class="toolbar-center">
        <span class="report-title">{{ template.name }}</span>
      </div>
      <div class="toolbar-right">
        <el-button-group>
          <el-button size="small" :type="mode === 'design' ? 'primary' : ''" @click="mode = 'design'">设计</el-button>
          <el-button size="small" :type="mode === 'preview' ? 'primary' : ''" @click="mode = 'preview'">预览</el-button>
        </el-button-group>
        <el-button size="small" type="success" :icon="Printer" @click="printReport" style="margin-left:8px">打印</el-button>
      </div>
    </div>

    <!-- ─── Body ─── -->
    <div class="body-layout">
      <!-- Left Panel -->
      <div class="left-panel">
        <el-tabs v-model="leftTab" class="left-tabs">
          <!-- Elements Palette -->
          <el-tab-pane label="元素" name="elements">
            <div class="palette-section">
              <div class="palette-title">基础元素</div>
              <div class="palette-grid">
                <div
                  v-for="item in elementTypes"
                  :key="item.type"
                  class="palette-item"
                  draggable="true"
                  @dragstart="onPaletteDragStart($event, item.type)"
                >
                  <el-icon class="palette-icon"><component :is="item.icon" /></el-icon>
                  <span>{{ item.label }}</span>
                </div>
              </div>
            </div>
            <div class="palette-section">
              <div class="palette-title">报表段</div>
              <div class="band-list">
                <div
                  v-for="band in template.bands"
                  :key="band.key"
                  class="band-list-item"
                  :class="{ active: currentBandKey === band.key }"
                  @click="currentBandKey = band.key"
                >
                  <el-icon><Menu /></el-icon>
                  <span>{{ band.label }}</span>
                  <span class="band-count">{{ band.elements.length }}</span>
                </div>
              </div>
            </div>
          </el-tab-pane>

          <!-- Data Fields -->
          <el-tab-pane label="数据" name="data">
            <div class="data-section">
              <div class="palette-title" style="margin-bottom:8px">示例数据 (JSON)</div>
              <JsonEditor v-model="previewDataText" height="200px" />
              <div style="margin-top:8px;display:flex;gap:6px;">
                <el-button size="small" type="primary" @click="parseDataFields">解析字段</el-button>
                <el-button size="small" @click="formatPreviewJson">格式化</el-button>
              </div>
              <div class="palette-title" style="margin-top:16px;margin-bottom:8px">可用字段</div>
              <div class="field-list">
                <div
                  v-for="field in dataFields"
                  :key="field"
                  class="field-item"
                  draggable="true"
                  @dragstart="onFieldDragStart($event, field)"
                >
                  <el-icon><Coin /></el-icon>
                  <span>{{ field }}</span>
                </div>
                <div v-if="!dataFields.length" class="field-empty">暂无字段，请先解析数据</div>
              </div>
            </div>
          </el-tab-pane>
        </el-tabs>
      </div>

      <!-- Center: Canvas -->
      <div class="canvas-main">
        <!-- Canvas Toolbar -->
        <div class="canvas-toolbar">
          <template v-if="selectedElement">
            <el-input-number v-model="selectedElement.x" :min="0" size="small" style="width:100px" @change="updateTemplate">
              <template #prefix>X</template>
            </el-input-number>
            <el-input-number v-model="selectedElement.y" :min="0" size="small" style="width:100px" @change="updateTemplate">
              <template #prefix>Y</template>
            </el-input-number>
            <el-input-number v-model="selectedElement.width" :min="10" size="small" style="width:110px" @change="updateTemplate">
              <template #prefix>W</template>
            </el-input-number>
            <el-input-number v-model="selectedElement.height" :min="10" size="small" style="width:110px" @change="updateTemplate">
              <template #prefix>H</template>
            </el-input-number>
            <el-divider direction="vertical" />
            <el-button size="small" type="danger" :icon="Delete" @click="deleteSelectedElement">删除</el-button>
            <el-button size="small" :icon="CopyDocument" @click="duplicateSelectedElement">复制</el-button>
          </template>
          <span v-else class="canvas-hint">点击选中元素，拖拽元素到画布</span>
        </div>

        <!-- Canvas Scroll -->
        <div class="canvas-scroll" @dragover.prevent @drop="onCanvasDrop($event, null)">
          <div
            class="page-canvas"
            :class="{ 'show-grid': showGrid }"
            :style="{
              width: template.page.width + 'px',
              transform: `scale(${zoom})`,
              transformOrigin: 'top left'
            }"
            @mousedown.self="clearSelection"
          >
            <!-- Bands -->
            <div
              v-for="band in template.bands"
              :key="band.key"
              class="band"
              :class="{ 'band-active': currentBandKey === band.key }"
              :style="{ height: band.height + 'px' }"
              @dragover.prevent
              @drop="onCanvasDrop($event, band)"
              @click.self="selectBand(band)"
            >
              <div class="band-label" @click="selectBand(band)">{{ band.label }}</div>

              <!-- Elements -->
              <div
                v-for="el in band.elements"
                :key="el.id"
                class="report-element"
                :class="{
                  selected: selectedElement && selectedElement.id === el.id,
                  'element-text': el.type === 'text' || el.type === 'staticText',
                  'element-image': el.type === 'image',
                  'element-line': el.type === 'line',
                  'element-rect': el.type === 'rectangle'
                }"
                :style="getElementStyle(el)"
                @mousedown.stop="startElementDrag($event, el, band)"
              >
                <component
                  :is="'div'"
                  class="element-content"
                  :style="getElementContentStyle(el)"
                >
                  <template v-if="el.type === 'staticText'">{{ el.props.text || '静态文本' }}</template>
                  <template v-else-if="el.type === 'text'">{{ fieldLabel(el) }}</template>
                  <template v-else-if="el.type === 'image'">
                    <el-icon style="font-size:24px;color:#aaa"><Picture /></el-icon>
                  </template>
                  <template v-else-if="el.type === 'line'">
                    <div class="line-element" :style="{ borderTopColor: el.props.color || '#333', borderTopWidth: (el.props.lineWidth || 1) + 'px' }"></div>
                  </template>
                  <template v-else-if="el.type === 'rectangle'">
                    <!-- rectangle is styled via CSS -->
                  </template>
                  <template v-else-if="el.type === 'pageNumber'">页码: {{ el.props.format || '第{页}页' }}</template>
                  <template v-else-if="el.type === 'systemDate'">当前日期</template>
                  <template v-else-if="el.type === 'barcode'">
                    <el-icon style="font-size:20px;color:#555"><Grid /></el-icon>
                    <span style="font-size:10px">{{ el.props.field || '条码' }}</span>
                  </template>
                </component>

                <!-- Resize Handles (only when selected) -->
                <template v-if="selectedElement && selectedElement.id === el.id">
                  <div class="resize-handle handle-se" @mousedown.stop="startResize($event, el, 'se')"></div>
                  <div class="resize-handle handle-e" @mousedown.stop="startResize($event, el, 'e')"></div>
                  <div class="resize-handle handle-s" @mousedown.stop="startResize($event, el, 's')"></div>
                  <div class="resize-handle handle-ne" @mousedown.stop="startResize($event, el, 'ne')"></div>
                  <div class="resize-handle handle-nw" @mousedown.stop="startResize($event, el, 'nw')"></div>
                  <div class="resize-handle handle-sw" @mousedown.stop="startResize($event, el, 'sw')"></div>
                  <div class="resize-handle handle-n" @mousedown.stop="startResize($event, el, 'n')"></div>
                  <div class="resize-handle handle-w" @mousedown.stop="startResize($event, el, 'w')"></div>
                </template>
              </div>

              <!-- Band Resize Handle -->
              <div class="band-resize-handle" @mousedown.stop="startBandResize($event, band)"></div>
            </div>
          </div>
        </div>
      </div>

      <!-- Right Panel -->
      <div class="right-panel">
        <el-tabs v-model="rightTab" class="right-tabs">
          <!-- Properties -->
          <el-tab-pane label="属性" name="props">
            <div class="props-scroll">
              <!-- Element Properties -->
              <template v-if="selectedElement">
                <div class="props-section">
                  <div class="props-title">位置与大小</div>
                  <el-form label-width="50px" size="small">
                    <el-form-item label="X">
                      <el-input-number v-model="selectedElement.x" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="Y">
                      <el-input-number v-model="selectedElement.y" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="宽">
                      <el-input-number v-model="selectedElement.width" :min="10" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="高">
                      <el-input-number v-model="selectedElement.height" :min="10" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Text/StaticText Props -->
                <div v-if="selectedElement.type === 'staticText' || selectedElement.type === 'text' || selectedElement.type === 'pageNumber' || selectedElement.type === 'systemDate'" class="props-section">
                  <div class="props-title">文本属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item v-if="selectedElement.type === 'staticText'" label="内容">
                      <el-input v-model="selectedElement.props.text" @input="updateTemplate" />
                    </el-form-item>
                    <el-form-item v-if="selectedElement.type === 'text'" label="字段">
                      <el-select v-model="selectedElement.props.field" placeholder="选择字段" @change="updateTemplate" style="width:100%">
                        <el-option v-for="f in dataFields" :key="f" :label="f" :value="f" />
                      </el-select>
                    </el-form-item>
                    <el-form-item v-if="selectedElement.type === 'pageNumber'" label="格式">
                      <el-input v-model="selectedElement.props.format" @input="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="字号">
                      <el-input-number v-model="selectedElement.props.fontSize" :min="6" :max="72" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="字体">
                      <el-select v-model="selectedElement.props.fontFamily" @change="updateTemplate" style="width:100%">
                        <el-option label="默认" value="inherit" />
                        <el-option label="微软雅黑" value="Microsoft YaHei" />
                        <el-option label="宋体" value="SimSun" />
                        <el-option label="黑体" value="SimHei" />
                        <el-option label="Arial" value="Arial" />
                        <el-option label="Times New Roman" value="Times New Roman" />
                      </el-select>
                    </el-form-item>
                    <el-form-item label="颜色">
                      <el-color-picker v-model="selectedElement.props.color" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="加粗">
                      <el-switch v-model="selectedElement.props.bold" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="斜体">
                      <el-switch v-model="selectedElement.props.italic" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="下划线">
                      <el-switch v-model="selectedElement.props.underline" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="对齐">
                      <el-radio-group v-model="selectedElement.props.align" size="small" @change="updateTemplate">
                        <el-radio-button value="left">左</el-radio-button>
                        <el-radio-button value="center">中</el-radio-button>
                        <el-radio-button value="right">右</el-radio-button>
                      </el-radio-group>
                    </el-form-item>
                    <el-form-item label="纵向">
                      <el-radio-group v-model="selectedElement.props.verticalAlign" size="small" @change="updateTemplate">
                        <el-radio-button value="top">上</el-radio-button>
                        <el-radio-button value="middle">中</el-radio-button>
                        <el-radio-button value="bottom">下</el-radio-button>
                      </el-radio-group>
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Line Props -->
                <div v-if="selectedElement.type === 'line'" class="props-section">
                  <div class="props-title">线条属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="颜色">
                      <el-color-picker v-model="selectedElement.props.color" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="线宽">
                      <el-input-number v-model="selectedElement.props.lineWidth" :min="1" :max="10" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="线型">
                      <el-select v-model="selectedElement.props.lineStyle" @change="updateTemplate" style="width:100%">
                        <el-option label="实线" value="solid" />
                        <el-option label="虚线" value="dashed" />
                        <el-option label="点线" value="dotted" />
                      </el-select>
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Rectangle Props -->
                <div v-if="selectedElement.type === 'rectangle'" class="props-section">
                  <div class="props-title">矩形属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="填充色">
                      <el-color-picker v-model="selectedElement.props.fillColor" show-alpha @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="边框色">
                      <el-color-picker v-model="selectedElement.props.borderColor" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="边框宽">
                      <el-input-number v-model="selectedElement.props.borderWidth" :min="0" :max="10" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="圆角">
                      <el-input-number v-model="selectedElement.props.borderRadius" :min="0" :max="50" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Image Props -->
                <div v-if="selectedElement.type === 'image'" class="props-section">
                  <div class="props-title">图片属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="字段">
                      <el-select v-model="selectedElement.props.field" placeholder="图片字段" @change="updateTemplate" style="width:100%">
                        <el-option v-for="f in dataFields" :key="f" :label="f" :value="f" />
                      </el-select>
                    </el-form-item>
                    <el-form-item label="适应">
                      <el-select v-model="selectedElement.props.objectFit" @change="updateTemplate" style="width:100%">
                        <el-option label="填充" value="fill" />
                        <el-option label="包含" value="contain" />
                        <el-option label="覆盖" value="cover" />
                      </el-select>
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Barcode Props -->
                <div v-if="selectedElement.type === 'barcode'" class="props-section">
                  <div class="props-title">条码属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="字段">
                      <el-select v-model="selectedElement.props.field" placeholder="数据字段" @change="updateTemplate" style="width:100%">
                        <el-option v-for="f in dataFields" :key="f" :label="f" :value="f" />
                      </el-select>
                    </el-form-item>
                    <el-form-item label="类型">
                      <el-select v-model="selectedElement.props.barcodeType" @change="updateTemplate" style="width:100%">
                        <el-option label="QR Code" value="qrcode" />
                        <el-option label="Code 128" value="code128" />
                        <el-option label="Code 39" value="code39" />
                        <el-option label="EAN-13" value="ean13" />
                      </el-select>
                    </el-form-item>
                  </el-form>
                </div>

                <!-- Border & Background (universal) -->
                <div class="props-section">
                  <div class="props-title">通用样式</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="背景色">
                      <el-color-picker v-model="selectedElement.props.bgColor" show-alpha @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="透明度">
                      <el-slider v-model="selectedElement.props.opacity" :min="0" :max="100" @change="updateTemplate" />
                    </el-form-item>
                  </el-form>
                </div>
              </template>

              <!-- Band Properties (when band is selected but no element) -->
              <template v-else-if="currentBandKey">
                <div class="props-section">
                  <div class="props-title">段属性: {{ currentBand && currentBand.label }}</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="高度">
                      <el-input-number v-model="currentBand.height" :min="20" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="可见">
                      <el-switch v-model="currentBand.visible" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="背景">
                      <el-color-picker v-model="currentBand.bgColor" show-alpha @change="updateTemplate" />
                    </el-form-item>
                  </el-form>
                </div>
              </template>

              <!-- Page Properties (nothing selected) -->
              <template v-else>
                <div class="props-section">
                  <div class="props-title">页面属性</div>
                  <el-form label-width="60px" size="small">
                    <el-form-item label="名称">
                      <el-input v-model="template.name" @input="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="宽度">
                      <el-input-number v-model="template.page.width" :min="100" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="高度">
                      <el-input-number v-model="template.page.height" :min="100" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="纸张">
                      <el-select style="width:100%" @change="applyPaperSize">
                        <el-option label="A4 竖向" value="a4p" />
                        <el-option label="A4 横向" value="a4l" />
                        <el-option label="A3 竖向" value="a3p" />
                        <el-option label="A3 横向" value="a3l" />
                        <el-option label="自定义" value="custom" />
                      </el-select>
                    </el-form-item>
                    <el-form-item label="上边距">
                      <el-input-number v-model="template.page.margin.top" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="下边距">
                      <el-input-number v-model="template.page.margin.bottom" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="左边距">
                      <el-input-number v-model="template.page.margin.left" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                    <el-form-item label="右边距">
                      <el-input-number v-model="template.page.margin.right" :min="0" style="width:100%" @change="updateTemplate" />
                    </el-form-item>
                  </el-form>
                </div>
              </template>
            </div>
          </el-tab-pane>

          <!-- Template JSON -->
          <el-tab-pane label="模板JSON" name="json">
            <div class="json-panel">
              <div style="margin-bottom:8px;display:flex;gap:6px;">
                <el-button size="small" type="primary" @click="applyJsonTemplate">应用 JSON</el-button>
                <el-button size="small" @click="copyJsonTemplate">复制</el-button>
              </div>
              <JsonEditor v-model="jsonTemplateText" height="calc(100vh - 180px)" />
            </div>
          </el-tab-pane>
        </el-tabs>
      </div>
    </div>

    <!-- Preview Overlay -->
    <div v-if="mode === 'preview'" class="preview-overlay">
      <div class="preview-toolbar">
        <el-button @click="mode = 'design'" size="small" :icon="Back">返回设计</el-button>
        <el-button @click="printReport" size="small" type="primary" :icon="Printer">打印</el-button>
      </div>
      <div class="preview-scroll">
        <div class="preview-page print-root" :style="{ width: template.page.width + 'px' }">
          <div v-for="band in template.bands" :key="band.key" :style="{ height: band.height + 'px', position: 'relative', background: band.bgColor || '' }">
            <div
              v-for="el in band.elements"
              :key="el.id"
              :style="getElementStyle(el)"
              class="preview-element"
            >
              <div :style="getElementContentStyle(el)">
                <template v-if="el.type === 'staticText'">{{ el.props.text }}</template>
                <template v-else-if="el.type === 'text'">{{ getPreviewFieldValue(el.props.field) }}</template>
                <template v-else-if="el.type === 'pageNumber'">第 1 页</template>
                <template v-else-if="el.type === 'systemDate'">{{ new Date().toLocaleDateString() }}</template>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { ElMessage } from 'element-plus'
import {
  Document, FolderOpened, Download, ZoomIn, ZoomOut, Grid, Delete,
  CopyDocument, Printer, Back, Menu, Coin, Picture
} from '@element-plus/icons-vue'
import JsonEditor from './JsonEditor.vue'

function fieldLabel(el) {
  return el.props.field ? `\u007B\u007B${el.props.field}\u007D\u007D` : '数据字段'
}

// ─── Helpers ───────────────────────────────────────────────────────────────

const BAND_LABEL_WIDTH = 56 // must match .band-label width in CSS

let _idCounter = 1
function uid() {
  return 'el_' + Date.now() + '_' + (_idCounter++)
}

function defaultTextProps() {
  return {
    text: '文本',
    field: '',
    fontSize: 12,
    fontFamily: 'inherit',
    color: '#303133',
    bold: false,
    italic: false,
    underline: false,
    align: 'left',
    verticalAlign: 'middle',
    bgColor: '',
    opacity: 100
  }
}

function createElementByType(type, x, y) {
  const base = { id: uid(), type, x, y }
  switch (type) {
    case 'staticText':
      return { ...base, width: 120, height: 24, props: { ...defaultTextProps(), text: '静态文本' } }
    case 'text':
      return { ...base, width: 120, height: 24, props: { ...defaultTextProps(), field: '' } }
    case 'image':
      return { ...base, width: 80, height: 60, props: { field: '', objectFit: 'contain', bgColor: '', opacity: 100 } }
    case 'line':
      return { ...base, width: 200, height: 2, props: { color: '#333', lineWidth: 1, lineStyle: 'solid', bgColor: '', opacity: 100 } }
    case 'rectangle':
      return { ...base, width: 100, height: 60, props: { fillColor: 'rgba(255,255,255,0)', borderColor: '#333', borderWidth: 1, borderStyle: 'solid', borderRadius: 0, bgColor: '', opacity: 100 } }
    case 'pageNumber':
      return { ...base, width: 100, height: 24, props: { ...defaultTextProps(), format: '第{页}页/共{总}页', align: 'center' } }
    case 'systemDate':
      return { ...base, width: 120, height: 24, props: { ...defaultTextProps(), align: 'right' } }
    case 'barcode':
      return { ...base, width: 100, height: 60, props: { field: '', barcodeType: 'qrcode', bgColor: '', opacity: 100 } }
    default:
      return { ...base, width: 100, height: 24, props: defaultTextProps() }
  }
}

const defaultTemplate = () => ({
  name: '新建报表',
  page: { width: 794, height: 1123, margin: { top: 20, right: 20, bottom: 20, left: 20 } },
  bands: [
    { key: 'title', label: '标题段', height: 60, visible: true, bgColor: '', elements: [] },
    { key: 'pageHeader', label: '页眉段', height: 40, visible: true, bgColor: '', elements: [] },
    { key: 'columnHeader', label: '列标题段', height: 30, visible: true, bgColor: '', elements: [] },
    { key: 'detail', label: '明细段', height: 60, visible: true, bgColor: '', elements: [] },
    { key: 'columnFooter', label: '列汇总段', height: 30, visible: true, bgColor: '', elements: [] },
    { key: 'pageFooter', label: '页脚段', height: 40, visible: true, bgColor: '', elements: [] },
    { key: 'summary', label: '汇总段', height: 60, visible: true, bgColor: '', elements: [] }
  ]
})

// ─── State ─────────────────────────────────────────────────────────────────

const template = ref(defaultTemplate())
const selectedElement = ref(null)
const selectedBand = ref(null)
const currentBandKey = ref('detail')
const mode = ref('design')
const showGrid = ref(false)
const zoom = ref(1)
const leftTab = ref('elements')
const rightTab = ref('props')

const previewDataText = ref(JSON.stringify({
  company: 'SyReport Inc.',
  date: '2024-01-15',
  orderNo: 'ORD-20240115-001',
  items: [
    { name: '产品A', qty: 10, price: 99.9 },
    { name: '产品B', qty: 5, price: 199.9 }
  ],
  total: 1998.5
}, null, 2))

const dataFields = ref([])
const jsonTemplateText = ref('')

// ─── Computed ──────────────────────────────────────────────────────────────

const currentBand = computed(() => template.value.bands.find(b => b.key === currentBandKey.value))

const elementTypes = [
  { type: 'staticText', label: '静态文本', icon: 'DocumentCopy' },
  { type: 'text', label: '数据字段', icon: 'Coin' },
  { type: 'image', label: '图片', icon: 'Picture' },
  { type: 'line', label: '线条', icon: 'Minus' },
  { type: 'rectangle', label: '矩形', icon: 'Stop' },
  { type: 'pageNumber', label: '页码', icon: 'List' },
  { type: 'systemDate', label: '当前日期', icon: 'Calendar' },
  { type: 'barcode', label: '条码/二维码', icon: 'Grid' }
]

// ─── Template JSON sync ────────────────────────────────────────────────────

function updateTemplate() {
  jsonTemplateText.value = JSON.stringify(template.value, null, 2)
}

watch(template, updateTemplate, { deep: true, immediate: true })

// ─── Selection ────────────────────────────────────────────────────────────

function clearSelection() {
  selectedElement.value = null
}

function selectElement(el, band) {
  selectedElement.value = el
  if (band) currentBandKey.value = band.key
}

function selectBand(band) {
  currentBandKey.value = band.key
  clearSelection()
}

// ESC key handler
function onKeydown(e) {
  if (e.key === 'Escape') {
    clearSelection()
  } else if (e.key === 'Delete' && selectedElement.value) {
    deleteSelectedElement()
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
  parseDataFields()
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
})

// ─── Element CRUD ─────────────────────────────────────────────────────────

function deleteSelectedElement() {
  if (!selectedElement.value) return
  for (const band of template.value.bands) {
    const idx = band.elements.findIndex(e => e.id === selectedElement.value.id)
    if (idx !== -1) {
      band.elements.splice(idx, 1)
      selectedElement.value = null
      updateTemplate()
      ElMessage.success('元素已删除')
      return
    }
  }
}

function duplicateSelectedElement() {
  if (!selectedElement.value) return
  for (const band of template.value.bands) {
    const idx = band.elements.findIndex(e => e.id === selectedElement.value.id)
    if (idx !== -1) {
      const copy = JSON.parse(JSON.stringify(selectedElement.value))
      copy.id = uid()
      copy.x += 10
      copy.y += 10
      band.elements.splice(idx + 1, 0, copy)
      selectedElement.value = copy
      updateTemplate()
      return
    }
  }
}

// ─── Drag from Palette ────────────────────────────────────────────────────

let dragType = null
let dragFieldName = null

function onPaletteDragStart(e, type) {
  dragType = type
  dragFieldName = null
  e.dataTransfer.effectAllowed = 'copy'
}

function onFieldDragStart(e, fieldName) {
  dragType = 'text'
  dragFieldName = fieldName
  e.dataTransfer.effectAllowed = 'copy'
}

function onCanvasDrop(e, band) {
  if (!dragType) return
  const targetBand = band || currentBand.value
  if (!targetBand) return

  const canvasEl = e.currentTarget
  const rect = canvasEl.getBoundingClientRect()
  // Account for band label width
  const x = Math.round((e.clientX - rect.left - BAND_LABEL_WIDTH) / zoom.value)
  const y = Math.round((e.clientY - rect.top) / zoom.value)

  const el = createElementByType(dragType, Math.max(0, x), Math.max(0, y))
  if (dragFieldName && el.props) {
    el.props.field = dragFieldName
  }

  targetBand.elements.push(el)
  selectedElement.value = el
  currentBandKey.value = targetBand.key
  dragType = null
  dragFieldName = null
  updateTemplate()
}

// ─── Element Drag (move) ──────────────────────────────────────────────────

let isDraggingElement = false
let dragStartX = 0
let dragStartY = 0
let dragElOrigX = 0
let dragElOrigY = 0
let draggingEl = null

function startElementDrag(e, el, band) {
  selectElement(el, band)
  isDraggingElement = true
  dragStartX = e.clientX
  dragStartY = e.clientY
  dragElOrigX = el.x
  dragElOrigY = el.y
  draggingEl = el

  window.addEventListener('mousemove', onElementDragMove)
  window.addEventListener('mouseup', onElementDragEnd)
  e.preventDefault()
}

function onElementDragMove(e) {
  if (!isDraggingElement || !draggingEl) return
  const dx = Math.round((e.clientX - dragStartX) / zoom.value)
  const dy = Math.round((e.clientY - dragStartY) / zoom.value)
  draggingEl.x = Math.max(0, dragElOrigX + dx)
  draggingEl.y = Math.max(0, dragElOrigY + dy)
}

function onElementDragEnd() {
  isDraggingElement = false
  draggingEl = null
  window.removeEventListener('mousemove', onElementDragMove)
  window.removeEventListener('mouseup', onElementDragEnd)
  updateTemplate()
}

// ─── Element Resize ────────────────────────────────────────────────────────

let isResizing = false
let resizeHandle = null
let resizeStartX = 0
let resizeStartY = 0
let resizeOrigX = 0
let resizeOrigY = 0
let resizeOrigW = 0
let resizeOrigH = 0
let resizingEl = null

function startResize(e, el, handle) {
  isResizing = true
  resizeHandle = handle
  resizeStartX = e.clientX
  resizeStartY = e.clientY
  resizeOrigX = el.x
  resizeOrigY = el.y
  resizeOrigW = el.width
  resizeOrigH = el.height
  resizingEl = el

  window.addEventListener('mousemove', onResizeMove)
  window.addEventListener('mouseup', onResizeEnd)
  e.preventDefault()
}

function onResizeMove(e) {
  if (!isResizing || !resizingEl) return
  const dx = Math.round((e.clientX - resizeStartX) / zoom.value)
  const dy = Math.round((e.clientY - resizeStartY) / zoom.value)
  const minW = 10, minH = 10

  if (resizeHandle.includes('e')) resizingEl.width = Math.max(minW, resizeOrigW + dx)
  if (resizeHandle.includes('s')) resizingEl.height = Math.max(minH, resizeOrigH + dy)
  if (resizeHandle.includes('w')) {
    const newW = Math.max(minW, resizeOrigW - dx)
    resizingEl.x = resizeOrigX + (resizeOrigW - newW)
    resizingEl.width = newW
  }
  if (resizeHandle.includes('n')) {
    const newH = Math.max(minH, resizeOrigH - dy)
    resizingEl.y = resizeOrigY + (resizeOrigH - newH)
    resizingEl.height = newH
  }
}

function onResizeEnd() {
  isResizing = false
  resizingEl = null
  window.removeEventListener('mousemove', onResizeMove)
  window.removeEventListener('mouseup', onResizeEnd)
  updateTemplate()
}

// ─── Band Resize ──────────────────────────────────────────────────────────

let isBandResizing = false
let resizingBand = null
let bandResizeStartY = 0
let bandOrigHeight = 0

function startBandResize(e, band) {
  isBandResizing = true
  resizingBand = band
  bandResizeStartY = e.clientY
  bandOrigHeight = band.height
  window.addEventListener('mousemove', onBandResizeMove)
  window.addEventListener('mouseup', onBandResizeEnd)
  e.preventDefault()
}

function onBandResizeMove(e) {
  if (!isBandResizing || !resizingBand) return
  const dy = Math.round((e.clientY - bandResizeStartY) / zoom.value)
  resizingBand.height = Math.max(20, bandOrigHeight + dy)
}

function onBandResizeEnd() {
  isBandResizing = false
  resizingBand = null
  window.removeEventListener('mousemove', onBandResizeMove)
  window.removeEventListener('mouseup', onBandResizeEnd)
  updateTemplate()
}

// ─── Style Helpers ─────────────────────────────────────────────────────────

function getElementStyle(el) {
  const style = {
    left: el.x + 'px',
    top: el.y + 'px',
    width: el.width + 'px',
    height: el.height + 'px',
    position: 'absolute',
    cursor: 'move',
    userSelect: 'none'
  }
  if (el.props.opacity !== undefined && el.props.opacity < 100) {
    style.opacity = el.props.opacity / 100
  }
  if (el.props.bgColor) {
    style.backgroundColor = el.props.bgColor
  }
  if (el.type === 'rectangle') {
    style.backgroundColor = el.props.fillColor || 'transparent'
    style.border = `${el.props.borderWidth || 1}px ${el.props.borderStyle || 'solid'} ${el.props.borderColor || '#333'}`
    style.borderRadius = (el.props.borderRadius || 0) + 'px'
  }
  return style
}

function getElementContentStyle(el) {
  if (!el.props) return {}
  const style = {
    width: '100%',
    height: '100%',
    display: 'flex',
    alignItems: el.props.verticalAlign === 'top' ? 'flex-start' : el.props.verticalAlign === 'bottom' ? 'flex-end' : 'center',
    justifyContent: el.props.align === 'center' ? 'center' : el.props.align === 'right' ? 'flex-end' : 'flex-start',
    overflow: 'hidden',
    padding: '2px 4px'
  }
  if (el.props.fontSize) style.fontSize = el.props.fontSize + 'px'
  if (el.props.fontFamily && el.props.fontFamily !== 'inherit') style.fontFamily = el.props.fontFamily
  if (el.props.color) style.color = el.props.color
  if (el.props.bold) style.fontWeight = 'bold'
  if (el.props.italic) style.fontStyle = 'italic'
  if (el.props.underline) style.textDecoration = 'underline'
  return style
}

// ─── Preview ──────────────────────────────────────────────────────────────

function getPreviewFieldValue(field) {
  try {
    const data = JSON.parse(previewDataText.value)
    return data[field] !== undefined ? String(data[field]) : `{{${field}}}`
  } catch {
    return `{{${field}}}`
  }
}

// ─── Data Fields ──────────────────────────────────────────────────────────

function parseDataFields() {
  try {
    const data = JSON.parse(previewDataText.value)
    const fields = []
    function extractFields(obj, prefix = '') {
      if (typeof obj !== 'object' || obj === null) return
      for (const key of Object.keys(obj)) {
        const fullKey = prefix ? `${prefix}.${key}` : key
        if (Array.isArray(obj[key]) && obj[key].length > 0 && typeof obj[key][0] === 'object') {
          extractFields(obj[key][0], fullKey + '[0]')
        } else if (typeof obj[key] === 'object' && !Array.isArray(obj[key])) {
          extractFields(obj[key], fullKey)
        } else {
          fields.push(fullKey)
        }
      }
    }
    extractFields(data)
    dataFields.value = fields
    ElMessage.success(`解析到 ${fields.length} 个字段`)
  } catch (e) {
    ElMessage.error('JSON 格式错误，无法解析字段')
  }
}

function formatPreviewJson() {
  try {
    previewDataText.value = JSON.stringify(JSON.parse(previewDataText.value), null, 2)
    ElMessage.success('JSON 已格式化')
  } catch (e) {
    ElMessage.error('JSON 格式错误，无法格式化')
  }
}

// ─── Template Operations ──────────────────────────────────────────────────

function applyJsonTemplate() {
  try {
    const parsed = JSON.parse(jsonTemplateText.value)
    template.value = parsed
    selectedElement.value = null
    ElMessage.success('模板已应用')
  } catch (e) {
    ElMessage.error('JSON 格式错误，无法应用')
  }
}

function copyJsonTemplate() {
  navigator.clipboard.writeText(jsonTemplateText.value).then(() => {
    ElMessage.success('已复制到剪贴板')
  }).catch(() => {
    ElMessage.warning('复制失败，请手动复制')
  })
}

function newTemplate() {
  template.value = defaultTemplate()
  selectedElement.value = null
  ElMessage.success('已创建新报表')
}

function exportTemplate() {
  const blob = new Blob([jsonTemplateText.value], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = (template.value.name || 'report') + '.json'
  a.click()
  URL.revokeObjectURL(url)
}

function importTemplate() {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = '.json'
  input.onchange = (e) => {
    const file = e.target.files[0]
    if (!file) return
    const reader = new FileReader()
    reader.onload = (ev) => {
      try {
        const parsed = JSON.parse(ev.target.result)
        template.value = parsed
        selectedElement.value = null
        ElMessage.success('模板导入成功')
      } catch {
        ElMessage.error('导入失败：JSON 格式错误')
      }
    }
    reader.readAsText(file)
  }
  input.click()
}

function applyPaperSize(size) {
  const sizes = {
    a4p: { width: 794, height: 1123 },
    a4l: { width: 1123, height: 794 },
    a3p: { width: 1123, height: 1587 },
    a3l: { width: 1587, height: 1123 }
  }
  if (sizes[size]) {
    template.value.page.width = sizes[size].width
    template.value.page.height = sizes[size].height
    updateTemplate()
  }
}

function printReport() {
  mode.value = 'preview'
  nextTick(() => window.print())
}
</script>

<style scoped>
/* ─── Shell & Layout ────────────────────────────────────────────────────── */
.designer-shell {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  outline: none;
  background: #f0f2f5;
}

.top-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 48px;
  padding: 0 12px;
  background: #fff;
  border-bottom: 1px solid #e4e7ed;
  flex-shrink: 0;
  gap: 8px;
}

.toolbar-left,
.toolbar-right {
  display: flex;
  align-items: center;
  gap: 4px;
}

.toolbar-center {
  flex: 1;
  text-align: center;
}

.report-title {
  font-weight: 600;
  font-size: 15px;
  color: #303133;
}

.body-layout {
  display: flex;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

/* ─── Left Panel ────────────────────────────────────────────────────────── */
.left-panel {
  width: 220px;
  flex-shrink: 0;
  background: #fff;
  border-right: 1px solid #e4e7ed;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.left-tabs {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

.left-tabs :deep(.el-tabs__content) {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 0;
}

.left-tabs :deep(.el-tab-pane) {
  padding: 8px;
}

.palette-section {
  margin-bottom: 12px;
}

.palette-title {
  font-size: 11px;
  font-weight: 600;
  color: #909399;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 6px;
  padding: 0 2px;
}

.palette-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px;
}

.palette-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 3px;
  padding: 8px 4px;
  border: 1px solid #e4e7ed;
  border-radius: 6px;
  cursor: grab;
  font-size: 11px;
  color: #606266;
  background: #fafafa;
  transition: all 0.15s;
  user-select: none;
}

.palette-item:hover {
  border-color: #409eff;
  color: #409eff;
  background: #ecf5ff;
}

.palette-icon {
  font-size: 18px;
}

.band-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.band-list-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 8px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  color: #606266;
  transition: background 0.15s;
}

.band-list-item:hover {
  background: #f5f7fa;
}

.band-list-item.active {
  background: #ecf5ff;
  color: #409eff;
}

.band-count {
  margin-left: auto;
  background: #e4e7ed;
  border-radius: 10px;
  padding: 0 6px;
  font-size: 10px;
  color: #909399;
}

.data-section {
  padding: 4px 0;
}

.field-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  max-height: 200px;
  overflow-y: auto;
}

.field-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 5px 8px;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  cursor: grab;
  font-size: 12px;
  color: #606266;
  background: #fafafa;
  transition: all 0.15s;
}

.field-item:hover {
  border-color: #67c23a;
  color: #67c23a;
  background: #f0f9eb;
}

.field-empty {
  font-size: 12px;
  color: #c0c4cc;
  text-align: center;
  padding: 12px 0;
}

/* ─── Canvas ────────────────────────────────────────────────────────────── */
.canvas-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-width: 0;
}

.canvas-toolbar {
  height: 40px;
  background: #fff;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  align-items: center;
  padding: 0 12px;
  gap: 8px;
  flex-shrink: 0;
}

.canvas-hint {
  font-size: 12px;
  color: #c0c4cc;
}

.canvas-scroll {
  flex: 1;
  overflow: auto;
  min-height: 0;
  min-width: 0;
  padding: 24px;
  background: #eef2f6;
}

.page-canvas {
  background: #fff;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.12);
  position: relative;
  margin: 0 auto;
}

.page-canvas.show-grid {
  background-image:
    linear-gradient(rgba(0, 0, 0, 0.05) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 0, 0, 0.05) 1px, transparent 1px);
  background-size: 10px 10px;
}

/* ─── Bands ─────────────────────────────────────────────────────────────── */
.band {
  position: relative;
  border-bottom: 1px dashed #d0d7de;
  box-sizing: border-box;
}

.band-active {
  background-color: rgba(64, 158, 255, 0.02);
}

.band-label {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 56px;
  background: #f0f2f5;
  border-right: 1px solid #e4e7ed;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  color: #909399;
  writing-mode: vertical-rl;
  text-orientation: mixed;
  cursor: pointer;
  user-select: none;
  letter-spacing: 1px;
}

.band:hover .band-label {
  background: #e9ecf0;
  color: #409eff;
}

.band-active .band-label {
  background: #ecf5ff;
  color: #409eff;
}

.band-resize-handle {
  position: absolute;
  bottom: 0;
  left: 56px;
  right: 0;
  height: 4px;
  cursor: s-resize;
  background: transparent;
  transition: background 0.15s;
}

.band-resize-handle:hover {
  background: rgba(64, 158, 255, 0.3);
}

/* ─── Elements ──────────────────────────────────────────────────────────── */
.report-element {
  position: absolute;
  box-sizing: border-box;
  border: 1px dashed transparent;
  transition: border-color 0.1s;
}

.report-element:hover {
  border-color: #b3d8ff;
}

.report-element.selected {
  border: 1px solid #409eff;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

.element-content {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.line-element {
  width: 100%;
  border-top-style: solid;
  margin-top: 50%;
  transform: translateY(-50%);
}

/* ─── Resize Handles ────────────────────────────────────────────────────── */
.resize-handle {
  position: absolute;
  width: 8px;
  height: 8px;
  background: #fff;
  border: 1px solid #409eff;
  border-radius: 1px;
  z-index: 10;
}

.handle-nw { top: -4px; left: -4px; cursor: nw-resize; }
.handle-n  { top: -4px; left: calc(50% - 4px); cursor: n-resize; }
.handle-ne { top: -4px; right: -4px; cursor: ne-resize; }
.handle-e  { top: calc(50% - 4px); right: -4px; cursor: e-resize; }
.handle-se { bottom: -4px; right: -4px; cursor: se-resize; }
.handle-s  { bottom: -4px; left: calc(50% - 4px); cursor: s-resize; }
.handle-sw { bottom: -4px; left: -4px; cursor: sw-resize; }
.handle-w  { top: calc(50% - 4px); left: -4px; cursor: w-resize; }

/* ─── Right Panel ───────────────────────────────────────────────────────── */
.right-panel {
  width: 260px;
  flex-shrink: 0;
  background: #fff;
  border-left: 1px solid #e4e7ed;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.right-tabs {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

.right-tabs :deep(.el-tabs__content) {
  flex: 1;
  overflow: hidden;
  padding: 0;
}

.right-tabs :deep(.el-tab-pane) {
  height: 100%;
  overflow: hidden;
}

.props-scroll {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 8px;
}

.props-section {
  margin-bottom: 16px;
}

.props-title {
  font-size: 11px;
  font-weight: 600;
  color: #909399;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
  padding: 4px 0;
  border-bottom: 1px solid #f0f2f5;
}

.json-panel {
  height: 100%;
  padding: 8px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* ─── Preview ───────────────────────────────────────────────────────────── */
.preview-overlay {
  position: absolute;
  inset: 0;
  z-index: 100;
  background: #eef2f6;
  display: flex;
  flex-direction: column;
}

.preview-toolbar {
  height: 48px;
  background: #fff;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  align-items: center;
  padding: 0 16px;
  gap: 8px;
  flex-shrink: 0;
}

.preview-scroll {
  flex: 1;
  overflow: auto;
  padding: 24px;
}

.preview-page {
  background: #fff;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.12);
  margin: 0 auto;
  overflow: hidden;
}

.preview-element {
  position: absolute;
  overflow: hidden;
}
</style>
