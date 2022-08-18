# Mac 终端(terminal) oh-my-zsh+solarized配置

[原文链接](https://zhuanlan.zhihu.com/p/281085762)

> 本片文章只记录个人平时换新设备配置系统 terminal 过程, 个人觉得 Mac 系统的终端足够日常中使用,也有用过 iTem2, 也不错, 个人喜好, 进入正文:

## 功能

- 命令行补全
- 自动提示符
- 语法高亮
- Git 仓库状态
- 优美的界面

<img width="100%" src="https://pic4.zhimg.com/80/v2-0708ffeadf113f6f2f04b321fa499e97_1440w.jpg" />

## 配色方案

使用 Solarized 的主题配色方案点击 官网下载链接:

[Click here to download latest version](http://ethanschoonover.com/solarized/files/solarized.zip)

得到解压目录:双击安装 Solarized Dark ansi.terminal

<img width="100%" src="https://pic3.zhimg.com/80/v2-14e134f72d171246f6580088d9ff1a72_1440w.jpg" />

## 安装 oh-my-zsh

使用 curl 安装：

```shell
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

或使用 wget：

```shell
`sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

## 主题

安装成功后，用 vim(或者在根目录下找到并)打开隐藏文件 .zshrc ，修改主题为 agnoster：

ZSH_THEME="agnoster"

应用 agnoster 这个主题需要特殊的字体支持，否则会出现乱码情况，这时我们来配置字体：

- 下载安装 [Meslo](https://raw.githubusercontent.com/powerline/fonts/master/Meslo%20Slashed/Meslo%20LG%20M%20Regular%20for%20Powerline.ttf) 字体
- 在 terminal 中应用 Meslo 字体:

<img width="100%" src="https://pic2.zhimg.com/80/v2-4406932469b5049506b5801bdb3b00a5_1440w.jpg" />

agnoster 为大多数人使用的主题,我个人用的是 ys, 可以去 oh-my-zsh 官网选择其他主题

[点击选择主题](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)

## 自动提示命令配置

当我们输入命令时，终端会自动提示你接下来可能要输入的命令，这时按 → 便可输出这些命令，非常方便，如图:

<img width="100%" src="https://pic1.zhimg.com/80/v2-9532498730c0b85414e7251514ca299c_1440w.png" />

完成自动提示功能如下:

1. 克隆仓库到本地 ~/.oh-my-zsh/custom/plugins 路径下

```shell
cd ~/.oh-my-zsh/custom/plugins
git clone https://github.com/zsh-users/zsh-autosuggestions.git
```

2. 用 vim(或者在根目录下找到并)打开 .zshrc 文件，找到插件设置命令，默认是 plugins=(git) ，我们把它修改为:

```shell
plugins=(zsh-autosuggestions git)
```

## 语法高亮配置

1. 使用 homebrew 安装 zsh-syntax-highlighting 插件。

```shell
brew install zsh-syntax-highlighting
```

2. 配置.zshrc 文件，插入一行。

```shell
source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

3. 输入命令。

```
source ~/.zshrc
```

end!