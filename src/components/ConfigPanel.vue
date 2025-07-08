<template>
  <div class="bg-white">
    <div class="space-y-6">
      <div class="hidden">
        <label for="ota-file" class="block text-sm font-medium text-gray-700 mb-2">OTA 固件文件：</label>
        <input type="file" id="ota-file" @change="handleFileChange" accept=".bin, .hex"
               class="block w-full text-sm text-gray-500
                      file:mr-4 file:py-2 file:px-4
                      file:rounded-md file:border-0
                      file:text-sm file:font-semibold
                      file:bg-blue-50 file:text-blue-700
                      hover:file:bg-blue-100" />
        <p v-if="selectedFile" class="mt-2 text-sm text-gray-600">已选择文件: {{ selectedFile.name }}</p>
        <button @click="sendOtaCommand"
                :disabled="!selectedFile"
                class="mt-3 w-full px-4 py-2 bg-green-500 text-white rounded-md hover:bg-green-600 disabled:opacity-50">
          OTA 在线升级
        </button>
      </div>

      <hr class="border-gray-200 hidden" />

      <button @click="sendRestoreDefaultCommand"
              class="w-full px-4 py-2 bg-yellow-500 text-white rounded-md hover:bg-yellow-600">
        恢复到默认参数
      </button>

      <button @click="sendSaveParamsCommand"
              class="w-full px-4 py-2 bg-indigo-500 text-white rounded-md hover:bg-indigo-600">
        将所有参数保存到 Flash
      </button>

      <button @click="sendGetVersionCommand"
              class="w-full px-4 py-2 bg-indigo-500 text-white rounded-md hover:bg-indigo-600">
        获取软件版本号
      </button>

      <hr class="border-gray-200" />

      <div v-for="param in parameters" :key="param.key" class="grid grid-cols-3 gap-2 items-center border p-2 rounded-md bg-gray-50">
        <label class="col-span-3 text-sm font-medium text-gray-700 mb-1">{{ param.label }}</label>

        <button @click="readParameter(param)" class="px-3 py-1 bg-gray-200 text-gray-800 rounded-md hover:bg-gray-300 text-sm">读取</button>

        <div class="col-span-1">
          <input
              v-if="param.type === 'number'"
              type="number"
              v-model.number="param.value"
              :placeholder="`默认: ${param.default}`"
              class="w-full p-1 border border-gray-300 rounded-md text-sm"
          />
          <select
              v-else-if="param.type === 'select'"
              v-model="param.value"
              class="w-full p-1 border border-gray-300 rounded-md text-sm"
          >
            <option v-for="option in param.options" :key="option.value" :value="option.value">{{ option.label }}</option>
          </select>
        </div>

        <button @click="setParameter(param)" class="px-3 py-1 bg-blue-500 text-white rounded-md hover:bg-blue-600 text-sm">设置</button>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, defineEmits, watch, defineProps } from 'vue';

const emit = defineEmits(['send-serial-data', 'log-message', 'show-toast']); // 添加 show-toast 事件

const selectedFile = ref(null);

const parameters = ref([
  {
    key: 'motionThreshold1m',
    label: '1 米内运动检测阈值',
    type: 'number',
    value: null,
    default: 200,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x03, 0x36],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x03],
    receiveCmd: [0x02, 0x80, 0x03],
    dataLength: 2
  },
  {
    key: 'motionThresholdBeyond1m',
    label: '1 米外运动检测阈值',
    type: 'number',
    value: null,
    default: 120,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x04, 0x37],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x04],
    receiveCmd: [0x02, 0x80, 0x04],
    dataLength: 2
  },
  {
    key: 'presenceThreshold1m',
    label: '1 米内存在检测阈值',
    type: 'number',
    value: null,
    default: 300,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x09, 0x3C],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x09],
    receiveCmd: [0x02, 0x80, 0x09],
    dataLength: 2
  },
  {
    key: 'presenceThresholdBeyond1m',
    label: '1 米外存在检测阈值',
    type: 'number',
    value: null,
    default: 300,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x0A, 0x3D],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x0A],
    receiveCmd: [0x02, 0x80, 0x0A],
    dataLength: 2
  },
  {
    key: 'minDetectionDistance',
    label: '最小检测距离（cm）',
    type: 'number',
    value: null,
    default: 10,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x0C, 0x3F],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x0C],
    receiveCmd: [0x02, 0x80, 0x0C],
    dataLength: 2
  },
  {
    key: 'maxMotionDetectionDistance',
    label: '运动最大检测距离（cm）',
    type: 'number',
    value: null,
    default: 600,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x0D, 0x40],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x0D],
    receiveCmd: [0x02, 0x80, 0x0D],
    dataLength: 2
  },
  {
    key: 'maxPresenceDetectionDistance',
    label: '存在最大检测距离（cm）',
    type: 'number',
    value: null,
    default: 450,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x0E, 0x41],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x0E],
    receiveCmd: [0x02, 0x80, 0x0E],
    dataLength: 2
  },
  {
    key: 'voOutputHoldTime',
    label: 'VO输出电平维持时间（ms）',
    type: 'number',
    value: null,
    default: 20000,
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x14, 0x47],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x06, 0x01, 0x80, 0x14],
    receiveCmd: [0x02, 0x80, 0x14],
    dataLength: 2
  },
  {
    key: 'voIndicationMode',
    label: 'VO指示模式',
    type: 'select',
    value: null,
    default: 0x00,
    options: [{ label: '高电平', value: 0x00 }, { label: '低电平', value: 0x01 }],
    readCmd: [0x55, 0x5A, 0x00, 0x04, 0x00, 0x80, 0x15, 0x48],
    setCmdPrefix: [0x55, 0x5A, 0x00, 0x05, 0x01, 0x80, 0x15],
    receiveCmd: [0x02, 0x80, 0x15],
    dataLength: 1
  },
]);

