<script setup lang="ts">
import "~/assets/css/vtxcore.css";
import {useTemplateRef} from "vue";
import init, {VTXScreen} from "@prestel-uk/vtxcore";

// The offset of contiguous mosaics in the font
const CONTIGUOUS_MOSAICS_OFFSET = 57856;
// The offset of separated mosaics in the font
const SEPARATED_MOSAICS_OFFSET = 58048;

// The prefix for messages coming from the bridge and not Genesis
const BRIDGE_MSG_PREFIX = "[bridge] ";
// The address of the bridge (https://github.com/prestel-uk/web-bridge) to connect to
const BRIDGE_ADDR = "ws://127.0.0.1:1234";

const ROWS = 24;
const COLUMNS = 40;

const outputContainer = useTemplateRef("outputContainer");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let screen: VTXScreen;
let ws: WebSocket;

// Clear the terminal
function clearTerminal() {
  screen.clear();
  drawScreen();
}

async function parseResponse(response: Blob) {
  let data = await new Response(response).text();

  // Don't try to parse messages from the bridge
  if (data.startsWith(BRIDGE_MSG_PREFIX)) {
    console.log(response);
    return "";
  }

  screen.parse_string(data);
  drawScreen();
}

function drawScreen() {
  outputContainer.value!.innerHTML = screen.display_for_html();
}

function connect() {
  screen = VTXScreen.new(ROWS, COLUMNS);

  ws = new WebSocket(BRIDGE_ADDR);
  ws.onopen = () => {
    toggleConnectedButton.value!.innerText = "Disconnect";
  }
  ws.onmessage = (event) => {
    parseResponse(event.data);
  }

  connected = true;
}

function disconnect() {
  ws.close();
  clearTerminal();
  toggleConnectedButton.value!.innerText = "Connect";
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
  init();

  window.addEventListener("keydown", (event) => {
    // We have to check the length of the key, otherwise things like "Enter", and "Shift" get sent too
    if (connected && event.key.length === 1) {
      ws.send(event.key);
    }
  });
});
</script>

<template>
  <div class="flex flex-col justify-center items-center">
    <div ref="outputContainer"></div>
    <br>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>