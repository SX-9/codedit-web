<script setup>
import { ref } from "vue";
import { Codemirror } from "vue-codemirror";
import { javascript } from "@codemirror/lang-javascript";
import { oneDark } from "@codemirror/theme-one-dark";
import { WebContainer } from '@webcontainer/api';
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

  const jsFile = await container.fs.readFile('index.js', 'utf-8');
  code.value = jsFile;
  document.getElementById('term').innerHTML = 'WebContainer Booted, Terminal Output Here';

  container.on('server-ready', (port, url) => {
    alert('Server Opened Port ' + port);
    webapp.value = url;
  });
});

async function getInput(msg) {
  return new Promise((resolve) => {
    const input = window.prompt(msg);
    resolve(input);
  });
}
async function install(...deps) {
  if (!deps) deps = await getInput('Enter Dependencies To Install:').split(' ');
  let installProcess = await container.spawn('pnpm', ['install', ...deps]);
  installProcess.output.pipeTo(new WritableStream({
    write(data) {
      console.log(data);
    }
  }));
  return installProcess.exit;
}
async function startServer() {
  let server = await container.spawn('node', ['index.js']);

  document.getElementById('term').innerHTML = '';
  server.output.pipeTo(new WritableStream({
    write(data) {
      document.getElementById('term').innerHTML += data;
    }
  }));
}
async function save(e) {
  await container.fs.writeFile('index.js', e);
}
async function exec() {
  let cmd = prompt('Enter Command To Execute:').split(' ');
  let server = await container.spawn(cmd[0], cmd.slice(1));

  document.getElementById('term').innerHTML = '';
  server.output.pipeTo(new WritableStream({
    write(data) {
      document.getElementById('term').innerHTML += data;
    }
  }));
}
</script>

<template>
  <div id="workspace">
    <div class="container bar">
      <p><a @click="startServer">Run Dev Server</a> | <a @click="install">Install Deps</a> | <a @click="exec">Run Command</a></p>
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
    <div class="container"><div id="term">
      Booting WebContainer...
    </div></div>
    <div class="container">
      <iframe id="preview" :src="webapp" frameborder="0"></iframe>
    </div>
  </div>
</template>

<style>
#workspace {
  width: 100%;
  height: 100%;
  display: grid;
  grid-gap: .01em;
  grid-template-columns: 30% 40% 30%;
  grid-template-rows: 1.5rem 1fr;
  grid-template-areas:
    'bar bar bar'
    'edit term prev';
}

#term {
  font-family: 'Courier New', Courier, monospace;
  overflow: scroll;
  word-wrap: break-word;
  z-index: -1;
}

.container:not(.bar), .container:not(.bar) > * {
  height: 100%;
  width: 100%;
  max-width: 100%;
  max-height: 100%;
}

.container.bar {
  grid-area: bar;
  width: 100%;
  background-color: #585858;
  display: flex;
  justify-content: center;
}
.container.bar > * {
  padding: 0;
  margin: 0;
}

@media (max-width: 600px) {
  #workspace {
    grid-template-columns: 1fr;
    grid-template-rows: 1.5rem repeat(3, 1fr);
    grid-template-areas: 
      "bar"
      "edit"
      "term"
      "prev";
  }
}
</style>
