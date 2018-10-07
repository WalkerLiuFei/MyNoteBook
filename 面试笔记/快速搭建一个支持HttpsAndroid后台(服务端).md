---
title: 快速搭建一个支持Https的服务（附有Android访问Https链接的配置）
---

>本篇文章通过一个小Demo来介绍利用Spring Boot快速搭建一个Https的移动客户端后台服务和Client(安卓)端在配置自定义双向验证的一些配置

# Server端

>我准备用Spring Boot来示例这个这个Server端demo，众所周知SpringBoot作为Spring.io推出的杀手级框架不仅使得后端开发效率大幅度提升，也使得学习门槛降低。随着时间推移Spring Boot将会更多的应用到实践当中

## 框架的简单介绍

![CSDN图标](/assets/kuangjia.png "框架图")

Server端应用主要分为三个部分

+ java 文件夹下的源文件
+ resource文件夹下面的配置文件和资源文件
+ build.gradle配置文件

## 利用gradle构造你的工程
<pre>
group 'person.walker'
version '1.0-SNAPSHOT'


sourceCompatibility = 1.5
buildscript{
    repositories{
        mavenCentral()
    }
    //安装Spring-boot 插件！
    dependencies{
        classpath("org.springframework.boot:spring-boot-gradle-plugin:1.4.1.RELEASE")
    }
}
sourceCompatibility = 1.5
apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'spring-boot'

repositories {
    mavenCentral()
}

dependencies {
    //我们只需要下面两个dependence！
    compile("org.springframework.boot:spring-boot-starter")
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile group: 'junit', name: 'junit', version: '4.11'
}
</pre>


gradle配置文件需要注意两个地方(添加注解的地方)第一个是添加Spring-boot-gradle-plugin的依赖和jar包的依赖。这两个依赖足够我们来开发最最基本的Restful应用和javaWeb应用（注意这里最最基本，不涉及dao层(⊙﹏⊙)b）。。。


## 编写你的应用

### Application:应用入口类

第一步编写你的应用入口类

<pre>
@SpringBootApplication
public class Application {
    public static void main(String[] args){
		
        new SpringApplicationBuilder(Application.class)
                .run(args);
    }
}
</pre>

这里需要注意的的是**@SpringBootApplicaiton**这个注解，这个注解相当于自动完成了注解@CompontScan ，@Auto-Configuration等等注解的功能，真正的实现了SpringBoot开箱即用的功能，节省大量繁琐的操作

### 编写你的Controller控制器

<pre>
//Restful 服务的注解,方便快捷！
@RestController
public class MyControllers {
    @RequestMapping("/getInfo")
    public Person getInfo(){
        //返回一个Object类,其会被jackson 格式化为Json格式的字符串
        return new Person("walker","23");
    }
}
</pre>

其中Person类是个bean类。

<pre>
public class Person {

    private String age;
    private String name;

    public Person(String age, String name) {
        this.age = age;
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

}
</pre>

到这里基本上就可以运行的应用了，我们通过Applicaiton的main函数来跑一下，观察到控制台输出的信息

![lala]("/assets/embeddedtomcat.png" "console 输出")

图例标出的就是启动SpringBoot 内置Tomcat容器的信息，这可能会消除一些你的疑问，为什么没有部署到Tomcat这个应用就可以访问。

现在我们通过浏览器访问就可以访问了

## 配置你的SSL

利用keytool生成SSL证书可以参考下面的命令:

<pre>
keytool -genkey -alias walker_server -keyalg RSA -keystore walker_server.jks
</pre>

上面的命令的含义是生成一个 文件名叫做"walker_server.jks"的公匙文件，证书的别名为walker_server。加密算法指定为RSA。
另外在生成证书时一定要记牢设置的密码！<br>
拷贝你生成的XXX.jks文件到你工程的resource文件夹下面<br>
然后配置的application.properties添加以下代码

<pre>
# 将Https端口映射到8444 ，端口号可以根据你自己的喜好和系统环境配置
server.port = 8444
server.ssl.key-store = classpath:walker_server.jks  // 注意将jks文件改成你的jks文件名字
server.ssl.key-store-password = 3252860 //这里的password是你生成ssl证书时第一次输入的密码
server.ssl.key-password = 3252860   //这里的password是你生成ssl证书时第二次次输入的密码
</pre>
现在运行你的应用，打开浏览器访问"https://localhost:8444/getInfo"

![](/assets/httpsrequest.png)

如果你用过12306的话，你一定对这个界面不会感到陌生。为什么会出现这样的界面呐？原因是12306和我们现在配置的server都是自生成的SSL证书进行的双向验证，而一般的网站都是通过CA机构颁发的证书来进行的验证。两种验证方式具体的区别已经优缺点请自行Google.....
<br><br>
点击信任后看到的界面

![](/assets/brosucces.png)

<br><br><br><br>

# 附：Android端访问Https服务的配置

>Android这边对于Https链接的访问一般都是正常的，不需要什么特殊配置。但是前提是这个Https的SSL证书第三方CA机构颁发，我们的SSL证书是自己生成的，自然不满足这个前提条件。所以还是需要一些配置。这里用Andriod中应用 最广的OkHttp3来做示范。

下面是生成一个包含公匙的证书的命令，XXX.jks就是上面生成的公匙文件，storepass是你的密码

<pre>keytool -export -alias zhy_server -file xxxx.cer 
 -keystore XXXX.jks  
 -storepass XXXXX    //你的store密码 
</pre>

下面是完整的代码
<pre>
    private static final String TAG = "HttpsTest";

