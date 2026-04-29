---
title: Vue3实现明日方舟小人展示
published: 2026-04-29
description: 惹啊啊啊啊啊啊
image: ""
tags:
  - 前端开发
category: 我得
draft: false
lang: zh-CN
---
# 前言

长话短说其实我一直都想在前端展示明日方舟小人，因为它实在是太可爱了，如果你深入研究一下你就会发现明日方舟使用的其实并不是Live2D而是[Spine](https://zh.esotericsoftware.com/)。

## Spine

多的我就不介绍了，关于这个东西是什么可以自行搜索，因为明日方舟是一个长达7年的老游戏，所以其使用的资产文件为3.8版本的，如果你想要简单播放，你只需要使用  [Spine Web Player](https://zh.esotericsoftware.com/spine-player#Spine-Web-Player)这是Spine官方的运行时，但是缺点也很明显，它作为一个WebPlayer，拥有一些Player的控件，如果你想要使用的话还需要自行导入其包。以及导入CSS。

那么有没有其他的呢？

## PIXI

Pixi JS 是一个web5渲染引擎，其拥有Pixi-Spine的支持，使得您可以渲染Spine在web中。

兼容性？如果你能看到这篇文章就没有兼容性问题。

# 环境

```json
 "dependencies": {

    "pixi-spine": "^3.1.2",

    "pixi.js": "^6.5.10",

    "vue": "^3.5.32"

  },

  "devDependencies": {

    "@tsconfig/node24": "^24.0.4",

    "@types/node": "^24.12.2",

    "@vitejs/plugin-vue": "^6.0.6",

    "@vue/eslint-config-typescript": "^14.7.0",

    "@vue/tsconfig": "^0.9.1",

    "eslint": "^10.2.1",

    "eslint-config-prettier": "^10.1.8",

    "eslint-plugin-oxlint": "~1.60.0",

    "eslint-plugin-vue": "~10.8.0",

    "jiti": "^2.6.1",

    "npm-run-all2": "^8.0.4",

    "oxlint": "~1.60.0",

    "prettier": "3.8.3",

    "typescript": "~6.0.0",

    "vite": "^8.0.8",

    "vite-plugin-vue-devtools": "^8.1.1",

    "vue-tsc": "^3.2.6"

  },

  "engines": {

    "node": "^20.19.0 || >=22.12.0"

  }
```

对于明日方舟这种老游戏来说，需要特定低版本的PIXI.

```bash
npm i pixi.js@6.5.10 pixi-spine@3.1.2
```

首先在写代码前，感谢一下 **[Ark-Models](https://github.com/isHarryh/Ark-Models)** 和 **[PRTS](https://prts.wiki/)**

前者提供了Spine文件，后者提供了Spine文件的预览及信息

# Vue

```vue
<script setup lang="ts">

import { onMounted, ref } from 'vue'

import * as PIXI from 'pixi.js'

import { Spine } from 'pixi-spine'

const container = ref<HTMLDivElement | null>(null)

onMounted(() => {

  const app = new PIXI.Application({

    backgroundAlpha: 0,

    width: 300,

    height: 300,

  })

  const loader = PIXI.Loader.shared

  loader.add('shu', '/build_char_2025_shu.skel').load((_, res) => {

    const sp = new Spine(res.shu!.spineData!)

    sp.x = app.screen.width / 2

    sp.y = app.screen.height

    sp.scale.set(0.5)

    sp.state.setAnimation(0, 'Move', true)

    sp.stateData.defaultMix = 0.6

    app.stage.addChild(sp)

  })

})

</script>

  

<template>

  <div class="main">

    <div ref="container"></div>

  </div>

</template>

  

<style scoped>

.main {

  background-color: #bdbdbd;

}

</style>
```

就是这么简单。

# 后日谈

为什么我想要展示呢？

因为我想在我的友情链接模仿明日方舟的基建喵！
