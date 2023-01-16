---
title: 【flutter 起步走】Flutter 使用api 上传图片到阿里云OSS 
---
> ### "不就是上传一张图片吗？能有多难？"
>
> 在我需要通过api在flutter中上传我的博客图片到阿里云的时候，我轻视地想着。
>
> 然后时间不知不觉就跳到了两天后~~



# 前言:为什么花了么长的时间？

一部分花在了找文档看文档，找可行方案。

另一部分花在了签名尝试。(签名校验真复杂)

再一部分花在了putObject和postObject方法的反复尝试。

阳光总在风雨后，曲折的道路阿终于还是成功了~



# 一.找方案踩坑

#### 第一步，首先[去官网](https://help.aliyun.com/document_detail/52834.html?spm=a2c4g.11186623.6.908.10287a3390S9fL)找阿里云oss有没有相关的可用sdk。

到网站找一找，不出意料，flutter这种还没啥人用的小玩意，这个时间点阿里还是没有出sdk。



#### 第二步，找到api请求上传的签名校验方式以及参数api

[签名校验官网说明](https://help.aliyun.com/document_detail/31950.html?spm=a2c4g.11186623.6.1595.7633c250ETf03e)

[Header添加签名的具体生成方式说明](https://help.aliyun.com/document_detail/31951.html?spm=a2c4g.11186623.6.1596.5bb67655W19fZe)

Api 地址:https://oss-example.oss-cn-hangzhou.aliyuncs.com

> oss-example使用自己的bucket名称代替



#### 第三步，找到可以成功的图片上传api

我最初找到的是putObject方法([文档](https://help.aliyun.com/document_detail/31978.html?spm=a2c4g.11186623.6.1673.631ec250fsEZCC))，在参考官方的java代码，终于在header添加可用的签名后，却老是报错xml 参数错误啥的。因为我明明没有传xml参数，只是`put`了`image/jpg`所以十分不解，一脸困惑。

**最终尝试使用postObject(****[文档](https://help.aliyun.com/document_detail/31988.html?spm=a2c4g.11186623.6.1681.68d83ea7O6mJV4)****)方法成功了。**



# 二.可行的详细路线

- #### 第一反应，如果是开发的非flutter 平台可以尝试找找官方sdk

[sdk列表链接](https://help.aliyun.com/document_detail/52834.html?spm=a2c4g.11186623.6.908.10287a3390S9fL)

- #### 第二步骤，然后开始参照官方签名文档开始签名。

官方说明的伪代码:

```
///Authorization字段计算的方法
Authorization = "OSS " + AccessKeyId + ":" + Signature
Signature = base64(hmac-sha1(AccessKeySecret,
            VERB + "\n"
            + Content-MD5 + "\n" 
            + Content-Type + "\n" 
            + Date + "\n" 
            + CanonicalizedOSSHeaders
            + CanonicalizedResource))
```

实际编写出来的dart代码:

```dart
class OssUtils {
  static var f = DateFormat('EEE, dd MMM yyyy HH:mm:ss');
  ///获取签名字符串
  static String buildSignature(String httpMethod, String resourcePath, RequestOptions request) {
    var Date = f.format(DateTime.now().toUtc()) + " GMT";
    request.headers['Date']=Date;
    var CanonicalizedOSSHeaders = buildCanonicalizedOSSHeaders(request);
    var CanonicalizedResource = resourcePath;
    final headers = request.headers.map((key, value) => MapEntry(key.toLowerCase(), value));
    var string = StringBuffer()..writeln(httpMethod);
    //有md5 包含md5
    if (headers.containsKey(HttpHeaders.contentMD5Header)) {
      string.write(headers[HttpHeaders.contentMD5Header]);
    }
    string.writeln();
    //有 type包含type
    if (headers.containsKey(HttpHeaders.contentTypeHeader)) {
      string.write(headers[HttpHeaders.contentTypeHeader]);
    }
    string.writeln();
    string
      ..writeln(Date)
      ..write(CanonicalizedOSSHeaders)
      ..write(CanonicalizedResource);
    var Signature = base64.encode(Hmac(sha1, AccessKeySecret.toBytes()).convert(string.toString().toBytes()).bytes);
    var Authorization = "OSS " + AccessKeyId + ":" + Signature;
    return Authorization;
  }

  static String buildCanonicalizedOSSHeaders(RequestOptions request) {
    final canonicalOssHeadersString = StringBuffer();
    final headers = request.headers;
    final canonicalizedOssHeadersToSign = SplayTreeMap<String, String>();
    headers.forEach((key, value) {
      String lowerKey = key.toLowerCase();
      if (lowerKey.startsWith("x-oss-")) {
        canonicalizedOssHeadersToSign[lowerKey] = value.toString().trim();
      }
    });

    // Append canonicalized oss headers to sign to canonical string
    canonicalizedOssHeadersToSign.forEach((key, value) {
      canonicalOssHeadersString
        ..write(key)
        ..write(':')
        ..writeln(value);
    });
    return canonicalOssHeadersString.toString();
  }
}
```

1. 最后在dio拦截器中调用生成签名方法，添加签名到Headers中

```dart
_ossdio!.interceptors.add(InterceptorsWrapper(
    onRequest: (options, handler) {
      var Authorization = OssUtils.buildSignature(options.method, "/guuguohome/", options);
      options.headers["Authorization"] = Authorization;
      handler.next(options);
    },
    onResponse: _onResponse));
```

- #### 第三接下来，使用 postObject 上传图片([官网文档](https://help.aliyun.com/document_detail/31988.html?spm=a2c4g.11186623.6.1681.68d83ea7O6mJV4))

```dart
Future<String> uploadToOss(File image) async {
  var host = "${bucket}.oss-cn-hangzhou.aliyuncs.com";
  var url = "https://${host}";
  ///文件名，后续作为key使用
  var fileName = path.split(image.path).last;
  var policyData = {
    "expiration": "2314-12-01T12:00:00.000Z", //必填项，随便写了个一百年后
    "conditions": [
      {"bucket": bucket},
    ]
  };
  var policy = base64.encode(json.encode(policyData).toBytes());
  var signature = base64.encode(Hmac(sha1, AccessKeySecret.toBytes()).convert(policy.toBytes()).bytes);
  final data = FormData.fromMap({
    "file": await MultipartFile.fromFile(
      image.path, //图片路径
      filename: fileName, //图片名称
    ),
    "key": fileName,
    "policy": policy,  //必须要有
    "Signature": signature,  //必须要有
    "OSSAccessKeyId": AccessKeyId,
  });

  ///参考 https://help.aliyun.com/document_detail/31978.html?spm=a2c4g.11186623.6.1673.50d8c2502BbrCV
  Options? options = Options(headers: {
    "x-oss-storage-class": "IA", //低频存储
    "Host": host,
    "Content-Encoding": "utf-8",
  });
  final res = await getOssDio().post(url, options: options, data: data);
  return url + "/" + fileName;

}
```

- #### 第四最后，上传大功告成了

去阿里云oss控制台 可以看到 bucket中已经多出了一个图片文件。

可喜可贺，可喜可贺



# 三.查看图片

上传图片后，自然需要拿到访问图片的url链接才能访问。

这时候有两种情况

#### 第一种很简单:公共读的bucket

> 图片要对外展示， 保证 bucket 权限设置为公共读

![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_438007d51a544003b49f2bd9264cc540~tplv-k3u1fbpfcp-zoom-1.image.jpg)

在公共读权限下，查看图片就变得简单，只需要使用key拼接好url链接就行了。

```dart
//拼接规则
var url="https://${bucket}.oss-cn-hangzhou.aliyuncs.com/${objectkey}"
//例子：https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-05dbfb795c3c84e2.jpg
```

#### 第二种[参考文档](https://help.aliyun.com/document_detail/39607.html?spm=a2c4g.11186623.6.665.5be81c913hPcmC)获取链接

[看文档看文档看文档](https://help.aliyun.com/document_detail/39607.html?spm=a2c4g.11186623.6.665.5be81c913hPcmC)



# 四.结尾

> ### "不就是上传一张图片吗？能有多难？"
>
> > 看着终于成功上传的图片，我小声呢喃。