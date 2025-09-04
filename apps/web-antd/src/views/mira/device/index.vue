<<template>
  <div class="p-4">
    <!-- 页面标题和操作按钮 -->
    <div class="header-section">
      <div>
        <h1 class="page-title">设备管理器</h1>
        <p class="page-description">实时监控和管理连接的设备</p>
      </div>
      <div class="header-actions">
        <Button @click="refreshDevices">
          <span style="margin-right: 4px;">🔄</span>
          刷新设备
        </Button>
        <Button type="primary" @click="connectAllDevices" :loading="connecting">
          <span style="margin-right: 4px;">🔗</span>
          连接所有设备
        </Button>
        <Button danger @click="disconnectAllDevices">
          <span style="margin-right: 4px;">⛓️‍💥</span>
          断开所有连接
        </Button>
      </div>
    </div>

    <!-- 设备统计卡片 -->
    <div class="stats-grid">
      <Card size="small">
        <Statistic
          title="总设备数"
          :value="deviceStats.total"
          :value-style="{ color: '#1890ff' }"
          prefix="📱"
        />
      </Card>
      <Card size="small">
        <Statistic
          title="在线设备"
          :value="deviceStats.online"
          :value-style="{ color: '#52c41a' }"
          prefix="🟢"
        />
      </Card>
      <Card size="small">
        <Statistic
          title="离线设备"
          :value="deviceStats.offline"
          :value-style="{ color: '#f5222d' }"
          prefix="🔴"
        />
      </Card>
      <Card size="small">
        <Statistic
          title="连接中设备"
          :value="deviceStats.connecting"
          :value-style="{ color: '#faad14' }"
          prefix="🟡"
        />
      </Card>
    </div>

    <!-- 设备列表 -->
    <Card title="设备列表" class="device-list-card">
      <Table
        :loading="loading"
        :dataSource="devices"
        :columns="deviceColumns"
        :pagination="{ pageSize: 10 }"
        row-key="id"
      >
        <template #bodyCell="{ column, record }">
          <template v-if="column.key === 'status'">
            <Tag :color="getStatusColor(record.status)">
              {{ getStatusText(record.status) }}
            </Tag>
          </template>
          <template v-else-if="column.key === 'actions'">
            <div class="action-buttons">
              <Button 
                v-if="record.status === 'offline'" 
                size="small" 
                type="primary" 
                @click="connectDevice(record as Device)"
                :loading="record.connecting"
              >
                连接
              </Button>
              <Button 
                v-else 
                size="small" 
                danger 
                @click="disconnectDevice(record as Device)"
              >
                断开
              </Button>
              <Button 
                size="small" 
                @click="sendTestMessage(record as Device)" 
                :disabled="record.status !== 'online'"
              >
                测试
              </Button>
              <Button 
                size="small" 
                @click="viewDeviceDetail(record as Device)"
              >
                详情
              </Button>
            </div>
          </template>
        </template>
      </Table>
    </Card>

    <!-- 设备详情对话框 -->
    <Modal 
      v-model:open="showDetailModal" 
      :title="`设备详情: ${selectedDevice?.name}`"
      width="800px"
    >
      <div v-if="selectedDevice" class="device-detail">
        <Descriptions :column="2" bordered>
          <DescriptionsItem label="设备名称">
            {{ selectedDevice.name }}
          </DescriptionsItem>
          <DescriptionsItem label="设备ID">
            {{ selectedDevice.id }}
          </DescriptionsItem>
          <DescriptionsItem label="IP地址">
            {{ selectedDevice.ip }}
          </DescriptionsItem>
          <DescriptionsItem label="端口">
            {{ selectedDevice.port }}
          </DescriptionsItem>
          <DescriptionsItem label="设备类型">
            {{ selectedDevice.type }}
          </DescriptionsItem>
          <DescriptionsItem label="连接状态">
            <Tag :color="getStatusColor(selectedDevice.status)">
              {{ getStatusText(selectedDevice.status) }}
            </Tag>
          </DescriptionsItem>
          <DescriptionsItem label="最后连接时间">
            {{ formatTime(selectedDevice.lastConnectedAt) }}
          </DescriptionsItem>
          <DescriptionsItem label="设备信息">
            {{ selectedDevice.deviceInfo || '无' }}
          </DescriptionsItem>
        </Descriptions>

        <!-- 消息发送区域 -->
        <div class="message-section">
          <h4>发送消息</h4>
          <div class="message-input-group">
            <Textarea
              v-model:value="testMessage"
              :rows="3"
              placeholder="输入要发送的测试消息..."
            />
            <Button 
              type="primary" 
              @click="sendCustomMessage" 
              :disabled="selectedDevice.status !== 'online'"
              class="send-button"
            >
              发送
            </Button>
          </div>
        </div>

        <!-- 最近消息历史 -->
        <div class="message-history">
          <h4>消息历史</h4>
          <div class="message-list">
            <div 
              v-for="msg in deviceMessages" 
              :key="msg.id" 
              class="message-item"
              :class="msg.type"
            >
              <div class="message-time">{{ formatTime(msg.timestamp) }}</div>
              <div class="message-content">{{ msg.content }}</div>
            </div>
            <div v-if="deviceMessages.length === 0" class="no-messages">
              暂无消息记录
            </div>
          </div>
        </div>
      </div>
    </Modal>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue';
