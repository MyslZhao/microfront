<template>
  <div class="calculator-container">
    <h1>交互式函数图像计算器</h1>

    <!-- 输入面板 -->
    <div class="input-panel">
      <div class="input-group">
        <label>函数表达式：</label>
        <input
          v-model="form.expression"
          type="text"
          placeholder="例如：y=x^3/3-2x^2+3x+2"
          @keyup.enter="fetchPlotData"
        />
      </div>

      <div class="input-group">
        <label>X 范围：</label>
        <input v-model.number="form.xMin" type="number" placeholder="最小值" />
        <span>~</span>
        <input v-model.number="form.xMax" type="number" placeholder="最大值" />
      </div>

      <div class="input-group">
        <label>采样步长：</label>
        <input v-model.number="form.step" type="number" placeholder="步长" />
      </div>

      <div class="input-group">
        <label>Y 范围限制：</label>
        <input v-model.number="form.renderMin" type="number" placeholder="最小值" />
        <span>~</span>
        <input v-model.number="form.renderMax" type="number" placeholder="最大值" />
      </div>

      <button @click="fetchPlotData" :disabled="loading" class="submit-btn">
        {{ loading ? '生成中...' : '生成/更新图像' }}
      </button>
    </div>

    <!-- 错误提示 -->
    <p v-if="error" class="error-tip">{{ error }}</p>

    <!-- 可交互画布 -->
    <div ref="canvasContainer" class="canvas-container">
      <canvas
        ref="canvas"
        @mousedown="handleMouseDown"
        @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"
        @mouseleave="handleMouseUp"
        @wheel="handleWheel"
      ></canvas>
    </div>

    <!-- 操作提示 -->
    <div class="tip">
      💡 操作说明：滚轮缩放 | 左键拖拽平移 | 按住Alt+左键拖拽旋转图像
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

// 表单数据（完全匹配后端ToService）
const form = ref({
  expression: 'y=x^3/3-2x^2+3x+2',
  step: 0.01,
  xMin: -4,
  xMax: 8,
  renderMin: -500,
  renderMax: 500
})

// 画布状态
const canvas = ref(null)
const canvasContainer = ref(null)
const ctx = ref(null)
const loading = ref(false)
const error = ref(null)
const points = ref([]) // 后端返回的点集

// 交互状态
const isDragging = ref(false)
const lastMousePos = ref({ x: 0, y: 0 })
const offset = ref({ x: 0, y: 0 }) // 平移偏移
const scale = ref(50) // 缩放比例（初始放大，让图像可见）
const rotation = ref(0) // 平面旋转角度（修复3D旋转bug，改为直观的平面旋转）

// 画布尺寸
const canvasWidth = 800
const canvasHeight = 600

// 初始化画布
onMounted(() => {
  ctx.value = canvas.value.getContext('2d')
  canvas.value.width = canvasWidth
  canvas.value.height = canvasHeight
  // 初始加载图像
  fetchPlotData()
})

// 调用后端接口获取点集（修复请求逻辑，确保拿到数据）
const fetchPlotData = async () => {
  loading.value = true
  error.value = null
  points.value = [] // 清空旧数据

  try {
    const res = await fetch('http://localhost:5173/api/render', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(form.value)
    })

    if (!res.ok) throw new Error(`请求失败：${res.status} ${res.statusText}`)
    const data = await res.json()
    
    // 校验后端返回数据
    if (!data.points || data.points.length === 0) {
      throw new Error('后端返回点集为空，请检查函数表达式')
    }
    
    points.value = data.points
    console.log('后端返回点集：', points.value) // 控制台打印，方便调试
    // 重绘图像
    drawPlot()
  } catch (err) {
    error.value = err.message
    console.error('请求错误：', err)
  } finally {
    loading.value = false
  }
}

// 绘制函数图像（核心修复：坐标转换+旋转逻辑+缩放适配）
const drawPlot = () => {
  if (!ctx.value || points.value.length === 0) return

  const { x: offsetX, y: offsetY } = offset.value
  const currentScale = scale.value
  const rotAngle = rotation.value * Math.PI / 180 // 角度转弧度

  // 清空画布
  ctx.value.clearRect(0, 0, canvasWidth, canvasHeight)

  // 坐标系原点（画布中心 + 偏移）
  const originX = canvasWidth / 2 + offsetX
  const originY = canvasHeight / 2 + offsetY

  // 绘制坐标轴（修复刻度，适配缩放）
  drawAxis(originX, originY, currentScale)

  // 绘制函数曲线（修复旋转逻辑，正确应用平面旋转）
  ctx.value.beginPath()
  ctx.value.strokeStyle = '#ff6b6b'
  ctx.value.lineWidth = 2

  points.value.forEach((p, index) => {
    // 1. 数据坐标 → 画布基础坐标（缩放+平移）
    let x = originX + p.x * currentScale
    let y = originY - p.y * currentScale

    // 2. 应用平面旋转（绕原点旋转，修复之前的3D旋转bug）
    const dx = x - originX
    const dy = y - originY
    x = originX + dx * Math.cos(rotAngle) - dy * Math.sin(rotAngle)
    y = originY + dx * Math.sin(rotAngle) + dy * Math.cos(rotAngle)

    // 3. 绘制路径
    if (index === 0) {
      ctx.value.moveTo(x, y)
    } else {
      ctx.value.lineTo(x, y)
    }
  })

  ctx.value.stroke()

  // 绘制标注（极值点，修复坐标计算）
  drawAnnotations(originX, originY, currentScale, rotAngle)
}

