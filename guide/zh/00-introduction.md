---
title: 介绍
---

### 概述

Rollup 是一个 JavaScript 模块打包工具，可以将多个小的代码片段编译为完整的库和应用。与传统的 CommonJS 和 AMD 这一类非标准化的解决方案不同，Rollup 使用的是 ES6 版本 Javascript 中的模块标准。新的 ES 模块可以让你自由、无缝地按需使用你最喜爱的库中那些有用的单个函数。这一特性在未来将随处可用，但 Rollup 让你现在就可以，想用就用。

### 安装

```
npm install --global rollup
```

这样安装可以让 Rollup 成为全局可用的命令行，你也可以仅将其安装于本地，详见[本地化安装 Rollup ](guide/en/#installing-rollup-locally)。

### 快速开始

 Rollup 可以通过两种方式使用：使用[命令行](guide/en/#command-line-reference)方式，可以为命令行传入一个可选的配置文件。或者使用 [JavaScript API](guide/en/#javascript-api) 方式。运行 `rollup --help` 可以查看可用的选项和参数。

> [rollup-starter-lib](https://github.com/rollup/rollup-starter-lib) 和
[rollup-starter-app](https://github.com/rollup/rollup-starter-app) 两个项目可分别作为 Rollup 
运用于库和应用的参考。

以下命令假设你以 `main.js` 文件作为应用的入口点，并将所有的引入编译为单文件 `bundle.js`。

用于浏览器：

```
# 编译为一个在 <script> 标签中可用的自运行函数 ('iife')
rollup main.js --file bundle.js --format iife
```

用于 Node.js：

```
# 编译为 CommonJS 模块 ('cjs')
rollup main.js --file bundle.js --format cjs
```

同时用于浏览器和 Node.js：

```
# 需要为 UMD 格式的包指定一个名称
rollup main.js --file bundle.js --format umd --name "myBundle"
```

### 缘起

通常情况下，如果可以将项目拆分为一个个更小的模块，开发工作就能变得更容易。因为更少的代码通常意味着更少的意外情况，并且能够显著降低我们在开发中所碰到问题的复杂程度，我们只需保证分别完成各个子项目即可（[虽然事实并非如此简单](https://medium.com/@Rich_Harris/small-modules-it-s-not-quite-that-simple-3ca532d65de4)）。不幸的是，JavaScript 之前并未将此作为语言的一项核心功能。

ES6 版本的 Javascript 最终带来了 import 和 export 的语法（ES6 Module，译者注），该语法可以让我们在多个脚本中共享函数和数据。虽然这一语法标准已确定下来，但目前仅在较新的浏览器中可以使用，并且 Node.js 也尚未完成该标准的导入。有了 Rollup，你现在就可以放心地使用新的模块标准来书写代码，并将其编译为当前被广泛支持的 CommonJS、AMD 以及 IIFE 风格等多种格式。也就是说，你可以用*符合未来标准的代码*来赢得当下广大用户的芳心和敬意。:)

### Tree-Shaking

除了能够让你使用标准的 ES 模块，Rollup 还可以对所用的代码进行静态分析，并将未实际用到的代码剔除。这一特性将允许你放心的使用已有的工具和模块来创建应用而无需担心存在冗余的依赖和代码。

比如，使用 CommonJS 的时候，*工具或库必须被整体导入*：

```js
// 在 CommonJS 中，utils 对象将被整体导入
const utils = require( './utils' );
const query = 'Rollup';
// 使用 utils 对象的 ajax 方法
utils.ajax(`https://api.example.com?search=${query}`).then(handleResponse);
```

与上述例子中只能导入整个 `utils` 对象不同，ES 模块可以让我们仅导入所需的 `ajax` 函数：

```js
// 通过 ES6 的 import 语句导入 ajax 函数
import { ajax } from './utils';
const query = 'Rollup';
// 调用 ajax 函数
ajax(`https://api.example.com?search=${query}`).then(handleResponse);
```

由于只保留最精简的所需代码，通过 Rollup 生成的库和应用就可以更加轻量、快速、清晰。由于 import 和 export 语句的使用都是明确的，就决定了这一方式要比那种简单的通过代码压缩工具查找未使用变量的方案更有效。


### 兼容性

#### 导入 CommonJS 模块

Rollup 可以通过[通过插件](https://github.com/rollup/plugins/tree/master/packages/commonjs)导入现有的 CommonJS 模块。 

#### 发布 ES 模块

为了确保你的 ES 模块能够尽快在 Node.js 和 webpack 等支持 CommonJS 格式的工具中可用，你可以使用 Rollup 将代码编译为 UMD 或 CommonJS 格式，然后在 `package.json` 文件的 `main` 属性中指向当前编译的版本。如果你的 `package.json` 文件具有 `module` 属性，像 Rollup 和 [webpack 2+](https://webpack.js.org/) 这类 ES 模块识别工具可以直接[导入模块的 ES module 版本](https://github.com/rollup/rollup/wiki/pkg.module)。
