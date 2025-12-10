<template>
  <div class="webrtc-container">
    <div class="video-section">
      <div class="video-container">
        <h3>本地视频</h3>
        <video 
          ref="localVideo" 
          autoplay 
          muted 
          playsinline
          class="video-element"
        ></video>
      </div>
      
      <div class="video-container">
        <h3>远程视频</h3>
        <video 
          ref="remoteVideo" 
          autoplay 
          playsinline
          class="video-element"
        ></video>
      </div>
    </div>

    <div class="controls">
      <button @click="startLocalVideo" :disabled="localStreamActive">
        {{ localStreamActive ? '本地视频已开启' : '开启本地视频' }}
      </button>
      
      <button @click="createOffer" :disabled="!localStreamActive">
        创建通话邀请
      </button>
      
      <button @click="createAnswer" :disabled="!localStreamActive || !hasRemoteOffer">
        接受通话邀请
      </button>
      
      <button @click="testNetwork" class="test-button">
        🔍 网络诊断
      </button>
      
      <button 
        @click="restartIceConnection" 
        :disabled="!peerConnection || iceConnectionState === 'connected'"
        class="retry-button"
      >
        🔄 重新连接
      </button>
      
      <button @click="hangup" :disabled="!localStreamActive">
        挂断
      </button>
    </div>

    <div class="signaling-section">
      <h3>信令交换区域</h3>
      <div class="signaling-box">
        <div class="signaling-header">
          <h4>发送给对方的信令 (复制给对方):</h4>
          <button 
            @click="copyLocalSignaling" 
            :disabled="!localSignaling.trim()"
            class="copy-button"
            :class="{ 'copied': localCopied }"
          >
            {{ localCopied ? '已复制!' : '📋 复制' }}
          </button>
        </div>
        <textarea 
          v-model="localSignaling" 
          readonly 
          placeholder="这里会显示需要发送给对方的信令数据"
          class="signaling-textarea"
        ></textarea>
      </div>
      
      <div class="signaling-box">
        <div class="signaling-header">
          <h4>接收对方的信令 (粘贴对方发来的):</h4>
          <button 
            @click="clearRemoteSignaling" 
            :disabled="!remoteSignaling.trim()"
            class="clear-button"
          >
            🗑️ 清空
          </button>
        </div>
        <textarea 
          v-model="remoteSignaling" 
          placeholder="请粘贴对方发来的信令数据"
          class="signaling-textarea"
        ></textarea>
        <button @click="processRemoteSignaling" :disabled="!remoteSignaling.trim()">
          处理信令
        </button>
      </div>
    </div>

    <div class="status">
      <p><strong>连接状态:</strong> {{ connectionState }}</p>
      <p><strong>ICE 状态:</strong> {{ iceConnectionState }}</p>
      <p><strong>收集到的候选数:</strong> {{ iceCandidates.length }}</p>
      <p v-if="retryCount > 0"><strong>重试次数:</strong> {{ retryCount }}/{{ maxRetries }}</p>
      <div v-if="networkDiagnostic" class="diagnostic">
        <h4>🔍 网络诊断结果:</h4>
        <div v-for="(result, server) in networkDiagnostic" :key="server" class="diagnostic-item">
          <span class="server-name">{{ server }}:</span>
          <span :class="['status-badge', result.success ? 'success' : 'failed']">
            {{ result.success ? '✅ 可达' : '❌ 失败' }}
          </span>
          <span v-if="result.error" class="error-msg">{{ result.error }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted } from 'vue'

// 视频元素引用
const localVideo = ref<HTMLVideoElement>()
const remoteVideo = ref<HTMLVideoElement>()

// 状态变量
const localStreamActive = ref(false)
const hasRemoteOffer = ref(false)
const connectionState = ref('未连接')
const iceConnectionState = ref('new')

// 信令数据
const localSignaling = ref('')
const remoteSignaling = ref('')

// 复制状态
const localCopied = ref(false)

// ICE 候选收集
const iceCandidates = ref<RTCIceCandidate[]>([])
const isGatheringComplete = ref(false)

// 网络诊断
const networkDiagnostic = ref<Record<string, { success: boolean; error?: string }> | null>(null)

// 连接重试
const retryCount = ref(0)
const maxRetries = 3

// WebRTC 相关变量
let localStream: MediaStream | null = null
let peerConnection: RTCPeerConnection | null = null

