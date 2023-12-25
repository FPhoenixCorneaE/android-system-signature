# Android 系统签名

### 1. 系统应用(使用 AOSP 系统签名或厂商自定义 keystore 进行签名)开发场景

* 静默安装（android.permission.INSTALL_PACKAGES）
* 屏幕抓取（SurfaceControl#createDisplay）
* 设备音频抓取（AudioSource.REMOTE_SUBMIX）
* 应用外悬浮窗
* ...

### 2. 获得权限

应用想要获取系统级权限有两种方法：

* (1) 设备 root
* (2) 在 AndroidManifest.xml 中根节点指定 `sharedUserId` :

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" android:sharedUserId="android.uid.system">
    ...
</manifest>
```

> 简单解释下sharedUserId这个属性，通过设置同一个User id的使得多个应用可以运行在同一个进程中。而将sharedUserId设置成android.uid.system则可以将该应用和系统应用运行在同一进程中，于是乎便有了系统权限。

### 3. 下载相关文件

配置完 `android:sharedUserId="android.uid.system"` 之后，此时的 app 是无法成功安装到设备的，控制台会提示 `INSTALL_FAILED_SHARED_USER_INCOMPATIBLE`，这是因为此时 app 已经被识别为系统应用，但是其签名信息却不是系统签名，于是无法通过系统检验。
**使用同一个 `sharedUserId` 的应用，需要使用同一个签名文件。**

进行系统签名需要准备好如下几个文件：

* **platform.pk8**：签名证书
* **platform.x509.pem**：签名证书
* **signapk.jar**：签名工具

在 google 的 git 上我们可以拿到我们想要的东西
[https://android.googlesource.com/platform/build/+/donut-release/target/product/security/](https://android.googlesource.com/platform/build/+/donut-release/target/product/security/)

> 注：如果是 Android 系统厂商合作开发 这种场景（自定义过系统签名），那么以上文件应该让合作厂商提供。

除此之外，还需要 keytool 工具：
[https://github.com/getfatday/keytool-importkeypair](https://github.com/getfatday/keytool-importkeypair)

### 4. 生成 .jks 签名文件

> keytool-importkeypair 是 shell 脚本，在 Unix 系统下可以直接运行，但是在 Windows 系统下（cmd 或 PowerShell）是无法直接运行的，这时可以借助 Git Bash 来执行该命令。

* a. 在项目根目录下新建文件夹 `security`，并将签名工具都放到该文件夹下
* b. 在该文件夹下新建 `signature.sh` 脚本文件，方便直接生成签名 编写 signature.sh 文件
  ###### 语法：
  ###### keytool-importkeypair [-k keystore] [-p storepass] [-pk8 pk8] [-cert cert] [-alias key_alias]
  ###### 示例：
  `./keytool-importkeypair -k system-signature.jks -p 123456 -pk8 platform.pk8 -cert platform.x509.pem -alias FPhoenixCorneaE`

> 注：system-signature.jks 是生成签名文件的名称，123456 是签名的密码，FPhoenixCorneaE 是签名的别名，
> 如果是在 windows 下的话，双击 `signature.sh` 便可以得到签名文件了。

### 5. 在 Android Studio 中使用

```groovy
signingConfigs {
    system {
        storeFile file('../security/system-signature.jks')
        storePassword "123456"
        keyAlias "FPhoenixCorneaE"
        keyPassword "123456"
    }
}
buildTypes {
    release {
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        signingConfig signingConfigs.system
    }
    debug {
        // 让当前 buildType（debug）复制其他 buildType（release）的配置，减少相同配置的代码量。虽然这很方便，但是一定要注意，
        // 如果是 debug 构建类型，一定要指定其 debuggable 为 true（因为 release 的 debuggable 默认为 false）
        initWith(buildTypes.release)
        debuggable true
        minifyEnabled false
    }
}
```
# android-system-signature
