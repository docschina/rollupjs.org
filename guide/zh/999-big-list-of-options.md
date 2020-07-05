---
title: Big list of options
---

### 核心功能 (Core functionality)

#### 外部依赖 (external)
类型: `(string | RegExp)[] | RegExp | string | (id: string, parentId: string, isResolved: boolean) => boolean`<br>
命令参数: `-e`/`--external <external-id,another-external-id,...>`

external 用于排除一些不需要打包到 bundle 中的模块，它的值可以是一个函数，这个函数接受模块 ID `id` 参数并且返回 `true`（表示排除）或 `false`（表示包含），也可以是一个由模块ID构成的数组，还可以是可以匹配到模块 ID 的正则表达式。除此之外。external 还可以取值单个模块 ID 或者 单个正则表达式。匹配得到的模块 ID 应该满足以下条件之一：
Either a function that takes an `id` and returns `true` (external) or `false` (not external), or an `Array` of module IDs, or regular expressions to match module IDs, that should remain external to the bundle. Can also be just a single ID or regular expression. The matched IDs should be either:

1. import 语句中外部依赖的包名。比如，如果标记 `import "dependency.js"` 为外部依赖，那么模块 ID 使用 `"dependency.js"`，而如果标记 `import "dependency"` 为外部依赖，那么模块 ID 就是 `"dependency"`。
1. the name of an external dependency, exactly the way it is written in the import statement. I.e. to mark `import "dependency.js"` as external, use `"dependency.js"` while to mark `import "dependency"` as external, use `"dependency"`.
2. 绝对路径。（比如文件的绝对路径）
2. a resolved ID (like an absolute path to a file).

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

