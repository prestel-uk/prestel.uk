<script setup lang="ts">
import "~/assets/css/terminal.css";
import {useTemplateRef} from "vue";
import init, {Colour, MosaicType, VTXScreen} from "@prestel-uk/vtxcore";

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

const outputGrid = useTemplateRef("outputGrid");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let screen: VTXScreen;
let ws: WebSocket;

// Clear the terminal
function clearTerminal() {
  screen.clear();
  drawScreen();
}

// Get a mosaic character using the number sent by the server
function getMosaicChar(character: string, mosaicType: MosaicType) {
  const code = character.charCodeAt(0);

  // The 6th bit is used for something else which I don't understand at the moment, so for now it just gets ignored
  const maskedCode = code & 0b01011111;
  switch (mosaicType) {
    case MosaicType.Contiguous:
      return String.fromCodePoint(maskedCode + CONTIGUOUS_MOSAICS_OFFSET);
    case MosaicType.Separated:
      return String.fromCodePoint(maskedCode + SEPARATED_MOSAICS_OFFSET);
    default:
      return "";
  }
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
  // Keep track of where in the grid we are
  let idx = 0;
  for (let c of screen.buffer) {
    // If the character shouldn't be displayed, do nothing
    if (c.to_be_displayed === undefined) {
      continue;
    }

    // Each character has its own <span> element which is created when the page loads
    let charSpan = document.getElementById("char" + idx);

    let colour = "";

    switch (c.colour) {
      case Colour.Red: colour = "red"; break;
      case Colour.Green: colour = "green"; break;
      case Colour.Yellow: colour = "yellow"; break;
      case Colour.Blue: colour = "blue"; break;
      case Colour.Magenta: colour = "magenta"; break;
      case Colour.Cyan: colour = "cyan"; break;
      case Colour.White: colour = "white"; break;
    }

    let heightMode = "normal-height";

    if (c.double_height) {
      heightMode = "double-height";
    }

    charSpan!.innerText = c.is_mosaic ? getMosaicChar(c.to_be_displayed, c.mosaic_type) : c.to_be_displayed;
    charSpan!.dataset.colour = colour;
    charSpan!.dataset.heightmode = heightMode;

    idx++;
  }
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

  // Create the <span> elements for each character
  for (let i = 0; i < ROWS * COLUMNS; i++) {
    const element = document.createElement("span");

    element.id = "char" + i;

    outputGrid.value!.appendChild(element);
  }
});
</script>

<template>
  <div class="flex flex-col justify-center items-center">
    <div ref="outputGrid" id="outputGrid"></div>
    <br>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>