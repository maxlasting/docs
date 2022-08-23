# script 标签中 defer 和 async 的区别

![](https://cdn.maxlasting.com/doc-assets/202208231423450.jpg)

@ 当浏览器加载 HTML 并遇到`<script>...</script>`标签时，它无法继续构建 DOM。它必须立即执行脚本。外部脚本`<script src="..."></script>`也是如此：浏览器必须等待脚本下载，执行下载的脚本，然后才能处理页面的其余部分。

这导致一个重要问题：

如果页面顶部有一个庞大的脚本，它会“阻塞页面”。在下载并运行之前，用户无法看到页面内容：

```html
<p>...content before script...</p>

<script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<!-- 以下在脚本加载完之前是不可见的 -->
<p>...content after script...</p>
```

有一个解决方法，就是把脚本放到最底部。

但是对于长 HTML 文档，这可能会有明显的延迟。

## defer

`defer`属性告诉浏览器不要等待脚本，浏览器会继续处理 HTML，构建 DOM。该脚本“在后台”加载，然后在 DOM 完全构建完成后再运行。

```html
<p>...content before script...</p>

<script defer src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<!-- 不等待脚本,立即显示 -->
<p>...content after script...</p>
```

另外，defer脚本总是在 DOM 准备好时执行（但在DOMContentLoaded事件之前）

```html
<p>...content before scripts...</p>

<script>
  document.addEventListener('DOMContentLoaded', () => alert("DOM fully loaded and parsed after defer!"));
</script>

<script defer src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<p>...content after scripts...</p>
```

1. 页面内容立即显示。
2. DOMContentLoaded事件处理程序等待defer脚本执行完之后执行。

> 当纯 HTML 被完全加载以及解析时，DOMContentLoaded 事件会被触发，而不必等待样式表，图片或者子框架完成加载。

defer脚本保持相对顺序来执行，就像常规脚本一样，例如：我们有两个延迟脚本：`long.js`和`small.js`：

```html
<script defer src="https://javascript.info/article/script-async-defer/long.js"></script>
<script defer src="https://javascript.info/article/script-async-defer/small.js"></script>
```

这两个脚本会并行下载`small.js`可能会比`long.js`先下载完成，但是执行的时候依然会先执行`long.js`。

所以defer可用于对脚本执行顺序有严格要求的情况。

## async

async属性意味着该脚本是完全独立的：

浏览器不会阻止`async`脚本，其他脚本也不会等待`async`脚本，async脚本也不会等待其他脚本，`DOMContentLoaded`和async脚本不会互相等待，`DOMContentLoaded`可能在`async`脚本执行之前触发（如果`async`脚本在页面解析完成后完成加载），或在`async`脚本执行之后触发（如果async脚本很快加载完成或在 HTTP 缓存中），简单来说就是`async`脚本在后台加载完就立即运行。

```html
<p>...content before scripts...</p>

<script>
  document.addEventListener('DOMContentLoaded', () => alert("DOM 完全加载以及解析"));
</script>

<script async src="https://javascript.info/article/script-async-defer/long.js"></script>
<script async src="https://javascript.info/article/script-async-defer/small.js"></script>

<p>...content after scripts...</p>
```

- 页面内容立即显示：async不阻塞。
- DOMContentLoaded可能发生在async之前或之后。
- small.js先加载完就会在long.js之前执行，但如果long.js在之前有缓存，那么long.js先执行。

应用场景：将独立的第三方脚本集成到页面中时，比如计数器，广告等。

> 注意：async和defer属性都仅适用于外部脚本，如果script标签没有src属性，尽管写了async、defer属性也会被忽略。

## Dynamic scripts

还有一种方式可以将脚本添加到页面：Dynamic scripts 动态脚本。

可以创建一个脚本并使用JavaScript将其动态添加到文档中：

```js
let script = document.createElement('script');
script.src = "/article/script-async-defer/long.js";
document.body.append(script); // (*)
```

当脚本被添加到文档后立即开始加载。

默认情况下，动态脚本表现为`async`。

当然也可以设置`script.async=false`，这样脚本会表现为`defer`。

```js
function loadScript(src) {
  let script = document.createElement('script');
  script.src = src;
  script.async = false;
  document.body.append(script);
}

// long.js runs first because of async=false
loadScript("/article/script-async-defer/long.js");
loadScript("/article/script-async-defer/small.js");
```

所以`long.js`会先执行。

## 总结

script 是会阻碍 HTML 解析的，只有下载好并执行完脚本才会继续解析 HTML。

defer 和 async有一个共同点：下载此类脚本都不会阻止页面呈现（异步加载），区别在于：

1. async 执行与文档顺序无关，先加载哪个就先执行哪个；defer会按照文档中的顺序执行
2. async 脚本加载完成后立即执行，可以在DOM尚未完全下载完成就加载和执行；而defer脚本需要等到文档所有元素解析完成之后才执行。

![](https://cdn.maxlasting.com/doc-assets/202208231437738.png)

end!