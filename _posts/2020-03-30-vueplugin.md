---
layout: post
title: vue插件开发详解
categories: qianku
description: qianku
keywords: qianku
---
>版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。  
本文链接：[https://gudepeng.github.io/note/2020/03/30/vueplugin/](https://gudepeng.github.io/note/2020/03/30/vueplugin/)

废话不多说，直接进入正题。在开发vue的时候我们经常会开发自己的插件以供大家使用，下面就具体介绍下怎么开发插件。

## 一.创建项目
### 1.安装vuecli
```
npm install -g @vue/cli
```
### 2.创建项目
```
vue create hello-world
```
[@vue/cli 4 创建项目](https://cli.vuejs.org/zh/guide/creating-a-project.html#vue-create)
### 3.目录文件创建
![项目目录]](https://gudepeng.github.io/note/images/posts/2020-03-30-vueplugin/1.jpg)
* build 打包时使用的脚本程序
* examples demo项目路径
* packages 组件目录
* src 主程序目录
* gitignore git上传忽略项
* npmignore npm上传忽略项
* vue.config.js 项目（webpack）配置文件

## 二.项目开发
### 1.在src目录下创建index.js文件
```
//读取packages目录下的文件
const modulesFiles = require.context('../packages', true, /\.js$/)

// 定义 install 方法
const install = function(Vue) {
  if (install.installed) return
  install.installed = true
  //遍历modulesFiles，并注册到vue实例中，名是组件内default.name
  modulesFiles.keys().reduce((modules, modulePath) => {
    const value = modulesFiles(modulePath)
    Vue.component(value.default.name, value.default || value)
  })
}

if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue)
}

export default { install }
```
这样就会自动注册packages下所有的组件

### 2.组件国际化
```
vue add i18n
```
在src下创建locales目录，创建cn.js和en.js
```
const cn = {
  //国际化的属性值
}
export default cn
```

### 3.打包命令
编辑package.json
```
{  
  //组件名
  "name": "hades-ui",
  //版本
  "version": "0.1.0",
  "private": false,
  "scripts": {
    //打包命令
    "lib": "vue-cli-service build --target lib --name hades-ui --dest lib src/index.js && node ./build/i18n.js"
  },
  //主程序路径
  "main": "liib/hades-ui.umd.min.js",
  "descriptiion": "this is a vue ui",
  "license": "MIT"
}
```
编辑国际化打包程序，在build目录下创建i18n.js，拷贝2个语言包到lib下
```
const fs = require('fs')
fs.mkdirSync('./lib/locales')
fs.copyFileSync('./src/locales/cn.js', './lib/locales/cn.js')
fs.copyFileSync('./src/locales/en.js', './lib/locales/en.js')
```
## 三.上传项目
### 1.配置npm上传忽略项
创建.npmignore文件
```
.DS_Store
node_modules/
examples/
packages/
public/
src/
vue.config.js
babel.config.js
*.map
*.html

# local env files
.env.local
.env.*.local

# Log files
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Editor directories and files
.idea
.vscode
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw*
```
### 2.登录账号
```
npm login
```
### 3.上传项目
```
npm publiish
```

## 四.插件使用
### 1.安装插件
```
npm install hades-ui
```
### 2.国际化使用
在项目中对应的语言包内插入
```
import enLocale 'hades-ui/locales/en.js'
const en ={
    ...enLocale
}
export default en
```

### 参考样例
[demo项目](https://github.com/gudepeng/hades-ui)