// ICE 服务器配置 - 添加更多 STUN 服务器和免费 TURN 服务器
const iceServers = {
  iceServers: [
    // Google STUN 服务器
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'stun:stun2.l.google.com:19302' },
    { urls: 'stun:stun3.l.google.com:19302' },
    { urls: 'stun:stun4.l.google.com:19302' },
    
    // 其他公共 STUN 服务器
    { urls: 'stun:stun.cloudflare.com:3478' },
    { urls: 'stun:stun.nextcloud.com:443' },
    
    // 免费 TURN 服务器 (OpenRelay)
    {
      urls: 'turn:openrelay.metered.ca:80',
      username: 'openrelayproject',
      credential: 'openrelayproject'
    },
    {
      urls: 'turn:openrelay.metered.ca:443',
      username: 'openrelayproject',
      credential: 'openrelayproject'
    },
    {
      urls: 'turn:openrelay.metered.ca:443?transport=tcp',
      username: 'openrelayproject',
      credential: 'openrelayproject'
    }
  ],
  // 增加连接超时时间
  iceCandidatePoolSize: 10
}

// 开启本地视频
const startLocalVideo = async () => {
  try {
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true
    })
    
    if (localVideo.value) {
      localVideo.value.srcObject = localStream
    }
    
    localStreamActive.value = true
    connectionState.value = '本地视频已开启'
  } catch (error) {
    console.error('获取本地媒体流失败:', error)
    alert('无法访问摄像头和麦克风，请检查权限设置')
  }
}

// 创建 PeerConnection
const createPeerConnection = () => {
  peerConnection = new RTCPeerConnection(iceServers)
  
  // 添加本地流到连接
  if (localStream) {
    localStream.getTracks().forEach(track => {
      peerConnection!.addTrack(track, localStream!)
    })
  }
  
  // 处理远程流
  peerConnection.ontrack = (event) => {
    if (remoteVideo.value && event.streams[0]) {
      remoteVideo.value.srcObject = event.streams[0]
    }
  }
  
  // 监听连接状态
  peerConnection.onconnectionstatechange = () => {
    connectionState.value = peerConnection!.connectionState
  }
  
  peerConnection.oniceconnectionstatechange = () => {
    iceConnectionState.value = peerConnection!.iceConnectionState
    console.log('ICE 连接状态变化:', peerConnection!.iceConnectionState)
    
    // 根据 ICE 状态更新连接状态
    switch (peerConnection!.iceConnectionState) {
      case 'checking':
        connectionState.value = '正在检查网络连接...'
        break
      case 'connected':
        connectionState.value = '连接成功！'
        retryCount.value = 0 // 重置重试计数
        break
      case 'completed':
        connectionState.value = '连接已建立'
        retryCount.value = 0 // 重置重试计数
        break
      case 'failed':
        console.error('ICE 连接失败，可能需要 TURN 服务器')
        handleConnectionFailure()
        break
      case 'disconnected':
        connectionState.value = '连接已断开'
        // 短暂断开可能会自动恢复，等待一段时间
        setTimeout(() => {
          if (peerConnection && peerConnection.iceConnectionState === 'disconnected') {
            console.log('连接断开超时，尝试重新连接')
            handleConnectionFailure()
          }
        }, 5000)
        break
      case 'closed':
        connectionState.value = '连接已关闭'
        break
    }
  }
  
  // 收集 ICE 候选
  peerConnection.onicecandidate = (event) => {
    if (event.candidate) {
      iceCandidates.value.push(event.candidate)
      console.log('新的 ICE 候选:', {
        type: event.candidate.type,
        protocol: event.candidate.protocol,
        address: event.candidate.address,
        port: event.candidate.port,
        priority: event.candidate.priority
      })
      
      // 实时更新信令数据，包含所有 ICE 候选
      updateSignalingWithCandidates()
    } else {
      // ICE 候选收集完成
      isGatheringComplete.value = true
      console.log('ICE 候选收集完成，总共收集到', iceCandidates.value.length, '个候选')
      updateSignalingWithCandidates()
    }
  }
  
  // ICE 收集状态变化
  peerConnection.onicegatheringstatechange = () => {
    console.log('ICE 收集状态:', peerConnection!.iceGatheringState)
    if (peerConnection!.iceGatheringState === 'complete') {
      isGatheringComplete.value = true
      updateSignalingWithCandidates()
    }
  }
}

