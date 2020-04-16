---
title: Android智能盯盘绘制View
date: 2020-04-16 18:27:39
tags: Android绘制UI
---

​    智能盯盘主要涉及两个自绘View，一个圆环，一个仪表盘，都是带圆的，用到的关键方法都是画布的Canvas.drawArc()方法。自绘View无非就是继承View复写View一些方法，得到我们自己想要的View，那你真的了解View的生命周期吗？我们自绘View用的最多的一个方法就是复写onDraw(Canvas canvas)方法，利用强大的Canvas以及画笔Paint来搞事。

- ##### **View的关键生命周期流程图**

![alt](/Ouyang/images/pan/1.png)

1. 首先是构造方法，Contructor

2. onFinishInflate，这个布局通过LayoutInflater进行填充的时候会走到这个方法。

3. onAttachedToWindow,当前View跟它对应的Window已经绑定。

4. onMeasure，开始测量，调用时间，当控件的父元素放置该控件时候，用于告诉父元素该控件的大小，父元素问该控件需要多大空间，该控件复写onMeasure(int widthMeasureSpec, int heightMeasureSpec)方法，调用setMeasuredDimension(int w, int h)告诉父元素具体大小。该控件怎么通过父元素的widthMeasureSpec, heightMeasureSpec告诉父元素具体大小呢？强大的MeasureSepc类，封装了父布局传递给子布局的布局要求，每个MeasureSpec代表了一组宽度和高度的要求，MeasureSpec由size和mode组成。Size可以通过 MeasureSpec. getSize(int measureSpec)拿到，mode通过MeasureSpec.getMode(int measureSpec)拿到，mode主要有以下三种类型：
   1）MeasureSpec.EXACTLY
   表示当layout_为match_parent或者具体的值时，mode代表具体精确的值。
   2）MeasureSpec.AT_MOST
   表示当layout_为wrap_content，能获得的最大尺寸，一般我们都会在自绘View设置一个默认最小宽度的minSize，我们通过取最小值。
   3）MeasureSpec.UNSPECIFIED
   表示当无法确定尺寸的时候，这个时候会用默认的最小值。一般代码都是如下这样设置：
   ![alt](/Ouyang/images/pan/2.png)
   ![alt](/Ouyang/images/pan/3.png)

5. onSizeChanged, 测量之后会回调这个方法，当尺寸变化后会调用，一般都是第一次测量后会调用，后面再测量，若尺寸没有变化就不会再去调用，我们获取的宽高基本在这个方法获取。
   ![alt](/Ouyang/images/pan/4.png)

6. onLayout，测量的时候进行布局，确定子View的位置。

7. onDraw, 确定完宽高和位置，利用强大的画布Canvas和画笔Paint绘制我们所需要的View。

8. onDetachedFromWindow，退出当前Activity后会走到这个方法。


   我们通过Log打印也可以看到View的大概生命周期流程如下：
   ![alt](/Ouyang/images/pan/5.png)
   一般我们自绘制view复写onMeasure, onSizeChanged, onDraw 这三个方法就可以了。

   

- ##### 圆环自绘View

  我们主要看下onDraw方法是怎么画的。

1. 首先我们画之前，需要将数据源设置进来，我们创建了个这样的数据model，包含数值，颜色；当然以后需要扩展，比如需要画标签，线条啥的，可以往这个model里面添加对应的属性进来。
   ![alt](/Ouyang/images/pan/6.png)

2. 在onDraw 方法里面，为了我们坐标的更好的对称换算，将默认的坐标系原点平移到当前画布的中心位置。
   ![alt](/Ouyang/images/pan/7.png)

3. 算出传入进来的所有数据model的总数值 total.

4. 循环画每一段圆环。我们用到的函数就是画布的drawArc方法，canvas.drawArc(RectF oval, float startAngle, float sweepAngle, Boolean useCenter, Paint paint)；oval代表圆环的外切矩形，startAngle代表画圆环的起始位置角度，sweepAngle代表当前圆环扫过的角度，sweepAngle大于0表示顺时针扫，小于0表示逆时针扫；useCenter表示是否包含中心原点连线，如果为true，画出的圆环将会是个衔接原点的扇形饼状，所以我们如果画饼状图，只要调成true就可以了；false就表示画圆环了。Paint表示画笔。
   1）根据当前数值和总数值total，先算当前要画的这段圆环占整个圆环的角度，即sweepAngle. 
   2）我们用了个变量startAngle来控制每一段圆环的起始位置角度，默认从0度开始画圆环，当然可以根据实际需要调整。大概代码如下，很简单。
   ![alt](/Ouyang/images/pan/8.png)

