---
layout: post
categories: [爬虫]
tags: [Python]
code: true
title:  Python抓取新浪微博配图
---

&emsp;&emsp;我女神是刘亦菲，看着女神微博，总觉得女神微博的哪一张照片都好看，于是想用脚本把她微博相册中的微博配图全部抓下来。  
&emsp;&emsp;一开始打开微博配图网页，alt+command+I打开chrome开发者工具，查看网页源代码；其后，我直接右键查看网页源代码，对比一下发现网页源代码和开发者工具下看见的不一样，其中应该是浏览器加载了js，开发者工具看到了更多。因为爬虫的话，首选移动端，于是我打开移动端网页查看，这里面两者就是一样的。但是移动版照片好小，但是通过和网页版的比较，发现图片的地址有一定的关系，地址栏uuid一样，这就好办多了。
&emsp;&emsp;分析完，剩下的就是编码实现了，虽然是Javaer，但是听说爬虫还是选择Python比较好，对网络这一块支持很好，有很多的包，于是一边看语法，一边写代码。下面是源代码，其中技术实现分为主要三点：  
&emsp;&emsp;1.登录。这里我直接cpoy浏览器的Cookie登录，直接copy浏览器的cookie字符串，没有模拟提交表单，省了不少代码，而且如果是模拟表单登录，过一段时间可能不适用了，因为新浪会修改接口，一大堆的JS真难看懂。其中把http请求User-Agent参数带上，否则403，因为新浪做了反爬虫处理。  
&emsp;&emsp;2.根据爬取移动版图片uuid拼凑出网页版大图片URL地址。其中移动版小图片没有logo，大图片的logo没有找到好的去除方式。
&emsp;&emsp;３.下载图片到本地。  
最后吐槽一句，当时爬下来的图片，仔细看了看，发现刘亦菲只为棒子宋承宪这一个男性单独配过图。没想到过几天就出来消息，刘亦菲和宋在一起了(大哭)。  
源码如下：

```
import urllib2　　　　　＃2.7版本
import re
import requests

#输入您的Cookie  在chrome浏览器请求网页时可以看到好长的字符串
headers = {'User-Agent':'Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.1.6) Gecko/20091201 Firefox/3.5.6',
'cookie': ''}   
count = 0
#微博相册页数，此处可以做优化，从１开始爬，直到没有。但是我为了按照时间顺序来爬，就简单粗暴地处理了。
page = 27　　　　　　
while page > 0:              
	#刘亦菲微博账号的uid:3261134763                   
	req = urllib2.Request('http://weibo.cn/album/albummblog/?rl=11&fuid=3261134763&page='+ str(page), headers=headers) 
	r = urllib2.urlopen(req)
	data = r.read()
	p = re.compile(r'src="http://ww(.).sinaimg.cn/square/(.{32}).jpg" alt=')
	uuids = p.findall(data)
	urls = []
	for uuid in uuids:
		url = 'http://ww' + uuid[0] + '.sinaimg.cn/mw1024/' + uuid[1] + '.jpg'
		urls.append(url)

	urls.reverse()
	for url in urls:
		response = requests.get(url)
		if response.status_code == 200:
			count += 1
			f = open("/home/chen/mytest/crystal/"+ str(count) +".jpg", 'wb')
			f.write(response.content)
			f.close()
	page -= 1

```