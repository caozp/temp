# WebView

Android WebView在Android平台上是一个特殊的View， 基于webkit引擎、展现web页面的控件，这个类可以被用来在你的app中显示一张在线的网页。WebView内部实现是采用渲染引擎来展示view的内容，提供网页前进后退，网页放大，缩小，搜索。Android的Webview在低版本和高版本采用了不同的webkit版本内核，4.4后直接使用了Chrome。

## 加载html

WebView都是通过loadUrl方法去加载html文件。

```
webView.loadUrl("http://139.196.35.30:8080/OkHttpTest/apppackage/test.html");//加载url

webView.loadUrl("file:///android_asset/test.html");//加载asset文件夹下html

//方式3：加载手机sdcard上的html页面
webView.loadUrl("content://com.ansen.webview/sdcard/test.html");

//方式4 使用webview显示html代码
webView.loadDataWithBaseURL(null,"<html><head><title> 欢迎您 </title></head>" +
        "<body><h2>使用webview显示 html代码</h2></body></html>", "text/html" , "utf-8", null);
```

## WebSettings

WebSettings是用来管理WebView配置的类。当WebView第一次创建时，内部会包含一个默认配置的集合。若我们想更改这些配置，便可以通过WebSettings里的方法来进行设置。                                

WebSettings对象可以通过WebView.getSettings()获得，它的生命周期是与它的WebView本身息息相关的，如果WebView被销毁了，那么任何由WebSettings调用的方法也同样不能使用。

```
WebSettings webSettings = mWebView .getSettings();

//支持获取手势焦点，输入用户名、密码或其他
webview.requestFocusFromTouch();
//===================JS====================
setJavaScriptEnabled(true);  //支持js
setPluginsEnabled(true);  //支持插件



//===================缩放处理====================
setSupportZoom(true);  //支持缩放，默认为true。是下面那个的前提。
setBuiltInZoomControls(true); //设置内置的缩放控件。
//若上面是false，则该WebView不可缩放，这个不管设置什么都不能缩放。
setDisplayZoomControls(false); //隐藏原生的缩放控件


//===================内容布局====================
setLayoutAlgorithm(LayoutAlgorithm.SINGLE_COLUMN); //支持内容重新布局
supportMultipleWindows();  //多窗口

//===================文件缓存====================
//10M缓存，api 18后，系统自动管理。
setAppCacheMaxSize(10 * 1024 * 1024);
//允许缓存，设置缓存位置
setAppCacheEnabled(true);
setAppCachePath(context.getDir("appcache", 0).getPath());
setAllowFileAccess(true);  //设置可以访问文件

//===================其他====================
setRenderPriority(RenderPriority.HIGH);  //提高渲染的优先级
设置自适应屏幕，两者合用
setUseWideViewPort(true);  //将图片调整到适合webview的大小
setLoadWithOverviewMode(true); // 缩放至屏幕的大小
setNeedInitialFocus(true); //当webview调用requestFocus时为webview设置节点
setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口
setLoadsImagesAutomatically(true);  //支持自动加载图片
setDefaultTextEncodingName("utf-8");//设置编码格式
setTextZoom(2);//设置文本的缩放倍数，默认为 100
setStandardFontFamily("");//设置 WebView 的字体，默认字体为 "sans-serif"
setDefaultFontSize(20);//设置 WebView 字体的大小，默认大小为 16
setMinimumFontSize(12);//设置 WebView 支持的最小字体大小，默认为 8
```

### 关于缓存

WebSetting提供了如下几种缓存模式：

| key                     | 含义                                                       |
| ----------------------- | ---------------------------------------------------------- |
| LOAD_CACHE_ONLY         | 不使用网络，只读取本地缓存数据                             |
| LOAD_DEFAULT            | 默认 根据cache-control决定是否从网络上取数据               |
| LOAD_NO_CACHE           | 不使用缓存，只从网络获取数据                               |
| LOAD_CACHE_ELSE_NETWORK | 只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据 |

结合使用（离线加载）

```
if (NetStatusUtil.isConnected(getApplicationContext())) {
    webSettings.setCacheMode(WebSettings.LOAD_DEFAULT);//根据cache-control决定是否从网络上取数据。
} else {
    webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);//没网，则从本地获取，即离线加载
}

webSettings.setDomStorageEnabled(true); // 开启 DOM storage API 功能
webSettings.setDatabaseEnabled(true);   //开启 database storage API 功能
webSettings.setAppCacheEnabled(true);//开启 Application Caches 功能

String cacheDirPath = getFilesDir().getAbsolutePath() + APP_CACAHE_DIRNAME;
webSettings.setAppCachePath(cacheDirPath); //设置  Application Caches 缓存目录
```



