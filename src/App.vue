<script setup>
import { ref, onMounted, onUnmounted, nextTick, watch } from 'vue'

// 响应式数据
const canvas = ref(null)
const ctx = ref(null)
const ws = ref(null)
const isDrawing = ref(false)
const currentTool = ref('brush') // 'brush' 或 'eraser'
const currentColor = ref('#000000')
const brushSize = ref(5)
const eraserSize = ref(20)
const isConnected = ref(false)
const lastX = ref(0)
const lastY = ref(0)

// 连接配置相关
const showDrawingBoard = ref(false)
const isConnecting = ref(false)
const connectionError = ref('')
const serverConfig = ref({
  ip: '127.0.0.1',
  port: 8080
})

// 预设颜色
const colors = [
  '#000000', '#FF0000', '#00FF00', '#0000FF', 
  '#FFFF00', '#FF00FF', '#00FFFF', '#FFA500',
  '#800080', '#FFC0CB', '#A52A2A', '#808080'
]

// 生成唯一ID
const generateUniqueId = () => {
  return Date.now().toString(36) + Math.random().toString(36).substr(2)
}

// WebSocket连接
const connectWebSocket = () => {
  try {
    const wsUrl = `ws://${serverConfig.value.ip}:${serverConfig.value.port}`
    ws.value = new WebSocket(wsUrl)
    
    ws.value.onopen = () => {
      console.log('连接到绘图服务器')
      isConnected.value = true
      showDrawingBoard.value = true
      isConnecting.value = false
      connectionError.value = ''
    }
    
    ws.value.onmessage = (event) => {
      const command = JSON.parse(event.data)
      handleRemoteCommand(command)
    }
    
    ws.value.onclose = () => {
      console.log('与服务器断开连接')
      isConnected.value = false
      isConnecting.value = false
      if (showDrawingBoard.value) {
        // 如果在绘画界面断开连接，尝试重连
        setTimeout(connectWebSocket, 3000)
      }
    }
    
    ws.value.onerror = (error) => {
      console.error('WebSocket错误:', error)
      isConnected.value = false
      isConnecting.value = false
      connectionError.value = '连接失败，请检查服务器地址和端口是否正确'
    }
  } catch (error) {
    console.error('连接失败:', error)
    isConnected.value = false
    isConnecting.value = false
    connectionError.value = '连接失败，请检查服务器地址和端口是否正确'
  }
}

// 连接到服务器
const connectToServer = () => {
  if (isConnecting.value) return
  
  isConnecting.value = true
  connectionError.value = ''
  connectWebSocket()
}

// 断开连接并返回配置页面
  const disconnectAndGoBack = () => {
    if (ws.value) {
      ws.value.close()
      ws.value = null
    }
    isConnected.value = false
    showDrawingBoard.value = false
    isConnecting.value = false
    connectionError.value = ''
  }

// 处理远程命令
const handleRemoteCommand = (command) => {
  if (!ctx.value) return
  
  switch (command.type) {
    case 'line':
      drawRemoteLine(command)
      break
    case 'erase':
      eraseRemote(command)
      break
    case 'clear':
      clearCanvas()
      break
  }
}

// 绘制远程线条
const drawRemoteLine = (command) => {
  ctx.value.globalCompositeOperation = 'source-over'
  ctx.value.strokeStyle = command.color
  ctx.value.lineWidth = command.brush_size
  ctx.value.lineCap = 'round'
  ctx.value.lineJoin = 'round'
  
  ctx.value.beginPath()
  ctx.value.moveTo(command.start_x, command.start_y)
  ctx.value.lineTo(command.end_x, command.end_y)
  ctx.value.stroke()
}

// 远程擦除
const eraseRemote = (command) => {
  ctx.value.globalCompositeOperation = 'destination-out'
  ctx.value.beginPath()
  ctx.value.arc(command.x, command.y, command.size / 2, 0, Math.PI * 2)
  ctx.value.fill()
}

// 发送WebSocket消息
const sendCommand = (command) => {
  if (ws.value && ws.value.readyState === WebSocket.OPEN) {
    ws.value.send(JSON.stringify(command))
  }
}

// 获取鼠标位置
const getMousePos = (e) => {
  const rect = canvas.value.getBoundingClientRect()
  return {
    x: e.clientX - rect.left,
    y: e.clientY - rect.top
  }
}

