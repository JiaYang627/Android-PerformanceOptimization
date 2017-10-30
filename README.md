# 性能优化
***
* [卡顿](#卡顿)
* [如何衡量卡顿](#如何衡量卡顿)
* ["卡顿" 产生的原因](#"卡顿"产生的原因)
* [Profile GPU Rendering](#ProfileGPURendering)
* [通用优化流程](#通用优化流程)
    * [第一步：UI层优化](#第一步：UI层优化)
        * [过度绘制](#过度绘制)
        * [自定义控件绘制优化](#自定义控件绘制优化)
        * [Hierarchy Viewer(层级查看器)工具使用](#HierarchyViewer(层级查看器)工具使用)
    * [第二步：代码问题查找](#第二步：代码问题查找)
    * [第三步：优化App的逻辑层](#第三步：优化App的逻辑层)


### 卡顿

> 前言：关于Android性能优化,为何把"卡顿"放在前面 ？ 对于用户而言,可能注重的是App的流畅度,而如果AppUI卡顿,用户体验就不是很好了。所以想说明一下什么是"卡顿",并了解一般卡顿的原因,这样才能解决卡顿。

* "卡顿"是什么？
    * "卡顿" 直观的来讲，就是人的一种视觉感受，比如我们滑动界面时，如果滑动不流程我们就会有卡顿的感觉，这种感觉我们需要有一个量化指标，在编程时如果开发的程序超过了这个指标我们认为其是卡顿的。
    
    * **FPS(帧率)** ：每秒显示帧数（Frames per Second）。表示图形处理器每秒钟能够更新的次数。高的帧率可以得到更流畅、更逼真的动画。一般来说12fps大概类似手动快速翻动书籍的帧率，这明显是可以感知到不够顺滑的。30fps就是可以接受的，但是无法顺畅表现绚丽的画面内容。提升至60fps则可以明显提升交互感和逼真感，但是一般来说超过75fps就不容易察觉到有明显的流畅度提升了，如果是VR设备需要高于75fps，才可能消除眩晕的感觉。
    * 开发app的性能目标就是保持60fps，这意味着每一帧你只有16ms≈1000/60的时间来处理所有的任务。Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需要的60fps。
    
> 如图所示


![Image](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/9FZ5rJefVqgeX9pmDv9CfehKayZSPr3tC9XSOrERK4c!/b/dA4BAAAAAAAA&bo=5QHWAAAAAAADBxA!&rf=viewer_4)

* 如果你的某个操作花费时间是24ms，系统在得到VSYNC信号的时候就无法进行正常渲染，这样就发生了丢帧现象。那么用户在32ms内看到的会是同一帧画面。

> 如图所示

![Image](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/9IIVaVopt5ds2b8y8Fx3DKUI930tEdZI.NGYlz0rd5Q!/b/dPkAAAAAAAAA&bo=wgE6AQAAAAADAN0!&rf=viewer_4)

* 如果此时用户在看动画的执行或者滚动屏幕（如RecyclerView），就会感觉到界面不流畅了（卡了一下）。丢帧导致卡顿产生。

**流畅的情况下：**

![Image](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/76qIAgqViA1Dw9aiyY6yIlCkdaNOyo2WzU0cE9UJTjE!/b/dBEBAAAAAAAA&bo=twFuAQAAAAADAPw!&rf=viewer_4)

**出现了丢帧现象（卡顿）：**

![Image](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/TirqJspyU0rgnJozFiZKCsqRYZjYf1S7taz5sThbfOE!/b/dBoBAAAAAAAA&bo=mgFxAQAAAAADAM4!&rf=viewer_4)

**严重丢帧（卡死了）：**

![Image](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/Fr*cTiv5FLJkHJJBOcIKpPQSlt*uucdX*zDvyeZWxpo!/b/dJYAAAAAAAAA&bo=nAF7AQAAAAADAMI!&rf=viewer_4)
    

***

### 如何衡量卡顿

* FPS的高低不能准确的反映应用的流程度。只有有更新的时候才刷新界面。当界面没有变动的时候，手机不需要对界面进行更新，所以此时的FPS会很低，如果1秒钟内都没有变动那么FPS=0。所以我们需要利用其他方式来衡量应用的流程度，比如可以利用丢帧数来衡量。
* 单位时间内丢帧数可以反映出应用是否流程。不丢帧是终极目标，但每秒丢帧在6-7帧左右可以接受，如果丢10帧以上就需要优化了。



丢帧情况（单位时间内均匀分布） | 卡顿情况
---|---
0-10帧 | 流畅
10-20帧 | 较卡
20-40帧 | 很卡
40-60帧 | 卡死了


对于我们开发人员来说,会使用一些工具找出卡顿比较集中的地方,找出原因,消除或减弱卡顿。(测试团队会有专门的工具去测试丢帧的情况)


### "卡顿" 产生的原因

* 核心：分析在16ms中我们的应用做了什么工作，哪些工作阻止我们在16ms时更新界面。
* 通常情况下，在16ms中我们有那些工作需要处理。单以XML布局被绘制出来为例进行说明。
* 处理过程：
    * CPU负责把UI组件计算成多边形和纹理
    * OpenGL负责绘制图像（Display List）
    * GPU栅格化需要显示内容并渲染到屏幕上

* 而实际开发中我们还加入交互、业务处理等工作，这些工作都需要在16ms中处理完成。对于开发人员来说，需要有一个工具，很直观的帮助我们判断出那些工作占用了多少时间。

### Profile GPU Rendering

* 通过手机开发者选项中提供的Profile GPU Rendering（GPU呈现模式分析）功能，我们可以清楚的看到处理流程中各部分的耗时。手机端工具（开发助手GPU渲染图）。建议大家在Android6.0及以上手机测试。


* 打开Profile GPU Rendering操作截图如下：

大家可以拿着真机配置一下。

![PRG-1](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/hr5n39BPJ8UvtYL0LUu97wRORkPjIeU8lF4ag0TPc7E!/b/dA4BAAAAAAAA&bo=8ACJAQAAAAADB1o!&rf=viewer_4)


![PRG-2](http://a3.qpic.cn/psb?/V14YlNrL2eQEkW/Hqe78BFqEqlP*mTMogfLogDhFyte*dimriU073YSKME!/b/dLMAAAAAAAAA&bo=YgEuAgAAAAADAGo!&rf=viewer_4)

![PRG-3](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/zBY9ChCCSjetzE94*BjBE*984NB8dLh5xKTIynCumqg!/b/dEIAAAAAAAAA&bo=UAETAgAAAAADAGU!&rf=viewer_4)


***

![PRG-4](http://a3.qpic.cn/psb?/V14YlNrL2eQEkW/s5kHMaPakIPbDMnKNwSP3dSuFt88jhtwQxMLXXUt8fU!/b/dBABAAAAAAAA&bo=xgCLAQAAAAADAGk!&rf=viewer_4)

![PRG-5](http://a3.qpic.cn/psb?/V14YlNrL2eQEkW/hRLbSRzY7XXBtNqqroy4JQ.DajqN457Df1wOjuxdiy0!/b/dBABAAAAAAAA&bo=kQJ7AQAAAAADAMw!&rf=viewer_4)

* 条形图说明:
    * 水平方向的一根绿线代表16ms。
    * 每条都代表一帧画面所有工作内容。
    * 每条中不同的颜色代表不同的工作内容。

* Android6.0及以上的手机颜色对应关系如下：

![PRG-6](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/SBnCoSgc4kyOYuHZgN0hhqkCv*I*D8WlnEFIXFD5XVI!/b/dA4BAAAAAAAA&bo=ngNnAgAAAAADAN0!&rf=viewer_4)

* 原因分析：
https://developer.android.google.cn/topic/performance/rendering/profile-gpu.html##visrep

***

## 通用优化流程


### 第一步：UI层优化

* UI问题比较容易查找
* 一旦出现问题影响范围广（xml、mesure、layout、draw、Display List、栅格化……）

工具：设备过渡绘制查看功能、Hierarchy Viewer等
常见问题：过渡绘制、布局复杂、层级过深……

### 过度绘制

* 在屏幕一个像素上绘制多次（超过两次）。如：文本框，如果设置了背景颜色（黑色），那么显示的文字（白色）就需要在背景之上再次绘制。
* 打开手机开发者中的**GPU过度绘制**即可查看。蓝色标识这个区域绘制了两次。(6.0以上)。

![过度绘制](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/7FoBxIpMd8lvyTGui*cqwyFyPeWr5tdkvsRVrzGcdjI!/b/dA4BAAAAAAAA&bo=KQFKASkBSgEDByI!&rf=viewer_4)


* 说明：
    * 如果大面积都是蓝色，属于正常情况。
    * 重点关注大面积绿色及以后的，表示存在过渡绘制。


#### 自定义控件绘制优化

* Clip Rect 与 Quick Reject

Clip Rect：识别可见区域
Quick Reject：控件所在的矩形区域是否有交集

* 在Canvas中有上述两个方法，帮助我们进行判断，避免出现过渡绘制。

> 案例

**案例效果：**

![自定义绘制-1](http://a2.qpic.cn/psb?/V14YlNrL2eQEkW/izHIIGndx9TxR9.Cf*8HsE*FDgmE9dnyoV2.wYq4*Hg!/b/dDQAAAAAAAAA&bo=QgEAAkIBAAIDByI!&rf=viewer_4)


* 自定义一View，初始化就不在贴代码，在onDraw的时候，通常我们都是这样写：

```
for (int i = 0; i < imgs.length; i++) {
    canvas.drawBitmap(imgs[i],i*20,0,paint);
}
```
* 这样写出来后，打开GPU过度绘制，结果：

![自定义绘制-2](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/ZrJn30ftd6mUCeu3gSaj.XJ31oVaVC9Bx4CwMQKOb90!/b/dLEAAAAAAAAA&bo=QgEAAkIBAAIDACU!&rf=viewer_4)


* 我们可以看出 中间的部分(第三张开始)已经开始了很重的过度绘制。原因很简单：对于“大王”这张牌来说，我们不需要绘制完整的图片，如果都绘制了就会出现上面的情况。
* 处理思路：
    * 找出牌需要绘制的区域，让canvas在绘制这张牌时仅仅按区域绘制一部分即可。对于“大王”这张牌来说我们仅仅绘制如下内容。

![自定义绘制-3](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/LdYVhclbJSlWwTp2SeBSLDtu..5Dw6fcm6.B1sSO5a0!/b/dA4BAAAAAAAA&bo=QgEAAkIBAAIDACU!&rf=viewer_4)

* 在Canvas中clipRect方法可以帮助我们划定一个区域，进行绘制。
* 方法参数说明：clipRect(int left, int top, int right, int bottom)

```
canvas.clipRect(0, 0, 20, imgs[i].getHeight());
```

* 设置完成后，我们来绘制大王这张牌。

```
canvas.drawBitmap(imgs[0],0,0,paint);
```
* 再增加循环，快速绘制所有的牌。

```
for (int i = 0; i < imgs.length; i++) {
    canvas.clipRect(i * 20, 0, (i + 1) * 20, imgs[i].getHeight());
    canvas.drawBitmap(imgs[i],i*20,0,paint);
}
```

但是运行后发现还是只显示大王一张图片，和上图一样。我们需要借助save和restore来完成裁剪的操作。

* save：用来保存Canvas的状态。save之后，可以调用Canvas的平移、放缩、旋转、错切、裁剪等操作。
* restore：用来恢复Canvas之前保存的状态。防止save后对Canvas执行的操作对后续的绘制有影响。
* save和restore要配对使用(restore可以比save少，但不能多)，如果restore调用次数比save多，会引发Error。save和restore之间，往往夹杂的是对Canvas的特殊操作。

代码修改如下：

```
for (int i = 0; i < imgs.length; i++) {
    canvas.save();
    canvas.clipRect(i * 20, 0, (i + 1) * 20, imgs[i].getHeight());
    canvas.drawBitmap(imgs[i],i*20,0,paint);
    canvas.restore();
}
```

![自定义绘制-4](http://a3.qpic.cn/psb?/V14YlNrL2eQEkW/h11.KUSgDhEvTTg0N99*2WHt7Ih0WzqqdBLi4jEy570!/b/dAEBAAAAAAAA&bo=QgEAAkIBAAIDACU!&rf=viewer_4)

可是还是发现最后一张还不是我们想要的效果。需要修改代码：

```
for (int i = 0; i < imgs.length; i++) {
    canvas.save();
    if(i<imgs.length-1) {
        canvas.clipRect(i * 20, 0, (i + 1) * 20, imgs[i].getHeight());
    }else if(i==imgs.length-1){
        canvas.clipRect(i * 20, 0, i * 20+imgs[i].getWidth(), imgs[i].getHeight());
    }
    canvas.drawBitmap(imgs[i],i*20,0,paint);
    canvas.restore();
}
```
也可以：

```
for (int i = 0; i < imgs.length; i++) {
    canvas.save();
    canvas.clipRect(i * 20, 0, i== imgs.lenth - 1 ? i * 20+imgs[i].getWidth():(i + 1) * 20, imgs[i].getHeight());
    canvas.drawBitmap(imgs[i],i*20,0,paint);
    canvas.restore();
}
```

#### Hierarchy Viewer(层级查看器)工具使用

* Hierarchy Viewer可以很直接的呈现布局的层次关系，视图组件的各种属性。 我们可以通过红，黄，绿三种不同的颜色来区分布局的Measure，Layout，Executive的相对性能表现如何。
打开工具

![Hierarchy Viewer-1](http://a2.qpic.cn/psb?/V14YlNrL2eQEkW/FwQpyp*WEYcO1VMNtvDy8lBe995qQZQ.fHkfGDxuwso!/b/dPcAAAAAAAAA&bo=iwM5AosDOQIDByI!&rf=viewer_4)

![Hierarchy Viewer-2](http://a2.qpic.cn/psb?/V14YlNrL2eQEkW/k2GqZCdU1R3uL*sr7As2z16fC1Kfp5ck2.vMpVDfwfM!/b/dB4BAAAAAAAA&bo=jQOiAo0DogIDACU!&rf=viewer_4)

查看各个节点Measure，Layout，Executive。

![Hierarchy Viewer-3](http://a2.qpic.cn/psb?/V14YlNrL2eQEkW/k2VodAvQNWB9z8*jOyC9ta5Sy7HHaXRi3HeJN6oYFz0!/b/dDQAAAAAAAAA&bo=LQOaAi0DmgIDACU!&rf=viewer_4)

* 三个小圆点, 依次表示Measure, Layout, Draw, 可以理解为对应View的onMeasure, onLayout, onDraw三个方法。
    * 绿色, 表示该View的此项性能比该View Tree中超过50%的View都要快。
    * 黄色, 表示该View的此项性能比该View Tree中超过50%的View都要慢。
    * 红色, 表示该View的此项性能是View Tree中最慢的。

一般来说：
* Measure红点, 可能是布局中嵌套RelativeLayout,或是嵌套LinearLayout都使用了weight属性.
* Layout红点, 可能是布局层级太深.
* Draw红点, 可能是自定义View的绘制有问题, 复杂计算等.


**常规做法**
* 没有用的父布局——没有背景绘制或没有大小限制的父布局，不会对界面效果产生任何影响。特别是** < include/> **进来的布局，很容易产生问题。可以通过** < merge /> **标签替代。
* 在布局层次一样的情况下，建议使用LinearLayout代替RelativeLayout。
* 使用LinearLayout导致的层次变深，可以使用RelativeLayout进行替换。同样的界面我们可以使用不同的方式去实现，选择一个层级最少的方案。
* 不常用的UI被设置成了GONE，尝试使用**< ViewStub />**代替。[附上一ViewStub使用](http://www.jianshu.com/p/5f64bacbd759)
* 去掉多余的背景颜色，减少过渡绘制，对于有多层背景色的布局来说，留最上面的一层即可。谨慎使用alpha，如果后渲染的元素有设置alpha值，那么这个元素就会和屏幕上已经渲染好的元素做blend处理，这样会导致不少性能问题，特别是出现在列表的Item中。
* 对于使用Selector当背景的布局，可以将normal状态的color设置为透明。
* 我们不能因为提高性能而忽略了界面需要达到的效果（平衡Design与Performance）。



### 第二步：代码问题查找

常见问题：我们重点关注Performance和Xml中的一些建议
* 在绘制时实例化对象（onDraw）
* 手机不能进入休眠状态（Wake lock）
* 资源忘记回收
* Handler使用不当倒置内存泄漏
* 没有使用SparseArray代替HashMap
* 未被使用的资源
* 布局中无用的参数
* 可优化布局（如：ImageView与TextView的组合是否可以使用TextView独立完成）
* 效率低下的weight
* 无用的命名空间等

**Lint工具使用**

Android Studio中开启Lint工具,选中需要分析的Module，点击工具栏中Analyze中的Inspect Code选项。

![Lint-1](http://b271.photo.store.qq.com/psb?/V14YlNrL2eQEkW/Zwx7cUKgzaf7c.HxQPNd0PmBqWpPcyxHzkgVr7o2K.Y!/b/dA8BAAAAAAAA&bo=gQLpAYEC6QEDByI!&rf=viewer_4)


选择需要分析的Module或整个项目


![Lint-2](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/pXALppPqtGPBPUFn0*bn5zQNvDDuI7frWuo6DPjKBhg!/b/dPkAAAAAAAAA&bo=jwFwAY8BcAEDACU!&rf=viewer_4)


我们可以逐一阅读一下，但是重点关注性能问题，xml中的一些问题也尽可能进行修复。

![Lint-3](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/jHd44oYmHQyC64DA.wsLnnu2LVeMRbAR8IsY5EgfZjU!/b/dBQBAAAAAAAA&bo=vAPWALwD1gADACU!&rf=viewer_4)


**问题处理**

* 案例中性能问题处理

![LintQuestion-1](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/j70H6Ik4a*hEL8oiT6yiuXtLq2Aor3wj9yLq8AGJ7Gw!/b/dMkAAAAAAAAA&bo=6QFJAOkBSQADACU!&rf=viewer_4)

其他的一些性能问题

![LintQuestion-2](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/Xv4KhqpJAyvpzlcLZgsfZ7DjSW6tx1UyMffgQE118p4!/b/dMkAAAAAAAAA&bo=ZgJ4AGYCeAADACU!&rf=viewer_4)

***

* 案例中xml提到的内容如下

![LintQuestion-3](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/DLm0ziaah.kqtciq9spSdV2eRUQTlqL2Xp69jZfArZY!/b/dBoBAAAAAAAA&bo=ZwFiAGcBYgADACU!&rf=viewer_4)

其他问题：
* 无效的命名空间

![LintQuestion-4](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/N87L5mYg1VeJ3UiBgo2loMcelpa1M0P50xGvCuzlOUw!/b/dMkAAAAAAAAA&bo=rwGgAK8BoAADACU!&rf=viewer_4)

* 无效的布局参数
比如在线性布局中的控件使用到了相对布局中的属性，运行时需要处理，影响代码的执行效率。

***
* 案例中关于定义声明变量的警告

![LintQuestion-5](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/WhaV9POiwwyi.QN*VQ6G4VXhPZUmrvbv1tAgir6zLgw!/b/dLEAAAAAAAAA&bo=cwFzAHMBcwADACU!&rf=viewer_4)



意见或建议:
* 不断关注Lint中提到的问题，将公司中命名规范中没有提到的内容逐一补全。
* Lint不是万能的。
* 可使用阿里推出的插件：Alibaba Java Coding Guidelines。



### 第三步：优化App的逻辑层

常见问题：主线程耗时大的函数、滑动过程中的CPU工作问题，工具可以提供每个函数的耗时和调用次数，我们重点关注两种类型的函数：
* 主线程里占用CUP时间很长的函数，特别关注IO操作（文件IO、网络IO、数据库操作等）。
* 主线程调用次数多的函数。

**使用Traceview找出卡住主线程的地方**

Traceview工具使用

通过Android Studio打开里面的Android Device Monitor，切换到DDMS窗口，点击左边栏上面想要跟踪的进程，再点击上面的Start Method Profiling的按钮，如下图所示：

![Traceview-1](http://a2.qpic.cn/psb?/V14YlNrL2eQEkW/fOQ1mtYLUlLqoH3rkNdbBaIUr.EelzJnWN6Hjkg2sts!/b/dPcAAAAAAAAA&bo=OQPXATkD1wEDByI!&rf=viewer_4)

启动跟踪之后，再操控app，做一些你想要跟踪的事件，例如滑动RecyclerView，点击某些视图进入另外一个页面等等。操作完之后，回到Android Device Monitor，再次点击相同的按钮停止跟踪。此时工具会为刚才的操作生成TraceView的详细视图。

![Tranceview-2](http://a1.qpic.cn/psb?/V14YlNrL2eQEkW/2jQgAjwFaxdLcHA1YDb0jhDXS2p0F9Qz1CrKwW1d208!/b/dHUAAAAAAAAA&bo=qwWAAtIFkQIDAE0!&rf=viewer_4)

重点关注Incl Cpu Time、Call+Recur Calls/Total、Real Time/Call
通过降序排序，我们可以分别找到这两列中数值比较大的内容。

指标说明:

* Incl(Inclusive) Cpu Time：方法本身和其调用的所有子方法占用CPU时间。
* Excl(Exclusive) Cpu Time：方法本身占用CPU时间。
* Incl Real Time：方法(包含子方法)开始到结束用时。
* Excl Real Time：方法本身开始到结束用时。
* Call + Recursion Calls/Total：方法被调用次数 + 方法被递归调用次数。
* Cpu Time/Call：方法调用一次占用CPU时间. 方法实际执行时间(不包括io等待时间)。
* Real Time/Call：方法调用一次实际执行时间. 方法开始结束时间差(包括等待时间)。

小案例：
我们可以在ViewHolder的设置数据中做点手脚，比如睡几毫秒（8ms），通过监控滚动，我们是否可以定位到问题代码。

**好的做法**

* 不要阻塞UI线程，占用CUP较多的工作尽可能放在子线程中执行。
* 需要结合使用场景选择不同的线程处理方案
	AsyncTask: 为UI线程与工作线程之间进行快速的切换提供一种简单便捷的机制。适用于当下立即需要启动，但是异步执行的生命周期短暂的使用场景。
    HandlerThread: 为某些回调方法或者等待某些任务的执行设置一个专属的线程，并提供线程任务的调度机制。
    ThreadPool: 把任务分解成不同的单元，分发到各个不同的线程上，进行同时并发处理。
    IntentService: 适合于执行由UI触发的后台Service任务，并可以把后台任务执行的情况通过一定的机制反馈给UI。
* 如果大量操作数据库数据时建议使用批处理操作。如：批量添加数据。

