    public static void test() {
        String rfc = "-----BEGIN CERTIFICATE-----\n" +
                "MIIDTTCCAjWgAwIBAgIEesCiAzANBgkqhkiG9w0BAQsFADBXMQswCQYDVQQGEwJjbjELMAkGA1UE\n" +
                "CBMCYmoxCzAJBgNVBAcTAmJqMQ8wDQYDVQQKEwZ3YWxrZXIxDzANBgNVBAsTBndhbGtlcjEMMAoG\n" +
                "A1UEAxMDbGl1MB4XDTE2MTEwMzAzNTAyMFoXDTE3MDIwMTAzNTAyMFowVzELMAkGA1UEBhMCY24x\n" +
                "CzAJBgNVBAgTAmJqMQswCQYDVQQHEwJiajEPMA0GA1UEChMGd2Fsa2VyMQ8wDQYDVQQLEwZ3YWxr\n" +
                "ZXIxDDAKBgNVBAMTA2xpdTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJWHIMofHvUM\n" +
                "YqXYffKQB+H6nNPmnvsR/WPsJ3AO12WdJSpqkhnG17Plc/3M3LVopm+2LgnkHnOuz7ZWpBoAVVtn\n" +
                "NpxbniO7ra/eVJLwtLIVzQN2O1yXB2ydSL1ie8XgUK/UUtyTOP3ZMrqrQBnbzA8Q7Nr2FxZaOt0Z\n" +
                "rgLh+P5nCz3SRoNp76ges9EeYew+lE+O1ETtsLXwEAcJfQFMNRzzJsE3ZEHxZEp/33wRDQFIeZ8H\n" +
                "37lsC8I84bCKYfkIAD/U/L0ZKeCTb8ivTwE/sxJvctK0z2xwWx1k4iz1wNtRWRPzIkNJm+P0rdnb\n" +
                "mg8sL739zt3dKNeNQ/09BiXX3QsCAwEAAaMhMB8wHQYDVR0OBBYEFEMDzffLfXXwsq9Z85qHQc/t\n" +
                "j6tWMA0GCSqGSIb3DQEBCwUAA4IBAQBwYEWeLgvKMRcINKTetypXJ2YmvD5QBCJafDLCdM+IkvKs\n" +
                "K21C3oIodjDTreXEEH8UxY8ku1qYodTT8egjenpdD9maTZp1G3SKrkJ3ARQwNuFfAP1HxeQ8y7LF\n" +
                "VX3ZDLfgIib+Xm1ygyAmn3Pd9X/J+4KYmqTJyy4RdSg0flY5c44PuvjT317MouuHbyb53mGrjp4b\n" +
                "+bOGyAJ8N2wFCso1y3367qpAJF3zYVIVWakTufA5nGjhuvN0GZZ109HuB+ry3aPi2mUQheBnQYu3\n" +
                "ggvxxytA58QkUxiqxnZZtAkEEa6tyPIJO5idAs1Lmza8+Wr+jBdm7JrVRb3wQJjOYlDz\n" +
                "-----END CERTIFICATE-----";
		//参考张鸿翔的开源框架<a href="https://github.com/hongyangAndroid/okhttputils">OkHttpUtils</a>
        HttpsUtils.SSLParams sslParams = HttpsUtils.getSslSocketFactory(new InputStream[]{new Buffer().writeUtf8(rfc).inputStream()}, null, "3252860");



        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .sslSocketFactory(sslParams.sSLSocketFactory, sslParams.trustManager)
				//<a href= "http://stackoverflow.com/questions/14619781/java-io-ioexception-hostname-was-not-verified">host 验证错误</a>
                .hostnameVerifier(new NullHostNameVerifier()) 
                .build();
        OkHttpUtils.initClient(okHttpClient);
        OkHttpUtils
                .get()
                .url(url)
                .build()
                .execute(new StringCallback() {
                    @Override
                    public void onError(Call call, Exception e, int id) {
                        Log.e(TAG,e.getMessage());
                    }

                    @Override
                    public void onResponse(String response, int id) {
                        Log.d(TAG,response);
                    }
                });

</pre>
上面是一个完整的请求示例的方法，ssl文件的配置可以利用张鸿翔已经封装好的库来做，我就不造重复造轮子了。

最后要注意NullHostNameVerifier类，这里是要验证HostName，由于OkhttpClient默认是返回false的，所以我们要进行重写！
<pre>
public class NullHostNameVerifier implements HostnameVerifier {
    @Override
    public boolean verify(String s, SSLSession sslSession) {
        Log.d(s,sslSession.getPeerHost());
        return true;
    }
}

</pre>

## 总结
源码在我的Github上面：<a href="https://github.com/WalkerLiuFei/SpringBootWithHttps">戳我！</a>