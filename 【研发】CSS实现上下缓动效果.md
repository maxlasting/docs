# CSS实现上下缓动效果

@ 经常可以看到上下（左右）来回缓动的 CSS 动画效果，例如本站点的回到顶部。

![](https://cdn.maxlasting.com/doc-assets/202208311352365.gif)

很简单的 CSS3 动画，但是想做到这种效果，还真就需要稍微思考一下，代码实现如下：

```css
.some-class {
    animation: up-down 1s linear infinite;
}

@keyframes up-down {
    25% {
        transform: translateY(-4px);
    }
    50%, 100% {
        transform: translateY(0);
    }
    75% {
        transform: translateY(4px);
    }
}
```