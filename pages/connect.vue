<script setup>
import { useTemplateRef } from "vue";

// The prefix for messages coming from the bridge and not Genesis
const bridge_msg_prefix = "[bridge] ";
// The address of the bridge (https://github.com/prestel-uk/web-bridge) to connect to
const bridge_addr = "ws://127.0.0.1:1234";

const output = useTemplateRef("output");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let ws;

function parseResponse(response) {
  // Don't try to parse messages from the bridge
  if (response.startsWith(bridge_msg_prefix)) {
    console.log(response);
    return "";
  }

  let json = JSON.parse(response);
  let result = "";
  for (const i in json) {
    // The response is received as a JSON array of numbers which need to be converted to a string first
    result += String.fromCharCode(json[i]);
  }
  return result;
}

function connect() {
  ws = new WebSocket(bridge_addr);
  ws.onopen = () => {
    toggleConnectedButton.value.innerText = "Disconnect";
  }
  ws.onmessage = (event) => {
    console.log(event.data);
    output.value.innerText = parseResponse(event.data);
  }
  connected = true;
}

function disconnect() {
  ws.close();
  output.value.innerText = "";
  toggleConnectedButton.value.innerText = "Connect";
  connected = false;
}

function toggleConnected() {
  if (connected) {
    disconnect();
  } else {
    connect();
  }
}

onMounted(() => {
  window.addEventListener("keydown", (event) => {
    if (connected) {
      ws.send(event.key);
    }
  });
});
</script>

<template>
  <div>
    <p ref="output"></p>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>