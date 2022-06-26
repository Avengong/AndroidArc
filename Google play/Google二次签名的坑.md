自己的签名 keystore.jks

上传apk到Google后，Google重新为你签名，生成新的签名信息。此时可以在Google开发者后台下载重新签名后的证书。

当需要填写第三方登录的时候，需要把Google重新签名的证书信息填入到第三方平台。 那么问题来了，如果我们还是在用自己的签名，那么假设第三方平台支持一套签名那么久没法登录了。如，微信。那怎么解决呢？

1. 直接把Google重新签名的证书改成我们自己的，相当于用我们的证书
2. 下载Google证书，然后我们本地开发都是使用Google的证书签名。 听起来很nice，可以试试。

# 操作方法

1. 从Google后台下载证书（设置-应用完整性），得到 ![img_2.png](img_2.png)
2. 通过命令将得到.cert转换成熟悉的jks/keystore格式

```
keytool -import -file deployment_cert.der -keystore deployment_cert.jks
```

3. 生成fb的 key hash

```
keytool -exportcert -keystore deployment_cert.jks | openssl sha1 -binary | openssl base64
```

得到格式： CkOkaKo6xQYOUIAADXwgZvVf0RQ= CkOkaKo6xQYOUIAADXwgZvVf0RQ=

4. 获取别名信息

```
keytool -list -v -keystore YOUR_RELEASE_KEY_PATH

```

# 验证

使用Google重新签名的证书，是不能签名的。只能提供给第三方开发平台验证。 还是需要用自己一开始创建的的jks来签名，这个跟Google后台的上传密钥是同一个。