当WebView的使用频率变得频繁的时候，对于其各方面的优化就变得逐渐重要了起来。可以知道的是，我们每加载一个 H5页面，都会有很多的请求。除了HTML主URL自身的请求外，HTML外部引用的 JS、CSS、字体文件、图片都是一个个独立的HTTP 请求，虽然请求是并发的，但当网页整体数量达到一定程度的时候，再加上浏览器解析、渲染的时间，Web整体的加载时间变得很长。同时请求文件越多，消耗的流量也会越多。那么对于加载的优化就变得非常重要。成熟的框架通常会采用离线包的方式。



## WebViewClient

WebViewClient就是帮助WebView处理各种通知、请求事件的。 如果页面中链接,如果希望点击链接继续在当前browser中响应,而不是新开Android的系统browser中响应该链接,必须覆盖 WebView的WebViewClient对象

```
mWebView.setWebViewClient(new WebViewClient(){
    public boolean shouldOverrideUrlLoading(WebView view, String url){ 
        view.loadUrl(url);
        return true;
    }
});
```

注意返回值为true，表示不离开当前应用的webview,并自己处理参数url。

其他方法还有

* **onLoadResource**(WebView view, String url)：该方法在加载页面资源时会回调，每一个资源（比如图片）的加载都会调用一次

* **onPageStarted**(WebView view, String url, Bitmap favicon)：该方法在WebView开始加载页面且仅在Main frame loading（即整页加载）时回调，一次Main frame的加载只会回调该方法一次。我们可以在这个方法里设定开启一个加载的动画，告诉用户程序在等待网络的响应。
* **onPageFinished**(WebView view, String url)：该方法只在WebView完成一个页面加载时调用一次（同样也只在Main frame loading时调用），我们可以可以在此时关闭加载动画，进行其他操作
* **onReceivedError**(WebView view, WebResourceRequest request, WebResourceError error)：该方法在web页面加载错误时回调，这些错误通常都是由无法与服务器正常连接引起的，最常见的就是网络问题。 这个方法有两个地方需要注意：
  1. 这个方法只在与服务器无法正常连接时调用，类似于服务器返回错误码的那种错误（即HTTP ERROR），该方法是不会回调的，因为你已经和服务器正常连接上了
  2. 这个方法是新版本的onReceivedError()方法，从API23开始引进，与旧方法onReceivedError(WebView view,int errorCode,String description,String failingUrl)不同的是，新方法在页面局部加载发生错误时也会被调用（比如页面里两个子Tab或者一张图片）。这就意味着该方法的调用频率可能会更加频繁，所以我们应该在该方法里执行尽量少的操作。
* **onReceivedHttpError**(WebView view, WebResourceRequest request, WebResourceResponse errorResponse)：上一个方法提到onReceivedError并不会在服务器返回错误码时被回调，那么当我们需要捕捉HTTP ERROR并进行相应操作时应该怎么办呢？API23便引入了该方法。当服务器返回一个HTTP ERROR并且它的status
  code>=400时，该方法便会回调。这个方法的作用域并不局限于Main Frame，任何资源的加载引发HTTP ERROR都会引起该方法的回调，所以我们也应该在该方法里执行尽量少的操作，只进行非常必要的错误处理等。
* **onReceivedSslError**(WebView view, SslErrorHandler handler, SslError error)：当WebView加载某个资源引发SSL错误时会回调该方法，这时WebView要么执行handler.cancel()取消加载，要么执行handler.proceed()方法继续加载（默认为cancel）。需要注意的是，这个决定可能会被保留并在将来再次遇到SSL错误时执行同样的操作。
* **WebResourceResponse shouldInterceptRequest**(WebView view, WebResourceRequest request)：当WebView需要请求某个数据时，这个方法可以拦截该请求来告知app并且允许app本身返回一个数据来替代我们原本要加载的数据。
* **boolean shouldOverrideKeyEvent**(WebView view, KeyEvent event)：默认值为false，重写此方法并return true可以让我们在WebView内处理按键事件。
* **onScaleChanged**(WebView view, float oldScale, float newScale)：当WebView得页面Scale值发生改变时回调。

