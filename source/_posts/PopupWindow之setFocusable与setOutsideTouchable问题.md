---
title: PopupWindow之setFocusable与setOutsideTouchable问题
date: 2017-11-05 18:03:51
categories: android
tags:
   - PopupWindow
---

这是我最近遇到的问题，由于之前对PopupWindow使用不熟悉，理解不透彻导致的，所以现在对这些方法进行了尽量的深入解析，另外总结就是不能只管去拷贝复制别人的代码，然后发现不符合自己的要求了，就一个一个去改着试试，这样尽管最后可能达到了要求，但却不知其所以然，后续还极有可能突然就蹦出来个bug。

* 是否正确理解`setFocusable(boolean focusable)`和`setOutsideTouchable(boolean touchable)`？
<!--more-->

要彻底弄清这个问题，就要分别查看android5.0以下和5.0以上（含5.0）的源码，因为android系统创建popup的时候在5.0以上和以上会有不同的处理，5.0以下的系统在创建popup的时候会根据你是否通过调用popup的`setBackgroundDrawable(Drawable background)`方法来判断是否把你的popup放到一个PopupViewContainer里面，5.0以下的源码如下(对无关代码做了省略）：

    private void preparePopup(WindowManager.LayoutParams p) {
       ......
        if (mBackground != null) {
            PopupViewContainer popupViewContainer = new PopupViewContainer(mContext);
            PopupViewContainer.LayoutParams listParams = new PopupViewContainer.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT, height
            );
            popupViewContainer.setBackgroundDrawable(mBackground);
            popupViewContainer.addView(mContentView, listParams);   //重点，只有mBackground != null时才会执行

            mPopupView = popupViewContainer;
        } else {
            mPopupView = mContentView;
        }
    }

而5.0以上（含5.0）的代码是这样的：

    private void preparePopup(WindowManager.LayoutParams p) {
        .......
        if (mBackground != null) {
            mBackgroundView = createBackgroundView(mContentView);
            mBackgroundView.setBackground(mBackground);
        } else {
            mBackgroundView = mContentView;
        }

        mDecorView = createDecorView(mBackgroundView);  //重点，和mBackground无关，都会执行

    }

5.0以上系统的PopupDecorView就是5.0以下的PopupViewContainer，它们是同一个东西，这个containerView继承自FrameLlayout，它对onTouchEvent方法进行了重写：

    private class PopupDecorView extends FrameLayout {
        ......
        @Override
        public boolean onTouchEvent(MotionEvent event) {
            final int x = (int) event.getX();
            final int y = (int) event.getY();

            if ((event.getAction() == MotionEvent.ACTION_DOWN)
                    && ((x < 0) || (x >= getWidth()) || (y < 0) || (y >= getHeight()))) {
                dismiss();
                return true;
            } else if (event.getAction() == MotionEvent.ACTION_OUTSIDE) {
                dismiss();
                return true;
            } else {
                return super.onTouchEvent(event);
            }

    }

看到这，恍然大悟，在5.0以下的系统，当你没有setBackgroundDrawable时，此时的popup是完全没有处理你的屏幕点击触摸事件的能力的，包括你的物理返回按键也一样不会响应，这时候不管你去如何设置setOutsideTouchable都是没有意义的，根本不会进入到onTouchEvent方法里面，也就走不到`if (event.getAction() == MotionEvent.ACTION_OUTSIDE)`这个逻辑判断了。

这个MotionEvent.ACTION_OUTSIDE很熟悉，就是通过这个action来判断是否响应外侧点击事件的，那么它是怎么触发的呢？

当把 `MotionEvent.ACTION_OUTSIDE`与`setOutsideTouchable(boolean touchable)`放到一起，就会很明了了。


#### 分析setOutsideTouchable方法：

先看看setOutsideTouchable(boolean touchable)的源码：

    public void setOutsideTouchable(boolean touchable) {  
       mOutsideTouchable = touchable;  
    }  

然后再看看mOutsideTouchable哪里会用到

    private int computeFlags(int curFlags) {  
        curFlags &= ~(WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH);  
        …………  
        if (mOutsideTouchable) {  
            curFlags |= WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;  
        }  
        …………  
        return curFlags;  
    }  

这段代码主要是用各种变量来设置window所使用的flag；

既然说到了FLAG_WATCH_OUTSIDE_TOUCH，那我们来看看FLAG_WATCH_OUTSIDE_TOUCH所代表的意义：

![WechatIMG1606.jpeg](http://upload-images.jianshu.io/upload_images/5878491-2473f2f61c065a13.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这段话的意思是说，如果窗体设置了FLAG_WATCH_OUTSIDE_TOUCH这个flag，那么 用户点击窗体以外的位置时，将会在窗体的MotionEvent中收到MotionEvetn.ACTION_OUTSIDE这个事件。

所以由于在PopupViewContainer中添加了对MotionEvent.ACTION_OUTSIDE事件的捕捉，在当用户点击PopupViewContainer以外的区域时，PopupWindow这个窗体就会收到ACTION_OUTSIDE事件，然后调用dismiss方法来关闭掉自己。

读到这，基本对setOutsideTouchable方法的来龙去脉了解的差不多了。

####分析setFocusable方法:

再看setFocusable方法，一般资料对此的解释都是：是否让Popupwindow获得焦点，这确实揭示了本质，但却很抽象，不能从方法调用的级别来解释该方法的作用。其实，这个方法的触发以及接收处理和setOutsideTouchable方法原理是一样的：

    public void setFocusable(boolean focusable) {
        mFocusable = focusable;
    }

接下来设置它的flag

    if (!mFocusable) {
            curFlags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
            if (mInputMethodMode == INPUT_METHOD_NEEDED) {
                curFlags |= WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM;
            }
    }

这个flag比较牛逼，它能接收屏幕的 `ACTION_UP，ACTION_DOWN`等Touch事件，所以当设置了setFoucus为true时，因为PopupViewContainer对MotionEvent.ACTION_DOWN事件进行了捕捉，然后同样会调用自己的dissmiss方法来关闭自己。

现在也就很好明白为什么在设置了setFoucus(true)后再去设置        setOutsideTouchable(false)为什么没有作用了，因为MotionEvent.ACTION_DOWN事件在MotionEvent.ACTION_OUTSIDE事件之前处理，消耗了MotionEvent事件。[这样也就能解释官方对setOutsideTouchable方法的说明了](https://developer.android.com/reference/android/widget/PopupWindow.html#setOutsideTouchable(boolean))

![WechatIMG1613.jpeg](http://upload-images.jianshu.io/upload_images/5878491-bfae4e1f7040c864.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
"控制是否通知popup窗体外的点击事件，这个方法只有在touchable为true而focusable为false的时候才有意义", 说的很严谨，是有没有意义而不是有没有作用，