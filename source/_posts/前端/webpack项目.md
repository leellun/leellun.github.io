---
title: webpack项目
date: 2019-01-24 17:38:02
categories:
  - 前端
tags:
  - webpack
author: leellun
---

1 快速入门

```
mkdir webpack-demo && cd webpack-demo 
# 初始化
npm init -y 
# 安装webpack
npm install webpack webpack-cli --save-dev 
```

2 本地基本结构

```
  webpack-demo
+ |- package.json
+ |- /dist
+   |- index.html
+ |- /src
+   |- index.js

```

3 相关依赖

```
# Lodash 是一个一致性、模块化、高性能的 JavaScript 实用工具库 查看各个构件版本的区别并选择一个适合你的版本
npm install --save lodash

# 构建webpack项目
npm install webpack webpack-cli --save-dev 

# 加载css
npm install --save-dev style-loader css-loader

# 加载sass
npm install sass-loader node-sass --save-dev

#--------------------PostCSS 处理 loader---------------------------
npm i -D postcss-loader
npm install autoprefixer --save-dev

# 以下可以不用安装
# cssnext可以让你写CSS4的语言，并能配合autoprefixer进行浏览器兼容的不全，而且还支持嵌套语法
$ npm install postcss-cssnext --save-dev

# 类似scss的语法，实际上如果只是想用嵌套的话有cssnext就够了
$ npm install precss --save-dev

# 在@import css文件的时候让webpack监听并编译
$ npm install postcss-import --save-dev

#样式表抽离成专门的单独文件并且设置版本号
npm install --save-dev mini-css-extract-plugin

# 压缩 CSS
npm i -D optimize-css-assets-webpack-plugin

#JS 压缩
npm i -D uglifyjs-webpack-plugin

#解决 CSS 文件或者 JS 文件名字哈希变化的问题
npm install --save-dev html-webpack-plugin

#清理 dist 目录
npm install clean-webpack-plugin --save-dev

#处理文件的导入
npm install --save-dev file-loader

#对图片进行压缩和优化
npm install image-webpack-loader --save-dev

#处理图片成 base64
npm install --save-dev url-loader

#合并两个webpack的js配置文件
npm install --save-dev webpack-merge

#使用 webpack-dev-server 和热更新
npm install --save-dev webpack-dev-server

#Babel优化
npm install babel-plugin-transform-runtime --save-dev
npm install babel-runtime --save

#ESLint校验代码格式规范
npm install eslint --save-dev
npm install eslint-loader --save-dev
# 以下是用到的额外的需要安装的eslint的解释器、校验规则等
npm i -D babel-eslint standard

```

```
npm install --save-dev  autoprefixer babel-eslint  babel-plugin-transform-runtime babel-runtime clean-webpack-plugin css-loader eslint eslint-loader file-loader html-webpack-plugin image-webpack-loader mini-css-extract-plugin node-sass optimize-css-assets-webpack-plugin postcss-import postcss-loader sass-loader standard style-loader uglifyjs-webpack-plugin url-loader webpack webpack-cli webpack-dev-server webpack-merge babel-loader @babel/core babel-preset-env
```







### [文件](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%96%87%e4%bb%b6)

