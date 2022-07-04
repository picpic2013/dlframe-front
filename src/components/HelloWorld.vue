<template>
  <div>
    <!-- 数据集 -->
    <div>
      <h2>数据集</h2>
      <div v-if="datasetList.length === 0">没有数据集</div>
      <el-radio-group v-model="datasetName">
        <el-radio v-for="(name, idx) in datasetList" :key="name" :label="name">{{ name }}</el-radio>
      </el-radio-group>
    </div>

    <el-divider />
    
    <!-- 数据集切分器 -->
    <div>
      <h2>数据集切分器</h2>
      <div v-if="splitterList.length === 0">没有数据集切分器</div>
      <el-radio-group v-model="splitterName">
        <el-radio v-for="(name, idx) in splitterList" :key="name" :label="name">{{ name }}</el-radio>
      </el-radio-group>
    </div>

    <el-divider />

    <!-- 模型 -->
    <div>
      <h2>模型</h2>
      <div v-if="modelList.length === 0">没有模型</div>
      <el-radio-group v-model="modelName">
        <el-radio v-for="(name, idx) in modelList" :key="name" :label="name">{{ name }}</el-radio>
      </el-radio-group>
    </div>

    <el-divider />

    <!-- 结果判别器 -->
    <div>
      <h2>结果判别器</h2>
      <div v-if="judgerList.length === 0">没有结果判别器</div>
      <el-radio-group v-model="judgerName">
        <el-radio v-for="(name, idx) in judgerList" :key="name" :label="name">{{ name }}</el-radio>
      </el-radio-group>
    </div>

    <el-divider />

    <el-button @click="clickButton" type="primary">运行</el-button>

    <el-divider />

    <div>
      <h2>运行结果</h2>
      <div class="running-res">
        <div v-for="(content, idx) in runningOutput" :key="idx">{{ content }}</div>
      </div>
    </div>
  </div>

  <el-dialog v-model="showConnectInfoWindow" title="连接地址" :show-close="false">
    <el-form label-width="120px">
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
        datasetList.value = received_msg_data.datasets
        splitterList.value = received_msg_data.splitters
        modelList.value = received_msg_data.models
        judgerList.value = received_msg_data.judgers
      }

      else if (received_msg.type === 'print') {
        runningOutput.value.push(received_msg_data.content as never)
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
    ElMessage.error('连接失败')
    showConnectInfoWindow.value = true
  } 
}

connect()

const datasetName = ref(null)
const datasetList = ref([])

const splitterName = ref(null)
const splitterList = ref([])

const modelName = ref(null)
const modelList = ref([])

const judgerName = ref(null)
const judgerList = ref([])

const isConnectedToServer = ref(false)
const onClickConnect = () => {
  connect()
}

const clickButton = () => {
  if (!isConnectedToServer.value) {
    ElMessage.error('与 server 的连接已断开。请重启 python 服务后刷新页面')
    return
  }
  if (!datasetName.value || !splitterName.value || !modelName.value || !judgerName.value) {
    ElMessage.error('您的选项不完整')
    return
  }
  runningOutput.value = []
  ws?.send(JSON.stringify({
    'type': 'run', 
    'params': {
      'datasetName': datasetName.value, 
      'splitterName': splitterName.value, 
      'modelName': modelName.value, 
      'judgerName': judgerName.value
    }
  }))
}

const runningOutput = ref([])
</script>

<style scoped>
.running-res {
  text-align: left;
  width: 100%;
  font-size: large;
}
</style>
