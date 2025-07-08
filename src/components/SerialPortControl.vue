<template>
  <div class="p-4 bg-white shadow rounded-lg">
    <h2 class="text-xl font-bold mb-4">
      <CpuChipIcon class="w-6 inline-block" aria-hidden="true" />
      雷达设备控制
    </h2>
    <div class="flex items-center space-x-4 mb-4">
      <button @click="connectSerialPort" :disabled="isConnected"
              class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 disabled:opacity-50">
        {{ isConnected ? '已连接' : '连接串口' }}
      </button>
      <button @click="disconnectSerialPort" :disabled="!isConnected"
              class="px-4 py-2 bg-red-500 text-white rounded-md hover:bg-red-600 disabled:opacity-50">
        断开串口
      </button>
    </div>

    <div class="text-gray-700 mb-4">
      <p class="mb-1">
        <span v-if="portName" class="text-gray-700">当前设备: {{ portName }}</span>
      <span v-else class="text-gray-500">未连接设备</span>
      </p>
      <p>波特率: {{ baudRate }}</p>
      <p>数据位: {{ dataBits }}, 停止位: {{ stopBits }}, 校验位: {{ parity }}</p>
    </div>
  </div>
</template>
<script setup>
import { ref, onMounted, onUnmounted, defineEmits, defineExpose } from 'vue';
import { CpuChipIcon } from "@heroicons/vue/24/outline";

const port = ref(null);
const reader = ref(null);
const writer = ref(null);
const isConnected = ref(false);
const portName = ref('');
const emit = defineEmits(['data-received', 'log-message', 'command-response', 'parameter-response', 'show-toast']);

const baudRate = 115200;
const dataBits = 8;
const stopBits = 1;
const parity = 'none';

const serialOptions = {
  baudRate: baudRate,
  dataBits: dataBits,
  stopBits: stopBits,
  parity: parity,
};

const connectSerialPort = async () => {
  try {
    if (!('serial' in navigator)) {
      logMessage('error', '您的浏览器不支持 Web Serial API。请使用 Chrome 或 Edge 浏览器。');
      emit('show-toast', 'error', '浏览器不支持 Web Serial API！');
      return;
    }

    port.value = await navigator.serial.requestPort();
    portName.value = port.value.getInfo().usbProductId ? `USB Device (PID: ${port.value.getInfo().usbProductId})` : '未知设备';
    await port.value.open(serialOptions);
    isConnected.value = true;
    logMessage('info', `已连接到串口设备: ${portName.value}`);
    emit('show-toast', 'success', `已连接到串口设备: ${portName.value}`);

    reader.value = port.value.readable.getReader();
    writer.value = port.value.writable.getWriter();

    readData();
  } catch (error) {
    logMessage('error', `连接串口失败: ${error.message}`);
    emit('show-toast', 'error', `连接串口失败: ${error.message}`);
    isConnected.value = false;
    portName.value = '';
  }
};

const disconnectSerialPort = async () => {
  try {
    if (reader.value) {
      await reader.value.cancel();
      reader.value.releaseLock();
      reader.value = null;
    }
    if (writer.value) {
      writer.value.releaseLock();
      writer.value = null;
    }
    if (port.value) {
      await port.value.close();
      port.value = null;
    }
    isConnected.value = false;
    portName.value = '';
    logMessage('info', '已断开串口连接。');
    emit('show-toast', 'info', '已断开串口连接。');
  } catch (error) {
    logMessage('error', `断开串口失败: ${error.message}`);
    emit('show-toast', 'error', `断开串口失败: ${error.message}`);
  }
};

