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

#### native调用javascript

1 JS中注册要被OC调用的方法

首先在JSBridge.js定义一个全局可用的方法变量

```
//这部分是固定的
var iosBridge = function (callback) {
          if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
          if (window.WVJBCallbacks) { return  window.WVJBCallbacks.push(callback); }
          window.WVJBCallbacks = [callback];
          var WVJBIframe = document.createElement('iframe');
          WVJBIframe.style.display = 'none';
          WVJBIframe.src = 'https://__bridge_loaded__';
          document.documentElement.appendChild(WVJBIframe);
          setTimeout(function() {         
                document.documentElement.removeChild(WVJBIframe) 
          }, 0)
 }
```

- 在`JSBridge.js`中注册提供给`OC`调用的方法。`OC`调用`JS`中的注册方法，`JS`有两种响应方式，分别为：
   1. `JS`接收到`OC`的调用请求，需要通过回调把一些数据给`OC`的
   
   ```
   /**
     *方法原型：bridge.registerHandler("handlerName", function(data,responseData) { ... })
     *@first param: 'OCCallJS', 是OC调用JS的方法名，需要事先注册
     *@second param: 回调函数，其中data 是OC通过 ‘OCCallJS’ 方法，传过来的值， ’responseCallBack‘ 是JS回调给OC的方法，通过’responseCallBack‘可以给OC传递一个值
    */
     bridge.registerHandler('OCCallJS', function(data,responseCallBack){
            console.log('js is called by oc,data from oc:' + data)
            document.getElementById('div1').innerText = data + '并准备向JS要一个橘子'
            responseCallBack('并向JS要一个橘子')
     })
   ```
   
    2. `JS`接收到`OC`的调用请求，无需回调传值的,分为需要传参，和不需要传参两种，当然你也可以直接把不需要的部分传`nil`
    
    ```
    /**
     *JS 方法原型：bridge.registerHandler("handlerName", function(responseData) { ... })
     *@first param: 'OCCallJS', 是OC调用JS的方法名，需要事先注册
     *@second param: 回调函数，其中data 是OC通过 ‘OCCallJS’ 方法，传过来的值,可能为null
    */
    bridge.registerHandler('OCCallJS', function (data){
              document.getElementById('div1').innerText = data ? data : 'Objective-C主动给JS一个苹果，并且不要回报'
    })
    ```
    
2 OC调用JS中已经注册的方法(OC中调用方法，必须要和JS中注册的方法参数相匹配，否则会报错)

- OC端方法原型有三种：


```
//1
[bridge callHandler:(NSString*)handlerName data:(id)data responseCallback:(WVJBResponseCallback)callback];
//2
[bridge callHandler:(NSString)handlerName data:(id)data
//3
 [bridge callHandler:(NSString)handlerName]
```

1. 针对于`JS`注册方法1

```
/**注册方法1必须对应`OC`中的方法，如果`OC`没有`callback`，`JS`端会报错.
     *OC方法原型：[bridge callHandler:(NSString*)handlerName data:(id)data responseCallback:(WVJBResponseCallback)callback]
     *@first param: 'OCCallJS', 是OC调用的JS方法名
     *@second param data: 是需要通过OCCallJS方法，传递给JS的数据
     *@third param responseCallback: JS 收到调用之后，回调给OC的值
     */
     [self.bridge callHandler:@"OCCallJS" data:@"Objective-C主动给JS一个苹果" responseCallback:^(id responseData) {
             NSLog(@"Objective-C给JS一个苹果后，%@",responseData);
             self.label.text = [NSString stringWithFormat:@"Objective-C给JS一个苹果后，%@",responseData];
     }];
```

2. 针对于`JS` 注册方法2

```
/**
     *此方法对应OC中下面两个方法
     *OC方法原型：[bridge callHandler:(NSString*)handlerName data:(id)data] or [bridge callHandler:(NSString*)handlerName]
     *如果是前者 则对应的JS回调函数data中有值，后者，则为null 
    */
    [self.bridge callHandler:@"OCCallJS" data:@"Objective-C主动给JS一个苹果，并且不要回报"]; 
    或者
    [self.bridge callHandler:@"OCCallJS"];
```


#### javascript调用native

1 引入头文件

`#import <WebViewJavascriptBridge/WebViewJavascriptBridge.h>`

2 在OC中注册JS需要调用的方法

```
/**js 调用原生，获取数据,通过回调给JS
*OC方法原型- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler
*@first param handlerName: 'JSCallOC', 是JS调用的OC方法名
*@second param handler: JS调用OC的回调，包括JS传过来的数据data，和OC给JS回调传值。
*/
    [self.bridge registerHandler:@"JSCallOC" handler:^(id data, WVJBResponseCallback responseCallback) {
        //其中data可能为nil,如果不想在此给JS传值，可responseCallback(nil);
        self.label.text = [NSString stringWithFormat:@"%@,并准备向Objective-C要一个苹果",data];
        responseCallback(@"并向Objective-C要了一个苹果");
    }];
```
3 JS 调用在OC中注册过的方法

同样JS调用OC的原型方法，也有三种

```
//1
bridge.callHandler("handlerName")
//2
bridge.callHandler("handlerName", data) 
//3
bridge.callHandler("handlerName", data, function responseCallback(responseData) 
```

下面的三种方法实例，分别对应上面的三种方法

```
//1
 bridge.callHandler('JSCallOC')
//2
 bridge.callHandler('JSCallOC','JS主动给Objective-C一个橘子')
//3,如果OC的注册方法中，有回调函数中有值，则必须采用此方法，否则js会报错
bridge.callHandler('JSCallOC','JS主动给Objective-C一个橘子',function(responseData){
          document.getElementById('div1').innerText = 'JS主动给Objective-C一个橘子，' + responseData
})
```

### 注意点

1 不管是原生还是用三方库，native需要调用的javascript的方法，貌似必须要在HTML中提前注册,尤其是单页面应用，因为单页面应用只有一个HTML，界面基本上都组件化渲染。

2 javascript调用原生时，同样也需要在调用前，需要先注册。


