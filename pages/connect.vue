<script setup>
import "~/assets/css/terminal.css";
import { useTemplateRef } from "vue";

// Control codes
const controlCodes = {
  "ESCAPE": { char: 0x1B },
  "ALPHANUMERIC_RED": { className: "alphanumeric-red", char: 0x41, type: "colour" },
  "ALPHANUMERIC_GREEN": { className: "alphanumeric-green", char: 0x42, type: "colour" },
  "ALPHANUMERIC_YELLOW": { className: "alphanumeric-yellow", char: 0x43, type: "colour" },
  "ALPHANUMERIC_BLUE": { className: "alphanumeric-blue", char: 0x44, type: "colour" },
  "ALPHANUMERIC_MAGENTA": { className: "alphanumeric-magenta", char: 0x45, type: "colour" },
  "ALPHANUMERIC_CYAN": { className: "alphanumeric-cyan", char: 0x46, type: "colour" },
  "ALPHANUMERIC_WHITE": { className: "alphanumeric-white", char: 0x47, type: "colour" },
  "FLASHING": { className: "flashing", char: 0x48 },
  "STEADY": { className: "steady", char: 0x49 },
  "END_EDIT": { char: 0x4A },
  "START_EDIT": { char: 0x4B },
  "NORMAL_HEIGHT": { className: "normal-height", char: 0x4C, type: "height" },
  "DOUBLE_HEIGHT": { className: "double-height", char: 0x4D, type: "height" },
  "MOSAICS_RED": { className: "mosaics-red", char: 0x51, type: "colour" },
  "MOSAICS_GREEN": { className: "mosaics-green", char: 0x52, type: "colour" },
  "MOSAICS_YELLOW": { className: "mosaics-yellow", char: 0x53, type: "colour" },
  "MOSAICS_BLUE": { className: "mosaics-blue", char: 0x54, type: "colour" },
  "MOSAICS_MAGENTA": { className: "mosaics-magenta", char: 0x55, type: "colour" },
  "MOSAICS_CYAN": { className: "mosaics-cyan", char: 0x56, type: "colour" },
  "MOSAICS_WHITE": { className: "mosaics-white", char: 0x57, type: "colour" },
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
let heightMode = controlCodes.NORMAL_HEIGHT;

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
      // Add the correct classes to the next elements
      const code = controlCodes[findCodeByCharCode(json[i])]
      if (code) {
        switch (code.type) {
          case "colour": currentColour = code; break;
          case "height": heightMode = code; break;
        }
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
    charSpan.classList.add(heightMode.className);

    const charAsString = String.fromCharCode(withoutParity);
    charSpan.innerText = charAsString;
    outputContainer.value.appendChild(charSpan);

    // Reset formatting at the end of a line
    if (charAsString === "\n") {
      currentColour = controlCodes.ALPHANUMERIC_WHITE;
      heightMode = controlCodes.NORMAL_HEIGHT;
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