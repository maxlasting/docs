# 让 vscode 正确识别 webpack alias 路径的方法

@ 一般的相对路径引入依赖文件，vscode能够正确识别，做出智能提示. 但是有时候项目目录层级太深，写相对路径很长，非常容易出错，所以一般我们会在webpack中配置alias，使用短名来减少路径层级。

但是vscode并不认识`@`，所以就不会提示后面的文件提示。解决办法就是在项目根目录下添加一个`jsconfig.json`的文件，配置如下：

```json
{
    "compilerOptions": {
        "baseUrl": ".",
        "paths": {
            "@/*": [
                "src/*"
            ]
        }
    },
    "exclude": [
        "node_modules",
        "dist"
    ],
    "include": [
        "src"
    ]
}
```

`paths`的配置跟`webpack`的`alias`差不多, `key`为别名，对应一条或多条实际项目路径. 配置之后，`vscode`就能正确识别`@`了。
