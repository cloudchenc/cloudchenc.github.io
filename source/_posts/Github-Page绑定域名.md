---
title: Github Page绑定域名
date: 2020-06-08 11:16:07
tags: 
    - Blog
    - Github Page
categories: Blog
---

###购买域名
默认的域名是 xxx.github.io, 如果github的名字不够响亮，就显得很没有排面。
所以研究了一番域名后缀，选择了 .me 后缀, 我是在Godaddy入坑的，第一年价格不高。
具体买什么后缀，这里不再赘述，全凭个人爱好。

###设置域名
打开Github博客项目，进入settings menu，下拉到Custom domain，输入你刚才购买的域名，点击保存即可。
![Custom domain](https://tva1.sinaimg.cn/large/d7f9b0f4gy1gfkwjy4eicj20k702zwep.jpg)

###解析域名
这里以Godaddy为例，由于Godaddy在国内的DNS解析不是很好，所以更换为了DNSPod

进入DNSPod网站，注册登陆，添加域名，点击解析按钮，添加两条解析记录 如下：

![DNSPod](https://tva3.sinaimg.cn/large/d7f9b0f4gy1gfkwjed3ehj211108vmyj.jpg)

第一个红框，你需要替换为自己的博客ip地址，可以通过ping github page域名来获取
![Github IP](https://tva3.sinaimg.cn/large/d7f9b0f4gy1gfkxg86t5jj20p805imyb.jpg)

第二个红框，你需要替换为自己的博客地址，如xxx.github.io

进入你刚才购买域名的平台，进入域名设置，打开DNS管理，将默认DNS服务器更换为 f1g1ns1.dnspod.net 和 f1g1ns2.dnspod.net
![Godaddy NS](https://tva2.sinaimg.cn/mw690/d7f9b0f4gy1gfkwkdxid5j20vu09smxr.jpg)

修改完域名服务器后，Godaddy DNS信息栏会显示
![Godaddy DNS](https://tva4.sinaimg.cn/mw690/d7f9b0f4gy1gfkwls9ck4j20vx05n0sz.jpg)
这是正常情况，无需担心

完成以上步骤，即可在浏览器输入之前购买的域名测试是否成功跳转

参考链接：
`https://cyclechen.me/2019/03/13/2019-03-13-setup-github-page-personal-with-godaddy/`