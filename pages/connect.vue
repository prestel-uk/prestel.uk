<script setup>
import "~/assets/css/terminal.css";
import { useTemplateRef } from "vue";

// Control codes
const controlCodes = {
  "ESCAPE": { char: 0x1B },
  "ALPHANUMERIC_RED": { name: "alphanumeric-red", char: 0x41, type: "text" },
  "ALPHANUMERIC_GREEN": { name: "alphanumeric-green", char: 0x42, type: "text" },
  "ALPHANUMERIC_YELLOW": { name: "alphanumeric-yellow", char: 0x43, type: "text" },
  "ALPHANUMERIC_BLUE": { name: "alphanumeric-blue", char: 0x44, type: "text" },
  "ALPHANUMERIC_MAGENTA": { name: "alphanumeric-magenta", char: 0x45, type: "text" },
  "ALPHANUMERIC_CYAN": { name: "alphanumeric-cyan", char: 0x46, type: "text" },
  "ALPHANUMERIC_WHITE": { name: "alphanumeric-white", char: 0x47, type: "text" },
  "FLASHING": { name: "flashing", char: 0x48 },
  "STEADY": { name: "steady", char: 0x49 },
  "END_EDIT": { char: 0x4A },
  "START_EDIT": { char: 0x4B },
  "NORMAL_HEIGHT": { name: "normal-height", char: 0x4C, type: "height" },
  "DOUBLE_HEIGHT": { name: "double-height", char: 0x4D, type: "height" },
  "MOSAICS_RED": { name: "mosaics-red", char: 0x51, type: "mosaic" },
  "MOSAICS_GREEN": { name: "mosaics-green", char: 0x52, type: "mosaic" },
  "MOSAICS_YELLOW": { name: "mosaics-yellow", char: 0x53, type: "mosaic" },
  "MOSAICS_BLUE": { name: "mosaics-blue", char: 0x54, type: "mosaic" },
  "MOSAICS_MAGENTA": { name: "mosaics-magenta", char: 0x55, type: "mosaic" },
  "MOSAICS_CYAN": { name: "mosaics-cyan", char: 0x56, type: "mosaic" },
  "MOSAICS_WHITE": { name: "mosaics-white", char: 0x57, type: "mosaic" },
  "CONCEAL_DISPLAY": { char: 0x58 },
  "CONTIGUOUS_MOSAICS": { name: "contiguous-mosaics", char: 0x59, type: "mosaicsMode" },
  "SEPARATED_MOSAICS": { name: "separated-mosaics", char: 0x5A, type: "mosaicsMode" },
  "BLACK_BACKGROUND": { name: "black-background", char: 0x5C },
  "NEW_BACKGROUND": { name: "new-background", char: 0x5D },
  "HOLD_MOSAICS": { char: 0x5E },
  "RELEASE_MOSAICS": { char: 0x5F },
};

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
let ws;

let textMode = controlCodes.ALPHANUMERIC_WHITE;
let heightMode = controlCodes.NORMAL_HEIGHT;
let mosaicsMode = controlCodes.CONTIGUOUS_MOSAICS;

// Converts a row and column to an index
function rowAndColumnToIndex(row, col) {
  return row * COLUMNS + col;
}

// Converts an index to a row and column
function indexToRowAndColumn(index) {
  const row = Math.floor(index / COLUMNS);
  const col = index % COLUMNS;

  return [row, col];
}

// Gets the row and column of an element in the terminal grid
function getElementPosition(el) {
  const index = el.id.split("char")[1];

  return indexToRowAndColumn(index);
}

// Clear the terminal
function clearTerminal() {
  for (const el of outputGrid.value.children) {
    el.textContent = "";
  }
}

// Find a control code by its char code, returns the key of the code in `controlCodes` if it was found, otherwise returns null
function findCodeByCharCode(charCode) {
  for (const [key, value] of Object.entries(controlCodes)) {
    if (value.char === charCode) {
      return key;
    }
  }
  return null;
}

