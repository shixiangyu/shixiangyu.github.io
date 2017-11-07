---
title: Android集成华为推送踩坑问题总结
date: 2017-11-03 18:39:48
categories: android
tags:
   - android 消息推送
---

恩,这个需求是cto老板提的, 所以不管华为push的集成过程有多扯淡多贱(华为push论坛说的,不是我ma的), 也得遇山开山, 逢水搭桥......

先总结遇到的几大坑位:

   1. 使用老版push还是新版push
   2. PushReceiver中的onEvent()回调触发问题
   3. APP接收到推送后,点击消息,总是会先打开启动页

<!--more-->
### 1. 先说新老push的问题

{% asset_img hupush.jpeg tupian %}

话说华为开发者平台提供了新老两个版本的Push SDK, 老版本已经不再维护, 而且也很扯淡的把接入文档下架了, 新的名字叫HMS Push,论坛上有人ma华为说它新的没整利索,就把老的给撤掉,什么东西,我就不说什么了, 刚开始的时候考虑到新版sdk需要依赖HMS服务以及覆盖率问题, 我们就集成了老版本的push, 华为文档有这么一句话"HMS Push的现有接口兼容对应的老版本的Push接口",于是客服端和服务端都开始开发了,结果悲剧了, 是的, 客服端兼容了, 但服务端不行了, 新老版本接口参数都不一样, 两端必须同样的版本,这点它的文档中没有丝毫说明...后来就开始网上找服务端接入文档, 成功接入后, 却发现又碰到了老push不稳定的问题......

最终的最终,决定重新集成新版本的push!!!


### 2.onEvent回调的问题

对消息的监听, 提供了4种回调方法,注释写得很清楚, 官网也有相关的解释, 不多说. 这里说一下onEvent方法,当时试了很多次没试出这个回调如何触发(可能是自己粗心大意)

{% asset_img onevent.jpeg tupian %}

后台发送push时必须在自定义内容中加入键值对后才会在客户端点击该条消息后回调onEvenet方法, 另外华为push是不能感知到消息的到达的, 这点需要区分开.


### 3.当APP接收到推送后,点击消息,会发现总是会先打开启动页的问题

这个问题是最坑的,不管你的app是否活着,收到push点击后都会重启一次你的app,这样的体验肯定很有问题,必须解决.同样,华为文档没有任何提示或者说明,我们想着可能需要通过自定义动作才能解决这个问题了,最后事实证明是这样的.于是接着查文档,恩,是的,找到不少自定义行为方面的说明文档, [通知栏消息格式说明,这是链接](http://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush.html?page=hmssdk_huaweipush_devguide_s#2.5%20Push通知栏消息格式说明),个人觉得说的很乱,而且这些都是自己服务端那边发push要使用的message格式,那么客服端代码怎么写呢,总之,比小米push文档差远了,当初接Mi push也没这么折腾人

先来看看自己服务端接入华为push需要的消息内容格式:

{% asset_img servergeshi.jpeg tupian %}

图里红框中的需要正确设置, 第一个type为3表示系统通知栏消息, 第二个是action的type标识, 1表示自定义行为,最重要的就是那个intent了, 至于这个intent怎么来的, 又是怎么用的, 一会详细说明.......


恩,开始客户端工作:

不要指望在华为的push文档中找到客户端自定义行为的任何代码示例, 至少目前没有, 还好, 华为还是给了你一点指示的, 进入华为发送push的页面:


{% asset_img client.jpeg tupian %}

瞅了半天,不知道有多少人看不懂,反正开始我没看明白.
   
1)  先在AndroidManifest文件中添加一个需要跳转到的Activity, 并加入intent-filter过滤器:    

       <activity
            android:name=".activity.HWPushTranslateActivity"
            android:theme="@style/Activity.Translucent">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <data
                    android:host="com.wonderfull.android.push"
                    android:path="/notification"
                    android:scheme="wonderfullpush" />
            </intent-filter>
       </activity> 
       
这个Activity是用来中转的,可以看到我给它的主题设置的是Translucent透明的, <data> 里面是需要我们自己来根据自己的项目情况来配置的, host可配置为自己项目的包名,path 的配置随意,但不要少了前面的"/", scheme的配置也是没有什么要求,但一会会用到这三个参数,只要保证前后一致便可,最后我们会在这个activity中根据服务端传递的参数进行其他页面的跳转,之所以这样做,是因为我们不知道服务端希望我们进入哪个页面, 你不可能在每个可能会跳转的activity中都配置一个过滤器吧, 这不可能, 所以我们收到自定义push, 我们统一都进入这个中转页面,之后在它里面进行其他跳转;



2) 这时候客服端其实已经有了跳转到这个被指定的activity的功能了, 下一步就是服务端如何发送push了, 参数不对,照样无法跳转,这个时候主要就看上面提到的这个 intent 了, 可以看到华为自己的推送后台自定义动作一栏,提示说填入执行的动作intent, 那这个intent怎么填,格式是什么, 是需要跳转的那个Activity的名字或者整个包名路径吗,当然不是! 上面这个图中下面的代码示例就是在这个要跳转的Activity在AndroidManifest中的配置, 上面部分说的就是告诉你如何去生成这个intnet, 这个intent就是一个固定格式的字符串, 需要你先通过那段代码生成, 当然你也可以直接照着固定格式自己拼凑出来, 但不建议, 很容易出错. 有了这个intentUri之后把它填给华为推送后台, 或者直接给你们自己的后台, 我们可以在这个intentUri中可以添加自定义消息, 之后在你的中转Activity中拿到并做处理就行了.

intent的格式基本类似下面这个格式,是通过生成代码生成的

`intent://com.wonderfull.android.push/notification?action=$action#Intent;scheme=wonderfullpush;action=android.intent.action.VIEW;launchFlags=0x10000000;end
`

"com.wonderfull.android.push" 是上面配置的中转Activity的host, "/notification" 是path的值, ?号后面就是可配置的选项参数了, 从 #Intent开始,scheme也是你配置的scheme值,后面的就是固定值了

3) 接下来就是在中转activity里根据参数进行跳转处理了,下面的代码是我们项目中的处理,很简单,没有什么复杂逻辑:

       try {
            String action = getIntent().getData().getQueryParameter("action");
            ActionUtil.startAction(this, ActionUtil.PREFIX + action, true);
        } catch (Exception e){
            e.printStackTrace();
        }
        finish();
        
取出执行下一步动作的action, 然后通过action进行相应的跳转处理, 最后直接关闭这个透明的中转Activity就行了.

到此, 已经差不多了

总结出来感觉并不多, 但操作的过程还是很折腾的.......