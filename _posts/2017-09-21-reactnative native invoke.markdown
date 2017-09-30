---
layout:     post
title:      React Native和iOS原生方法交互
author:     慢慢
tags: 		ReactNative iOS
subtitle:  	
category:  blog
---



# 原生传递参数给React Native
## 初始化时传值
```
- (instancetype)initWithBundleURL:(NSURL *)bundleURL
                       moduleName:(NSString *)moduleName
                initialProperties:(NSDictionary *)initialProperties
                    launchOptions:(NSDictionary *)launchOptions;
```
通过初始化方法中的 initialProperties 可以给 RCTRootView 传值

## appProperties传值

同上面的方法类似，只是不是在初始化时传值，在设置 appProperties 之后，React Native应用将会根据新的属性重新渲染。当然，只有在新属性和之前的属性有区别时更新才会被触发。

```
/**
 * The properties to apply to the view. Use this property to update
 * application properties and rerender the view. Initialized with
 * initialProperties argument of the initializer.
 *
 * Set this property only on the main thread.
 */
 
@property (nonatomic, copy, readwrite) NSDictionary *appProperties;
```

> 传递的参数为OC的字典类型，在js中可以通过属性值.props访问字典的key，就可以取到参数值

# React Native执行原生方法

Native端：

1. 首先需要调用的原生方法需要实现 `RCTBridgeModule` 协议
2. 类实现中需要添加 `RCT_EXPORT_MODULE()` 的宏
3. 通过 `RCT_EXPORT_METHOD()`宏 来实现需要导出 javascript 的方法

```
@implementation YFTestObject

RCT_EXPORT_MODULE();

RCT_EXPORT_METHOD(doSomethingCallback:(NSString *)string callback:(RCTResponseSenderBlock)callback)
{
  NSLog(@"%@",string);
  callback(@[[NSNull null] ,@[@"test1", @"test2"]]); // 第一个error，第二个回调参数
}
@end
```

JS端：

1. 需要引用 `NativeModules` 模块
2. 通过 `NativeModules.xxx` 获取 Native 的类，xxx为Native的类名
3. 上一步获取到的 Native 类，调用 Native 中导出的方法

```
render() {
    return (
      <View style={styles.container}> 
        <TouchableHighlight onPress={()=>testObject.doSomethingCallback('点击按钮（回调）', (error,events)=>{
                 if (error) {
                   console.error(error);
                 } else {
                   console.log(events);
                 }
           })}>
          <Text style={styles.welcome}>点击（回调）</Text>
        </TouchableHighlight>
      </View>
    );
  }
```

> 如果需要Natvie方法回调，需要将导出的方法增加`RCTResponseSenderBlock`类型的参数，该参数则为方法的回调方法

# 原生调用React Native方法

之前可以使用`sendAppEventWithName:body:`方法，但是已经废弃了。在0.28版本之后，iOS 向 JS 端发射消息的代码如下。 

1. 自定义的模块类头文件要继承自`RCTEventEmitter`。(因为 React 需要调用该类，所以也需要实现 `RCTBridgeModule` 协议)
2. 在 Native 方法中发送消息

	```
	[self sendEventWithName:@"MessageName"
	                   body:params];
	```
3. 在 React 方法中监听消息（需要在 `componentWillUnmount` 中移除监听）

```
var nativeBridge = NativeModules.YourClassName;// 你的类名
const nativeModule = new NativeEventEmitter(nativeBridge);
...

componentDidMount(){
	nativeModule.addListener('MessageName',(data)=>this._method(data));
}

```


简单的Demo：https://github.com/wf96390/YFReactNativeDemo


参考：

http://www.cnblogs.com/wujy/p/5864591.html
http://blog.csdn.net/xiangzhihong8/article/details/75092576
