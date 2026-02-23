<template>
  <div class="qmk-tester">
    <header>
      <h1>QMK 矩陣測試器</h1>
      <div class="actions">
        <button 
          @click="connectDevice" 
          :disabled="device !== null"
          class="btn connect"
        >
          {{ device ? '已連線' : '連接鍵盤' }}
        </button>
        <button 
          @click="disconnectDevice" 
          v-if="device" 
          class="btn disconnect"
        >
          斷開
        </button>
        <button @click="resetMatrix" class="btn clear">重置測試</button>
      </div>
    </header>

    <div v-if="matrixRows > 0" class="matrix-container">
      <div class="matrix-grid" :style="{ gridTemplateColumns: `repeat(${matrixCols}, 1fr)` }">
        <template v-for="r in matrixRows" :key="`row-${r-1}`">
          <div 
            v-for="c in matrixCols" 
            :key="`cell-${r-1}-${c-1}`"
            class="key-unit"
            :class="{ 'tested': testedState[r-1]?.[c-1] }"
          >
            <div class="coord">{{ r-1 }},{{ c-1 }}</div>
          </div>
        </template>
      </div>
    </div>

    <div v-else class="placeholder-box">
      <p v-if="device">請按下鍵盤任意鍵，以偵測矩陣大小...</p>
      <p v-else>請先連接支援 QMK Console 的鍵盤</p>
    </div>

    <details class="log-details">
      <summary>顯示原始 Console Log</summary>
      <div class="console-window" ref="consoleRef">
        <div v-for="(log, index) in logs" :key="index" class="log-line">
          <span class="timestamp">[{{ log.time }}]</span> {{ log.text }}
        </div>
      </div>
    </details>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, onUnmounted } from 'vue';

// --- 型別定義 ---
interface LogEntry {
  time: string;
  text: string;
}

// 避免 TypeScript 檢查 navigator.hid
const nav = navigator as any;

// --- 狀態變數 ---
const device = ref<any>(null);
const logs = ref<LogEntry[]>([]);
const consoleRef = ref<HTMLElement | null>(null);

// 矩陣相關狀態
const matrixRows = ref(0);
const matrixCols = ref(0);
const testedState = ref<boolean[][]>([]);

const QMK_USAGE_PAGE = 0xFF31;
const QMK_USAGE = 0x0074;

// --- 核心邏輯: 解析 QMK Log ---
let logBuffer = "";

const parseMatrixData = (text: string) => {
  logBuffer += text;
  
  // 避免 Buffer 過大，保留最後 5000 字元
  if (logBuffer.length > 5000) logBuffer = logBuffer.slice(-5000);

  // --- A. 偵測 Column 數量 ---
  const headerMatch = logBuffer.match(/r\/c\s+([0-9A-F]+)/i);
  if (headerMatch && matrixCols.value === 0) {
    const colString = headerMatch[1] || '';
    if (colString) {
      matrixCols.value = colString.length;
      addLog('系統', `偵測到 ${matrixCols.value} 個 Columns`);
    }
  }

  // --- B. 偵測 Row 與按鍵狀態 (使用 Split 解決換行問題) ---
  // 原理：直接用 "XX:" (Row ID) 來切開字串
  // 範例 split 結果: ["...雜訊", "00", " 0000\n0000 ", "01", " 1111..."]
  const parts = logBuffer.split(/([0-9A-F]{2}):/);

  // 從 1 開始遍歷 (parts[1] 是第一個 Row ID, parts[2] 是它的資料)
  // 我們只處理成對的 (ID + Data)，最後一段如果是殘缺的也先處理，反正之後補齊會更新
  for (let i = 1; i < parts.length - 1; i += 2) {
    const rowHex = parts[i];       // 例如 "00"
    const rawData = parts[i + 1];  // 例如 " 0000\n[06:31:12]0000 "

    const rowIdx = parseInt(rowHex, 16);

    // 關鍵修正：把 rawData 裡面所有不是 0 或 1 的字元全部刪掉 (包含換行、空白、時間戳)
    const binStr = rawData.replace(/[^01]/g, '');

    if (!binStr) continue;

    // 1. 動態擴充 Rows
    if (rowIdx >= matrixRows.value) {
      expandMatrixRows(rowIdx + 1);
    }

    // 2. 動態擴充 Cols (如果沒抓到 Header)
    if (matrixCols.value === 0) {
      matrixCols.value = binStr.length;
      expandMatrixCols(binStr.length);
    }

    // 3. 更新狀態
    const targetRow = testedState.value[rowIdx];
    if (targetRow) {
      for (let c = 0; c < binStr.length; c++) {
        // 只有 '1' 會觸發變黑，確保不會因為資料長度不同而誤將 '1' 蓋回 '0'
        if (binStr[c] === '1') {
           targetRow[c] = true;
        }
      }
    }
  }
};