// 更新信令数据，包含 ICE 候选
const updateSignalingWithCandidates = () => {
  if (!peerConnection || !peerConnection.localDescription) return
  
  const signaling = {
    type: peerConnection.localDescription.type,
    sdp: peerConnection.localDescription,
    candidates: iceCandidates.value,
    gatheringComplete: isGatheringComplete.value
  }
  
  localSignaling.value = JSON.stringify(signaling, null, 2)
}

// 创建通话邀请 (Offer)
const createOffer = async () => {
  if (!peerConnection) {
    createPeerConnection()
  }
  
  // 重置 ICE 候选
  iceCandidates.value = []
  isGatheringComplete.value = false
  
  try {
    const offer = await peerConnection!.createOffer()
    await peerConnection!.setLocalDescription(offer)
    
    // 初始信令（不包含 ICE 候选）
    localSignaling.value = JSON.stringify({
      type: 'offer',
      sdp: offer,
      candidates: [],
      gatheringComplete: false
    }, null, 2)
    
    connectionState.value = '正在收集网络信息...'
    
    // 等待 ICE 候选收集完成或超时 - 延长到15秒
    setTimeout(() => {
      if (!isGatheringComplete.value) {
        console.log('Offer ICE 候选收集超时，使用当前候选:', iceCandidates.value.length, '个')
        isGatheringComplete.value = true
        updateSignalingWithCandidates()
        connectionState.value = `等待对方响应... (已收集 ${iceCandidates.value.length} 个网络候选)`
      }
    }, 15000) // 15秒超时
    
  } catch (error) {
    console.error('创建 Offer 失败:', error)
  }
}

// 创建应答 (Answer)
const createAnswer = async () => {
  if (!peerConnection) {
    createPeerConnection()
  }
  
  // 重置 ICE 候选
  iceCandidates.value = []
  isGatheringComplete.value = false
  
  try {
    const answer = await peerConnection!.createAnswer()
    await peerConnection!.setLocalDescription(answer)
    
    // 初始信令（不包含 ICE 候选）
    localSignaling.value = JSON.stringify({
      type: 'answer',
      sdp: answer,
      candidates: [],
      gatheringComplete: false
    }, null, 2)
    
    connectionState.value = '正在收集网络信息...'
    
    // 等待 ICE 候选收集完成或超时 - 延长到15秒
    setTimeout(() => {
      if (!isGatheringComplete.value) {
        console.log('Answer ICE 候选收集超时，使用当前候选:', iceCandidates.value.length, '个')
        isGatheringComplete.value = true
        updateSignalingWithCandidates()
        connectionState.value = `已发送应答，等待连接建立... (已收集 ${iceCandidates.value.length} 个网络候选)`
      }
    }, 15000) // 15秒超时
    
  } catch (error) {
    console.error('创建 Answer 失败:', error)
  }
}

// 复制本地信令到剪贴板
const copyLocalSignaling = async () => {
  try {
    await navigator.clipboard.writeText(localSignaling.value)
    localCopied.value = true
    setTimeout(() => {
      localCopied.value = false
    }, 2000)
  } catch (error) {
    console.error('复制失败:', error)
    // 降级方案：选择文本
    const textarea = document.querySelector('.signaling-textarea[readonly]') as HTMLTextAreaElement
    if (textarea) {
      textarea.select()
      document.execCommand('copy')
      localCopied.value = true
      setTimeout(() => {
        localCopied.value = false
      }, 2000)
    }
  }
}

// 清空远程信令
const clearRemoteSignaling = () => {
  remoteSignaling.value = ''
}