const props = defineProps({
  parameterResponse: Object
});

watch(() => props.parameterResponse, (newResponse) => {
  if (newResponse) {
    emit('log-message', { type: 'debug', text: `ConfigPanel 收到参数响应：功能码: 0x${newResponse.functionCode?.toString(16)}, 命令码1: 0x${newResponse.commandCode1?.toString(16)}, 命令码2: 0x${newResponse.commandCode2?.toString(16)}, 原始数据: [${Array.from(newResponse.data || []).map(b => b.toString(16).padStart(2, '0')).join(' ')}]` });

    const matchedParam = parameters.value.find(p =>
        p.receiveCmd[0] === newResponse.functionCode &&
        p.receiveCmd[1] === newResponse.commandCode1 &&
        p.receiveCmd[2] === newResponse.commandCode2
    );

    if (matchedParam) {
      if (newResponse.data && newResponse.data.length > 0) {
        let parsedValue = null;
        if (matchedParam.type === 'number' && matchedParam.dataLength === 2) {
          if (newResponse.data.length >= 2) {
            parsedValue = (newResponse.data[0] << 8) | newResponse.data[1];
            emit('log-message', {
              type: 'info',
              text: `${matchedParam.label} 读取原始值: 0x${newResponse.data[0].toString(16).padStart(2, '0')} 0x${newResponse.data[1].toString(16).padStart(2, '0')}, 解析为十进制: ${parsedValue}`
            });
            matchedParam.value = parsedValue;
            emit('show-toast', 'success', `${matchedParam.label} 读取成功：${parsedValue}`);
          } else {
            emit('log-message', {
              type: 'error',
              text: `${matchedParam.label} 读取数据长度不足，预期2字节，实际 ${newResponse.data.length} 字节。原始数据: [${Array.from(newResponse.data).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`
            });
            emit('show-toast', 'error', `${matchedParam.label} 读取失败：数据长度不足`);
          }
        } else if (matchedParam.type === 'select' && matchedParam.dataLength === 1) {
          if (newResponse.data.length >= 1) {
            parsedValue = newResponse.data[0];
            const selectedOption = matchedParam.options.find(opt => opt.value === parsedValue);
            emit('log-message', {
              type: 'info',
              text: `${matchedParam.label} 读取原始值: 0x${newResponse.data[0].toString(16).padStart(2, '0')}, 解析为: ${selectedOption ? selectedOption.label : parsedValue} (0x${parsedValue.toString(16)})`
            });
            matchedParam.value = parsedValue;
            emit('show-toast', 'success', `${matchedParam.label} 读取成功：${selectedOption ? selectedOption.label : '未知值'}`);
          } else {
            emit('log-message', {
              type: 'error',
              text: `${matchedParam.label} 读取数据长度不足，预期1字节，实际 ${newResponse.data.length} 字节。原始数据: [${Array.from(newResponse.data).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`
            });
            emit('show-toast', 'error', `${matchedParam.label} 读取失败：数据长度不足`);
          }
        } else {
          emit('log-message', {
            type: 'warn',
            text: `${matchedParam.label} 读取响应数据格式不匹配 (类型: ${matchedParam.type}, 预期长度: ${matchedParam.dataLength}, 实际数据: [${Array.from(newResponse.data).map(b => b.toString(16).padStart(2, '0')).join(' ')}])。`
          });
          emit('show-toast', 'warning', `${matchedParam.label} 读取失败：数据格式不匹配`);
        }
      } else {
        emit('log-message', {
          type: 'warn',
          text: `${matchedParam.label} 读取响应数据为空或格式错误。接收到的响应: ${JSON.stringify(newResponse)}`
        });
        emit('show-toast', 'error', `${matchedParam.label} 读取失败：未接收到数据`);
      }
    } else {
      emit('log-message', {
        type: 'warn',
        text: `ConfigPanel 未找到匹配的参数定义，无法解析响应。接收到的响应: 功能码: 0x${newResponse.functionCode?.toString(16)}, 命令码1: 0x${newResponse.commandCode1?.toString(16)}, 命令码2: 0x${newResponse.commandCode2?.toString(16)}。`
      });
    }
  }
}, {deep: true});


