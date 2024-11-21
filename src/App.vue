<script setup lang="ts">
import { BrowserQRCodeReader } from "@zxing/browser";
import { onMounted, ref } from "vue";

const wsClient = ref<WebSocket | null>(null);
const connectionStatus = ref("未連接");
const gyroscopeData = ref({ alpha: 0, beta: 0, gamma: 0 });
const isGyroscopeAvailable = ref(false);
const isScanning = ref(false);
const videoElement = ref<HTMLVideoElement | null>(null);
const roomId = ref("");
const errorMessage = ref("");

interface DeviceOrientationEventiOS extends DeviceOrientationEvent {
  requestPermission?: () => Promise<"granted" | "denied" | "default">;
}

async function startQRScanner() {
  isScanning.value = true;
  const codeReader = new BrowserQRCodeReader();

  try {
    const videoInputDevices = await BrowserQRCodeReader.listVideoInputDevices();
    const backCamera = videoInputDevices.find(
      (device) =>
        device.label.toLowerCase().includes("back") ||
        device.label.toLowerCase().includes("後")
    );
    const deviceId = backCamera
      ? backCamera.deviceId
      : videoInputDevices[0].deviceId;

    const controlsPromise = codeReader.decodeFromVideoDevice(
      deviceId,
      videoElement.value!,
      async (result) => {
        if (result) {
          controlsPromise.then((controls) => controls.stop());
          isScanning.value = false;

          const wsUrl = result.getText();
          try {
            const url = new URL(wsUrl);
            const roomId = url.searchParams.get("room");
            console.log("url", url);
            console.log("roomId", roomId);

            if (!roomId) {
              errorMessage.value = "Missing room ID";
              throw new Error("Missing room ID");
            }

            connectToWebSocket(wsUrl);
          } catch (error) {
            errorMessage.value = "無效的 QR Code";
          }
        }
      }
    );
  } catch (error) {
    errorMessage.value = "無法存取相機";
  }
}

function connectToWebSocket(url: string) {
  console.log("connect to :", url);

  try {
    wsClient.value = new WebSocket(url);

    wsClient.value.onopen = () => {
      connectionStatus.value = "已連接";
      startGyroscope();
    };

    wsClient.value.onerror = (error) => {
      connectionStatus.value = "連線失敗";
      // 可以在這裡添加重試邏輯
      // setTimeout(() => {
      //   if (wsClient.value?.readyState === WebSocket.CLOSED) {
      //     console.log("Attempting to reconnect...");
      //     connectToWebSocket(roomId);
      //   }
      // }, 3000);
    };

    wsClient.value.onclose = (event) => {
      console.log("WebSocket closed:", event.code, event.reason);
      connectionStatus.value = "連接已關閉";
      wsClient.value = null;
    };
  } catch (error) {
    console.error("WebSocket 初始化錯誤:", error);
    connectionStatus.value = "連線失敗";
  }
}

async function startGyroscope() {
  if (!isGyroscopeAvailable.value || !wsClient.value) {
    errorMessage.value = "陀螺儀不可用或 WebSocket 未連接";
    return;
  }

  try {
    // iOS 權限請求
    if (
      typeof (DeviceOrientationEvent as unknown as DeviceOrientationEventiOS)
        .requestPermission === "function"
    ) {
      const permission = await (
        DeviceOrientationEvent as unknown as DeviceOrientationEventiOS
      ).requestPermission?.();

      if (permission !== "granted") {
        errorMessage.value = "陀螺儀權限被拒絕";
        throw new Error("陀螺儀權限被拒絕");
      }
    }

    let lastSend = 0;
    const THROTTLE_MS = 30;
    const SENSITIVITY = 10;
    const THRESHOLD = 100;
    let previousData = { alpha: 0, beta: 0, gamma: 0 };

    window.addEventListener("deviceorientation", handleDeviceOrientation);

    function handleDeviceOrientation(event: DeviceOrientationEvent) {
      const now = Date.now();
      if (now - lastSend < THROTTLE_MS) return;

      if (event.alpha !== null && event.beta !== null && event.gamma !== null) {
        const alphaDiff = previousData.alpha - event.alpha;
        const betaDiff = previousData.beta - event.beta;
        const gammaDiff = previousData.gamma - event.gamma;

        if (Math.abs(alphaDiff) < THRESHOLD && Math.abs(betaDiff) < THRESHOLD) {
          gyroscopeData.value = {
            alpha: alphaDiff * SENSITIVITY,
            beta: betaDiff * SENSITIVITY,
            gamma: gammaDiff * SENSITIVITY,
          };

          if (wsClient.value?.readyState === WebSocket.OPEN) {
            wsClient.value.send(
              JSON.stringify({
                type: "gyroscope",
                data: gyroscopeData.value,
              })
            );
          }
        }
        previousData = {
          alpha: event.alpha,
          beta: event.beta,
          gamma: event.gamma,
        };
        lastSend = now;
      }
    }
  } catch (error) {
    errorMessage.value = "陀螺儀啟動失敗";
  }
}

