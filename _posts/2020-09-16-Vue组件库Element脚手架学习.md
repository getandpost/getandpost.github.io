## 2020-09-16-Vue组件库Element脚手架学习

前提是安装了node和npm，因为我之前捣鼓过node.js和react native，所以win的电脑上已安装了。所以在此就不多做说明了。

Element官网地址：
http://element-cn.eleme.io/#/zh-CN

Element 脚手架 代码git地址：
https://github.com/ElementUI/element-starter.git

git clone下来后，进入element-starter目录

文档上写明了需要安装yarn
两种方式安装：
- 下载安装地址：https://classic.yarnpkg.com/zh-Hans/docs/install/#windows-stable
- 执行命令npm install -g yarn全局安装

接下来按照文档上步骤试运行，一切正常。

#### 开始写一个自己的页面

这里要用到路由，所以需要执行下面的命令安装一下
```$xslt
npm install vue-router --save
```

- 修改App.vue文件，添加一个路由的标签
```$xslt
<template>
  <div id="app">
    <transition name="fade" mode="out-in">
         <router-view></router-view>
    </transition>
  </div>
</template>

<script>
export default {
  methods: {
  }
}
</script>

<style>
#app {
  font-family: Helvetica, sans-serif;
  text-align: center;
}
</style>
```

- 在src目录下的新建router目录并新建一个名为 routes.js 的文件，内容如下：
```$xslt
import Home from '../views/home.vue'
import Login from '../views/login.vue'
import NotFound from '../views/404.vue'
let routes = [
    {
        path: '/',
        redirect: '/home'
    },
    {
        path: '/home',
        name: '后台管理中心',
        component: Home
    },
    {
        path: '/404',
        component: NotFound,
        name: "互联网的荒原"
    },
    {
        path: '*',
        name: '404',
        redirect: "/404"
    },
    {
        path: '/login',
        component: Login,
        name: '登录',
        hidden: true
    }
];
export default routes;
```

- 在src目录下的新建一个名为 views 的文件夹，在该文件夹下再新建一个名为home.vue的文件，内容如下：
```$xslt
<template>
  <div id="home">
    <img src="../assets/logo.png">
    <el-button @click="startHacking">Start</el-button>
  </div>
</template>

<script>
export default {
  methods: {
    startHacking () {
      this.$notify({
        title: 'It works!',
        type: 'success',
        message: 'We\'ve laid the ground work for you. It\'s time for you to build something epic!',
        duration: 5000
      })
    }
  }
}
</script>

<style>
#home {
  font-family: Helvetica, sans-serif;
  text-align: center;
}
</style>

```

- 最后修改 main.js文件:
```$xslt
import Vue from 'vue'
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'
import App from './App.vue'

import VueRouter from 'vue-router'
import routes from './router/routes'

Vue.use(ElementUI)
Vue.use(VueRouter)

const router = new VueRouter({
  routes
})

new Vue({
  router,
  el: '#app',
  render: h => h(App)
})

```

到此已完成简单demo的编码工作。

**启动查看效果**
```$xslt
npm run dev
```

**打包**
```$xslt
npm run build
```
在src同级目录下会生成一个名为dist的文件夹，打开index.html，后面跟上#/login即可看到手写的页面