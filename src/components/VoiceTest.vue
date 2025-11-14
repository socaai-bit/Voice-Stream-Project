<template>
  <div class="p-6">
    <h2 class="text-xl font-bold mb-4">🎤 Voice Stream WebSocket Test</h2>

    <div class="mb-4">
      <label class="block mb-1 font-semibold">Server URL</label>
      <input
        v-model="serverUrl"
        class="border p-2 rounded w-full"
        placeholder="wss://stg-api-genesist.soca.ai/voice-ws"
      />
    </div>

    <div class="mb-4">
      <button
        @click="connect"
        :disabled="isConnected"
        class="bg-green-500 hover:bg-green-600 text-white px-4 py-2 rounded mr-2"
      >
        Connect
      </button>
      <button
        @click="disconnect"
        :disabled="!isConnected"
        class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded"
      >
        Disconnect
      </button>
    </div>

    <div class="mb-4">
      <button
        @click="sendStart"
        :disabled="!isConnected"
        class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded mr-2"
      >
        Send Start Event
      </button>
      <button
        @click="sendTestMessage"
        :disabled="!isConnected"
        class="bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 rounded"
      >
        Send Test Message
      </button>
    </div>

    <div class="border p-3 rounded bg-gray-50 h-64 overflow-auto">
      <p v-for="(log, i) in logs" :key="i" class="text-sm font-mono">
        {{ log }}
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const ws = ref(null);
const serverUrl = ref("wss://stg-api-genesist.soca.ai/voice-ws");
const isConnected = ref(false);
const logs = ref([]);

const log = (msg) => {
  console.log(msg);
  logs.value.push(`[${new Date().toLocaleTimeString()}] ${msg}`);
};

const connect = () => {
  ws.value = new WebSocket(serverUrl.value);
  ws.value.binaryType = "arraybuffer";

  ws.value.onopen = () => {
    log("✅ Connected to WebSocket server");
    isConnected.value = true;
  };

  ws.value.onmessage = (e) => {
    log("📩 Message: " + e.data);
  };

  ws.value.onerror = (e) => {
    log("❌ WebSocket error");
    console.error(e);
  };

  ws.value.onclose = () => {
    log("🔌 Disconnected");
    isConnected.value = false;
  };
};

const disconnect = () => {
  if (ws.value) {
    ws.value.close();
    ws.value = null;
  }
};

const sendStart = () => {
  const payload = {
    "type": "start",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZF9jbGllbnQiOiI2N2IzZmMzMTdjNjI0NDJkYjU3NzAwZDIiLCJ0b2tlbl92ZXJzaW9uIjoyLCJpYXQiOjE3NjE1NjEzMTJ9.h63bFtvVs-k9u2oP-h_4CPxZ90S6V6-agZmGxVefxkM",
    "session_id": "3f62decb-b0e8-4331-9124-5e41d1944fef",
    "agent_id": "97eaedbb-f1cd-4cef-830e-6af4f75fb726",
    "lang": "id",
    "sr": 16000
  };
  ws.value.send(JSON.stringify(payload));
  log("🚀 Sent start payload");
};

const sendTestMessage = () => {
  const payload = { type: "ping", time: new Date().toISOString() };
  ws.value.send(JSON.stringify(payload));
  log("➡️ Sent test message: " + JSON.stringify(payload));
};
</script>

<style scoped>
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
