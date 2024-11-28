---
tags:
  - 博客
dg-publish: true
modified: 2024-11-28T10:38:51+08:00
created: 2024-11-28T09:36:37+08:00
dg-permalink: soduqifei
---
# 背景
最近新开发的一个项目体量比较大，而且能够预期后续会更大。每次启动webpack都要好几十秒，等待的过程让人很容易产生焦虑感，很影响开发体验。
正好几个月前vite5发布了，宣称构建速度再一次提升。趁这个机会来一波vite5的接入。
# 关键点
这次接入的关键点在于：低成本。
首先是生产上的低成本，我并不打算production模式下使用vite，而只是在development模式下使用。这样能够避免过多的风险，降低很多不必要的测试成本。换句话说，需要在原来的webpack构建模式全套保留的前提下来接入vite5.

另外就是工程上的低成本，考虑以下两点：
1. 原项目的改动点越少越好
2. 新增的内容尽量收敛

# 配置记录
这里主要记录一些额外的配置来适配项目的目前情况。
## 环境变量
在vue-cli里是这样定义的：
``` javascript
chainWebpack: (config) => {
    config.plugin('define').tap((args) => {
      args[0]['process.env'].PROXY_ENV = JSON.stringify(process.env.PROXY_ENV);
      return args;
    });
  }
```
在vite里则需要改成这样：
``` javascript
define: {
    'process.env': {
      PROXY_ENV: process.env.PROXY_ENV,
    },
  }
```
## 简写
vue-cli里是有默认的简写配置，而vite里则需要自己配置：
``` javascript
resolve: {
    alias: { '@': path.resolve(__dirname, 'src'), '~@': path.resolve(__dirname, 'src') },
  },
```
## 扩展名
解释：文件没有带扩展名时自动查找的扩展名列表。
vue-cli里有默认的扩展名配置，而vite里则需要自己配置：
``` javascript
resolve: {
    extensions: ['.vue', '.js', '.jsx'],
  },
```
注：官方**不** 建议忽略自定义导入类型的扩展名（例如：`.vue`），因为它会影响 IDE 和类型支持。
但这次改造的原则是尽量不动原先的项目，所以还是配置上了。
## index.html模板文件和根目录
在vue-cli中，根目录默认为`public`，`index.html`也在`public`目录中。
而vite则不是，这是官方文档的说明：

> 你可能已经注意到，在一个 Vite 项目中，`index.html` 在项目最外层而不是在 `public` 文件夹内。这是有意而为之的：在开发期间 Vite 是一个服务器，而 `index.html` 是该 Vite 项目的入口文件。

另外，vue-cli中，是会自动在`index.html`里插入`script`标签来引用`main.js`的。而vite里，则需要手动的在`index.html`中引入。

同样遵循原项目尽量不动的原则，我们引入`vite-plugin-html`插件来解决这些问题。
```javascript
import { createHtmlPlugin } from 'vite-plugin-html';
plugins: [
    createHtmlPlugin({
      minify: true,
      /**
       * 在这里写entry后，你将不需要在`index.html`内添加 script 标签，原有标签需要删除
       * @default
       */
      entry: '/src/main.js',
      /**
       * 如果你想将 `index.html`存放在指定文件夹，可以修改它，否则不需要配置
       * @default index.html
       */
      template: 'public/index.html',
    }),
]
```
## require问题
vue-cli能够支持commonJS语法，而vite则只支持ESM的语法。为了避免改动原项目，所以需要用`vite-plugin-require-transform`插件来解决。
配置如下：
```javascript
import requireTransform from 'vite-plugin-require-transform';

plugins: [
    requireTransform({
      fileRegex: /.js$|.vue$/,
    }),
  ],
```

## JSX支持，自动引入
这几个功能，vue-cli也没有默认配置，所以直接找到vite版的插件，配置上即可。
vite配置如下：
```javascript
import vueJsx from '@vitejs/plugin-vue-jsx';
import AutoImport from 'unplugin-auto-import/vite';

plugins: [
    vueJsx(),
    AutoImport({
      imports: ['vue'],
      eslintrc: {
        enabled: true,
      },
      dts: false,
    }),
  ],
```

