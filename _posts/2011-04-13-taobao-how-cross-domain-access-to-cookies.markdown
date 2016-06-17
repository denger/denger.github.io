---
title: 淘宝如何跨域获取 Cookie 分析
layout: post
---

最近在发现使用Taobao的时候的一个小细节，于是便萌发起了写这篇文章。 
当我们在 www.taobao.com 中进行登录之后，然后直接切换到 www.tmall.com 域名下，发现www.tmall.com首页的最顶部马上显示成了”您好, andyfaces“，于是便对此处的实现机制进行分析。 
首先，用户名应该是存储在cookie中的，于是在taobao.com的域名中用 firefox看到用户名确实是存储在 cookie, 而tmall.com中没有存储该cookie:


![alt "IMG"](/images/2011/03/13/1.png)

可以确定的是对于cookie来说肯定是不允许垮域访问的。无论是通过JS还是Server端程序来说都是如此，那么tmall.com是如何访问到taobao.com下的cookie的呢? 于是打开 tmall.com，然后使用firebug来进行调试，发现了一条这样的请求语句：

![alt "IMG"](/images/2011/03/13/2.png)

其页面的JS代码为：

{% highlight js %}
KISSY.getScript("http://www.taobao.com/go/app/tmall/login-api.php?"+Math.random());
{% endhighlight js %}

看到这里之后于是也大概知道他如何处理了的，为了确认一下，于是搜索一下 [KISSY.getScript](http://docs.kissyui.com/kissy/build/kissy.js) 函数代码，确实采用了JS跨域的 [JSONP](http://remysharp.com/2007/10/08/what-is-jsonp/) 解决方案： 
{% highlight js %}
getScript: function(url, success, charset) {  
      var isCSS = RE_CSS.test(url),  
          node = doc.createElement(isCSS ? 'link' : 'script'),  
          config = success, error, timeout, timer;  

          node.src = url;  
          node.async = true;  

      scriptOnload(node, function() {  
              if (timer) {  
                  timer.cancel();  
                  timer = undef;  
              }  

              S.isFunction(success) && success.call(node);  

              // remove script  
              if (head && node.parentNode) {  
                  head.removeChild(node);  
              }  
          });  
      head.insertBefore(node, head.firstChild);  
}  
{% endhighlight js %}
其原理是通过动态create js include 动态加载js，然后为该script节点bind onload事件或判断onreadystatechange，其具体细节可以参考以上 scriptOnload 的函数的处理。 当js加载完成之后 采用回调方式来执行 success 函数。 
为了进一步确实，于是使用 Jquery的 $.getScript 来测试一把，首先在 taobao.com下进行登录成功，然后随便在本地写了一个测试页，通过以下语句: 
{% highlight js %}
$.getScript('http://www.taobao.com/go/app/tmall/login-api.php?0.6783450077710154', function(){  
    console.log("the taobao.com cookie object:" + userCookie + " username:" + userCookie._nk_);  
});  
{% endhighlight js %}
![alt "IMG"](/images/2011/03/13/3.png)

其实大致原理如此，通过在www.taobao.com 的server端提供一个获取当前域下所有cookie的 php的请求地址，然后该php获取到cookie之后将期并成 js 代码，也就是以上第二个截图所看到的。然后再在 tmall 采用 jsonp 的方式跨域加载该 js 代码，从而实现 cookie 的跨域访问。 







