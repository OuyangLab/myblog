---
title: 在onCreate里面显示PopWindow会发生什么?
date: 2020-05-28 14:50:07
categories: Android
tags: Android开发
---

#### 1.在onCreate()方法里面调用PopupWindow的showAsDropDown或者showAtLocation会发生什么？

​       我们来实践下，在onCreate里面写入如下代码

```java
   @Override
   protected void onCreate(Bundle savedInstanceState)
{
   super.onCreate(savedInstanceState);

   setContentView(R.layout.activity_main);
   Toolbar toolbar = findViewById(R.id.toolbar);
   setSupportActionBar(toolbar);

   mTextView = findViewById(R.id.hello);
   FloatingActionButton fab = findViewById(R.id.fab);
   fab.setOnClickListener(new View.OnClickListener()
   {
      @Override
      public void onClick(View view)
      {
         Snackbar.make(view, "Replace with your own action", Snackbar.LENGTH_LONG)
               .setAction("Action", null).show();
      }
   });

   mPopupWindow = new PopupWindow(this.getBaseContext());
   mPopupWindow.setContentView(new View(this.getBaseContext()));
   mPopupWindow.showAtLocation(mTextView, 0, 0, 0);
}
```

运行跑下出现如下异常，wtf

![alt](/Ouyang/images/popWindow/1.png)

这是什么鬼？why？

#### 2.为啥会发生这个异常？

我们断点进去一步一步顺藤摸瓜，最终在ViewRootImpl的setView方法里面找到这个异常

![alt](/Ouyang/images/popWindow/2.png)

![alt](/Ouyang/images/popWindow/3.png)

也就是说res这个鬼东西命中了ADD_BAD_SUBWINDOW_TOKEN，那这个res是从哪里来的呢？关键代码如下

```java
  res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
```

它调用了Session里面的一个addToDisplay方法，最终会走到WMS的addWindow方法

```java
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
        Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
            outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
}
```

我们来看下wms这个addWindow方法是干嘛的，贴代码如下：

```java
    public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
        int[] appOp = new int[1];
//  权限检测，有兴趣的可以自己点进去看看
        int res = mPolicy.checkAddPermission(attrs, appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }

        ...
        synchronized(mWindowMap) { 
        /** final WindowState win = new WindowState(this, session, client,    token, parentWindow,
             mWindowMap.put(client.asBinder(), win);
              win = mWindow = new W(this);
             public ViewRootImpl(Context context, Display display) {
                ...
                mWindow = new W(this);
                ...
             }
            
        */
          appOp[0], seq, attrs, viewVisibility, session.mUid,
          session.mCanAddInternalSystemWindow);
          
            //异常1
            if (mWindowMap.containsKey(client.asBinder())) {
                Slog.w(TAG_WM, "Window " + client + " is already added");
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }

            /**
              window类型type:
              type表示Window的类型，Window有三种类型，分别是应用Window，子
              Window和系统Window。

              应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window中，比如Dialog就是一个子Window。系统Window
              是需要声明权限才能创建的Window，比如Toast和系统状态栏这些都是系统Window。

              Window是分层的，每个Window都有对应的z-ordered，层级大的会覆盖    在层级小的Window上。在三类Window中，应用Window的层级范围是1~99，子
              Window的层级范围是1000~1999，系统Window的层级范围是2000~2999。很显然系统Window的层级是最大的，而且系统层级有很多值，一
              般我们可以选用TYPE_SYSTEM_ERROR或者TYPE_SYSTEM_OVERLAY，另外重要的是要记得在清单文件中声明权限。

            */
            if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                parentWindow = windowForClientLocked(null, attrs.token, false);
                if (parentWindow == null) {
                    Slog.w(TAG_WM, "Attempted to add window with token that is not a window: "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
                //子Window
                if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                        && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add window with token that is a sub-window: "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
            }

            if (type == TYPE_PRIVATE_PRESENTATION && !displayContent.isPrivate()) {
                Slog.w(TAG_WM, "Attempted to add private presentation window to a non-private display.  Aborting.");
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }

           ...

            if (token == null) {
               //应用Window
                if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                
               
                if (rootType == TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
          
               ....

                if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with unknown token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }

                if (type == TYPE_TOAST) {
                    // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                    if (doesAddToastWindowRequireToken(attrs.packageName, callingUid,
                            parentWindow)) {
                        Slog.w(TAG_WM, "Attempted to add a toast window with unknown token "
                                + attrs.token + ".  Aborting.");
                        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                    }
                }
             ...
            if (type == TYPE_TOAST) {
                if (!getDefaultDisplayContentLocked().canAddToastWindowForUid(callingUid)) {
                    Slog.w(TAG_WM, "Adding more than one toast window for UID at a time.");
                    return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                }
                ...
            }

            // From now on, no exceptions or errors allowed!

            res = WindowManagerGlobal.ADD_OKAY;
            if (mCurrentFocus == null) {
                mWinAddedSinceNullFocus.add(win);
            }
            ...

            win.attach();
            mWindowMap.put(client.asBinder(), win);
           ...

            boolean imMayMove = true;

            win.mToken.addWindow(win);
            ...

        return res;
    }
```

