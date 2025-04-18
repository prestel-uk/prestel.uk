<script setup>
import "~/assets/css/terminal.css";
import { useTemplateRef } from "vue";

// Control codes
const controlCodes = {
  "ESCAPE": { char: 0x1B },
  "ALPHANUMERIC_RED": { className: "alphanumeric-red", char: 0x41 },
  "ALPHANUMERIC_GREEN": { className: "alphanumeric-green", char: 0x42 },
  "ALPHANUMERIC_YELLOW": { className: "alphanumeric-yellow", char: 0x43 },
  "ALPHANUMERIC_BLUE": { className: "alphanumeric-blue", char: 0x44 },
  "ALPHANUMERIC_MAGENTA": { className: "alphanumeric-magenta", char: 0x45 },
  "ALPHANUMERIC_CYAN": { className: "alphanumeric-cyan", char: 0x46 },
  "ALPHANUMERIC_WHITE": { className: "alphanumeric-white", char: 0x47 },
  "FLASHING": { className: "flashing", char: 0x48 },
  "STEADY": { className: "steady", char: 0x49 },
  "END_EDIT": { char: 0x4A },
  "START_EDIT": { char: 0x4B },
  "NORMAL_HEIGHT": { char: 0x4C },
  "DOUBLE_HEIGHT": { className: "double-height", char: 0x4D },
  "MOSAICS_RED": { className: "mosaics-red", char: 0x51 },
  "MOSAICS_GREEN": { className: "mosaics-green", char: 0x52 },
  "MOSAICS_YELLOW": { className: "mosaics-yellow", char: 0x53 },
  "MOSAICS_BLUE": { className: "mosaics-blue", char: 0x54 },
  "MOSAICS_MAGENTA": { className: "mosaics-magenta", char: 0x55 },
  "MOSAICS_CYAN": { className: "mosaics-cyan", char: 0x56 },
  "MOSAICS_WHITE": { className: "mosaics-white", char: 0x57 },
  "CONCEAL_DISPLAY": { char: 0x58 },
  "CONTIGUOUS_MOSAICS": { className: "contiguous-mosaics", char: 0x59 },
  "SEPARATED_MOSAICS": { className: "separated-mosaics", char: 0x5A },
  "BLACK_BACKGROUND": { className: "black-background", char: 0x5C },
  "NEW_BACKGROUND": { className: "new-background", char: 0x5D },
  "HOLD_MOSAICS": { char: 0x5E },
  "RELEASE_MOSAICS": { char: 0x5F },
};

// The prefix for messages coming from the bridge and not Genesis
const bridge_msg_prefix = "[bridge] ";
// The address of the bridge (https://github.com/prestel-uk/web-bridge) to connect to
const bridge_addr = "ws://127.0.0.1:1234";

const outputContainer = useTemplateRef("outputContainer");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let ws;
let currentColour = controlCodes.ALPHANUMERIC_WHITE;

// Find a control code by its char code, returns the key of the code in `controlCodes` if it was found, otherwise returns null
function findCodeByCharCode(charCode) {
  for (const [key, value] of Object.entries(controlCodes)) {
    if (value.char === charCode) {
      return key;
    }
  }
  return null;
}

function parseResponse(response) {
  // Don't try to parse messages from the bridge
  if (response.startsWith(bridge_msg_prefix)) {
    console.log(response);
    return "";
  }

  let json = JSON.parse(response);
  let nextIsControlCode = false;
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
      // Add the correct class to the next elements
      const code = controlCodes[findCodeByCharCode(json[i])]
      if (code) {
        currentColour = code;
      } else {
        console.error("Found unknown escape code: 0x" + json[i].toString(16));
      }
      continue;
    }

    // Create a new <span> element with a unique id so that it can be referenced later, and set the content to the character. If a span already exists for that character, use that instead.
    let charSpan = document.getElementById("char" + i);
    if (!charSpan) {
      charSpan = document.createElement("span");
    }

    charSpan.id = "char" + i;
    charSpan.classList.add(currentColour.className);
    const charAsString = String.fromCharCode(withoutParity);
    charSpan.innerText = charAsString;
    outputContainer.value.appendChild(charSpan);

    // Reset colours at the end of a line
    if (charAsString === "\n") {
      currentColour = controlCodes.ALPHANUMERIC_WHITE;
    }
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