// 网络诊断功能
const testNetwork = async () => {
  networkDiagnostic.value = null
  connectionState.value = '正在进行网络诊断...'
  
  const testServers = [
    { name: 'Google STUN', url: 'stun:stun.l.google.com:19302' },
    { name: 'Cloudflare STUN', url: 'stun:stun.cloudflare.com:3478' },
    { name: 'OpenRelay TURN', url: 'turn:openrelay.metered.ca:80' }
  ]
  
  const results: Record<string, { success: boolean; error?: string }> = {}
  
  for (const server of testServers) {
    try {
      const testPC = new RTCPeerConnection({
        iceServers: [
          server.name.includes('TURN') 
            ? { urls: server.url, username: 'openrelayproject', credential: 'openrelayproject' }
            : { urls: server.url }
        ]
      })
      
      // 创建一个数据通道来触发 ICE 收集
      testPC.createDataChannel('test')
      
      const offer = await testPC.createOffer()
      await testPC.setLocalDescription(offer)
      
      // 等待 ICE 候选
      const candidatePromise = new Promise<boolean>((resolve) => {
        let hasCandidate = false
        testPC.onicecandidate = (event) => {
          if (event.candidate) {
            hasCandidate = true
            console.log(`${server.name} 产生候选:`, event.candidate.type)
          } else if (hasCandidate) {
            resolve(true)
          }
        }
        
        // 3秒超时
        setTimeout(() => resolve(hasCandidate), 3000)
      })
      
      const success = await candidatePromise
      results[server.name] = { success }
      testPC.close()
      
    } catch (error) {
      results[server.name] = { 
        success: false, 
        error: error instanceof Error ? error.message : '未知错误' 
      }
    }
  }
  
  networkDiagnostic.value = results
  connectionState.value = '网络诊断完成'
}

// 处理连接失败
const handleConnectionFailure = () => {
  retryCount.value++
  
  if (retryCount.value <= maxRetries) {
    connectionState.value = `连接失败，正在重试 (${retryCount.value}/${maxRetries})...`
    console.log(`连接失败，开始第 ${retryCount.value} 次重试`)
    
    // 等待2秒后重试
    setTimeout(() => {
      restartIceConnection()
    }, 2000)
  } else {
    connectionState.value = '连接失败 - 已达到最大重试次数'
    console.error('连接失败，已达到最大重试次数')
    
    // 显示详细的错误信息和建议
    alert(`连接失败！可能的原因：
1. 网络环境过于复杂，需要更强的 TURN 服务器
2. 防火墙阻止了 WebRTC 连接
3. 双方网络都在严格的 NAT 后面

建议：
- 尝试使用不同的网络环境
- 检查防火墙设置
- 点击"网络诊断"查看详细信息`)
  }
}

// 重启 ICE 连接
const restartIceConnection = async () => {
  if (!peerConnection) return
  
  try {
    // ICE 重启
    const offer = await peerConnection.createOffer({ iceRestart: true })
    await peerConnection.setLocalDescription(offer)
    
    // 重置候选收集
    iceCandidates.value = []
    isGatheringComplete.value = false
    
    // 更新信令
    localSignaling.value = JSON.stringify({
      type: 'offer',
      sdp: offer,
      candidates: [],
      gatheringComplete: false,
      iceRestart: true
    }, null, 2)
    
    connectionState.value = '正在重新收集网络信息...'
    
  } catch (error) {
    console.error('ICE 重启失败:', error)
    connectionState.value = '重试失败'
  }
}

// 处理远程信令
const processRemoteSignaling = async () => {
  try {
    const signaling = JSON.parse(remoteSignaling.value)
    
    if (!peerConnection) {
      createPeerConnection()
    }
    
    if (signaling.type === 'offer') {
      await peerConnection!.setRemoteDescription(signaling.sdp)
      
      // 添加远程 ICE 候选
      if (signaling.candidates && Array.isArray(signaling.candidates)) {
        for (const candidate of signaling.candidates) {
          try {
            await peerConnection!.addIceCandidate(candidate)
            console.log('添加远程 ICE 候选成功:', candidate)
          } catch (error) {
            console.error('添加远程 ICE 候选失败:', error)
          }
        }
      }
      
      hasRemoteOffer.value = true
      connectionState.value = '收到通话邀请，可以接受'
      
    } else if (signaling.type === 'answer') {
      await peerConnection!.setRemoteDescription(signaling.sdp)
      
      // 添加远程 ICE 候选
      if (signaling.candidates && Array.isArray(signaling.candidates)) {
        for (const candidate of signaling.candidates) {
          try {
            await peerConnection!.addIceCandidate(candidate)
            console.log('添加远程 ICE 候选成功:', candidate)
          } catch (error) {
            console.error('添加远程 ICE 候选失败:', error)
          }
        }
      }
      
      connectionState.value = '正在建立连接...'
      
    } else if (signaling.type === 'ice-candidate') {
      // 兼容单个 ICE 候选格式
      await peerConnection!.addIceCandidate(signaling.candidate)
    }
    
    remoteSignaling.value = ''
  } catch (error) {
    console.error('处理远程信令失败:', error)
    alert('信令格式错误，请检查粘贴的内容')
  }
}

