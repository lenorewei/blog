---
title: React Native 内的webview重新封装并添加图片上传
---

Rn中的webview相比与原生的安卓的webview，有许多原生很容易实现的功能没有接入，但当我们需要使用到那些功能，例如去添加网页的加载进度百分比，添加对网页中type为file的input的支持等，rn中并没有与这些相关的api支持我们去做这些，当这样的时候，最好的方式就是在rn的代码基础上，重新去封装一个webview，利用原生的api去实现一个功能。
## 如何去封装一个webview
1. 拷贝reactnative内部的webview（ReactWebViewManager.class，WebViewConfig.class）实现代码到安卓目录下

  ![](https://lenore-1254182071.cos.ap-shanghai.myqcloud.com/blog/2018-06-13-072824.png)

2. 拷贝（`/node_modules/react-native/Libraries/Components/WebView/WebView.android.js`）到自己的文件夹中并修改
   
```java
    protected static final String REACT_CLASS = "RCTWebView";
```

为

```java
    protected static final String REACT_CLASS = "RCTWebView1";
```
1. 新建`ReactWebViewPackage.java`添加`ReactWebViewManager`
```java
    import com.facebook.react.ReactPackage;
    import com.facebook.react.bridge.NativeModule;
    import com.facebook.react.bridge.ReactApplicationContext;
    import com.facebook.react.uimanager.ViewManager;
    
    import java.util.Arrays;
    import java.util.Collections;
    import java.util.List;
    
    public class ReactWebViewPackage implements ReactPackage {
    
    
        @Override
        public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    
            return Collections.emptyList();
        }
    
        @Override
        public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
            return Arrays.<ViewManager>asList(new ReactWebViewManager());
        }
    
    }
```
4. 在MainApplication.java中添加引用
```java
    @Override
        protected List<ReactPackage> getPackages() {
          return Arrays.<ReactPackage>asList(
                  new MainReactPackage(),
                  new ReactWebViewPackage(),
          );
        }
```
这样就能在封好的webview中即`ReactWebViewPackage.java`中添加原生方法了。

## 如何添加图片的上传。
Rn的webview很坑爹的并不支持input类型为file，就是当遇到内嵌页面中有图片上传的需求的时候，就必须我们额外去添加上这一支持。
以下显示的是针对原生的rnwebview的封装，而不是针对我们上面自己封装的。
1. 添加`AdvancedWebView.java`,对webview多封装一层
```java
public class AdvancedWebviewManager extends ReactWebViewManager {

    private ValueCallback mUploadMessage;
    private final static int FCR=1;

    public WebView webview = null;

    private AdvancedWebviewPackage aPackage;
    public String getName() {

        return "AdvancedWebView";
    }

    @ReactProp(name = "enabledUploadAndroid")
    public void enabledUploadAndroid(WebView view, boolean enabled) {
        if(enabled) {
            webview = view;
            final AdvancedWebviewModule module = this.aPackage.getModule();
            view.setWebChromeClient(new WebChromeClient(){

                //For Android 3.0+
                public void openFileChooser(ValueCallback uploadMsg){
                    module.setUploadMessage(uploadMsg);
                    mUploadMessage = uploadMsg;
                    Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                    i.addCategory(Intent.CATEGORY_OPENABLE);
                    i.setType("*/*");
                    module.getActivity().startActivityForResult(Intent.createChooser(i, "File Chooser"), FCR);
                }
                // For Android 3.0+, above method not supported in some android 3+ versions, in such case we use this
                public void openFileChooser(ValueCallback uploadMsg, String acceptType){
                    module.setUploadMessage(uploadMsg);
                    mUploadMessage = uploadMsg;
                    Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                    i.addCategory(Intent.CATEGORY_OPENABLE);
                    i.setType("*/*");
                    module.getActivity().startActivityForResult(
                            Intent.createChooser(i, "File Browser"),
                            FCR);
                }
                //For Android 4.1+
                public void openFileChooser(ValueCallback uploadMsg, String acceptType, String capture){
                    module.setUploadMessage(uploadMsg);
                    mUploadMessage = uploadMsg;
                    Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                    i.addCategory(Intent.CATEGORY_OPENABLE);
                    i.setType("*/*");
                    module.getActivity().startActivityForResult(Intent.createChooser(i, "File Chooser"), FCR);
                }
                //For Android 5.0+
                public boolean onShowFileChooser(
                        WebView webView, ValueCallback<Uri[]> filePathCallback,
                        WebChromeClient.FileChooserParams fileChooserParams) {
                    module.setmUploadCallbackAboveL(filePathCallback);
                    /*if(mUMA != null){
                        mUMA.onReceiveValue(null);
                    }*/
                    if (module.grantPermissions()) {
                        module.uploadImage(filePathCallback);
                    }
                    return true;
                }
            });

        }
    }

    public void setPackage(AdvancedWebviewPackage aPackage){
        this.aPackage = aPackage;
    }

    public AdvancedWebviewPackage getPackage(){
        return this.aPackage;
    }
}
```
2. 添加`AdvancedWebviewPackage.java`
```java
public class AdvancedWebviewPackage implements ReactPackage {
    private AdvancedWebviewManager manager;
    private AdvancedWebviewModule module;

    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override public List createViewManagers(ReactApplicationContext reactContext) {
        manager = new AdvancedWebviewManager();
        manager.setPackage(this);
        return Arrays.<ViewManager>asList(manager);
    }

    @Override public List createNativeModules( ReactApplicationContext reactContext) {
        List modules = new ArrayList<>();
        module = new AdvancedWebviewModule(reactContext);
        module.setPackage(this);
        modules.add(module);
        return modules;
    }

    public AdvancedWebviewManager getManager(){
        return manager;
    }

    public AdvancedWebviewModule getModule(){
        return module;
    }
}
```

3. 在MainApplication.java中添加引用
```java
    @Override
        protected List<ReactPackage> getPackages() {
          return Arrays.<ReactPackage>asList(
                  new MainReactPackage(),
                  new AdvancedWebviewPackage(),
          );
        }
```
4. 在`WebView.android.js`中添加新的prop属性支持
```js
enabledUploadAndroid: PropTypes.bool,
...
enabledUploadAndroid={this.props.enabledUploadAndroid}
```