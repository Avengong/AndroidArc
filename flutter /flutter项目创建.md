# 一、创建项目

1、创建Flutter项目

```
flutter create 项目名字
flutter create --org com.example 项目名字
flutter create -i <objc或者swift> -a <kotlin或者java> 项目名字
flutter create -i <objc或者swift> -a <kotlin或者java> --org com.example 项目名字
flutter create --sample widgets.SliverFillRemaining.1  wigsfr1
flutter create --sample widgets.Navigator.1  wigsfr1
flutter create --sample widgets.SliverFillRemaining.2  wigsfr1
flutter create --sample widgets.SliverFillRemaining.3  wigsfr1
flutter create --sample widgets.SliverFillRemaining.4  wigsfr1
```

--org表示指定bundleId或者包名 -i 和 -a 表示设置语言（iOS默认是swift，android默认是kottlin） --sample表示创建示例文档

2、创建Flutter组件包 如果要在项目的某个目录下创建一个模块，需要先进入这个目录

```
flutter create -t module --org com.example 组件名字
module表示要创建的是一个组件而不是完整的app
```

3、创建插件包

```
flutter create --template=plugin  --org com.example --platforms android,ios 插件名字
flutter create -i objc -a java  --template=plugin  --org com.example --platforms android,ios 插件名字
```

--template=plugin表示创建的是跟原生有交互的插件 --platforms表示指定平台

4、创建Dart包

```
flutter create  --template=package 插件名字
--template=package表示创建的是纯dart语言的插件
```

注意：--template=package，等号两边不能有空格，纯Dart库是不会自动创建example项目的，但可以在库文件夹里自己创建一个example项目
然后在pubspec.yaml通过路径引用

# 二、打包

> https://flutter.cn/docs/deployment/android

## aab

```
flutter build appbundle

flutter build appbundle --build-name=${versionName} --build-number=${BUILD_NUMBER} --no-shrink --release -v --flavor aab

flutter build appbundle --build-name=Breathing --build-number=1 --no-shrink --release -v
```

## apk

```__
 flutter build apk --split-per-abi
 
 flutter build apk --build-name=${versionName} --build-number=${BUILD_NUMBER} --release -v --flavor apk
 flutter build apk --build-name=Breathing --build-number=1 --release -v --flavor apk


```

## 检测违规 apk

```
java -jar ${WORKSPACE}/client/checkWgSdk.jar apkPath ${WORKSPACE}/client/black_sugar_app/build/app/outputs/apk/release/${APP_NAME}.V${versionName}.${BUILD_NUMBER}.apk


```