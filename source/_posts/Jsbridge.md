---
title: 前端与appNative的交互 jsbridge

---
### 背景
 
 > 目前前端实现app的方式有很多，facebook的react Native 、阿里的weex(vue),但是相比原生还是存在很多问题，比如加载速度、调硬件设备、体验等方面。于是就有了中庸的一种实现方式 混合开发，用原生的壳内嵌H5页面，最近公司就有了这样一个项目，把之前写的公众号的项目内嵌进app。并且要保留之前的微信支付，添加直接拨打电话、分享朋友圈等功能，这就需要前端调用原生的方法，同时原生也会调用前端的方法。本文的重点就来了"jsbridge"，它就相当于前端与原生的一个中间连接桥。

 ### jsbridge 实现方式
 
 > #### 目前主要有三种实现方式， js的WebViewJavascriptBridge方法、api注入、URL SCHEME。 

 >1.WebViewJavascriptBridge

 原生引入WebViewJavascriptBridge的库，前端代码主要如下
 

	//注册事件监听，初始化
    const setupWebViewJavascriptBridge = (callback) => {
	  let userAgent = navigator.userAgent
	  let isAndroid = userAgent.indexOf('Android') > -1 || userAgent.indexOf('Adr') > -1
	  let isIOS = !!userAgent.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/)

	  // Android使用
	  if (isAndroid) {
	    if (window.WebViewJavascriptBridge) {
	      // eslint-disable-next-line
	      callback(WebViewJavascriptBridge)
	    } else {
	      document.addEventListener(
	        'WebViewJavascriptBridgeReady',
	        function () {
	          // eslint-disable-next-line
	          callback(WebViewJavascriptBridge)
	        },
	        false
	      )
	    }
	  }

	  // iOS使用
	  if (isIOS) {
	    if (window.WebViewJavascriptBridge) {
	      // eslint-disable-next-line
	      return callback(WebViewJavascriptBridge)
	    }
	    if (window.WVJBCallbacks) {
	      return window.WVJBCallbacks.push(callback)
	    }
	    window.WVJBCallbacks = [callback]
	    var WVJBIframe = document.createElement('iframe')
	    WVJBIframe.style.display = 'none'
	    WVJBIframe.src = 'https://__bridge_loaded__'
	    document.documentElement.appendChild(WVJBIframe)
	    setTimeout(function () {
	      document.documentElement.removeChild(WVJBIframe)
	    }, 0)
	  }
	}
	// 分享
	export const share = (platform, url, title) => {
	  setupWebViewJavascriptBridge((bridge) => {
	  	
	  	//window.WebViewJavascriptBridge

	    bridge.callHandler('appBridgeShareToWechat', JSON.stringify({platform: platform, url: url, title: title}), (response) => {
	      // let repObj = JSON.parse(response)
	      // if (repObj.success) {
	      //   tips('分享成功', 1000)
	      // }
	    })
	  })
	}
<!--more-->
	js传递数据
	var data = '发送数据给java默认接收';
   	window.WebViewJavascriptBridge.send(
       	data
       	, function(responseData) { //处理java回传的数据
          document.getElementById("show").innerHTML = responseData;
       	}
   	);

>2.注入api
>此方法对前端的调用比较简单，只需要原生把方法注入到js的上下文中，前端直接调用，如下

	ios:
	JSContext *context = [uiWebView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
		context[@"postBridgeMessage"] = ^(NSArray<NSArray *> *calls) {
	    // Native 逻辑
	};

	js:
	window.postBridgeMessage(message); 
	WKWebView window.webkit.messageHandlers.nativeBridge.postMessage(message);
	
	android:
	public class JavaScriptInterfaceDemoActivity extends Activity {
		private WebView Wv;
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        Wv = (WebView)findViewById(R.id.webView);
	        final JavaScriptInterface myJavaScriptInterface = new JavaScriptInterface(this);
	        Wv.getSettings().setJavaScriptEnabled(true);
	        Wv.addJavascriptInterface(myJavaScriptInterface, "nativeBridge");
	        // TODO 显示 WebView
	    }
	    public class JavaScriptInterface {
	         Context mContext;
	         JavaScriptInterface(Context c) {
	             mContext = c;
	         }
	         public void postMessage(String webMessage){
	             // Native 逻辑
	         }
	     }
	}

	js:

	window.nativeBridge.postMessage(message);

> 3.URL SCHEME
> 先解释一下 URL SCHEME：URL SCHEME是一种类似于url的链接，是为了方便app直接互相调用设计的，形式和普通的 url 近似，主要区别是 protocol 和 host 一般是自定义的，例如: qunarhy://hy/url?url=http://ymfe.tech，protocol 是 qunarhy，host 则是 hy。

> 拦截 URL SCHEME 的主要流程是：Web 端通过某种方式（例如 iframe.src）发送 URL Scheme 请求，之后 Native 拦截到请求并根据 URL SCHEME（包括所带的参数）进行相关操作。

> 在时间过程中，这种方式有一定的 缺陷：

> 使用 iframe.src 发送 URL SCHEME 会有 url 长度的隐患。
> 创建请求，需要一定的耗时，比注入 API 的方式调用同样的功能，耗时会较长。
> 但是之前为什么很多方案使用这种方式呢？因为它 支持 iOS6。而现在的大环境下，iOS6 占比很小，基本上可以忽略，所以并不推荐为了 iOS6 使用这种 并不优雅 的方式。

> 【注】：有些方案为了规避 url 长度隐患的缺陷，在 iOS 上采用了使用 Ajax 发送同域请求的方式，并将参数放到 head 或 body 里。这样，虽然规避了 url 长度的隐患，但是 WKWebView 
> 并不支持这样的方式。

> 【注2】：为什么选择 iframe.src 不选择 locaiton.href ？因为如果通过 location.href 连续调用 Native，很容易丢失一些调用。

### [参考资料 ymfe](https://blog.ymfe.org/%E6%B7%B7%E5%90%88%E5%BC%80%E5%8F%91%E4%B8%AD%E7%9A%84JSBridge/) [参考资料 zhifu ](https://zhuanlan.zhihu.com/p/35200622)




 
