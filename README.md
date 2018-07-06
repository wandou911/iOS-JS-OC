# iOS-JS-OC
iOS JS和OC的相互调用

### 前言

在移动端开发中，有时可能考虑到开发效率，会在原生应用中集成Web界面，这样只需要web开发人员写完之后，原生只需要加载出来就可以了。但是，也存在native和js之间的交互情况，下面是我在解决交互中使用的方法，和遇见的问题

在原生用加载web界面，在iOS8之后，我们有两种选择，一种是UIKit里的UIWebView，一种是WebKit的WKWebView，其中WKWebView是在iOS8之后才推出的新的web展示控件，目的也很明确，就是为了解决UIWebView加载速度慢，消耗内存高的问题，据说同时加载100个页面，WKWebView内存的消耗量，要比UIWebView少很多。相对而言，如果不支持iOS8之前的版本的话，选择WKWebView可能会更好一点

Demo中含有两种方式，一种不借助三方库，直接用WKWebView，一种是借助三方库。另外，本篇只是讲了一些native和Javascript之间的交互问题，对于WKWebView的使用，会在以后的文章中继续更新


### 1 直接使用WKWebView的相关代理，进行交互（参考demo中的Native文件夹）

#### native调用javascript

1 JS中注册要被OC调用的方法

`javascript 代码`

```
//JS响应OC的调用
function OCCallJS(msg){
      document.getElementById('div1').innerText = msg ? msg : '收到了来自OC的调用'
      //如果OC需要在调用了js之后，获取返回值，可以通过return
      return 'JS收到了OC的调用，并返回了我'
}
```

2 OC调用JS中已经注册的方法

`OC代码`

```
-(void)click {
        //OC 主动调用JS
        [_wkWebView evaluateJavaScript:@"OCCallJS('JS被调用了')" completionHandler:^(id _Nullable response, NSError * _Nullable error) {
            //response是js方法OCCallJS的返回值
            NSLog(@"response:%@,error:%@",response,error);
        }];
}
```



#### javascript调用native

1 引入头文件 
`#import <WebKit/WebKit.h>`

2 实现协议
`@interface NativeJSViewController ()<WKScriptMessageHandler,WKNavigationDelegate,WKUIDelegate>`

3 在OC中注册JS需要调用的方法

```
WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
self.wkWebView = [[WKWebView alloc] initWithFrame:CGRectMake(0, 20, CGRectGetWidth(self.view.bounds), CGRectGetHeight(self.view.bounds)-120) configuration:config];
self.wkWebView.navigationDelegate = self;
[self.view addSubview:self.wkWebView];
//注册方法，必须在HTML加载之前添加
WKUserContentController *userCC = self.wkWebView.configuration.userContentController;
//JSCallOC,是javascript 主动调用OC时，
[userCC addScriptMessageHandler:self name:@"JSCallOC"];
[self loadLocalHtmlWithFileName:@"Native"];
```

4 JS调用在OC中注册过的方法

`javascript 代码`

```
function btnClick() {
    /**
     * JS 调用 OC，同样也需要现在OC注册方法名，然后才能在JS中调用
     * 原型：window.webkit.messageHandlers.<name>.postMessage(<messageBody>)
     */
    window.webkit.messageHandlers.JSCallOC.postMessage('JS主动调用OC')
}


```

### 2 借助第三方库 推荐[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)(参考Demo的JSBridge文件夹)
