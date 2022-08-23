# 解读 node 中的 events 模块

@ 在 node 中，流（stream）是许许多多原生对象的父类，但是，当我们沿着“族谱”往上看时，会发现 EventEmitter 类是流（stream）类的父类，所以可以说，EventEmitter 类 是 node 的根基类之一，地位可显一般。虽然EventEmitter 类暴露的接口并不多而且十分简单，并且是少数纯 JavaScript 实现的模块之一，但因为它的应用实在是太广泛，身份太基础，所以在它的实现里处处闪光着一些优化代码执行效率，和保证极端情况下代码结果正确性的小细节。在了解之后，我们也可以将其使用到我们的日常编码之后，学以致用。

## events 模块的基本使用

在使用 events 模块之前我们先需要：

```js
const Event = require('events')
```

可以使用ES6的extends继承这个类，例如：

```js
const Events = require('events')

class Ev extends Events {}

const ev = new Ev()

ev.once('newListener', (event, listener) => {
  if (event === 'a') {
    console.log('a 被触发之前的事件')
    // listener 为要触发的事件
  }
})

const fn = function () {
  console.log('fn事件触发了')
}

ev.on('a', fn)

ev.emit('a')
```

基本的使用就简单的说到这里，请接着看下面的部分，看完了你马上就会明白每个API都是干什么的了。


## events 模块源码解析

如果你对这个模块的实现比较感兴趣，为了更进一步的了解这个模块，好，现在就让我们跟随 node 项目中的 lib/events.js 中的代码，来主意解析每一行源码，在这个过程中你将会学到：

	* 效率更高的 键 / 值 对存储对象的创建。
	* 如何防止在一个事件监听器中监听同一个事件导致死循环。
	* 效率更高的不定参数的函数调用。
	* 效率更高的从数组中去除一个元素。
	* emitter.once 是怎么办到的？

> 带着上面的问题来看看下面的源码解析把，加完注释本人已经肾虚了，可能有一些错别字，欢迎纠错。

为了节省篇幅，只对源码中涉及的主要流程进行讲解。

首先先从构造函数开始：

```js
// 对象的构造函数
function EventEmitter() {
  // 只做一件事，使用自身的 init 函数初始化所有属性
  EventEmitter.init.call(this);
}
// 导出的就是这个构造函数，所以使用的时候可以直接实例化或者继承这个构造函数
module.exports = EventEmitter;
// 这里是应该是 12.x 中的源码，应该是后来某个版本新加上的，方便直接使用 once 方法
module.exports.once = once;
```

接下来声明了一些自身的属性：

```js
// 储存undefined值，用来给自身使用，因为 undefined 可以被赋值成其它数据 例如：let undefined = 1
EventEmitter.prototype._events = undefined;
// 在设置最大监听函数时候使用的属性
EventEmitter.prototype._maxListeners = undefined;

// 默认监听函数个数的最大值，默认为10，如果超过这个数值，会有警告信息，以避免内存泄漏
var defaultMaxListeners = 10;

// 为EventEmitter函数挂载defaultMaxListeners属性的访问描述符
Object.defineProperty(EventEmitter, 'defaultMaxListeners', {
  enumerable: true, // 可以被枚举
  get: function() {  // 取值的时候直接返回这个变量的值
    return defaultMaxListeners;
  },
  set: function(arg) {
    // force global console to be compiled.
    // see https://github.com/nodejs/node/issues/4467
    console;
    // 设置的时候先检查是否是一个大于0并且不是 NaN 的数字类型
    if (typeof arg !== 'number' || arg < 0 || arg !== arg)
      throw new TypeError('"defaultMaxListeners" must be a positive number');
    defaultMaxListeners = arg;
  }
});
```

在准备好这些工作之后，接下来就是定义 init 方法：

```js
// 初始化 EventEmitter 构造函数的所有属性的 init 方法
EventEmitter.init = function() {
  /**
   * 通过这段代码可以知道：当实例化一个 EventEmitter 实例的时候
   * 实质上就是初始化了一个 _events 的空对象，用来储存后序要监听的事件
   */
  // 如果 _events 属性不存在（第一次初始化的时候）那么就初始化为一个空对象
  // 这里的 Object.getPrototypeOf(this)._events 用来获取 undefined 值
  if (!this._events || this._events === Object.getPrototypeOf(this)._events) {
    // 效率更高的 键 / 值 对存储对象的创建。
    // 去除了{}原型中无用的属性和方法，这样遍历会更高效
    this._events = Object.create(null);
    // 初始化当前事件的个数的变量
    this._eventsCount = 0;
  }
  // 用于保存当前最大监听数目，后面会用到
  this._maxListeners = this._maxListeners || undefined;
};
```

