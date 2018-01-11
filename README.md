### 说明

资源包插件（MiHomeResoucePlugin）是为米家 iOS 合作开发（原生语言嵌入开发）的厂商推出地插件系统，旨在分离代码与资源文件，提供资源热更新功能，同时为米家 App 瘦身，减小用户下载量。

### 插件申请

1. 在[米家开放平台](https://open.home.mi.com)登录自己的企业开发者账号； 
2. 进入 **产品管理** ，找到对应产品，点击进入。选择 **米家插件管理** 下的 **插件版本管理** ，选择 **ios插件** ，点击**创建创建**；
3. 插件类型选择 **ios资源包**，填入信息。创建插件的过程中需要填写一个插件包名，命名规则一般为：若开发者标识为 *aaa*，设备 model 为*aaa.bbb.v1*，则 iOS 插件包名一般为 *com.aaa.bbb.ios.iosr* 。例如，*xiaomi* 公司开发了一款火箭筒，设备 *model* 为 *xiaomi.rocketlauncher.v1*，则该火箭筒在米家iOS客户端中的插件包名为 *com.xiaomi.rocketlauncher.ios.iosr*

### 插件创建

1. 进入该项目根目录

2. 运行 createPlugin 脚本

   ```shell
   ./createPlugin plugin_name
   ```

   其中 plugin_name 为之前申请创建时的名称 com.aaa.bbb.ios.iosr

3. 插件创建后，会在 SDK 目录下，生成一个  plugin_name 目录。目录下包含：

   - `packageInfo.json` ，插件包信息文件
   - `config.plist`， 插件包配置文件，**开发者无需修改，不必关心**，随 SDK 更新
   - `Resources`目录，资源文件存放处

### Resources目录

资源文件存放处，将自己项目中所有的资源文件放入该目录，**不能有子目录**。

暂时支持的资源为图片类型，包含 `.png` `.jpg` `jpeg`。后续会根据需求增加音频，视频文件。

### packageInfo.json

该文件关系到插件包的打包与上传，请用文本编辑器打开并编辑其中内容：

```
{
 "package_name":"com.aaa.bbb.ios.iosr", // 插件包名，不用修改
 "developer_id":"123456789", // 企业开发者账号小米ID
 "models":"aaa.bbb.v1|your_device_model2", // 插件支持的设备 model，一个插件包可以支持多个model，用|分割。合作开发提供的代码支持哪些产品 model ，需要全部填入
 "asset_level":"1", // 与合作开发提供的原生代码中的 asset_level 对应，以1开始计数。具体更改规则见后续章节
 "version":"1", // 插件包的版本，每个上传的插件包都要不同且递增，每次上传新的插件包之前都需要修改
 "min_api_level":"1",//保留字段，默认为1，不用修改
 "platform":"iphone" // 插件包支持的平台，目前只支持iphone，不用修改
}
```

### 插件包 asset_level 的确定

为配合资源包插件，合作开发提供的代码中**，有且只有一处需要额外修改**：

在入口 ViewController.m 中，该 VC 继承自 MHDeviceViewControllerBase，添加方法：

```objective-c
+ (NSInteger)getResourcePluginAssetLevel{
    return 1; //该数值需与packageInfo.json 中的 asset_level 值相同
}
```

**米家 App 运行时，会根据该方法返回的值，去开放平台拉取相同值的插件包。** 即，代码与插件包由 asset_level 形成一一对应的关系：

1. 代码中未添加该方法，不会拉取资源包；
2. 后续代码更新，涉及图片资源变更，需更新资源包，提高资源包中的 asset_level 值，代码中该方法的返回值相应提高。不涉及图片资源更变，则无需更新资源包，代码该方法的返回值也不用更改；
3. 对于已经上线的版本，更新资源包，保持 asset_level 不变，则线上 App 会应用最新的对应 asset_level 值的资源包。保持资源名称不变，可实现图片替换，即资源的热更新。

### 插件的打包和签名

#### 准备插件签名文件

1. 生成属于米家开发者账号的 keystore 证书文件。 注意此文件 iOS 与 android 通用并且需要保持一致，如果已经开发过 android 插件，请使用当时生成的 keystore 文件，并跳过步骤1和2。

   ```shell
   keytool -genkey -dname CN=YourName,OU=YourCompany,O=YourCompany,L=Beijing,ST=Beijing,C=86 -alias yourKeyAlias -keypass 123456 -storepass 123456 -keystore ./your.keystore -validity 18000 -keyalg RSA -keysize 2048
   ```

2. 在小米 IoT 开发者平台，个人开发者选项中，填入公钥。即 keystore 文件的证书 MD5 指纹：

   ```shell
   keytool -list -v -keystore your.keystore
   ```

   取出其中的 MD5 指纹并去掉冒号。

3. 使用 keystore 文件按照下述流程提取出 iOS 能够识别的公钥和私钥文件。

4. 导出公钥文件 public.cer:（需要安装keytool）

   ```shell
   keytool -export -keystore your.keystore -alias yourKeyAlias -file public.cer
   ```

   其中 yourKeyAlias 与生成 keystore时的同名参数保持一致。如果是安卓生成的，可以通过下面的命令来查看设置的别名。

   ```shell
   keytool -list -keystore your.keystore
   ```

5. 导出私钥 pem 文件 private.pem: （需要 openSSL）

   ```shell
   keytool -importkeystore -srckeystore your.keystore -destkeystore private.pkcs -srcstoretype JKS -deststoretype PKCS12 -alias yourKeyAlias

   openssl pkcs12 -in private.pkcs -out private.pem
   ```

6. 保留好生成的 public.cer 以及 private.pem，插件包签名时将用到这两个文件。

#### 打包并给插件签名

1. 修改本地插件包的 packageInfo.json 一般是将上一次成功上传的 version + 1，并确定 **asset_level** 等其他信息是否填写正确。

2. 进入该项目根目录

3. 运行 packagePluginAndSign 脚本进行打包：

   ```shell
   python packagePluginAndSign plugin_name /path/to/private.pem /path/to/public.cer yourDeveloperId
   ```

   其中 plugin_name 是插件包的目录名，private.pem 和 public.cer 分别是准备好的私钥和公钥文件，yourDeveloperId 是开发此插件的米家开发者账号（数字小米ID）

4. 签名过程中会要求输入私钥文件的密码。

5. 打包成功后会在当前目录下生成 plugin_name.signed.zip 的已签名插件包。

6. 用开发者账号登录[米家开放平台](https://open.home.mi.com/)，在插件管理里选择相应的iOS插件，点击“上传插件包”进行上传。

7. 成功后点击该插件包的“白名单测试”，即可用 appstore 版本的客户端下载在白名单范围内下载到此插件进行测试。

### 插件的测试和发布

插件的测试和发布流程请联系米家的工作人员。
