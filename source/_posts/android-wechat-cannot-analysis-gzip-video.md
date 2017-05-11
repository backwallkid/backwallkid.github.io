---
title: 2017年05月10日记一次微项目投产 | 安卓版微信内置浏览器不能解析gzip压缩过的mp4视频的问题
date: 2017-05-11 12:30:00
categories:
- Tech
tags:
- PHP
- Apache
---
### 前言
今天投产了一个小项目，一个很简单的H5，有播放视频功能，使用了videojs插件。
之前也做过数个视频播放，视频的转压都按照既定流程进行，文件放到FTP后，iphone和安卓机测试下来都没有问题。
于是给链接，业务组直接在微信公众号里投放了。那个企业号有不少关注的人，推送发出去1分钟就有近千阅读量。
但是我在点击链接后，发现项目打不开了，而且该企业官网的主站也挂了，在经过pc端和手机4G下测试发现问题依然存在后，赶紧报bug给其他同事。
通过询问FTP管理员得知，那个“大”企业的网站带宽只有10M。假如只是普通H5的话，绰绰有余，但是现在里面有个13m的mp4视频，当然请求量集中飙升的时候，带宽瞬间耗尽。
经过多方协商，高层决定使用紧急方案，把视频和其他图片文件放到我们公司的一个cdn服务器上，修改H5内的资源链接，使之直接请求cdn，以缓解那个“大”企业网站带宽不足的问题。
前端同事在修改路径后，又发现了新的问题：
mp4视频在iphone手机的微信内置浏览器内能够正常播放，而测试用的安卓机均提示不能解析。
通过网络搜索，一开始以为是[视频格式的问题](https://segmentfault.com/q/1010000008880318)，向视频来源确认过后排除了。

### 用于测试的安卓机型
* 微信6.5.7 华为P10 PLUS Android 7.0 华为浏览器版本10.7.2.4038
微信内置浏览器user agent:
``` bash
Mozilla/5.0 (Linux; Android 7.0; VKY-AL00 Build/HUAWEIVKY-AL00; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/53.0.2785.49 Mobile MQQBrowser/6.2 TBS/043220 Safari/537.36 MicroMessenger/6.5.7.1041 NetType/WIFI Language/zh_CN
```
华为浏览器user agent: 
``` bash
Mozilla/5.0 (Linux; Android 7.0; VKY-AL00 Build/HUAWEIVKY-AL00) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30
```
* 微信6.5.7 三星Galaxy S6 edge+ Android 6.0.1 三星浏览器版本4.0.20-47
微信内置浏览器user agent: 
``` bash
Mozilla/5.0 (Linux; Android 6.0.1; SM-G9280 Build/MMB29K; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/53.0.2785.49 Mobile MQQBrowser/6.2 TBS/043220 Safari/537.36 MicroMessenger/6.5.7.1041 NetType/WIFI Language/zh_CN
```
三星浏览器user agent: 
``` bash
Mozilla/5.0 (Linux; Android 6.0.1; SAMSUNG SM-G9280 Build/MMB29K) AppleWebKit/537.36 (KHTML, like Gecko) SamsungBrowser/4.0 Chrome/44.0.2403.133 Mobile Safari/537.36
```

### 过程
以下是问题解决思路
1. 同样的视频文件，在该大企业的网站内时，安卓机是可以正常播放的。
2. 而视频文件到了cdn服务器里，安卓机提示不能解析。
3. 使用chrome浏览器的network对比两边请求头 _(Request Header)_ 和响应头 *(Response Header)* 
4. 发现从cdn服务器请求到的mp4视频比大企业的网站内请求到的mp4视频的请求头多了一行`Accept-Encoding:gzip, deflate, sdch`
5. 发现从cdn服务器请求到的mp4视频比大企业的网站内请求到的mp4视频的响应头多了一行`Content-Encoding: gzip`
6. 进行有针对性的进行搜索

经过查询发现：
* 4中请求头`Accept-Encoding:gzip, deflate, sdch`的含义是浏览器告诉服务器，其能够解析的压缩种类是`gzip, deflate, sdch`；
* 5中响应头`Content-Encoding: gzip`的含义是服务器告诉浏览器，其所响应的mp4视频所用的压缩方式是`gzip`。
* cdn服务器会为了提升效率，配置了给输出的mp4视频进行gzip视频流压缩，以达到加速的效果。
* 微信内置浏览器不支持gzip压缩，详情可见[此处](http://www.qcyoung.com/2015/11/11/%E5%BE%AE%E4%BF%A1%E5%86%85%E7%BD%AE%E6%B5%8F%E8%A7%88%E5%99%A8%E4%B8%8D%E6%94%AF%E6%8C%81gzip%E5%8E%8B%E7%BC%A9%E5%8F%8Agzip%E6%A8%A1%E5%9D%97%E9%85%8D%E7%BD%AE%E7%AE%80%E8%BF%B0/)。

接着又测试了一下安卓微信内置浏览器和安卓默认浏览器的视频播放差异，发现视频在华为浏览器内是可以播放的，三星浏览器和微信内置浏览器不行。而该项目的投放主要是在微信中进行，必须要把微信内置浏览器的兼容放在首位。
于是联系cdn管理员进行重新配置，关闭mp4的gzip压缩，当响应头内的`Content-Encoding: gzip`消失时，测试用安卓机的微信内置浏览器也能够播放视频了。

由于正式环境已经正常了而且也没有办法输出信息，于是决定在本地搭模拟环境，自由的获取请求头和响应头以观察区别。

### 模拟环境
在本机上模拟了一下环境，将同样的mp4视频放到目录内，apache服务器配置了gzip压缩环境，参照[此处](http://cl314413.blog.163.com/blog/static/190507976201006105628622/)。
打开httpd.conf，将下面两个模块打开。
``` bash
LoadModule deflate_module modules/mod_deflate.so
LoadModule headers_module modules/mod_headers.so
```
在配置文件中写入，配置指令可参考[Apache模块 mod_mime文档](http://man.chinaunix.net/newsoft/Apache2.2_chinese_manual/mod/mod_mime.html#addoutputfilter)和[Apache模块 mod_deflate文档](http://man.chinaunix.net/newsoft/Apache2.2_chinese_manual/mod/mod_deflate.html#deflatecompressionlevel)
``` bash
<ifmodule mod_deflate.c>
DeflateCompressionLevel 1
AddOutputFilter DEFLATE mp4
</ifmodule>
```
建个php，上代码
``` php
<?php
function get_head($szUrl){
    $curl = curl_init();
    curl_setopt($curl, CURLOPT_URL, $szUrl);
    curl_setopt($curl, CURLOPT_HEADER, 1);  //输出header信息
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);  //不显示网页内容
    curl_setopt($curl, CURLOPT_ENCODING, ''); //允许执行gzip
    $data=curl_exec($curl);
    if(!curl_errno($curl))
    {
        $info = curl_getinfo($curl);
        $httpHeaderSize = $info['header_size'];  //header字符串体积
        $pHeader = substr($data, 0, $httpHeaderSize); //获得header字符串
        return $pHeader;
    }else{
        return curl_errno($curl);
    }
}
$header=get_head('http://10.*.*.*/video/video.mp4');
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Video test</title>
</head>
<body>
<h4>请求头(Request Header)</h4>
<span>HTTP_USER_AGENT:</span>&nbsp;&nbsp;<?php echo $_SERVER['HTTP_USER_AGENT'] ?><br>
<span>HTTP_ACCEPT_ENCODING:</span>&nbsp;&nbsp;<?php echo $_SERVER['HTTP_ACCEPT_ENCODING'] ?>
<h4>响应头(Response Header)</h4>
<?php echo str_replace(array("\r\n","\r","\n"),'<br>',$header) ?>
<hr>
<video src="video.mp4" controls="controls">
    您的浏览器不支持 video 标签。
</video>
</body>
</html>
```
使用PC端chrome运行后，页面如下图，同一局域网内手机也能够获取差不多的数据。
![image](/images/vt_pc_chrome.jpg)
同一局域网内华为P10PLUS微信内置浏览器和华为浏览器运行video test的截图如下：
![image](/images/vt_hw_comb.jpg)
三星Galaxy S6 edge+微信内置浏览器和三星默认浏览器运行video test的截图如下：
![image](/images/vt_s6_comb_fix.jpg)
经过详细的测试后，证实了安卓版微信6.5.7内置浏览器不能正确处理gzip压缩过的视频的问题。

### 总结
正式环境发现问题，经过本机模拟环境验证，得出的结论是：
**测试安卓机的微信版本6.5.7的内置浏览器没有正确解压gzip的功能，或者解压gzip功能损坏。**
该问题仅限于安卓微信版本6.5.7还是其他已经没有办法继续，毕竟公司内能拿来的安卓机都拿来测过了，暂时先写下这么一篇笔记，做个记录。

**在微信内置浏览器中播放视频，当iphone可以播放，安卓无法播放时，可以考虑是否是安卓版微信无法正常处理gzip的问题。**

同时也总结出一个教训，当投放的内容有占用大带宽(视频)的操作时，要先了解清楚投放网站的带宽，做好备选措施。比如这次事故，假如提前做好调查和上级打好招呼，至少可以省掉部份协商的时间。

### Reference
[Apache HTTP Server 版本2.2指令索引](http://man.chinaunix.net/newsoft/Apache2.2_chinese_manual/mod/directives.html)
[Apache启用GZIP压缩优化网站](http://cl314413.blog.163.com/blog/static/190507976201006105628622/)
[微信内置浏览器不支持gzip压缩](http://www.qcyoung.com/2015/11/11/%E5%BE%AE%E4%BF%A1%E5%86%85%E7%BD%AE%E6%B5%8F%E8%A7%88%E5%99%A8%E4%B8%8D%E6%94%AF%E6%8C%81gzip%E5%8E%8B%E7%BC%A9%E5%8F%8Agzip%E6%A8%A1%E5%9D%97%E9%85%8D%E7%BD%AE%E7%AE%80%E8%BF%B0/)
[mod_gzip和mod_deflate的区别](https://seonoco.com/the-difference-between-mod-deflate-and-mod-gzip)
[Which browsers handle 'Content-Encoding: gzip'](https://webmasters.stackexchange.com/questions/22217/which-browsers-handle-content-encoding-gzip-and-which-of-them-has-any-special)
[HTTP 协议中的 Content-Encoding](https://imququ.com/post/content-encoding-header-in-http.html)
[How to set Content-Encoding with gzip](http://stackoverflow.com/questions/864448/how-to-set-content-encoding-with-gzip)
