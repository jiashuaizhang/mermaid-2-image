<script setup>
import { ref, watch } from 'vue'
import { ElMessage } from 'element-plus'
import { Loading, WarningFilled } from '@element-plus/icons-vue'
import mermaid from 'mermaid'

const DEFAULT_CODE = `flowchart LR
  A[开始] --> B{判断}
  B -->|是| C[执行]
  B -->|否| D[结束]
  C --> D`

const source = ref(DEFAULT_CODE)
const previewSvg = ref('')
const previewError = ref('')
const loading = ref(false)
const exportLoading = ref(false)

const baseMermaidConfig = {
  startOnLoad: false,
  theme: 'default',
  securityLevel: 'loose',
  fontFamily: 'JetBrains Mono, monospace',
  flowchart: {
    diagramPadding: 24,
    nodeSpacing: 56,
    rankSpacing: 56,
    padding: 18,
  },
}
mermaid.initialize(baseMermaidConfig)

async function renderPreview() {
  const code = source.value.trim()
  if (!code) {
    previewSvg.value = ''
    previewError.value = ''
    return
  }
  loading.value = true
  previewError.value = ''
  try {
    const { svg } = await mermaid.render('mermaid-preview-' + Date.now(), code)
    previewSvg.value = svg
  } catch (e) {
    previewError.value = e.message || '渲染失败'
    previewSvg.value = ''
  } finally {
    loading.value = false
  }
}

/** Parse width/height from SVG string (mermaid outputs pixel values) */
function getSvgDimensions(svgString) {
  const wMatch = svgString.match(/width="([^"]+)"/)
  const hMatch = svgString.match(/height="([^"]+)"/)
  let w = 800
  let h = 600
  if (wMatch) w = Math.ceil(parseFloat(wMatch[1]) || w)
  if (hMatch) h = Math.ceil(parseFloat(hMatch[1]) || h)
  return { w, h }
}

/** Resize SVG string to target pixel dimensions (keeps viewBox so content scales) */
function resizeSvgString(svgString, outW, outH) {
  return svgString
    .replace(/width="[^"]*"/, `width="${outW}"`)
    .replace(/height="[^"]*"/, `height="${outH}"`)
}

/** Export: ensure both long and short side have enough pixels (LR stays sharp, TD too) */
const EXPORT_MIN_LONG_SIDE = 3200
const EXPORT_MIN_SHORT_SIDE = 1600

/** Export: resize SVG to target pixels then rasterize with browser */
async function svgToPng(svgString) {
  const { w, h } = getSvgDimensions(svgString)
  const longSide = Math.max(w, h)
  const shortSide = Math.min(w, h)
  const scaleLong = EXPORT_MIN_LONG_SIDE / longSide
  const scaleShort = EXPORT_MIN_SHORT_SIDE / shortSide
  const scale = Math.max(3, scaleLong, scaleShort)
  const outW = Math.round(w * scale)
  const outH = Math.round(h * scale)

  const dataUrl = 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(resizeSvgString(svgString, outW, outH))
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => {
      const canvas = document.createElement('canvas')
      canvas.width = img.naturalWidth
      canvas.height = img.naturalHeight
      const ctx = canvas.getContext('2d')
      ctx.fillStyle = '#ffffff'
      ctx.fillRect(0, 0, canvas.width, canvas.height)
      ctx.drawImage(img, 0, 0)
      try {
        canvas.toBlob((blob) => (blob ? resolve(blob) : reject(new Error('Canvas to blob failed'))), 'image/png', 1)
      } catch (e) {
        reject(e)
      }
    }
    img.onerror = () => reject(new Error('图片加载失败，请重试'))
    img.src = dataUrl
  })
}