import {
  Button,
  Card,
  Descriptions,
  DescriptionsItem,
  Modal,
  Statistic,
  Table,
  Tag,
  Textarea,
  message
} from 'ant-design-vue';
import miraApiClient from '#/api/mira/client';


defineOptions({ name: 'MiraDeviceManager' });

// 接口定义
interface Device {
  id: string;
  name: string;
  ip: string;
  port: number;
  type: string;
  status: 'online' | 'offline' | 'connecting';
  lastConnectedAt?: string;
  deviceInfo?: string;
  connecting?: boolean;
}

interface DeviceMessage {
  id: string;
  deviceId: string;
  type: 'sent' | 'received';
  content: string;
  timestamp: string;
}

// 响应式数据
const loading = ref(false);
const connecting = ref(false);
const showDetailModal = ref(false);
const selectedDevice = ref<Device | null>(null);
const testMessage = ref('');
const devices = ref<Device[]>([]);
const deviceMessages = ref<DeviceMessage[]>([]);
const wsConnection = ref<WebSocket | null>(null);

// 设备列表表格列定义
const deviceColumns = [
  {
    title: '设备名称',
    dataIndex: 'name',
    key: 'name',
    width: 200,
  },
  {
    title: 'IP地址',
    dataIndex: 'ip',
    key: 'ip',
    width: 150,
  },
  {
    title: '端口',
    dataIndex: 'port',
    key: 'port',
    width: 100,
  },
  {
    title: '设备类型',
    dataIndex: 'type',
    key: 'type',
    width: 120,
  },
  {
    title: '连接状态',
    key: 'status',
    width: 120,
  },
  {
    title: '操作',
    key: 'actions',
    width: 200,
  }
];

// 计算属性
const deviceStats = computed(() => {
  const total = devices.value.length;
  const online = devices.value.filter(d => d.status === 'online').length;
  const offline = devices.value.filter(d => d.status === 'offline').length;
  const connecting = devices.value.filter(d => d.status === 'connecting').length;
  
  return { total, online, offline, connecting };
});

// 方法
const refreshDevices = async () => {
  loading.value = true;
  try {
    const response = await miraApiClient.get('/devices');
    devices.value = response.data || [];
  } catch (error) {
    message.error('获取设备列表失败');
    console.error('Failed to load devices:', error);
  } finally {
    loading.value = false;
  }
};

const connectDevice = async (device: Device) => {
  device.connecting = true;
  try {
    await miraApiClient.post(`/devices/${device.id}/connect`);
    message.success(`正在连接设备 ${device.name}`);
    // 更新设备状态
    device.status = 'connecting';
  } catch (error) {
    message.error(`连接设备失败: ${device.name}`);
    console.error('Failed to connect device:', error);
  } finally {
    device.connecting = false;
  }
};

const disconnectDevice = async (device: Device) => {
  try {
    await miraApiClient.post(`/devices/${device.id}/disconnect`);
    message.success(`已断开设备 ${device.name}`);
    device.status = 'offline';
  } catch (error) {
    message.error(`断开设备失败: ${device.name}`);
    console.error('Failed to disconnect device:', error);
  }
};

