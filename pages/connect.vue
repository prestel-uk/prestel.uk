<script setup>
import "~/assets/css/terminal.css";
import { useTemplateRef } from "vue";

// Control codes
const controlCodes = {
  "ESCAPE": { char: 0x1B },
  "ALPHANUMERIC_RED": { className: "red", char: 0x41, type: "text" },
  "ALPHANUMERIC_GREEN": { className: "green", char: 0x42, type: "text" },
  "ALPHANUMERIC_YELLOW": { className: "yellow", char: 0x43, type: "text" },
  "ALPHANUMERIC_BLUE": { className: "blue", char: 0x44, type: "text" },
  "ALPHANUMERIC_MAGENTA": { className: "magenta", char: 0x45, type: "text" },
  "ALPHANUMERIC_CYAN": { className: "cyan", char: 0x46, type: "text" },
  "ALPHANUMERIC_WHITE": { className: "white", char: 0x47, type: "text" },
  "FLASHING": { className: "flashing", char: 0x48 },
  "STEADY": { className: "steady", char: 0x49 },
  "END_EDIT": { char: 0x4A },
  "START_EDIT": { char: 0x4B },
  "NORMAL_HEIGHT": { className: "normal-height", char: 0x4C, type: "height" },
  "DOUBLE_HEIGHT": { className: "double-height", char: 0x4D, type: "height" },
  "MOSAICS_RED": { className: "red", char: 0x51, type: "mosaic" },
  "MOSAICS_GREEN": { className: "green", char: 0x52, type: "mosaic" },
  "MOSAICS_YELLOW": { className: "yellow", char: 0x53, type: "mosaic" },
  "MOSAICS_BLUE": { className: "blue", char: 0x54, type: "mosaic" },
  "MOSAICS_MAGENTA": { className: "magenta", char: 0x55, type: "mosaic" },
  "MOSAICS_CYAN": { className: "cyan", char: 0x56, type: "mosaic" },
  "MOSAICS_WHITE": { className: "white", char: 0x57, type: "mosaic" },
  "CONCEAL_DISPLAY": { char: 0x58 },
  "CONTIGUOUS_MOSAICS": { className: "contiguous-mosaics", char: 0x59, type: "mosaicsMode" },
  "SEPARATED_MOSAICS": { className: "separated-mosaics", char: 0x5A, type: "mosaicsMode" },
  "BLACK_BACKGROUND": { className: "black-background", char: 0x5C },
  "NEW_BACKGROUND": { className: "new-background", char: 0x5D },
  "HOLD_MOSAICS": { char: 0x5E },
  "RELEASE_MOSAICS": { char: 0x5F },
};

// The offset of contiguous mosaics in the font
const contiguousMosaicsOffset = 57856;
// The offset of separated mosaics in the font
const separatedMosaicsOffset = 58048;

// The prefix for messages coming from the bridge and not Genesis
const bridge_msg_prefix = "[bridge] ";
// The address of the bridge (https://github.com/prestel-uk/web-bridge) to connect to
const bridge_addr = "ws://127.0.0.1:1234";

const outputContainer = useTemplateRef("outputContainer");
const toggleConnectedButton = useTemplateRef("toggleConnectedButton");

let connected = false;
let ws;

let textMode = controlCodes.ALPHANUMERIC_WHITE;
let heightMode = controlCodes.NORMAL_HEIGHT;
let mosaicsMode = controlCodes.CONTIGUOUS_MOSAICS;

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
      return String.fromCodePoint(maskedCode + contiguousMosaicsOffset);
    case controlCodes.SEPARATED_MOSAICS:
      return String.fromCodePoint(maskedCode + separatedMosaicsOffset);
    default:
      return null;
  }
}

function parseResponse(response) {
  // Don't try to parse messages from the bridge
  if (response.startsWith(bridge_msg_prefix)) {
    console.log(response);
    return "";
  }

  let json = JSON.parse(response);
  let nextIsControlCode = false;
  // How many chars until a new line, so when this is 1, the next character should be a new line
  let charsUntilNewLine = 40;
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

    // Create a new <span> element with a unique id so that it can be referenced later, and set the content to the character. If a span already exists for that character, use that instead.
    let charSpan = document.getElementById("char" + i);
    if (!charSpan) {
      charSpan = document.createElement("span");
    }

    charSpan.id = "char" + i;

    charSpan.classList.add(textMode.className);
    charSpan.classList.add(heightMode.className);

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
    outputContainer.value.appendChild(charSpan);

    // Reset formatting at the end of a line
    if (charAsString === "\n") {
      onNewLine();
      charsUntilNewLine = 40;
    } else if (charsUntilNewLine === 0) {
      onNewLine();
      charSpan.innerText += "\n";
      charsUntilNewLine = 40;
    } else {
      charsUntilNewLine--;
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
    // We have to check the length of the key, otherwise things like "Enter", and "Shift" get sent too
    if (connected && event.key.length === 1) {
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