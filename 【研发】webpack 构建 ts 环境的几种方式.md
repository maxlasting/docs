# webpack 构建 ts 环境的几种方式

![](https://cdn.maxlasting.com/doc-assets/202208252231079.jpg)

@ 最近在捣鼓项目的研发环境升级，发现 webpack5 的生态发展的方向是变得越来越简单了，在我看来已经很接近开箱即用了。这里主要记录下配置 ts 的一些方式，因为本人很反感使用 ts，但是对于团队来说 ts 又是不可或缺的，所以把升级的过程和查阅的资料整理一下。

## ts-loader

这是用于`webpack`的`typescript`加载器，将`typescript`编译成js，`ts-loader`在内部是调用了`typescript`的官方编译器`-- tsc`。所以，`ts-loader`和`tsc`是共享`tsconfig.json`的。

**安装**

```
npm i -D ts-loader  typescript
```

正常使用`webpack`即可。 

有两种类型的 `Options：typescript options`（也称为 “编译器 options” ）和 `loader options。typescript options` 应该通过 `tsconfig.json` 文件设置。`loader options`可以通过`webpack`配置中的 options 属性指定：

```js
module.exports = {
  ...
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: [
          {
            loader: 'ts-loader',
            options: {
              transpileOnly: true // 只做语言转换，而不做类型检查
            }
          }
        ]
      }
    ]
  }
};
```

`transpileOnly`快速构建一个项目。

- `transpileOnly: false`语言转换 + 类型检查 = 3290 ms

```shell
> webpack --mode production --config ./build/webpack.config.js

clean-webpack-plugin: options.output.path not defined. Plugin disabled...
asset index.html 327 bytes [emitted]
asset app.js 89 bytes [emitted] [minimized] (name: main)
./src/index.ts 102 bytes [built] [code generated]
webpack 5.27.2 compiled successfully in 3290 ms
```


- `transpileOnly: true` only 语言转换 = 598 ms

```shell
> webpack --mode production --config ./build/webpack.config.js

clean-webpack-plugin: options.output.path not defined. Plugin disabled...
asset index.html 327 bytes [compared for emit]
asset app.js 89 bytes [compared for emit] [minimized] (name: main)
./src/index.ts 102 bytes [built] [code generated]
webpack 5.27.2 compiled successfully in 598 ms
```


- `transpileOnly: true` + `fork-ts-checker-webpack-plugin`插件配合，可以补全类型检查的功能。

```
npm i fork-ts-checker-webpack-plugin -D
```

```js
const ForkTsCheckerWebpackPlugin = require("fork-ts-checker-webpack-plugin");

module.exports = {
  ...
  plugins:[
    ...
    new ForkTsCheckerWebpackPlugin()
  ]
};
```

这时，你的项目中有类型错误，编译就会报错。编译通过耗时：2289 ms。

```shell
> webpack --mode production --config ./build/webpack.config.js

clean-webpack-plugin: options.output.path not defined. Plugin disabled...
asset index.html 327 bytes [compared for emit]
asset app.js 89 bytes [emitted] [minimized] (name: main)
./src/index.ts 102 bytes [built] [code generated]
webpack 5.27.2 compiled successfully in 2289 ms
```

## awesome-typescript-loader

`awesome-typescript-loader`的创建主要是为了加快项目中的编译速度。与`ts-loader`的主要区别：

- 更适合与 Babel 集成，使用 Babel 的转义和缓存。
- 不需要安装独立的插件，就可以把类型检查放在独立进程中。

**安装**：

```
npm i -D awesome-typescript-loader
```

```js
module.exports = {
  ...
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: [
          {
            loader: 'awesome-typesscript-loader',
            options: {
              transpileOnly: false
            }
          }
        ]
      }
    ]
  }
};
```

跑一下：

- transpileOnly: false

```
webpack 5.27.2 compiled successfully in 3392 ms
```

- transpileOnly: true

```
webpack 5.27.2 compiled successfully in 2411 ms
```

- transpileOnly: true + 自带插件 CheckerPlugin

```
webpack 5.27.2 compiled successfully in 2529 ms
```

## 编译时间对比

| loader | 默认配置 | transpileOnly | transpileObly + 类型检查进程 |
| -- | -- | -- | -- |
| ts-loader | 3200+ | 600 | 2200+ |
| awesome-typescript-loader	| 3300+ | 2400+ | 2500+ (类型检查有疏漏) |

## Babel

为什么有了`typescript`，还需要 Babel？看一下对比：

| | 编译能力 | 类型检查 | 插件 |
| -- | -- | -- | -- |
| tsc | ts(x),js(x) --> es5| 有 | 无 |
| Babel | ts(x),js(x) --> es5 | 无 | 丰富 |

Babel 7 之前，是不支持 TS 的，编译流程是这样的：`TS > TS 编译器 > JS > Babel > JS (再次)`

Babel 7+ 实现了“只有一个 Javascript 编译器” 的梦想！通过允许 Babel 作为唯一的编译器来工作，就再也没必要利用一些复杂的 Webpack 魔法来管理、配置或者合并两个编译器。

**Babel 是如何处理 typescript 的？**

它移除了`typescript`。

是的，它将`typescript`全部转换为常规 JS，然后再一如既往的操作。

Babel 为什么在编译过程中剥离`typescript`？

1 基于 Babel 的优秀的缓存和单文件散发架构，`Babel + typescript` 的组合套餐会提供了更快的编译。

2 而 类型检查 呢？ 那只在当你准备好的时候进行。

`webpack5 + Babel7+`，正常配置 `babel-loader`，在`.babelrc`中配置如下：

```json
{
    "presets": [
        "@babel/preset-env",  // 这个要放第一个，并且不要配置 module: false
        [
            "@babel/preset-react",
            {
                "runtime": "automatic"  // 解决 React is not defined 问题
            }
        ],
        "@babel/preset-typescript"
    ],
    "plugins": [
        [
            "@babel/plugin-transform-runtime",  // babel 转一些 esnext 的 api 必备 runtime
            {
                "corejs": {
                    "version": 3,
                    "proposals": true
                }
            }
        ],
        "@babel/plugin-syntax-dynamic-import"  // 动态导入
    ]
}
```

**结合 typescript**

此时，只具备编译功能，再安装`typescript`补全类型检查功能。

```json
// tsconfig.json
{
  ...
  "compilerOptions":{
    "noEmit":true // 不输出文件，只做类型检查
  }
}
```

配置一下脚本：

```json
// package.json
{
  ...
  "script":{
    ...
    "check": "tsc --watch"
  }
}
```

我们需要新开一个`terminal`，跑`npm run check`，就 ok，或者使用 `npm-run-all`。

## 如何选择 typescript 编译工具

- 如果没有使用 Babel，首选 typescript 自带编译器（配合 `ts-loader` 使用）
- 如果项目中有 Babel，安装 `@babel/preset-typescript`，配合`tsc`做类型检查。

两种编译器不要混用。

end!