## antd按需引入和theme覆盖
参考antd的官方文档配置即可，vite配置如下：
```javascript
const fs = require('fs');
const lessToJs = require('less-vars-to-js');
const themeVariables = lessToJs(fs.readFileSync(path.join(__dirname, './src/assets/styles/var.less'), 'utf8'));
import Components from 'unplugin-vue-components/vite';
const { AntDesignVueResolver } = require('unplugin-vue-components/resolvers');

plugins: [
    Components({
      resolvers: [
        AntDesignVueResolver({
          importStyle: false, // css in js
        }),
      ],
      dts: false,
      dirs: [],
    }),
  ],
  css: {
    preprocessorOptions: {
      less: {
        modifyVars: themeVariables,
        javascriptEnabled: true,
      },
    },
  },
```

## proxy
vue-cli和vite的proxy配置总体一致，但有一些细微的区别。
```javascript
'/api/test': {
    target: 'http://172.31.197.177:32089/',
    changeOrigin: true,
    logLevel: 'debug',
    pathRewrite: { //这是vue-cli的配置
      '^/api/test/': '/',
    },
    rewrite: (path) => path.replace(/^\/api\/test/, '/'), //这是vite的配置
  },
```
应该可以写一个plugin来抹平它们的配置区别，后面有时间再研究一下。

## 配置汇总
下面放出总体的vite配置：
```javascript
import { defineConfig } from 'vite';
const path = require('path');
import vue from '@vitejs/plugin-vue';
const fs = require('fs');
const lessToJs = require('less-vars-to-js');
const themeVariables = lessToJs(fs.readFileSync(path.join(__dirname, './src/assets/styles/var.less'), 'utf8'));
import requireTransform from 'vite-plugin-require-transform';
import vueJsx from '@vitejs/plugin-vue-jsx';
import { createHtmlPlugin } from 'vite-plugin-html';
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
const { AntDesignVueResolver } = require('unplugin-vue-components/resolvers');
const proxyTable = require('./proxy-table');

export default defineConfig({
  define: {
    'process.env': {
      PROXY_ENV: process.env.PROXY_ENV,
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src'), '~@': path.resolve(__dirname, 'src') },
    extensions: ['.vue', '.js', '.jsx'],
  },
  plugins: [
    vue(),
    createHtmlPlugin({
      minify: true,
      /**
       * 在这里写entry后，你将不需要在`index.html`内添加 script 标签，原有标签需要删除
       * @default
       */
      entry: '/src/main.js',
      /**
       * 如果你想将 `index.html`存放在指定文件夹，可以修改它，否则不需要配置
       * @default index.html
       */
      template: 'public/index.html',
    }),

    requireTransform({
      fileRegex: /.js$|.vue$/,
    }),
    vueJsx(),
    AutoImport({
      imports: ['vue'],
      eslintrc: {
        enabled: true,
      },
      dts: false,
    }),
    Components({
      resolvers: [
        AntDesignVueResolver({
          importStyle: false, // css in js
        }),
      ],
      dts: false,
      dirs: [],
    }),
  ],
  css: {
    preprocessorOptions: {
      less: {
        modifyVars: themeVariables,
        javascriptEnabled: true,
      },
    },
  },
  server: {
    port: 8081,
    proxy: {
      ...proxyTable,
    },
  },
});
```

# 效果展示
最后，验证一下引入vite后的效果。
在vue-cli的命令里添加`--report`参数，能够查看构建时间：`"serve": "vue-cli-service serve --report",`
vite构建完成后，默认会有时间显示。
下面给出对比图。
vue-cli构建时间：
![[f9630b4009ff07471389e683d8b7b1bc_36937.png]]
vite构建时间：
![[4f4b18650b6b2b9f3d6ddfa5dd7300ee_68087.png]]

可以看出来提升还是很大的。使用vite构建，相比vue-cli，时间大约缩短为原先的10倍。