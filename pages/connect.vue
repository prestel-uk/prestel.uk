<script setup>
import { useTemplateRef } from "vue";

// The prefix for messages coming from the bridge and not Genesis
const bridge_msg_prefix = "[bridge] ";
// The address of the bridge (https://github.com/prestel-uk/web-bridge) to connect to
const bridge_addr = "ws://127.0.0.1:1234";

const outputContainer = useTemplateRef("outputContainer");
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
  for (const i in json) {
    // The response is received as a JSON array of character codes as numbers which need to have the parity bit removed, then converted into a string
    const withoutParity = json[i] & 0b01111111;
    // Create a new <span> element with a unique id so that it can be referenced later, and set the content to the character. If a span already exists for that character, use that instead.
    let charSpan = document.getElementById("char" + i);
    if (!charSpan) {
      charSpan = document.createElement("span");
    }

    charSpan.id = "char" + i;
    charSpan.innerText = String.fromCharCode(withoutParity);
    outputContainer.value.appendChild(charSpan);
  }
}

function connect() {
  ws = new WebSocket(bridge_addr);
  ws.onopen = () => {
    toggleConnectedButton.value.innerText = "Disconnect";
  }
  ws.onmessage = (event) => {
    console.log(event.data);
    parseResponse(event.data);
  }
  connected = true;
}

function disconnect() {
  ws.close();
  outputContainer.value.innerHTML = "";
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
    <div ref="outputContainer"></div>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>