5. 可以看我画的草图如下：
   ![alt](/Ouyang/images/pan/9.png)

6. 注意，圆环真正的半径是mRadius – mPaint.getStrokeWidth()*0.5f，即大圆的半径减去画笔粗度的一半。参数rectF也为这个圆环半径的外切矩形。

7. 测试结果：
   ![alt](/Ouyang/images/pan/10.png)

   

8. 如果需要往每段圆环上画折线以及标签，可以在每一段循环画圆环后补充相应的代码，有兴趣的可以去看下被注释掉的“功能拓展部分”这段代码，里面涉及到一些数学的三角函数的坐标换算。在分支smart_stare类TransactionActivityRingChartView里面可以了解。
   测试结果：
   ![alt](/Ouyang/images/pan/11.png)

   

- ##### 仪表盘自绘View

1. 首先将坐标系原点移动到距离原来中心3/4个距离的地方，因为底部要画标签，所以这里空出点距离给标签，canvas.translate(mWidth / 2, mHeight *3 / 4)。
   ![alt](/Ouyang/images/pan/12.png)

2. 画半圆弧。原理类似。这里由于视觉上半圆弧有渐变色，需要给画笔设置渐变色shader, 三种色值分别位于九点钟、三点钟和12点钟方向，对应的postions为{0.5f, 0.75f, 1f}，代码如下：
   ![alt](/Ouyang/images/pan/13.png)
   ![alt](/Ouyang/images/pan/14.png)

3. 画圆弧刻度线，采用画布旋转的方法，结合画布的保存与恢复方法可以很方便的画出来。视觉稿刻度间距总共分为60等分，在0，10，20，30，40，50，60对画笔以及刻度长度值做一些特殊的属性值设置。
   ![alt](/Ouyang/images/pan/15.png)

4. 画文字标签，这里给大家普及下很key的一个参数，baseLine, 即基准Y坐标。canvas.drawText(String text, float x, float y, Paint paint), x指的是文字的起始坐标，这里的y指的是文字标签基准线的baseLine坐标.
   ![alt](/Ouyang/images/pan/16.png)
   如上图标注，通常用FontMetrics的top, bottom变量就可以算出基准线的坐标。top是相对于BaseLine偏移量，因为它在基准线上面，所以为负数，bottom是相对于BaseLine偏移量，为正数。通常我们计算文字的高度就是bottom – top;
   ![alt](/Ouyang/images/pan/17.png)
   baseLine的坐标 = 1/2 * getFontHeight() + getHeight() / 2 – FontMetrics.bottom.
   画文字标签的主要代码如下：
   ![alt](/Ouyang/images/pan/18.png)

   

5. 画指针，通过canvas.drawBitmap(Bitmap bitmap, Rect src, Rect dst, Paint paint)；bitmap为图片资源对象，src表示对图片进行裁剪，若为null表示显示整个图片;dst是图片在Canvas画布中显示的区域，src会被自动缩放/平移以适应它。我们通过画布的保存，恢复以及旋转方法，并用变量mStartAngle控制坐标系的旋转角度，默认mStartAngle = 0,即坐标系不旋转，从90度开始画，而且这个变量是通过动画类ValueAnimator.getAnimatedValue实时拿到的，这样就可以实时看到指针在旋转动起来。
   1）首先我们要将drawable图片资源转换成bitmap
   ![alt](/Ouyang/images/pan/19.png)
   2）通过画布的保存，画布的旋转，画布的恢复方法画指针
   ![alt](/Ouyang/images/pan/20.png)
   草图大概如下：
   ![alt](/Ouyang/images/pan/21.png)

6. 通过动画类ValueAnimator执行动画，我们根据外部传入的数值算出终止角度mTotalAngle，传入 ValueAnimator.ofFloat(mStartAngle, mTotalAngle);通过动画接口回调，函数getAnimatedValue拿到实时值mStartAngle，然后重新绘，得到我们想要的动画。代码如下：
   ![alt](/Ouyang/images/pan/22.png)

7. 测试结果：
   ![alt](/Ouyang/images/pan/23.png)