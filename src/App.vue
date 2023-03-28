<script setup>
import { ref } from "vue";
import { Codemirror } from "vue-codemirror";
import { javascript } from "@codemirror/lang-javascript";
import { oneDark } from "@codemirror/theme-one-dark";
import { WebContainer } from '@webcontainer/api';
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import { common } from 'xterm-style';
console.clear();
const extensions = [javascript(), oneDark];
const webapp = ref("loading.html");
const port = ref(0);
const code = ref('');
const w = ref(window);

let container;

window.addEventListener('load', async () => {
  container = await WebContainer.boot();
  await container.mount({
    'index.js': {
      file: {
        contents: `\
// Welcome To CodedIt WebContainers!

console.log('hi nodejs ' + process.version);`
      }
    },
    'http.js': {
      file: {
        contents: `\
const app = require('express')();
app.get('/', (req, res) => res.send('hi'));
app.listen(3000, () => console.log('example server running...'));
        `
      }
    },
    'package.json': {
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
        `
      }
    }
  });

  const fitAddon = new FitAddon();
  const terminal = new Terminal({
    convertEol: true,
    theme: common,
  });
  terminal.loadAddon(fitAddon);
  terminal.open(document.getElementById('term'));
  fitAddon.fit();

  const jsFile = await container.fs.readFile('index.js', 'utf-8');
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
    alert('Server Opened Port ' + portH + ', Opening In New Tab...');
    port.value = portH;
    window.open(url);
    webapp.value = url;
  });
  container.on('error', console.error);
  
  const i = await container.spawn('npm', ['i']);
  i.output.pipeTo(new WritableStream({
    write(data) {
      console.log(data);
    }
  }));

  document.querySelector('#loading').remove();
});

async function save(e) {
  await container.fs.writeFile('index.js', e);
}
async function shell(t) {
  let shellP = await container.spawn('jsh', {
    env: {
      SHELL: 'bash',
      TERM_PROGRAM: 'codedit-term',
    },
    terminal: {
      cols: t.cols,
      rows: t.rows,
    },
  });

  t.write(`Try Typing "node index.js"
  
`);
  shellP.output.pipeTo(new WritableStream({
    write(data) {
      t.write(data);
    }
  }));

  const input = shellP.input.getWriter();
  t.onData((data) => {
    input.write(data);
  });

  return shellP;
}
</script>

<template>
  <div id="workspace">
    <div class="container bar">
    <a href="https://github.com/SX-9/codedit-web">Source Code</a>
    <a v-if="port" @click="w.open(webapp)" >Open Port {{ port }}</a>
    </div>
    <Codemirror
      class="container"
      v-model="code"
      placeholder="Code Here..."
      @change="save($event)"
      :autofocus="true"
      :indent-with-tab="true"
      :tab-size="2"
      :extensions="extensions"
    />
    <div class="container"><div id="term"></div></div>
  </div>
  <div id="loading">
    <noscript>ERROR: CANT RUN JAVASCRIPT<br>PLEASE TRY ANOTHER BROWSER</noscript>
    <h2>Booting WebContainer...</h2>
    <p>Get Stuck? Try Refreshing!
      <br><a href="https://webcontainers.io/guides/troubleshooting">Troubleshoot</a> 
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
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  text-align: center;
}
#loading > * {
  margin: .3em;
}
#loading > h2 {
  animation: blink 750ms infinite;
}

#workspace {
  overflow: hidden;
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  display: grid;
  grid-gap: .05em;
  grid-template-columns: 50% 50%;
  grid-template-rows: 4% 96%;
  grid-template-areas:
    'bar bar'
    'edit term';
}

#term {
  margin: 0;
  padding: 0;
  font-family: 'Courier New', Courier, monospace;
  overflow: hidden;
  word-wrap: break-word;
}

.container:not(.bar), .container:not(.bar) > * {
  height: 100%;
  width: 100%;
  max-width: 100%;
  max-height: 100%;
}

.container.bar {
  grid-area: bar;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 2em;
}
.container.bar > * {
  padding: 0;
  margin: 0;
}

noscript {
  color: red;
}

@media (max-width: 600px) {
  #workspace {
    grid-template-columns: 100%;
    grid-template-rows: 4% 40% 56%;
    grid-template-areas: 
      "bar"
      "edit"
      "term";
  }
}

@keyframes blink {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}
</style>