// Run at the end of every line
function onNewLine() {
  textMode = controlCodes.ALPHANUMERIC_WHITE;
  heightMode = controlCodes.NORMAL_HEIGHT;
  mosaicsMode = controlCodes.CONTIGUOUS_MOSAICS;
}

// Get a mosaic character using the number sent by the server
function getMosaicChar(code) {
  // The 6th bit is used for something else which I don't understand at the moment, so for now it just gets ignored
  const maskedCode = code & 0b01011111;
  switch (mosaicsMode) {
    case controlCodes.CONTIGUOUS_MOSAICS:
      return String.fromCodePoint(maskedCode + CONTIGUOUS_MOSAICS_OFFSET);
    case controlCodes.SEPARATED_MOSAICS:
      return String.fromCodePoint(maskedCode + SEPARATED_MOSAICS_OFFSET);
    default:
      return null;
  }
}

function parseResponse(response) {
  // Don't try to parse messages from the bridge
  if (response.startsWith(BRIDGE_MSG_PREFIX)) {
    console.log(response);
    return "";
  }

  let json = JSON.parse(response);
  let nextIsControlCode = false;
  // The current row and column of the "cursor"
  let cursor = [0, 0];
  for (const i in json) {
    // The response is received as a JSON array of character codes as numbers which need to have the parity bit removed, then converted into a string
    const withoutParity = json[i] & 0b01111111;

    // Don't try to display escape/control codes
    if (withoutParity === controlCodes.ESCAPE.char) {
      console.log("found escape code");
      nextIsControlCode = true;
      continue;
    } else if (nextIsControlCode) {
      nextIsControlCode = false;
      // Add the correct classes to the next elements
      const code = controlCodes[findCodeByCharCode(withoutParity)]
      if (code) {
        switch (code.type) {
          case "mosaic":
          case "text": textMode = code; break;
          case "height": heightMode = code; break;
          case "mosaicsMode": mosaicsMode = code; break;
        }
      } else {
        console.error("Found unknown escape code: 0x" + withoutParity.toString(16));
      }
      continue;
    }

    // Each character has its own <span> element which is created when the page loads
    let charSpan = document.getElementById("char" + rowAndColumnToIndex(cursor[0], cursor[1]));

    charSpan.dataset.textmode = textMode.name;
    charSpan.dataset.heightmode = heightMode.name;

    let charAsString;
    // If the current mode is mosaic, get the mosaic character instead of just converting to a string
    if (textMode.type === "mosaic") {
      charAsString = getMosaicChar(withoutParity);
      if (!charAsString) {
        console.error("getMosaicChar returned null\n Current textMode is: " + textMode.char.toString(16));
      }
    } else {
      charAsString = String.fromCharCode(withoutParity);
    }

    charSpan.innerText = charAsString;

    // Reset formatting at the end of a line
    if (charAsString === "\n") {
      onNewLine();
      // Add one to the current row
      cursor[0]++;
      // Set the column to 0
      cursor[1] = 0;
    }

    // Update cursor position and wrap if needed
    if (cursor[1] === COLUMNS - 1) {
      // Wrap the cursor position
      // Add one to the current row
      cursor[0]++;
      // Set the column to 0
      cursor[1] = 0;

      onNewLine();
    } else {
      // Add one to the current column
      cursor[1]++;
    }
  }
}

function connect() {
  ws = new WebSocket(BRIDGE_ADDR);
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
  clearTerminal();
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
    // We have to check the length of the key, otherwise things like "Enter", and "Shift" get sent too
    if (connected && event.key.length === 1) {
      ws.send(event.key);
    }
  });

  // Create the <span> elements for each character
  for (let i = 0; i < ROWS * COLUMNS; i++) {
    const element = document.createElement("span");

    element.id = "char" + i;
    element.dataset.textmode = textMode.name
    element.dataset.heightmode = heightMode.name;

    outputGrid.value.appendChild(element);
  }
});
</script>

<template>
  <div>
    <div ref="outputGrid" id="outputGrid"></div>
    <br>
    <button ref="toggleConnectedButton" @click="toggleConnected">Connect</button>
  </div>
</template>