##  WebChromeClient

如果说WebViewClient是帮助WebView处理各种通知、请求事件的“内政大臣”的话，那么WebChromeClient就是辅助WebView处理Javascript的对话框，网站图标，网站title，加载进度等偏外部事件的“外交大臣”。

* onProgressChanged(WebView view, int newProgress)：当页面加载的进度发生改变时回调，用来告知主程序当前页面的加载进度。
* onReceivedIcon(WebView view, Bitmap icon)：用来接收web页面的icon，我们可以在这里将该页面的icon设置到Toolbar。
* onReceivedTitle(WebView view, String title)：用来接收web页面的title，我们可以在这里将页面的title设置到Toolbar。

以下两个方法是为了支持web页面进入全屏模式而存在的（比如播放视频），如果不实现这两个方法，该web上的内容便不能进入全屏模式

* **onShowCustomView**(View view, WebChromeClient.CustomViewCallback callback)：该方法在当前页面进入全屏模式时回调，主程序必须提供一个包含当前web内容（视频 or Something）的自定义的View。
* **onHideCustomView()**：该方法在当前页面退出全屏模式时回调，主程序应在这时隐藏之前show出来的View
* **boolean onJsAlert**(WebView view, String url, String message, JsResult result)：处理Javascript中的Alert对话框。
* **boolean onJsPrompt**(WebView view, String url, String message, String defaultValue, JsPromptResult result)：处理Javascript中的Prompt对话框。
* **boolean onJsConfirm**(WebView view, String url, String message, JsResult result)：处理Javascript中的Confirm对话框
* **boolean onShowFileChooser**(WebView webView, ValueCallback filePathCallback, WebChromeClient.FileChooserParams fileChooserParams)：该方法在用户进行了web上某个需要上传文件的操作时回调。我们应该在这里打开一个文件选择器，如果要取消这个请求我们可以调用filePathCallback.onReceiveValue(null)并返回true。
* **onPermissionRequest**(PermissionRequest request)：该方法在web页面请求某个尚未被允许或拒绝的权限时回调，主程序在此时调用grant(String [])或deny()方法。如果该方法没有被重写，则默认拒绝web页面请求的权限。
* **onPermissionRequestCanceled**(PermissionRequest request)：该方法在web权限申请权限被取消时回调，这时应该隐藏任何与之相关的UI界面。

## WebView加载一个网页的过程中该做些什么

如果说加载一个网页只需要调用`WebView.loadUrl(url)`这么简单，那肯定没程序员啥事儿了。往往事情没有这么简单。加载网页是一个复杂的过程，在这个过程中，我们可能需要执行一些操作，包括：

1. 加载网页前，重置WebView状态以及与业务绑定的变量状态。WebView状态包括重定向状态(`mTouchByUser`)、前端控制的回退栈(`mBackStep`)等，业务状态包括进度条、当前页的分享内容、分享按钮的显示隐藏等。
2. 加载网页前，根据不同的域拼接本地客户端的参数，包括基本的机型信息、版本信息、登录信息以及埋点使用的Refer信息等，有时候涉及交易、财产等还需要做额外的配置。
3. 开始执行页面加载操作时，会回调`WebViewClient.onPageStarted(webview, url, favicon)`。在此方法中，可以重置重定向保护的变量(`mRedirectProtected`)，当然也可以在页面加载前重置，由于历史遗留代码问题，此处尚未省去优化。
4. 加载页面的过程中，WebView会回调几个方法。
   * `WebChromeClient.onReceivedTitle(webview, title)`，用来设置标题。需要注意的是，在部分Android系统版本中可能会回调多次这个方法，而且有时候回调的title是一个url，客户端可以针对这种情况进行特殊处理，避免在标题栏显示不必要的链接
   * `WebChromeClient.onProgressChanged(webview, progress)`，根据这个回调，可以控制进度条的进度（包括显示与隐藏）。一般情况下，想要达到100%的进度需要的时间较长（特别是首次加载），用户长时间等待进度条不消失必定会感到焦虑，影响体验。其实当progress达到80的时候，加载出来的页面已经基本可用了。事实上，国内厂商大部分都会提前隐藏进度条，让用户以为网页加载很快。
   * `WebViewClient.shouldInterceptRequest(webview, request)`，无论是普通的页面请求(使用GET/POST)，还是页面中的异步请求，或者页面中的资源请求，都会回调这个方法，给开发一次拦截请求的机会。在这个方法中，我们可以进行静态资源的拦截并使用缓存数据代替，也可以拦截页面，使用自己的网络框架来请求数据。包括后面介绍的WebView免流方案，也和此方法有关
   * `WebViewClient.shouldOverrideUrlLoading(webview, request)`，如果遇到了重定向，或者点击了页面中的a标签实现页面跳转，那么会回调这个方法。可以说这个是WebView里面最重要的回调之一，后面`WebView与Native页面交互`一节将会详细介绍这个方法。
   * `WebViewClient.onReceived**Error(webview, handler, error)`，加载页面的过程中发生了错误，会回调这个方法。主要是http错误以及ssl错误。在这两个回调中，我们可以进行异常上报，监控异常页面、过期页面，及时反馈给运营或前端修改。在处理ssl错误时，遇到不信任的证书可以进行特殊处理，例如对域名进行判断，针对自己公司的域名“放行”，防止进入丑陋的错误证书页面。也可以与Chrome一样，弹出ssl证书疑问弹窗，给用户选择的余地。
