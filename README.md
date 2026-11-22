# HTML2APK · 手机端 HTML → APK 转换器完整安卓工程）

一个**不需要电脑**的安卓应用：手机上选一个 HTML 文件，在本机直接打包成一个可安装的 APK。
整个打包链路（二进制清单编码、ZIP 组装、v1/v2/v3 签名）全部是纯 Java，跑在手机里，不依赖 aapt2 / d8 / 服务器 / 网络。

---

## 目录结构

```
HTML2APK-Android/
├── app/
│   ├── AndroidManifest.xml           应用清单（包名 com.htmitoapk.A）
│   ├── res/
│   │   ├── values/strings.xml        中文应用名
│   │   ├── values-en/strings.xml     英文应用名
│   │   └── mipmap-xxhdpi/ic_launcher.png
│   └── assets/
│       ├── shell.dex                 网页外壳的 dex（2.3KB，所有生成的应用共用）
│       ├── key.pk8                   给生成的 APK 签名用的私钥（PKCS#8 DER）
│       └── cert.der                  对应证书（X.509 DER）
├── src/com/doubao/html2apk/
│   ├── ConverterActivity.java        主界面：选文件 / 填名称 / 生成 / 安装（中英双语）
│   ├── ApkFileProvider.java          极简 ContentProvider，把 APK 交给系统安装器
│   ├── core/MiniApkBuilder.java      ★ 核心引擎：AXML 编码 + ZIP 组装 + apksig 签名
│   └── shell/ShellActivity.java      生成出来的 APP 的 WebView 外壳
├── libs/apksigner.jar                apksig 签名库（纯 Java，随 App 一起打进 dex）
├── keys/app.p12                      本应用自身的签名证书（alias appkey，密码 android）
└── tools/build.sh                    无 Gradle 的构建脚本
```

---

## 核心原理

### 1. 为什么手机上能打包
普通 APK 需要 `aapt2`（有 native 依赖）+ `d8` + `javac`，手机上没有这些。
这里把它们各自的作用用纯 Java 替代：

| 原本由谁做 | 手机端替代方案 |
|---|---|
| aapt2 编译 AndroidManifest.xml 成二进制 | `MiniApkBuilder.buildManifest()` 手写 AXML：文件头 + UTF-16 字符串池 + 资源映射表 + 元素节点树 |
| aapt2 生成 resources.arsc | 不做资源，应用无图标（manifest 里不引用任何资源） |
| d8 生成 classes.dex | 预置一份 `shell.dex`，外壳类名固定为 `com.doubao.html2apk.shell.ShellActivity`，与包名解耦，任意包名都能复用 |
| apksigner 签名 | 内嵌 `com.android.apksig`（纯 Java），运行时直接对临时文件签名 |

### 2. 包名与类名解耦（关键设计）
生成的应用清单里：

```xml
package="用户填的包名"
<activity android:name="com.doubao.html2apk.shell.ShellActivity" ... />
```

`android:name` 写的是**全限定类名**，与 manifest 的 `package` 无关，
所以同一份 2.3KB 的 `shell.dex` 可以服务任意包名 / 任意应用名，这是"手机端无法编译 dex"却能生成 APK 的核心技巧。

### 3. 清单里用到的框架属性 ID
AXML 的属性必须带资源 ID，否则系统解析时会忽略。用到的 ID 取自 android-34 平台资源表：

```
attr/theme=0x01010000   attr/label=0x01010001   attr/icon=0x01010002
attr/name=0x01010003    attr/exported=0x01010010
attr/minSdkVersion=0x0101020c   attr/versionCode=0x0101021b
attr/versionName=0x0101021c     attr/targetSdkVersion=0x01010270
attr/hardwareAccelerated=0x010102d3   attr/usesCleartextTraffic=0x010104ec
style/Theme.NoTitleBar=0x01030006
```

### 4. 签名密钥
`app/assets/key.pk8` + `cert.der` 是**给生成的应用签名用**的固定密钥（DN `CN=HTML2APK OnDevice`）。
固定密钥的好处：同一包名重新生成可以覆盖升级，不会提示"签名不一致"。
它是调试性质的密钥，发布正式产品时应替换成自己的（见下方"替换密钥"）。

---

## 构建

### 方式一：命令行脚本（推荐）
需要 JDK 17 与 Android build-tools 34 + platform 34：

```bash
export JDK=/path/to/jdk-17
export SDK=/path/to/android-sdk          # 里面要有 build-tools/34.0.0 和 platforms/android-34/android.jar
bash tools/build.sh
# 产物：build/HTML2APK.apk
```

