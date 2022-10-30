<script setup lang="ts">
import { reactive, onMounted, nextTick } from 'vue'
import Fuse from 'fuse.js'
const fs = require('fs')
const path = require('path')
const Jimp = require('jimp')
const jsonfile = require('jsonfile')
const screenshot = require('screenshot-desktop')
const { ipcRenderer } = require('electron')
const { windowManager } = require('node-window-manager')
// 状态
interface State {
  working: boolean
  result: string[]
  bounds: {
    x: number
    y: number
    width: number
    height: number
  }
  error: string
}
const state: State = reactive({
  working: false,
  result: [],
  bounds: { x: 0, y: 0, width: 0, height: 0 },
  error: '',
})
// 导入文本数据
let textData = [] as any[]
const filePath = path.resolve('./data')
fs.readdir(filePath, (err: any, files: any[]) => {
  if (!files || !files.length) {
    state.error = '未检测到译文数据，请在软件根目录添加data文件夹并放置译文数据，然后点击本窗口重试'
    return
  }
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
  if (!target) {
    state.error = '未检测到游戏窗口，请打开游戏后点击本窗口重试'
    return
  }
  const bounds = target.getBounds()
  console.log(bounds)
  if (bounds.x < 0 || bounds.y < 0) {
    state.error = '获取游戏窗口位置失败，请保证游戏窗口可见并点击本窗口重试'
    return
  }
  console.log(target)
  state.bounds = {
    x: bounds.x * 1.5,
    y: bounds.y * 1.5,
    width: bounds.width * 1.5,
    height: bounds.height * 1.5
  }
})
// 截图
const takeScreenshot = () => {
  console.log('准备截图')
  screenshot({filename: './output/shot.png'}).then(async (img: any) => {
    console.log('截图完成' + img)
    const image = await Jimp.read(img)
    image.crop(state.bounds.x, state.bounds.y, state.bounds.width, state.bounds.height)
      .grayscale()
      .contrast(+1)    
      .normalize()
      .invert()
      .write('./output/shot-opt.png')
    console.log('图片处理完成')
    setTimeout(() => {
      console.log('开始识别')
      ipcRenderer.send('ocr-send', img.replace('shot', 'shot-opt'))
      console.log('发送识别请求')
    }, 50)
  })
}
// 获取OCR结果
ipcRenderer.on('ocr-reply', (_event, arg) => {
  console.log('获取ocr')
  console.log(arg)
  if (!arg) {
    state.error = 'OCR未返回结果，请检查团子OCR是否正常运行，然后点击本窗口重试'
    return
  }
  if (arg.Data.length >= 20) {
    if (!state.result.includes('OCR返回结果过多，因此跳过本次识别')) {
      state.result.push('OCR返回结果过多，因此跳过本次识别')
    }
    if (state.working) {
      takeScreenshot()
    }
    return
  }
  arg.Data.forEach((item: { Score: number; Words: any }) => {
    if (item.Words.length > 1 && item.Score > 0.8 && fuse) {
      const search = fuse.search(item.Words, { limit: 1 })
      console.log('搜寻 ' + item.Words)
      console.log(search)
      if (search.length) {
        if (item.Words.length <= 5 && search[0].item.trans.length >= 10) {
          return
        }
        if (state.result.length >= 10) {
          state.result.shift()
        }
        if (!state.result.includes(search[0].item.trans)) {
          state.result.push(search[0].item.trans)
          nextTick(() => {
            window.scrollTo(0, 9999)
          })
        }
      }
    }
  })
  if (state.working) {
    takeScreenshot()
  }
})
// 获取OCR失败
ipcRenderer.on('ocr-error', (_event, arg) => {
  state.error = '请求OCR失败，请确保团子OCR已启动，且端口号为6666，若确认无误，请尝试使用任务管理器关闭所有python进程后重新打开团子OCR，然后点击本窗口重试'
})
// 开关
const changeWorking = () => {
  state.working = !state.working
  if (state.working) {
    takeScreenshot()
  }
}
const reload = () => {
  location.reload()
}
</script>

<template>
  <div class="drag"></div>
  <div v-if="!state.error" @click="changeWorking()" class="content">
    <div v-if="!state.working">
      <p>使用须知：</p>
      <p>本程序暂不自带OCR功能，你需要先启动团子翻译器的本地OCR后再使用</p>
      <p>点击窗口任意位置开始运行，再次点击则停止运行</p>
      <p>拖动窗口上方可调整窗口位置，拖动窗口四周可调整大小</p>
      <p>可通过在窗口上方位置右击关闭本程序</p>
    </div>
    <div class="translate" v-else>
      <p v-for="item in state.result">{{item}}</p>
    </div>
  </div>
  <div v-else @click="reload()" class="content">
    <p>{{state.error}}</p>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  /* height: 100vh; */
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
  /* display: flex;
  align-items: center;
  justify-content: center; */
  color: #fff;
  /* flex-wrap: wrap; */
  padding: 0.5em;
  min-height: 100vh;
}
/* .translate {
  text-align: center;
} */
p {
  text-align: left;
  text-shadow: 0 0 2px #000;
  background: rgba(0, 0, 0, 0.5);
  padding: 0.3em 0.5em;
  margin: 0.2em;
  border-radius: 0.2em;
  width: fit-content;
  /* display: inline-block; */
}
</style>
