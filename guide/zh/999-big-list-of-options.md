---
title: 选项大列表
---

### 核心功能 (Core functionality)

#### external
类型：`(string | RegExp)[] | RegExp | string | (id: string, parentId: string, isResolved: boolean) => boolean`<br>
命令行参数：`-e`/`--external <external-id,another-external-id,...>`

该选项用于匹配需要保留在 bundle 外部的模块，它的值可以是一个接收模块 `id` 参数并且返回 `true`（表示排除）或 `false`（表示包含）的函数，也可以是一个由模块 ID 构成的数组，还可以是可以匹配到模块 ID 的正则表达式。除此之外，它还可以是单个模块 ID 或者单个正则表达式。匹配得到的模块 ID 应该满足以下条件之一：

1. import 语句中外部依赖的名称。例如，如果标记 `import "dependency.js"` 为外部依赖，那么模块 ID 为 `"dependency.js"`，而如果标记 `import "dependency"` 为外部依赖，那么模块 ID 为 `"dependency"`。
2. 绝对路径。（例如，文件的绝对路径）

```js
// rollup.config.js
import path from 'path';

export default {
  ...,
  external: [
    'some-externally-required-library',
    path.resolve( __dirname, 'src/some-local-file-that-should-not-be-bundled.js' ),
    /node_modules/
  ]
};
```

