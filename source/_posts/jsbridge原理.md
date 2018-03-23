> jsbridge相信大家经常用到。本文总结了js与ios和android交互的原理。

#### Javascript与IOS交互

##### Native调用javascript方法
Native调用javascript是通过UIWebView组件的stringByEvaluatingJavaScriptFromString方法来实现的，该方法返回js脚本的执行结果。

```swift
// Swift
webview.stringByEvaluatingJavaScriptFromString("Math.random()")  
// OC
[webView stringByEvaluatingJavaScriptFromString:@"Math.random();"];
```
可以看到它就是调用了window下的一个对象（执行了Math.random()方法），所以如果我们要让native来调用js写的方法，就要让这个方法在window下能访问到。但从全局考虑，只要暴露一个对象如JSBridge就可以了，而且通常我们会在这个对象上实现emit、on这种发布订阅的方法来方便调用。如：
```swift
// 下面为伪代码
// 执行了暴露在window下的对象JSBridge的trigger方法
webview.setDataToJs = function(data) {  
   webview.stringByEvaluatingJavaScriptFromString("JSBridge.trigger(event, data)")
}

```

##### javascript调用Native方法
javascript调用Native并没有现成的API，而是间接的通过一些方法来实现的。UIWebView有个特性，在UIWebView内发起的所有网络请求，都可以通过delegate函数在Native层得到通知。这样，我们就可以在UIWebView内发起一个自定义的网络请求，通常是这样的格式：jsbridge://methodName?param1=value1&param2=value2

Native在监听到这种地址之后，就不会加载内容，而是执行相应的某段逻辑。

发起这样一个网络请求有两种方式：1. 通过localtion.href；2. 通过iframe方式； 通过location.href有个问题，就是如果我们连续多次修改window.location.href的值，在Native层只能接收到最后一次请求，前面的请求都会被忽略掉。

所以更稳妥的方式是通过iframe, 简单的例子如下：

```javascript
var url = 'jsbridge://doAction?title=分享标题&desc=分享描述&link=http%3A%2F%2Fwww.baidu.com';  
var iframe = document.createElement('iframe');  
iframe.style.width = '1px';  
iframe.style.height = '1px';  
iframe.style.display = 'none';  
iframe.src = url;  
document.body.appendChild(iframe);
// 100毫秒后移除
setTimeout(function() {  
    iframe.remove();
}, 100);

```
然后Webview就可以拦截这个请求，并且解析出相应的方法和参数。如下代码所示：
```swift
func webView(webView: UIWebView, shouldStartLoadWithRequest request: NSURLRequest, navigationType: UIWebViewNavigationType) -> Bool {  
        print("shouldStartLoadWithRequest")
        let url = request.URL
        let scheme = url?.scheme
        let method = url?.host
        let query = url?.query

        if url != nil && scheme == "jsbridge" {
            print("scheme == \(scheme)")
            print("method == \(method)")
            print("query == \(query)")

            switch method! {
                case "getData":
                    self.getData()
                case "putData":
                    self.putData()
                case "gotoWebview":
                    self.gotoWebview()
                case "gotoNative":
                    self.gotoNative()
                case "doAction":
                    self.doAction()
                case "configNative":
                    self.configNative()
                default:
                    print("default")
            }

            return false
        } else {
            return true
        }
    }
```

#### Javascript与Android交互

##### Native调用javascript方法
在androidLi是使用weibiew的loadUrl进行调用的，如：
```java
// 调用js中的JSBridge.trigger方法
webView.loadUrl("javascript:JSBridge.trigger('webviewReady')");

```

##### javascript调用Native方式
在android中javascript有三种调用native的方法：
1. 同构schema方式，对url的协议进行解析，这种方式与ios的一样，使用iframe来调用
2. 通过在webview页面里直接注入原生js代码的方式，使用addJavascriptInterface方法来实现。

在android里实现如下：
```java
class JSInterface {  
    @JavascriptInterface //注意这个代码一定要加上
    public String getUserData() {
        return "UserData";
    }
}
webView.addJavascriptInterface(new JSInterface(), "AndroidJS"); 
```
上面的代码就是在页面的window对象里注入了AndroidJS对象。在js里可以直接调用
```javascript
alert(AndroidJS.getUserDate())  // UserData
```

3. 使用prompt，console.log, alert方法，这三个方法是js里的原生方法，在android webview里可以重写这三个方法，一般我们使用prompt，因为这个在js里使用不多。
```java
class YouzanWebChromeClient extends WebChromeClient {  
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
        // 这里就可以对js的prompt进行处理，通过result返回结果
    }
    @Override
    public boolean onConsoleMessage(ConsoleMessage consoleMessage) {

    }
    @Override
    public boolean onJsAlert(WebView view, String url, String message, JsResult result) {

    }

}

```

#### 总结

![](https://haitao.nos.netease.com/61ffebfa-deb6-4cfc-9b04-76d1f6023318.png)


参考文档：
- [H5与Native交互之JSBridge技术](https://tech.youzan.com/jsbridge/)
