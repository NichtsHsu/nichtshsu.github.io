---
title: 关于我github page踩过的坑
date: 2023-11-05 14:50 +0800
categories: [通用]
tags: [网站,博客]     # TAG names should always be lowercase
---

## 前言
从我刚开始准备搭静态博客开始，花了将近完整的3天才把简单的基本内容和魔改内容搞完(我是小菜鸡），接下来给大家分享一下自己踩过的哪些坑和搭建github静态博客(Chripy主题)需要做的事情。

## 准备工作
首先你需要一个Github账号(什么？！你说你没有账号，那直接ctrl+w就行了)，Fork一个你自己喜欢的网页主题的仓库，将仓库id改为{username}.github.io(username为你自己的githubID)。再进入该仓库，点击`setting`选项，再点`Pages`，这时你就能看到`Build and deployment`下有`source`选项中有两个选项，选择`GitHub Actions`,再安装提示部署，commit一下workflow文件后，github就能自动帮你搭建一个博客了！浏览器url输入{username}.github.io(username为你自己的githubID)即可访问网站。

## 后续细节
- 如果你对github提供的域名不满意，可以同样在`page`页面下的`Custom domain`中输入你自己的域名（关于域名购买问题在此不多赘述），Save。然后在你的域名提供商页面，配置DNS，这里有个小分支：
1. 如果你不使用cdn加速，那就直接在域名提供商的配置页面下，对DNS进行配置：
> 添加一个CNAME记录，HostName为值www，Target HostName值为上面的{username}.github.io。
> 或者通过添加4条A记录，HostName值为www，Target HostName值分别为{username}.github.io对应的IP地址，在`cmd`输入
```cmd
ping {username}.github.io
```
即可返回类似如下信息:
```
正在 Ping xxx.github.io [xxx.xxx.xxx.xx] 具有 32 字节的数据:
来自 xxx.xxx.xxx.xx 的回复: 时间=184ms
来自 xxx.xxx.xxx.xx 的回复: 时间=183ms
来自 xxx.xxx.xxx.xx 的回复: 时间=183ms
来自 xxx.xxx.xxx.xx 的回复: 时间=177ms
```
其中方框中的一串数字即为{username}.github.io对应的ip。
2. 如果使用cdn加速，则需要先在域名提供商这里设置一下NameServers，将其指向cdn服务商，下面介绍使用Cloudflare进行cdn加速。
3. 因为github page访问速度慢的问题，网站经常一个图片都得加载半天，所以得用cdn加速服务。虽然Cloudflare经常被诟病速度慢，反向加速等问题(笑)，但因为方便和免费的原因，本文依然介绍使用cloudflare，如有需要可以尝试其他cdn。
首先进入[Cloudflare官网](https://dash.cloudflare.com/),注册登录后将你购买的域名绑定在Cloudflare上，然后关键的一步是：
> 将Cloudflare名称服务器替换原域名提供商的域名下的名称服务器(NameServers),通常来说有两条，类型为NS，值为x.x.cloudflare.com.

然后即可将上述步骤重复一遍：
> 添加一个CNAME记录，HostName为值www，Target HostName值为上面的{username}.github.io。添加到Cloudflare的DNS记录中。记得将代理状态开关打开，就是那个橙色的云。

这样后等待片刻即可发现自己的域名已经可以访问了。
- 如果想给网站添加一些小功能，比如评论功能，本文采用的waline评论系统，在本网站使用的Chirpy主题中使用方法如下：
1. 首先因为静态网站的原因，评论系统也是需要一个云部署和云存储的过程的，本网站采用的是[Vercel](https://vercel.com/)提供的免费部署和[leancloud](https://console.leancloud.app/)的云数据库。先登录官网[leancloud](https://console.leancloud.app/)，本文提供的是国际版的leancloud，进入后注册一个账号，登录后创建应用，名称随意。然后在`设置`中点击`应用凭证`,可以看到有3个Credentials：
`AppID`
`AppKey`
`MasterKey`
这都是接下来要用的，先别关闭leancloud页面。
2. 打开[Vercel](https://vercel.com/)网页后选择Github登录，然后创建项目，再在项目中点击`Settings`配置一下`Environment Variables`.  
Add 3个键值对(Key,Value),分别为:
> LEAN_ID: AppID
LEAN_KEY: AppKey
LEAN_MASTER_KEY: MasterKey

其中`AppId`,`AppKey`,`MasterKey`分别为步骤1中leancloud所给的信息.
3. 完成之后点击`Deployments`部署(`Redeploy`)一下项目,等待部署完成,会有一个app结尾的网站,即为部署后的服务端.但是很搞的是,点进去发现访问不了,刚开始我还以为是没部署成功,后来发现是正确的,只不过vercel被dns污染了,所以需要其他的步骤,接下来先在项目`Settings`中的Domains中添加一个域名,随便什么都行,最方便的是直接配置你购买的域名,比如说你买的域名是`asuka.com`,那你可以添加一个`waline.asuka.com`,vercel会提示你valid通过失败,然后按照他给你一段记录,填到Cloudflare的DNS解析中,一般来说是CNAME记录,名称为waline,内容为它提供给你的一段url.接下来回到vercel项目中,等待一会你会发现域名的验证通过了.
4. 这时候马上用访问你刚刚填入的域名(比如上面的`waline.asuka.com`),可能能看到评论框的内容了,但是当你多次访问后可能还是不行(如果一直都可以的话跳过这一步).这里我想了半天为什么,后来发现是cloudflare的加密问题,进入cloudflare中的`SSL/TLS`的`概述`,将`加密模式`的`灵活`改为`完全`,就发现可以一直访问你配置的域名服务器了.
5. 最后一步就是在你的网页`_config`中配置一下waline选项,按jerkll举例,进入`_config`文件后找到`comments:`,在`active:`后填入`"waline"`,然后在`waline:`的`server:`后输入刚刚自己创建的域名服务器地址,然后commit一下就可以重新构建博客了,过一会上去就可以看看你的评论功能上线了没有~

## 小吐槽
由于我个人也是第一次搭静态博客,这个经历也算是一波三折,不过总算是有点收获,在此和大家分享,希望大家不要踩我踩过的坑(有时候琢磨半天也想不出来真的很难受),之前研究Iconfont的时候半天找不到iconId在哪里,后来发现上传一份icon class就好了.