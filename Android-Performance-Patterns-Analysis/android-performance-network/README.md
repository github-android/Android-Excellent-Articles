Android性能优化之移动端网络优化
====
一个网络请求可以简单分为**连接服务器** -> **获取数据**两个部分。</br>
其中连接服务器前还包括 DNS 解析的过程；获取数据后可能会对数据进行缓存。

## 一、连接服务器优化策略 ##
### 1. 不用域名，用 IP 直连 ###
省去 DNS 解析过程，DNS 全名 Domain Name System，解析意指根据域名得到其对应的 IP 地址。 如 [http://www.codekk.com](http://www.codekk.com) 的域名解析结果就是 104.236.147.76。
 
首次域名解析一般需要几百毫秒，可通过直接向 IP 而非域名请求，节省掉这部分时间，同时可以预防域名劫持等带来的风险。
 
当然为了安全和扩展考虑，这个 IP 可能是一个动态更新的 IP 列表，并在 IP 不可用情况下通过域名访问。

除了速度差异之外，执行GC操作的时候，所有线程的任何操作都会需要暂停，等待GC操作完成之后，其他操作才能够继续运行。

### 2. 服务器合理部署 ###
服务器多运营商多地部署，一般至少含三大运营商、南中北三地部署。
 
配合上面说到的动态 IP 列表，支持优先级，每次根据地域、网络类型等选择最优的服务器 IP 进行连接。
 
对于服务器端还可以调优服务器的 TCP 拥塞窗口大小、重传超时时间(RTO)、最大传输单元(MTU)等。

## 二、获取数据优化策略 ##
### 1. 连接复用 ###
节省连接建立时间，如开启 keep-alive。
 
Http 1.1 默认启动了 keep-alive。对于 Android 来说默认情况下 HttpURLConnection 和 HttpClient 都开启了 keep-alive。只是 2.2 之前 HttpURLConnection 存在影响连接池的 Bug，具体可见：[Android HttpURLConnection及HttpClient()选择](http://www.trinea.cn/android/android-http-api-compare/) 

### 2. 请求合并 ###
即将多个请求合并为一个进行请求，比较常见的就是网页中的 CSS Image Sprites。 如果某个页面内请求过多，也可以考虑做一定的请求合并。

### 3. 减小请求数据大小 ###
1. 对于 POST 请求，Body 可以做 Gzip 压缩，如日志。
2. 对请求头进行压缩</br>
这个 Http 1.1 不支持，SPDY 及 Http 2.0 支持。 Http 1.1 可以通过服务端对前一个请求的请求头进行缓存，后面相同请求头用 md5 之类的 id 来表示即可。

### 4. CDN 缓存静态资源 ###
缓存常见的图片、JS、CSS 等静态资源。

### 5. 减小返回数据大小 ###
1. 压缩</br>
一般 API 数据使用 Gzip 压缩，下图是之前测试的 Gzip 压缩前后对比图。 
![network.jpg](images/network.jpg)
2. 精简数据格式</br>
如 JSON 代替 XML，WebP 代替其他图片格式。关注公众号 codekk，回复 20 查看关于 WebP 的介绍。
3. 对于不同的设备不同网络返回不同的内容 如不同分辨率图片大小。
4. 增量更新</br>
需要数据更新时，可考虑增量更新。如常见的服务端进行 bsdiff，客户端进行 bspatch。
5. 大文件下载</br>
支持断点续传，并缓存 Http Resonse 的 ETag 标识，下次请求时带上，从而确定是否数据改变过，未改变则直接返回 304。

### 数据缓存 ###
缓存获取到的数据，在一定的有效时间内再次请求可以直接从缓存读取数据。
 
关于 Http 缓存规则 Grumoon 在 [Volley 源码解析](../../Android-Open-Project-Analysis/volley)最后杂谈中有详细介绍。

## 三、其他优化手段 ##
1. 预取</br>
包括预连接、预取数据。
 
2. 分优先级、延迟部分请求</br>
将不重要的请求延迟，这样既可以削峰减少并发、又可以和后面类似的请求做合并。
 
3. 多连接</br>
对于较大文件，如大图片、文件下载可考虑多连接。 需要控制请求的最大并发量，毕竟移动端网络受限。
 
## 四、监控 ##
优化需要通过数据对比才能看出效果，所以监控系统必不可少，通过前后端的数据监控确定调优效果。
 
注：服务器部署方面的优化有参考手 Q 和 QZone 去年的技术分享。


>以下是网络优化中一些客户端和服务器端需要尽量遵守的准则：

* 图片必须缓存，最好根据机型做图片做图片适配
* 所有http请求必须添加httptimeout
* 开启gzip压缩
* api接口数据以json格式返回，而不是xml或html
* 根据http头信息中的Cache-Control及expires域确定是否缓存请求结果。
* 确定网络请求的connection是否keep-alive
* 减少网络请求次数，服务器端适当做请求合并。
* 减少重定向次数
* api接口服务器端响应时间不超过100ms

google正在做将移动端网页速度降至1秒的项目，关注中[https://developers.google.com/speed/docs/insights/mobile](https://developers.google.com/speed/docs/insights/mobile)
