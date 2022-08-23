# 使用 request 发送流数据

@ 这篇笔记是很多年前在学而思开发语音识别服务时候，遇到的请求数据的坑总结的，时隔多年，翻出来看看回忆满满。虽然目前大多数请求都使用 axios，但是在某些特殊场景下 axios 还是有一定局限性的，这里也当是复习一下，以后可能还会用到。

![](https://cdn.maxlasting.com/doc-assets/202208191914845.png)

## request 模块

request 模块主要是在 node 中发送数据请求的，可以使用 Promise 封装的版本：

```shell
npm i -S request request-promise-native
```

那么基本的使用方法在其[示例](https://www.npmjs.com/package/request)中已经有了，就不多做介绍了。这里主要说的是，使用 request 发送流对象，以及一些注意事项，但在此之前需要先了解下可读流对象。

## 流数据
通常使用流数据会有两种情况：

1. 一种是发送的文件较大，为了性能考虑，使用流数据发送
2. 需要对不同类型的数据进行拼装，然后一起发送

那么想发送流对象，就需要创建一个可读流，在 node 中很容易办到：

```js
const { Readable } = require('stream')

const readable = new Readable()

// 接下来可以把需要的数据 push 到流对象中
const data = {}
readable.push(data)
// 注意 push 完所有数据之后，一定要 push 一个 null
// 来告诉 流对象 此时已经读取完了
readable.push(null)
```

重点来了，如果你需要发送的数据中，一会有字符串 一会有 二进制 一会有其它的数据，然后还需要一股脑的发送给后端，那么怎么办呢？

这个时候 node 提供的 Buffer 对象就派上用场了，现将所有的数据全部转为 Buffer，具体的看个例子：

```js
const BOUNDARY = 'PP**OCR**LIB'

const a_boundary = '\r\n--' + BOUNDARY + '_a\r\n'
const b_boundary = '\r\n--' + BOUNDARY + '_b\r\n'
const c_boundary = '\r\n--' + BOUNDARY + '_c\r\n'

const json = JSON.stringify({
  token: 'xxx',
  nor_ans: 'xxx'
})

const data = await readFile(xxx)

const d = Buffer.concat([
   Buffer.from(a_boundary),
   Buffer.from(jsonData),
   Buffer.from(b_boundary),
   Buffer.from(imgData),
   Buffer.from(c_boundary)
])

const readable = new Readable()
readable.push(d)
readable.push(null)
```

那么这么做有什么坑呢，回到 request ～

## 分包发送和禁用分包

http 1.1 版本中，开启了 keep-alive 模式，那么在发送不确定大小的数据的时候，会在请求头 header 中，设置一个属性：Transfer-Encoding: chunked。
那么这个时候数据会会分包发送，那么在正常情况下是没什么问题的。但是也有非正常情况。

看看下面的这个请求：

![](https://cdn.maxlasting.com/doc-assets/202208191902161.jpg)

这是发送给 C++ 服务的一个混合数据流，那么分包发送，服务端就无法正确处理数据，因为会在每个包之前放上一个字符串数字。

在使用 request 发送的时候，如果不是流数据，可以这样配置：

```js
const res = await requestApi.post('http://127.0.0.1:8209/', {
  headers: {
    'Content-Type': 'multipart/form-data; boundary=xxx',
    'Connection': 'Keep-Alive'
  },
  multipart: {
	  // 设置这个字段
    chunked: false,
    data: [
      { body: xxx }
    ]
  }
})
```

那么如果是一个流对象，那么请求就会默认变成 chunked 模式，解决办法也是有的：设定一个正确的`Content-Length`

那么这个正确的 length 从哪里来呢，还记得上面的 Buffer 么，是的～

```js
const d = Buffer.concat([
   Buffer.from(a_boundary),
   Buffer.from(jsonData),
   Buffer.from(b_boundary),
   Buffer.from(imgData),
   Buffer.from(c_boundary)
])

const len = d.length
```

上面的 len 就是正确的 `Content-Length`

于是，在请求的 header 中加上`Content-Length`字段吧：

```js
const res = await requestApi.post('http://127.0.0.1:8209/', {
  headers: {
    'Content-Type': 'multipart/form-data; boundary=xxx',
    'Connection': 'Keep-Alive',
    'Content-Length': len
  },
  multipart: {
	  // 设置这个字段
    chunked: false,
    data: [
      { body: 可读流对象 }
    ]
  }
})
```

到此为止，就记录完毕如何使用 request 发送 流对象。

完！