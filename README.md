# Android原生接入RN模块包

## 创建RN模块
使用命令,创建RN模块

```
npm react-native init rn_module --version 0.62.2
```
然后我们将rn_module里面的android和ios目录删除，因为目前我们是要将RN集成到Native工程中，对于这俩目录是用不上的



## 为Android应用添加RN所需要的依赖

项目目录：

/Users/lgy/Documents/RNWorkspace/AndroidWithRNTest

文件结构如下：

```
|-AndroidWithRNTest  // 项目根目录
|----app //主模块
|----rn_module //rn模块
```

### 配置maven

到主模块的build.gradle里添加如下内容：

```
//rn
project.ext.react = [
        entryFile: "index.android.js",
        enableHermes: false
]

def enableHermes = project.ext.react.get("enableHermes", false);
def jscFlavor = 'org.webkit:android-jsc:+'
def safeExtGet(prop, fallback) {
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
}
//rn end
```

到主模块的build.gradle里的dependencies标签中配置依赖：

```
    //rn
    if (enableHermes) {
        def hermesPath = "../../rn_module/node_modules/hermesvm/android/";
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
    implementation "com.facebook.react:react-native:+" // From node_modules
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:1.0.0"
    //rn end
```

结果如下：

```
plugins {
    id 'com.android.application'
}

android {
    namespace 'com.lgy.androidwithrntest'
    compileSdk 32

    defaultConfig {
        applicationId "com.lgy.androidwithrntest"
        minSdk 21
        targetSdk 32
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

//rn
project.ext.react = [
        entryFile: "index.android.js",
        enableHermes: false
]

def enableHermes = project.ext.react.get("enableHermes", false);
def jscFlavor = 'org.webkit:android-jsc:+'
def safeExtGet(prop, fallback) {
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
}
//rn end



dependencies {

    implementation 'androidx.appcompat:appcompat:1.4.1'
    implementation 'com.google.android.material:material:1.5.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.3'
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'

    //rn
    if (enableHermes) {
        def hermesPath = "../../rn_module/node_modules/hermesvm/android/";
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
    implementation "com.facebook.react:react-native:+" // From node_modules
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:1.0.0"
    //rn end

}
```

然后，为项目配置使用的本地React Native maven目录。到项目根 目录下的build.gradle（即/Users/lgy/Documents/RNWorkspace/AndroidWithRNTest）配置

```
allprojects {
    repositories {
        google()
        mavenCentral()
        jcenter()
        //rn
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/rn_module/node_modules/react-native/android"
        }
        maven {
            // Android JSC is installed from npm
            url("$rootDir/rn_module/node_modules/jsc-android/dist")
        }
        //rn end
    }
}
```

如果你用的是gradle 6.8以上版本，这时候编译还是无法通过，需要到根目录下的setting.gradle将RepositoriesMode.FAIL_ON_PROJECT_REPOS改为RepositoriesMode.PREFER_PROJECT，如下：

```
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_PROJECT)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "AndroidWithRNTest"
include ':app'
```

这时候，项目编译成功了。

### 指定要ndk需要兼容的架构(重要)

Android不能同时加载多种架构的so库，现在很多Android第三方sdks对于abi的支持比较全，可能会包含armeabi、armeabi-v7a、x86、arm64-v8a、x86_64五种abi，如果不加限制直接引用会自动编译出支持5种abi的APK，而Android设备会从这些abi进行优先选择某一个，比如：arm64-v8a，但如果其他sdk不支持这个架构的abi的话就会出现crash.

到主模块的build.gradle下配置

```
/**
 * Set this to true to create two separate APKs instead of one:
 *   - An APK that only works on ARM devices
 *   - An APK that only works on x86 devices
 * The advantage is the size of the APK is reduced by about 4MB.
 * Upload all the APKs to the Play Store and people will download
 * the correct one based on the CPU architecture of their device.
 */
def enableSeparateBuildPerCPUArchitecture = false
```

和

```
    splits {
        abi {
            reset()
            enable enableSeparateBuildPerCPUArchitecture
            universalApk false  // If true, also generate a universal APK
            include "armeabi-v7a", "x86", "arm64-v8a", "x86_64"
        }
    }
```

具体如下：