请注意，如果你想要通过 `/node_modules/` 正则表达式将 `node_modules` 中引入的包当成外部依赖处理，比如 `import {rollup} from 'rollup'` 中的 `rollup`，你须要先引入 [@rollup/plugin-node-resolve](https://github.com/rollup/plugins/tree/master/packages/node-resolve) 插件。

当用作命令行参数时，该选项的值为通过逗号分隔的模块 ID 列表：

```bash
rollup -i src/main.js ... -e foo,bar,baz
```

当该选项的值为函数时，它会提供三个参数 `(id, parent, isResolved)`，这些参数可以为你提供更细粒度的控制：

* `id` 值为相关模块的 id
* `parent` 值为执行引入的模块的 id
* `isResolved` 值为布尔值，指是否已经通过插件等方式解决模块依赖

创建 `iife` 或 `umd` 格式的 bundle 时，你需要通过 `output.globals` 选项来提供全局变量名称，以替换外部引入。

如果引入路径为相对路径（即以 `./` 或 `../` 开头），那么此路径会被标记为 "external"。rollup 会把模块 id 解析为系统绝对文件路径，以便不同的外部模块可以合并到一起。当写入 bundle 以后，这些引入模块将会再次被转换为相对引入。例如：

```js
// 输入
// src/main.js （入口文件）
import x from '../external.js';
import './nested/nested.js';
console.log(x);

// src/nested/nested.js
// 如果引入依赖已存在，它将指向同一个文件
import x from '../../external.js';
console.log(x);

// 输出
// 不同的依赖将会被合并
import x from '../external.js';

console.log(x);

console.log(x);
```

如果存在多个入口，rollup 会把它们转换会相对引入的方式，就像 `output.file` 或 `output.dir` 与入口文件或所有入口文件位于相同的目录。

#### input
类型：`string | string [] | { [entryName: string]: string }`<br>
命令行参数：`-i`/`--input <filename>`

该选项是指 bundle 的入口文件，比如你的 `main.js` 或 `app.js` 或 `index.js` 文件。如果你使用数组或者对象作为 input 的值，那么它们将被打包到独立的输出区块（chunks）。除非使用 [`output.file`](guide/en/#outputfile) 选项，否则生成的区块名称会根据 [`output.entryFileNames`](guide/en/#outputentryfilenames) 选项来确定。该选项值为对象形式时，对象的键将作为文件名中的 `[name]`，而对于值为数组形式，数组的值将作为入口的文件名。

请注意，使用对象形式时，只要在入口名称中添加 '/'，就可以将入口打包到不同的子文件夹中。下面是个例子，根据 `entry-a.js` 和 `entry-b/index.js`，至少产生两个入口区块（chunks），其中 `index.js` 将输出在 `entry-b` 文件夹中：

```js
// rollup.config.js
export default {
  ...,
  input: {
    a: 'src/main-a.js',
    'b/index': 'src/main-b.js'
  },
  output: {
    ...,
    entryFileNames: 'entry-[name].js'
  }
};
```

在使用命令行时，多个入口只需要多次使用 `--input` 提供选项即可。当作为第一个选项提供时，等同于不加 `--input` 选项：

```sh
rollup --format es --input src/entry1.js --input src/entry2.js
# 等同于
rollup src/entry1.js src/entry2.js --format es
```

可以通过 `=` 为入口命名 chunk：

```sh
rollup main=src/entry1.js other=src/entry2.js --format es
```

可以使用引号指定包含空格的文件名：

```sh
rollup "main entry"="src/entry 1.js" "src/other entry.js" --format es
```

#### output.dir
类型：`string`<br>
命令行参数：`-d`/`--dir <dirname>`

该选项用于指定所有生成 chunk 文件所在的目录。如果生成多个 chunk，则此选项是必须的。否则，可以使用 `file` 选项代替。

#### output.file
类型：`string`<br>
命令行参数：`-o`/`--file <filename>`

该选项用于指定要写入的文件名。如果该选项生效，那么同时也会生成源码映射（sourcemap）文件。只有当生成的 chunk 不超过一个时，该选项才会生效。

#### output.format
类型：`string`<br>
命令行参数：`-f`/`--format <formatspecifier>`

该选项用于指定生成 bundle 的格式。可以是以下之一：

* `amd` - 异步模块定义，适用于 RequireJS 等模块加载器
* `cjs` - CommonJS，适用于 Node 环境和其他打包工具（别名：`commonjs`）
* `es` - 将 bundle 保留为 ES 模块文件，适用于其他打包工具以及支持 `<script type=module>` 标签的浏览器（别名: `esm`，`module`）
* `iife` - 自执行函数，适用于 `<script>` 标签。（如果你要为你的应用创建 bundle，那么你很可能用它。）
* `umd` - 通用模块定义，生成的包同时支持 `amd`、`cjs` 和 `iife` 三种格式
* `system` - SystemJS 模块加载器的原生格式（别名: `systemjs`）

#### output.globals
类型：`{ [id: string]: string } | ((id: string) => string)`<br>
命令行参数：`-g`/`--globals <external-id:variableName,another-external-id:anotherVariableName,...>`

该选项用于使用 `id: variableName` 键值对指定的、在 `umd` 或 `iife` 格式 bundle 中的外部依赖。下面就是一个外部依赖的例子：

```js
import $ from 'jquery';
```

在这个例子中，我们想要告诉 Rollup `jquery` 是外部依赖，并且 `jquery` 模块的 ID 为全局变量 `$`：

```js
// rollup.config.js
export default {
  ...,
  external: ['jquery'],
  output: {
    format: 'iife',
    name: 'MyBundle',
    globals: {
      jquery: '$'
    }
  }
};

/*
var MyBundle = (function ($) {
  // code goes here
}($));
*/
```

或者，我们也可以使用将外部依赖映射为全局变量的函数作为 `output.globals` 的值。

在作为命令行参数时，该选项的值应该是以逗号分隔的 `id:variableName` 键值对：

```
rollup -i src/main.js ... -g jquery:$,underscore:_
```

要告诉 Rollup 用全局变量替换本地文件，请使用系统绝对文件路径：

```js
// rollup.config.js
import path from 'path';
const externalId = path.resolve( __dirname, 'src/some-local-file-that-should-not-be-bundled.js' );

export default {
  ...,
  external: [externalId],
  output: {
    format: 'iife',
    name: 'MyBundle',
    globals: {
      [externalId]: 'globalVariable'
    }
  }
};
```

#### output.name
类型：`string`<br>
命令行: `-n`/`--name <variableName>`

该选项用于，在想要使用全局变量名来表示你的 bundle 时，输出格式必须指定为 `iife` 或 `umd`。同一个页面上的其他脚本可以通过这个变量名来访问你的 bundle 导出。

```js
// rollup.config.js
export default {
  ...,
  output: {
    file: 'bundle.js',
    format: 'iife',
    name: 'MyBundle'
  }
};

// var MyBundle = (function () {...
```

该选项也支持命名空间，即包含 `.` 的名字。最终生成 bundle 将包含命名空间所需要的设置。

```
rollup -n "a.b.c"

/* ->
this.a = this.a || {};
this.a.b = this.a.b || {};
this.a.b.c = ...
*/
```

#### output.plugins
类型：`OutputPlugin | (OutputPlugin | void)[]`

该选项用于指定输出插件，这是设置插件的唯一入口。你查看 [Using output plugins](guide/en/#using-output-plugins) 了解更多关于如何设置指定输出插件的信息，另外，[Plugins](guide/en/#plugin-development) 会告诉你如何写一个你自己的插件。对于从包中引入的插件，请记住要调用引入的函数（例如，应该使用 `commonjs()`，而不是 `commonjs`）。返回值为 Falsy 的插件将会被忽略，这样可以用于灵活启用和禁用插件。

请注意，并不是所有的插件都可以通过该选项使用。`output.plugins` 选项是受限的，例如，只有在 Rollup 的主分析阶段完成以后，在 `bundle.generate()` 或者 `bundle.write()` 阶段执行的 hooks 的插件才可以使用该选项。如果你是一个插件的作者，你可以查看 [output generation hooks](guide/en/#output-generation-hooks) 了解更多关于 hooks 的使用方法。

以下是一个使用压缩插件的例子：

```js
// rollup.config.js
import {terser} from 'rollup-plugin-terser';

export default {
  input: 'main.js',
  output: [
    {
      file: 'bundle.js',
      format: 'es'
    },
    {
      file: 'bundle.min.js',
      format: 'es',
      plugins: [terser()]
    }
  ]
};
```

#### plugins
类型：`Plugin | (Plugin | void)[]`

查看 [Using plugins](guide/en/#using-plugins) 文档，了解更多关于如何使用插件的信息，另外，根据 [Plugins](guide/en/#plugin-development)，你可以写一个自定义插件（动手试试看，写一个插件并不困难，你可以通过 Rollup 插件做很多拓展）。对于从包中引入的插件，请记住调用引入的函数（例如，应该使用 `commonjs()`，而不是 `commonjs`）。返回值为 Falsy 的插件将会被忽略，这样可以用于灵活启用和禁用插件。

```js
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';

const isProduction = process.env.NODE_ENV === 'production';

export default (async () => ({
  input: 'main.js',
  plugins: [
    resolve(),
    commonjs(),
    isProduction && (await import('rollup-plugin-terser')).terser()
  ],
  output: {
    file: 'bundle.js',
    format: 'cjs'
  }
}))();
```

（上述例子还演示了如何使用异步 IIFE 和动态引入来避免引入会使 Rollup 打包变慢的不必要的模块。）

### 进阶功能（Advanced functionality）

#### cache
类型：`RollupCache | false`

该选项用于指定先前的 bundle 的缓存。当它设置后，Rollup 只会对改变的部分进行重新分析，从而加速观察模式（watch mode）中的后续构建。如果将它设置为 `false`，则会阻止 bundle 生成缓存，还会导致插件的缓存失效。

```js
const rollup = require('rollup');
let cache;

async function buildWithCache() {
  const bundle = await rollup.rollup({
    cache, // 如果 cache 值为 falsy，那么该选项会被忽略
    // ... 其他输入选项
  });
  cache = bundle.cache; // 保存之前构建的数据缓存
  return bundle;
}

buildWithCache()
  .then(bundle => {
    // ... do something with the bundle
  })
  .then(() => buildWithCache()) // 将会使用之前构建的缓存
  .then(bundle => {
    // ... do something with the bundle
  })
```

#### onwarn
类型：`(warning: RollupWarning, defaultHandler: (warning: string | RollupWarning) => void) => void;`

该选项用于拦截警告消息。如果未使用该选项，警告消息就会去重并打印在控制台（console）。当命令行中使用 [`--silent`](guide/en/#--silent) 参数时，该选项是唯一能够获取警告通知的方法。

该选项是函数类型，接收两个输入参数：警告对象（warning object）和默认处理函数（default handler）。其中，警告对象至少包含 `code` 和 `message` 两个属性，用于处理不同类型的警告。另外，根据不同的警告类型，警告对象上会有其他的属性。

```js
// rollup.config.js
export default {
  ...,
  onwarn (warning, warn) {
    // 忽略指定类型的警告
    if (warning.code === 'UNUSED_EXTERNAL_IMPORT') return;

    // 抛出其他类型错误
    if (warning.code === 'NON_EXISTENT_EXPORT') throw new Error(warning.message);

    // 使用默认处理函数兜底
    warn(warning);
  }
};
```

很多警告还具有 `loc` 和 `frame` 属性，它们可以用来定位警告来源。

```js
// rollup.config.js
export default {
  ...,
  onwarn ({ loc, frame, message }) {
    if (loc) {
      console.warn(`${loc.file} (${loc.line}:${loc.column}) ${message}`);
      if (frame) console.warn(frame);
    } else {
      console.warn(message);
    }
  }
};
```


#### output.assetFileNames
类型：`string`<br>
命令行参数：`--assetFileNames <pattern>`<br>
默认值：`"assets/[name]-[hash][extname]"`

该选项用于自定义构建结果中的静态文件名称。它支持以下占位符：
 * `[extname]`：包含点的静态文件扩展名，例如：`.css`。
 * `[ext]`：不包含点的文件扩展名，例如：`css`。
 * `[hash]`：基于静态文件的名称和内容的哈希。
 * `[name]`：静态文件的名称，不包含扩展名。

正斜杆 `/` 可以用来划分文件到子目录。又见 [`output.chunkFileNames`](guide/en/#outputchunkfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames)。

#### output.banner/output.footer
类型：`string | (() => string | Promise<string>)`<br>
命令行参数：`--banner`/`--footer <text>`

该选项用于在 bundle 顶部添加一个字符串，或者在构建结果末尾添加一个字符串。当然，它也支持返回一个字符串的异步函数。（注意：`banner` 和 `footer` 选项不会破坏 sourcemaps。）

```js
// rollup.config.js
export default {
  ...,
  output: {
    ...,
    banner: '/* my-library version ' + version + ' */',
    footer: '/* follow me on Twitter! @rich_harris */'
  }
};
```

了解更多可以参考 [`output.intro/output.outro`](guide/en/#outputintrooutputoutro)。

#### output.chunkFileNames
类型：`string`<br>
命令行参数：`--chunkFileNames <pattern>`<br>
默认值：`"[name]-[hash].js"`

该选项用于对代码分割中产生的 chunk 文件自定义命名。它支持以下形式：
 * `[format]`：输出（output）选项中定义的 `format` 的值，例如：`es` 或 `cjs`。
 * `[hash]`：哈希值，由 chunk 文件本身的内容和所有它依赖的文件的内容共同组成。
 * `[name]`：chunk 的名字。它可以通过 [`output.manualChunks`](guide/en/#outputmanualchunks) 显示设置，或者通过插件调用 [`this.emitFile`](guide/en/#thisemitfileemittedfile-emittedchunk--emittedasset--string) 设置。如果没有做任何设置，它将会根据 chunk 的内容来确定。

正斜杆 `/` 可以用来划分 chunk 文件到子目录。又见 [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames)。

#### output.compact
类型：`boolean`<br>
命令行参数：`--compact`/`--no-compact`<br>
默认值：`false`

该选项用于压缩 Rollup 产生的额外代码。请注意，这个选项不会影响用户的代码。在构建已经压缩好的代码时，这个选择是很有用的。

#### output.entryFileNames
类型：`string`<br>
命令行参数：`--entryFileNames <pattern>`<br>
默认值：`"[name].js"`

该选项用于指定 chunks 的入口文件名。支持以下形式：
* `[format]`：输出（output）选项中定义的 `format` 的值，例如：`es` 或 `cjs`。
* `[hash]`：哈希值，由入口文件本身的内容和所有它依赖的文件的内容共同组成。
* `[name]`：入口文件的文件名（不包含扩展名），当入口文件（entry）定义为对象时，它的值时对象的键。

正斜杆 `/` 可以用来划分文件到子目录。又见 [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.chunkFileNames`](guide/en/#outputchunkfilenames)。

使用 [`output.preserveModules`](guide/en/#outputpreservemodules) 选项时，该模式也会生效。不过，此时它有如下一组不同的占位符：
* `[format]`：输出（output）选项中定义的 `format` 的值。
* `[name]`：文件名（不包含扩展名）
* `[ext]`：文件扩展名。
* `[extname]`：文件扩展名，如果存在则以 `.` 开头。

#### output.extend
类型：`boolean`<br>
命令行参数：`--extend`/`--no-extend`<br>
默认值：`false`

该选项用于指定是否扩展 `umd` 或 `iife` 格式中 `name` 选项定义的全局变量。当值为 `true` 时，`name`选项指定的全局变量将定义为 `(global.name = global.name || {})`。当值为 `false` 时，`name` 选项指定的全局变量将被覆盖为 `(global.name = {})`。

#### output.hoistTransitiveImports
类型：`boolean`<br>
命令行参数：`--hoistTransitiveImports`/`--no-hoistTransitiveImports`<br>
默认值：`true`

默认情况下，创建多个块 (chunk) 时，入口块的可传递引入 (transitive imports) 将会以空引入的形式被打包。详细信息和背景请查阅 ["为什么在拆分代码时我的输入块中会出现其他引入？"](guide/en/#why-do-additional-imports-turn-up-in-my-entry-chunks-when-code-splitting)。将值设置为 `false` 将禁用此行为。当使用 [`output.preserveModules`](guide/en/#outputpreservemodules) 选项时，该选项会被忽略，使得永远不会取消引入。

#### output.inlineDynamicImports
类型：`boolean`<br>
命令行参数：`--inlineDynamicImports`/`--no-inlineDynamicImports`
默认值：`false`

该选项用于内联动态引入，而不是用于创建包含新 Chunk 的独立 bundle。它只在单一输入源时产生作用。请注意，它会影响执行顺序：如果该模块是内联动态引入，那么它将会被立即执行。

#### output.interop
类型：`boolean`<br>
命令行参数：`--interop`/`--no-interop`<br>
默认值：`true`

该选项用于决定是否添加 "interop block"。默认情况下（`interop: true`），出于安全起见，如果要区分默认导出和命名导出，Rollup 会将所有外部依赖的默认导出（`default` exports）赋值给一个单独的变量。通常情况下，这仅在外部依赖被转译（例如 Babel）时才适用 - 如果你不需要它，你可以将它设置为 `interop: false`。

#### output.intro/output.outro
类型：`string | (() => string | Promise<string>)`<br>
命令行参数：`--intro`/`--outro <text>`

除了在特定格式中代码不同外，该选项功能和 [`output.banner/output.footer`](guide/en/#outputbanneroutputfooter) 类似。

```js
export default {
  ...,
  output: {
    ...,
    intro: 'const ENVIRONMENT = "production";'
  }
};
```

#### output.manualChunks
类型：`{ [chunkAlias: string]: string[] } | ((id: string, {getModuleInfo, getModuleIds}) => string | void)`

该选项允许你创建自定义的公共模块。当它的值为对象形式时，每个属性代表一个大块，其中包含列出的模块及其所有依赖（除非他们已经在其他 chunk 中，否则将会是模块图（module graph）的一部分）。chunk 的名称由对象属性的键（key）决定。

请注意，列出的模块本身不一定是模块图（module graph）的一部分，该特性对于使用 `@rollup/plugin-node-resolve` 并从中使用深度引入（deep imports）是非常有用的。例如：

```
manualChunks: {
  lodash: ['lodash']
}
```

上述例子中，即使你只是使用 `import get from 'lodash/get'` 形式，Rollup 也会将 lodash 的所有模块放到一个自定义 chunk 中。

当该选项值为函数形式时，每个被解析的模块都会经过该函数处理。如果函数返回字符串，那么该模块及其所有依赖将被添加到以返回字符串命名的自定义 chunk 中。例如，以下例子会创建一个命名为 `vendor` 的 chunk，它包含所有在 `node_modules` 中的依赖。

```javascript
manualChunks(id) {
  if (id.includes('node_modules')) {
    return 'vendor';
  }
}
```

请注意，如果自定义 chunk 在使用相应模块之前触发了副作用，那么它可能改变整个应用程序的行为。

当 `manualChunks` 使用函数形式时，它的第二个参数是一个对象，包含 `getMouduleInfo` 函数和 `getMoudleIds` 函数，其工作方式与插件上下文中的 [`this.getModuleInfo`](guide/en/#thisgetmoduleinfomoduleid-string--moduleinfo) 和 [`this.getModuleIds`](guide/en/#thisgetmoduleids--iterableiteratorstring) 相同。

该选项可以用于根据模块在模块图中的位置动态确定它应该被放在哪个自定义 chunk 中。例如，考虑这样一个场景，有一组组件，每个组件动态引入一组已转译的依赖，即

```js
// 在 “foo” 组件中

function getTranslatedStrings(currentLanguage) {
  switch (currentLanguage) {
    case 'en': return import('./foo.strings.en.js');
    case 'de': return import('./foo.strings.de.js');
    // ...
  }
}
```

如果将很多这样的组件一起使用，则会导致生成很多很小的动态引入 chunk：尽管我们知道由同一块引入的所有相同语言的语言文件将始终一起使用，但是 Rollup 并不知道。

下面代码将会合并仅由单个入口使用的相同语言的所有文件：

```js
manualChunks(id, { getModuleInfo }) {
  const match = /.*\.strings\.(\w+)\.js/.exec(id);
  if (match) {
    const language = match[1]; // 例如 "en"
    const dependentEntryPoints = [];

    // 在这里，我们使用 Set 一次性处理每个依赖模块
    // 它可以阻止循环依赖中的无限循环
    const idsToHandle = new Set(getModuleInfo(id).dynamicImporters);

    for (const moduleId of idsToHandle) {
      const { isEntry, dynamicImporters, importers } = getModuleInfo(moduleId);
      if (isEntry || dynamicImporters.length > 0) dependentEntryPoints.push(moduleId);

      // 使用 Set 迭代器是明智之选
      for (const importerId of importers) idsToHandle.add(importerId);
    }

    // 如果仅有一个入口，那么我们会根据入口名将它放到独立的 chunk 中
    if (dependentEntryPoints.length === 1) {
      return `${dependentEntryPoints[0].split('/').slice(-1)[0].split('.')[0]}.strings.${language}`;
    }
    // 对于多个入口，我们会把它放到共享 chunk 中
    if (dependentEntryPoints.length > 1) {
      return `shared.strings.${language}`;
    }
  }
}
```

#### output.minifyInternalExports
类型：`boolean`<br>
命令行参数：`--minifyInternalExports`/`--no-minifyInternalExports`<br>
默认值：在 `es` 格式和 `system` 格式或者 `output.compact` 值为 `true`的情况下为 `true`，否则为 `false`

默认情况下，该选项的值在 `es` 格式、 `system` 格式或者 `output.compact` 值为 `true` 时为 `true`，意味着 Rollup 会尝试把内部变量导出为单个字母的变量，以便更好地压缩代码。

**示例**<br>
输入:

```js
// main.js
import './lib.js';

// lib.js
import('./dynamic.js');
export const value = 42;

// dynamic.js
import {value} from './lib.js';
console.log(value);
```

输出（在 `output.minifyInternalExports: true` 时）:


```js
// main.js
import './main-5532def0.js';

// main-5532def0.js
import('./dynamic-402de2f0.js');
const importantValue = 42;

export { importantValue as i };

// dynamic-402de2f0.js
import { i as importantValue } from './main-5532def0.js';

console.log(importantValue);
```

输出（在 `output.minifyInternalExports: false` 时）:


```js
// main.js
import './main-5532def0.js';

// main-5532def0.js
import('./dynamic-402de2f0.js');
const importantValue = 42;

export { importantValue };

// dynamic-402de2f0.js
import { importantValue } from './main-5532def0.js';

console.log(importantValue);
```

该选项值为 `true` 时，尽管表面上会导致代码输出变大，但是实际上，如果你使用压缩工具（minifier），代码会更小。在这种情况下，理论上，`export { importantValue as i }` 能够会被压缩成 `export{a as i}`，甚至是 `export{i}`，但是实际上是 `export{ a as importantValue }`，因为压缩工具通常不会改变导出签名。

#### output.paths
类型：`{ [id: string]: string } | ((id: string) => string)`

该选项用于将外部依赖映射为路径。其中，外部依赖是指该选项 [无法解析](guide/en/#warning-treating-module-as-external-dependency) 的模块或者通过 [`external`](guide/en/#external) 选项明确指定的模块。`output.paths` 提供的路径会取代模块 ID，在生成的 bundle 中使用，例如，你可以从 CDN 加载依赖：

```js
// app.js
import { selectAll } from 'd3';
selectAll('p').style('color', 'purple');
// ...

// rollup.config.js
export default {
  input: 'app.js',
  external: ['d3'],
  output: {
    file: 'bundle.js',
    format: 'amd',
    paths: {
      d3: 'https://d3js.org/d3.v4.min'
    }
  }
};

// bundle.js
define(['https://d3js.org/d3.v4.min'], function (d3) {

  d3.selectAll('p').style('color', 'purple');
  // ...

});
```

#### output.preserveModules
类型：`boolean`<br>
命令行参数：`--preserveModules`/`--no-preserveModules`<br>
默认值：`false`

该选项将使用原始模块名作为文件名，为所有模块创建单独的 chunk，而不是创建尽可能少的 chunk。它需要配合 [`output.dir`](guide/en/#outputdir) 选项一起使用。Tree-shaking 仍会对没有被入口使用或者执行阶段没有副作用的文件生效。该选项可以用于将文件结构转换为其他模块格式。

请注意，默认情况下，在转换为 `cjs` 或 `amd` 格式时，设置 [`output.exports`](guide/en/#outputexports) 的值为 `auto` 可以把每个文件会作为入口点。这意味着，例如对于 `cjs`，只包含默认导出的文件会将值直接赋值给 `module.exports`。

```js
// input main.js
export default 42;

// output main.js
'use strict';

var main = 42;

module.exports = main;
```

如果其他模块引入此文件，它们可以通过以下方式访问默认导出

```js
const main = require('./main.js');
console.log(main); // 42
```

与常规入口点一样，混合使用默认导出和命名导出的模块将会产生警告。你可以通过设置 `output.exports: "named"`，强制所有文件使用命名导出模式，来避免出现警告。在这种情况下，可以通过导出的 `.default` 属性访问默认导出：

```js
// 输入 main.js
export default 42;

// 输出 main.js
'use strict';

Object.defineProperty(exports, '__esModule', { value: true });

var main = 42;

exports.default = main;

// 使用
const main = require('./main.js');
console.log(main.default); // 42
```

#### output.sourcemap
类型：`boolean | 'inline' | 'hidden'`<br>
命令行参数：`-m`/`--sourcemap`/`--no-sourcemap`<br>
默认值：`false`

如果该选项值为 `true`，那么每个文件都将生成一个独立的 sourcemap 文件。如果值为 `“inline”`，那么 sourcemap 会以 data URI 的形式附加到输出文件末尾。如果值为 `“hidden”`，那么它的表现和 `true` 相同，不同的是，bundle 文件中没有 sourcemap 的注释。

#### output.sourcemapExcludeSources
类型：`boolean`<br>
命令行参数：`--sourcemapExcludeSources`/`--no-sourcemapExcludeSources`<br>
默认值：`false`

如果该选项的值为 `true`，那么实际源代码将不会被添加到 sourcemap 文件中，从而使其变得更小。

#### output.sourcemapFile
类型：`string`<br>
命令行参数：`--sourcemapFile <file-name-with-path>`

该选项用于指定生成 sourcemap 文件的位置。如果是一个绝对路径，那么 sourcemap 文件中的所有 `源文件（sources）` 路径都相对于该路径。`map.file` 属性是 `sourcemapFile` 的基本名称，因为 sourcemap 文件一般是和其构建后的 bundle 处于同一目录。

如果 `output` 设置了值，那么 `sourcemapFile` 不是必须的，这种情况下，`sourcemapFile` 的值会通过输出文件名中添加 “.map” 推断出来。

#### output.sourcemapPathTransform
类型：`(relativeSourcePath: string, sourcemapPath: string) => string`

该选项用于 sourcemap 的路径转换。其中，`relativeSourcePath` 是指从生成的 `.map` 文件到相对应的源文件的相对路径，而 `sourcemapPath` 是指生成源码映射文件的绝对路径。

```js
import path from 'path';
export default ({
  input: 'src/main',
  output: [{
    file: 'bundle.js',
    sourcemapPathTransform: (relativeSourcePath, sourcemapPath) => {
      // 将会把相对路径替换为绝对路径
      return path.resolve(path.dirname(sourcemapPath), relativeSourcePath)
    },
    format: 'es',
    sourcemap: true
  }]
});
```

#### preserveEntrySignatures
类型：`"strict" | "allow-extension" | false`<br>
命令行参数：`--preserveEntrySignatures <strict|allow-extension>`/`--no-preserveEntrySignatures`<br>
默认值：`"strict"`

该选项用于控制 Rollup 尝试确保入口块与基础入口模块具有相同的导出。

- 如果它的值设置为 `"strict"`，Rollup 将在入口 chunk 中创建与相应入口模块中完全相同的导出。如果因为需要向 chunk 中添加额外的内部导出而无法这样做，那么 Rollup 将创建一个 `facade` 入口 chunk，它将仅从前其他 chunk 中导出必要的绑定，但不包含任何其他代码。对于库，推荐使用此设置。
- 值为 `"allow-extension"`，则 Rollup 会将在入口 chunk 中创建入口模块的所有导出，但是如果有必要，还可以添加其他导出，从而避免出现 “facade” 入口 chunk。对于不需要严格签名的库，此设置很有意义。
- 值为 `false`，则 Rollup 不会将入口模块中的任何导出内容添加到相应的 chunk 中，甚至不包含相应的代码，除非这些导出内容在 bundle 的其他位置使用。但是，可以将内部导出添加到入口 chunks 中。对于将入口 chunks 放置在脚本标记中的 Web 应用，推荐使用该设置，因为它可能同时减少 bundle 的尺寸大小 和 chunks 的数量。

**示例**<br>
输入：

```js
// main.js
import { shared } from './lib.js';
export const value = `value: ${shared}`;
import('./dynamic.js');

// lib.js
export const shared = 'shared';

// dynamic.js
import { shared } from './lib.js';
console.log(shared);
```

`preserveEntrySignatures: "strict"` 时输出为：

```js
// main.js
export { v as value } from './main-50a71bb6.js';

// main-50a71bb6.js
const shared = 'shared';

const value = `value: ${shared}`;
import('./dynamic-cd23645f.js');

export { shared as s, value as v };

// dynamic-cd23645f.js
import { s as shared } from './main-50a71bb6.js';

console.log(shared);
```

`preserveEntrySignatures: "allow-extension"` 时输出为：

```js
// main.js
const shared = 'shared';

const value = `value: ${shared}`;
import('./dynamic-298476ec.js');

export { shared as s, value };

// dynamic-298476ec.js
import { s as shared } from './main.js';

console.log(shared);
```

`preserveEntrySignatures: false` 时输出为：

```js
// main.js
import('./dynamic-39821cef.js');

// dynamic-39821cef.js
const shared = 'shared';

console.log(shared);
```

目前，为独立的入口 chunks 覆盖此设置的唯一方法是，使用插件 API 并通过 [`this.emitFile`](guide/en/#thisemitfileemittedfile-emittedchunk--emittedasset--string) 触发这些 chunks，而不是使用 [`input`](guide/en/#input) 选项。

#### strictDeprecations
类型：`boolean`<br>
命令行参数：`--strictDeprecations`/`--no-strictDeprecations`<br>
默认值：`false`

启用此标志后，当使用废弃的功能时，Rollup 将抛出错误而不是警告。此外，如果使用了在下一个主要版本（major version）被标记为废弃警告的功能时，也会抛出错误。

例如，该选项被用于，插件作者能尽早地为即将发布的主要版本调整其插件配置。

### 慎用选项（Danger zone）

除非你知道自己在做什么，否则尽量别使用这些选项！

#### acorn
类型：`AcornOptions`

该选项用于指定要传递给 Acorn `parse` 函数的选项，比如 `allowReserved: true`。查看 [Acorn 文档](https://github.com/acornjs/acorn/tree/master/acorn#interface) 了解更多可用选项。

#### acornInjectPlugins
类型：`AcornPluginFunction | AcornPluginFunction[]`

该选项用于指定注入到 Acorn 中的单个插件或者插件数组。例如要支持 JSX 预发，你可以在你的 Rollup 配置通过该选项配置 jsx 插件，例如：

```javascript
import jsx from 'acorn-jsx';

export default {
    // … other options …
    acornInjectPlugins: [
        jsx()
    ]
};
```

请注意，这与使用 Babel 不同，因为生成的输出文件仍将包含 JSX，而 Babel 会将其替换为有效的 JavaScript。

#### context
类型：`string`<br>
命令行参数：`--context <contextVariable>`<br>
默认值：`undefined`

默认情况下，模块的上下文（即，全局 `this`）为 `undefined`。在极其少数情况下，你可能需要将其修改为其他名称，比如 `'window'`。

#### moduleContext
类型：`((id: string) => string) | { [id: string]: string }`<br>

与 [`context`](guide/en/#context) 相同，但每个模块要么是由 `id: context` 键值对构成的对象，要么是 `id => context` 函数。

#### output.amd
类型：`{ id?: string, define?: string}`

该选项的值可以是包含以下属性的对象：

**output.amd.id**<br>
类型：`string`<br>
命令行参数：`--amd.id <amdId>`

该选项用于指定 AMD/UMD bundles 的 ID：

```js
// rollup.config.js
export default {
  ...,
  format: 'amd',
  amd: {
    id: 'my-bundle'
  }
};

// -> define('my-bundle', ['dependency'], ...
```

**output.amd.define**<br>
类型：`string`<br>
命令行参数：`--amd.define <defineFunctionName>`

该选项用于指定代替 AMD 中的 `define` 的函数名称：

```js
// rollup.config.js
export default {
  ...,
  format: 'amd',
  amd: {
    define: 'def'
  }
};

// -> def(['dependency'],...
```

#### output.esModule
类型：`boolean`<br>
命令行参数：`--esModule`/`--no-esModule`<br>
默认值：`true`

该选项用于决定是否在生成非 ES（non-ES）格式导出时添加 `__esModule: true` 属性。

#### output.exports
类型：`string`<br>
命令行参数：`--exports <exportMode>`<br>
默认值：`'auto'`

该选项用于指定导出模式。默认是 `auto`，指根据 `input` 模块导出推测你的意图：

* `default` - 适用于只使用 `export default ...` 的情况
* `named` - 适用于导出超过一个模块的情况
* `none` - 适用于没有导出的情况（比如，当你在构建应用而非库时）

`default` 和 `named` 之间的差异会影响其他人使用你的 bundle 的方式。例如，如果该选项的值时 `default`，那么 CommonJS 用户可以通过以下方式使用你的库：

```js
const yourLib = require( 'your-lib' );
```

如果该选项的值是 `named`，那么用户通过以下方式使用你的库：

```js
const yourMethod = require( 'your-lib' ).yourMethod;
```

存在问题的是，如果你使用 `named` 导出但还具有 `default` 导出，则用户必须使用以下方式才能使用你的库的默认导出（default export）：

```js
const yourMethod = require( 'your-lib' ).yourMethod;
const yourLib = require( 'your-lib' ).default;
```

#### output.externalLiveBindings
类型：`boolean`<br>
命令行参数：`--externalLiveBindings`/`--no-externalLiveBindings`<br>
默认值：`true`

当该选项的值为 `false` 时，Rollup 不会为外部依赖生成支持动态绑定的代码，而是假定外部依赖永远不会改变。这使得 Rollup 会生成更多优化代码。请注意，当外部依赖存在循环引用时，该选项值为 `false` 可能会引起问题。

在大多数情况下，该选项值为 `false` 将避免 Rollup 生成多余代码的 getters，因此在很多情况下，可以使代码兼容 IE8。

例：

```js
// 输入
export {x} from 'external';

// 使用 externalLiveBindings: true 选项时 CJS 格式的输出
'use strict';

Object.defineProperty(exports, '__esModule', { value: true });

var external = require('external');

Object.defineProperty(exports, 'x', {
	enumerable: true,
	get: function () {
		return external.x;
	}
});

// 使用 externalLiveBindings: false 选项时 CJS 格式的输出
'use strict';

Object.defineProperty(exports, '__esModule', { value: true });

var external = require('external');

exports.x = external.x;
```

#### output.freeze
类型：`boolean`<br>
命令行参数：`--freeze`/`--no-freeze`<br>
默认值：`true`

该选项用于指定是否使用 `Object.freeze()` 冻结动态访问的引入对象的命名空间，例如 `import * as namespaceImportObject from...`。

#### output.indent
类型：`boolean | string`<br>
命令行参数：`--indent`/`--no-indent`<br>
默认值：`true`

该选项用于指定在 `amd`，`iife`，`umd`，`system` 格式中代码缩进的缩进字符串。它的值可以是 `false`（没有缩进）或 `true`（默认值 - 自动缩进）。

```js
// rollup.config.js
export default {
  ...,
  output: {
    ...,
    indent: false
  }
};
```

#### output.namespaceToStringTag
类型：`boolean`<br>
命令行参数：`--namespaceToStringTag`/`--no-namespaceToStringTag`<br>
默认值：`false`

该选项用于指定是否将符合规范的 `.toString` 标签加到命名空间对象。如果该选项设置为 `true`，那么下列代码会输出为 `[object Module]`。

```javascript
import * as namespace from './file.js';
console.log(String(namespace));
```

#### output.noConflict
类型：`boolean`<br>
命令行参数：`--noConflict`/`--no-noConflict`<br>
默认值：`false`

该选项默认将生成额外的 `noConflict` 导出到 UMD 格式的 bundle。在 IIFE 场景中调用时，此方法将返回 bundle 的导出，同时将相应的全部变量恢复为先前的值。

#### output.preferConst
类型：`boolean`<br>
命令行参数：`--preferConst`/`--no-preferConst`<br>
默认值：`false`

该选项用于指定为导出生成 `const` 声明，而不是 `var` 声明。

#### output.strict
类型：`boolean`<br>
命令行参数：`--strict`/`--no-strict`<br>
默认值：`true`

该选项用于决定是否在生成非 ES bundle 的顶部包含 “use strict” 用法。严格地讲，ES 模块总是使用严格模式，所以不要无缘无故禁用该选项。

#### output.systemNullSetters
类型：`boolean`<br>
命令行参数：`--systemNullSetters`/`--no-systemNullSetters`<br>
默认值：`false`

在导出 `system` 模块格式时，该选项将代替空的 setter 函数，以 `null` 形式简化输出。该选项仅在 *SystemJS 6.3.3 及以上版本中支持*。

#### preserveSymlinks
类型：`boolean`<br>
命令行参数：`--preserveSymlinks`<br>
默认值：`false`

当该选项值为 `false` 时，引用的文件为软链接实际指向的文件。当该选项值为 `true` 时，引入的文件为软链接所在目录的文件。为了更好的解释，想想以下例子：

```js
// /main.js
import {x} from './linked.js';
console.log(x);

// /linked.js
// 这是 /nested/file.js 的软链接

// /nested/file.js
export {x} from './dep.js';

// /dep.js
export const x = 'next to linked';

// /nested/dep.js
export const x = 'next to original';
```

如果 `preserveSymlinks` 值为 `false`，那么从 `/main.js` 将会输出 “next to original”，因为它将使用软链接文件的位置来解决其依赖。然而，如果 `preserveSymlinks` 值为 `true`，那么它将会输出 “next to linked”，因为软链接将无法正确解析。

#### shimMissingExports
类型：`boolean`<br>
命令行参数：`--shimMissingExports`/`--no-shimMissingExports`<br>
默认值：`false`

如果该选项值为 `true`，那么从未定义绑定的文件中引入依赖时，打包不会失败。相反，将为这些绑定创建值为 `undefined` 的新变量。

#### treeshake
类型：`boolean | { annotations?: boolean, moduleSideEffects?: ModuleSideEffectsOption, propertyReadSideEffects?: boolean, tryCatchDeoptimization?: boolean, unknownGlobalSideEffects?: boolean }`<br>
命令行参数：`--treeshake`/`--no-treeshake`<br>
默认值：`true`

该选项用于决定是否应用 tree-shake 并微调 tree-shake 过程。该选项的值设置为 `false` 时，Rollup 将生成更大的 bundle，但是可能会提高构建性能。如果你发现由 tree-shake 造成的 bug，请给我们提 issue ！
将此选项的值设置为对象意味着启用 tree-shaking，并启用以下选项：

**treeshake.annotations**<br>
类型：`boolean`<br>
命令行参数：`--treeshake.annotations`/`--no-treeshake.annotations`<br>
默认值：`true`

如果该选项值为 `false`，那么在确定函数调用和构造函数调用的副作用时，将会忽略纯注释的提示，比如，包含 `@__PURE__` 或 `#__PURE__` 的注释。这些注释需要紧接在调用代码之前才能生效。该选项的值设置为 `false`，以下代码将原封不动，否则以下包含 `@__PURE__` 注释的代码将被完全删除。

```javascript
/*@__PURE__*/console.log('side-effect');

class Impure {
  constructor() {
    console.log('side-effect')
  }
}

/*@__PURE__*/new Impure();
```

**treeshake.moduleSideEffects**<br>
类型：`boolean | "no-external" | string[] | (id: string, external: boolean) => boolean`<br>
命令行参数：`--treeshake.moduleSideEffects`/`--no-treeshake.moduleSideEffects`/`--treeshake.moduleSideEffects no-external`<br>
默认值：`true`

如果该选项的值为 `false`，则假定，像改变全局变量或不执行检查就记录等行为一样，没有引入任何内容的模块和外部依赖没有其他副作用。对于外部依赖，该选项将影响未使用的 import：

```javascript
// input file
import {unused} from 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// output with treeshake.moduleSideEffects === true
import 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// output with treeshake.moduleSideEffects === false
console.log(42);
```

对于非外部依赖模块，该选项值为 `false` 时，除非来自改模块的 import 被使用，否则输出中将不会包含来自该模块的任何语句：

```javascript
// input file a.js
import {unused} from './b.js';
console.log(42);

// input file b.js
console.log('side-effect');
```

```javascript
// output with treeshake.moduleSideEffects === true
console.log('side-effect');

console.log(42);
```

```javascript
// output with treeshake.moduleSideEffects === false
console.log(42);
```

该选项的值也可以是具有副作用的模块列表，或者是一个返回指定模块的函数。该选项的值为 `“no-external”` 将意味着如果可能，则仅删除外部依赖，它等同于函数 `(id, external) => !external`；

**treeshake.propertyReadSideEffects**
类型：`boolean`<br>
命令行参数：`--treeshake.propertyReadSideEffects`/`--no-treeshake.propertyReadSideEffects`<br>
默认值：`true`

如果该选项值为 `false`，则假定读取对象的属性将永远不会有副作用。根据你的代码，禁用该属性能够显著缩小 bundle 的大小，但是如果你依赖了 geters 或非法属性访问的造成的错误，那么可能会破坏该功能。

```javascript
// Will be removed if treeshake.propertyReadSideEffects === false
const foo = {
  get bar() {
    console.log('effect');
    return 'bar';
  }
}
const result = foo.bar;
const illegalAccess = foo.quux.tooDeep;
```

**treeshake.tryCatchDeoptimization**
类型：`boolean`<br>
命令行参数：`--treeshake.tryCatchDeoptimization`/`--no-treeshake.tryCatchDeoptimization`<br>
默认值：`true`

默认情况下，Rollup 假定在进行 tree-shaking 是，很多运行时内置的全局变量的行为均符合最新规范，并且不会引发意外错误。为了支持，像依赖于抛出错误的特征检测工作流，Rollup 将默认禁用 try 语句中的 tree-shaking。如果函数参数在 try 语句中被使用，那么该参数也不会被优化处理。如果你不需要此功能，想要在 try 语句中使用 tree-shaking，则可以设置 `treeshake.tryCatchDeoptimization` 为 `false`。

```js
function otherFn() {
  // 尽管该函数时在 try 语句中使用，但是下列代码
  // 依然会被当做 side-effect-free 而被移除
  Object.create(null);
}

function test(callback) {
  try {
    // 对于 tryCatchDeoptimization: true , 在 try 语句中的
    // 全局函数不会遵循 side-effect-free，而是会被保留
    Object.create(null);

    // 其他函数的调用也将被保留，但是函数内部代码
    // 可能会支持 tree-shaking
  	// calls to other function are retained as well but the body of this
  	// function may again be subject to tree-shaking
    otherFn();

    // 如果函数类型参数被调用，那么所有被该函数使用的
    // 参数将不会被优化
    callback();
  } catch {}
}

test(() => {
  // 会被保留
  Object.create(null)
});

// 调用会被保留，而 otherFn 不会被优化
test(otherFn);

```

**treeshake.unknownGlobalSideEffects**
类型：`boolean`<br>
命令行参数：`--treeshake.unknownGlobalSideEffects`/`--no-treeshake.unknownGlobalSideEffects`<br>
默认值：`true`

因为访问不存在的全局变量会引起错误，所以默认情况下 Rollup 保留对非内置全局变量的所有访问。该选项的值设置为 `false` 可以避免该检查。对于大多数代码库而言，这样做很可能更安全。

```js
// input
const jQuery = $;
const requestTimeout = setTimeout;
const element = angular.element;

// 输出（当 unknownGlobalSideEffects == true 时）
const jQuery = $;
const element = angular.element;

// 输出（当 unknownGlobalSideEffects == false 时）
const element = angular.element;
```

在这个例子中，最后一行将被始终保留，用于访问 `element` 属性，但如果 `angular` 值为 `null`，也可能抛出错误。为了避免这种情况的检查，可以设置 `treeshake.propertyReadSideEffects` 为 `false`。

### 实验选项(Experimental options)

这些选项反应了尚未完全确定的新功能。因此，它们的可行性、行为和用法在次要版本中可能发生变化。

#### experimentalCacheExpiry
类型：`number`<br>
命令行参数：`--experimentalCacheExpiry <numberOfRuns>`<br>
默认值：`10`

该选项用于确定在多少次执行以后，应该删除不再被插件使用的静态缓存。

#### perf
类型：`boolean`<br>
命令行参数：`--perf`/`--no-perf`<br>
默认值：`false`

该选项用于决定是否收集打包执行耗时。当使用命令行或者配置文件时，将会展示与当前构建过程有关的详细指标。当在 [JavaScript API](guide/en/#javascript-api) 中使用时，返回的 bundle 对象将包含额外的 `getTimings()` 函数，可以随时调用该函数来获取所有累计的指标。

`getTimings()` 返回以下对象形式：

```json
{
  "# BUILD": [ 698.020877, 33979632, 45328080 ],
  "## parse modules": [ 537.509342, 16295024, 27660296 ],
  "load modules": [ 33.253778999999994, 2277104, 38204152 ],
  ...
}
```

对于每个键的值，是一个数组，其中，第一个数值表示经过的时间，第二个数值表示内存消耗的变化，第三个数值表示此步骤完成后的总内存消耗。这些步骤的顺序是通过 `Object.keys` 确定的。Top level 键以 `#` 开头，包含嵌套步骤的耗时，例如，在上面例子中的 `# BUILD` 步骤耗时（698ms）包含了 `## parse modules` 步骤的耗时（539ms）。

### 观察选项（Watch options）

类型：`{ buildDelay?: number, chokidar?: ChokidarOptions, clearScreen?: boolean, exclude?: string, include?: string, skipWrite?: boolean } | false`<br>
默认值：`{}`<br>

该选项用于指定 watch 模式的选项，或防止 Rollup 配置被 watch。指定该选项为 `false` ，仅对 Rollup 使用数组配置时有用。在这种情况下，Rollup 配置将不会根据观察模式中的变更构建或重新构建，而是在 Rollup 运行时定期构建。

```js
// rollup.config.js
export default [
  {
    input: 'main.js',
    output: { file: 'bundle.cjs.js', format: 'cjs' }
  },
  {
    input: 'main.js',
    watch: false,
    output: { file: 'bundle.es.js', format: 'es' }
  }
]
```

这些选项仅在使用 `--watch` 标志或使用 `rollup.watch` 运行 Rollup 时生效。

#### watch.buildDelay
类型：`number`<br>
命令行参数：`--watch.buildDelay <number>`<br>
默认值：`0`

该选项用于配置 Rollup 触发重新构建（rebuild）到执行进一步构建需要等待的时间（以毫秒为单位）。默认情况下，Rollup 不会等待，但是在 chokidar 实例中配置了一个小的抖动时间（debounce timeout）。该选项的值大于 `0` 将意味着如果配置的毫秒数没有发生变化，Rollup 只会触发一次重新构建。如果观察到多个配置变化，Rollup 将使用配置的最大构建延迟。

#### watch.chokidar
类型：`ChokidarOptions`<br>

在观察选项中，该选项是可选对象，将传递给 [chokidar](https://github.com/paulmillr/chokidar) 实例。查阅 [chokidar documentation](https://github.com/paulmillr/chokidar#api) 文档以了解可用的选项。

#### watch.clearScreen
类型：`boolean`<br>
命令行参数：`--watch.clearScreen`/`--no-watch.clearScreen`<br>
默认值：`true`

该选项用于决定在触发重建是是否清除屏幕。

#### watch.exclude
类型：`string`<br>
命令行参数：`--watch.exclude <files>`

该选项用于指定不需要被 watch 的文件：

```js
// rollup.config.js
export default {
  ...,
  watch: {
    exclude: 'node_modules/**'
  }
};
```

#### watch.include
类型：`string`<br>
命令行参数：`--watch.include <files>`

该选项用于限制只能对指定文件进行观察。请注意，该选项只过滤 module graph 中的文件，不在其中的文件将不会被观察：

```js
// rollup.config.js
export default {
  ...,
  watch: {
    include: 'src/**'
  }
};
```

#### watch.skipWrite
类型：`boolean`<br>
命令行参数：`--watch.skipWrite`/`--no-watch.skipWrite`<br>
默认值：`false`

该选项用于决定是否在触发重新构建时忽略 `bundle.write()` 步骤。

### 废弃选项（Deprecated options）

☢️ 这些选项已经废弃，可能从未来的 Rollup 版本中移除。

#### inlineDynamicImports
请使用具有相同签名的 [`output.inlineDynamicImports`](guide/en/#outputinlinedynamicimports) 选项代替。

#### manualChunks
请使用具有相同签名的 [`output.manualChunks`](guide/en/#outputmanualchunks) 选项代替。

#### preserveModules
请使用具有相同签名的 [`output.preserveModules`](guide/en/#outputpreservemodules) 选项代替。

#### output.dynamicImportFunction
请使用 [`renderDynamicImport`](guide/en/#renderdynamicimport) 插件 hook 代替。<br>
类型：`string`<br>
命令行参数：`--dynamicImportFunction <name>`<br>
默认值：`import`

当输出为 ES bundle 时，该选项将会把动态引入函数重命名为该选项指定的名称。这对于使用了 dynamic import polyfil 的代码非常有用，比如[这个 dynamic import polyfill 库](https://github.com/uupaa/dynamic-import-polyfill)。

#### treeshake.pureExternalModules
请使用 [`treeshake.moduleSideEffects: 'no-external'`](guide/en/#treeshake) 选项代替。<br>
类型：`boolean | string[] | (id: string) => boolean | null`<br>
命令行参数：`--treeshake.pureExternalModules`/`--no-treeshake.pureExternalModules`<br>
默认值：`false`

如果该选项的值为 `true`, 那么Rollup 会假定没有任何引入的外部依赖没有其他副作用，就如修改全局变量或者记录日志。

```javascript
// 输入文件
import {unused} from 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// 在 treeshake.pureExternalModules === false 时的输出
import 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// 在 treeshake.pureExternalModules === true 时的输出
console.log(42);
```

该选项也支持一个被视为纯净的外部依赖的 id 列表，或者是一个可以在移除外部引入时调用的函数。
