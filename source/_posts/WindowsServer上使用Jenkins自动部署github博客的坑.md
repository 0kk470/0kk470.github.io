---
title: WindowsServer使用Jenkins自动部署github博客网站的坑
date: 2022-02-21 17:13:09
tags: Jenkins
---

# 前言

摸索了大半天终于弄好了。

有关```github webhook``` 和 ```jenkins```的配置可以参考[这篇文章](https://www.blazemeter.com/blog/how-to-integrate-your-github-repository-to-your-jenkins-project)。


# 踩坑

### SSL验证的问题
***

使用的```https```来传递信息触发构建事件，结果push上去后```github webhook```抛了个错误信息。

![错误](github_error.png)

原因是没有配置正确的SSL证书验证，于是从对应的云服务商（我用的腾讯云）那下载了之前申请好的SSL证书，下载格式选择为```jks```。

![证书下载](download_ssl.png)

将下载好的证书放到jenkins的根目录，然后在配置文件```jenkins.xml```中的```arguments```节点项里添加如下参数。
```
--httpsKeyStore="%JENKINS_HOME%\证书名.jks" --httpsKeyStorePassword="证书校验密码"
```
![argument](argument.png)

之后重启```jenkins```，然后让repo对应的webhook页面```redelivery```一下就行。

![redelivery](redelivery.png)


### 批处理脚本命令无法执行的问题
***

构建时可以让```jenkins```调用本地bat脚本

![build_bat](build_bat.png)

逻辑如下
```bat
echo  update blog
cd ./repo

if exist node_modules\ (
  echo skip install npm_module 
) else (
  call npm install
)

call hexo clean

call hexo g
```

但是会报错找不到```npm```和```hexo```等可执行程序。

原因是jenkins的运行环境是一个独立的沙盒环境，不会用到Windows系统自带的环境变量，所以这些可执行文件的PATH路径需要添加在全局配置当中。该配置在```.jenkins```文件夹下的```config.xml```文件里。


![config](config.png)

也可以通过jenkins 图形化控制台的```Dashboard -> ConfigureSystem -> Global properties-> Environment variables```配置。

![gui_config](gui_config.png)。