// 获取触摸位置
const getTouchPos = (e) => {
  const rect = canvas.value.getBoundingClientRect()
  const touch = e.touches[0] || e.changedTouches[0]
  return {
    x: touch.clientX - rect.left,
    y: touch.clientY - rect.top
  }
}

// 移动端触摸事件处理
const handleTouchStart = (e) => {
  e.preventDefault()
  const pos = getTouchPos(e)
  isDrawing.value = true
  lastX.value = pos.x
  lastY.value = pos.y
}

const handleTouchMove = (e) => {
  e.preventDefault()
  if (!isDrawing.value) return
  
  const pos = getTouchPos(e)
  
  if (currentTool.value === 'brush') {
    // 绘制线条
    ctx.value.globalCompositeOperation = 'source-over'
    ctx.value.strokeStyle = currentColor.value
    ctx.value.lineWidth = brushSize.value
    ctx.value.lineCap = 'round'
    ctx.value.lineJoin = 'round'
    
    ctx.value.beginPath()
    ctx.value.moveTo(lastX.value, lastY.value)
    ctx.value.lineTo(pos.x, pos.y)
    ctx.value.stroke()
    
    // 发送线条命令
    const command = {
      type: 'line',
      id: generateUniqueId(),
      start_x: lastX.value,
      start_y: lastY.value,
      end_x: pos.x,
      end_y: pos.y,
      color: currentColor.value,
      brush_size: brushSize.value,
      timestamp: Date.now()
    }
    sendCommand(command)
    
  } else if (currentTool.value === 'eraser') {
    // 擦除
    ctx.value.globalCompositeOperation = 'destination-out'
    ctx.value.beginPath()
    ctx.value.arc(pos.x, pos.y, eraserSize.value / 2, 0, Math.PI * 2)
    ctx.value.fill()
    
    // 发送擦除命令
    const command = {
      type: 'erase',
      id: generateUniqueId(),
      x: pos.x,
      y: pos.y,
      size: eraserSize.value,
      timestamp: Date.now()
    }
    sendCommand(command)
  }
  
  lastX.value = pos.x
  lastY.value = pos.y
}

const handleTouchEnd = (e) => {
  e.preventDefault()
  isDrawing.value = false
}

// 设置canvas尺寸
const setCanvasSize = () => {
  if (!canvas.value) return
  
  const container = canvas.value.parentElement
  if (!container) return
  
  // 获取容器的实际可用空间
  const containerRect = container.getBoundingClientRect()
  const padding = 0 // 移除内边距，让画布完全填满容器
  
  const availableWidth = containerRect.width - padding
  const availableHeight = containerRect.height - padding
  
  // 设置canvas尺寸为容器的完整大小
  canvas.value.width = Math.max(300, availableWidth)
  canvas.value.height = Math.max(200, availableHeight)
  
  // 重新设置背景
  ctx.value.fillStyle = '#FFFFFF'
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height)
}

// 开始绘制
const startDrawing = (e) => {
  isDrawing.value = true
  const pos = getMousePos(e)
  lastX.value = pos.x
  lastY.value = pos.y
}

// 绘制过程
const draw = (e) => {
  if (!isDrawing.value) return
  
  const pos = getMousePos(e)
  
  if (currentTool.value === 'brush') {
    // 绘制线条
    ctx.value.globalCompositeOperation = 'source-over'
    ctx.value.strokeStyle = currentColor.value
    ctx.value.lineWidth = brushSize.value
    ctx.value.lineCap = 'round'
    ctx.value.lineJoin = 'round'
    
    ctx.value.beginPath()
    ctx.value.moveTo(lastX.value, lastY.value)
    ctx.value.lineTo(pos.x, pos.y)
    ctx.value.stroke()
    
    // 发送线条命令
    const command = {
      type: 'line',
      id: generateUniqueId(),
      start_x: lastX.value,
      start_y: lastY.value,
      end_x: pos.x,
      end_y: pos.y,
      color: currentColor.value,
      brush_size: brushSize.value,
      timestamp: Date.now()
    }
    sendCommand(command)
    
  } else if (currentTool.value === 'eraser') {
    // 擦除
    ctx.value.globalCompositeOperation = 'destination-out'
    ctx.value.beginPath()
    ctx.value.arc(pos.x, pos.y, eraserSize.value / 2, 0, Math.PI * 2)
    ctx.value.fill()
    
    // 发送擦除命令
    const command = {
      type: 'erase',
      id: generateUniqueId(),
      x: pos.x,
      y: pos.y,
      size: eraserSize.value,
      timestamp: Date.now()
    }
    sendCommand(command)
  }
  
  lastX.value = pos.x
  lastY.value = pos.y
}

