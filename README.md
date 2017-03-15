# Animator

A small (1.6kb compressed and 1.1kb gziped!) high-performance animation library. 

Provide promise-based API.

## Installation

In a browser:

```html
<script src="https://s1.ssl.qhres.com/!bd39e7fb/animator-0.2.0.min.js"></script>
```

## API

### class Animator(duration, progress, easing)

Create an animation with `duration` millisecond.

```js
var a1 = new Animator(1000,  function(p){
  var tx = 100 * p;
  block.style.transform = 'translateX(' 
    + tx + 'px)';     
});

var a2 = new Animator(1000,  function(p){
  var ty = 100 * p;
  block.style.transform = 'translate(100px,' 
    + ty + 'px)';     
});

var a3 = new Animator(1000,  function(p){
  var tx = 100 * (1-p);
  block.style.transform = 'translate(' 
    + tx + 'px, 100px)';     
});

var a4 = new Animator(1000,  function(p){
  var ty = 100 * (1-p);
  block.style.transform = 'translateY('  
    + ty + 'px)';     
});


block.addEventListener('click', async function(){
  while(1){
    await a1.animate();
    await a2.animate();
    await a3.animate();
    await a4.animate();
  }
});
```

### animate()

Start the animation and return a promise.

### ease(easing)

Return a new animation with a new easing.

```js
var easeInOutBack = BezierEasing(0.68, -0.55, 0.265, 1.55);
//easeInOutBack

var a1 = new Animator(2000, function(ep,p){
  var x = 200 * ep;

  block.style.transform = 'translateX(' + x + 'px)';
}, easeInOutBack);

var a2 = a1.ease(p => easeInOutBack(1 - p)); //reverse a1

block.addEventListener('click', async function(){
  await a1.animate();
  await a2.animate();
});

```

### cancel()

Cancel the animation and reject the promise.

## Develop & Build

Download the codebase and run:

```bash
npm install
```

You can start a sever through:

```bash
npm start
```

Build and deploy the JS file:

```bash
npm run build
```

## License

MIT

具体实现
其实整个库实现起来并不复杂，只需要将基础动画封装为 Promise 就可以了。

不过在这里，为了兼容老版本的浏览器，我们先对一些基础函数进行封装：

function nowtime(){
  if(typeof performance !== 'undefined' && performance.now){
    return performance.now();
  }
  return Date.now ? Date.now() : (new Date()).getTime();
}
我们说动画是关于时间的函数，因此我们需要一个简单的获取时间功能。在新的 requestAnimationFrame 规范中，frame 回调的参数 timestamp 是一个 DOMHighResTimeStamp 对象，它比 Date 的计时要更精确（可以精确到纳秒）。因此获取时间我们优先使用 performance.now()，如果浏览器不支持 performance.now()，我们再降级使用 Date.now()。

接下来，我们对 requestAnimationFrame 进行 polyfill：

if(typeof global.requestAnimationFrame === 'undefined'){
  global.requestAnimationFrame = function(callback){
    return setTimeout(function(){ //polyfill
      callback.call(this, nowtime());
    }, 1000/60);
  }
  global.cancelAnimationFrame = function(qId){
    return clearTimeout(qId);
  }
}
然后，是具体的 Animator 实现：

function Animator(duration, update, easing){
  this.duration = duration;
  this.update = update;
  this.easing = easing;
}

Animator.prototype = {

  animate: function(){

    var startTime = 0,
        duration = this.duration,
        update = this.update,
        easing = this.easing,
        self = this;

    return new Promise(function(resolve, reject){
      var qId = 0;

      function step(timestamp){
        startTime = startTime || timestamp;
        var p = Math.min(1.0, (timestamp - startTime) / duration);

        update.call(self, easing ? easing(p) : p, p);

        if(p < 1.0){
          qId = requestAnimationFrame(step);
        }else{
          resolve(self);
        }
      }

      self.cancel = function(){
        cancelAnimationFrame(qId);
        update.call(self, 0, 0);
        reject('User canceled!');
      }

      qId = requestAnimationFrame(step);
    });
  },
  ease: function(easing){
    return new Animator(this.duration, this.update, easing);
  }
};

module.exports = Animator;
Animator 构造的时候可以传三个参数，第一个是动画的总时长，第二个是动画每一帧的 update 事件，在这里可以改变元素的属性，从而实现动画，第三个参数是 easing。其中第二个参数 update 事件回调提供两个参数，一是 ep，是经过 easing 之后的动画进程，二是 p，是不经过 easing 的动画进程，ep 和 p 的值都是从 0 开始，到 1 结束。（为什么要使用 ep 和 p，在前一个动画教程里已经说明了。）

Animator 有一个 animate 的对象方法，它返回一个 promise，当动画播放完成时，它的 promise 被 resolve，使用者还可以在 promise resolve 前调用 cancel 方法，这样它的 promise 会被 reject。

于是这样，很简单地我们就通过将 animator 封装为带有返回 Promise 接口的方法，实现了动画序列。它的实现虽然简单，但功能却是很强大的，用它实现的动画代码也很优雅：


我们还提供了一个 ease 方法（0.2.0+版），能够传入新的 easing，并返回新的 Animator 对象，这样我们就可以在原动画的基础上扩展我们的动画效果：


用 CSS3 如何？
的确，许多动画可以用 CSS3 来实现。不过 JavaScript 动画与 CSS3 动画有其不同的特点和使用场景。总体来说， CSS3 动画适用于任何纯展现效果的简单动画。虽然它也能提供基本的动画组合方法（有 animationEnd 时间，但标准化较晚），但操作起来依然不方便，而且还需要 JavaScript 来控制。有些动画库用降级的方式，能采用 CSS3 动画的采用 CSS3 动画，不能的自动降级为 JavaScript 动画，这不失为一种好方式，但也有利有弊。因为 CSS3 动画是绑定为操作元素属性的，而 JavaScript 更灵活一些。就像我们这个封装的动画库，其实提供的是更底层的 API，操作的只是时间和进度，并没有耦合任何元素、属性或者其他展示类的东西，因此它完全可以用来操作 DOM、Canvas、SVG、音频/视频流甚至是其他异步动作。另外，如果在动画过程中需要有其他一些精细的动作处理，也还是应该使用 JavaScript 动画而不是 CSS3 动画。
