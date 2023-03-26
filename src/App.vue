<script setup>
import { ref } from "vue";
import { Codemirror } from "vue-codemirror";
import { javascript } from "@codemirror/lang-javascript";
import { oneDark } from "@codemirror/theme-one-dark";
import { WebContainer } from '@webcontainer/api';
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
console.clear();
const extensions = [javascript(), oneDark];
const webapp = ref("loading.html");
const code = ref('');

let container;

window.addEventListener('load', async () => {
  container = await WebContainer.boot();
  await container.mount({
    'index.js': {
      file: {
        contents: `\
console.log('hi');
        `
      }
    },
    'express-example.js': {
      file: {
        contents: `\
const app=require('express');
app.get('/',(req,res)=>res.send('hi'));
app.listen(3000,()=>console.log('hi'));
        `
      }
    },
    'package.json': {
      file: {
        contents: `\
{
  "name": "codedit",
  "version": "1.0.0"
}
        `
      }
    }
  });

  const fitAddon = new FitAddon();
  const terminal = new Terminal({
    convertEol: true,
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
  container.on('server-ready', (port, url) => {
    alert('Server Opened Port ' + port);
    window.open(url);
    webapp.value = url;
  });
  container.on('error', console.error);
});

async function save(e) {
  await container.fs.writeFile('index.js', e);
}
async function shell(t) {
  let shellP = await container.spawn('jsh', {
    terminal: {
      cols: t.cols,
      rows: t.rows,
    },
  });

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
      <p>CodedIt WebContainers</p>
    </div>
    <Codemirror
      class="container"
      v-model="code"
      placeholder="console.log('hi');"
      @change="save($event)"
      :autofocus="true"
      :indent-with-tab="true"
      :tab-size="2"
      :extensions="extensions"
    />
    <div class="container"><div id="term"></div></div>
  </div>
</template>

<style>
#workspace {
  overflow: hidden;
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  display: grid;
  grid-gap: .01em;
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
  overflow-x: hidden;
  overflow-y: scroll;
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
  background-color: black;
  display: flex;
  justify-content: center;
  align-items: center;
}
.container.bar > * {
  padding: 0;
  margin: 0;
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
</style>
