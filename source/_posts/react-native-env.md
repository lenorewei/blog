---
title: react-native区分环境（安卓）
---

在项目中，会遇到一个问题就是项目的环境比较多包括测试环境，预发布环境和线上环境，而引用的热更新等sdk又需要对环境的不同进行不同的配置，这样就导致上线的时候要去修改很多个地方的配置，很容易导致问题，一旦修改错了地方，打出来的包上到线上后果就很严重了。
## 方法一： buildTypes
在之前的项目里面，我是参照[Article | 打造超溜666的ReactNative工作流](https://coderge.com/articles/201803/react-native-run.html)在`android/app/build.gradles`里面添加buildTypes方法，然后在里面添加相应的区分变量

参照的方法是
```java
    buildTypes {
            release_lastest {
                debuggable false
                signingConfig signingConfigs.config
            }
            release {
                debuggable false
                signingConfig signingConfigs.config
                buildConfigField "boolean", "IS_DEBUG", "false"
                buildConfigField "boolean", "IS_DEV", "false"
            }
            debugTest {
                debuggable false
                signingConfig signingConfigs.config
                buildConfigField "boolean", "IS_DEBUG", "true"
                buildConfigField "boolean", "IS_DEV", "false"
            }
            debug {
                debuggable true
                signingConfig signingConfigs.config
                buildConfigField "boolean", "IS_DEBUG", "true"
                buildConfigField "boolean", "IS_DEV", "false"
            }
        }
```

然后在java中就可以通过引`BuildConfig`类拿到对应的定义了
```java
    import *******.BuildConfig;
    String SERVER_HOST = BuildConfig.IS_DEBUG?"http://xx.xx.com":"https://xx.xx.com";
```   

需要在js中获取到对应的值的话首先需要`android/app/src/main/java/xxx/MainActivity.java`中进行对应的修改：
```java
    ...
    public class MainActivity extends ReactActivity {
        ...
        class MyReactDelegate extends ReactActivityDelegate {
            final Context context;
    
            public MyReactDelegate(Activity activity, @javax.annotation.Nullable String mainComponentName) {
                super(activity, mainComponentName);
                context = activity;
            }
    
            @javax.annotation.Nullable
            @Override
            protected Bundle getLaunchOptions() {
                Bundle bundle = new Bundle();
                bundle.putBoolean("isDebugMode", BuildConfig.IS_DEBUG);
                bundle.putBoolean("isDevMode", BuildConfig.IS_DEV);
                return bundle;
            }
        }
    }
```

然后就能在js的根元素中获取到对应的信息
```javascript
    ...
    class App extends Component {
      constructor (props) {
        super(props)
        console.log('isDebugMode', props.isDebugMode)
        console.log('isDevMode', props.isDevMode)
      }
      ...
    }
    ...
```
但这个方法的话，配置写在buildTypes里面，不够直观。需要更改的话也不是很方便，要改的地方也很多，而且只是针对安卓的。所以后来新建项目的时候就选择了用react-native-config来进行区分环境变量。
## 方法二： react-native-config
使用方法很简单
```bash
    $ yarn add react-native-config

    $ react-native link react-native-config
```

在`android/app/build.gradle`里添加

```java
    apply from: project(':react-native-config').projectDir.getPath() + "/dotenv.gradle"
```

然后就可以在根目录里面新建配置文件了如：

<img src="https://lenore-1254182071.cossh.myqcloud.com/blog/2018-08-30-025718.png" style="height: 300px;width: auto;" />

在`.env`中对所需要注入的配置项进行配置
```
    ENV=TEST
    PREFIX=******
    HOT_PATCH=******
    APP_VERSION_CODE=******
    APP_NAME=******
```
然后在js中就能通过引入Config拿到对应的配置值了
```javascript
    import Config from 'react-native-config'
    Config.ENV  
    Config.APP_NAME 
```
在安卓中的不同部分去取值稍稍会有些不一样

在应用下面与上面那个方法基本一样，也是引入BuildConfig类
```java
    import 你的包名.BuildConfig;
    
    ...
    if(BuildConfig.ENV == "TEST") {
    	...
    }
    ...
```
而在`android/app/build.gradle`中则需要通过`project.env.get(配置名)`来获取，需要注意的是所有传过来的类型都是字符串，如果你需要其他类型如数值等则需要对它进行转义，而且不要将如签名等敏感信息放到配置文件中
```java
    defaultConfig {
            applicationId project.env.get("APP_ID")
            minSdkVersion 22
            targetSdkVersion 26
            versionCode project.env.get("APP_VERSION_CODE").toInteger()
            versionName project.env.get("APP_VERSION_NAME")
            buildConfigField 'Integer','jsBundleVersionCode', project.env.get("APP_jsBundleVersionCode")
    			...
    }
```
如果需要在`android/app/src/main/AndroidManifest.xml`进行配置，则需要在`android/app/build.gradle`中添加`manifestPlaceholders`
```java
    defaultConfig {
            applicationId project.env.get("APP_ID")
            minSdkVersion 22
            targetSdkVersion 26
            versionCode project.env.get("APP_VERSION_CODE").toInteger()
            versionName project.env.get("APP_VERSION_NAME")
            buildConfigField 'Integer','jsBundleVersionCode', project.env.get("APP_jsBundleVersionCode")
    			...
    				manifestPlaceholders = [
                PUSH_NAME: project.env.get("PUSH_NAME")
            ]
    }
```
然后就可以在`AndroidManifest.xml`中通过
```java
    <activity
       ...
       android:label="${APP_NAME}"/>
```
去引用到了

我这边因为应用是安卓的所以没有对ios端去进行配置，react-native-config是支持ios端变量的配置的，当配置好变量了以后就可以通过设置不同的配置文件达到运行起不同环境，和打不同环境的包的目的了
```bash
    ENVFILE=.env.pre react-native run-android
    ENVFILE=.env.pro ./gradlew assembleRelease
```