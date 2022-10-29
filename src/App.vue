<script setup lang="ts">
import { reactive, onMounted } from 'vue'
import Fuse from 'fuse.js'
const fs = require('fs')
const path = require('path')
const Jimp = require('jimp')
const jsonfile = require('jsonfile')
const screenshot = require('screenshot-desktop')
const { ipcRenderer } = require('electron')
const { windowManager } = require('node-window-manager')
// 状态
const state = reactive({
  working: false,
  result: new Set(),
  bounds: { x: 0, y: 0, width: 0, height: 0 },
})
// 导入文本数据
let textData = [] as any[]
const filePath = path.resolve('./data')
fs.readdir(filePath, (err: any, files: any[]) => {
  files.forEach((file: string, index) => {
    jsonfile.readFile(filePath + '/' + file, (err: any, obj: any[]) => {
      textData = textData.concat(obj)
      if (index === files.length - 1) {
        ready()
      }
    })
  })
})
// 初始化搜索引擎
let fuse: Fuse<any> | null = null
const ready = () => {
  fuse = new Fuse(textData, {
    keys: ['ori'],
    threshold: 0.2,
    includeScore: true,
    fieldNormWeight: 2,
  })
}
// 获取游戏窗口初始化状态
onMounted(async () => {
  const window = windowManager.getWindows()
  const target = window.find((w: { path: string }) => w.path.includes('HeavenBurnsRed'))
  const bounds = target.getBounds()
  state.bounds = {
    x: bounds.x * 1.5,
    y: bounds.y * 1.5,
    width: bounds.width * 1.5,
    height: bounds.height * 1.5
  }
})
// 截图
const takeScreenshot = () => {
  screenshot({filename: './output/shot.png'}).then(async (img: any) => {
    const image = await Jimp.read(img)
    image.crop(state.bounds.x, state.bounds.y, state.bounds.width, state.bounds.height)
      .grayscale()
      .contrast(+1)    
      .normalize()
      .invert()
      .write('./output/shot-opt.png')
    setTimeout(() => {
      ipcRenderer.send('ocr-send', img.replace('shot', 'shot-opt'))
    }, 50)
  })
}
// 获取OCR结果
ipcRenderer.on('ocr-reply', (_event, arg) => {
  console.log('获取ocr')
  console.log(arg)
  let newData = [] as string[]
  arg?.Data?.forEach((item: { Score: number; Words: any }) => {
    if (item.Words.length > 1 && item.Score > 0.8 && fuse) {
      const search = fuse.search(item.Words, { limit: 1 })
      console.log('搜寻 ' + item.Words)
      console.log(search)
      if (search.length) {
        if (item.Words.length <= 5 && search[0].item.trans.length >= 10) {
          return
        }
        newData.push(search[0].item.trans)
      }
    }
  })
  state.result.clear()
  newData.forEach((item) => {
    state.result.add(item)
  })
  if (state.working) {
    takeScreenshot()
  }
})
// 开关
const changeWorking = () => {
  state.working = !state.working
  if (state.working) {
    takeScreenshot()
  }
}
</script>

<template>
  <div class="drag"></div>
  <div @click="changeWorking()" class="content">
    <p v-if="!state.working">开始运行</p>
    <div class="translate" v-else>
      <p v-for="item in state.result">{{item}}</p>
    </div>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  height: 100vh;
  background: rgba(0, 0, 0, 0.2);
}
body::-webkit-scrollbar {
  width: 10px;
  background: rgba(0, 0, 0, 0.2);
}
body::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 100px;
  background-clip: padding-box;
  border: 2px solid hsla(0,0%,100%,0);
  min-height: 10px;
}
.drag {
  -webkit-app-region: drag;
  position: fixed;
  top: 0;
  width: 100%;
  height: 30px;
}
.content {
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  flex-wrap: wrap;
  padding: 0.5em;
  min-height: 100vh;
}
.translate {
  text-align: center;
}
p {
  text-align: left;
  text-shadow: 0 0 2px #000;
  background: rgba(0, 0, 0, 0.5);
  padding: 0.3em 0.5em;
  margin: 0.2em;
  border-radius: 0.2em;
  width: fit-content;
  display: inline-block;
}
</style>