// 挂断通话
const hangup = () => {
  if (peerConnection) {
    peerConnection.close()
    peerConnection = null
  }
  
  if (localStream) {
    localStream.getTracks().forEach(track => track.stop())
    localStream = null
  }
  
  if (localVideo.value) {
    localVideo.value.srcObject = null
  }
  
  if (remoteVideo.value) {
    remoteVideo.value.srcObject = null
  }
  
  localStreamActive.value = false
  hasRemoteOffer.value = false
  connectionState.value = '已断开连接'
  iceConnectionState.value = 'closed'
  localSignaling.value = ''
  remoteSignaling.value = ''
  localCopied.value = false
  iceCandidates.value = []
  isGatheringComplete.value = false
  retryCount.value = 0
  networkDiagnostic.value = null
}

// 组件卸载时清理资源
onUnmounted(() => {
  hangup()
})
</script>

<style scoped>
.webrtc-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.video-section {
  display: flex;
  gap: 20px;
  margin-bottom: 30px;
  justify-content: center;
  flex-wrap: wrap;
}

.video-container {
  text-align: center;
}

.video-container h3 {
  margin-bottom: 10px;
  color: #333;
}

.video-element {
  width: 300px;
  height: 200px;
  background-color: #000;
  border: 2px solid #ddd;
  border-radius: 8px;
}

.controls {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin-bottom: 30px;
  flex-wrap: wrap;
}

.controls button {
  padding: 10px 20px;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
}

.controls button:hover:not(:disabled) {
  background-color: #369870;
}

.controls button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.test-button {
  background-color: #17a2b8 !important;
}

.test-button:hover:not(:disabled) {
  background-color: #138496 !important;
}

.retry-button {
  background-color: #fd7e14 !important;
}

.retry-button:hover:not(:disabled) {
  background-color: #e8690b !important;
}

.signaling-section {
  margin-bottom: 30px;
}

.signaling-section h3 {
  color: #333;
  margin-bottom: 20px;
}

.signaling-box {
  margin-bottom: 20px;
  text-align: left;
}

.signaling-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.signaling-box h4 {
  color: #666;
  margin: 0;
}

.copy-button, .clear-button {
  padding: 6px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.copy-button {
  background-color: #42b883;
  color: white;
}

.copy-button:hover:not(:disabled) {
  background-color: #369870;
}

.copy-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.copy-button.copied {
  background-color: #28a745;
}

.clear-button {
  background-color: #dc3545;
  color: white;
}

.clear-button:hover:not(:disabled) {
  background-color: #c82333;
}

.clear-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.signaling-textarea {
  width: 100%;
  height: 120px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-family: monospace;
  font-size: 12px;
  resize: vertical;
}

.signaling-box button {
  margin-top: 10px;
  padding: 8px 16px;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.signaling-box button:hover:not(:disabled) {
  background-color: #369870;
}

.signaling-box button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.status {
  background-color: #f5f5f5;
  padding: 15px;
  border-radius: 5px;
  text-align: left;
}

.status p {
  margin: 5px 0;
  color: #333;
}

.diagnostic {
  margin-top: 15px;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 5px;
  border-left: 4px solid #17a2b8;
}

.diagnostic h4 {
  margin: 0 0 10px 0;
  color: #17a2b8;
}

.diagnostic-item {
  display: flex;
  align-items: center;
  margin: 5px 0;
  gap: 10px;
}

.server-name {
  font-weight: bold;
  min-width: 120px;
}

.status-badge {
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: bold;
}

.status-badge.success {
  background-color: #d4edda;
  color: #155724;
}

.status-badge.failed {
  background-color: #f8d7da;
  color: #721c24;
}

.error-msg {
  font-size: 12px;
  color: #6c757d;
  font-style: italic;
}

@media (max-width: 768px) {
  .video-section {
    flex-direction: column;
    align-items: center;
  }
  
  .video-element {
    width: 250px;
    height: 167px;
  }
  
  .controls {
    flex-direction: column;
    align-items: center;
  }
  
  .controls button {
    width: 200px;
  }
}
</style>