// 停止绘制
const stopDrawing = () => {
  isDrawing.value = false
}

// 清空画布
const clearCanvas = () => {
  if (ctx.value) {
    ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  }
}

// 清空画布并发送命令
const clearCanvasAndSync = () => {
  clearCanvas()
  const command = {
    type: 'clear',
    id: generateUniqueId(),
    timestamp: Date.now()
  }
  sendCommand(command)
}

// 切换工具
const setTool = (tool) => {
  currentTool.value = tool
}

// 设置颜色
const setColor = (color) => {
  currentColor.value = color
  if (currentTool.value !== 'brush') {
    currentTool.value = 'brush'
  }
}

// 组件挂载
onMounted(async () => {
    await nextTick()
    
    // 监听窗口大小变化
    window.addEventListener('resize', () => {
      if (showDrawingBoard.value && canvas.value) {
        setCanvasSize()
      }
    })
  })
  
  // 监听showDrawingBoard变化，初始化canvas
  watch(showDrawingBoard, async (newValue) => {
    if (newValue) {
      await nextTick()
      if (canvas.value) {
        ctx.value = canvas.value.getContext('2d')
        setCanvasSize()
      }
    }
  })

// 组件卸载
onUnmounted(() => {
  if (ws.value) {
    ws.value.close()
  }
  window.removeEventListener('resize', setCanvasSize)
})
</script>

