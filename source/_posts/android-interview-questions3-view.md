---
title: Android面试题（三）——View的事件体系和工作原理
date: 2016-07-17 14:36:33
tags: android
---
## 引言
***
View在Android的地位堪比四大组件，Android为我们提供了很多的系统空间。但是为了区别一般性，我们往往需要自定义View，这就要求我们对View的事件体系和工作原理有深入的理解，只有这样才能做出完美的自定义控件
<!--more-->
## 面试题
***
  1. ### View中onTouch，onTouchEvent和onClick的执行顺序
onTouch->onTouchEvent->onClick
      1. 当一个View需要处理事件时，如果它设置了OnTouchListener，那么OnTouchListener的onTouch方法会被回调。
      2. 这时事件如何处理还得看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用；如果返回true，那么onTouchEvent方法将不会被调用。由此可见，给View设置的onTouchListener，其优先级比onTouchEvent要高。
      3. 如果当前方法中设置了onClickListener，那么它的onClick方法会被调用。可以看出，常用的OnClickListener，其优先级别最低。

  2. ### View的滑动方式
    - **三种**方式：
      a. **通过View本身提供的scrollTo/scrollBy方法**
移动的是View的内容，View本身不移动
![mScrollX和mScrollY的变换规律示意.png](http://upload-images.jianshu.io/upload_images/1944615-f0459909d2eee53f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
      b. **通过动画给View施加平移效果实现滑动**
移动的View的影像，View本身位置不发生改变。Android3.0以下，移动后的View点击事件发生的位置不会改变
      c. **通过改变View的LayoutParams使View重新布局实现滑动**
改变布局参数，代码如下：
```
MarginLayoutParams params = (MarginLayoutParams) mButton.getLayoutParams();
params.width += 10;
params.height += 10;
mButton.requestLayout();
//获知 mButton.setLayoutParams(params);
```
    - **三种方法的使用对比**
      - scrollTo/scrollBy：操作简单，适合对View内容的滑动；
      - 动画：操作简单，主要适合于没有交互的View和实现复杂的动画效果；
      - 改变布局参数：操作稍微复杂，适用于有交互的View。

  3. ### View的事件分发机制
事件的分发机制由**三个重要方法**来共同完成：**dispatchTouchEvent**、**onInterceptTouchEvent**和**onTouchEvent**
    - **事件分发**：public boolean dispatchTouchEvent(MotionEvent ev)
用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的DispatchTouchEvent方法的影响，表示是否消耗当前事件。
    - **事件拦截**：public boolean onInterceptTouchEvent(MotionEvent event)
在上述方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。
    - **事件响应**：public boolean onTouchEvent(MotionEvent event)
在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。
    - **三者的关系**可以总结为如下伪代码：
```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean consume = false;
        if (onInterceptTouchEvent(ev)) {
            consume = onTouchEvent(ev);
        } else {
            consume = child.dispatchTouchEvent(ev);
        }
        
        return consume;
    }
```
    - 事件传递机制的11个结论：
      1. 同一个事件序列是从手指触摸屏幕的那一刻起，到手指离开屏幕那一刻结束，这个过程中所产生的一系列事件。这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。
      2. 一个事件序列只能被一个View拦截且消耗，不过通过事件代理TouchDelegate，可以将onTouchEvent强行传递给其他View处理。
      3. 某个View一旦决定拦截，那么这一事件序列就都只能由它来处理
      4. 某个View一旦开始处理事件，如果不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么事件会重新交给它的父元素处理，即父元素的onTouchEvent会被调用。
      5. 如果View不消耗除ACTION_DOWN以外的事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会调用，并且当前View可以持续收到后续的事件，最终这些消失的事件会传递到Activity。
      6. ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouchEvent方法默认返回false。
      7. View没有onIntercepteTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
      8. View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable和longClickable同时为false）。View的longClickable默认都为false，clickable要分情况看，比如Button默认为true，TextView默认为false。
      9. View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true。
      10. onClick会发生的前提是当前View是可点击的，并且它受到down和up的事件。
      11. 事件传递是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法就可以在子元素中干扰父元素的事件分发过程，但ACTION_DOWN事件除外。

  4. ### View的绘制流程
    1. 三个过程
      - measure：**测量View的宽和高**
      - layout：**确定View在父控件中的放置位置**
      - draw：**负责将View绘制在屏幕上。**

    2. 几个常用回调方法
      - 构造方法
      - onAttachToWindow：在包含View的Activity启动时调用
      - onDetachFromWindow：在包含View的Activity退出或者View被remove时回调
      - onVisibilityChanged：当View的可见状态发生改变时调用
    3. 两个重要概念

      - ViewRoot：连接WindowManager(外界访问Window的入口)和DecorView（顶级View）的纽带，View的三大流程均是通过ViewRoot来完成的。
      - DecorView：顶级View
      
    4. View的绘制流程
 View的绘制流程是从ViewRoot的PerformTraversals方法开始的。
![performTraversals的工作流程图.png](http://upload-images.jianshu.io/upload_images/1944615-33ea92d34ecac092.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图所示：
      - performTraversals会依次调用**performMeasure, performLayout, performDraw**三个方法，这三个方法分别完成顶层View的measure,layout,draw方法，**onMeasure又会调用所有子元素的measure过程，直到完成整个View树的遍历。**同理，performLayout, performDraw的传递流程与performMeasure相似。**唯一不同**在于，performDraw的传递过程在draw方法中通过dispatchDraw实现，但没有本质区别。
      - Measure过程后可以调用getMeasureWidth和getMeasureHeight方法**获取View测量后的宽高**，**与getWidth和getHeight的区别**是：**getMeasuredHeight()返回的是原始测量高度，与屏幕无关**，**getHeight()返回的是在屏幕上显示的高度**。实际上在当屏幕可以包裹内容的时候，他们的值是相等的，只有当view超出屏幕后，才能看出他们的区别。当超出屏幕后，getMeasuredHeight()等于getHeight()加上屏幕之外没有显示的高度。
      - Layout过程确定View四个顶点的位置和实际的宽高。
      - Draw过程确定View的显示，只有draw方法完成后View的内容才会出现在屏幕上。

  5. ### MeasureSpec的使用
measureSpec的作用：**很大程度上决定了一个View的尺寸规格**
下面是它的一些常量和方法：
```
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * 精确模式，对应LayoutParams中的match_parent和具体数值这两种模式
         */
        public static final int EXACTLY = 1 << MODE_SHIFT;

        /**
         * 最大模式，大小不定，但是不能超过窗口的大小
         */
        public static final int AT_MOST = 2 << MODE_SHIFT;

        public static int makeMeasureSpec(int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }

        public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }

        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
```
**MeasureSpec和LayoutParams的关系**
View的MeasureSpec由父容器的MeasureSpec和自身的LayoutParams决定。
View的masure过程由ViewGroup传递，具体观察ViewGroup的measureChildWithMargins方法
DecorView的MeasureSpec由窗口尺寸和自身的LayoutParams决定。
子元素的MeasureSpec还和View的margin和padding有关。
具体情况如下图：
![普通View的MeasureSpec的创建规则.png](http://upload-images.jianshu.io/upload_images/1944615-d2ee1f7debd381be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  6. ### 如何让自定义View支持自定义属性
    1. 在values目录下创建自定义属性的XML，比如attrs.xml，format负责定义属性的格式，可以是“color”代表颜色，也可以是reference代表资源id，dimension代表尺寸。
```
	<?xml version="1.0" encoding="utf-8"?>
	<resources>
	    <declare-styleable name="CircleView">
		<attr name="circle_color" format="color" />
	    </declare-styleable>
	</resources>
```
    2. 在View的构造函数中解析自定义属性的值并做相应处理，解析circle_color这个属性的值：
```
    public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
            mColor = a.getColor(R.styleable.CircleView_circle_color, Color.RED);
            a.recycle();
            init();
    }
```
    3. 在布局文件中使用自定义属性，使用前需要在布局文件中添加schemas生命：`xmlns:app="http://schemas.android.com/apk/res-auto"`在这个声明中app是自定义属性的前缀，也可以换成其他名字，不过要与CircleView中的自定义属性一致。
```
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#ffffff"
            android:orientation="vertical" >

            <com.ryg.chapter_4.ui.CircleView
            android:id="@+id/circleView1"
            android:layout_width="wrap_content"
            android:layout_height="100dp"
            android:layout_margin="20dp"
            android:background="#000000"
            android:padding="20dp"
            app:circle_color="@color/light_green" />
    </LinearLayout>
```

  7. ### 重写View应该注意哪些方面
自定义View的分类大致可以分为4类： 
    1. 继承View重写onDraw方法 
    2. 继承ViewGroup派生特殊的Layout 
    3. 继承特定的View（比如TextView） 
    4. 继承特定的ViewGroup（比如LinearLayout）

    5. 自定义View须知：
      1. 让View支持wrap_content
      2. 如果有必要，让View支持padding
      3. 尽量不要在View中使用Handler，没必要，有post系列方法
      4. View中如果有线程或动画，需要及时停止，参考View#onDetachedFromWindow
      5. View带有滑动嵌套情形时，需要处理好滑动冲突

|        | 实用范围 | 注意事项|
|------|-------------|----------|
| 继承View重写onDraw方法 | 不规则效果 | 重写onDraw方法，需要自己支持wrap_content,处理padding |
| 继承ViewGroup派生特殊的Layou | 自定义布局 | 处理ViewGroup的测量、布局两个过程，子元素的测量和布局 |
| 继承特定的View（比如TextView） | 扩展已有的View的功能 | 不需要自己支持wrap_content、padding |
| 继承特定的ViewGroup（比如LinearLayout） | 自定义布局 | 不需要自己处理ViewGroup的测量、布局两个过程 |

## 推荐
***
[我的个人博客](yoxin.github.io)
[Android面试题（二）——IPC机制
](https://yoxin.github.io/2016/07/11/android-interview-questions2-IPC/)

## 知识点
  - View的事件分发机制
  - 滑动冲突解决方案
  - View的测量、布局以及绘制流程
  - View的常见回调方法
  - 自定义View实现的一般步骤

## 参考资料
《android开发艺术探究》
[官方文档Creating a View Class](https://developer.android.com/training/custom-views/create-view.html)
