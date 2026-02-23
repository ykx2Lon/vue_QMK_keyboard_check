<template>
  <div class="qmk-reader">
    <header>
      <h1>QMK HID Console 閱讀器</h1>
      <div class="actions">
        <button 
          @click="connectDevice" 
          :disabled="device !== null"
          class="btn connect"
        >
          {{ device ? '已連線' : '連接 QMK 鍵盤' }}
        </button>
        
        <button 
          @click="disconnectDevice" 
          v-if="device" 
          class="btn disconnect"
        >
          斷開
        </button>

        <button @click="clearLogs" class="btn clear">清除畫面</button>
      </div>
    </header>

    <div class="status-bar">
      <span>狀態: <b :class="{ active: device }">{{ device ? '監聽中' : '等待連線' }}</b></span>
      <span v-if="device">裝置: {{ device.productName }}</span>
    </div>

    <div class="console-window" ref="consoleRef">
      <div v-if="logs.length === 0" class="placeholder">
        等待數據輸入... (請確保韌體已啟用 CONSOLE_ENABLE = yes)
      </div>
      <div v-for="(log, index) in logs" :key="index" class="log-line">
        <span class="timestamp">[{{ log.time }}]</span>
        <span class="content">{{ log.text }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, onUnmounted } from 'vue';

// --- 定義介面 (Interface) ---
interface LogEntry {
  time: string;
  text: string;
}

// 為了避免 TypeScript 報錯 "Property 'hid' does not exist on type 'Navigator'"
// 我們這裡暫時將 navigator 轉型為 any，或者你需要安裝 @types/w3c-web-hid
const nav = navigator as any;

// --- 狀態變數 ---
// 使用 <any> 是因為 WebHID 的類型定義可能未安裝，這樣可以避免 TS 檢查錯誤
const device = ref<any>(null);
const logs = ref<LogEntry[]>([]);
const consoleRef = ref<HTMLElement | null>(null);

// --- QMK 標準 Console 設定 ---
const QMK_USAGE_PAGE = 0xFF31;
const QMK_USAGE = 0x0074;

// --- 連接裝置 ---
const connectDevice = async () => {
  try {
    if (!('hid' in nav)) {
      alert('您的瀏覽器不支援 WebHID API，請使用 Chrome 或 Edge。');
      return;
    }

    const devices = await nav.hid.requestDevice({
      filters: [{ usagePage: QMK_USAGE_PAGE, usage: QMK_USAGE }]
    });

    if (devices.length === 0) return;

    device.value = devices[0];
    
    if (!device.value.opened) {
      await device.value.open();
    }

    device.value.addEventListener('inputreport', handleInputReport);
    
    addLog('系統', `成功連接至 ${device.value.productName}`);
    
    nav.hid.addEventListener('disconnect', handleDisconnectEvent);

  } catch (err: any) {
    console.error(err);
    addLog('錯誤', `連線失敗: ${err.message}`);
  }
};

// --- 處理 HID 報告 ---
const handleInputReport = (event: any) => {
  const { data } = event;
  const decoder = new TextDecoder();
  let text = decoder.decode(data);
  
  text = text.replace(/\0/g, '').trim();

  if (text.length > 0) {
    addLog('QMK', text);
  }
};

// --- 輔助功能 ---
const addLog = (source: string, text: string) => {
  const now = new Date();
  const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;
  
  logs.value.push({
    time: timeStr,
    text: source === '系統' ? `--- ${text} ---` : text
  });

  nextTick(() => {
    if (consoleRef.value) {
      consoleRef.value.scrollTop = consoleRef.value.scrollHeight;
    }
  });
};

const disconnectDevice = async () => {
  if (device.value) {
    device.value.removeEventListener('inputreport', handleInputReport);
    await device.value.close();
    device.value = null;
    addLog('系統', '已手動斷開連線');
  }
};

const handleDisconnectEvent = (event: any) => {
  if (device.value && event.device === device.value) {
    device.value = null;
    addLog('系統', '裝置已從電腦移除');
  }
};

const clearLogs = () => {
  logs.value = [];
};

onUnmounted(() => {
  if (device.value) {
    device.value.close();
  }
  if ('hid' in nav) {
    nav.hid.removeEventListener('disconnect', handleDisconnectEvent);
  }
});
</script>

<style scoped>
/* 樣式保持不變 */
.qmk-reader {
  font-family: 'Consolas', 'Monaco', monospace;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background: #2d2d2d;
  color: #e0e0e0;
  border-radius: 12px;
  min-height: 500px;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #444;
  padding-bottom: 15px;
  margin-bottom: 15px;
}

h1 { margin: 0; font-size: 1.2rem; color: #fff; }

.btn {
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  margin-left: 10px;
}

.connect { background: #4caf50; color: white; }
.connect:disabled { background: #444; cursor: not-allowed; }
.disconnect { background: #f44336; color: white; }
.clear { background: #555; color: white; }

.status-bar {
  margin-bottom: 10px;
  font-size: 0.9rem;
  color: #aaa;
  display: flex;
  gap: 20px;
}

.active { color: #4caf50; }

.console-window {
  background: #1e1e1e;
  border: 1px solid #333;
  height: 400px;
  overflow-y: auto;
  padding: 10px;
  border-radius: 6px;
  box-shadow: inset 0 0 10px rgba(0,0,0,0.5);
}

.log-line {
  margin-bottom: 4px;
  line-height: 1.4;
  word-break: break-all;
}

.timestamp {
  color: #61afef;
  margin-right: 10px;
  font-size: 0.85rem;
}

.content { color: #dcdcdc; }
.placeholder { color: #555; text-align: center; margin-top: 150px; }
</style>