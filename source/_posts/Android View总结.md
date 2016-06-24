title: Android View总结
date: 2015-12-17 15:13:06
tags: ["Android"]
---
## 关于Android View控件

Android中控件大致被分为两类ViewGroup,View。ViewGroup作为容器管理View。Android视图，是类似于Dom树的架构。父视图负责测量定位绘制等操作。我们经常在用的`findViewById` 方法代价昂贵的原因，就是因为他负责至上而下遍历整棵控件树，来寻找View实例，在重复操作中尽量少用。现在在用的很多控件都是直接或者间接继承自View的，如下图。

![view 继承树](http://ww2.sinaimg.cn/mw690/87dacc16gw1ez2rbxd8lsj20mw0d9wfy.jpg)
<!--more-->

## Android UI界面架构

每个Activity包含一个`PhoneWindow`对象，`PhoneWindow`设置`DecorView`为应用窗口的根视图。在里面就是熟悉的`TitleView`和`ContentView`,没错，平时使用的`setContentView()`就是设置的`ContentView`。

![UI 架构](http://hujiaweibujidao.github.io/images/androidheros_ui.png)

## Android是如何绘制View的？

当一个Activity启动时，会被要求绘制出它的布局。Android框架会处理这个请求，当然前提是Activity提供了合理的布局。绘制从根视图开始，从上至下遍历整棵视图树，每一个`ViewGroup`负责让自己的子`View`被绘制，每一个`View`负责绘制自己，通过`draw()`方法.绘制过程分三步走。

*   Measure
*   Layout
*   Draw

整个绘制流程是在`ViewRoot`中的`performTraversals()`方法展开的。部分源代码如下。

```java
private void performTraversals() {
    ......
    //最外层的根视图的widthMeasureSpec和heightMeasureSpec由来
    //lp.width和lp.height在创建ViewGroup实例时等于MATCH_PARENT
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    ......
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ......
    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
    ......
    mView.draw(canvas);
    ......
}
```
在绘制之前当然要知道view的尺寸和绘制。所以先进行`measu`和`layout`（测量和定位），如下图。

![绘制流程](http://hi.csdn.net/attachment/201112/29/0_132516479540V4.gif)

## Measure过程
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {  
    //....  
  
    //回调onMeasure()方法    
    onMeasure(widthMeasureSpec, heightMeasureSpec);  
     
    //more  
}
```
计算view的实际大小，获得高宽存入`mMeasuredHeight`和`mMeasureWidth`，`measure(int, int)`传入的两个参数。`MeasureSpec`是一个32位int值，高2位为测量的模式，低30位为测量的大小。测量的模式可以分为以下三种。

*   EXACTLY
精确值模式，当`layout_width`或`layout_height`指定为具体数值，或者为`match_parent`时，系统使用EXACTLY。

*   AT_MOST
最大值模式，指定为`wrap_content`时，控件的尺寸不能超过父控件允许的最大尺寸。

*   UNSPECIFIED
不指定测量模式，View想多大就多大，一般不太使用。

根据上面的源码可知，measure方法不可被重写，自定义时需要重写的是`onMeasure`方法
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
查看源码可知最终的高宽是调用`setMeasuredDimension()`设定的,如果不重写，默认是直接调用`getDefaultSize`获取尺寸的。

使用View的getMeasuredWidth()和getMeasuredHeight()方法来获取View测量的宽高，必须保证这两个方法在onMeasure流程之后被调用才能返回有效值。

## Layout过程

Layout方法就是用来确定view布局的位置，就好像你知道了一件东西的大小以后，总要知道位置才能画上去。

`mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());`

layout获取四个参数，左，上，右，下坐标，相对于父视图而言。这里可以看到，使用了刚刚测量的宽和高。
```java
public void layout(int l, int t, int r, int b) {
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {
        .....
        onLayout(changed, l, t, r, b);
        .....
}
```
通过`setFrame`设置坐标。如果坐标改变过了，则重新进行定位。如果是View对象，那么onLayout是个空方法。因为定位是由ViewGroup确定的。

当layout结束以后`getWidth()`与`getHeight()`才会返回正确的值。

这里出现一个问题，`getWidth/Height()` and `getMeasuredWidth/Height()`有什么区别？  

*   `getWidth()`:View在設定好佈局後整個View的寬度。
*   `getMeasuredWidth()`:對View上的內容進行測量後得到的View內容佔據的寬度

![getwidth](http://img.blog.csdn.net/20141106113343781?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3lfYno=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## Draw过程
```java
public void draw(Canvas canvas) {
        ......
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        ......
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        ......

        // Step 2, save the canvas' layers
        ......
            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }
        ......

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        ......
        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
        ......

        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
        ......
    }
```

重点是第三步调用onDraw方法。其它几步都是绘制一些边边角角的东西比如背景、scrollBar之类的。其中`dispatchDraw`，是用来递归调用子View,如果没有则不需要。

onDraw方法是需要自己实现的，因为每个控件绘制的内容不同。主要用canvas对象进行绘制，这里就不说了。

## 参考资料

1.  [Android视图绘制流程完全解析，带你一步步深入了解View(二)](http://blog.csdn.net/guolin_blog/article/details/16330267)
2.  [Android应用层View绘制流程与源码分析](http://blog.csdn.net/yanbober/article/details/46128379)
3.  [How Android Draws Views](http://developer.android.com/intl/zh-cn/guide/topics/ui/how-android-draws.html)
4.  [《Android群英传》](http://www.amazon.cn/Android%E7%BE%A4%E8%8B%B1%E4%BC%A0-%E5%BE%90%E5%AE%9C%E7%94%9F/dp/B01481RAA4/ref=sr_1_1?s=books&amp;ie=UTF8&amp;qid=1450359477&amp;sr=1-1&amp;keywords=android%E7%BE%A4%E8%8B%B1%E4%BC%A0)
5.  [What is the difference between getWidth/Height() and getMeasuredWidth/Height() in Android SDK?](http://stackoverflow.com/questions/8657540/what-is-the-difference-between-getwidth-height-and-getmeasuredwidth-height-i)