async function requestGyroscopePermission() {
  try {
    if (
      typeof (DeviceOrientationEvent as unknown as DeviceOrientationEventiOS)
        .requestPermission === "function"
    ) {
      const permission = await (
        DeviceOrientationEvent as unknown as DeviceOrientationEventiOS
      ).requestPermission?.();
      if (permission === "granted") {
        isGyroscopeAvailable.value = true;
      }
    }
  } catch (error) {
    errorMessage.value = "請求陀螺儀權限失敗";
  }
}

onMounted(() => {
  if (window.DeviceOrientationEvent) {
    window.addEventListener(
      "deviceorientation",
      (event) => {
        if (
          event.alpha !== null ||
          event.beta !== null ||
          event.gamma !== null
        ) {
          console.log("陀螺儀可用");
          isGyroscopeAvailable.value = true;
        } else {
          console.log("陀螺儀數值為 null");
          isGyroscopeAvailable.value = false;
        }
      },
      { once: true }
    );
  } else {
    errorMessage.value = "設備不支援陀螺儀";
    isGyroscopeAvailable.value = false;
  }
});
</script>

<template>
  <div class="app">
    <div class="status">連接狀態: {{ connectionStatus }}</div>
    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
    <div v-if="!wsClient" class="scanner-container">
      <div class="input-container">
        <input
          v-model="roomId"
          type="text"
          placeholder="輸入房間 ID"
          class="room-input"
        />
      </div>

      <button
        @click="startQRScanner"
        :disabled="isScanning"
        class="scan-button"
      >
        {{ isScanning ? "掃描中..." : "掃描QR Code" }}
      </button>
      <button
        @click="connectToWebSocket(`wss://192.168.168.124:8080?room=${roomId}`)"
        :disabled="isScanning || !roomId"
        class="connect-button"
      >
        直接連線測試
      </button>
      <video
        ref="videoElement"
        class="scanner-video"
        :class="{ scanning: isScanning }"
      ></video>
    </div>

    <div v-if="!isGyroscopeAvailable" class="error">
      您的裝置不支援陀螺儀功能
    </div>

    <div v-else-if="wsClient" class="gyroscope-data">
      <h2 class="text-black">陀螺儀數據</h2>
      <p class="text-black">
        前後傾斜 (Beta): {{ gyroscopeData.beta.toFixed(2) }}°
      </p>
      <!-- <p class="text-black">
        左右傾斜 (Gamma): {{ gyroscopeData.gamma.toFixed(2) }}°
      </p> -->
      <p class="text-black">
        旋轉 (Alpha): {{ gyroscopeData.alpha.toFixed(2) }}°
      </p>
    </div>

    <button
      v-if="!isGyroscopeAvailable"
      @click="requestGyroscopePermission"
      class="permission-button"
    >
      請求陀螺儀權限
    </button>
  </div>
</template>

<style scoped>
.app {
  padding: 20px;
  text-align: center;
}

.status {
  margin: 20px 0;
  padding: 10px;
  background-color: #000;
  border-radius: 4px;
}

.scanner-container {
  margin: 20px auto;
  max-width: 100%;
  width: 300px;
}

.scanner-video {
  display: none;
  width: 100%;
  height: 300px;
  object-fit: cover;
  border-radius: 8px;
  margin-top: 10px;
}

.scanner-video.scanning {
  display: block;
}

.scan-button {
  background-color: #4caf50;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin: 10px 0;
}

.scan-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.error {
  color: red;
  margin: 20px 0;
}

.gyroscope-data {
  margin-top: 20px;
  padding: 20px;
  background-color: #f8f8f8;
  border-radius: 8px;
}

h1 {
  color: #2c3e50;
  margin-bottom: 30px;
}

h2 {
  color: #34495e;
  margin-bottom: 15px;
}

.connect-button {
  background-color: #2196f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin: 10px 0;
  margin-left: 10px;
}

.connect-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.text-black {
  color: #2c3e50;
}

.permission-button {
  background-color: #2196f3;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin: 10px 0;
  margin-left: 10px;
}

.permission-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.connect-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.text-black {
  color: #2c3e50;
}

.input-container {
  margin: 10px 0;
}

.room-input {
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 16px;
  width: 200px;
}

.room-input:focus {
  outline: none;
  border-color: #2196f3;
}
</style>