const handleFileChange = (event) => {
  const file = event.target.files[0];
  if (file) {
    selectedFile.value = file;
  } else {
    selectedFile.value = null;
  }
};

const calculateChecksum = (arr) => {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum = (sum + arr[i]) & 0xFF;
  }
  return sum;
};

const sendOtaCommand = () => {
  const dataToSend = new Uint8Array([0x55, 0x5A, 0x00, 0x04, 0x01, 0x20, 0x01, 0xD5]);
  emit('send-serial-data', dataToSend);
  emit('log-message', {type: 'info', text: '发送 OTA 在线升级命令'});
};

const sendRestoreDefaultCommand = () => {
  const dataToSend = new Uint8Array([0x55, 0x5A, 0x00, 0x04, 0x01, 0x20, 0x02, 0xD6]);
  emit('send-serial-data', dataToSend);
  emit('log-message', {type: 'info', text: '发送恢复到默认参数命令'});
};

const sendSaveParamsCommand = () => {
  const dataToSend = new Uint8Array([0x55, 0x5A, 0x00, 0x04, 0x01, 0x20, 0x04, 0xD8]);
  emit('send-serial-data', dataToSend);
  emit('log-message', {type: 'info', text: '发送保存所有参数到 Flash 命令'});
};

const sendGetVersionCommand = () => {
  const dataToSend = new Uint8Array([0x55, 0x5A, 0x00, 0x04, 0x00, 0x00, 0x01, 0xB4]);
  emit('send-serial-data', dataToSend);
  emit('log-message', {type: 'info', text: '发送获取软件版本命令'});
};

const readParameter = (param) => {
  const dataToSend = new Uint8Array(param.readCmd);
  emit('send-serial-data', dataToSend);
  emit('log-message', {
    type: 'info',
    text: `发送读取 ${param.label} 命令。发送数据: [${Array.from(dataToSend).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`
  });
};

const setParameter = (param) => {
  if (param.value === null || param.value === undefined || (typeof param.value === 'number' && isNaN(param.value))) {
    emit('log-message', {type: 'warn', text: `请为 ${param.label} 输入有效值`});
    emit('show-toast', 'warning', `请为 ${param.label} 输入有效值！`);
    return;
  }

  if (param.key === 'voIndicationMode' && ![0x00, 0x01].includes(param.value)) {
    emit('log-message', {
      type: 'error',
      text: `VO指示模式的值必须是 0x00 (高电平) 或 0x01 (低电平)。当前值: 0x${param.value.toString(16)}`
    });
    emit('show-toast', 'error', `VO指示模式值无效，请选择高电平或低电平！`);
    return;
  }

  let dataBytes = [];
  if (param.type === 'number' && param.dataLength === 2) {
    if (param.value < 0 || param.value > 65535) {
      emit('log-message', {type: 'error', text: `${param.label} 的值超出有效范围 (0-65535)当前值: ${param.value}`});
      emit('show-toast', 'error', `${param.label} 值超出有效范围！`);
      return;
    }
    dataBytes.push((param.value >> 8) & 0xFF);
    dataBytes.push(param.value & 0xFF);
  } else if (param.type === 'select' && param.dataLength === 1) {
    if (param.value < 0 || param.value > 255) {
      emit('log-message', {type: 'error', text: `${param.label} 的值超出有效范围 (0-255)当前值: ${param.value}`});
      emit('show-toast', 'error', `${param.label} 值超出有效范围！`);
      return;
    }
    dataBytes.push(param.value & 0xFF);
  } else {
    emit('log-message', {type: 'error', text: `参数 ${param.label} 的类型或数据长度配置错误`});
    emit('show-toast', 'error', `${param.label} 配置错误！`);
    return;
  }

  const commandWithoutChecksum = [...param.setCmdPrefix, ...dataBytes];
  const checksum = calculateChecksum(commandWithoutChecksum);

  const dataToSend = new Uint8Array([...commandWithoutChecksum, checksum]);

  emit('send-serial-data', dataToSend);
  emit('log-message', {
    type: 'info',
    text: `发送设置 ${param.label} 命令。发送数据: [${Array.from(dataToSend).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`
  });
};
</script>
