---
title: Android面试题（一）——Activity的生命周期和启动模式
date: 2016-07-01 09:04:38
tags: android
---
## 引言
***
  - 这份面试题系列文章旨在查漏补缺，通过常见的面试题发现自己在Android基础知识上的遗漏和欠缺，验证所学是否扎实。
  - 这是系列的第一章，后面我会根据安卓知识模块分类并网罗分析各种常见面试题。
<!--more-->
## 面试题：
***

  1. ### Activity的生命周期
答：onCreate->onStart->onResume->Activity运行->新的Activity运行->onPause->onStop->onDestroy->Activity销毁
![Activity生命周期的切换过程.png](http://upload-images.jianshu.io/upload_images/1944615-97050739d9be01dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  2. ### Activity的启动方式
答：四种启动模式，standard, singleTask, singleTop, singleInstance。 
    - **standard**：标准模式，在当前的任务栈上创建新的Activity，不论之前有没有创建过该Activity。注意：ApplicationContext无法启动standard模式的Activity。
    - **singleTask**：栈内复用模式，分两种情况，第一种情况：如果有任务栈里已经创建了该Acitiviy，直接销毁该Acitivity栈上面的所有Acitivity，无须新创建一个Activity；第二种情况：如果没有任务栈里已经创建该Activity，创建一个新的任务栈并在新栈上创建新Activity。注意：该模式下复用Activity，系统会调用Activity的onNewIntent方法。

    - **singleTop**：栈顶复用模式，如果该Activity在任务栈栈顶，即当前活动的Acitivty就是要创建的Activity，那么不会创建新的Activity。注意：该模式下复用Activity，系统会调用Activity的onNewIntent方法。

    - **singleInstance**：单实例模式，加强版的singleTask，当每次都直接创建一个新的任务栈，再在该新栈上创建新Activity。注意：singleInstance永远是单栈单Activity

  3. ### onSaveInstanceState和onRestoreInstanceState调用的过程和时机
    - **调用时机**：Activity的异常情况下（例如转动屏幕或者被系统回收的情况下），会调用到onSaveInstanceState和onRestoreInstanceState。其他情况不会触发这个过程。但是按Home键或者启动新Activity仍然会单独触发onSaveInstanceState的调用。
    - **调用过程**：旧的Activity要被销毁时，由于是**异常情况**下的，所以除了正常调用onPause, onStop, onDestroy方法外，还会在**调用onStop方法前，调用onSaveInstanceState方法**。新的Activity重建时，我们就可以通过onRestoreInstanceState方法取出之前保存的数据并恢复，**onRestoreInstanceState的调用时机在onCreate之后**。
![Activity的异常情况下的生命周期.png](http://upload-images.jianshu.io/upload_images/1944615-f0a751d7a50bec9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  4. ### Activity A调用Activity B时，A调用什么函数？
    - 旧的Activity的onPause方法执行结束，才执行新的Activity的onCreat, onState和onResume方法。
    - onPause方法中不能有重量级的任务，不然会影响新Activity的创建。

  5. ### onNewIntent的作用和调用时机？
    - **调用时机**：如果Activity的启动模式是：singleTop, singleTask, singleInstance，在复用这些Acitivity时就会在调用onStart方法前调用onNewIntent方法
    - **作用**：让已经创建的Activity处理新的Intent。

  6. ### fragment的生命周期
![Fragment的生命周期.png](http://upload-images.jianshu.io/upload_images/1944615-75a6a50f39fc3ea2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  7. ### fagement和Activity的通信
有五种方法
    - Fragment可以通过getActivity()访问Activity实例，并轻松地执行在Activity布局中查找视图等任务。
```
View listView = getActivity().findViewById(R.id.list);
```
    - Activity可以通过FragmentManager的findFragmentById()或findFragmentByTag()，获取对Fragment的引用。
```
ExampleFragment fragment = (ExampleFragment) getFragmentManager().findFragmentById(R.id.example_fragment);
```

    - 创建对Activity的事件回调
      - 在fragment中定义一个回调接口，并要求宿主Activity实现它。当Activity通过该接口收到回调时，可以根据需要与布局中的其他片段共享这些信息。
      - 代码示例：
```
    public static class FragmentA extends ListFragment {
        ...
        // Container Activity must implement this interface
        public interface OnArticleSelectedListener {
            public void onArticleSelected(Uri articleUri);
        }
        ...
        @Override
        public void onAttach(Activity activity) {
            super.onAttach(activity);
            try {
                mListener = (OnArticleSelectedListener) activity;
            } catch (ClassCastException e) {
                throw new ClassCastException(activity.toString() + " must implement OnArticleSelectedListener");
            }
        }
        @Override
        public void onListItemClick(ListView l, View v, int position, long id) {
            // Append the clicked item's row ID with the content provider Uri
            Uri noteUri = ContentUris.withAppendedId(ArticleColumns.CONTENT_URI, id);
            // Send the event and Uri to the host activity
            mListener.onArticleSelected(noteUri);
        }
    }
```
    - 向操作栏添加项目
    - 通过Fragment.setArguments向Fragment传递参数
```
public void setArguments(Bundle args)
final pubilc Bundle getArguments()
```

## 知识点
***
  1. Activity典型情况下的生命周期分析
  2. Activity异常情况下的生命周期分析
  3. Activity的启动模式
  4. Fragment的生命周期
  5. Activity和Fragment的通信方法

## 参考资料：
***
[fagement官方文档](https://developer.android.com/guide/components/fragments.html)
《Android开发艺术探究》
《Android开发完全讲义》
