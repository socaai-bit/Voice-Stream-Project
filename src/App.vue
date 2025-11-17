<template>
  <div class="p-6 max-w-lg mx-auto">
    <h1 class="text-2xl font-bold mb-4">🎤 Voice Stream Test</h1>

    <div class="flex gap-4">
      <button
        @click="startRecording"
        :disabled="isRecording"
        class="bg-green-500 text-white px-4 py-2 rounded"
      >
        Start Recording
      </button>
      <button
        @click="stopRecording"
        :disabled="!isRecording"
        class="bg-red-500 text-white px-4 py-2 rounded"
      >
        Stop Recording
      </button>
    </div>

    <p class="mt-4 text-gray-600">
      Status: <strong>{{ status }}</strong>
    </p>

    <div class="mt-6">
      <h2 class="font-semibold mb-2">STT Output</h2>
      <pre class="bg-gray-100 p-3 rounded h-48 overflow-auto">{{ logs }}</pre>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const status = ref("Idle");
const logs = ref("");
const isRecording = ref(false);

let ws;
let audioContext;
let processor;
let source;
let stream;

const WS_URL = "wss://api.soca.ai/voice-ws";

// ------------------------------
// 1️⃣ Connect WebSocket
// ------------------------------
function connectWebSocket() {
  ws = new WebSocket(WS_URL);
  ws.binaryType = "arraybuffer";

  ws.onopen = () => {
    status.value = "Connected";
    ws.send(
      JSON.stringify({
        "type": "start",
        "private_key": "<private_key>",
        "session_id": "<session_id>",
        "agent_id": "<agent_id>",
        "lang": "id",
        "sr": 16000
      })
    );
  };

  ws.onmessage = (e) => {
    try {
      const msg = JSON.parse(e.data);
      if (msg.type === "ready") {
        logs.value += "🟢 Ready: " + msg.session_id + "\n";
      } else if (msg.type === "stt.partial") {
        logs.value += "Partial: " + msg.text + "\n";
      } else if (msg.type === "stt.final") {
        logs.value += "✅ Final: " + msg.text + "\n";
      } else {
        logs.value += JSON.stringify(msg) + "\n";
      }
    } catch {
      logs.value += "Received binary/audio chunk\n";
    }
  };

  ws.onclose = () => {
    status.value = "Disconnected";
    stopRecording();
  };
}

// ------------------------------
// 2️⃣ Start recording
// ------------------------------
async function startRecording() {
  if (isRecording.value) return;

  connectWebSocket();

  // Wait a bit for WS to open
  await new Promise((r) => setTimeout(r, 800));

  stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  audioContext = new (window.AudioContext || window.webkitAudioContext)({
    sampleRate: 16000, // match server expectation
  });
  source = audioContext.createMediaStreamSource(stream);
  processor = audioContext.createScriptProcessor(4096, 1, 1);

  processor.onaudioprocess = (e) => {
    const input = e.inputBuffer.getChannelData(0);
    const pcm16 = floatTo16BitPCM(input);
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(pcm16);
    }
  };

  source.connect(processor);
  processor.connect(audioContext.destination);
  isRecording.value = true;
  status.value = "Recording...";
}

// ------------------------------
// 3️⃣ Stop recording
// ------------------------------
function stopRecording() {
  if (!isRecording.value) return;

  if (processor) {
    processor.disconnect();
    source.disconnect();
  }
  if (stream) {
    stream.getTracks().forEach((t) => t.stop());
  }
  if (ws && ws.readyState === WebSocket.OPEN) {
    ws.close();
  }

  isRecording.value = false;
  status.value = "Stopped";
}

// ------------------------------
// 🔧 Helper: convert Float32 → Int16
// ------------------------------
function floatTo16BitPCM(float32Array) {
  const buffer = new ArrayBuffer(float32Array.length * 2);
  const view = new DataView(buffer);
  let offset = 0;
  for (let i = 0; i < float32Array.length; i++, offset += 2) {
    let s = Math.max(-1, Math.min(1, float32Array[i]));
    view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
  }
  return buffer;
}
</script>

<style>
body {
  font-family: sans-serif;
}
</style>
