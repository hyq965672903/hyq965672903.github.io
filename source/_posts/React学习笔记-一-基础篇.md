---
title: React学习笔记-(一)-基础篇
index_img: /img/default.png
banner_img: /img/default.png
date: 2022-11-09 20:44:31
tags:
categories:
description:
---

点击阅读前文前, 首页能看到的文章的简短描述

React入门，以create-react-app 脚手架创建React 逐渐深入

<!-- more -->

## 前言

狭义来讲 React 是 Facebook 内部开源出来的一个前端 UI 开发框架，广义来讲 React 不仅仅是 js 框架本身，更是一套完整的前端开发生态体系，这套体系包括：

1. React.js
2. ReactRenders: ReactDOM / ReactServer / ReactCanvas
3. Flux 模式及其实现（Redux , Fluxxor）
4. React 开源组件
5. React Native
6. GraphQl + Relay

## 环境

当前环境

`NodeJs`：v18.12.1 `npm`: v8.19.2

选择本地文件夹

```js
npx create-react-app my-app
cd my-app
npm start
```

npx create-react-app my-app --template typescript 采用typescript来编写

> https://create-react-app.dev/docs/getting-started

工具vscode

### [扩展]npm对比yarn

从 package.json 中安装项目依赖 中 :
npm install
或
yarn

向 package.json 添加添 /安装新的项目依赖 安 :
npm install {库名} --save 

或
yarn add {库名}

向 package.json 添加添 /安装新的 安 dev项目依赖（ 项 devDependency）:
npm install {库名} --save-dev 

或
yarn add {库名} --dev

删除依赖项目 删 :
npm uninstall package --save 

或
yarn remove package

升级某个依赖项目 升 :
npm update --save 

或
yarn upgrade

全局安装某项目依赖（慎用） 全 :
npm install package -g
或
yarn global add package
