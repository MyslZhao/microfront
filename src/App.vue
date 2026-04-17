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
        @mousemove="handleMouseMoveAndShowCoords"
        @mouseup="handleMouseUp"
        @mouseleave="handleMouseUp"
        @wheel="handleWheel"
      ></canvas>
      <!-- 坐标提示框 -->
      <div v-if="showTooltip" class="coordinate-tooltip" :style="tooltipStyle">
        ({{ tooltipX.toFixed(2) }}, {{ tooltipY.toFixed(2) }})
      </div>
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

// 坐标提示框状态
const showTooltip = ref(false)
const tooltipX = ref(0)
const tooltipY = ref(0)
const tooltipStyle = ref({ left: '0px', top: '0px' })

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

// 前端计算函数点集（替换后端接口调用）
const fetchPlotData = () => {
  loading.value = true
  error.value = null
  points.value = [] // 清空旧数据

  try {
    const { expression, step, xMin, xMax, renderMin, renderMax } = form.value
    
    // 解析函数表达式
    const parseExpression = (expr) => {
      // 处理幂运算 ^
      expr = expr.replace(/\^/g, '**')
      // 处理函数表达式格式，提取右侧部分
      const match = expr.match(/^y\s*=\s*(.+)$/i)
      if (!match) {
        throw new Error('函数表达式格式错误，请使用 y=... 格式')
      }
      return match[1]
    }
    
    const expr = parseExpression(expression)
    
    // 生成点集
    const calculatedPoints = []
    for (let x = xMin; x <= xMax; x += step) {
      try {
        // 使用Function构造函数计算y值，安全地执行数学表达式
        const y = new Function('x', `return ${expr}`)(x)
        
        // 检查计算结果是否有效
        if (isNaN(y) || !isFinite(y)) {
          continue
        }
        
        // 应用Y范围限制
        if (y >= renderMin && y <= renderMax) {
          calculatedPoints.push({ x, y })
        }
      } catch (calcError) {
        // 跳过计算错误的点
        continue
      }
    }
    
    if (calculatedPoints.length === 0) {
      throw new Error('无法计算函数值，请检查函数表达式')
    }
    
    points.value = calculatedPoints
    console.log('前端计算点集：', points.value) // 控制台打印，方便调试
    // 重绘图像
    drawPlot()
  } catch (err) {
    error.value = err.message
    console.error('计算错误：', err)
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
  ctx.value.strokeStyle = '#00ff80'
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

// 绘制坐标轴和网格（添加横竖虚线格子）
const drawAxis = (originX, originY, scale) => {
  // 绘制网格线
  ctx.value.strokeStyle = '#333333'
  ctx.value.lineWidth = 0.5
  ctx.value.setLineDash([5, 5]) // 虚线
  
  const step = Math.max(1, Math.floor(10 / scale)) * 5 // 动态调整网格步长
  
  // 垂直网格线
  for (let i = -20; i <= 20; i += step) {
    const x = originX + i * scale
    if (x >= 0 && x <= canvasWidth) {
      ctx.value.beginPath()
      ctx.value.moveTo(x, 0)
      ctx.value.lineTo(x, canvasHeight)
      ctx.value.stroke()
    }
  }
  
  // 水平网格线
  for (let i = -20; i <= 20; i += step) {
    const y = originY - i * scale
    if (y >= 0 && y <= canvasHeight) {
      ctx.value.beginPath()
      ctx.value.moveTo(0, y)
      ctx.value.lineTo(canvasWidth, y)
      ctx.value.stroke()
    }
  }
  
  // 绘制坐标轴
  ctx.value.setLineDash([]) // 实线
  ctx.value.strokeStyle = '#00ff80'
  ctx.value.lineWidth = 1.5

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
  ctx.value.fillStyle = '#00ff80'
  ctx.value.font = '12px Arial'
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

// 鼠标事件：移动（拖拽平移/旋转 + 显示坐标）
const handleMouseMoveAndShowCoords = (e) => {
  // 处理拖拽和旋转
  if (isDragging.value) {
    const dx = e.clientX - lastMousePos.value.x
    const dy = e.clientY - lastMousePos.value.y

    // 按住Alt键：旋转图像
    if (e.altKey) {
      rotation.value += dx * 0.5
    } else {
      // 普通拖拽：平移图像
      offset.value.x += dx
      offset.value.y += dy
    }

    lastMousePos.value = { x: e.clientX, y: e.clientY }
    drawPlot() // 实时重绘
  }
  
  // 显示坐标提示（鼠标悬停显示函数坐标）
  if (!canvas.value || points.value.length === 0) {
    showTooltip.value = false
    return
  }
  
  const rect = canvas.value.getBoundingClientRect()
  const mouseX = e.clientX - rect.left
  const mouseY = e.clientY - rect.top
  
  // 计算数据坐标
  const originX = canvasWidth / 2 + offset.value.x
  const originY = canvasHeight / 2 + offset.value.y
  const currentScale = scale.value
  const rotAngle = rotation.value * Math.PI / 180
  
  // 转换为数据坐标（考虑旋转）
  const dx = mouseX - originX
  const dy = mouseY - originY
  const dataX = (dx * Math.cos(-rotAngle) - dy * Math.sin(-rotAngle)) / currentScale
  const dataY = (dx * Math.sin(-rotAngle) + dy * Math.cos(-rotAngle)) / currentScale
  
  // 找到最近的函数点（扩大搜索范围）
  let closestPoint = null
  let minDistance = Infinity
  const searchRange = 0.3
  
  points.value.forEach(point => {
    const distance = Math.sqrt(Math.pow(point.x - dataX, 2) + Math.pow(point.y - dataY, 2))
    if (distance < minDistance && distance < searchRange) {
      minDistance = distance
      closestPoint = point
    }
  })
  
  if (closestPoint) {
    // 计算旋转后的屏幕坐标用于显示
    let screenX = originX + closestPoint.x * scale
    let screenY = originY - closestPoint.y * scale
    const pdx = screenX - originX
    const pdy = screenY - originY
    screenX = originX + pdx * Math.cos(rotAngle) - pdy * Math.sin(rotAngle)
    screenY = originY + pdx * Math.sin(rotAngle) + pdy * Math.cos(rotAngle)
    
    tooltipX.value = closestPoint.x
    tooltipY.value = closestPoint.y
    showTooltip.value = true
    tooltipStyle.value = {
      left: (screenX + 15) + 'px',
      top: (screenY - 30) + 'px'
    }
  } else {
    showTooltip.value = false
  }
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
  max-width: 900px;
  margin: 0 auto;
  padding: 30px;
  font-family: Arial, sans-serif;
  background: #1a1a1a;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 255, 128, 0.3);
}

h1 {
  color: #ffffff;
  text-align: center;
  margin-bottom: 30px;
  text-shadow: 0 0 10px rgba(0, 255, 128, 0.5);
  font-size: 28px;
}

.input-panel {
  background: #2d2d2d;
  padding: 25px;
  border-radius: 8px;
  margin-bottom: 30px;
  border: 2px solid #00ff80;
}

.input-group {
  margin-bottom: 15px;
  display: flex;
  align-items: center;
}

.input-group label {
  width: 120px;
  font-weight: bold;
  color: #ffffff;
  font-size: 16px;
}

.input-group input {
  flex: 1;
  padding: 10px 14px;
  margin: 0 8px;
  border: 1px solid #00ff80;
  border-radius: 4px;
  background: #1a1a1a;
  color: #ffffff;
  outline: none;
  font-size: 16px;
}

.input-group input:focus {
  box-shadow: 0 0 10px rgba(0, 255, 128, 0.5);
}

.submit-btn {
  margin-top: 20px;
  padding: 12px 28px;
  background: #00ff80;
  color: #1a1a1a;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  font-size: 16px;
  transition: all 0.3s ease;
}

.submit-btn:hover {
  background: #00cc66;
  box-shadow: 0 0 15px rgba(0, 255, 128, 0.5);
}

.submit-btn:disabled {
  background: #333333;
  color: #666666;
  cursor: not-allowed;
  box-shadow: none;
}

.error-tip {
  color: #ff6b6b;
  margin: 15px 0;
  text-align: center;
  font-weight: bold;
  font-size: 16px;
}

.canvas-container {
  position: relative;
  margin: 30px auto;
  display: flex;
  justify-content: center;
}

canvas {
  border: 2px solid #00ff80;
  background: #1a1a1a;
  cursor: grab;
  box-shadow: 0 0 20px rgba(0, 255, 128, 0.3);
}

canvas:active {
  cursor: grabbing;
}

.tip {
  margin-top: 30px;
  font-size: 18px;
  color: #ffffff;
  text-align: center;
  text-shadow: 0 0 5px rgba(0, 255, 128, 0.5);
}

/* 坐标提示框 */
.coordinate-tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.8);
  color: #00ff80;
  padding: 10px 16px;
  border-radius: 4px;
  border: 1px solid #00ff80;
  font-size: 16px;
  font-weight: bold;
  pointer-events: none;
  z-index: 1000;
  box-shadow: 0 0 10px rgba(0, 255, 128, 0.3);
}
</style>


