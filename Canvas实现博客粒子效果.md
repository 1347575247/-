## Canvas实现博客粒子效果

#### Canvas API 快速了解

> ###### [参考博客](https://www.jianshu.com/p/e70c9cfbdb38)

```js
//demo.html
<canvas id='canvas' width='500' height='500'></canvas> 
[注]
设置canvas的宽高要在元素或在其API，canvas.width || canvas.height，在CSS上设置为得到意想不到的结果，不信试试看哈···

//demo.js
var canvas = document.getElementById('canvas');
var context = canvas.getContext('2d');

//路径或图形
context.fillRect();//填充矩形
context.strokeRect();//勾勒矩形轮廓
context.arc(x,y,r,anglestart,angleend,clockwise);//画弧形

context.beginPath();//开始路径
context.moveTo();//定义路径起始点
context.lineTo();//路径的去向
context.closePath();//画完后，关闭路径
context.fill() || context.stroke();//最后画出由路径构成的图形
[注]
本质上，所有的多边形都可以由路径画出；

context.save();//保存save以上context对象设置的属性值
context.restore();//恢复到先前保存在context对象上的所有属性值
```

#### 动画API

```js
window.requestAnimationFrame(callback)
/*先前我已经说过，动画是在单位时间内按照一定顺序的图像序列的变化形成的；
这个API的功能就是，你可以在回调函数里面写一个脚本改变图形的宽高，然后这一API就会根据浏览器的刷新频率而在一定时间内调用callback；
然后，根据递归的思想，实现callback的反复调用，最终实现动画效果；
不明白，上代码*/


(function drawFrame(){
    window.requestAnimationFrame(drawFrame);
    
    //some code for animation effect here
})();

/*上面的代码意思是立即执行drawFrame这个函数，发现  window.requestAnimationFrame(drawFrame)，okay根据浏览器的刷新频率，在一定时间之后执行；
接下来执行你所编写的改变图像内容（图像的位置、宽高、颜色等等）的脚本，执行回调；
循环反复，形成动画效果*/
```

#### 动画API兼容

```js
window.requestAnimationFrame这个API你可以理解为window.setTimeout(callback,time)

//事实上，当部分浏览器不兼容这个API时，我们也可以写成以下形式：


if(!window.requestAnimationFrame){
    window.requestAnimationFrame = (
      window.webkitRequestAnimationFrame || 
      window.mozRequestAnimationFrame ||
      window.msRquestAniamtionFrame ||
      window.oRequestAnimationFrame || 
      function (callback){
          return setTimeout(callback,Math.floor(1000/60))
    }
  )
}

```

#### 动画的梳理分析：

有了前面的基础知识，现在我们就会想：如果我们能够在每16ms（1秒60帧，1000/60）内渲染1张图像，并且每一张图像的内容发生微调，那么在1秒钟整个画面就会产生动画效果了。

内容的微调可以是图形的移动的距离、转动的方向以及缩放的比例等等，而“捕获”这些数据的方法就用使用到我们以前忽视的解析几何的知识了。



## 实现Canvas粒子效果

### 一、创建基本canvas对象

#### 	html

```html
//这里的宽高无所谓，等会会通过js设置
<canvas id="canvas" width="400" height="400"></canvas>
```

#### 	js

```js
const canvas = document.getElementById("canvas")
const context = canvas.getContext("2d")
canvas.width = document.body.clientWidth
canvas.height = document.body.clientHeight
```

### 二、创建粒子数据结构

粒子分两种，一种是特殊粒子，一种是浏览器随机生成的粒子，因此分为两种数据结构：

##### 随机粒子：

注意，我们保证粒子能够往各个方向进行运动的前提就是加速度必须有正有负！因此这里xa和ya生成加速度范围为(-1,1)的数值。