const expandMatrixRows = (newRowCount: number) => {
  const currentRows = testedState.value.length;
  for (let i = currentRows; i < newRowCount; i++) {
    testedState.value.push(new Array(matrixCols.value || 16).fill(false));
  }
  matrixRows.value = newRowCount;
};

const expandMatrixCols = (newColCount: number) => {
    testedState.value.forEach(row => {
        while(row.length < newColCount) row.push(false);
    });
    matrixCols.value = newColCount;
}

// --- HID 連線與事件 ---
const connectDevice = async () => {
  try {
    if (!('hid' in nav)) return alert('不支援 WebHID');
    
    const devices = await nav.hid.requestDevice({
      filters: [{ usagePage: QMK_USAGE_PAGE, usage: QMK_USAGE }]
    });

    if (devices.length === 0) return;
    device.value = devices[0];
    if (!device.value.opened) await device.value.open();

    device.value.addEventListener('inputreport', handleInputReport);
    nav.hid.addEventListener('disconnect', handleDisconnectEvent);
    
    addLog('系統', `已連線: ${device.value.productName}`);
  } catch (err: any) {
    addLog('錯誤', err.message);
  }
};

const handleInputReport = (event: any) => {
  const decoder = new TextDecoder();
  const rawText = decoder.decode(event.data).replace(/\0/g, '');
  const cleanText = rawText.trim();

  if (cleanText) {
    addLog('QMK', cleanText);
    parseMatrixData(cleanText);
  }
};

const addLog = (source: string, text: string) => {
  const timeStr = new Date().toLocaleTimeString('en-GB');
  logs.value.push({ time: timeStr, text: `${source === '系統' ? 'INFO' : ''} ${text}` });
  
  nextTick(() => {
    if (consoleRef.value) consoleRef.value.scrollTop = consoleRef.value.scrollHeight;
  });
};

const disconnectDevice = async () => {
  if (device.value) {
    await device.value.close();
    device.value = null;
    logBuffer = "";
  }
};

const handleDisconnectEvent = (event: any) => {
  if (device.value === event.device) {
    device.value = null;
  }
};

const resetMatrix = () => {
  testedState.value = testedState.value.map(row => row.map(() => false));
  logs.value = [];
  addLog('系統', '矩陣狀態已重置');
};

onUnmounted(() => {
  if (device.value) device.value.close();
});
</script>

<style scoped>
.qmk-tester {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
  background: #2d2d2d;
  color: #e0e0e0;
  border-radius: 12px;
  font-family: 'Segoe UI', sans-serif;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #444;
  padding-bottom: 15px;
  margin-bottom: 20px;
}

.matrix-container {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
  padding: 10px;
  background: #222;
  border-radius: 8px;
  overflow-x: auto;
}

.matrix-grid {
  display: grid;
  gap: 4px; /* 格子間距 */
  /* grid-template-columns 會在 style 裡動態設定 */
}

.key-unit {
  width: 40px;
  height: 40px;
  background-color: #555; /* 預設灰色 */
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  font-size: 0.75rem;
  color: #aaa;
  transition: background-color 0.1s;
  user-select: none;
}

/* 變黑的狀態 */
.key-unit.tested {
  background-color: #000; /* 黑色 */
  color: #4caf50; /* 文字變綠色，比較看得到 */
  border: 1px solid #333;
  box-shadow: inset 0 0 5px #222;
}

.placeholder-box {
  text-align: center;
  padding: 40px;
  color: #666;
  border: 2px dashed #444;
  border-radius: 8px;
  margin-bottom: 20px;
}

.log-details summary {
  cursor: pointer;
  color: #888;
  margin-bottom: 10px;
}

.console-window {
  background: #111;
  height: 150px;
  overflow-y: auto;
  padding: 10px;
  font-family: monospace;
  font-size: 0.85rem;
  border-radius: 4px;
}

.log-line { margin-bottom: 2px; }
.timestamp { color: #569cd6; margin-right: 8px; }

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
.clear { background: #ff9800; color: white; }
</style>