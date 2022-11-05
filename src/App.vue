<script setup lang="ts">
import { reactive, onMounted, nextTick } from 'vue'
import Fuse from 'fuse.js'
import { it } from 'node:test';
const fs = require('fs')
const path = require('path')
const Jimp = require('jimp')
const jsonfile = require('jsonfile')
const screenshot = require('screenshot-desktop')
const { windowManager } = require('node-window-manager')
const cmd = require('node-cmd')
const ini = require('ini')
interface Result {
  ocr: string
  ori: string
  trans: string
}
// 状态
interface State {
  working: boolean
  result: Result[]
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
const config = ini.parse(fs.readFileSync('config.ini', 'utf-8'))
config.exclude = Object.keys(config.exclude)
// 排除项
const exclude: string[] = config.exclude
// 测试加密数据
let textData = [] as string[]
const resourcePath = path.resolve('./data')
fs.readdir(resourcePath, async (err: any, files: any) => {
  if (!files || !files.length) {
    state.error = '未检测到译文数据，请在软件根目录添加data文件夹并放置译文数据，然后点击本窗口重试'
    return
  }
  files.forEach((file: string, index: number) => {
    const file1 = fs.readFileSync(`${resourcePath}/${file}/${file}.json`, 'utf-8')
    const file2 = fs.readFileSync(`${resourcePath}/${file}/key`, 'utf-8')
    const xorData = BigInt(file1) ^ BigInt(file2)
    const decodedData = Buffer.from(xorData.toString(16), 'hex').toString('utf-8')
    let json = ''
    try {
      json = JSON.parse(decodedData)
    } catch (e) {
      console.log('读取译文失败 ' + file)
    }
    textData = textData.concat(json)
    if (index === files.length - 1) {
      ready()
    }
  })
})
// 导入文本数据
// let textData = [] as any[]
// const filePath = path.resolve('./data')
// fs.readdir(filePath, (err: any, files: any[]) => {
//   if (!files || !files.length) {
//     state.error = '未检测到译文数据，请在软件根目录添加data文件夹并放置译文数据，然后点击本窗口重试'
//     return
//   }
//   files.forEach((file: string, index) => {
//     jsonfile.readFile(filePath + '/' + file, (err: any, obj: any[]) => {
//       textData = textData.concat(obj)
//       if (index === files.length - 1) {
//         ready()
//       }
//     })
//   })
// })
// 初始化搜索引擎
let fuse: Fuse<any> | null = null
const ready = () => {
  fuse = new Fuse(textData, {
    keys: ['ori'],
    threshold: config.setting.threshold,
    distance: 20,
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
// 截屏
const takeScreenshot = () => {
  console.log('准备截图')
  screenshot({filename: './output/shot.jpg'}).then(async (img: any) => {
    console.log('截图完成' + img)
    const image = await Jimp.read(img)
    await image
      .crop(state.bounds.x + 20, state.bounds.y + 40, state.bounds.width - 40, state.bounds.height - 60)
      .grayscale()
      .contrast(+1)
      .normalize()
      .invert()
      .write('./output/shot-opt.jpg')
    console.log('图片处理完成，开始识别')
    console.time('识别完成，耗时')
    const cmdData: { [key: string]: string } = {
      models: 'ocr/models',
      det: 'ch_PP-OCRv3_det_infer.onnx',
      cls: 'ch_ppocr_mobile_v2.0_cls_infer.onnx',
      rec: 'model.onnx',
      keys: 'japan_dict.txt',
      image: 'output/shot-opt.jpg',
      numThread: '16',
      padding: '50',
      maxSideLen: '1024',
      boxScoreThresh: '0.5',
      boxThresh: '0.3',
      unClipRatio: '1.6',
      doAngle: '0',
      mostAngle: '0',
      GPU: '-1'
    }
    const cmdText = Object.keys(cmdData).map(key => `--${key} ${cmdData[key]}`).join(' ')
    cmd.run(`.\\ocr\\RapidOcrOnnx_${config.setting.ocrVersion}.exe ${cmdText}`,(err: any, data: any, stderr: any) => {
      console.timeEnd('识别完成，耗时')
      let result = data.split('=====End detect=====')[1].split('\r\n').filter((item: string) => item != '')
      result.shift()
      result = result.map((item: string) => ToCDB(item))
      // 跳过非日文的文本
      result = result.filter((item: string) => /[\u0800-\u4e00]+/.test(item))
      // 跳过小于1个字符的文本
      result = result.filter((item: string) => item.length > 2)
      // 跳过排除项
      result = result.filter((item: string) => !exclude.includes(item))
      if (result.length >= config.setting.skimNum && state.result[state.result.length - 1]?.trans == 'OCR识别结果过多，因此跳过本次搜索') {
        state.result.push({
          ocr: '',
          ori: '',
          trans: 'OCR识别结果过多，因此跳过本次搜索',
        })
      } else {
        const merge = result.filter((item: string) => !exclude.includes(item)).join('')
        result.unshift(merge)
        result = [...new Set(result)]
        result.forEach((item: string) => {
          if (!fuse) { return }
          const pushItem = { ocr: item, ori: '', trans: '' }
          // 进行搜索
          const search = fuse.search(item, { limit: 1 })
          console.log('搜寻 ' + item)
          console.log(search)
          const searchResult = search[0]
          // 跳过搜索结果为空的项
          if (!search.length || !config.setting.showFailed) { return }
          // 跳过当前历史记录中已存在的项
          if (searchResult && state.result.some((r: any) => r.trans != '' && r.trans === searchResult.item.trans)) { return }
          if (state.result.some((r: any) => r.ocr === item)) { return }
          // 排除原文与译文差距过大的项
          // if (searchResult && Math.abs(searchResult.item.ori.length - item.length) >= 20) { return }
          // 删除历史记录
          if (state.result.length >= config.setting.maxShowNum) {
            state.result.shift()
          }
          pushItem.ori = searchResult?.item.ori
          pushItem.trans = searchResult?.item.trans
          state.result.push(pushItem)
          nextTick(() => { window.scrollTo(0, 9999) })
        })
      }
      // 若工作状态，则继续调用截屏
      if (state.working) {
        takeScreenshot()
      }
    })
  })
}
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
function ToCDB(str: string) {
  var tmp = "";
  for (var i = 0; i < str.length; i++) {
    if (str.charCodeAt(i) > 65248 && str.charCodeAt(i) < 65375) {
      tmp += String.fromCharCode(str.charCodeAt(i) - 65248);
    } else {
      tmp += String.fromCharCode(str.charCodeAt(i));
    }
  }
  return tmp
}
</script>

<template>
  <div class="drag"></div>
  <div v-if="!state.error" @click="changeWorking()" class="content">
    <div v-if="!state.result.length">
      <p class="item">使用须知：</p>
      <p class="item">点击窗口任意位置开始运行，再次点击则停止运行</p>
      <p class="item">拖动窗口上方可调整窗口位置，拖动窗口四周可调整大小</p>
      <p class="item">可通过在窗口上方位置右击关闭本程序</p>
    </div>
    <div class="translate">
      <p class="item" v-for="item in state.result">
        <p v-if="config.setting.showOCR" class="small-text">识别:{{item.ocr}}</p>
        <p v-if="item.ori && config.setting.showOriginal" class="small-text">原文:{{item.ori}}</p>
        <p>{{item.trans}}</p>
      </p>
      <p v-if="!state.working" class="item stop">当前已停止运行，再次点击继续</p>
    </div>
  </div>
  <div v-else @click="reload()" class="content">
    <p class="item">{{state.error}}</p>
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
  background: rgba(0, 0, 0, 0.3);
}
body::-webkit-scrollbar {
  width: 10px;
  background: rgba(0, 0, 0, 0.3);
}
body::-webkit-scrollbar-thumb {
  background: rgba(0, 0, 0, 0.6);
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
  height: 50px;
}
.content {
  color: #fff;
  padding: 0.5em;
  min-height: 100vh;
}
.item {
  text-align: left;
  text-shadow: 0 0 2px #000;
  background: rgba(0, 0, 0, 0.6);
  padding: 0.3em 0.5em;
  margin: 0.2em;
  border-radius: 0.2em;
  width: fit-content;
}
.small-text {
  font-size: 0.75em;
  opacity: 0.8;
}
.stop {
  position: fixed;
  right: 0;
  bottom: 0;
}
</style>
