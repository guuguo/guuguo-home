---
title: 读光sdk 设计思路
---
# 重点类
- `SailOcrSdk` :对外api类
- `IOcrType` :Ocr类型
- `ISailFingerOcr` :指尖能力接口
- `ISailLensOcr` :图像能力接口
- `ISailScanOcr` :通用识别能力接口

# 流程分析
## 一, 初始化sdk
调用`SailOcrSdk.init` 完成初始化工作。经历以下几步
### 1. 设备信息获取，完善鉴权信息

```java
//完善设备签名信息
addDeviceSign();
//设备鉴权的时候需要开启文件读写权限
private static void addDeviceSign() {
    String deviceId = DEFAULT_DEVICE_ID;
    String appId = SignUtil.getPackageName(context) == null ? "" : SignUtil.getPackageName(context);
    String appSignature = SignUtil.getAppSignatureMD5(context, appId) == null ? "" : SignUtil.getAppSignatureMD5(context, appId);
    appSignature = appSignature.replace(":", "").toLowerCase();

    //移动设备（手机）传真正的deviceId，否则传默认的
    if (authInfoStore.authType == AuthTypeEnum.AUTH_TYPE_MOBILE) {
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.M) {
            if (context.checkSelfPermission(Manifest.permission.READ_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED ||
                    context.checkSelfPermission(Manifest.permission.WRITE_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED) {
                throw new RuntimeException("请开启文件读写权限");
            }
        }
        deviceId = DeviceIdUtil.getDeviceId(context);
    }
    authInfoStore.deviceId = deviceId;
    authInfoStore.appSignature = appSignature;
    authInfoStore.appId = appId;
}
```
### 2. 初始化低阶sdk 

```java
private static void initSdk(SailOcrConfig sailOcrConfig, int abilityFlag) throws AuthenticatorException {
    ...
    if (!isInit()) {
        System.loadLibrary("mind-duguang");
        ModelConfig config = new ModelConfig
                .Builder(context)
                .setOutPutPath(getModelOutputPath(context))
                .configJson("models/sailModelConfig.json")
                .build();
        ModelManagerHolder.init(config);

        ModelManagerHolder.get().loadAllModelsSync((code, err, msg) -> {
            if (code == ModelConstant.LOAD_SUCCESS) ScanLog.debug("module", "load model done");
            else ScanLog.debug("module", "load model fail");
        });
    }
    //初始化所有flag配置的sdk能力
    int[] flagsArray = {FLAG_OCR_ID_CARD_FRONT, FLAG_OCR_ID_CARD_BACK,FLAG_OCR_ID_CARD_FRONT_SIMPLE, FLAG_OCR_ID_CARD_BACK_SIMPLE, FLAG_OCR_BANK, FLAG_OCR_MOBILE, FLAG_OCR_PHONE_SHEET, FLAG_OCR_FINGER, FLAG_OCR_LENS};
    IOcrType[] typeArray = {OcrTypeCommon.idCardFront, OcrTypeCommon.idCardBack,OcrTypeCommon.idCardFrontSimple, OcrTypeCommon.idCardBackSimple, OcrTypeCommon.bankCard, OcrTypeCommon.mobile, OcrTypeCommon.phoneSheet, OcrTypeFinger.handGesture, OcrTypeLens.lens};
    for (int i = 0; i < flagsArray.length; i++) {
        if ((abilityFlag & flagsArray[i]) != 0) {
            ScanLog.debug("ocrSdk", typeArray[i].getName());
            //使用反射初始化sdk接口，调用 IOcrType的initInterface 初始化接口
            DuguangErrCode errCode = typeArray[i].initInterface(authInfoStore, gestureConfig,ocrConfig);
            checkErrorCode(errCode);
        }
    }
    init = true;
}
```
以 OcrTypeCommon 为例，用反射的方式，避免强依赖，让具体的初始化代码可以被无缝剥离。初始化代码如下：
```java
@Override
public DuguangErrCode initInterface(AuthInfoStore authInfoStore, GestureConfig config, OcrConfig ocrConfig) {
    try {
        Class<?> threadClazz = Class.forName("com.alibaba.sail.duguang.ocr.ocr.OcrImageDealer");
        Method method = threadClazz.getMethod("initInterface", OcrTypeCommon.class, AuthInfoStore.class, OcrConfig.class);
        return (DuguangErrCode) method.invoke(null, this, authInfoStore,ocrConfig);
    } catch (Exception e) {
        e.printStackTrace();
        return DuguangErrCode.D_ERR_NULL_POINTER;
    }
}
```
具体的实现在`OcrImageDealer``.``initInterface`。传入配置参数，完成低阶sdk的初始化工作。
该代码只在编译类型包含通用ocr识别的时候，才会被编译在内。
初始化低阶sdk过程分为以下几步：