注意，如果你想要将 `node_modules` 中导入的包当成外部依赖处理，你须要提前导入 [@rollup/plugin-node-resolve](https://github.com/rollup/plugins/tree/master/packages/node-resolve)，这样才能将 `import {rollup} from 'rollup'` 中 通过 `/node_modules/` 正则表达式匹配到的 rollup 当成外部依赖处理。
Note that if you want to filter out package imports, e.g. `import {rollup} from 'rollup'`, via a `/node_modules/` regular expression, you need something like [@rollup/plugin-node-resolve](https://github.com/rollup/plugins/tree/master/packages/node-resolve) to resolve the imports to `node_modules` first.

当用作命令行参数时，应该用逗号分隔的模块 ID 列表：
When given as a command line argument, it should be a comma-separated list of IDs:

```bash
rollup -i src/main.js ... -e foo,bar,baz
```

使用函数时会提供三个参数 `(id, parent, isResolved)` ，这些参数可以为您提供更细粒度的控制：
When providing a function, it is actually called with three parameters `(id, parent, isResolved)` that can give you more fine-grained control:

* `id` 指相关模块中的 ID
* `id` is the id of the module in question
* `parent` 指执行导入的模块的 ID
* `parent` is the id of the module doing the import
* `isResolved` 指是否已经通过插件等方式解决模块依赖
* `isResolved` signals whether the `id` has been resolved by e.g. plugins

创建 iife 或 umd 包时，您需要通过 `output.globals` 选项来提供全局变量名称，以替换外部导入。
When creating an `iife` or `umd` bundle, you will need to provide global variable names to replace your external imports via the `output.globals` option.

如果是相对导入（即以 `./` 或 `../`开头）模块被标记为”外部依赖“，rollup 会把它们处理成系统绝对文件路径，以便不同的外部模块可以合并到一起。当写入 bundle 以后，它们将会再次被转换为相对导入。举例：
If a relative import, i.e. starting with `./` or `../`, is marked as "external", rollup will internally resolve the id to an absolute file system location so that different imports of the external module can be merged. When the resulting bundle is written, the import will again be converted to a relative import. Example:

```js
// 输入
// input
// src/main.js （入口文件）
// src/main.js (entry point)
import x from '../external.js';
import './nested/nested.js';
console.log(x);

// src/nested/nested.js
// 如果导入依赖已存在，它们将指向同一个文件
// the import would point to the same file if it existed
import x from '../../external.js';
console.log(x);

// 输出
// output
// 不同的依赖将会被合并
// the different imports are merged
import x from '../external.js';

console.log(x);

console.log(x);
```

如果存在多个入口，rollup 会把它们转换会相对导入的方式，就像 `output.file` 或 `output.dir` 入口文件或所有入口文件的公共目录。
The conversion back to a relative import is done as if `output.file` or `output.dir` were in the same location as the entry point or the common base directory of all entry points if there is more than one.

#### 入口 (input)
类型: `string | string [] | { [entryName: string]: string }`<br>
命令行参数: `-i`/`--input <filename>`

input 是指打包的入口文件，比如 `main.js` 或 `app.js` 或 `index.js` 文件。如果你使用数组或者对象作为 input，那么它们将被打包到单独的输出区块（chunks）。除非使用 [`output.file`](guide/en/#outputfile) 选项，否则生成的区块名称会根据 [`output.entryFileNames`](guide/en/#outputentryfilenames) 选项来确定。使用对象形式时，文件名中的 `[name]` 将作为对象属性的一部分，而对于数组形式，它将作为入口的文件名。
The bundle's entry point(s) (e.g. your `main.js` or `app.js` or `index.js`). If you provide an array of entry points or an object mapping names to entry points, they will be bundled to separate output chunks. Unless the [`output.file`](guide/en/#outputfile) option is used, generated chunk names will follow the [`output.entryFileNames`](guide/en/#outputentryfilenames) option. When using the object form, the `[name]` portion of the file name will be the name of the object property while for the array form, it will be the file name of the entry point.

请注意，使用对象形式时，只要在入口名称中添加 '/'，就可以将入口打包到不同的子文件夹中。下面是个例子，根据 `entry-a.js` 和 `entry-b/index.js`，至少产生两个入口区块（chunks），其中 `index.js` 将输出在 `entry-b` 文件夹中：
Note that it is possible when using the object form to put entry points into different sub-folders by adding a `/` to the name. The following will generate at least two entry chunks with the names `entry-a.js` and `entry-b/index.js`, i.e. the file `index.js` is placed in the folder `entry-b`:

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

在使用命令行操作时，多个入口只需要提供多个选项即可。当作为第一个选项提供时，等同于不给它们加上 `--input` 选项：
When using the command line interface, multiple inputs can be provided by using the option multiple times. When provided as the first options, it is equivalent to not prefix them with `--input`:

```sh
rollup --format es --input src/entry1.js --input src/entry2.js
# 等同于
# is equivalent to
rollup src/entry1.js src/entry2.js --format es
```

可以通过 `=` 为入口命名：
Chunks can be named by adding an `=` to the provided value:

```sh
rollup main=src/entry1.js other=src/entry2.js --format es
```

可以使用引号指定包含空格的文件名：
File names containing spaces can be specified by using quotes:

```sh
rollup "main entry"="src/entry 1.js" "src/other entry.js" --format es
```

#### 输出目录 (output.dir)
类型: `string`<br>
命令行参数: `-d`/`--dir <dirname>`

指定所有生产文件所在的目录。如果生成多个 chunk，则此选项是必须的。否则，可以使用 `file` 选项代替。
The directory in which all generated chunks are placed. This option is required if more than one chunk is generated. Otherwise, the `file` option can be used instead.

#### 输出文件 (output.file)
类型: `string`<br>
命令行参数: `-o`/`--file <filename>`

指定要写入的文件名。如果该选项生效时，那么同时也会产生 sourcemaps 。只有当生成的 chunk 不超过一个时，该选项才会生效。
The file to write to. Will also be used to generate sourcemaps, if applicable. Can only be used if not more than one chunk is generated.

#### 输出格式 (output.format)
类型: `string`<br>
命令行参数: `-f`/`--format <formatspecifier>`

指定产生 bundle 的格式。以下之一：
Specifies the format of the generated bundle. One of the following:

* `amd` - 异步模块定义，适用于 RequireJS 等
* `amd` – Asynchronous Module Definition, used with module loaders like RequireJS
* `cjs` - CommonJS，适用于 Node 以及其他打包器（别名：`commonjs`）
* `cjs` – CommonJS, suitable for Node and other bundlers (alias: `commonjs`)
* `es` - 将 bundle 保留为 ES 模块文件，适用于其他打包器以及支持 `<script type=module>` 标签的浏览器（别名: `esm`，`module`）
* `es` – Keep the bundle as an ES module file, suitable for other bundlers and inclusion as a `<script type=module>` tag in modern browsers (alias: `esm`, `module`)
* `iife` - 自执行函数，适用于 `<script>` 标签。（如果你要为你的应用创建 bundle，那么你很可能用它。）
* `iife` – A self-executing function, suitable for inclusion as a `<script>` tag. (If you want to create a bundle for your application, you probably want to use this.)
* `umd` - 通用模块定义，生成支持 `amd`、`cjs` 和 `iife` 三种格式
* `umd` – Universal Module Definition, works as `amd`, `cjs` and `iife` all in one
* `system` - SystemJS 加载器的格式（别名: `systemjs`）
* `system` – Native format of the SystemJS loader (alias: `systemjs`)

#### 输出全局变量 (output.globals)
类型: `{ [id: string]: string } | ((id: string) => string)`<br>
命令行参数: `-g`/`--globals <external-id:variableName,another-external-id:anotherVariableName,...>`

在使用 `umd` 或 `iife` 时，使用 `id: variableName` 键值对指定外部依赖。下面就是一个外部依赖的例子：
Specifies `id: variableName` pairs necessary for external imports in `umd`/`iife` bundles. For example, in a case like this…

```js
import $ from 'jquery';
```

在这个例子中，我们想要告诉 Rollup，`jquery`是外部依赖，并且 `jquery` 模块的全局 ID 为 `$`：
…we want to tell Rollup that `jquery` is external and the `jquery` module ID equates to the global `$` variable:

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
Alternatively, supply a function that will turn an external module ID into a global variable name.

在作为命令行参数时，它应该是以逗号分隔的 `id:variableName` 键值对：
When given as a command line argument, it should be a comma-separated list of `id:variableName` pairs:

```
rollup -i src/main.js ... -g jquery:$,underscore:_
```

要告诉 Rollup 用全局变量替换本地文件，请使用绝对 ID：
To tell Rollup that a local file should be replaced by a global variable, use an absolute id:

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

#### 输出命名 (output.name)
类型: `string`<br>
命令行: `-n`/`--name <variableName>`

在想要使用全局变量来表示你的 bundle 时，输出格式必须指定为 `iife` 或 `umd`。同一个页面上的其他脚本可以通过这个全局便来来访问你的 bundle 导出。
Necessary for `iife`/`umd` bundles that exports values in which case it is the global variable name  representing your bundle. Other scripts on the same page can use this variable name to access the exports of your bundle.

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

输出命名也支持命名空间，可以通过 `.` 访问。bundle 将根据设置生成对应的命名空间。
Namespaces are supported i.e. your name can contain dots. The resulting bundle will contain the setup necessary for the namespacing.

```
rollup -n "a.b.c"

/* ->
this.a = this.a || {};
this.a.b = this.a.b || {};
this.a.b.c = ...
*/
```

#### 输出插件 (output.plugins)
类型: `OutputPlugin | (OutputPlugin | void)[]`

指定输出插件，这是唯一设置的地方。你查看 [Using output plugins](guide/en/#using-output-plugins) 了解更多关于如何设置指定输出插件的信息，[Plugins](guide/en/#plugin-development) 会告诉你如何写一个你自己的插件。对于从包中导入的插件，请记住调用导入的函数（例如，应该使用 `commonjs()`，而不是 `commonjs`）。如果函数的值返回为 Falsy，那么该插件将会被忽略，这样可以灵活启用和禁用插件。
Adds a plugin just to this output. See [Using output plugins](guide/en/#using-output-plugins) for more information on how to use output-specific plugins and [Plugins](guide/en/#plugin-development) on how to write your own. For plugins imported from packages, remember to call the imported plugin function (i.e. `commonjs()`, not just `commonjs`). Falsy plugins will be ignored, which can be used to easily activate or deactivate plugins.

注意，并不是所有的插件都可以通过该选项使用。只有在 Rollup 主分析阶段完成以后，调用在 `bundle.generate()` 或者 `bundle.write()` 阶段的 hooks 的插件才可以使用该选项。如果你是一个插件的作者，你可以查看 [output generation hooks](guide/en/#output-generation-hooks) 了解更多关于 hooks 的使用方法。
Not every plugin can be used here. `output.plugins` is limited to plugins that only use hooks that run during `bundle.generate()` or `bundle.write()`, i.e. after Rollup's main analysis is complete. If you are a plugin author, see [output generation hooks](guide/en/#output-generation-hooks) to find out which hooks can be used.

以下是一个使用压缩插件的例子：
The following will add minification to one of the outputs:

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

#### 插件 (plugins)
类型: `Plugin | (Plugin | void)[]`

查看 [Using plugins](guide/en/#using-plugins) 了解更多关于如何使用插件的信息，其中根据 [Plugins](guide/en/#plugin-development)，你可以写一个自定义插件（动手试试看，写一个插件并不困难，你可以通过 Rollup 插件做很多拓展）。对于从包中导入的插件，请记住调用导入的函数（例如，应该使用 `commonjs()`，而不是 `commonjs`）。如果插件函数的值返回为 Falsy，那么该插件将会被忽略，这样可以灵活启用和禁用插件。
See [Using plugins](guide/en/#using-plugins) for more information on how to use plugins and [Plugins](guide/en/#plugin-development) on how to write your own (try it out, it's not as difficult as it may sound and very much extends what you can do with Rollup). For plugins imported from packages, remember to call the imported plugin function (i.e. `commonjs()`, not just `commonjs`). Falsy plugins will be ignored, which can be used to easily activate or deactivate plugins.

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

（上述例子还演示了如何使用异步 IIFE 和动态导入来避免导入会使 Rollup 打包变慢的不必要的模块。）
(This example also demonstrates how to use an async IIFE and dynamic imports to avoid unnecessary module loading, which can be surprisingly slow.)

### 进阶功能（Advanced functionality）

#### 缓存（cache）
类型: `RollupCache | false`

指定之前的 bundle 的缓存。用于加速观察模式（watch mode）下的后续构建 - 这样 Rollup 只会对改变的部分重新分析。将此选项显式设置为 `false` 将会阻止 bundle 产生缓存，还会使插件的缓存失效。
The `cache` property of a previous bundle. Use it to speed up subsequent builds in watch mode — Rollup will only reanalyse the modules that have changed. Setting this option explicitly to `false` will prevent generating the `cache` property on the bundle and also deactivate caching for plugins.

```js
const rollup = require('rollup');
let cache;

async function buildWithCache() {
  const bundle = await rollup.rollup({
    cache, // is ignored if falsy 如果 cache 值为 falsy，那么该选项会被忽略
    // ... other input options 其他输入选项
  });
  cache = bundle.cache; // store the cache object of the previous build 保存之前构建的数据缓存
  return bundle;
}

buildWithCache()
  .then(bundle => {
    // ... do something with the bundle
  })
  .then(() => buildWithCache()) // will use the cache of the previous build 将会使用之前构建的缓存
  .then(bundle => {
    // ... do something with the bundle
  })
```

#### 警告（onwarn）
类型: `(warning: RollupWarning, defaultHandler: (warning: string | RollupWarning) => void) => void;`

拦截警告消息的功能。如果未提供，那么警告将会去重并打印在控制台（console）。当命令行参数中使用 [`--silent`](guide/en/#--silent)，该函数是唯一能够获取警告通知的方法。
A function that will intercept warning messages. If not supplied, warnings will be deduplicated and printed to the console. When using the [`--silent`](guide/en/#--silent) CLI option, this handler is the only way to get notified about warnings.

该函数接受两个参数：警告对象（warning object）和默认处理函数（default handler）。警告对象至少包含 `code` 和 `message` 两个属性，使你可以控制如何处理不同类型的警告。另外，根据不同的警告类型，警告对象上会有其他的属性。
The function receives two arguments: the warning object and the default handler. Warnings objects have, at a minimum, a `code` and a `message` property, allowing you to control how different kinds of warnings are handled. Other properties are added depending on the type of warning.

```js
// rollup.config.js
export default {
  ...,
  onwarn (warning, warn) {
    // 忽略指定类型的警告
    // skip certain warnings
    if (warning.code === 'UNUSED_EXTERNAL_IMPORT') return;

    // 抛出其他类型错误
    // throw on others
    if (warning.code === 'NON_EXISTENT_EXPORT') throw new Error(warning.message);

    // 使用默认处理函数兜底
    // Use default for everything else
    warn(warning);
  }
};
```

许多警告还具有 `loc` 和 `frame` 属性，它们可以用来定位警告来源。
Many warnings also have a `loc` property and a `frame` allowing you to locate the source of the warning:

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


#### 静态文件名（output.assetFileNames）
类型: `string`<br>
命令行参数: `--assetFileNames <pattern>`<br>
默认值: `"assets/[name]-[hash][extname]"`

用于自定义构建结果中的静态文件名称。它支持以下占位符：
The pattern to use for naming custom emitted assets to include in the build output. Pattern supports the following placeholders:
 * `[extname]`: 包含点的静态文件扩展名，例如：`.css`。
 * `[extname]`：The file extension of the asset including a leading dot, e.g. `.css`.
 * `[ext]`：不包含点的文件扩展名，例如：`css`。
 * `[ext]`: The file extension without a leading dot, e.g. `css`.
 * `[hash]`：基于静态文件的名称和内容的哈希。
 * `[hash]`: A hash based on the name and content of the asset.
 * `[name]`: 静态文件的名称，不包含扩展名。
 * `[name]`: The file name of the asset excluding any extension.

正斜杆 `/` 可以用来划分文件到子目录。又见[`output.chunkFileNames`](guide/en/#outputchunkfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames)。
Forward slashes `/` can be used to place files in sub-directories. See also [`output.chunkFileNames`](guide/en/#outputchunkfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames).

#### output.banner/output.footer
类型: `string | (() => string | Promise<string>)`<br>
命令行参数: `--banner`/`--footer <text>`

用于在构建结构前或后加一个字符串。当然，它也支持返回一个字符串的异步函数。（注意：`banner` 和 `footer` 选项不会破坏 sourcemaps。）
A string to prepend/append to the bundle. You can also supply a function that returns a `Promise` that resolves to a `string` to generate it asynchronously (Note: `banner` and `footer` options will not break sourcemaps).

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
See also [`output.intro/output.outro`](guide/en/#outputintrooutputoutro).

#### output.chunkFileNames
类型: `string`<br>
命令行参数: `--chunkFileNames <pattern>`<br>
默认值: `"[name]-[hash].js"`

该选项用于对代码分割中产生的文件自定义命名。它支持以下形式：
The pattern to use for naming shared chunks created when code-splitting. Pattern supports the following placeholders:
 * `[format]`：输出（output）选项中定义的 `format` 的值，例如：`es` 或 `cjs`。
 * `[format]`: The rendering format defined in the output options, e.g. `es` or `cjs`.
 * `[hash]`：基于 chunk 文件本身的内容，再加上所有它依赖的文件的内容，组成的哈希。
 * `[hash]`: A hash based on the content of the chunk and the content of all its dependencies.
 * `[name]`：chunk 的名字。它可以通过 [`output.manualChunks`](guide/en/#outputmanualchunks) 显示设置，或者通过插件调用 [`this.emitFile`](guide/en/#thisemitfileemittedfile-emittedchunk--emittedasset--string) 设置。如果没有做任何设置，它将会根据 chunk 的内容来确定。
 * `[name]`: The name of the chunk. This can be explicitly set via the [`output.manualChunks`](guide/en/#outputmanualchunks) option or when the chunk is created by a plugin via [`this.emitFile`](guide/en/#thisemitfileemittedfile-emittedchunk--emittedasset--string). Otherwise it will be derived from the chunk contents.

正斜杆 `/` 可以用来划分文件到子目录。又见 [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames)。
Forward slashes `/` can be used to place files in sub-directories. See also [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.entryFileNames`](guide/en/#outputentryfilenames).

#### output.compact
类型: `boolean`<br>
命令行参数: `--compact`/`--no-compact`<br>
默认值: `false`

该选项用于压缩 Rollup 产生的代码。请注意，这个选项不会影响用户的代码。在构建已经压缩好的代码时，这个选择是很有用的。
This will minify the wrapper code generated by rollup. Note that this does not affect code written by the user. This option is useful when bundling pre-minified code.

#### output.entryFileNames
类型: `string`<br>
命令行参数: `--entryFileNames <pattern>`<br>
默认值: `"[name].js"`

该选项用于指定 chunks 的入口文件名。支持以下形式：
The pattern to use for chunks created from entry points. Pattern supports the following placeholders:
* `[format]`：输出（output）选项中定义的 `format` 的值，例如：`es` 或 `cjs`。
* `[format]`: The rendering format defined in the output options, e.g. `es` or `cjs`.
* `[hash]`：基于入口文件本身的内容，再加上所有它依赖的文件的内容，组成的哈希。
* `[hash]`: A hash based on the content of the entry point and the content of all its dependencies.
* `[name]`：入口文件的文件名（不包含扩展名），当入口文件（entry）定义为对象时，它的值时对象的键。
* `[name]`: The file name (without extension) of the entry point, unless the object form of input was used to define a different name.

正斜杆 `/` 可以用来划分文件到子目录。又见 [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.chunkFileNames`](guide/en/#outputchunkfilenames)。
Forward slashes `/` can be used to place files in sub-directories. See also [`output.assetFileNames`](guide/en/#outputassetfilenames), [`output.chunkFileNames`](guide/en/#outputchunkfilenames).

使用 [`output.preserveModules`](guide/en/#outputpreservemodules) 选项时，该模式也会生效。不过，以下有一组不同的占位符：
This pattern will also be used when using the [`output.preserveModules`](guide/en/#outputpreservemodules) option. Here there is a different set of placeholders available, though:
* `[format]`：输出（output）选项中定义的 `format` 的值。
* `[format]`: The rendering format defined in the output options.
* `[name]`：文件名（不包含扩展名）
* `[name]`: The file name (without extension) of the file.
* `[ext]`：文件扩展名。
* `[ext]`: The extension of the file.
* `[extname]`：文件扩展名，如果存在则以 `.` 开头。
* `[extname]`: The extension of the file, prefixed by `.` if it is not empty.

#### output.extend
类型: `boolean`<br>
命令行参数: `--extend`/`--no-extend`<br>
默认值: `false`

用于确定是否扩展 `umd` 或 `iife` 格式中 `name` 选项定义的全局变量。当值为 `true` 时，`name`选项指定的全局变量将定义为 `(global.name = global.name || {})`。当值为 `false` 时，`name`选项指定的全局变量将被覆盖为 `(global.name = {})`。
Whether or not to extend the global variable defined by the `name` option in `umd` or `iife` formats. When `true`, the global variable will be defined as `(global.name = global.name || {})`. When false, the global defined by `name` will be overwritten like `(global.name = {})`.

#### output.hoistTransitiveImports
类型: `boolean`<br>
命令行参数: `--hoistTransitiveImports`/`--no-hoistTransitiveImports`<br>
默认值: `true`

默认情况下，创建多个块 (chunk) 时，入口块的可传递导入 (transitive imports) 将会以空导入的形式被打包。详细信息和背景请查阅 ["为什么在拆分代码时我的输入块中会出现其他导入？"](guide/en/#why-do-additional-imports-turn-up-in-my-entry-chunks-when-code-splitting)。将值设置为 `false` 将禁用此行为。当使用 [`output.preserveModules`](guide/en/#outputpreservemodules) 选项时，该选项会被忽略，使得永远不会取消导入。
By default when creating multiple chunks, transitive imports of entry chunks will be added as empty imports to the entry chunks. See ["Why do additional imports turn up in my entry chunks when code-splitting?"](guide/en/#why-do-additional-imports-turn-up-in-my-entry-chunks-when-code-splitting) for details and background. Setting this option to `false` will disable this behaviour. This option is ignored when using the [`output.preserveModules`](guide/en/#outputpreservemodules) option as here, imports will never be hoisted.

#### output.inlineDynamicImports
类型: `boolean`<br>
命令行参数: `--inlineDynamicImports`/`--no-inlineDynamicImports`
默认值: `false`

该选项用于内联动态导入，而不是创建新的块来创建当个 bundle。并且它只在但输入源时产生作用。请注意，它会影响执行顺序：如果该模块是内联动态导入，那么它将会被立即执行。
This will inline dynamic imports instead of creating new chunks to create a single bundle. Only possible if a single input is provided. Note that this will change the execution order: A module that is only imported dynamically will be executed immediately if the dynamic import is inlined.

#### output.interop
类型: `boolean`<br>
命令行参数: `--interop`/`--no-interop`<br>
默认值: `true`

该选项用于决定是否去添加 "interop block"。默认情况下（`interop: true`），出于安全起见，如果要区分默认导出和命名导出，Rollup 会将所有外部依赖的默认导出（`default` exports）赋值给一个单独的变量。通常，这仅在外部依赖被转译（例如 Babel）时才适用 - 如果你不需要它，你可以将它的值置为 `interop: false`。
Whether or not to add an 'interop block'. By default (`interop: true`), for safety's sake, Rollup will assign any external dependencies' `default` exports to a separate variable if it is necessary to distinguish between default and named exports. This generally only applies if your external dependencies were transpiled (for example with Babel) – if you are sure you do not need it, you can save a few bytes with `interop: false`.

#### output.intro/output.outro
类型: `string | (() => string | Promise<string>)`<br>
命令行参数: `--intro`/`--outro <text>`

除了在特定格式中代码不同外，该选项功能和 [`output.banner/output.footer`](guide/en/#outputbanneroutputfooter) 类似。
Similar to [`output.banner/output.footer`](guide/en/#outputbanneroutputfooter), except that the code goes *inside* any format-specific wrapper.

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
类型: `{ [chunkAlias: string]: string[] } | ((id: string, {getModuleInfo, getModuleIds}) => string | void)`

该选项允许你创建自定义的共享公共模块。当值为对象形式时，每个属性代表一个大块，其中包含列出的模块及其所有依赖（如果他们是模块的一部分，除非他们已经在另个手动块中）。块的名称由对象属性的键（key）决定。
Allows the creation of custom shared common chunks. When using the object form, each property represents a chunk that contains the listed modules and all their dependencies if they are part of the module graph unless they are already in another manual chunk. The name of the chunk will be determined by the property key.

请注意，列出的模块本书不必成为模块图（module graph）的一部分，这对于使用 `@rollup/plugin-node-resolve` 并从中使用深度导入（deep imports）是非常有用的。例如：
Note that it is not necessary for the listed modules themselves to be be part of the module graph, which is useful if you are working with `@rollup/plugin-node-resolve` and use deep imports from packages. For instance

```
manualChunks: {
  lodash: ['lodash']
}
```

上述例子中，即使你只是使用 `import get from 'lodash/get'` 形式，Rollup 也会将 lodash 的所有模块放到一个手动块中。
will put all lodash modules into a manual chunk even if you are only using imports of the form `import get from 'lodash/get'`。

当使用函数形式时，每个被解析的模块都会传递给这个函数。如果函数返回字符串，那么该模块及其所有依赖将被添加到以返回字符串命名的手动块中。例如，以下例子会创建一个命名为 `vendor` 的块，它包含所有在 `node_modules` 中的依赖。
When using the function form, each resolved module id will be passed to the function. If a string is returned, the module and all its dependency will be added to the manual chunk with the given name. For instance this will create a `vendor` chunk containing all dependencies inside `node_modules`:

```javascript
manualChunks(id) {
  if (id.includes('node_modules')) {
    return 'vendor';
  }
}
```

请注意，如果在实际使用相应的模块之前触发了副作用，那么手动块可以更改整个应用程序的行为。
Be aware that manual chunks can change the behaviour of the application if side-effects are triggered before the corresponding modules are actually used.

当 `manualChunks` 使用函数形式时，它的第二个参数是 `getMouduleInfo` 函数和 `getMoudleIds` 函数，其工作方式与插件上下文中 [`this.getModuleInfo`](guide/en/#thisgetmoduleinfomoduleid-string--moduleinfo) 和 [`this.getModuleIds`](guide/en/#thisgetmoduleids--iterableiteratorstring) 相同。
When using the function form, `manualChunks` will be passed an object as second parameter containing the functions `getModuleInfo` and `getModuleIds` that work the same way as [`this.getModuleInfo`](guide/en/#thisgetmoduleinfomoduleid-string--moduleinfo) and [`this.getModuleIds`](guide/en/#thisgetmoduleids--iterableiteratorstring) on the plugin context.

这可以用于根据模块在模块图中的位置动态确定它应该被放在哪个手动块中。例如，考虑一个场景，其中有一组组件，每个组件动态导入一组已转译的依赖，即
This can be used to dynamically determine into which manual chunk a module should be placed depending on its position in the module graph. For instance consider a scenario where you have a set of components, each of which dynamically imports a set of translated strings, i.e.

```js
// 在 “foo” 组件中
// Inside the "foo" component

function getTranslatedStrings(currentLanguage) {
  switch (currentLanguage) {
    case 'en': return import('./foo.strings.en.js');
    case 'de': return import('./foo.strings.de.js');
    // ...
  }
}
```

如果将许多这样的组件一起使用，则会导致生成很多很小的动态导入块：尽管我们知道由同一块导入的所有相同语言的语言文件将始终一起使用，但是 Rollup 并不知道。
If a lot of such components are used together, this will result in a lot of dynamic imports of very small chunks: Even though we known that all language files of the same language that are imported by the same chunk will always be used together, Rollup does not have this information.

下面代码将会合并仅由单个入口使用的相同语言的所有文件：
The following code will merge all files of the same language that are only used by a single entry point:

```js
manualChunks(id, { getModuleInfo }) {
  const match = /.*\.strings\.(\w+)\.js/.exec(id);
  if (match) {
    const language = match[1]; // e.g. "en"
    const dependentEntryPoints = [];

    // we use a Set here so we handle each module at most once. This
    // prevents infinite loops in case of circular dependencies
    const idsToHandle = new Set(getModuleInfo(id).dynamicImporters);

    for (const moduleId of idsToHandle) {
      const { isEntry, dynamicImporters, importers } = getModuleInfo(moduleId);
      if (isEntry || dynamicImporters.length > 0) dependentEntryPoints.push(moduleId);

      // The Set iterator is intelligent enough to iterate over elements that
      // are added during iteration
      for (const importerId of importers) idsToHandle.add(importerId);
    }

    // If there is a unique entry, we put it into into a chunk based on the entry name
    if (dependentEntryPoints.length === 1) {
      return `${dependentEntryPoints[0].split('/').slice(-1)[0].split('.')[0]}.strings.${language}`;
    }
    // For multiple entries, we put it into a "shared" chunk
    if (dependentEntryPoints.length > 1) {
      return `shared.strings.${language}`;
    }
  }
}
```

#### output.minifyInternalExports
类型: `boolean`<br>
命令行参数: `--minifyInternalExports`/`--no-minifyInternalExports`<br>
默认值: 在 `es` 格式和 `system` 格式或者 `output.compact` 值为 `true`的情况下为 `true`，否则为 `false`

默认情况下，该选项的值在 `es` 格式和 `system` 格式或者 `output.compact` 值为 `true` 时为 `true`，意味着 Rollup 会尝试把内部变量导出为单个字母的变量，以便更好地压缩代码。
By default for formats `es` and `system` or if `output.compact` is `true`, Rollup will try to export internal variables as single letter variables to allow for better minification.

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

该选项值为 `true` 时，尽管表面上会导致代码输出变大，但是实际上，如果你使用压缩工具（minifier），代码会更小。在这种情况下，`export { importantValue as i }` 能够变成 `export{a as i}`，甚至是 `export{i}`，而实际上是 `export{ a as importantValue }`，因为压缩工具通常不会改变导出签名。
Even though it appears that setting this option to `true` makes the output larger, it actually makes it smaller if a minifier is used. In this case, `export { importantValue as i }` can become e.g. `export{a as i}` or even `export{i}`, while otherwise it would produce `export{ a as importantValue }` because a minifier usually will not change export signatures.

#### output.paths
类型: `{ [id: string]: string } | ((id: string) => string)`

该选项用于将外部依赖映射为路径。其中，外部依赖是指该选项 [无法解析](guide/en/#warning-treating-module-as-external-dependency) 的模块或者通过 [`external`](guide/en/#external) 选项明确指定的模块。`output.paths` 提供的路径会被用在生成的 bundle中，而不是模块中，例如，你可以从 CDN 加载依赖：
Maps external module IDs to paths. External ids are ids that [cannot be resolved](guide/en/#warning-treating-module-as-external-dependency) or ids explicitly provided by the [`external`](guide/en/#external) option. Paths supplied by `output.paths` will be used in the generated bundle instead of the module ID, allowing you to, for example, load dependencies from a CDN:

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
类型: `boolean`<br>
命令行参数: `--preserveModules`/`--no-preserveModules`<br>
默认值: `false`

该选项将使用原始模块名作为文件名，为所有模块创建单独的块，而不是创建尽可能少的块。它需要配合 [`output.dir`](guide/en/#outputdir) 选项使用。Tree-shaking 仍会对没有被入口点使用或者执行阶段没有副作用的文件生效。该选项可以用于将文件结构转换为其他模块格式。
Instead of creating as few chunks as possible, this mode will create separate chunks for all modules using the original module names as file names. Requires the [`output.dir`](guide/en/#outputdir) option. Tree-shaking will still be applied, suppressing files that are not used by the provided entry points or do not have side-effects when executed. This mode can be used to transform a file structure to a different module format.

请注意，在转换为 `cjs` 或 `amd` 格式时，默认情况下，通过设置 [`output.exports`](guide/en/#outputexports) 为 `auto`，每个文件会作为入口点。这意味着，例如对于 `cjs`，只包含默认导出的文件会呈现为
Note that when transforming to `cjs` or `amd` format, each file will by default be treated as an entry point with [`output.exports`](guide/en/#outputexports) set to `auto`. This means that e.g. for `cjs`, a file that only contains a default export will be rendered as

```js
// input main.js
export default 42;

// output main.js
'use strict';

var main = 42;

module.exports = main;
```

将值直接赋值给 `module.exports`。如果其他模块导入此文件，它们可以通过以下方式访问默认导出
assigning the value directly to `module.exports`. If someone imports this file, they will get access to the default export via

```js
const main = require('./main.js');
console.log(main); // 42
```

与常规入口点一样，混合使用默认导出和命名导出的模块将会产生警告。你可以通过设置 `output.exports: "named"`，强制所有文件使用命名导出模式，来避免出现警告。在这种情况下，可以通过导出的 `.default` 属性访问默认导出：
As with regular entry points, files that mix default and named exports will produce warnings. You can avoid the warnings by forcing all files to use named export mode via `output.exports: "named"`. In that case, the default export needs to be accessed via the `.default` property of the export:

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
类型: `boolean | 'inline' | 'hidden'`<br>
命令行参数: `-m`/`--sourcemap`/`--no-sourcemap`<br>
默认值: `false`

如果该选项值为 `true`，那么每个文件都将生成一个独立的源码映射（sourcemap）文件。如果值为 `“inline”`，那么源码映射会以 data URI 的形式添加到输出文件末尾。如果值为 `“hidden”`，表现和 `true` 相同，不同的是，bundle 文件中没有源码映射的注释。
If `true`, a separate sourcemap file will be created. If `"inline"`, the sourcemap will be appended to the resulting `output` file as a data URI. `"hidden"` works like `true` except that the corresponding sourcemap comments in the bundled files are suppressed.

#### output.sourcemapExcludeSources
类型: `boolean`<br>
命令行参数: `--sourcemapExcludeSources`/`--no-sourcemapExcludeSources`<br>
默认值: `false`

如果该选项的值为 `true`，那么实际源代码将不会被添加到源码映射中，从而使其变得更小。
If `true`, the actual code of the sources will not be added to the sourcemaps making them considerably smaller.

#### output.sourcemapFile
类型: `string`<br>
命令行参数: `--sourcemapFile <file-name-with-path>`

该选项用于指定生成源码映射文件的位置。如果是一个绝对路径，那么源码映射文件中的所有 `源文件（sources）` 路径都相对于它。`map.file` 属性是 `sourcemapFile` 的基本名称，因为源码隐射的位置是被假定与该包相邻。
The location of the generated bundle. If this is an absolute path, all the `sources` paths in the sourcemap will be relative to it. The `map.file` property is the basename of `sourcemapFile`, as the location of the sourcemap is assumed to be adjacent to the bundle.

如果 `output` 设置了，那就不必设置 `sourcemapFile`，这种情况下，`sourcemapFile` 的值会通过输出文件名中添加 “.map” 推断出来。
`sourcemapFile` is not required if `output` is specified, in which case an output filename will be inferred by adding ".map"  to the output filename for the bundle.

#### output.sourcemapPathTransform
类型: `(relativeSourcePath: string, sourcemapPath: string) => string`

该选项用于源码映射中的路径转换。其中，`relativeSourcePath` 是指从生成的 `.map` 文件到相对应的源文件的相对路径，而 `sourcemapPath` 是指生成源码映射文件的绝对路径。
A transformation to apply to each path in a sourcemap. `relativeSourcePath` is a relative path from the generated `.map` file to the corresponding source file while `sourcemapPath` is the fully resolved path of the generated sourcemap file.

```js
import path from 'path';
export default ({
  input: 'src/main',
  output: [{
    file: 'bundle.js',
    sourcemapPathTransform: (relativeSourcePath, sourcemapPath) => {
      // 将会把相对路径替换为绝对路径
      // will replace relative paths with absolute paths
      return path.resolve(path.dirname(sourcemapPath), relativeSourcePath)
    },
    format: 'es',
    sourcemap: true
  }]
});
```

#### preserveEntrySignatures
类型: `"strict" | "allow-extension" | false`<br>
命令行参数: `--preserveEntrySignatures <strict|allow-extension>`/`--no-preserveEntrySignatures`<br>
默认值: `"strict"`

控制 Rollup 是否尝试确保入口块与基础入口模块具有相同的导出。
Controls if Rollup tries to ensure that entry chunks have the same exports as the underlying entry module.

- 如果设置为 `"strict"`，Rollup 将在入口 chunk 中创建与相应入口模块中完全相同的导出。如果因为需要向 chunk 中添加额外的内部导出而无法这样做，那么 Rollup 将创建一个 `facade` 入口 chunk，它将仅从前其他 chunk 中导出必要的绑定，但不包含任何其他代码。对于库，推荐使用此设置。
- If set to `"strict"`, Rollup will create exactly the same exports in the entry chunk as there are in the corresponding entry module. If this is not possible because additional internal exports need to be added to a chunk, Rollup will instead create a "facade" entry chunk that reexports just the necessary bindings from other chunks but contains no code otherwise. This is the recommended setting for libraries.
- `"allow-extension"` 将在入口 chunk 中创建入口模块的所有导出，但是如果有必要，还可以添加其他导出，从而避免出现 “facade” 入口 chunk。对于不需要严格签名的库，此设置很有意义。
- `"allow-extension"` will create all exports of the entry module in the entry chunk but may also add additional exports if necessary, avoiding a "facade" entry chunk. This setting makes sense for libraries where a strict signature is not required.
- `false` 不会将入口模块中的任何导出内容添加到相应的 chunk 中，甚至不包含相应的代码，除非这些导出内容在 bundle 的其他位置使用。但是，可以将内部导出添加到入口 chunks 中。对于将入口 chunks 放置在脚本标记中的 Web apps，推荐使用该设置，因为它可能同时减少 bundle 的尺寸大小 和 chunks 的数量。
- `false` will not add any exports of an entry module to the corresponding chunk and does not even include the corresponding code unless those exports are used elsewhere in the bundle. Internal exports may be added to entry chunks, though. This is the recommended setting for web apps where the entry chunks are to be placed in script tags as it may reduce both the number of chunks and possibly the bundle size.

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
At the moment, the only way to override this setting for individual entry chunks is to use the plugin API and emit those chunks via [`this.emitFile`](guide/en/#thisemitfileemittedfile-emittedchunk--emittedasset--string) instead of using the [`input`](guide/en/#input) option.

#### strictDeprecations
类型: `boolean`<br>
命令行参数: `--strictDeprecations`/`--no-strictDeprecations`<br>
默认值: `false`

启用此标志后，当使用废弃的功能时，Rollup 将抛出错误而不是显示警告。此外，如果使用了在下一个主要版本（major version）被标记为废弃警告的功能时，也会抛出错误。
When this flag is enabled, Rollup will throw an error instead of showing a warning when a deprecated feature is used. Furthermore, features that are marked to receive a deprecation warning with the next major version will also throw an error when used.

例如，插件作者打算使用此标志，以便能够尽快为即将发布的主要版本调整其插件。
This flag is intended to be used by e.g. plugin authors to be able to adjust their plugins for upcoming major releases as early as possible.

### 慎用选项(Danger zone)

除非你知道自己在做什么，否则可能不需要使用这些选项！
You probably don't need to use these options unless you know what you are doing!

#### acorn
类型: `AcornOptions`

应该传递给 Acorn `parse` 函数的选项，比如 `allowReserved: true`。查看 [Acorn 文档](https://github.com/acornjs/acorn/tree/master/acorn#interface) 了解更多可用选项。
Any options that should be passed through to Acorn's `parse` function, such as `allowReserved: true`. Cf. the [Acorn documentation](https://github.com/acornjs/acorn/tree/master/acorn#interface) for more available options.

#### acornInjectPlugins
类型: `AcornPluginFunction | AcornPluginFunction[]`

注入到 Acorn 中的单个插件或者插件数组。例如要支持 JSX 预发，你可以指定
A single plugin or an array of plugins to be injected into Acorn. For instance to use JSX syntax, you can specify

```javascript
import jsx from 'acorn-jsx';

export default {
    // … other options …
    acornInjectPlugins: [
        jsx()
    ]
};
```

在你的 Rollup 配置。请注意，这与使用 Babel 不同，因为生成的输出仍将包含 JSX，而 Babel 会将其替换为有效的 JavaScript。
in your rollup configuration. Note that this is different from using Babel in that the generated output will still contain JSX while Babel will replace it with valid JavaScript.

#### context
类型: `string`<br>
命令行参数: `--context <contextVariable>`<br>
默认值: `undefined`

默认情况下，模块的上下文（即，顶级中的`this`）为 `undefined`。在极其少数情况下，你可能需要将其更改为其他名称，比如 `'window'`。
By default, the context of a module – i.e., the value of `this` at the top level – is `undefined`. In rare cases you might need to change this to something else, like `'window'`.

#### moduleContext
类型: `((id: string) => string) | { [id: string]: string }`<br>

与 [`context`](guide/en/#context) 相同，但每个模块要么是由 `id: context` 构成的对象，要么是 `id => context` 函数。
Same as [`context`](guide/en/#context), but per-module – can either be an object of `id: context` pairs, or an `id => context` function.

#### output.amd
类型: `{ id?: string, define?: string}`

可以包含以下属性的对象：
An object that can contain the following properties:

**output.amd.id**<br>
类型: `string`<br>
命令行参数: `--amd.id <amdId>`

用于 AMD/UMD bundles 的 ID：
An ID to use for AMD/UMD bundles:

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
类型: `string`<br>
命令行参数: `--amd.define <defineFunctionName>`

要使用的函数名称，代替 AMD 中的 `define`：
A function name to use instead of `define`:

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
类型: `boolean`<br>
命令行参数: `--esModule`/`--no-esModule`<br>
默认值: `true`

决定是否在生成非 ES（non-ES）格式导出时添加 `__esModule: true` 属性。
Whether or not to add a `__esModule: true` property when generating exports for non-ES formats.

#### output.exports
类型: `string`<br>
命令行参数: `--exports <exportMode>`<br>
默认值: `'auto'`

使用哪种导出模式。默认是 `auto`，指根据 `input` 模块导出猜测你的意图：
What export mode to use. Defaults to `auto`, which guesses your intentions based on what the `input` module exports:

* `default` - 适用于只使用 `export default ...` 的情况
* `default` – suitable if you are only exporting one thing using `export default ...`
* `named` - 适用于导出超过一个模块的情况
* `named` – suitable if you are exporting more than one thing
* `none` - 适用于没有导出的情况（比如，你这时是在构建应用而非库）
* `none` – suitable if you are not exporting anything (e.g. you are building an app, not a library)

`default` 和 `named` 之间的差异会影响其他人如何使用你的包。例如，如果你使用 `default`，那么 Commonjs 用户可以执行以下操作：
The difference between `default` and `named` affects how other people can consume your bundle. If you use `default`, a CommonJS user could do this, for example:

```js
const yourLib = require( 'your-lib' );
```

使用 `named`，用户可以执行以下操作：
With `named`, a user would do this instead:

```js
const yourMethod = require( 'your-lib' ).yourMethod;
```

问题是，如果你使用 `named` 导出但还具有 `default` 导出，则用户必须执行以下操作才能使用默认导出：
The wrinkle is that if you use `named` exports but *also* have a `default` export, a user would have to do something like this to use the default export:

```js
const yourMethod = require( 'your-lib' ).yourMethod;
const yourLib = require( 'your-lib' ).default;
```

#### output.externalLiveBindings
类型: `boolean`<br>
命令行参数: `--externalLiveBindings`/`--no-externalLiveBindings`<br>
默认值: `true`

当设置为 `false` 时，Rollup 不会为外部依赖生成支持动态绑定的代码，而是假设外部依赖永远不会改变。这使得 Rollup 会生成更多优化代码。请注意，当外部依赖存在循环引用时，这可能会引起问题。
When set to `false`, Rollup will not generate code to support live bindings for external imports but instead assume that exports do not change over time. This will enable Rollup to generate more optimized code. Note that this can cause issues when there are circular dependencies involving an external dependency.

在大多数情况下，这将避免 Rollup 生成多余代码的 getters，因此可以在许多情况下，是代码兼容 IE8。
This will avoid most cases where Rollup generates getters in the code and can therefore be used to make code IE8 compatible in many cases.

例：
Example:

```js
// 输入
// input
export {x} from 'external';

// 使用 externalLiveBindings: true 选项时 CJS 格式的输出
// CJS output with externalLiveBindings: true
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
// CJS output with externalLiveBindings: false
'use strict';

Object.defineProperty(exports, '__esModule', { value: true });

var external = require('external');

exports.x = external.x;
```

#### output.freeze
类型: `boolean`<br>
命令行参数: `--freeze`/`--no-freeze`<br>
默认值: `true`

是否使用 `Object.freeze()` 冻结动态访问的导入对象的命名空间，例如 `import * as namespaceImportObject from...`。
Whether to `Object.freeze()` namespace import objects (i.e. `import * as namespaceImportObject from...`) that are accessed dynamically.

#### output.indent
类型: `boolean | string`<br>
命令行参数: `--indent`/`--no-indent`<br>
默认值: `true`

缩进字符串，用于需要代码缩进的格式（`amd`，`iife`，`umd`，`system`）。可以是 `false`（没有缩进）或 `true`（默认 - 自动缩进）。
The indent string to use, for formats that require code to be indented (`amd`, `iife`, `umd`, `system`). Can also be `false` (no indent), or `true` (the default – auto-indent)

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
类型: `boolean`<br>
命令行参数: `--namespaceToStringTag`/`--no-namespaceToStringTag`<br>
默认值: `false`

是否将符合规范的 `.toString` 标签加到命名空间对象。如果该选项设置为 `true`，那么下列代码会输出为 `[object Module]`。
Whether to add spec compliant `.toString()` tags to namespace objects. If this option is set,

```javascript
import * as namespace from './file.js';
console.log(String(namespace));
```

will always log `[object Module]`;

#### output.noConflict
类型: `boolean`<br>
命令参数: `--noConflict`/`--no-noConflict`<br>
默认值: `false`

这将生成额外的 `noConflict` 导出到 UMD 包。在 IIFE 场景中调用时，此方法将返回包的导出，同时将相应的全部变量恢复为先前的值。
This will generate an additional `noConflict` export to UMD bundles. When called in an IIFE scenario, this method will return the bundle exports while restoring the corresponding global variable to its previous value.

#### output.preferConst
类型: `boolean`<br>
命令行参数: `--preferConst`/`--no-preferConst`<br>
默认值: `false`

为导出生成 `const` 声明，而不是 `var` 声明。
Generate `const` declarations for exports rather than `var` declarations.

#### output.strict
类型: `boolean`<br>
命令行参数: `--strict`/`--no-strict`<br>
默认值: `true`

是否在生成非 ES 包的顶部包含 “use strict” 用法。严格地讲，ES 模块总是处于严格模式，所以不要无缘无故禁用该选项。
Whether to include the 'use strict' pragma at the top of generated non-ES bundles. Strictly speaking, ES modules are *always* in strict mode, so you shouldn't disable this without good reason.

#### output.systemNullSetters
类型: `boolean`<br>
命令行参数: `--systemNullSetters`/`--no-systemNullSetters`<br>
默认值: `false`

在导出 `system` 模块格式时，该选项将代替空的 setter 函数，以 `null` 形式简化输出。该选项仅在 *SystemJS 6.3.3 及以上版本中支持*。
When outputting the `system` module format, this will replace empty setter functions with `null` as an output simplification. This is *only supported in SystemJS 6.3.3 and above*.

#### preserveSymlinks
类型: `boolean`<br>
命令行参数: `--preserveSymlinks`<br>
默认值: `false`

当设置为 `false` s时，符号链接将跟随在解析文件后面。当设置为 `true` 时，符号链接不是跟随在后面，而是被视为文件所在的链接。为了说明白，考虑以下情况：
When set to `false`, symbolic links are followed when resolving a file. When set to `true`, instead of being followed, symbolic links are treated as if the file is where the link is. To illustrate, consider the following situation:

```js
// /main.js
import {x} from './linked.js';
console.log(x);

// /linked.js
// 这是 /nested/file.js 的符号链接
// this is a symbolic link to /nested/file.js

// /nested/file.js
export {x} from './dep.js';

// /dep.js
export const x = 'next to linked';

// /nested/dep.js
export const x = 'next to original';
```

如果 `preserveSymlinks` 值为 `false`，那么从 `/main.js` 中创建包将会记为 “next to original”，因为它将使用符号链接文件的位置来解决其依赖。然而，如果 `preserveSymlinks` 值为 `true`，那么它将会标记为 “next to linked”，因为符号链接将无法解析。
If `preserveSymlinks` is `false`, then the bundle created from `/main.js` will log "next to original" as it will use the location of the symbolically linked file to resolve its dependencies. If `preserveSymlinks` is `true`, however, it will log "next to linked" as the symbolic link will not be resolved.

#### shimMissingExports
类型: `boolean`<br>
命令行参数: `--shimMissingExports`/`--no-shimMissingExports`<br>
默认值: `false`

如果提供该选项，那么从未定义绑定的文件中导入绑定时，打包不会失败。相反，将为这些绑定创建值为 `undefined` 的新变量。
If this option is provided, bundling will not fail if bindings are imported from a file that does not define these bindings. Instead, new variables will be created for these bindings with the value `undefined`.

#### treeshake
类型: `boolean | { annotations?: boolean, moduleSideEffects?: ModuleSideEffectsOption, propertyReadSideEffects?: boolean, tryCatchDeoptimization?: boolean, unknownGlobalSideEffects?: boolean }`<br>
命令行参数: `--treeshake`/`--no-treeshake`<br>
默认值: `true`

是否应用 tree-shake 并微调 tree-shake 过程。该选项设置为 `false` 时，Rollup 将生成更大的包，但是可能会提高构建性能。如果你发现由 tree-shake 造成的 bug，请给我们提 issue ！
Whether or not to apply tree-shaking and to fine-tune the tree-shaking process. Setting this option to `false` will produce bigger bundles but may improve build performance. If you discover a bug caused by the tree-shaking algorithm, please file an issue!
将此选项设置为对象意味着启用 tree-shaking，并启用以下选项：
Setting this option to an object implies tree-shaking is enabled and grants the following additional options:

**treeshake.annotations**<br>
类型: `boolean`<br>
命令行参数: `--treeshake.annotations`/`--no-treeshake.annotations`<br>
默认值: `true`

如果该选项值为 `false`，那么在确定函数调用和构造函数调用的副作用时，将会忽略纯注释的提示，比如，包含 `@__PURE__` 或 `#__PURE__` 的注释。这些注释需要紧接在调用之前才能生效。除非将此选项设置为 `false`，否则以下代码将被完全删除，在这种情况下，它将保持不变。
If `false`, ignore hints from pure annotations, i.e. comments containing `@__PURE__` or `#__PURE__`, when determining side-effects of function calls and constructor invocations. These annotations need to immediately precede the call invocation to take effect. The following code will be completely removed unless this option is set to `false`, in which case it will remain unchanged.

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
类型: `boolean | "no-external" | string[] | (id: string, external: boolean) => boolean`<br>
命令行参数: `--treeshake.moduleSideEffects`/`--no-treeshake.moduleSideEffects`/`--treeshake.moduleSideEffects no-external`<br>
默认值: `true`

如果值为 `false`，则假定，像改变全局变量或不执行检查就记录等行为一样，没有导入任何内容的模块和外部依赖没有其他副作用。对于外部依赖，这将限制空导入：
If `false`, assume modules and external dependencies from which nothing is imported do not have other side-effects like mutating global variables or logging without checking. For external dependencies, this will suppress empty imports:

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

对于非外部依赖模块，该选项值为 `false` 时，除非至少从该模块导入一次，否则将不会包含来自该模块的任何语句：
For non-external modules, `false` will not include any statements from a module unless at least one import from this module is included:

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

你也可以具有副作用的模块列表，或者提供一个函数分别确定每个模块。值为 `“no-external”` 将意味着如果可能，则仅删除外部依赖，并且它等同于函数 `(id, external) => !external`；
You can also supply a list of modules with side-effects or a function to determine it for each module individually. The value `"no-external"` will only remove external imports if possible and is equivalent to the function `(id, external) => !external`;

**treeshake.propertyReadSideEffects**
类型: `boolean`<br>
命令行参数: `--treeshake.propertyReadSideEffects`/`--no-treeshake.propertyReadSideEffects`<br>
默认值: `true`

如果该选项值为 `false`，则假设 读取对象的属性将永远不会有副作用。根据你的代码，禁用该属性能够显著缩小包的大小，但是如果你依赖了 geters 或非法属性访问的造成的错误，那么可能会破坏该功能。
If `false`, assume reading a property of an object never has side-effects. Depending on your code, disabling this option can significantly reduce bundle size but can potentially break functionality if you rely on getters or errors from illegal property access.

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
类型: `boolean`<br>
命令行参数: `--treeshake.tryCatchDeoptimization`/`--no-treeshake.tryCatchDeoptimization`<br>
默认值: `true`

默认情况下，Rollup 假设在进行 tree-shaking 是，许多运行时内置的全局变量的行为均符合最新规范，并且不会引发意外错误。为了支持像依赖于抛出这些错误的特征检测工作流，Rollup 将默认禁用 try 语句中的 tree-shaking。如果从 try 语句中调用函数参数，那么该参数也不会被优化掉。如果你不需要此功能或不想要在 try 语句中使用 tree-shaking，则可以设置 `treeshake.tryCatchDeoptimization` 为 `false`。
By default, Rollup assumes that many builtin globals of the runtime behave according to the latest specs when tree-shaking and do not throw unexpected errors. In order to support e.g. feature detection workflows that rely on those errors being thrown, Rollup will by default deactivate tree-shaking inside try-statements. If a function parameter is called from within a try-statement, this parameter will be deoptimized as well. Set `treeshake.tryCatchDeoptimization` to `false` if you do not need this feature and want to have tree-shaking inside try-statements.

```js
function otherFn() {
  // even though this function is called from a try-statement, the next line
  // will be removed as side-effect-free
  Object.create(null);
}

function test(callback) {
  try {
  	// calls to otherwise side-effect-free global functions are retained
  	// inside try-statements for tryCatchDeoptimization: true
    Object.create(null);

  	// calls to other function are retained as well but the body of this
  	// function may again be subject to tree-shaking
    otherFn();

    // if a parameter is called, then all arguments passed to that function
    // parameter will be deoptimized
    callback();
  } catch {}
}

test(() => {
  // will be ratained
  Object.create(null)
});

// call will be retained but again, otherFn is not deoptimized
test(otherFn);

```

**treeshake.unknownGlobalSideEffects**
类型: `boolean`<br>
命令行参数: `--treeshake.unknownGlobalSideEffects`/`--no-treeshake.unknownGlobalSideEffects`<br>
默认值: `true`

因为访问不存在的全局变量会引起错误，所以默认情况下 Rollup 保留对非内置全局变量的所有访问。该选项的值设置为 `false` 可以避免该检查。对于大多数代码库而言，这可能是安全的。
Since accessing a non-existing global variable will throw an error, Rollup does by default retain any accesses to non-builtin global variables. Set this option to `false` to avoid this check. This is probably safe for most code-bases.

```js
// input
const jQuery = $;
const requestTimeout = setTimeout;
const element = angular.element;

// output with unknownGlobalSideEffects == true
const jQuery = $;
const element = angular.element;

// output with unknownGlobalSideEffects == false
const element = angular.element;
```

在这个例子中，最后一行将被始终保留，用于访问 `element` 属性，但如果 `angular` 值为 `null`，也可能抛出错误。为了避免这种情况的检查，可以设置 `treeshake.propertyReadSideEffects` 为 `false`。
In the example, the last line is always retained as accessing the `element` property could also throw an error if `angular` is e.g. `null`. To avoid this check, set `treeshake.propertyReadSideEffects` to `false` as well.

### 实验选项(Experimental options)

这些选项反应了尚未完全确定的新功能。因此，可行性，行为和用法在次要版本中可能发生变化。
These options reflect new features that have not yet been fully finalized. Availability, behaviour and usage may therefore be subject to change between minor versions.

#### experimentalCacheExpiry
类型: `number`<br>
命令行参数: `--experimentalCacheExpiry <numberOfRuns>`<br>
默认值: `10`

用于确定在多少次执行以后，应该删除不再被插件使用的静态缓存。
Determines after how many runs cached assets that are no longer used by plugins should be removed.

#### perf
类型: `boolean`<br>
命令行参数: `--perf`/`--no-perf`<br>
默认值: `false`

决定是否收集打包执行耗时。当使用命令行或者配置为监事，将会展示与当前构建进程有关的详细指标。当在 [JavaScript API](guide/en/#javascript-api) 中使用试试，返回的 bundle 对象将包含额外的 `getTimings()` 函数，可以随时调用该函数来查找所有累计的指标。
Whether to collect performance timings. When used from the command line or a configuration file, detailed measurements about the current bundling process will be displayed. When used from the [JavaScript API](guide/en/#javascript-api), the returned bundle object will contain an additional `getTimings()` function that can be called at any time to retrieve all accumulated measurements.

`getTimings()` 返回以下对象形式：
`getTimings()` returns an object of the following form:

```json
{
  "# BUILD": [ 698.020877, 33979632, 45328080 ],
  "## parse modules": [ 537.509342, 16295024, 27660296 ],
  "load modules": [ 33.253778999999994, 2277104, 38204152 ],
  ...
}
```

对于每个值，第一个数值表示经过的时间，第二个数值表示内存消耗的变化，第三个数值表示此步骤完成后的总内存消耗。这些步骤的顺序是使用 `Object.keys` 确定的。Top level 键以 `#` 开头，并包含嵌套步骤的耗时，例如，在上面例子值班费的 `# BUILD` 步骤耗时（698ms）包含了 `## parse modules` 步骤的耗时（539ms）。
For each key, the first number represents the elapsed time while the second represents the change in memory consumption and the third represents the total memory consumption after this step. The order of these steps is the order used by `Object.keys`. Top level keys start with `#` and contain the timings of nested steps, i.e. in the example above, the 698ms of the `# BUILD` step include the 538ms of the `## parse modules` step.

### 观察选项(Watch options)

类型: `{ buildDelay?: number, chokidar?: ChokidarOptions, clearScreen?: boolean, exclude?: string, include?: string, skipWrite?: boolean } | false`<br>
默认值: `{}`<br>

指定观察模式的选项，或防止 Rollup 配置被观察。当使用配置数组时，指定为 `false` 是非常有用的。在这种情况下，将不会根据观察模式中的变更构建或重建 Rollup 配置，但是当定期运行 Rollup 时，它会被构建。
Specify options for watch mode or prevent this configuration from being watched. Specifying `false` is only really useful when an array of configurations is used. In that case, this configuration will not be built or rebuilt on change in watch mode, but it will be built when running Rollup regularly:

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
These options only take effect when running Rollup with the `--watch` flag, or using `rollup.watch`.

#### watch.buildDelay
类型: `number`<br>
命令行参数: `--watch.buildDelay <number>`<br>
默认值: `0`

配置 Rollup 将等待进一步构建直到触发重建的时间（以毫秒为单位）。默认情况下，Rollup 不会等待，但是在 chokidar 实例中配置了一个小的抖动时间（debounce timeout）。此值大于 `0` 将意味着 如果配置的毫秒数没有发生变化，Rollup 只会触发一次重建。如果观察到多个配置变化，Rollup 将使用配置的最大构建延迟。
Configures how long Rollup will wait for further changes until it triggers a rebuild in milliseconds. By default, Rollup does not wait but there is a small debounce timeout configured in the chokidar instance. Setting this to a value greater than `0` will mean that Rollup will only triger a rebuild if there was no change for the configured number of milliseconds. If several configurations are watched, Rollup will use the largest configured build delay.

#### watch.chokidar
类型: `ChokidarOptions`<br>

在观察选项中，该选项是可选对象，将传递给 [chokidar](https://github.com/paulmillr/chokidar) 实例。查阅 [chokidar documentation](https://github.com/paulmillr/chokidar#api) 文档以了解可用的选项。
An optional object of watch options that will be passed to the bundled [chokidar](https://github.com/paulmillr/chokidar) instance. See the [chokidar documentation](https://github.com/paulmillr/chokidar#api) to find out what options are available.

#### watch.clearScreen
类型: `boolean`<br>
命令行参数: `--watch.clearScreen`/`--no-watch.clearScreen`<br>
默认值: `true`

在触发重建是是否清除屏幕。
Whether to clear the screen when a rebuild is triggered.

#### watch.exclude
类型: `string`<br>
命令行参数: `--watch.exclude <files>`

防止文件被观察：
Prevent files from being watched:

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
类型: `string`<br>
命令行参数: `--watch.include <files>`

限制只能对指定文件进行观察。请注意，该选项只过滤模块图（module graph），但不允许添加其他观察文件：
Limit the file-watching to certain files. Note that this only filters the module graph but does not allow to add additional watch files:

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
类型: `boolean`<br>
命令行参数: `--watch.skipWrite`/`--no-watch.skipWrite`<br>
默认值: `false`

是否在触发重新构建时忽略 `bundle.write()` 步骤。
Whether to skip the `bundle.write()` step when a rebuild is triggered.

### 废弃选项(Deprecated options)

☢️ These options have been deprecated and may be removed in a future Rollup version.

#### inlineDynamicImports
_Use the [`output.inlineDynamicImports`](guide/en/#outputinlinedynamicimports) output option instead, which has the same signature._

#### manualChunks
_Use the [`output.manualChunks`](guide/en/#outputmanualchunks) output option instead, which has the same signature._

#### preserveModules
_Use the [`output.preserveModules`](guide/en/#outputpreservemodules) output option instead, which has the same signature._

#### output.dynamicImportFunction
_Use the [`renderDynamicImport`](guide/en/#renderdynamicimport) plugin hook instead._<br>
Type: `string`<br>
CLI: `--dynamicImportFunction <name>`<br>
Default: `import`

This will rename the dynamic import function to the chosen name when outputting ES bundles. This is useful for generating code that uses a dynamic import polyfill such as [this one](https://github.com/uupaa/dynamic-import-polyfill).

#### treeshake.pureExternalModules
_Use [`treeshake.moduleSideEffects: 'no-external'`](guide/en/#treeshake) instead._<br>
Type: `boolean | string[] | (id: string) => boolean | null`<br>
CLI: `--treeshake.pureExternalModules`/`--no-treeshake.pureExternalModules`<br>
Default: `false`

If `true`, assume external dependencies from which nothing is imported do not have other side-effects like mutating global variables or logging.

```javascript
// input file
import {unused} from 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// output with treeshake.pureExternalModules === false
import 'external-a';
import 'external-b';
console.log(42);
```

```javascript
// output with treeshake.pureExternalModules === true
console.log(42);
```

You can also supply a list of external ids to be considered pure or a function that is called whenever an external import could be removed.