5. 页面加载结束后，会回调`WebViewClient.onPageFinished(webview, url)`。这时候可以根据回退栈的情况判断是否显示关闭WebView按钮。通过`mActivityWeb.canGoBackOrForward(-1)`判断是否可以回退。



## WebView与JavaScript交互——JsBridge

通常，Java调用js方法有两种：

* WebView.loadUrl("javascript:" + javascript);
* WebView.evaluateJavascript(javascript, callbacck);

第一种方式已经不推荐使用了，第二种方式不仅更方便，也提供了结果的回调，但仅支持API 19以后的系统。

js调用Java的方法有四种，分别是：

* 通过WebView的addJavascriptInterface（）进行**对象映射** 被JS调用的方法必须加入@JavascriptInterface注解
* 通过 WebViewClient 的shouldOverrideUrlLoading ()方法回调拦截 url
* 通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt()方法回调拦截JS对话框alert()、confirm()、prompt()消息

这边主要介绍的是`addJavascriptInterface`的方法

1. 编写Java原生方法并用使用 @JavascriptInterface 注解

   ```
    @JavascriptInterface 
     public String getMessage(){
       return "Hello,boy~";
     }
   ```

2. 添加JavaScript的映射。

   ```
   webView.addJavaScriptInterface(this,"Android");
   ```

   把这个类添加到WebView的JavascriptInterface中。webView.addJavascriptInterface(this,"Android") 在Js代码中就能直接通过“Android”直接调用了该Native的类的方法getMessage()

3. 通过JavaScript调用Java方法

   ```
   function showHello(){
       var str=window.Android.getMessage();
       console.log(str);
   }
   ```

   **调用格式为window.jsInterfaceName.methodName(parameterValues)**

## 避免WebView内存泄露的方式
1. 不再xml中定义WebView，而是用代码动态添加

```
    LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
    mWebView = new WebView(getApplicationContext());
    mWebView.setLayoutParams(params);
    mLayout.addView(mWebView);
```
2. 在 Activity 销毁的时候，可以先让 WebView 加载null内容，然后移除 WebView，再销毁 WebView，最后置空。

```
@Override
 protected void onDestroy() {
      if (mWebView != null) {
          mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
          mWebView.clearHistory();

         ((ViewGroup) mWebView.getParent()).removeView(mWebView);
          mWebView.destroy()；            
          mWebView = null;
        }
        super.onDestroy();
    }

```

## HTTP和HTTPS混合