- `raw-loader` 加载文件原始内容（utf-8）
- `val-loader` 将代码作为模块执行，并将 exports 转为 JS 代码
- `url-loader` 像 file loader 一样工作，但如果文件小于限制，可以返回 [data URL](https://tools.ietf.org/html/rfc2397)
- `file-loader` 将文件发送到输出文件夹，并返回（相对）URL

### [JSON](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=json)

- `json-loader` 加载 [JSON](http://json.org/) 文件（默认包含）
- `json5-loader` 加载和转译 [JSON 5](https://json5.org/) 文件
- `cson-loader` 加载和转译 [CSON](https://github.com/bevry/cson#what-is-cson) 文件

### [转换编译(Transpiling)](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e8%bd%ac%e6%8d%a2%e7%bc%96%e8%af%91transpiling)

- `script-loader` 在全局上下文中执行一次 JavaScript 文件（如在 script 标签），不需要解析
- `babel-loader` 加载 ES2015+ 代码，然后使用 [Babel](https://babeljs.io/) 转译为 ES5
- `buble-loader` 使用 [Bublé](https://buble.surge.sh/guide/) 加载 ES2015+ 代码，并且将代码转译为 ES5
- `traceur-loader` 加载 ES2015+ 代码，然后使用 [Traceur](https://github.com/google/traceur-compiler#readme) 转译为 ES5
- [`ts-loader`](https://github.com/TypeStrong/ts-loader) 或 [`awesome-typescript-loader`](https://github.com/s-panferov/awesome-typescript-loader) 像 JavaScript 一样加载 [TypeScript](https://www.typescriptlang.org/) 2.0+
- `coffee-loader` 像 JavaScript 一样加载 [CoffeeScript](http://coffeescript.org/)

### [模板(Templating)](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%a8%a1%e6%9d%bftemplating)

- `html-loader` 导出 HTML 为字符串，需要引用静态资源
- `pug-loader` 加载 Pug 模板并返回一个函数
- `jade-loader` 加载 Jade 模板并返回一个函数
- `markdown-loader` 将 Markdown 转译为 HTML
- [`react-markdown-loader`](https://github.com/javiercf/react-markdown-loader) 使用 markdown-parse parser(解析器) 将 Markdown 编译为 React 组件
- `posthtml-loader` 使用 [PostHTML](https://github.com/posthtml/posthtml) 加载并转换 HTML 文件
- `handlebars-loader` 将 Handlebars 转移为 HTML
- [`markup-inline-loader`](https://github.com/asnowwolf/markup-inline-loader) 将内联的 SVG/MathML 文件转换为 HTML。在应用于图标字体，或将 CSS 动画应用于 SVG 时非常有用。

### [样式](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%a0%b7%e5%bc%8f)

- `style-loader` 将模块的导出作为样式添加到 DOM 中
- `css-loader` 解析 CSS 文件后，使用 import 加载，并且返回 CSS 代码
- `less-loader` 加载和转译 LESS 文件
- `sass-loader` 加载和转译 SASS/SCSS 文件
- `postcss-loader` 使用 [PostCSS](http://postcss.org/) 加载和转译 CSS/SSS 文件
- `stylus-loader` 加载和转译 Stylus 文件

### [清理和测试(Linting && Testing)](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%b8%85%e7%90%86%e5%92%8c%e6%b5%8b%e8%af%95linting-ampamp-testing)

- `mocha-loader` 使用 [mocha](https://mochajs.org/) 测试（浏览器/NodeJS）
- [`eslint-loader`](https://github.com/webpack-contrib/eslint-loader) PreLoader，使用 [ESLint](https://eslint.org/) 清理代码
- `jshint-loader` PreLoader，使用 [JSHint](http://jshint.com/about/) 清理代码
- `jscs-loader` PreLoader，使用 [JSCS](http://jscs.info/) 检查代码样式
- `coverjs-loader` PreLoader，使用 [CoverJS](https://github.com/arian/CoverJS) 确定测试覆盖率

### [框架(Frameworks)](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%a1%86%e6%9e%b6frameworks)

- `vue-loader` 加载和转译 [Vue 组件](https://vuejs.org/v2/guide/components.html)
- `polymer-loader` 使用选择预处理器(preprocessor)处理，并且 `require()` 类似一等模块(first-class)的 Web 组件
- `angular2-template-loader` 加载和转译 [Angular](https://angular.io/) 组件
- Awesome 更多第三方 loader，查看 [awesome-webpack 列表](https://github.com/webpack-contrib/awesome-webpack#loaders)。

## [打包分析优化](https://malun666.github.io/aicoder_vip_doc/#/pages/vip_2webpack?id=%e6%89%93%e5%8c%85%e5%88%86%e6%9e%90%e4%bc%98%e5%8c%96)

`webpack-bundle-analyzer`插件可以帮助我们分析打包后的图形化的报表。