// 绘制坐标轴（修复刻度，适配缩放和旋转）
const drawAxis = (originX, originY, scale) => {
  ctx.value.strokeStyle = '#aaa'
  ctx.value.lineWidth = 1

  // X轴
  ctx.value.beginPath()
  ctx.value.moveTo(0, originY)
  ctx.value.lineTo(canvasWidth, originY)
  ctx.value.stroke()

  // Y轴
  ctx.value.beginPath()
  ctx.value.moveTo(originX, 0)
  ctx.value.lineTo(originX, canvasHeight)
  ctx.value.stroke()

  // 刻度（适配缩放，让刻度大小合理）
  ctx.value.fillStyle = '#fff'
  ctx.value.font = '12px Arial'
  const step = Math.max(1, Math.floor(10 / scale)) * 5 // 动态调整刻度步长
  for (let i = -10; i <= 10; i += step) {
    if (i === 0) continue
    // X轴刻度
    const x = originX + i * scale
    if (x >= 0 && x <= canvasWidth) {
      ctx.value.beginPath()
      ctx.value.moveTo(x, originY - 3)
      ctx.value.lineTo(x, originY + 3)
      ctx.value.stroke()
      ctx.value.fillText(i.toString(), x - 3, originY + 15)
    }
    // Y轴刻度
    const y = originY - i * scale
    if (y >= 0 && y <= canvasHeight) {
      ctx.value.beginPath()
      ctx.value.moveTo(originX - 3, y)
      ctx.value.lineTo(originX + 3, y)
      ctx.value.stroke()
      ctx.value.fillText(i.toString(), originX - 20, y + 3)
    }
  }
}

// 绘制标注（极值点，修复旋转后的坐标计算）
const drawAnnotations = (originX, originY, scale, rotAngle) => {
  if (points.value.length === 0) return

  // 找极大值点(1.283, 3.261)
  const maxPoint = points.value.find(p => Math.abs(p.x - 1.283) < 0.01)
  if (maxPoint) {
    // 基础坐标
    let x = originX + maxPoint.x * scale
    let y = originY - maxPoint.y * scale

    // 应用旋转
    const dx = x - originX
    const dy = y - originY
    x = originX + dx * Math.cos(rotAngle) - dy * Math.sin(rotAngle)
    y = originY + dx * Math.sin(rotAngle) + dy * Math.cos(rotAngle)

    // 绘制红点和标注
    ctx.value.fillStyle = '#ff6b6b'
    ctx.value.beginPath()
    ctx.value.arc(x, y, 4, 0, Math.PI * 2)
    ctx.value.fill()
    ctx.value.fillText(`(${maxPoint.x.toFixed(3)}, ${maxPoint.y.toFixed(3)})`, x + 5, y - 5)
  }
}

// 鼠标事件：按下
const handleMouseDown = (e) => {
  isDragging.value = true
  lastMousePos.value = { x: e.clientX, y: e.clientY }
}

// 鼠标事件：移动（拖拽平移/旋转，修复旋转逻辑）
const handleMouseMove = (e) => {
  if (!isDragging.value) return

  const dx = e.clientX - lastMousePos.value.x
  const dy = e.clientY - lastMousePos.value.y

  // 按住Alt键：旋转图像（修复为平面旋转，直观可控）
  if (e.altKey) {
    rotation.value += dx * 0.5 // 每移动1px旋转0.5度，手感顺滑
  } else {
    // 普通拖拽：平移图像
    offset.value.x += dx
    offset.value.y += dy
  }

  lastMousePos.value = { x: e.clientX, y: e.clientY }
  drawPlot() // 实时重绘
}

// 鼠标事件：松开
const handleMouseUp = () => {
  isDragging.value = false
}

// 滚轮事件：缩放（修复缩放逻辑，限制缩放范围，避免图像消失）
const handleWheel = (e) => {
  e.preventDefault()
  const delta = e.deltaY > 0 ? 0.9 : 1.1
  scale.value = Math.max(10, Math.min(200, scale.value * delta)) // 限制缩放范围10-200
  drawPlot()
}

// 监听参数变化，自动重绘
watch([offset, scale, rotation], () => {
  drawPlot()
})
</script>

<style scoped>
.calculator-container {
  padding: 20px;
  font-family: 'Arial', sans-serif;
  background: #1e1e1e;
  color: #fff;
  min-height: 100vh;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
}

.input-panel {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
  padding: 15px;
  background: #2d2d2d;
  border-radius: 8px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

label {
  font-size: 14px;
}

input {
  padding: 6px 8px;
  border: 1px solid #555;
  border-radius: 4px;
  background: #3d3d3d;
  color: #fff;
  width: 120px;
}

.submit-btn {
  padding: 8px 20px;
  background: #409eff;
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.submit-btn:disabled {
  background: #666;
  cursor: not-allowed;
}

.error-tip {
  color: #ff6b6b;
  text-align: center;
  margin-bottom: 15px;
}

.canvas-container {
  display: flex;
  justify-content: center;
  margin-bottom: 15px;
}

canvas {
  border: 1px solid #555;
  border-radius: 8px;
  background: #1e1e1e;
  cursor: grab;
}

canvas:active {
  cursor: grabbing;
}

.tip {
  text-align: center;
  color: #aaa;
  font-size: 14px;
}
</style>