- 判断本地模型是否存在
- 创建识别能力接口实例
- 初始化本地识别接口(鉴权，配置信息等)
## 二, 获取sdk能力功能类
能力功能类也是通过SailOcrSdk 获取，可以调用如 `SailOcrSdk.phoneSheet()` 获取面单识别功能实例。
```java
private static <T> T getInterface(IOcrType info) {
    try {
        Class<?> clazz = info.getOcrClass();
        Method method = clazz.getMethod("get", IOcrType.class);
        return (T) method.invoke(null, info);
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}
```
通过`IOcrType`类型获取相关的`ocr`功能类
利用`IOcrType`内写好的功能类信息，调用其get方法获取实例。getOcrClass代码如下：

```java
@Override
public Class getOcrClass() throws ClassNotFoundException {
    return Class.forName("com.alibaba.sail.duguang.ocr.ocr.SailScanOCR");
}
```
最终是通过get方法获取实例：
```java
/**使用反射在 SailOcrSdk 中调用*/
public static SailScanOCR get(IOcrType ocrType) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
    Method builderMethod = Builder.class.getMethod(((OcrTypeCommon)ocrType).name());
    Builder object= (Builder) builderMethod.invoke(null);
    return object.build();
}
```
这里使用Builder的同名方法获取，当然建议直接创建实例返回(兼容旧的使用方式)
```java
public static Builder<String> phoneSheet() {
    return new Builder<>(OcrTypeCommon.phoneSheet, String.class);
}
```
这样就成功获取了`ISailScanOcr` 功能实例
## 三, 使用识别能力
通过获取的`ISailScanOcr`作为入口，直接调用其`recognize`方法进行识别，用法示例如下：
```java
///获取ocr 功能实例
ISailScanOcr ocr=SailOcrSdk.idCardBackSimple();
//识别身份证(其他也一样)
ImageResult res = ocr.recognize(request, roi);
///对识别结果进行操作
idCardBack.postValue(res.data);
```
卡证识别调用了`SailScanOcr#recognize`，现在看看recognize到结果的实现：
```java
@Override
public ImageResult<T> recognize(ImageRequest request, ArrayList<Integer> roi) {
    ImageResult<T> result = null;
    OcrTypeCommon info = (OcrTypeCommon) scanType;
    result = OcrImageDealer.get().recognize(request, info, roi, type);
    return result;
}
```
最终实现是在`OcrImageDealer``#``recognize`
```java
/**
 * @param roi 识别图片区域 左上宽高
 */
public <T> ImageResult<T> recognize(ImageRequest request, OcrTypeCommon info, ArrayList<Integer> roi, Class<T> type) {
    long start = System.currentTimeMillis();
    ImageResult<T> result = new ImageResult<>();
    DuguangOcrInterface ocrInterface = getInterface(info);
    if (ocrInterface == null) {
        result.err = "模型初始化失败";
        return result;
    }
    RecognizeResult recognizeResult = ocrInterface
            .recognizeVideo(request.data, request.width, request.height, request.format, 0, roi);
    result.id = request.id;
    result.resultByte = recognizeResult.image;
    result.width = roi.get(2);
    result.height = roi.get(3);
    result.data = analyzeRes(recognizeResult, type);
    result.err = recognizeResult.error;
    result.time = System.currentTimeMillis() - start;
    return result;
}
```
这一块实现是直接调用低阶sdk的识别方法，最后将低阶sdk返回的通用结果，经过反射，映射成具体的结果实体类，然后返回给用户。
# 按需编译
这一块设计以SailOcrSdk 为入口，对外提供了 
- 鉴权信息等初始化
- 获取ocr能力类