在调用 init 方法之后，就完成了整个初始化动作，接着看：

```js
// 可以设置最大的监听函数的个数，如果设置为0或者Infinity可以去除限制
EventEmitter.prototype.setMaxListeners = function setMaxListeners(n) {
  if (typeof n !== 'number' || n < 0 || isNaN(n))
    throw new TypeError('"n" argument must be a positive number');
  this._maxListeners = n;
  return this;
};

// 私有方法，如果没设置 _maxListeners，那么返回默认的最大限制数量
function $getMaxListeners(that) {
  if (that._maxListeners === undefined)
    return EventEmitter.defaultMaxListeners;
  return that._maxListeners;
}

// 获取当可以设置的最大监听函数的个数
EventEmitter.prototype.getMaxListeners = function getMaxListeners() {
  return $getMaxListeners(this);
};
```

```js
///////////////////////////////////////////////////////////////////////////
// 在使用 emit 触发某个事件的时候，可以传入事件对应的参数，这些参数通常式不固定的
// 如果使用 apply 很显然可以很容易的实现，但是这样的效率比直接指定参数的个数要
// 慢的多，nodejs为此用了一个比较 “笨” 的方法来解决这个问题，就是当参数的个数是
// 1、2、3的时候调用对应的函数，如果超过了3个参数才去调用使用 apply 的执行函数

// 没有参数的情况，以这种情况为例，下面几种都一样
/**
 * 用来触发对应事件的辅助函数
 * @param  {function || []}   handler 可以是个函数或者是一个数组
 * @param  {Boolean} isFn     用来说明第一个参数是否是个函数的布尔值
 * @param  {eventTarget} self 用来修正 this 指向
 * @param  {arg} arg1 ...     要使用的参数
 */
function emitNone(handler, isFn, self) {
  if (isFn)  // 如果是一个函数，那么直接执行
    handler.call(self);
  else { // 第二种情况是一个数组
    // 储存数组的长度
    var len = handler.length;
    // 对当前储存对应监听函数的数组进行拷贝（具体实现看最后面，这里先清楚这个函数的作用）
    /**
     * 为什么要这么做，这就涉及到第二个问题：
     *  - 如何防止在一个事件监听器中监听同一个事件，接而导致死循环？
     * 'use strict'
     *  const EventEmitter = require('events')
     *
     *  let myEventEmitter = new EventEmitter()
     *
     *  myEventEmitter.on('wtf', function wtf () {
     *    myEventEmitter.on('wtf', wtf)
     *  })
     *
     *  myEventEmitter.emit('wtf')
     *  运行上述代码，是否会直接导致死循环？答案是不会，
     *  因为拷贝出另一个一模一样的数组，来执行它，
     *  这样一来，当我们在监听器内监听同一个事件时，
     *  的确给原监听器数组添加了新的函数，
     *  但并没有影响到当前这个被拷贝出来的副本数组。
     */
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].call(self);
  }
}
// 参数一个的情况
function emitOne(handler, isFn, self, arg1) {
  if (isFn)
    handler.call(self, arg1);
  else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].call(self, arg1);
  }
}
// 参数两个的情况
function emitTwo(handler, isFn, self, arg1, arg2) {
  if (isFn)
    handler.call(self, arg1, arg2);
  else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].call(self, arg1, arg2);
  }
}
// 参数三个的情况
function emitThree(handler, isFn, self, arg1, arg2, arg3) {
  if (isFn)
    handler.call(self, arg1, arg2, arg3);
  else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].call(self, arg1, arg2, arg3);
  }
}
// 超过三个参数的时候调用
function emitMany(handler, isFn, self, args) {
  if (isFn)
    handler.apply(self, args);
  else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      listeners[i].apply(self, args);
  }
}
///////////////////////////////////////////////////////////////////////////

EventEmitter.prototype.emit = function emit(type) {
  /**
   * er 错误事件变量
   * handler 监听函数或者列表
   * len 要触发监听函数的列表的长度
   * args arguments中除了第一个参数的其它参数集合
   * i 循环时候使用
   * events 保存_event的引用
   * domain 无视这个变量
   */
  var er, handler, len, args, i, events, domain;
  var needDomainExit = false;

  // 获取要触发的事件名对应的 handler
  handler = events[type];

  // 判断要触发的事件名是否存在
  if (!handler)
    return false;

  // 用来判断handler对应的数据是一个函数还是一个列表
  var isFn = typeof handler === 'function';
  // 获取对应参数的个数
  len = arguments.length;
  // 根据参数的个数触发对应的 emit
  // 效率更高的不定参数的函数调用就是这么来的，很简单~
  switch (len) {
    // fast cases
    case 1:
      emitNone(handler, isFn, this);
      break;
    case 2:
      emitOne(handler, isFn, this, arguments[1]);
      break;
    case 3:
      emitTwo(handler, isFn, this, arguments[1], arguments[2]);
      break;
    case 4:
      emitThree(handler, isFn, this, arguments[1], arguments[2], arguments[3]);
      break;
    // slower
    default:
      // 初始化特定长度的数组，储存所有的参数
      args = new Array(len - 1);
      for (i = 1; i < len; i++)
        args[i - 1] = arguments[i];
      emitMany(handler, isFn, this, args);
  }

  return true;
};
```