### 方式二：Android Studio
本工程是"平铺式"结构（`src/` + `app/`），没有 Gradle 配置。
如果要用 Android Studio，新建一个空工程，把 `src/`、`app/res`、`app/assets`、`app/AndroidManifest.xml`、
`libs/apksigner.jar` 拷进去，并在 `build.gradle` 加：

```gradle
android {
    namespace 'com.htmitoapk.A'
    compileSdk 34
    defaultConfig { minSdk 26; targetSdk 33 }
}
dependencies { implementation files('libs/apksigner.jar') }
```

> minSdk 用 **26**（Android 8.0）而不是 21：`apksig` 内部用到 `java.nio.file` / `java.time`，API 26 才有。

---

## 关键改动指引

### 换应用包名
改 `app/AndroidManifest.xml` 的 `package="..."`，同时把两处 provider authority 一起改：
- `app/AndroidManifest.xml` 里 `android:authorities`
- `ConverterActivity.install()` 里的 `content://` 前缀

（两处都改，否则安装时会出现 provider 冲突）。

### 替换给生成应用用的签名密钥
```bash
keytool -genkeypair -keystore my.p12 -storetype PKCS12 -alias mykey \
  -keyalg RSA -keysize 2048 -validity 20000 \
  -storepass android -keypass android -dname "CN=MyApp, O=Me, C=CN"

openssl pkcs12 -in my.p12 -passin pass:android -nodes -nocerts \
  | openssl pkcs8 -topk8 -nocrypt -outform DER -out app/assets/key.pk8
openssl x509 -in <(openssl pkcs12 -in my.p12 -passin pass:android -nokeys -clcerts) \
  -outform DER -out app/assets/cert.der
```

### 改生成应用的默认值
`MiniApkBuilder.buildManifest()` 的调用处（`convert()` 里）可改：
- `minSdk` / `targetSdk`（当前 21 / 33）
- `versionCode` / `versionName`
- 是否加 `INTERNET` 权限、是否 `usesCleartextTraffic`

### 给生成的应用加图标（当前不支持）
图标需要 `resources.arsc`，只有 aapt2 能正确生成，所以手机端刻意不引用图标，
生成的应用在桌面显示系统默认图标。要图标只能改用电脑端工具（aapt2 全流程）。

---

## 已知限制

1. **生成的应用无自定义图标**（原因见上）。
2. **包名不可重复**：同包名再次生成会覆盖安装（签名固定，可正常升级）。
3. **网页通过 `file://` 加载**：请求远程接口会遇到跨域限制，纯本地页面不受影响。
4. **本应用自身的包名含大写 `A`**：Android 允许，但应用商店普遍要求全小写。

---

## 版本
- v1.2 包名 `com.htmitoapk.A`
- 中英双语（跟随系统语言，也可在标题栏手动切换）


版本二:
# HTML2APK · 手机端 HTML → APK 转换器

一个不需要电脑的安卓应用：在手机上选一个 HTML 文件，直接打包成一个可安装的 APK。
整个打包链路（二进制清单编码、ZIP 组装、v1/v2/v3 签名）全部是纯 Java，全程离线，不依赖 aapt2 / d8 / 服务器 / 网络。

## 功能

- 选择 HTML 文件，填写应用名和包名，一键生成 APK 并直接安装
- 中英双语，跟随系统语言，也可在标题栏手动切换
- 生成的应用无需联网即可打开（HTML 内置在 APK 中）
- 不依赖 Gradle / Android SDK，手机端完成打包和签名

## 安全说明

本工程不包含任何私钥：

- 工具运行时按包名为每个应用生成独立的 RSA 2048 密钥和自签名证书，保存在应用私有目录
- 同一包名复用同一套密钥，重新生成仍可覆盖升级
- 不同包名、不同设备互不影响
- 构建本工具自身 APK 的签名密钥由构建者本地生成，不随仓库分发

## 构建

需要 JDK 17 与 Android build-tools 34 + platform 34：

```bash
export JDK=/path/to/jdk-17
export SDK=/path/to/android-sdk
bash tools/build.sh
# 产物：build/HTML2APK.apk
```

首次运行会自动生成构建签名密钥到 `keys/`（已加入 .gitignore）。
也可以设置 `KS_PATH` / `KS_ALIAS` / `KS_PASS` 使用自己的密钥。

## 已知限制

1. 生成的应用没有自定义图标（手机端无法生成 resources.arsc）
2. 同一包名的覆盖升级依赖本机保存的签名密钥，清除应用数据后需先卸载旧版
3. 网页通过 file:// 加载，请求远程接口会遇到跨域限制
4. 本应用自身的包名含大写 A，应用商店通常要求全小写

## 许可

MIT License，见 [LICENSE](LICENSE)。
`libs/apksigner.jar` 来自 Android Open Source Project，使用 Apache License 2.0。
