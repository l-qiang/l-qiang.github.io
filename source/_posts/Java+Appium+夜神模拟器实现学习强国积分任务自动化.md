---
title: 'Java + Appium + 夜神模拟器实现学习强国积分任务自动化'
date: 2020-01-15 19:34:00
categories:
 - 其他
tags:
 - Java
 - Appium
 - 自动化测试
---

首先，先放源码[LitterBaby](https://github.com/l-qiang/LitterBaby)。

我想装了《学习强国》App的同学都为每天30积分的任务苦恼过。我之前也是因为这30积分非常头疼，不是《学习强国》App不好，而是真的不适合我。我们会学习，但可能不是被迫学习指定内容，为了指标而学习。



GitHub上面有很多Python写的《学习强国》自动化学习项目。我也有在用，但是我想有一个Java编写的，自己改起来顺手的《学习强国》自动学习。



为此，我就写了一个Java的，目前已经实现刷30分。

<!-- more -->

## 环境准备

1. [JDK8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

2. [Android SDK 24.4.1](https://www.androiddevtools.cn/)

3. [Appium 1.15.1](http://appium.io/)

4. [夜神模拟器 6.5.0.3](https://www.yeshen.com/)，当然里面还需要安装《学习强国》

5. [Sping Tool Suite](https://spring.io/tools)

6. [Lombok](https://projectlombok.org/setup/eclipse)

   （1、5、6我电脑上本来就已经装了，所以我就只用装2、3、4就行了

   安装基本都比较顺利，网上教程也挺多。）

{% note info %}

上面的环境准备好之后，启动`夜神模拟器`后，命令行执行下列命令

```
adb devices
```

如果`List of devices attached`没有显示内容

```
List of devices attached
127.0.0.1:62002 device
```

那么，需要使用SDK下`platform-tools`文件夹的**adb.exe**覆盖掉夜神模拟器安装目录下`bin`目录中的**adb.exe**和**nox_adb.exe**

如果做了上面的操作还是不行，试试以下指令。还是不行的话，看看adb的[用户指南](https://developer.android.google.cn/studio/command-line/adb?hl=zh-cn)。

```
adb connect device_ip_address
```

我这里夜神模拟器的`device_ip_address`是`127.0.0.1:62001`

{% endnote %}

##  Appium

[LitterBaby](https://github.com/l-qiang/LitterBaby)的核心就是`Appium`了。

Appium是客户端/服务器架构。我们下载的[Appium Desktop](http://appium.io/)就是服务器。而[LitterBaby](https://github.com/l-qiang/LitterBaby)做的就是使用Appium的Java客户端库定义一些操作来完成我的目标。

我没有去了解太多关于Appium的知识。感兴趣的可以去官方文档查看[详细介绍](https://appium.io/docs/en/about-appium/intro/)。

### Appium Desktop

打开Appium Desktop之后，显示如下界面，我将Host设置为`localhost`了

{% asset_img Appium_start.png %}

在{% button #, Start Server%} 之前我们需要，先点击下面的按钮配置JDK和Android SDK 的路径

然后启动，显示如下界面

{% asset_img Appium_cmd.png %}

点击放大镜。得到如下界面

{% asset_img Appium_conf.png %}

然后填入如下信息

```
{
  "platformName": "android",
  "version": "5.1.1",
  "appPackage": "cn.xuexi.android",
  "appActivity": "com.alibaba.android.rimet.biz.SplashActivity",
  "deviceName": "127.0.0.1:62001",
  "noReset": true,
  "newCommandTimeout": 600
}
```

点击{% button #, Start Session %}，就能得到如下界面。（在这之前得保证`adb devices`能看到你的设备，我这里是夜神模拟器）

{% asset_img Appium_op.png %}

这时候我们应该能看到`夜神模拟器`中的《学习强国》App也已经打开了。此时`Appium`和`夜神模拟器`看到的页面是同步的。以上界面对页面元素的操作按钮的所有操作都会反馈到夜神模拟器上的《学习强国》App。同样，在夜神模拟器上的操作也会反馈到上面的页面，点击`刷新`按钮就能刷新页面。



到此为止就准备好了，可以开发了。

## 开始开发

### 新建工程

用 [Sping Tool Suite](https://spring.io/tools)建一个Spring Boot工程

### 添加依赖

{% codeblock pom.xml %}

​	<dependency>
  			<groupId>io.appium</groupId>
  			<artifactId>java-client</artifactId>
  			<version>7.3.0</version>
​		</dependency>
​		<dependency>
​			<groupId>org.projectlombok</groupId>
​			<artifactId>lombok</artifactId>
​			<optional>true</optional>
​		</dependency>
​		<dependency>
​			<groupId>org.springframework.boot</groupId>
​			<artifactId>spring-boot-configuration-processor</artifactId>
​			<optional>true</optional>
​		</dependency>
​		<dependency>
​			<groupId>com.h2database</groupId>
​			<artifactId>h2</artifactId>
​		</dependency>
​		<dependency>
​			<groupId>org.springframework.boot</groupId>
​			<artifactId>spring-boot-starter-data-jpa</artifactId>
​		</dependency>
​		<dependency>
​			<groupId>org.springframework.boot</groupId>
​			<artifactId>spring-boot-starter-web</artifactId>
​	</dependency>

{% endcodeblock %}

主要还是Appium客户端的依赖，其他的都是为了方便完成学习强国的积分任务功能而添加的



上面这些完成了就可以开始写代码了。当然你可以直接clone这个[LitterBaby](https://github.com/l-qiang/LitterBaby)。

### 添加功能

下面我以《学习强国》里的`登录`和`阅读文章`为例，讲述怎么使用Appium的Java客户端库。

因为几乎所有的操作都是从`AndroidDriver`开始的，所以我们需要构建`AndroidDriver`

```
{
  "platformName": "android",
  "version": "5.1.1",
  "appPackage": "cn.xuexi.android",
  "appActivity": "com.alibaba.android.rimet.biz.SplashActivity",
  "deviceName": "127.0.0.1:62001",
  "noReset": true,
  "newCommandTimeout": 600
}
```

[LitterBaby](https://github.com/l-qiang/LitterBaby)中的代码如下：

{% codeblock AppiumConfig.java lang:java %}

@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public AndroidDriver<AndroidElement> androidDriver() throws Exception {
	DesiredCapabilities capabilities = new DesiredCapabilities();
	capabilities.setPlatform(Platform.ANDROID);
	capabilities.setVersion(version); // 安卓版本
	capabilities.setCapability(AndroidMobileCapabilityType.APP_PACKAGE, appPackage);
	capabilities.setCapability(AndroidMobileCapabilityType.APP_ACTIVITY, appActivity);
	capabilities.setCapability(MobileCapabilityType.DEVICE_NAME, deviceName); // adb devices 查看
	capabilities.setCapability("noReset", noReset); 
	capabilities.setCapability("newCommandTimeout", newCommandTimeout); 
	return new AndroidDriver<AndroidElement>(new URL(appiumServer), capabilities);
}

{% endcodeblock %}

#### 登录

`手机号`和`密码`登录就非常简单了。

1. 找到`手机号输入框`。我们在Appium Desktop上用选择元素的按钮选中`手机号输入框`。

   {% asset_img login_phone_input.png %}

   我们可以在右侧的`Selected Element`下看到`手机号输入框`的`XPath`，`Text`, `resource-id`等信息。

2. 使用`AndroidDriver`的`findElementByXPath`方法。(所有的元素我都用的XPath查找，我们可以在网上搜索XPath的教程来学习XPath，这个并不复杂)

   这里我使用以下XPath查找，意思就是查找所有元素中`resource-id`为`cn.xuexi.android:id/et_phone_input`的。

   ```
   //*[@resource-id='cn.xuexi.android:id/et_phone_input']
   ```

   {% note info %}

   所有的XPath都可以在Appium Desktop的搜索中按XPath搜索尝试能不能搜到元素。

   {% endnote %}

3. 然后调用第2步中`findElementByXPath`返回的`AndroidElement`对象的`sendKeys`方法，并传入手机号作为参数。

4. 同样的操作找到`密码输入框`，并输入密码。找到`登录`按钮，然后调用`click()`方法点击

到此为止，登录就实现了。

#### 阅读文章

阅读文章不外乎是这些操作

1. 找文章
2. 进入文章
3. 上下滑动阅读文章
4. 等待阅读到达指定时间
5. 返回

这5步中比较特别的就是`上下滑动`和`返回`了，其他的就跟`登录`差不多，都是先`findElementByXPath`找到元素然后调用`click()`方法。

##### 那么滑动怎么做呢? 

1. 我们需要创建一个新的对象`AndroidTouchAction`

{% codeblock DriverService.java lang:java  %}
androidTouchAction = new AndroidTouchAction(androidDriver);
{% endcodeblock %}

2. 获取设备的`宽`和`高`

{% codeblock DriverService.java lang:java  %}
val size = androidDriver.manage().window().getSize();
height = size.height;
width = size.width;
{% endcodeblock %}

3. 然后相继调用`AndroidTouchAction`的`press`，`moveTo`，`release`，`perform`方法

{% codeblock DriverService.java lang:java  %}
public void swipeUp() {
		androidTouchAction.press(PointOption.point(width / 2, height / 2 + 200))
						  .waitAction()
						  .moveTo(PointOption.point(width / 2, height / 2 - 400))
						  .release()
						  .perform();
}
{% endcodeblock %}



##### 返回操作

```
androidDriver.navigate().back();
```

{% note warning %}

**常见问题**

1. XPath对了但是找不到元素

   这种情况基本就是元素还没加载。我们可以用WebDriverWait对象等待

   ```
   wait = new WebDriverWait(androidDriver,30);
   wait.until(ExpectedConditions.visibilityOfElementLocated(By.xpath(xpath)));
   ```

2. 滑动的时候报不能在指定元素上执行touch操作

   解决办法也是跟上面类似，都是需要等待元素加载完成。

{% endnote %}

{% note info%}

**XPath的常用写法**

- 指定属性为\<xxx\>的元素，例

  ```
  //*[@text='<xxxx>']
  //*[@resource-id='<xxx>']
  ```

- 指定某个元素的父元素，例

  ```
  //*[@resource-id='cn.xuexi.android:id/general_card_title_id']/parent::*
  ```

  `resource-id`为`cn.xuexi.android:id/general_card_title_id`的元素的父元素

- 指定元素之后的同级元素

  ```
  //android.widget.ImageView/following-sibling::android.widget.TextView[ends-with(@text, "学习平台")]
  ```

  `android.widget.ImageView`元素之后的同级元素为`android.widget.TextView`的text内容以`学习平台`结尾的元素

- 指定元素之前的同级元素

  ```
  //*[@resource-id='cn.xuexi.android:id/action_bar_root']//android.widget.TextView[@text='分享到学习强国']/preceding-sibling::*
  ```

- 倒数第二个元素

  ```
  //android.widget.TextView[@text='欢迎发表你的观点']/following-sibling::*[last()-1]
  ```

  `android.widget.TextView`的text内容为`欢迎发表你的观点`的元素之后的同级元素中的倒数第二个元素

  [更多用法示例](https://github.com/l-qiang/LitterBaby/blob/master/src/main/resources/application.properties)

  

{% endnote %}