到这里就完成了最重要的 emit 方法，还是挺简单的不是，下面就是一些添加事件的处理了：

```js
/**
 * 对应addListener、on、once、prependListener、prependOnceListener的api
 * @param {EventTarget} target   this 指向的对象
 * @param {string} type     事件名称
 * @param {function} listener 事件名称对应的监听函数
 * @param {boolean} prepend  如果是 true 那么就添加到所有事件的最前面，否则在最后面添加
 */
function _addListener(target, type, listener, prepend) {
  var m;
  var events; // 用于保存 _events 的引用
  var existing; //用于保存事件名字对应的数据的引用

  // 判断 listener 是否是一个函数
  if (typeof listener !== 'function')
    throw new TypeError('"listener" argument must be a function');

  // 保存 _events 的引用
  events = target._events;

  // 如果不存在 _events 属性那么创建一下，和上面初始化时候的原理一样
  // 这种情况在为一个时间名第一次指定监听函数的时候会触发
  if (!events) {
    events = target._events = Object.create(null);
    target._eventsCount = 0;
  } else {
    // Node v0.1.26 中的 'newListener' 事件
    // EventEmitter 实例会在一个监听器被添加到其内部监听器数组之前触发自身的 'newListener' 事件。
    // 'newListener' 回调中任何额外的被注册到相同名称的监听器会在监听器被添加之前被插入 。
    if (events.newListener) { // 如果注册了 newListener 事件
      // 在触发 type 事件之前先触发 newListener 事件
      // 可以通过下面的代码知道：newListener 的监听函数的参数有两个 type 和对应的 listener
      // 这里的 listener.listener 是在使用 once 绑定一次性事件的时候，挂载到包装对象函数
      // 身上的 listener 监听函数，一会看到下面处理 once 的部分的时候就会理解了，这里可以
      // 先略过，值需要知道第二个参数是事件名对应的 listener
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    // 保存事件名字对应的数据的引用
    existing = events[type];
  }

  if (!existing) {
    // Optimize the case of one listener. Don't need the extra array object.
    // 如果 existing 不存在，也就是第一次给某个事件名添加监听函数的时候
    // 让其指向对应的 listener，上面英文的意思是做一个优化。
    // 那么 existing 保存的便是 events[type] 对应数据的引用
    existing = events[type] = listener;
    // 更新事件的个数，没添加一个 就 +1
    ++target._eventsCount;
  } else {// 如果不是第一次为某个事件名添加监听函数走这里
    // 如果 existing 的类型是一个函数，也就是说第二次添加的时候
    if (typeof existing === 'function') {
      // Adding the second element, need to change to array.
      // 这个时候一个时间名对应多个监听函数，那么需要变为数组
      // 如果prepend参数为true，那么监听函数在前，时间名在后
      // 否则就反过来
      existing = events[type] = prepend ? [listener, existing] :
                                          [existing, listener];
    } else {
      // If we've already got an array, just append.
      // 如果同一个事件名是第二次之后再添加，那么在第二次之后 existing 已经是要给数组了
      if (prepend) { // 如果prepend参数为true
        // 那么添加到所有监听函数的最前面，例如 prependListener
        existing.unshift(listener);
      } else {
        // 否则就添加到最后面 也就是 on 的形式
        existing.push(listener);
      }
    }

    ////////////////////////////////////////////////////////////////////////////////////
    // 下面这部分用来处理如果一个事件名的监听函数超过了最大值时候的警告和错误信息，不需要关心 //
    // Check for listener leak
    if (!existing.warned) {
      m = $getMaxListeners(target);
      if (m && m > 0 && existing.length > m) {
        existing.warned = true;
        const w = new Error('Possible EventEmitter memory leak detected. ' +
                            `${existing.length} ${String(type)} listeners ` +
                            'added. Use emitter.setMaxListeners() to ' +
                            'increase limit');
        w.name = 'MaxListenersExceededWarning';
        w.emitter = target;
        w.type = type;
        w.count = existing.length;
        process.emitWarning(w);
      }
    }
  }
  ////////////////////////////////////////////////////////////////////////////////////

  return target;
}

// addListener 的 api
EventEmitter.prototype.addListener = function addListener(type, listener) {
  // 这里的 false 也就是添加到所有事件列队的最后面
  return _addListener(this, type, listener, false);
};

// on 和 addListener 是一回事
EventEmitter.prototype.on = EventEmitter.prototype.addListener;

// prependListener 的api
EventEmitter.prototype.prependListener =
    function prependListener(type, listener) {
       //这里的 true 也就是添加到当事件列队的最前面
      return _addListener(this, type, listener, true);
    };

////////////////////////////////////////////////////////////
// 从这里开始便是触发一次性事件的 api 实现的非常巧妙，值得学习 //

/**
 * 看下面的几个个函数要从 step 1 开始向上看
 */

// step 3
function onceWrapper() {
  // this 也就是 step 2 中的 state
  // 移除对应的事件监听
  this.target.removeListener(this.type, this.wrapFn);

  /**
   * 这里 removeListener 的第二个参数是 this.wrapFn
   * this.wrapFn 有两重身份，即是 once 中绑定的监听函数m，又是 onceWrapper.bind(state) 的引用
   * 这样在这里虽然把事件列队里面的 this.type 对应的 this.wrapFn 移除了
   * 但是 state 保存的是真正的 listener，所以不耽误 listener 的执行
   */

  if (!this.fired) { // 如果这个事件没被触发过，就进行触发
    this.fired = true; // 修改触发状态
    this.listener.apply(this.target, arguments); // 触发这个事件
  }
  // 感觉上面还是说的不太好理解，说的在通俗一点就是：触发一次性事件，
  // 使用 on 绑定的监听函数实际上是 onceWrapper.bind(state)
  // 所以这里在 emit 的时候移出的也是 onceWrapper.bind(state)
  // 压根儿跟 listener 就没扯上关系，而 listener 被储存在 onceWrapper
  // 的 this 对象中，_onceWrap 和 onceWrapper 通过 state 的关联
  // 形成了闭包，当整个事件函数执行完成之后，state 就会等着被垃圾回收
  // 这里的代码整个感觉就是高大上，不太好理解
}

// step 2
/**
 * 这个函数的设计过程：(也是我阅读代码时候想的)
 *  1 先自定义一个对象 state，用来保存需要的属性
 *  2 使用 onceWrapper 绑定这个对象
 *  3 将只触发一次的事件挂载到这个函数身上
 *  4 返回绑定了state之后的 onceWrapper
 */
function _onceWrap(target, type, listener) {
  // 设置一个对象 state：（这个对象的做用就是保存各种引用）
  // fired 是否被触发过
  // wrapFn 储存 bind 之后的返回函数
  // target 事件对象
  // type 事件名
  // listener 事件名对应的监听函数
  var state = { fired: false, wrapFn: undefined, target, type, listener };
  // 储存 onceWrapper.bind(state) 的返回函数
  // 这里 onceWrapper 实际并没有执行，只是绑定上了 state 对象而已
  var wrapped = onceWrapper.bind(state);
  // 把要触发的一次性事件挂载到上面的 wrapped 函数上
  // 还记得上面的 newListener 嘛？ 是不是一下子明白了 linstener.linstener ?
  wrapped.listener = listener;
  // 让 state 对象的 wrapFn 指向 wrapped
  state.wrapFn = wrapped;
  // 返回这个包装好的函数
  return wrapped;
}

// step 1
// api of once
// 当使用 once 方法的时候，函数的调用栈 ：
// once -> _onceWrap -> onceWrapper.bind(state) -> on
EventEmitter.prototype.once = function once(type, listener) {
  if (typeof listener !== 'function')
    throw new TypeError('"listener" argument must be a function');
  // 这里的 _onceWrap(this, type, listener) 返回的实际上 onceWrapper 经过柯里化之后函数
  this.on(type, _onceWrap(this, type, listener));
  return this;
};

// 和上面的 once 原理一样
// api of prependOnceListener
EventEmitter.prototype.prependOnceListener =
    function prependOnceListener(type, listener) {
      if (typeof listener !== 'function')
        throw new TypeError('"listener" argument must be a function');
      this.prependListener(type, _onceWrap(this, type, listener));
      return this;
    };
////////////////////////////////////////////////////////////


// 移出一个事件的api
// emits a 'removeListener' event iff the listener was removed
EventEmitter.prototype.removeListener =
    function removeListener(type, listener) {
      var list, events, position, i, originalListener;

      // 如果要移除的 listener 不是要给函数刨错
      if (typeof listener !== 'function')
        throw new TypeError('"listener" argument must be a function');

      // 保存一下_events的引用
      events = this._events;

      // 如果不存在则直接返回
      if (!events)
        return this;

      // list可能为一个函数也可能是一个数组
      list = events[type];
      // 说明要移出的事件并没有被注册过
      if (!list)
        return this;

      // 如果 list 是一个函数或者 list.listener 也就是 once 的情况
      // 还记得上面 wrapped.listener = listener; 这句代码嘛？
      if (list === listener || list.listener === listener) {
        // 如果移出这个 linstener 之后 this._eventsCount 变为0
        // 说明已经没有其它的监听函数了，那么直接清空整个 _events
        if (--this._eventsCount === 0)
          this._events = Object.create(null);
        else {
          // 否则就是删除这个事件名对应的属性
          delete events[type];
          if (events.removeListener)
            // api removeListener ，如果注册了这个事件
            // 则在删除某个事件之后触发 removeListener 事件
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') {// 如果不是一个函数，那么说明是个数组
        // 用来储存对应的事件的位置
        position = -1;
        // 下面这个循环用来寻找和 listener 匹配的监听函数的位置
        for (i = list.length - 1; i >= 0; i--) {
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }
        //上面类似数组的 indexOf 方法如果没找到返回 -1
        if (position < 0)
          return this;

        if (list.length === 1) {
          // 这里和上面执行相同的操作
          if (--this._eventsCount === 0) {
            this._events = Object.create(null);
            return this;
          } else {
            delete events[type];
          }
        } else if (position === 0) { // 如果要删除的监听函数是第一个
          list.shift(); //使用 shift 效率更高
          if (list.length === 1) // 如果删除之后 list 只剩下一个监听函数
            events[type] = list[0]; //那么就让 events[type] 的 value 变为这个监听函数就行
        } else {
          /**
           * 更高效的从一个数组中去删除一个数据
           * 详细的看下面的这个函数，原理很简单
           * 后一个覆盖前一个
           */
          spliceOne(list, position);
          if (list.length === 1)
            events[type] = list[0];
        }
        // 如果注册了 removeListener 则在移出之后触发这个函数
        if (events.removeListener)
          this.emit('removeListener', type, originalListener || listener);
      }
      // 此方法可以链式调用其它方法
      return this;
    };

// 移出所有监听函数的 api
// 和上面的原理基本一样，这里只给不一样的地方注释
// 这个方法，如果不传参数，那么会删除所有的事件
EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      /**
       * listeners 对应某个 key 的监听函数或者数组
       * events 储存 _events 的引用
       */
      var listeners, events, i;

      events = this._events;
      if (!events)
        return this;

      // not listening for removeListener, no need to emit
      if (!events.removeListener) { // 如果没注册 removeListener 事件
        if (arguments.length === 0) { // 如果没传参数
          // 直接初始化_events 属性
          this._events = Object.create(null);
          this._eventsCount = 0;
        } else if (events[type]) { // 如果参数的个数不是 0 个
          if (--this._eventsCount === 0)
            // 如果删除这个事件名之后，_eventsCount 为 0，
            // 那么对 _events 进行初始化
            this._events = Object.create(null);
          else
            // 否则就删除这个事件 key 即可
            delete events[type];
        }
        // 可以链式调用
        return this;
      }

      // 下面是如果注册了 removeListener 事件的时候，而且没有传参数
      // 那么就删除所有的事件
      // emit removeListener for all listeners on all events
      if (arguments.length === 0) { // 如果没传参数
        var keys = Object.keys(events); //得到所有的事件名 key
        var key;
        for (i = 0; i < keys.length; ++i) {
          key = keys[i];
          // 如果事件的 key 是 removeListener 则跳过
          if (key === 'removeListener') continue;
          this.removeAllListeners(key); // 递归开始删除每个事件，走下面的 if
        }
        // 全部调用完成之后，删除剩下的 removeListener 事件
        this.removeAllListeners('removeListener');
        // 初始化 _events
        this._events = Object.create(null);
        this._eventsCount = 0;
        return this;
      }

      // 储存事件名 key 的对应 value 的引用
      listeners = events[type];

      // 如果传了参数，那么删除对应的事件
      // 如果对应的 value 是个函数那么直接调用自身的 removeListener
      // 这里也是上面递归的时候走的位置
      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners) { // 如果不是函数是个数组
        // LIFO order
        // 这里使用数组的栈方法，后注册的先被删除
        // 不直接初始化 _events 是因为每次删除都去触发一次 removeListener 事件
        for (i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };

// 用来获取某个事件名的所有监听函数的数组的 api
EventEmitter.prototype.listeners = function listeners(type) {
  var evlistener;
  var ret;
  var events = this._events;

  // 如果 events 不存在 那么证明没有返回空数组
  if (!events)
    ret = [];
  else {
    // 如果存在，evlistener 保存一下 events[type] 的引用
    evlistener = events[type];
    if (!evlistener) // 如果这个引用并不存在，那么说明没注册过，返回空数组
      ret = [];
    else if (typeof evlistener === 'function')
      // 如果  evlistener 是个函数，说明只有一个监听函数而已
      // 那么返回一个数组，里面只有这个监听函数（包括 once 和 on 注册的）
      ret = [evlistener.listener || evlistener];
    else
      // 这种情况也就是对应的 value 是个数组，
      // 那么调用解除包装并返回真是的注册的监听函数的方法
      ret = unwrapListeners(evlistener);
  }

  return ret;
};

// 被废弃的 api 推荐使用下面的 listenerCount
EventEmitter.listenerCount = function(emitter, type) {
  if (typeof emitter.listenerCount === 'function') {
    return emitter.listenerCount(type);
  } else {
    return listenerCount.call(emitter, type);
  }
};

// 返回正在被监听的事件名对应监听函数的数量
EventEmitter.prototype.listenerCount = listenerCount;
function listenerCount(type) {
  const events = this._events;

  if (events) {
    const evlistener = events[type];
    // 这里如果对应的 value 是个函数
    // 那么说明只有 1 个
    if (typeof evlistener === 'function') {
      return 1;
    } else if (evlistener) {
      // 如果是个数组，那么返回数组的长度
      return evlistener.length;
    }
  }
  // 如果都不是就说明没这测这个事件监听，返回 0
  return 0;
}

// 用来获取当前对象已经注册的所有事件名的列表
EventEmitter.prototype.eventNames = function eventNames() {
  // Reflect 为 es6 的新品种
  // Reflect.ownKeys() 方法用来遍历一个对象的所有属性名
  return this._eventsCount > 0 ? Reflect.ownKeys(this._events) : [];
};

// 从数组指定位置删除一个元素，比原生的splice方法快1.5倍
// About 1.5x faster than the two-arg version of Array#splice().
function spliceOne(list, index) {
  // 这种代码组织方式，看完了又进步了
  // 总觉得那种别人不能一下看出来的代码高深莫测，这是病得治！
  // 当然下面的循环还是很简单的，就是让后面的覆盖前面的，然后把最后一个多余的干掉
  for (var i = index, k = i + 1, n = list.length; k < n; i += 1, k += 1)
    list[i] = list[k];
  list.pop();
}

// 用来克隆一个事件名对应的多个事件的数组
// 做用: 解决递归调用问题，防止死循环
function arrayClone(arr, n) {
  var copy = new Array(n);
  for (var i = 0; i < n; ++i)
    copy[i] = arr[i];
  return copy;
}

// 用来解除一次性事件的包装
// 上面的一次性事件将 linstener 挂载到了 onceWrapper 上
// 这里在获取某个事件的监听函数的时候应该获取到的是实际触发的那个
function unwrapListeners(arr) {
  const ret = new Array(arr.length);
  for (var i = 0; i < ret.length; ++i) {
    ret[i] = arr[i].listener || arr[i];
  }
  return ret;
}
```

至此，关于 events 模块的解析就结束了，感谢阅读～