在上面的addWindow方法里面我们看到当type在1000-1999时候，如果parentWindow为空会返回WindowManagerGlobal的ADD_BAD_SUBWINDOW_TOKEN, 那为啥PopWindow的type是在这个范围呢？我们查看PopWindow源码的createPopupLayoutParams方法的时候发现其赋值type的地方

```java
protected final WindowManager.LayoutParams createPopupLayoutParams(IBinder token) {
    final WindowManager.LayoutParams p = new WindowManager.LayoutParams();

    // These gravity settings put the view at the top left corner of the
    // screen. The view is then positioned to the appropriate location by
    // setting the x and y offsets to match the anchor's bottom-left
    // corner.
    p.gravity = computeGravity();
    p.flags = computeFlags(p.flags);
    p.type = mWindowLayoutType; // 就是这个变量！！
    p.token = token;
    p.softInputMode = mSoftInputMode;
    p.windowAnimations = computeAnimationResource();

    if (mBackground != null) {
        p.format = mBackground.getOpacity();
    } else {
        p.format = PixelFormat.TRANSLUCENT;
    }
  ....
}
```

我们看到里面有个全局变量mWindowLayoutType，这个初始值为

```java
private int mWindowLayoutType = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL;
```

```java
public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW = 1000;
```

所以看到PopWindow的type就是1000。那为什么parentWindow获取空呢？我们继续跟踪这个方法

```java
parentWindow = windowForClientLocked(null, attrs.token, false);
```

```java
final WindowState windowForClientLocked(Session session, IBinder client, boolean throwOnError) {
    WindowState win = mWindowMap.get(client);
    if (localLOGV) Slog.v(TAG_WM, "Looking up client " + client + ": " + win);
    if (win == null) {
        if (throwOnError) {
            throw new IllegalArgumentException(
                    "Requested window " + client + " does not exist");
        }
        Slog.w(TAG_WM, "Failed looking up window callers=" + Debug.getCallers(3));
        return null;
    }
    if (session != null && win.mSession != session) {
        if (throwOnError) {
            throw new IllegalArgumentException("Requested window " + client + " is in session "
                    + win.mSession + ", not " + session);
        }
        Slog.w(TAG_WM, "Failed looking up window callers=" + Debug.getCallers(3));
        return null;
    }

    return win;
}
```

上面代码传入的attr.token为空了 导致获取win直接为空，最后直接返回空了命中！那attr.token为啥为空呢？

还记得我们showAtLocation吗，里面有个token传参，间接导致PopWindow里面的WindowManager.LayoutParams中的token参数为空，最后直接命中！原来一切根源都是这个外部传参token为空！那最后来追究为啥在onCreate()方法里面 这个token为啥空了。

#### 3.根源mWindowToken为空

```java
public void showAtLocation(View parent, int gravity, int x, int y) {
    mParentRootView = new WeakReference<>(parent.getRootView());
    showAtLocation(parent.getWindowToken(), gravity, x, y); // 此处的parent.getWindowToken()为空了
}
```

断点查看到parent的View的getWindowToken方法里面的mAttachInfo为空了，那我们再继续查询mAttachInfo为啥空了，mAttachInfo在哪里赋值的？我们全局搜索下在View的dispatchAttachedToWindow方法里面赋值的，原来这个mAttachInfo是在ViewRootImpl初始化的时候赋值的

```java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    mWindowSession = WindowManagerGlobal.getWindowSession();
    mDisplay = display;
    .......
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this, context); // 看到没，就是这儿！！
    .....
}
```

我们知道ViewRootImpl是在WindowManagerGlobal的addView时候添加进来的，而addView是在ActivityThread的handleResumeActivity方法添加的。

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);     // performResumeActivity会走到Activity的onResume方法
    if (r == null) {
        // We didn't actually resume the activity, so skipping any follow-up actions.
        return;
    }

    final Activity a = r.activity;
    .........
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l); // 就在此处添加View
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }
    .........
    .........
}
```

我们看到addView其实在performResumeActivity后才添加进来的，你在Activity的onCreate里面当然不会addView了。这下终于水落石出了！

最后总结下：

当我们在onCreate里面显示PopWindow的时候，由于还没把顶层View添加进来导致ViewRootImpl还没创建，最后mAttachInfo也为空，导致最终的mWindowToken也空了，最终引发我们崩溃的血案！

