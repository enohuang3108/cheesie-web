<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";

interface GyroscopeData {
  alpha: number;
  beta: number;
  gamma: number;
}

const gyroData = ref<GyroscopeData>({
  alpha: 0,
  beta: 0,
  gamma: 0,
});

const isSupported = ref(false);
const errorMessage = ref("");
const permissionRequested = ref(false);
const logMessage = ref("");

function handleOrientation(event: DeviceOrientationEvent) {
  gyroData.value = {
    alpha: event.alpha || 0,
    beta: event.beta || 0,
    gamma: event.gamma || 0,
  };
}

async function requestPermission() {
  if ((DeviceOrientationEvent as any).requestPermission) {
    try {
      const permission = await (
        DeviceOrientationEvent as any
      ).requestPermission();
      console.log("Permission response:", permission); // 調試用
      log("Permission response:" + permission); // 調試用
      if (permission === "granted") {
        window.addEventListener("deviceorientation", handleOrientation);
        permissionRequested.value = true;
      } else {
        errorMessage.value = "使用者拒絕了陀螺儀權限";
      }
    } catch (error) {
      console.error("Error requesting permission:", error); // 調試用
      errorMessage.value = "請求權限時發生錯誤";
    }
  } else {
    errorMessage.value = "此裝置不支援權限請求";
  }
}

function log(message: string) {
  logMessage.value = message;
}
log("這是一條日誌信息");

onMounted(() => {
  if ("DeviceOrientationEvent" in window) {
    isSupported.value = true;
  } else {
    errorMessage.value = "此裝置不支援陀螺儀";
    isSupported.value = false;
  }
});

onUnmounted(() => {
  window.removeEventListener("deviceorientation", handleOrientation);
});
</script>

<template>
  <div class="gyroscope-display">
    <h2>陀螺儀資訊</h2>

    <button
      v-if="isSupported && !permissionRequested"
      @click="requestPermission"
    >
      請求陀螺儀權限
    </button>

    <div
      v-if="isSupported && !errorMessage && permissionRequested"
      class="data-container"
    >
      <div class="data-row">
        <span>Alpha (Z軸旋轉):</span>
        <span>{{ gyroData.alpha.toFixed(2) }}°</span>
      </div>
      <div class="data-row">
        <span>Beta (X軸旋轉):</span>
        <span>{{ gyroData.beta.toFixed(2) }}°</span>
      </div>
      <div class="data-row">
        <span>Gamma (Y軸旋轉):</span>
        <span>{{ gyroData.gamma.toFixed(2) }}°</span>
      </div>
    </div>

    <div v-if="errorMessage" class="error">
      {{ errorMessage }}
    </div>

    <div v-if="logMessage" class="log">
      {{ logMessage }}
    </div>
  </div>
</template>

<style scoped>
.gyroscope-display {
  padding: 20px;
  max-width: 600px;
  margin: 0 auto;
}

.data-container {
  margin-top: 20px;
  background: #f5f5f5;
  padding: 15px;
  border-radius: 8px;
}

.data-row {
  display: flex;
  justify-content: space-between;
  padding: 10px 0;
  border-bottom: 1px solid #ddd;
}

.data-row:last-child {
  border-bottom: none;
}

.error {
  color: #ff4444;
  margin-top: 20px;
  padding: 10px;
  background: #ffebee;
  border-radius: 4px;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

.log {
  margin-top: 20px;
  padding: 10px;
  background: #e0e0e0;
  border-radius: 4px;
}
</style>