const connectAllDevices = async () => {
  connecting.value = true;
  try {
    await miraApiClient.post('/devices/connect-all');
    message.success('正在连接所有设备');
    // 更新所有离线设备状态为连接中
    devices.value.forEach(device => {
      if (device.status === 'offline') {
        device.status = 'connecting';
      }
    });
  } catch (error) {
    message.error('连接所有设备失败');
    console.error('Failed to connect all devices:', error);
  } finally {
    connecting.value = false;
  }
};

const disconnectAllDevices = async () => {
  try {
    await miraApiClient.post('/devices/disconnect-all');
    message.success('已断开所有设备连接');
    // 更新所有设备状态为离线
    devices.value.forEach(device => {
      device.status = 'offline';
    });
  } catch (error) {
    message.error('断开所有设备失败');
    console.error('Failed to disconnect all devices:', error);
  }
};

const sendTestMessage = async (device: Device) => {
  try {
    await miraApiClient.post(`/devices/${device.id}/test`);
    message.success(`测试消息已发送到 ${device.name}`);
  } catch (error) {
    message.error(`发送测试消息失败: ${device.name}`);
    console.error('Failed to send test message:', error);
  }
};

const viewDeviceDetail = async (device: Device) => {
  selectedDevice.value = device;
  showDetailModal.value = true;
  
  // 加载设备消息历史
  try {
    const response = await miraApiClient.get(`/devices/${device.id}/messages`);
    deviceMessages.value = response.data || [];
  } catch (error) {
    console.error('Failed to load device messages:', error);
    deviceMessages.value = [];
  }
};

const sendCustomMessage = async () => {
  if (!selectedDevice.value || !testMessage.value.trim()) {
    message.warning('请输入要发送的消息');
    return;
  }
  
  try {
    await miraApiClient.post(`/devices/${selectedDevice.value.id}/message`, {
      content: testMessage.value
    });
    message.success('消息发送成功');
    testMessage.value = '';
    
    // 添加到消息历史
    deviceMessages.value.unshift({
      id: Date.now().toString(),
      deviceId: selectedDevice.value.id,
      type: 'sent',
      content: testMessage.value,
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    message.error('发送消息失败');
    console.error('Failed to send custom message:', error);
  }
};

// 工具方法
const getStatusColor = (status: string) => {
  switch (status) {
    case 'online': return 'green';
    case 'offline': return 'red';
    case 'connecting': return 'orange';
    default: return 'default';
  }
};

const getStatusText = (status: string) => {
  switch (status) {
    case 'online': return '在线';
    case 'offline': return '离线';
    case 'connecting': return '连接中';
    default: return '未知';
  }
};

const formatTime = (timeString?: string) => {
  if (!timeString) return '无';
  return new Date(timeString).toLocaleString('zh-CN');
};

// 生命周期
onMounted(() => {
  refreshDevices();
});

onUnmounted(() => {
  if (wsConnection.value) {
    wsConnection.value.close();
  }
});
</script>

<style scoped>
.p-4 {
  padding: 1rem;
}

.header-section {
  margin-bottom: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.page-title {
  font-size: 1.5rem;
  line-height: 2rem;
  font-weight: 700;
}

.page-description {
  margin-top: 0.25rem;
  font-size: 0.875rem;
  line-height: 1.25rem;
}

.header-actions {
  display: flex;
  gap: 0.5rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.device-list-card {
  margin-bottom: 1rem;
}

.action-buttons {
  display: flex;
  gap: 0.25rem;
  flex-wrap: wrap;
}

.device-detail {
  max-height: 600px;
  overflow-y: auto;
}

.message-section {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #f0f0f0;
}

.message-input-group {
  display: flex;
  gap: 0.5rem;
  align-items: flex-end;
  margin-top: 0.5rem;
}

.send-button {
  flex-shrink: 0;
}

.message-history {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #f0f0f0;
}

.message-list {
  max-height: 200px;
  overflow-y: auto;
  border: 1px solid #f0f0f0;
  border-radius: 4px;
  padding: 0.5rem;
}

.message-item {
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border-radius: 4px;
}

.message-item.sent {
  text-align: right;
}

.message-time {
  font-size: 0.75rem;
  margin-bottom: 0.25rem;
}

.message-content {
  word-break: break-word;
}

.no-messages {
  text-align: center;
  padding: 1rem;
}

h4 {
  margin-bottom: 0.5rem;
  font-weight: 600;
}
</style>
