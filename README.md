# Jenkins+Gitlab+Docker 实现CI/CD
## 搭建 Docker 私有仓库

> 参考：
>
> - Docker官方文档：[Registry Deploying][1]
> - [手把手教你搭建Docker Registry私服][2]

Docker 官方提供一个搭建私有仓库的镜像 registry ，只需根据此镜像启动一个容器即可。

### 拉取 Registry 镜像

```shell
docker pull registry
```

Registry 服务默认会将上传的镜像保存在容器内的`/var/lib/registry`，我们将主机的`/opt/registry`目录挂载到该目录，即可实现将镜像保存到主机。

### 启动 Registry 容器

```shell
# docker run -itd -v /usr/local/registry:/var/lib/registry -p 5000:5000 \
	--restart=always --name registry registry:2.7.1 
```

 ### 查看仓库

使用`curl`或浏览器访问： `http://localhost:5000/v2/_catalog`

### 推送镜像到仓库

示例：

```
# docker pull ubuntu:16.04
# docker tag ubuntu:16.04  localhost:5000/my-ubuntu
# docker push localhost:5000/my-ubuntu
```

该示例从Docker Hub拉取镜像 `ubuntu:16.04`，然后在本地将其重新标记为`localhost:5000/my-ubuntu`，再将其推送到私有仓库。

再查看 `http://localhost:5000/v2/_catalog`可以看到`my-ubuntu`已经出现在私有仓库了。

### 私有仓库域名配置SSL

待续

### 访问限制

> 参考： [Restricting Access][3]

除了运行在本地安全网络中的仓库，所有仓库都应该总是实现访问限制。

本地基本认证是最简单的实现，使用`htpasswd`存储密钥。

1. 为用户`testuser`创建一个带入口的密码文件，密码是`testpassword`

```shell
$ mkdir auth
$ docker run \
  --entrypoint htpasswd \
  registry:2.7 -Bbn testuser testpassword > auth/htpasswd
```

> 这一步在Windows或Linux下可能发生错误：
>
> `Error response from daemon: OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"htpasswd\": executable file not found in $PATH": unknown.`
>
> 解决办法：
>
> 将 `registry`镜像版本改为`2.7.0`:
>
> ```shell
> # docker run --entrypoint htpasswd registry:2.7.0 -Bbn testuser testpassword > auth/htpasswd
> ```

2. 停止仓库

```shell
$ docker container stop registry
```

3. 使用基本认证启动 Registry

```shell
$ docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v "$(pwd)"/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v "$(pwd)"/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2.7.0
```

4. 这时当试图拉取镜像时会失败
5. 登录到Registry

```shell
$ docker login myregistrydomain.com:5000
```

登录后，就能成功从仓库拉取或推送镜像。

> 更多高级认证
>
> You may want to leverage more advanced basic auth implementations by using a proxy in front of the registry. See the [recipes list](https://github.com/docker/docker.github.io/blob/master/registry/recipes/index.md).
>
> The registry also supports delegated authentication which redirects users to a specific trusted token server. This approach is more complicated to set up, and only makes sense if you need to fully configure ACLs and need more control over the registry's integration into your global authorization and authentication systems. Refer to the following [background information](https://github.com/docker/docker.github.io/blob/master/registry/spec/auth/token.md) and [configuration information here](https://github.com/docker/docker.github.io/blob/master/registry/configuration.md#auth).
>
> This approach requires you to implement your own authentication system or leverage a third-party implementation.

### 用 Compose 文件部署私有仓库

```yaml
registry:
  restart: always
  image: registry:2.7.0
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /path/data:/var/lib/registry
    - /path/certs:/certs
    - /path/auth:/auth
```

将`/path`替换为真正存放`/data`和证书`/certs`的目录。



## 配置 Jenkins



### 启动 Jenkins 容器

Compose 方式启动：

```yaml
enkins:
        image: jenkinsci/blueocean:latest
        user: root
        container_name: jenkins
        ports:
            - 5010:8080
            - 50000:50000
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - ./home:/var/jenkins_home
            - /usr/bin/docker:/usr/bin/docker
        restart: always
```

> 启动后可能遇到的问题：
>
> 在日志输出`Jenkins is fully up and running`之后，进入解锁页面输入初次生成的密钥，点击下一步时，容器突然挂掉，导致错误提示：`无法连接到Jenkins Server`。
>
> 经排查，可能和内存不足有关，Jenkins 容器占用600MB-800MB内存。如果是Linux，请使用`htop`检查可用内存是否充足。

参考：[Docker安装](https://www.jenkins.io/doc/book/installing/#docker)





[1]:https://github.com/docker/docker.github.io/blob/master/registry/deploying.md
[2]:https://juejin.im/post/6844903614859739150
[3]:https://github.com/docker/docker.github.io/blob/master/registry/deploying.md#Restricting-access