```js
var dots = [] // 所有粒子`
for (var i = 0; i < 500; i++) {
    dots.push({
        x: Math.random() * canvas.width, // x  , y  为  粒子坐标
        y: Math.random() * canvas.height,
        xa: Math.random() * 2 - 1, // xa , ya 为  粒子 xy 轴加速度
        ya: Math.random() * 2 - 1,
        max: 6000 //max为  连线的最大距离
    })
}
```

##### 特殊粒子：

```js
let warea = {
    x: null,
    y: null,
    max: 20000
}
```

特殊粒子就是用户操作鼠标时，鼠标所在之处即为特殊粒子，因此他不像随机粒子一样需要加速度属性(xa和ya)，max属性是两个粒子之间连线的距离，为了方便，这里max用的是平方数。



### 三、给特殊粒子绑定鼠标事件

当用户鼠标在页面移动，则在其数据结构（warea）记录对应的坐标位置，当鼠标移出页面，重置warea的x和y为null（这里不能为0）。

```js
window.onmousemove = e => { //获取鼠标活动时的鼠标坐标
    warea.x = e.clientX
    warea.y = e.clientY
}
window.onmouseout = e => { //鼠标移出界面时清空
    warea.x = null
    warea.y = null
}
```

#### 四、添加帧处理函数

由于动画是一帧一帧转换的，当当前帧执行完毕时，下一帧应该重新开始绘制，只是参数变化了，因此我们在animate函数里面第一步就是调用clearRect函数覆盖上一帧的内容：

```js
context.clearRect(0, 0, canvas.width, canvas.height)
```

创建一个新数组，将特殊粒子对象和普通粒子对象数组合并，

```js
/*[a].concat(b)就是将a对象放到b数组首部中，跟Array.unshift()函数作用类似，但是前者结果可以存放在另一个数组中而不改变原数组*/
var ndots = [warea].concat(dots)
```

遍历普通粒子对象数组，做以下处理：

###### 1. 设置一帧的移动距离

```js
dot.x += dot.xa
dot.y += dot.ya
```

###### 2. 判断是否超过边界，如果是则将加速度设置为反向

```js
dot.xa *= dot.x > canvas.width || dot.x < 0 ? -1 : 1
dot.ya *= dot.y > canvas.height || dot.y < 0 ? -1 : 1
```

###### 3. 绘制点

```js
context.fillRect(dot.x - 1, dot.y - 1, 2, 2)
context.fillStyle = "rgba(255,218,27,1)";
```

###### 4. 循环对比当前粒子与其他粒子的距离（用来判断是否需要在两个粒子之间画线以及通过两个点的距离远近值设置线的粗细程度）

其他粒子：d2(包含特殊粒子)

当前粒子：dot

```js
for (var i = 0; i < ndots.length; i++) {
    var d2 = ndots[i]
    // 当遍历到d2等于当前粒子dot或者d2等于特殊粒子但是用户没有将鼠标移动到页面，则不继续执行
    if (dot === d2 || d2.x === null || d2.y === null) continue
    let [xc, yc, dis, ratio] = [dot.x - d2.x, dot.y - d2.y, '' , '']
    // 两个粒子之间的距离（取平方）
    dis = xc * xc + yc * yc
    // 如果两个粒子之间的距离小于粒子对象的max值
    //则在两个粒子间画线
    if (dis < d2.max) {
        // 如果是鼠标，则让粒子向鼠标的位置移动当前位置的百分之3
        if (d2 === warea && dis > d2.max / 2) {
            dot.x -= xc * 0.03
            dot.y -= yc * 0.03
    	}
        // 计算距离比
        ratio = (d2.max - dis) / d2.max
        // 画线
        context.beginPath()
        context.lineWidth = ratio / 2
        context.strokeStyle = "rgba(255,218,27,1)"
        context.moveTo(dot.x, dot.y)
        context.lineTo(d2.x, d2.y)
        context.stroke()
    }
}
```

###### 5. 在上面的for循环外，将计算过的dot从ndots数组中移除

```js
ndots.splice(ndots.indexOf(dot), 1) // 从数组中删除已经计算过的粒子
```

#### 为什么要移除?

因为我们当前只是一帧图像，在计算某个粒子与其他所有粒子的距离，当这个粒子已经计算完毕了且已经画完线了，那么下个粒子就没有必要再继续计算与上一个粒子的距离且不用重复画线，可以提高性能，且不会让线变粗，读者可以尝试将该语句删除，会发现线条变粗了且运动速度变慢了（是变卡了！），就是这个原因。

### 六、动画生成

调用window.requestAnimationFrame这个API让所有的动画帧连起来，生成动画：

```js
window.requestAnimationFrame(animate)
```

在外面调用animate()函数即可生成动画。



谢谢观看！  源代码在下方

```js
<!DOCTYPE html>
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>tong-h:粒子效果</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
      html, body {
        margin: 0;
        padding: 0;
        height: 100%;
        overflow: hidden;
        background:black;
      }
    </style>
  </head>
  <body>
    <canvas id="canvas" width="1536" height="791"></canvas>
    <script>
        const canvas = document.getElementById("canvas")
        const context = canvas.getContext("2d")
        canvas.width = document.body.clientWidth
        canvas.height = document.body.clientHeight
        let warea = {
            x: null,
            y: null,
            max: 20000
        }
        window.onmousemove = e => { //获取鼠标活动时的鼠标坐标
            warea.x = e.clientX
            warea.y = e.clientY
        }
        window.onmouseout = e => { //鼠标移出界面时清空
            warea.x = null
            warea.y = null
        }
        var dots = [] // 所有粒子`
        for (var i = 0; i < 500; i++) {
            dots.push({
                x: Math.random() * canvas.width, // x  , y  为  粒子坐标
                y: Math.random() * canvas.height,
                xa: Math.random() * 2 - 1, // xa , ya 为  粒子 xy 轴加速度
                ya: Math.random() * 2 - 1,
                max: 6000 //max为  连线的最大距离
            })
        }
        // setTimeout(animate(), 100)// 延迟100秒开始执行动画，如果立即执行有时位置计算会出错
        function animate() {
            context.clearRect(0, 0, canvas.width, canvas.height)
            var ndots = [warea].concat(dots)
            dots.forEach(dot => {  // 粒子位移
                dot.x += dot.xa
                dot.y += dot.ya
                // 遇到边界将加速度反向
                dot.xa *= dot.x > canvas.width || dot.x < 0 ? -1 : 1
                dot.ya *= dot.y > canvas.height || dot.y < 0 ? -1 : 1
                // 绘制点
                context.fillRect(dot.x - 1, dot.y - 1, 2, 2)
                context.fillStyle = "rgba(255,218,27,1)";
                // 循环比对粒子间的距离
                for (var i = 0; i < ndots.length; i++) {
                    var d2 = ndots[i]
                    if (dot === d2 || d2.x === null || d2.y === null) continue
                    let [xc, yc, dis, ratio] = [dot.x - d2.x, dot.y - d2.y, '' , '']
                    // 两个粒子之间的距离
                    dis = xc * xc + yc * yc
                    // 如果两个粒子之间的距离小于粒子对象的max值
                    //则在两个粒子间画线
                    if (dis < d2.max) {
                         // 如果是鼠标，则让粒子向鼠标的位置移动
                        if (d2 === warea && dis > d2.max / 2) {
                            dot.x -= xc * 0.03
                            dot.y -= yc * 0.03
                        }
                        // 计算距离比
                        ratio = (d2.max - dis) / d2.max
                        // 画线
                        context.beginPath()
                        context.lineWidth = ratio / 2
                        context.strokeStyle = "rgba(255,218,27,1)"
                        context.moveTo(dot.x, dot.y)
                        context.lineTo(d2.x, d2.y)
                        context.stroke()
                    }
                }
                ndots.splice(ndots.indexOf(dot), 1) // 从数组中删除已经计算过的粒子
            })
            window.requestAnimationFrame(animate)
        }
        // window.requestAnimationFrame(animate)
        animate()
    </script>
  

</body></html>
```

