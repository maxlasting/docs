# Mac下的 tree 命令的使用

@ 是不是见过很多博客里面有那种文件目录树的结构展示，你也想要生成这样的目录树层结构吗？

```
├── build
├── config
├── docs
│   └── static
│       ├── css
│       └── js
├── src
│   ├── assets
│   ├── components
│   ├── store
│   │   └── modules
│   └── views
│       ├── book
│       └── movie
└── static
```

那么，跟我一起来按照下面的操作即可～

## 安装 tree

```shell
brew install tree
```

安装成功后，可以使用 `tree --help` 查看它的基本功能，有很多，这里不一一举例，只说一些常用的～

## 使用 tree

1. 输出你的树层目录结构

	* cd 目标文件夹路径
	* 让后 tree 一下，会将该层级下所有文件都遍历了输出，不管层级多深

2. 可以在目录遍历时使用 -L 参数指定遍历层级


```shell
tree -L 2
```

3. 如果你想把一个目录的结构树导出到文件 Readme.md ,可以这样操作

```shell
tree -L 2 >README.md # 然后我们看下当前目录下的 README.md 文件
```

4. 只显示文件夹

```shell
tree -d
```

5. 过滤不想要显示的文件或者文件夹，比如要过滤项目中的node_modules文件夹

```shell
tree -I “node_modules”
```