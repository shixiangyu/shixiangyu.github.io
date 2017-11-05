---
title: FragmentPagerAdapter刷新fragment无效问题
date: 2017-11-04 10:35:00
categories: android
tags:
   - fragment
---

前段时间在版本迭代里，一个activity中嵌套了多个fragment，当其中一个或多个数据发生变化时，通过pagerAdapter的notifyDataSetChanged方法去更新或者清空fragment集合重新给fragment设置数据并添加至集合时，最后结果都不能达到预期，数据还是原来的数据。

于是，我们一起来看看原因所在，看看FragmenPagerAdaptere的instantiateItem方法源码就明白了()：

<!--more-->

        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;

原来它会先去FragmentManager里面去查找有没有缓存该fragment，如果有就直接使用，如果没有才会触发fragmentpageadapter的getItem方法获取一个fragment。所以你更新的fragmentList集合是没有作用的，即使清除fragment集合也是没有用的，因为缓存依旧在，效果仍不达啊，所以我们需要去清除FragmentManager里面缓存的fragment，或者说替换了它，我们只需重写它的instantiateItem方法：

    public Object instantiateItem(ViewGroup container, int position) {
          //拿到缓存的fragment，如果没有缓存的，就新建一个，新建发生在fragment的第一次初始化时
            RecyclerFragment f = (RecyclerFragment) super.instantiateItem(container, position);
            String fragmentTag = f.getTag();
            if (f != getItem(position)) {
          //如果是新建的fragment，f 就和getItem(position)是同一个fragment，否则进入下面
                FragmentTransaction ft =mFm.beginTransaction();
                //移除旧的fragment
                ft.remove(f);
                //换成新的fragment
                f =getItem(position);
                //添加新fragment时必须用前面获得的tag
                ft.add(container.getId(), f, fragmentTag);
                ft.attach(f);
                ft.commitAllowingStateLoss();
            }
            return f;
        }



这样就可以达到想要的效果了.