const readData = async () => {
  let receivedBuffer = new Uint8Array();
  const frameHeaderRadarToHost = new Uint8Array([0x55, 0xA5]);

  while (isConnected.value && reader.value) {
    try {
      const { value, done } = await reader.value.read();
      if (done) {
        logMessage('info', '串口读取完成');
        break;
      }
      if (value) {
        const newBuffer = new Uint8Array(receivedBuffer.length + value.length);
        newBuffer.set(receivedBuffer);
        newBuffer.set(value, receivedBuffer.length);
        receivedBuffer = newBuffer;

        while (receivedBuffer.length >= 4) { // 至少需要帧头2字节 + 数据长度2字节
          let headerIndex = -1;
          for (let i = 0; i <= receivedBuffer.length - 2; i++) {
            if (receivedBuffer[i] === frameHeaderRadarToHost[0] && receivedBuffer[i + 1] === frameHeaderRadarToHost[1]) {
              headerIndex = i;
              break;
            }
          }

          if (headerIndex === -1) {
            receivedBuffer = new Uint8Array();
            break;
          }

          if (headerIndex > 0) {
            logMessage('warn', `发现无效数据，跳过 ${headerIndex} 字节。`);
            receivedBuffer = receivedBuffer.slice(headerIndex);
          }

          if (receivedBuffer.length < 4) {
            break;
          }

          const dataLength = (receivedBuffer[2] << 8) | receivedBuffer[3];
          const totalFrameLength = 4 + dataLength;

          if (receivedBuffer.length >= totalFrameLength) {
            const frame = receivedBuffer.slice(0, totalFrameLength);

            const checksumIndex = totalFrameLength - 1;
            if (checksumIndex < 4) {
              logMessage('error', `帧长度异常，校验和位置无效。总长度: ${totalFrameLength}, 帧: ${Array.from(frame).map(b => b.toString(16).padStart(2, '0')).join(' ')}`);
              emit('show-toast', 'error', '接收帧异常：长度无效');
              receivedBuffer = receivedBuffer.slice(1);
              continue;
            }

            let calculatedChecksum = 0;
            for (let i = 0; i < checksumIndex; i++) {
              calculatedChecksum = (calculatedChecksum + frame[i]) & 0xFF;
            }
            const receivedChecksum = frame[checksumIndex];

            if (calculatedChecksum === receivedChecksum) {
              logMessage('debug', `收到完整帧，校验和正确。帧内容: ${Array.from(frame).map(b => b.toString(16).padStart(2, '0')).join(' ')}`);
              parseFrame(frame);
            } else {
              logMessage('error', `校验和错误：计算值 0x${calculatedChecksum.toString(16).padStart(2, '0')}, 接收值 0x${receivedChecksum.toString(16).padStart(2, '0')}，帧：${Array.from(frame).map(b => b.toString(16).padStart(2, '0')).join(' ')}`);
              emit('show-toast', 'error', '校验和错误！');
            }

            receivedBuffer = receivedBuffer.slice(totalFrameLength);
          } else {
            break;
          }
        }
      }
    } catch (error) {
      if (error.name === 'NetworkError' && error.message.includes('The port is already closed')) {
        logMessage('warn', '串口已关闭，停止读取');
      } else {
        logMessage('error', `读取串口数据失败: ${error.message}`);
        emit('show-toast', 'error', `串口读取失败: ${error.message}`);
      }
      disconnectSerialPort();
      break;
    }
  }
};