async function exportPng() {
  if (!previewSvg.value) {
    ElMessage.warning('请先输入并成功渲染 Mermaid 代码')
    return
  }
  exportLoading.value = true
  try {
    const code = source.value.trim()
    const exportConfig = {
      ...baseMermaidConfig,
      flowchart: {
        ...baseMermaidConfig.flowchart,
        diagramPadding: 40,
        nodeSpacing: 80,
        rankSpacing: 80,
        padding: 22,
      },
    }
    mermaid.initialize(exportConfig)
    const { svg } = await mermaid.render('mermaid-export-' + Date.now(), code)
    mermaid.initialize(baseMermaidConfig)

    const blob = await svgToPng(svg)
    const a = document.createElement('a')
    a.href = URL.createObjectURL(blob)
    a.download = `mermaid-${Date.now()}.png`
    a.click()
    URL.revokeObjectURL(a.href)
    ElMessage.success('已导出 PNG')
  } catch (e) {
    ElMessage.error(e.message || '导出失败')
  } finally {
    exportLoading.value = false
  }
}

watch(source, () => {
  const t = setTimeout(renderPreview, 400)
  return () => clearTimeout(t)
}, { immediate: true })
</script>

<template>
  <div class="page">
    <header class="header">
      <h1 class="title">Mermaid → PNG</h1>
      <p class="subtitle">编辑或粘贴 Mermaid 源码，实时预览并导出为 PNG</p>
    </header>

    <div class="layout">
      <el-card class="card editor-card" shadow="never">
        <template #header>
          <span>源码</span>
        </template>
        <el-input
          v-model="source"
          type="textarea"
          :autosize="{ minRows: 14, maxRows: 24 }"
          placeholder="在此粘贴或编辑 Mermaid 代码..."
          class="source-input"
        />
      </el-card>

      <div class="preview-column">
        <el-card class="card preview-card" shadow="never">
          <template #header>
            <span>预览</span>
            <el-button
              type="primary"
              :loading="exportLoading"
              :disabled="!previewSvg"
              class="export-btn"
              @click="exportPng"
            >
              导出 PNG
            </el-button>
          </template>
          <div class="preview-wrap">
            <div v-if="loading" class="preview-loading">
              <el-icon class="is-loading"><Loading /></el-icon>
              <span>渲染中...</span>
            </div>
            <div v-else-if="previewError" class="preview-error">
              <el-icon><WarningFilled /></el-icon>
              <pre>{{ previewError }}</pre>
            </div>
            <div v-else-if="previewSvg" class="preview-svg" v-html="previewSvg" />
            <div v-else class="preview-empty">输入 Mermaid 代码后将在此显示预览</div>
          </div>
        </el-card>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  max-width: 1200px;
  margin: 0 auto;
}

.header {
  text-align: center;
  margin-bottom: 2rem;
}

.title {
  font-size: 1.75rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 0 0 0.35rem 0;
  color: var(--text);
}

.subtitle {
  color: var(--text-muted);
  font-size: 0.95rem;
  margin: 0;
}

.layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
  align-items: start;
}

@media (max-width: 900px) {
  .layout {
    grid-template-columns: 1fr;
  }
}

.card {
  border-radius: var(--radius);
}

.card :deep(.el-card__header) {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.9rem 1.1rem;
}

.source-input {
  width: 100%;
}

.source-input :deep(.el-textarea__inner) {
  border-radius: var(--radius-sm);
  padding: 0.85rem 1rem;
  line-height: 1.55;
}

.preview-column {
  position: sticky;
  top: 1.5rem;
}

.export-btn {
  font-weight: 500;
}

.preview-wrap {
  min-height: 280px;
  border-radius: var(--radius-sm);
  background: #ffffff;
  border: 1px dashed var(--border);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  overflow: auto;
}

.preview-loading,
.preview-error,
.preview-empty {
  color: var(--text-muted);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.preview-error {
  color: var(--error);
  text-align: left;
  align-items: flex-start;
}

.preview-error pre {
  margin: 0;
  font-size: 0.8rem;
  white-space: pre-wrap;
  max-width: 100%;
}

.preview-svg {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.preview-svg :deep(svg) {
  max-width: 100%;
  height: auto;
}
</style>