最终各种能力的识别是由ocr能力类来具体实现。
不同的 ocr 能力以及ocr模型通过文件夹区分，文件夹结构如下。
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MjY3N2IzZTJkY2M4NmE1YTgyYzVkZGEyODgyYWZlODhfOE92bUZtQWg1RXptaGl0dTVrUU5kdklLSUJwWEdJWmxfVG9rZW46Ym94Y25yODFyTUtCYVlVbEhXc2dEb0tFQ0NjXzE2MjkyNjg5NjM6MTYyOTI3MjU2M19WNA)
编译的时候分为以下几步操作：
- 按照定义的sdk类型拷贝模型
```groovy
preBuild.dependsOn(copyModelFiles)
task copyModelFiles() {
    dependsOn 'cleanTempFiles'
    doLast {
        atomList = atomList.replace(" ", "")
        println("需要拷贝的模型个数:" + list.size())
        def intoBasePath = "${project.projectDir.path}/src/main/assets/models"
        list.toList().forEach { file ->
            if (file.startsWith('ocr')) {
                def currentType = file.replace('ocr-', '')
                def fromPath = "${project.projectDir.path}/model/ocr/$currentType"
                copy {
                    from(fromPath)
                    into("$intoBasePath/$currentType")
                }
            } else {
                def fromPath = "${project.projectDir.path}/model/$file"
                copy {
                    from(fromPath)
                    into("$intoBasePath/$file")
                }
            }
        }

    }
}
```
- 手动执行`createConfig` gradle任务(在编译类型改变的时候可以执行)，创建模型信息描述文件：`sailModelConfig.json`。该任务定义在` module_ocr/createModelConfigJson.gradle `中。
- 在sourceSets中根据不同的编译类型，编译不同的源码文件夹
关键代码如下：
```groovy
sourceSets {
    main {
        def javaPaths = ["src/main/java"]
        def assetsPaths = ["src/main/assets"]

        def javaGraphCheck = "src/main/javaGraphcheck"
        def javaFingerDetect = "src/main/javaFingerdetect"
        def javaOcr = "src/main/javaOcr"

        if (gradle.ext.anyAtomStartWith('ocr-')) {
            javaPaths.add(javaOcr)
        }
        if (gradle.ext.hasAtom('Lens')) {
            javaPaths.add(javaGraphCheck)
        }
        if (gradle.ext.hasAtom('FingerDetect')) {
            javaPaths.add(javaFingerDetect)
        }
        java.srcDirs = javaPaths
        assets.srcDirs = assetsPaths
    }
}
```
`anyAtomStarWith` 和 `hasAtom`方法定义在`setting.gradle` 中，通过`gradle.ext `共享。
定义方式如下：
```groovy
def hasAtom = { it ->
    println "check hasAtom $it in $list"
    return list.contains(it)
}
def anyAtomStartWith = { start ->
    return list.any {
        println "check anyAtom $it star with $start"
        return it.startsWith(start)
    }
}
```
# 最后
通过`SailOcrConfig` 作为对外中央入口。
内部通过
- 反射
- 分文件夹按需依赖源码
- 预编译拷贝模型文件
等方式完成按需编译解耦对应ocr能力模块。
就是OCR Sdk 的设计整体思路了。