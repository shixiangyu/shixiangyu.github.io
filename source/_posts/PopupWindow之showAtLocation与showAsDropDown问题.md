---
title: PopupWindow之showAtLocation与showAsDropDown问题
date: 2017-11-05 18:20:53
categories: android
tags:
   - PopupWindow
---

该文接着上次的PopupWindow之setFocusable与setOutsideTouchable问题, 来说一下它的另外两个方法，分别是 `showAsDropDown(View anchor, int xoff, int yoff)`和` showAtLocation(View parent, int gravity, int x, int y)`

这两个都是我们用来show出popupwindowd的方法，但是策略有所不同。

<!--more-->

#### showAsDropDown(View anchor, int xoff, int yoff)方法
这个方法没有什么好说的，在锚点anchor的正下方弹出popup，参数xoff和yoff分别是x轴和y轴的偏移量，偏移量是相对锚点anchor来说的，以anchor的左下角为参考点；

但是，需要注意的是，Android 7.0版本之前，在指定位置弹出popupwindow用showAsDropDown(View anchor, int xoff, int yoff)毫无问题，但在android 7.0上，用showAsDropDown（）就需要注意下面两点了：

- 如果指定 PopupWindow 的高度为 MATCH_PARENT，调用 showAsDropDown(View anchor) 时，在 7.0 之前，会在锚点 anchor 下边缘到屏幕底部之间显示 PopupWindow；而在 7.0、7.1 系统上的 PopupWindow 会占据整个屏幕（除状态栏之外）。

- 如果指定 PopupWindow 的高度为自定义的值height，调用 showAsDropDown(View anchor)时， 如果 height > 锚点 anchor 下边缘与屏幕底部的距离， 则还是会出现7.0、7.1上显示异常的问题；否则，不会出现该问题。

所以在无法避免上述的两个问题时，这时候就需要用showAtLocation（）来处理可能出现的popup显示异常问题。

#### showAtLocation(View parent, int gravity, int x, int y)

对showAtLocation()方法的很多解释都是：“相对于父控件的位置，x和y为偏移量。”这种解释绝对是错的离谱，而且很可笑，如果是这样，那它和showAsDropDown（）方法还有什么异同，瞪大眼睛看看这个方法的参数，x不再是xoff,y也不再是yoff，从字面上来看也不是什么所谓的偏移量吧，首先，这个方法是对于整个window的屏幕以坐标来定位置的，不存在相对于某一个view，有相对也是相对整个屏幕，参数x和y是坐标。

该方法的第一个参数是parent，类型是个View，这个参数名很让人误解，其实，并不是把PopupWindow放到这个parent里，并不要求这个parent是一个ViewGroup。官方文档对这个参数的解释是“[a parent view to get the token from](https://developer.android.com/reference/android/widget/PopupWindow.html#showAtLocation(android.view.View,int,int,int))”，这个parent的作用应该是调用其getWindowToken()方法获取窗口的Token，所以，只要是该窗口上的控件就可以了。我个人开发中，一般是拿该parent在屏幕中的坐标拿来作为参考的，通过`parent.getLocationInWindow(location)`获得parent在屏幕上的坐标，其中location是一个大小为2的int数组。

第二个参数很让人理解为对齐方式，当然不是，而是通过设置gravity来设置坐标原点：

- Gravity.TOP | Gravity.LEFT   以屏幕左上角为坐标原点
- Gravity.BOTTOM | Gravity.RIGHT  以屏幕右下角为坐标原点
- Gravity.LEFT  以屏幕左侧，屏幕高度 1/2 处为坐标原点

以此类推，可根据实际情况是设置坐标原点的位置。

设置了第二个参数，后面的参数x,y也就是相对于坐标原点的位置了。