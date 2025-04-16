<script setup>
import { useTemplateRef } from "vue";

const output = useTemplateRef("output");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let ws;

function responseToString(response) {
  let json = JSON.parse(response);
  let result = "";
  for (const i in json) {
    result += String.fromCharCode(json[i]);
  }
  return result;
}

function connect() {
  ws = new WebSocket("ws://127.0.0.1:1234");
  ws.onopen = () => {
    toggleConnectedButton.value.innerText = "Disconnect";
  }
  ws.onmessage = (event) => {
    console.log(event.data);
    output.value.innerText = responseToString(event.data);
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
</script>

<template>
  <div>
    <p ref="output"></p>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>