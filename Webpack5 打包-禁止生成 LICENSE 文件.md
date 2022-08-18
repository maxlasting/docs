# Webpack5 打包-禁止生成 LICENSE 文件

@ 最近使用 webpack5 在构建的时候发现会多出来一个 `LICENSE` 文件，顿时强迫症发作…

![](&&&SFLOCALFILEPATH&&&077630CC-5FBB-43FA-979E-F4868F817F22.png)

## 问题分析

多输出了一些文件，那这可能是打包的输出配置出了岔子或者是用的插件中配置不太对。

于是检查自己的配置…

没发现什么问题呀…没看到有啥配置会让我打包输出这些文件的…

于是再仔细看看官方文档吧。

果然在这里发现了蛛丝马迹：

![](&&&SFLOCALFILEPATH&&&6F9654CD-D55A-4C53-9A94-271F99A9288F.png)

![](&&&SFLOCALFILEPATH&&&BC15DE33-093D-4D47-912E-B2DFF52FAF0B.png)

extractComments默认为true，所以会将注释提取到单独的文件中，生成本文头发所见的LICENSE文件。

# 解决方案

问题原因已经找到了，开始着手解决吧。

修改 `terser-webpack-plugin` 配置，由于Webpack5已经集成 `terser-webpack-plugin`无需单独安装，直接引入：

```js
const TerserPlugin = require("terser-webpack-plugin")

const config = {
	  // ....
    optimization: {
        minimize: true,
        minimizer: [
            // webpack5 默认继承此插件，这么做的目的是为了去掉 LICENSE 文件
            new TerserPlugin({
                extractComments: false,
            }),
        ]
    },
    // ....
}
```

重新打包，问题解决。