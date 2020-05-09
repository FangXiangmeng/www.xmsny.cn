---
layout: post
title: Docker部署Golang的hello world服务
subtitle:   "Golang简单高效，而且可以很方便的编写web服务，因此在微服务领域非常受欢迎。而我也非常喜欢这门语言，第一次通
过docker部署服务就很自然的选择了golang编写的hello world服务。"
date: 2018-07-07T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - Docker
---

<!--more-->
Golang简单高效，而且可以很方便的编写web服务，因此在微服务领域非常受欢迎。而我也非常喜欢这门语言，第一次通过docker部署服务就很自然的选择了golang编写的hello world服务。

### 1. 编写Golang  Hello World服务
hello world是每个程序员都熟知的程序了，这里所说的hello world服务其实就是用golang编写http服务，接受请求并在响应中返回hello world字符，下面是代码：
```go
// httpHello.go
package main
import (
	"net/http"
)

func main()  {
	http.HandleFunc("/", func(res http.ResponseWriter, req *http.Request) {
		res.Write("hello world!")
	})
	http.ListenAndServe(":8080", nil)
}
```
使用```go run httpHello.go```运行上面的程序，然后用浏览器访问localhost:8080就可以看到"hello world!"字符，这样就编写了一个极为简单的Hello World服务。

### 2. 编写Dockerfile
有了Hello World服务的程序，就可以使用Dockerfile创建镜像来运行了。通过Dockerfile可以非常简单的从一个基础镜像构建所需要的镜像。

- 在程序所在目录创建Dockerfile文件
- 通过From定义基础容器，运行go程序的话需要填写From golang
- 通过ADD将程序引入docker镜像：```ADD httpHello.go /```表示将httpHello.go引入到镜像的根目录下
- 通过EXPOSE指定从镜像启动的容器需要监听的端口：因为程序监听了8080端口，所以设置为```EXPOSE 8080```
- 指定CMD指定容器启动时执行的名令：```CMD ["go", "run", "/httpHello.go"]```表示在容器启动时值执行```go run /httpHello.go```来运行已经包含在根目录中的程序。
下面是完整的Dockerfile文件

```dockerfile
FROM golang
ADD httpHello.go /

EXPOSE 8080
CMD ["go", "run", "/httpHello.go"]
```
### 3. 构建镜像并从镜像启动docker容器
有了Dockerfile就可以构建docker镜像了，通过docker镜像则可以启动docker容器

#### 3.1 构建镜像
命令行进入到Dockerfile所在目录，执行```docker build .```成功后会有如下输出：

```shell
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM golang
latest: Pulling from library/golang
0bd44ff9c2cf: Pull complete
047670ddbd2a: Pull complete
ea7d5dc89438: Pull complete
ae7ad5906a75: Pull complete
15f6351ddb37: Pull complete
823ef4e8c9c9: Pull complete
dca1089cfb86: Pull complete
Digest: sha256:3b70f4747eb2c74ddf517b548c4ca8071a0c0b8b9c4ef5933e9a98372e03390f
Status: Downloaded newer image for golang:latest
 ---> 4e611157870f
Step 2/4 : ADD httpHello.go /
 ---> e9cb475ff8bd
Step 3/4 : EXPOSE 8080
 ---> Running in 8bb9f8cd5af8
Removing intermediate container 8bb9f8cd5af8
 ---> 99f651e71bc2
Step 4/4 : CMD ["go", "run", "/httpHello.go"]
 ---> Running in dc525b84a581
Removing intermediate container dc525b84a581
 ---> 654b2f885150
Successfully built 654b2f885150
```
#### 3.2 启动容器
从上一步的输出的最后一行中我们可以的得到镜像的id，即"654b2f885150"，然后就可以通过```docker run```从这个镜像启动容器了。为了从宿主机访问容器监听的的8080，需要将容器的8080端口映射到宿主机的容器中，所以需要指定参数-p 8080:8080将容器的8080映射到宿主机的8080端口。还可以通过--name参数指定容器的唯一名字，最后再通过镜像的id指定镜像，就可以启动一个docker容器了，完整命令如下：

```shell
docker run -p 8080:8080 --name httpHello 654b2f885150
```
