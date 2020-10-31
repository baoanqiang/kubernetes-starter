# 安装要求

第一次使用 Jenkins，您需要：

- 机器要求：
  - 256 MB 内存，建议大于 512 MB
  - 10 GB 的硬盘空间（用于 Jenkins 和 Docker 镜像）
- 需要安装以下软件：
  - Java 8 ( JRE 或者 JDK 都可以)
  - [Docker](https://www.docker.com/) （导航到网站顶部的Get Docker链接以访问适合您平台的Docker下载）



**一. 安装jdk**

1.新建安装文件夹（例如 mkdir /usr/java）。

2.在安装文件夹中利用wget下载对应的rpm文件（例如 wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" + jdk下载链接）。

3.利用rpm安装rpm文件（例如rpm -ivh jdk-8u171-linux-x64.rpm，出现下图表示安装成功）。

![img](https://img-blog.csdn.net/20180522180053730)

4.配置安环境变量

  1）利用vi /etc/profile编辑profile文件

  2）加入如下内容（jdk文件夹名称根据实际的填写）

```shell
#set java environment
JAVA_HOME=/usr/java/jdk1.8.0_171-amd64
JRE_HOME=/usr/java/jdk1.8.0_171-amd64/jre
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH
```
  3）利用source /etc/profile让配置生效（无需该操作也可完成配置）。

  4）配置完成输入java -version 下图表示成功。

​    ![img](https://img-blog.csdn.net/20180522181246514)

注意：wget下载完成后检查文件是否损坏，利用 ls -lht检查文件大小。

### 下载并运行 Jenkins

1. [下载 Jenkins](http://mirrors.jenkins.io/war-stable/latest/jenkins.war).
2. 打开终端进入到下载目录.
3. 运行命令 `java -jar jenkins.war --httpPort=8080`.
4. 打开浏览器进入链接 `http://localhost:8080`.
5. 按照说明完成安装.

第一次登录生成的密码，到时候会让你设置密码

![image-20201031152403621](E:\student\kubernetes-starter\jenkins-docs\images\image-20201031152403621.png)

Pipelines的插件全部安装

![image-20201031145422628](E:\student\kubernetes-starter\jenkins-docs\images\image-20201031145422628.png)