```
plugins {
    id 'com.android.application'
}
/**
 * Set this to true to create two separate APKs instead of one:
 *   - An APK that only works on ARM devices
 *   - An APK that only works on x86 devices
 * The advantage is the size of the APK is reduced by about 4MB.
 * Upload all the APKs to the Play Store and people will download
 * the correct one based on the CPU architecture of their device.
 */
def enableSeparateBuildPerCPUArchitecture = false
android {
    namespace 'com.lgy.androidwithrntest'
    compileSdk 32

    defaultConfig {
        applicationId "com.lgy.androidwithrntest"
        minSdk 21
        targetSdk 32
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    splits {
        abi {
            reset()
            enable enableSeparateBuildPerCPUArchitecture
            universalApk false  // If true, also generate a universal APK
            include "armeabi-v7a", "x86", "arm64-v8a", "x86_64"
        }
    }
}

//rn
project.ext.react = [
        entryFile: "index.android.js",
        enableHermes: false
]

def enableHermes = project.ext.react.get("enableHermes", false);
def jscFlavor = 'org.webkit:android-jsc:+'
def safeExtGet(prop, fallback) {
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
}
//rn end



dependencies {

    implementation 'androidx.appcompat:appcompat:1.4.1'
    implementation 'com.google.android.material:material:1.5.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.3'
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'

    //rn
    if (enableHermes) {
        def hermesPath = "../../rn_module/node_modules/hermesvm/android/";
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
    implementation "com.facebook.react:react-native:+" // From node_modules
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:1.0.0"
    //rn end

}
```

到这里，不要忘了在app模块的清单文件里加入网络权限,否则，你会发现怎么也连不上服务器

```
<uses-permission android:name="android.permission.INTERNET" />
```

### 新建一个rn容器Activity

```
package com.lgy.androidwithrntest;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;

import com.facebook.react.ReactInstanceManager;
import com.facebook.react.ReactRootView;
import com.facebook.react.common.LifecycleState;
import com.facebook.react.shell.MainReactPackage;

public class LgyRNActivity extends AppCompatActivity {

    ReactRootView mReactRootView;
    ReactInstanceManager mReactInstanceManager;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        initRN();
        setContentView(mReactRootView);
    }

    private void initRN() {
        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setCurrentActivity(this)
                .setBundleAssetName("index.android.bundle")
                .setJSMainModulePath("index")
                .addPackage(new MainReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)//设置是否开启开发者模式
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager,"rn_module");
    }
}
```



等你跑起来，打开rn页面的时候，会报如下错误：

```
                                                                                                    java.lang.NullPointerException: Attempt to invoke interface method 'boolean com.facebook.react.common.SurfaceDelegate.isShowing()' on a null object reference

```

其实Android studio的logcat打印出一个提示,我用的React Native版本明明是0.62.2，怎么会是0.71.0-rc.0

```
onsole.error: React Native version mismatch.
                                                                                                    
                                                                                                    JavaScript version: 0.62.2
                                                                                                    Native version: 0.71.0-rc.0
```

所以需要限制版本为0.62.2。具体操作是到项目根目录下的build.gradle,加入如下内容：

```
   configurations.all {
        resolutionStrategy {
            // Remove this override in 0.65+, as a proper fix is included in react-native itself.
            force "com.facebook.react:react-native:0.62.2"
        }
    }
```

具体如下：

```
// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id 'com.android.application' version '7.3.1' apply false
    id 'com.android.library' version '7.3.1' apply false
}


allprojects {
    configurations.all {
        resolutionStrategy {
            // Remove this override in 0.65+, as a proper fix is included in react-native itself.
            force "com.facebook.react:react-native:0.62.2"
        }
    }
    repositories {
        google()
        mavenCentral()
        jcenter()
        //rn
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/rn_module/node_modules/react-native/android"
        }
        maven {
            // Android JSC is installed from npm
            url("$rootDir/rn_module/node_modules/jsc-android/dist")
        }
        //rn end


    }
}
```

上面这种写死版本的做法不太好，如果rn_module的package.json里设置的react-native版本变了，那我还要跑回来改，太麻烦了。有没有办法同步改变？

网上其实提供了一个方式,定义一个REACT_NATIVE_VERSION，通过执行脚步获取react-native/package.json里react-native的配置。

```
def REACT_NATIVE_VERSION = new File(['node', '--print',"JSON.parse(require('fs').readFileSync(require.resolve('react-native/package.json'), 'utf-8')).version"].execute(null, rootDir)).text.trim())

allprojects {
    configurations.all {
        resolutionStrategy {
            // Remove this override in 0.65+, as a proper fix is included in react-native itself.
            force "com.facebook.react:react-native:" + REACT_NATIVE_VERSION
        }
    }
}
```



但由于我把rn模块（rn_module）放到了AndroidWithRNTest目录下,所以不能用上面的定义，点击execute方法，跳进去可以看到execute(null, rootDir)中的rootDir其实是一个File类型，我只要把rn模块的根目录指定给他即可，代码如下：

```
def REACT_NATIVE_VERSION = new File(['node', '--print',"JSON.parse(require('fs').readFileSync(require.resolve('react-native/package.json'), 'utf-8')).version"].execute(null, new File("$rootDir/rn_module")).text.trim())

allprojects {
    configurations.all {
        resolutionStrategy {
            // Remove this override in 0.65+, as a proper fix is included in react-native itself.
            force "com.facebook.react:react-native:" + REACT_NATIVE_VERSION
        }
    }
}
```

这时候，就能正常跑起来了。