const parseFrame = (frame) => {
  try {
    if (frame.length < 8) {
      logMessage('error', `接收到的帧（总长度${frame.length}）过短，无法解析。帧头：0x${frame[0]?.toString(16)} 0x${frame[1]?.toString(16)}`);
      emit('show-toast', 'error', '接收帧过短，无法解析！')
      return;
    }

    const dataLengthInHeader = (frame[2] << 8) | frame[3];
    const functionCode = frame[4];
    const commandCode1 = frame[5];
    const commandCode2 = frame[6];

    const actualDataBytesLength = dataLengthInHeader - 4;
    const dataStartIndex = 7;

    const receivedDataBytes = frame.slice(dataStartIndex, dataStartIndex + actualDataBytesLength);

    logMessage('info', `解析帧：功能码: 0x${functionCode.toString(16)}, 命令码1: 0x${commandCode1.toString(16)}, 命令码2: 0x${commandCode2.toString(16)}, 数据长度(in header): ${dataLengthInHeader}, 实际数据部分长度: ${actualDataBytesLength}。` +
        `接收到原始数据: [${Array.from(receivedDataBytes).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`);

    // 1. 雷达主动上报数据帧 (功能码 0x3, 命令码1 0x81, 命令码2 0x0)
    if (functionCode === 0x03 && commandCode1 === 0x81 && commandCode2 === 0x00 && actualDataBytesLength === 10) {
      if (receivedDataBytes.length < 10) {
        logMessage('error', `目标数据帧的数据部分长度不足10字节。实际长度: ${receivedDataBytes.length}`);
        emit('show-toast', 'error', '目标数据帧长度不足！');
        return;
      }

      const targetID = receivedDataBytes[0];
      const targetStatus = receivedDataBytes[1];
      const distance = (receivedDataBytes[2] << 8) | receivedDataBytes[3];
      const speed = new Int16Array([(receivedDataBytes[4] << 8) | receivedDataBytes[5]])[0];
      const azimuthAngle = new Int8Array([receivedDataBytes[6]])[0];
      const elevationAngle = new Int8Array([receivedDataBytes[7]])[0];
      const signalStrength = (receivedDataBytes[8] << 8) | receivedDataBytes[9];

      emit('data-received', {
        timestamp: Date.now(),
        functionCode,
        commandCode1,
        commandCode2,
        targetID,
        targetStatus,
        distance,
        speed,
        azimuthAngle,
        elevationAngle,
        signalStrength,
      });
      logMessage('info', '成功解析目标数据帧 (主动上报)。');
    }
    // 2. 雷达响应上位机命令的帧 (功能码 0x2)
    else if (functionCode === 0x02) {
      // 检查是否是参数读取或设置的响应 (命令码1 0x80)
      if (commandCode1 === 0x80) {
        emit('parameter-response', {
          functionCode,
          commandCode1,
          commandCode2,
          data: receivedDataBytes,
        });
        logMessage('info', `接收到参数配置响应帧：功能码: 0x${functionCode.toString(16)}, 命令码: 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}。` +
            `响应原始数据: [${Array.from(receivedDataBytes).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`);
      } else if (commandCode1 === 0x20) {
        let isCommandSuccess = false;
        let commandName = '未知命令';

        if (commandCode2 === 0x01) {
          commandName = 'OTA 在线升级';
          isCommandSuccess = true;
        }
        else if (commandCode2 === 0x02) {
          commandName = '恢复到默认参数';
          isCommandSuccess = true;
        }
        else if (commandCode2 === 0x04) {
          commandName = '将所有参数保存到 Flash';
          isCommandSuccess = true;
        }

        if (isCommandSuccess) {
          logMessage('success', `${commandName} 命令响应成功！`);
          emit('show-toast', 'success', `${commandName} 成功！`);
        } else {
          logMessage('info', `接收到命令响应帧 (功能码 0x${functionCode.toString(16)}, 命令码: 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)})。`);
          emit('show-toast', 'warning', `${commandName} 响应状态异常！`);
        }

        emit('command-response', {
          functionCode,
          commandCode1,
          commandCode2,
          status: isCommandSuccess ? 0x01 : 'N/A',
          data: receivedDataBytes,
        });
      } else if (commandCode1 === 0x00) {
        if (commandCode2 === 0x01 && receivedDataBytes.length === 22) {
          emit('parameter-response', {
            functionCode,
            commandCode1,
            commandCode2,
            data: receivedDataBytes,
          });

          // 将 receivedDataBytes 转为 ascii 字符串
          const version = String.fromCharCode(...receivedDataBytes.slice(0, 20)).trim();
          logMessage('info', `接收到软件版本响应：功能码: 0x${functionCode.toString(16)}, 命令码: 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}。` +
              `版本信息: ${version}`);
          emit('show-toast', 'success', `软件版本: ${version}`);
        } else {
          logMessage('warn', `接收到获取软件版本响应，但数据长度异常或命令码不匹配：功能码 0x${functionCode.toString(16)}, 命令码 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}。` +
              `响应原始数据: [${Array.from(receivedDataBytes).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`);
          emit('show-toast', 'warning', `获取软件版本响应异常: 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}`);
        }
      } else {
        logMessage('warn', `接收到功能码 0x02 的未知命令响应：命令码 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}。` +
            `响应原始数据: [${Array.from(receivedDataBytes).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`);
        emit('show-toast', 'warning', `未知命令响应 (0x02): 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}`);
        emit('command-response', {
          functionCode,
          commandCode1,
          commandCode2,
          status: 'UNKNOWN_COMMAND_RESPONSE',
          data: receivedDataBytes,
        });
      }
    }
    // 3. 其他未知帧类型
    else {
      logMessage('warn', `接收到未知帧类型：功能码 0x${functionCode.toString(16)}, 命令码 0x${commandCode1.toString(16)} 0x${commandCode2.toString(16)}，实际数据部分长度: ${actualDataBytesLength}。` +
          `接收到原始数据: [${Array.from(receivedDataBytes).map(b => b.toString(16).padStart(2, '0')).join(' ')}]`);
      emit('show-toast', 'warning', `未知帧类型: 0x${functionCode.toString(16)}`);
      emit('command-response', {
        functionCode,
        commandCode1,
        commandCode2,
        status: 'UNKNOWN_FRAME_TYPE',
        data: receivedDataBytes,
      });
    }
  } catch (e) {
    logMessage('error', `解析帧数据时出错: ${e.message}。原始帧: ${Array.from(frame).map(b => b.toString(16).padStart(2, '0')).join(' ')}`);
    emit('show-toast', 'error', `帧解析错误: ${e.message}`);
  }
};

const sendData = async (data) => {
  if (writer.value) {
    try {
      await writer.value.write(data);
      logMessage('info', `已发送数据: ${Array.from(data).map(byte => byte.toString(16).padStart(2, '0')).join(' ')}`);
      emit('show-toast', 'info', '数据已发送！');
    } catch (error) {
      logMessage('error', `发送数据失败: ${error.message}`);
      emit('show-toast', 'error', `发送数据失败: ${error.message}`);
    }
  } else {
    logMessage('warn', '串口未连接或写入器不可用。');
    emit('show-toast', 'warning', '串口未连接或写入器不可用！');
  }
};

const logMessage = (type, text) => {
  emit('log-message', { type, text, timestamp: new Date().toLocaleTimeString() });
};

onMounted(() => {
});

onUnmounted(() => {
  disconnectSerialPort();
});

defineExpose({ sendData });
</script>
