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

// WebRTC 相关变量
let localStream: MediaStream | null = null
let peerConnection: RTCPeerConnection | null = null

// ICE 服务器配置
const iceServers = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' }
  ]
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
  }
  
  // 收集 ICE 候选
  peerConnection.onicecandidate = (event) => {
    if (event.candidate) {
      iceCandidates.value.push(event.candidate)
      console.log('新的 ICE 候选:', event.candidate)
      
      // 实时更新信令数据，包含所有 ICE 候选
      updateSignalingWithCandidates()
    } else {
      // ICE 候选收集完成
      isGatheringComplete.value = true
      console.log('ICE 候选收集完成')
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
    
    // 等待 ICE 候选收集完成或超时
    setTimeout(() => {
      if (!isGatheringComplete.value) {
        console.log('ICE 候选收集超时，使用当前候选')
        isGatheringComplete.value = true
        updateSignalingWithCandidates()
      }
    }, 5000) // 5秒超时
    
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
    
    // 等待 ICE 候选收集完成或超时
    setTimeout(() => {
      if (!isGatheringComplete.value) {
        console.log('ICE 候选收集超时，使用当前候选')
        isGatheringComplete.value = true
        updateSignalingWithCandidates()
      }
    }, 5000) // 5秒超时
    
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