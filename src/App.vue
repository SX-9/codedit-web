<script setup>
import { ref } from "vue";
import { Codemirror } from "vue-codemirror";
import { javascript } from "@codemirror/lang-javascript";
import { oneDark } from "@codemirror/theme-one-dark";
import { WebContainer } from "@webcontainer/api";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import { common } from "xterm-style";
import { VueWinBox } from "vue-winbox";

console.clear();
const extensions = [javascript(), oneDark];
const webapp = ref("loading.html");
const port = ref(0);
const code = ref("");
const w = ref(window);

let container;

window.addEventListener("load", async () => {
  document.querySelectorAll(".wb-close").forEach((b) => b.remove());
  document.querySelectorAll(".wb-full").forEach((b) => b.remove());
  document.querySelectorAll(".wb-min").forEach((b) => b.click());

  container = await WebContainer.boot();
  await container.mount({
    "index.js": {
      file: {
        contents: `\
// Welcome To CodedIt WebContainers!

console.log('hi nodejs ' + process.version);`,
      },
    },
    "http.js": {
      file: {
        contents: `\
const app = require('express')();
app.get('/', (req, res) => res.send('hi from http.js'));
app.listen(3000, () => console.log('example server running...'));
        `,
      },
    },
    "package.json": {
      file: {
        contents: `\
{
  "name": "codedit",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
        `,
      },
    },
  });

  const fitAddon = new FitAddon();
  const terminal = new Terminal({
    convertEol: true,
    theme: common,
  });
  terminal.loadAddon(fitAddon);
  terminal.open(document.getElementById("term"));
  fitAddon.fit();

  const jsFile = await container.fs.readFile("index.js", "utf-8");
  code.value = jsFile;

  const shellProcess = await shell(terminal);
  window.addEventListener('resize', () => {
    fitAddon.fit();
    shellProcess.resize({
      cols: terminal.cols,
      rows: terminal.rows,
    });
  });
  container.on('server-ready', (portH, url) => {
    alert('Server Opened Port ' + portH);
    port.value = portH;
    webapp.value = url;
  });
  container.on("error", console.error);

  const i = await container.spawn("npm", ["i"]);
  i.output.pipeTo(
    new WritableStream({
      write(data) {
        console.log(data);
      },
    })
  );

  document.querySelector("#loading").remove();
});

async function save(e) {
  await container.fs.writeFile("index.js", e);
}
async function shell(t) {
  let shellP = await container.spawn("jsh", {
    env: {
      SHELL: "bash",
      TERM_PROGRAM: "codedit-term",
    },
    terminal: {
      cols: t.columns,
      rows: t.rows,
    },
  });

  t.write(`Try Typing "node index.js"\x1b[1;36m
  ____          _          _ ___ _   
 / ___|___   __| | ___  __| |_ _| |_ 
| |   / _ \\ / _\` |/ _ \\/ _\` || || __|
| |__| (_) | (_| |  __/ (_| || || |_ 
 \\____\\___/ \\__,_|\\___|\\__,_|___|\\__|
\x1b[0m
`);
  shellP.output.pipeTo(
    new WritableStream({
      write(data) {
        t.write(data);
      },
    })
  );

  const input = shellP.input.getWriter();
  t.onData((data) => {
    input.write(data);
  });

  return shellP;
}
</script>

<template>
  <div id="workspace">
    <div class="bar">
      <a href="https://github.com/SX-9/codedit-web">Github</a>
      <a v-if="port" @click="w.open(webapp)">Open Port {{ port }}</a>
    </div>
    <div id="term"></div>
    <VueWinBox :options="{ title: 'Code Editor', y: 'bottom', x: 'right' }">
      <Codemirror
        v-model="code"
        placeholder="Code Here..."
        @change="save($event)"
        :autofocus="true"
        :indent-with-tab="true"
        :tab-size="2"
        :extensions="extensions"
      />
    </VueWinBox>
    <VueWinBox :options="{ title: 'Web Browser', y: 'bottom', x: 'left' }"
      ><iframe :src="webapp"></iframe
    ></VueWinBox>
  </div>
  <div id="loading">
    <noscript
      >ERROR: CANT RUN JAVASCRIPT<br />PLEASE TRY ANOTHER BROWSER</noscript
    >
    <div id="loading-logo"></div>
    <h2>Booting WebContainer...</h2>
    <p>
      Get Stuck? Try Refreshing! <br /><a
        href="https://webcontainers.io/guides/troubleshooting"
        >Troubleshoot</a
      >
      - <a href="https://sx9.is-a.dev">Made By Me</a>
    </p>
  </div>
</template>

<style>
#loading {
  cursor: progress;
  background: #000000b9;
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 100;
}
#loading > * {
  margin: 0.3em;
}
#loading > h2 {
  animation: blink 750ms infinite;
}

#workspace {
  overflow: hidden;
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

#term {
  height: 100%;
  margin: 0;
  padding: 0;
  font-family: "Courier New", Courier, monospace;
  overflow: hidden;
  word-wrap: break-word;
  width: 100%;
}

#loading-logo {
  width: 3em;
  height: 3em;
  border-radius: 50%;
  border: 0.3em solid white;
  border-top: 0.3em solid black;
  border-bottom: 0.3em solid black;
  animation: spin 500ms infinite;
}

.bar {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 2em;
  background: #272727;
}
.bar > * {
  padding: 0;
  margin: 0;
}

.winbox,
.wb-body {
  border-radius: 0.5em;
}
.winbox {
  background-color: #384a6db9;
}
.wb-body {
  margin: 0.5em;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: black;
}
.wb-body .cm-editor {
  position: relative;
  top: 0;
  left: 0;
  height: 100%;
  width: 100%;
}

noscript {
  color: red;
}

@keyframes spin {
  from {
    transform: rotate(360deg);
  }
  to {
    transform: rotate(0deg);
  }
}

@keyframes blink {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}
</style>
