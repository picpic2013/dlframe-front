<template>
  <div>
    <div v-for="(value, key) in configDict" :key="key">
      <h2>{{ key }}</h2>
      <div v-if="value.length === 0">没有 {{ key }}</div>
      <el-radio-group v-model="configValue[key]">
        <el-radio v-for="(name, idx) in value" :key="idx" :label="name">{{ name }}</el-radio>
      </el-radio-group>
    </div>

    <el-divider />

    <el-button @click="clickButton" type="primary">运行</el-button>

    <el-divider />

    <div>
      <h2>运行结果</h2>
      <div class="running-res">
        <div v-for="(content, idx) in runningOutput" :key="idx">
          <p v-if="content.type == 'string'">{{ content.content }}</p>
          <img v-if="content.type == 'image'" :src="'data:image/jpeg;base64,'+content.content" />
        </div>
      </div>
    </div>
  </div>

  <el-dialog v-model="showConnectInfoWindow" title="连接地址" :show-close="false">
    <el-form label-width="40px">
      <el-form-item label="Url">
        <el-input v-model="connectUrl" />
      </el-form-item>
      <el-form-item label="Port">
        <el-input v-model="connectPort" />
      </el-form-item>
    </el-form>
    <el-button type="primary" @click="onClickConnect">连接</el-button>
  </el-dialog>
</template>

<script setup lang="ts">
import { ElMessage } from 'element-plus'
import { ref } from 'vue'

const connectUrl = ref('localhost')
const connectPort = ref('8765')
const showConnectInfoWindow = ref(false)

interface DlFrameConfigInterface {
  [key: string]: string;
}
interface DlFrameInspectionInterface {
  [key: string]: Array<string>;
}

let ws: WebSocket | null = null
const connect = () => {
  ws = new WebSocket('ws://' + connectUrl.value + ':' + connectPort.value)
  ws.onopen = () => {
    isConnectedToServer.value = true
    showConnectInfoWindow.value = false
    ElMessage({
      message: '连接成功',
      type: 'success',
    })
    ws?.send(JSON.stringify({
      'type': 'overview', 
      'params': {}
    }))
  }
  ws.onmessage = (evt) => {
    var received_msg = JSON.parse(evt.data);
    // console.log(received_msg);
    
    if (received_msg.status === 200) {
      const received_msg_data = received_msg.data
      if (received_msg.type === 'overview') {
        // console.log(received_msg.data)
        configDict.value = received_msg.data
        
        const tmpDict: DlFrameConfigInterface = {}
        for (let i in configDict.value) {
          tmpDict[i] = ''
        }
        configValue.value = tmpDict
        runningOutput.value = []
      }

      else if (received_msg.type === 'print') {
        runningOutput.value.push({
          'type': 'string', 
          'content': received_msg_data.content
        })
      }

      else if (received_msg.type === 'imshow') {
        runningOutput.value.push({
          'type': 'image', 
          'content': received_msg_data.content
        })
      }
    } else {
      console.error(received_msg.data);
    }
  }
  ws.onclose = () => {
    isConnectedToServer.value = false
    showConnectInfoWindow.value = true
    ElMessage.error('连接已断开')
  }
  ws.onerror = () => {
    ElMessage.error('连接失败(地址错误 / 协议错误 / 服务器错误)')
    showConnectInfoWindow.value = true
  } 
}

connect()

const configDict = ref<DlFrameInspectionInterface>({})
const configValue = ref<DlFrameConfigInterface>({})

const isConnectedToServer = ref(false)
const onClickConnect = () => {
  connect()
}

const clickButton = () => {
  if (!isConnectedToServer.value) {
    ElMessage.error('与 server 的连接已断开。请重启 python 服务后刷新页面')
    return
  }
  for (let k in configDict.value) {
    if (configDict.value[k].length === 0) {
      ElMessage.error('没有 ' + k)
      return
    }
  }
  for (let k in configValue.value) {
    if (configValue.value[k] == '') {
      ElMessage.error('您的选项不完整')
      return
    }
  }
  runningOutput.value = []
  ws?.send(JSON.stringify({
    'type': 'run', 
    'params': configValue.value
  }))
}

interface RunningOutputInterface {
  [key: string]: string;
}
const runningOutput = ref<Array<RunningOutputInterface>>([])
</script>

<style scoped>
.running-res {
  text-align: left;
  width: 100%;
  font-size: large;
}
</style>