<template>
  <!-- 连接配置页面 -->
  <div v-if="!isConnected && !showDrawingBoard" class="min-h-screen bg-slate-50 flex items-center justify-center p-4">
    <div class="bg-white rounded-2xl shadow-xl border border-slate-200 p-8 w-full max-w-md">
      <div class="text-center mb-8">
       
        <h1 class="text-2xl font-bold text-slate-800 mb-2">协作绘图板</h1>
        <p class="text-slate-600">请输入服务器地址以开始协作绘画</p>
      </div>

      <form @submit.prevent="connectToServer" class="space-y-6">
        <div class="space-y-4">
          <div>
            <label for="serverIp" class="block text-sm font-medium text-slate-700 mb-2">
              服务器IP地址
            </label>
            <input
              id="serverIp"
              v-model="serverConfig.ip"
              type="text"
              placeholder="127.0.0.1"
              class="w-full px-4 py-3 border border-slate-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
              :disabled="isConnecting"
              required
            />
          </div>

          <div>
            <label for="serverPort" class="block text-sm font-medium text-slate-700 mb-2">
              端口号
            </label>
            <input
              id="serverPort"
              v-model="serverConfig.port"
              type="number"
              placeholder="8080"
              min="1"
              max="65535"
              class="w-full px-4 py-3 border border-slate-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
              :disabled="isConnecting"
              required
            />
          </div>
        </div>

        <!-- 错误提示 -->
        <div v-if="connectionError" class="bg-red-50 border border-red-200 rounded-xl p-4">
          <div class="flex items-center">
            <svg class="w-5 h-5 text-red-500 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
            </svg>
            <p class="text-red-700 text-sm">{{ connectionError }}</p>
          </div>
        </div>

        <button
          type="submit"
          :disabled="isConnecting"
          class="w-full bg-blue-500 text-white font-medium py-3 px-4 rounded-xl transition-all duration-200 flex items-center justify-center"
        >
          <svg v-if="isConnecting" class="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          {{ isConnecting ? '连接中...' : '连接服务器' }}
        </button>
      </form>

      <div class="mt-6 pt-6 border-t border-slate-200">
        <p class="text-xs text-slate-500 text-center">
          确保WebSocket服务器正在运行并且可以访问
        </p>
      </div>
    </div>
  </div>

  <!-- 绘画界面 -->
  <div v-else class="min-h-screen bg-slate-50">
    <!-- 顶部导航栏 -->
    <header class="bg-white/80 backdrop-blur-sm border-b border-slate-200 sticky top-0 z-10">
      <div class="max-w-full mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex items-center justify-between h-16">
          <div class="flex items-center space-x-3">
            <div class="w-8 h-8 bg-gradient-to-r from-blue-500 to-purple-600 rounded-lg flex items-center justify-center">
              <svg class="w-5 h-5 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z"></path>
              </svg>
            </div>
            <h1 class="text-xl font-semibold text-slate-800 hidden sm:block">协作绘图板</h1>
          </div>
          
          <div class="flex items-center space-x-2">
            <div class="flex items-center space-x-2 px-3 py-1.5 rounded-full bg-slate-100">
              <div class="w-2 h-2 rounded-full" :class="isConnected ? 'bg-emerald-500' : 'bg-red-500'"></div>
              <span class="text-xs font-medium text-slate-600">
                {{ isConnected ? `已连接 ${serverConfig.ip}:${serverConfig.port}` : '未连接' }}
              </span>
            </div>
            <button
              @click="disconnectAndGoBack"
              class="px-3 py-1.5 text-xs font-medium text-slate-600 hover:text-slate-800 hover:bg-slate-200 rounded-full transition-colors"
            >
              断开连接
            </button>
          </div>
        </div>
      </div>
    </header>

    <!-- 移动端工具栏 -->
    <div class="lg:hidden bg-white/90 backdrop-blur-sm border-b border-slate-200 sticky top-16 z-10">
      <div class="px-4 py-3">
        <div class="space-y-3">
          <!-- 工具选择 -->
          <div class="flex items-center justify-between">
            <div class="flex space-x-1 bg-slate-100 rounded-lg p-1">
              <button
                @click="setTool('brush')"
                :class="[
                  'px-3 py-1.5 rounded-md text-sm font-medium transition-all duration-200',
                  currentTool === 'brush' 
                    ? 'bg-white text-slate-900 shadow-sm' 
                    : 'text-slate-600'
                ]"
              >
                画笔
              </button>
              <button
                @click="setTool('eraser')"
                :class="[
                  'px-3 py-1.5 rounded-md text-sm font-medium transition-all duration-200',
                  currentTool === 'eraser' 
                    ? 'bg-white text-slate-900 shadow-sm' 
                    : 'text-slate-600'
                ]"
              >
                橡皮擦
              </button>
            </div>
            
            <button
              @click="clearCanvasAndSync"
              class="px-3 py-1.5 bg-red-500 hover:bg-red-600 text-white text-sm font-medium rounded-lg transition-colors duration-200"
            >
              清空
            </button>
          </div>

          <!-- 颜色选择 -->
          <div class="flex items-center justify-between">
            <div class="flex space-x-1.5">
              <button
                v-for="color in colors.slice(0, 8)"
                :key="color"
                @click="setColor(color)"
                :style="{ backgroundColor: color }"
                :class="[
                  'w-7 h-7 rounded-lg border-2 transition-all duration-200',
                  currentColor === color ? 'border-slate-400 ring-2 ring-slate-300' : 'border-slate-200'
                ]"
              ></button>
            </div>
            <input
              type="color"
              v-model="currentColor"
              @change="setTool('brush')"
              class="w-7 h-7 rounded-lg border-2 border-slate-200 cursor-pointer"
            />
          </div>

          <!-- 大小调节 -->
          <div class="flex items-center space-x-3" v-if="currentTool === 'brush'">
            <span class="text-sm font-medium text-slate-700 w-16">画笔: {{ brushSize }}</span>
            <input
              type="range"
              v-model="brushSize"
              min="1"
              max="20"
              class="flex-1 accent-blue-500"
            />
          </div>

          <div class="flex items-center space-x-3" v-if="currentTool === 'eraser'">
            <span class="text-sm font-medium text-slate-700 w-16">橡皮: {{ eraserSize }}</span>
            <input
              type="range"
              v-model="eraserSize"
              min="5"
              max="50"
              class="flex-1 accent-blue-500"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- 主要内容区域 -->
    <div class="flex h-[calc(100vh-4rem)] lg:h-[calc(100vh-4rem)]">
      <!-- 桌面端左侧工具栏 -->
      <aside class="hidden lg:block w-80 bg-white/90 backdrop-blur-sm border-r border-slate-200">
        <div class="p-6 space-y-6">
          <!-- 工具选择 -->
          <div class="space-y-3">
            <h3 class="text-sm font-semibold text-slate-800 uppercase tracking-wide">工具</h3>
            <div class="flex space-x-1 bg-slate-100 rounded-lg p-1">
              <button
                @click="setTool('brush')"
                :class="[
                  'flex-1 px-3 py-2 rounded-md text-sm font-medium transition-all duration-200',
                  currentTool === 'brush' 
                    ? 'bg-white text-slate-900 shadow-sm' 
                    : 'text-slate-600 hover:text-slate-900'
                ]"
              >
                画笔
              </button>
              <button
                @click="setTool('eraser')"
                :class="[
                  'flex-1 px-3 py-2 rounded-md text-sm font-medium transition-all duration-200',
                  currentTool === 'eraser' 
                    ? 'bg-white text-slate-900 shadow-sm' 
                    : 'text-slate-600 hover:text-slate-900'
                ]"
              >
                橡皮擦
              </button>
            </div>
          </div>

          <!-- 颜色选择器 -->
          <div class="space-y-3">
            <h3 class="text-sm font-semibold text-slate-800 uppercase tracking-wide">颜色</h3>
            <div class="grid grid-cols-4 gap-2">
              <button
                v-for="color in colors"
                :key="color"
                @click="setColor(color)"
                :style="{ backgroundColor: color }"
                :class="[
                  'w-12 h-12 rounded-xl border-2 transition-all duration-200 hover:scale-105',
                  currentColor === color ? 'border-slate-400 ring-2 ring-slate-300' : 'border-slate-200'
                ]"
              ></button>
            </div>
            <input
              type="color"
              v-model="currentColor"
              @change="setTool('brush')"
              class="w-full h-12 rounded-xl border-2 border-slate-200 cursor-pointer"
            />
          </div>

          <!-- 大小调节 -->
          <div class="space-y-3" v-if="currentTool === 'brush'">
            <h3 class="text-sm font-semibold text-slate-800 uppercase tracking-wide">画笔大小</h3>
            <div class="space-y-2">
              <div class="flex items-center justify-between">
                <span class="text-sm text-slate-600">大小</span>
                <span class="text-sm font-medium text-slate-900">{{ brushSize }}px</span>
              </div>
              <input
                type="range"
                v-model="brushSize"
                min="1"
                max="20"
                class="w-full accent-blue-500"
              />
            </div>
          </div>

          <div class="space-y-3" v-if="currentTool === 'eraser'">
            <h3 class="text-sm font-semibold text-slate-800 uppercase tracking-wide">橡皮擦大小</h3>
            <div class="space-y-2">
              <div class="flex items-center justify-between">
                <span class="text-sm text-slate-600">大小</span>
                <span class="text-sm font-medium text-slate-900">{{ eraserSize }}px</span>
              </div>
              <input
                type="range"
                v-model="eraserSize"
                min="5"
                max="50"
                class="w-full accent-blue-500"
              />
            </div>
          </div>

          <!-- 操作按钮 -->
          <div class="pt-4 border-t border-slate-200">
            <button
              @click="clearCanvasAndSync"
              class="w-full px-4 py-3 bg-red-500 hover:bg-red-600 text-white font-medium rounded-xl transition-colors duration-200 shadow-sm"
            >
              清空画布
            </button>
          </div>
        </div>
      </aside>

      <!-- 画布区域 -->
      <main class="flex-1 flex flex-col overflow-hidden">
        <div class="flex-1 p-4 lg:p-6">
          <div class="h-full bg-white rounded-2xl shadow-lg border border-slate-200 p-4">
            <canvas
              ref="canvas"
              @mousedown="startDrawing"
              @mousemove="draw"
              @mouseup="stopDrawing"
              @mouseleave="stopDrawing"
              @touchstart="handleTouchStart"
              @touchmove="handleTouchMove"
              @touchend="handleTouchEnd"
              class="w-full h-full border border-slate-200 rounded-xl cursor-crosshair"
              style="touch-action: none;"
            ></canvas>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>
