---
title: 仪表盘自绘View
date: 2020-05-19 15:50:06
index_img: /images/yibiao/19.png
categories: Android
tags: Android绘制UI
---

主要介绍里面一些实现功能点的方法。

首先这个View主要涉及这几个技能点。

canvas几个技能点

- 画布中心点平移
- 画布旋转
- 画布状态保存与恢复
- 画布画圆弧

画笔Paint几个技能点

- 画文字
- 画线

渐变色

- SweepGradient

动画

- ValueAnimator

…………………………………………………………………………………………………………………………….......

1. 画布中心点平移 canvas.translate(float dx, float dy), 我们知道画布中心点（0, 0）默认在左上角, 我们把中心点通过canvas.translate(mWidth / 2, mHeight)平移到画布中心，这样更好的算坐标。这样中心点就到半圆弧中心处。
   ![alt](/Ouyang/images/yibiao/1.png)
2.  画布旋转canvas.rotate(float a), 这里的a值表示度数，a > 0表示以画布中心顺时针旋转a度，反之，a < 0表示逆时针旋转a度；canvas.rotate(float a, float px, float py),以中心坐标点（px, py）进行旋转a度。
3.  canvas.save 将画布之前的状态先保存起来，之后画布可以进行任意操作，代码进行到canvas.restore后又可以恢复到画布原始的状态。
4. 利用canvas.drawArc(RectF oval, float startAngle, float sweepAngle, Boolean useCenter, Paint paint) 画圆弧，参数oval为以圆弧中心点外切圆的矩形，startAngle为扫描的起始角度，大于0表示顺时针旋转startAngle，小于0逆时针旋转，这个参数的作用是设置圆弧是从哪个角度来顺时针绘画的。sweepAngle为扫过的角度。useCenter
   是否经过圆中心点。
5. 画文字，canvas.drawText(String text, float x, float baseLineY, Paint paint)
6. 画线，canvas.drawLine(float startX, float startY, float stopX, float stopY, Paint paint)
7.  渐变色效果采用SweepGradient
8.  动画ValueAnimator

#####   **画外层圆弧**

​          首先我们把坐标中心移动到半圆弧中心

![alt](/Ouyang/images/yibiao/2.png)

![alt](/Ouyang/images/yibiao/3.png)          

 圆弧真正的半径其实是radius,我们可以通过看下方的草图得知：

![alt](/Ouyang/images/yibiao/4.png)  

   这里腾出的偏移量mOffset是为了放刻度值显示的，所以半径radius等于mRadius减去mOffset再减去画笔粗度的一半。rectFArc 是以半径radius外切矩形，扫描的起始角度从180度开始，顺时针扫描180就可以得到半圆弧了。

##### **画内层圆弧**

​         同理一样，画内层圆弧的不同点就是半径不一样   ![alt](/Ouyang/images/yibiao/5.png)  

#####  画进度圆弧

![alt](/Ouyang/images/yibiao/6.png)      

​      在这个函数，我们用一个变量mCurrentAngle代表当前进度角度，也就是扫过的角度，这个值怎么来的呢？后面会说明。

##### 进度圆弧渐变色

​        做这个点卡了下，也是个稍微难点的地方，其实都知道就是给画笔设置Shader,但尝试了很多方法最后就是出不来效果，后来做出来的时候其实代码很简单。

​    首先，我们来了解下渐变色的知识，Android支持三种颜色渐变，LinearGradient（线性渐变） RadialGradient （径向渐变） SweepGradient（扫描渐变）。这三种渐变继承自android.graphics.Shader， Paint 类通过setShader支持渐变。

 （1）线性渐变LinearGradient

![alt](/Ouyang/images/yibiao/7.png)  

![alt](/Ouyang/images/yibiao/8.png)  

1.  float x0, float y0, float x1, float y1 定义了起始点的和结束点的坐标。
2.  int[] colors 颜色数组，在线性方向上渐变的颜色。
3.  float[] positions 和上边的数组对应，取值[0..1]，表示每种颜色的在线性方向  上所占的百分比。可以为Null,为Null 是均匀分布。
4.  Shader.TileMode tile 表示绘制完成，还有剩余空间的话的绘制模式。

  第二个构造函数参数：

1. 1. float x0, float y0, float x1, float y1 定义了起始点的和结束点的坐标。
   2. int color0, int color1, 开始颜色和结束的颜色
   3. Shader.TileMode tile 表示绘制完成，还有剩余空间的话的绘制模式。  

 (2) 径向渐变RadialGradient

   圆环的渐变，有两个构造函数    ![alt](/Ouyang/images/yibiao/9.png)  

1. float centerX, float centerY 渐变的中心点 圆心
2. float radius 渐变的半径
3. int[] colors 渐变颜色数组
4. float[] stops 和颜色数组对应， 每种颜色在渐变方向上所占的百分比取值[0, 1]
5. Shader.TileMode tileMode 表示绘制完成，还有剩余空间的话的绘制模式。

​    ![alt](/Ouyang/images/yibiao/10.png)  