从Android5.0开始，WebView默认不支持同时加载Https和Http混合模式。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
     webView.getSettings().setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
}
```

从Android5.0以后，当一个安全的站点（https）去加载一个非安全的站点（http）时，需要配置Webview加载内容的混合模式，一共有如下三种模式：

* **MIXED_CONTENT_ALWAYS_ALLOW**：Webview不允许一个安全的站点（https）去加载非安全的站点内容（http）,比如，https网页内容的图片是http链接。强烈建议App使用这种模式，因为这样更安全
* **MIXED_CONTENT_NEVER_ALLOW**：在这种模式下，WebView是可以在一个安全的站点（Https）里加载非安全的站点内容（Http）,这是WebView最不安全的操作模式，尽可能地不要使用这种模式
* **MIXED_CONTENT_COMPATIBILITY_MODE**：在这种模式下，当涉及到混合式内容时，WebView会尝试去兼容最新Web浏览器的风格。一些不安全的内容（Http）能被加载到一个安全的站点上（Https），而其他类型的内容将会被阻塞。这些内容的类型是被允许加载还是被阻塞可能会随着版本的不同而改变，并没有明确的定义。这种模式主要用于在App里面不能控制内容的渲染，但是又希望在一个安全的环境下运行。

## WebView文件上传功能

WebView中的文件上传功能，当我们在Web页面上点击选择文件的控件(`<input type="file">`)时，会产生不同的回调方法

> void openFileChooser(ValueCallback uploadMsg) works on Android 2.2 (API level 8) up to Android 2.3 (API level 10)
>
> ​    
>
> openFileChooser(ValueCallback uploadMsg, String acceptType) works on Android 3.0 (API level 11) up to Android 4.0 (API level 15)
>
> ​    
>
> openFileChooser(ValueCallback uploadMsg, String acceptType, String capture) works on Android 4.1 (API level 16) up to Android 4.3 (API level 18)
>
> 
>
> onShowFileChooser(WebView webView, ValueCallback<Uri[]> filePathCallback, WebChromeClient.FileChooserParams fileChooserParams) works on Android 5.0 (API level 21) and above

最坑的点是在**Android4.4系统上没有回调**，这将导致功能的不完整，需要前端去做兼容。解决方案就是和前端另外约定一个jsbridge来解决此类问题



## WebView的其他方法

```
goBack()//后退
goForward()//前进
goBackOrForward(intsteps) //以当前的index为起始点前进或者后退到历史记录中指定的steps，
                              如果steps为负数则为后退，正数则为前进

canGoForward()//是否可以前进
canGoBack() //是否可以后退

清除数据:
clearCache(true);//清除网页访问留下的缓存，由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
clearHistory()//清除当前webview访问的历史记录，只会webview访问历史记录里的所有记录除了当前访问记录.
clearFormData()//这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据。

WebView状态:
onResume() //激活WebView为活跃状态，能正常执行网页的响应
onPause()//当页面被失去焦点被切换到后台不可见状态，需要执行onPause动过， onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。

pauseTimers()//当应用程序被切换到后台我们使用了webview， 这个方法不仅仅针对当前的webview而是全局的全应用程序的webview，它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
resumeTimers()//恢复pauseTimers时的动作。

destroy()//销毁，关闭了Activity时，音乐或视频，还在播放。就必须销毁。


判断WebView是否已经滚动到页面底端 或者 顶端:
因为webview可以缩放
if (webView.getContentHeight() * webView.getScale() == (webView.getHeight() + webView.getScrollY())) {
        //已经处于底端
    }

if(webView.getScrollY() == 0){
        //处于顶端 
        }

```



## 坑

### setJavaScriptEnabled

在Android 4.3版本调用`WebSettings.setJavaScriptEnabled()`方法时会调用一下reload方法，同时会回调多次`WebChromeClient.onJsPrompt()`。如果有业务逻辑依赖于这两个方法，就需要注意判断回调多次是否会带来影响了。

同时，如果启用了JavaScript，务必做好安全措施，防止**远程执行漏洞**。

```
@TargetApi(11)
private static final void removeJavascriptInterfaces(WebView webView) {
    try {
        if (Build.VERSION.SDK_INT >= 11 && Build.VERSION.SDK_INT < 17) {
	        webView.removeJavascriptInterface("searchBoxJavaBridge_");
	        webView.removeJavascriptInterface("accessibility");
	        webView.removeJavascriptInterface("accessibilityTraversal");
        }
    } catch (Throwable tr) {
        tr.printStackTrace();
    }
}
```

### setAllowFileAccess

**域控制不严格漏洞** ，A 应用可以通过 B 应用导出的 Activity 让 B 应用加载一个恶意的 file 协议的 url，从而可以获取 B 应用的内部私有文件，从而带来数据泄露威胁

阿里巴巴的开发手册中**强制**要求

```
setAllowFileAccess(false); 
setAllowFileAccessFromFileURLs(false);
setAllowUniversalAccessFromFileURLs(false);
```



## 参考

[**如何设计一个优雅健壮的Android WebView？**](https://blog.klmobile.app/2018/02/16/design-an-elegant-and-powerful-android-webview-part-one/#fn:5)