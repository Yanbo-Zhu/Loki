
# 1 安装

## 1.1 通过二进制文件 

下载地址：[github.com/grafana/lok…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgrafana%2Floki%2Freleases "https://github.com/grafana/loki/releases") 下载**promtail-windows-amd64.exe.zip**安装包，并解压到**F:\soft\grafana\promtail**目录，得到**promtail-windows-amd64.exe** 在**F:\soft\grafana\promtail**目录下添加**promtail-local-config.yaml**文件，内容如下

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: viplogs
      __path__: F:\soft\grafana\testlogs\*.log
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: viplogs
      __path__: F:\soft\grafana\testlogs\*.log

```

打开cmd定位到exe目录，执行命令： .\promtail-windows-amd64.exe --config.file=promtail-local-config.yaml，loki服务启动成功。
```
.\promtail-windows-amd64.exe --config.file=promtail-local-config.yaml
```

## 1.2 通过docker_例子1

根据这个我对 Promtail 的 Docker 镜像进行了改造，具体的 ​​Dockerfile​​ 为：

```shell
FROM grafana/promtail:2.2.1
LABEL AUTHOR = felord.cn
VOLUME ["/var/log/"]
EXPOSE 9080
ENV LOKI_HOST="localhost"
ENV LOKI_PORT=3100
ENV APP_NAME="APP"
ENV LOG_HOST="localhost"
COPY config.yml /etc/promtail/
CMD ["-config.file=/etc/promtail/config.yml", "-config.expand-env"]
```

你可以通过​​ docker build -t loki-promtail:1.0 .​​命令构建这个自定义 Promtail 镜像。基本的启动命令：

```
docker run -d  --name promtail-service --network loki -v c:/docker/log:/var/log/  -e LOKI_HOST=loki -e APP_NAME=SpringBoot  loki-promtail:1.0
```


其中挂载的目录​​ c:/docker/log​​​ 依然是应用的日志目录，​​LOKI_HOST ​​要保证能够同 Loki 服务器通信，无论你通过直连还是 Docker 网络(这里用了 Docker 网桥)。你可以可以使用 Docker Compose 将应用和 Promtail 进行捆绑，所有的 Promtail 将把对应的日志发往 Loki 进行集中式的管理。另外通过自定义的 ​​Label​​ 我们可以通过应用名称来搜索日志了。

![](https://img-blog.csdnimg.cn/211fb22b2ae04589a90573c50866918f.png)


## 1.3 通过docker_例子2

启动 promtail ，将 promtail 配置文件拷贝到宿主机
```
$ docker run -ti --name promtail grafana/promtail:2.4.1 -config.file=/etc/promtail/config.yml
$ mkdir /data/soft/promtail
$ docker cp promtail:/etc/promtail/config.yml /data/soft/promtail/config.yml

```