1.  float centerX, float centerY 渐变的中心点 圆心
2.  float radius 渐变的半径
3.  int centerColor, int edgeColor 中心点颜色和边缘颜色
4.  Shader.TileMode tileMode 表示绘制完成，还有剩余空间的话的绘制模式

 (3) 扫描渐变SweepGradient

  扫描渐变是和角度有关的渐变。以某一点为圆心，随着角度的大小发生渐变。

![alt](/Ouyang/images/yibiao/11.png)  

1. float cx, float cy 中心点坐标
2. int[] colors 颜色数组
3. float[] positions 数组颜色在渐变方向上所占的百分比

![alt](/Ouyang/images/yibiao/12.png)  

  1.float cx, float cy 中心点坐标

  2.int color0, int color1 开始颜色 结束颜色

   我们仪表盘的渐变色采用的就是扫描渐变，起初我采用的是第二个构造函数，以为传入渐变的开始颜色值和结束的颜色值就行，后来发现达不到预期效果。原因应该是扫描的位置不对，SweepGradient这个扫描默认是三点钟方向开始顺时针旋转360，但是我通过进入矩阵matirx进行旋转它的扫描起始位置还是没达到预期效果，原因未知。后来采用引入参数数组positions的构造方法。具体代码如下：

![alt](/Ouyang/images/yibiao/13.png)  

​    positions的取值范围[0, 1]，由于步进圆弧终点位置不停的变化，这里我们采用传入数组postions参数进行设置，每一个position对应colors数组中每个颜色在360度中的相对位置，position取值范围为[0,1]，0和1都表示3点钟位置，0.25表示6点钟位置，0.5表示9点钟位置，0.75表示12点钟位置，这样根据当前扫过的角度mCurrentAngle算出终点position, 数组colors的长度必须和positions长度一致！！我们从180度开始，也就是9点钟方向。所以当前位置currentPosition = (mCurrentAngle + 180) / 360; 这样数组postions就为{0.5f, currentPosition}。

##### **画刻度线及刻度值**

​       怎样画刻度线且都是均匀显示的呢？可能普通的做法就是寻找每个刻度线的坐标去画线，刚开始也想这样去计算某个刻度线坐标，但这样太复杂了。我们采用画布的保存、画布的旋转以及画布的恢复方法，再用画笔的画线方法就可以做到刻度线均匀分布。包括画指定的刻度值也是一样，如果用三角函数去算其坐标也比较复杂。而且绘制的时候会出现很难对准刻度线。

截取部分代码如下： ![alt](/Ouyang/images/yibiao/14.png)  

![alt](/Ouyang/images/yibiao/15.png)  

每一次循环画刻度线的时候，将画布先保存起来，这样在下一次循环的时候画布又重新回到最初的状态。100格，顺时针旋转的角度为 （float）180/100 * i ;注意这里角度为浮点型，之前做的时候没注意到导致画的刻度不够 =-=；在i = 0，20，40，60，80，100 需要画刻度值。  

​    这里画刻度值采用画布中心点平移以及旋转的方法来做。我们以画20为例子。如下图准备画刻度值的时候，我们的画布由于在画刻度线的时候已经顺时针旋转到坐标系（x2, y2）了，这个时候将y2平移offset偏移量到y3，此时我们的坐标系变成(x2, y3),中心点从A移动到B了，再逆时针旋转angle角度，坐标系变成（x3, y­4）了，这样我们在坐标系(x3, y4)上画刻度值20，我们采用画笔的getTextBounds方法获取文字边框，在边框里面绘制20，之后再按原路的动作反方向返回，这样我们又回到刚开始的坐标系(x2, y2)。我们将这一系列的动作写进一个drawScaleNumber方法。  

![alt](/Ouyang/images/yibiao/16.png)      

![alt](/Ouyang/images/yibiao/17.png)  

##### **动画**

​           仪表盘的动画是怎么实现的呢，这里通过属性动画ValueAnimator来实现。通过改变属性值来创造动画。

![alt](/Ouyang/images/yibiao/18.png)  

  我们创造了两个ValueAnimator对象，参数mTotalAngle, mMaxNum都是通过公有方法外部传入，第一个ValueAnimator对象的作用是用来做进度圆弧动画的，第二个ValueAnimator对象是用来做中间百分比数字值渐变。首先我们创建一个ofFloat(mCurrentAngle, mTotalAngle)对象，这个属性值范围就是从mCurrentAngle到mTotalAngle，mCurrentAngle默认为0; 通过设置一个时间插值器同时设置监听addUpdateListener(new ValueAnimator.AnimatorUpdateListener())就可以拿到在这个时间插值器里从mCurrentAngle递增到mTotalAngle之中的值。拿到mCurrentAngle后重新绘制UI，这样就达到我们的动画效果。前面介绍的画进度圆弧用的mCurrentAngle就是从这里拿到的。同理，做百分比数字递增动画也是这样做的。

​       最后我们实现出来的的效果图

![alt](/Ouyang/images/yibiao/19.png)  

 