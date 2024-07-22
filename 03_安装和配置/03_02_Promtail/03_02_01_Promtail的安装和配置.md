
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

# 2 配置 


## 2.1 默认配置

和 Loki 类似，Promtail 也要在本地挂载的 c:/docker/promtail目录下配置 promtail.yml，这里也使用默认配置：
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          # 这个跟挂载的位置有点关系，你可以猜猜
          __path__: /var/log/*log
```

我猜测 `/var/log/*log` 就是读取日志的位置，所以我把它挂载到本地 `c:/docker/log`，等下弄点日志到本地这个目录下，看看能读取出来不。

```

#  配置 promtail 程序运行时行为。如指定监听的ip、port等信息。
server:
  http_listen_port: 9080
  grpc_listen_port: 0
# positions 文件用于记录 Promtail 发现的目标。该字段用于定义如何保存 postitions.yaml 文件
# Promtail 发现的目标就是指日志文件。
positions:
  filename: /etc/promtail/positions.yaml   # 游标记录上一次同步位置
  sync_period: 10s #10秒钟同步一次
# 配置 Promtail 如何连接到 Loki 的多个实例，并向每个实例发送日志。
# Note：如果其中一台远程Loki服务器无法响应或发生任何可重试的错误，这将影响将日志发送到任何其他已配置的远程Loki服务器。
# 发送是在单个线程上完成的！ 如果要发送到多个远程Loki实例，通常建议并行运行多个Promtail客户端。
clients:
  - url: http://loki:3100/api/prom/push
# 配置 Promtail 如何发现日志文件，以及如何从这些日志文件抓取日志。
scrape_configs:  # 指定要抓取日志的目标。
- job_name: http-log  
  pipeline_stages: 
    - regex:
        expression: "^(?P<time>(\\d{4}-\\d{2}-\\d{2})\\s(\\d{2}:\\d{2}:\\d{2})) (?P<logtype>(\\[(.+?)\\])) (?P<appname>(\\[(.+?)\\])) (?P<requesttype>(\\[(.+?)\\])) (?P<tid>(\\[(.+?)\\])) (?P<cid>(\\[(.+?)\\])) (?P<traceid>(\\[(.+?)\\])) (?P<spanid>(\\[(.+?)\\])) (?P<parentid>(\\[(.+?)\\])) (?P<trace>(trace\\[(.+?)\\])) (?P<data>(.*)) (?P<logcate>(.*))"
    - labels:
        logtype:
        logcate:
    - timestamp:
        source: time
        format: RFC3339Nano
  static_configs:
  - targets:
      - localhost # 指定抓取目标，i.e.抓取哪台设备上的文件
    labels: # 指定该日志流的标签
      job: http-log 
      __path__: /var/logs/http/*.log  # 指定抓取路径，该匹配标识抓取 /var/log/host 目录下的所有文件。注意：不包含子目录下的文件
```


## 2.2 promtail的动态配置


我们只需要为 Loki 应用部署相关的 Promtail 守护程序即可。这里我仍然使用 Docker 对 Promtail 进行部署，不过我不能再使用默认配置了，这时的​​ config.yml​​ 应该是：

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/log/positions.yaml

client:
  url: http://${LOKI_HOST}:${LOKI_PORT}/loki/api/v1/push

scrape_configs:
  - job_name: system
    pipeline_stages:
    static_configs:
      - labels:
          app: ${APP_NAME}
          job: varlogs
          host: ${LOG_HOST}
          __path__: /var/log/*log
```

为了构建一个通用的配置，我将一些参数进行了动态化。这是 ​​Loki2.1+​​​ 版本提供的特性，可以使用​​ ${} 来引用环境变量，甚至你可以为其指定默认值​​ ${VAR:default_value}​​。但是你必须得知道为了开启这一特性需要在 Promtail 启动命令中添加选项​​ -config.expand-env​​。

## 2.3 promtai config 文件中 配置去收集那些日志


这里展示的是promtail容器里面/var/log目录中的日志
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```

这里的job就是varlog，文件路径就是/var/log/*log
[https://raw.githubusercontent.com/grafana/loki/master/cmd/promtail/promtail-local-config.yaml](https://raw.githubusercontent.com/grafana/loki/master/cmd/promtail/promtail-local-config.yaml)

如果是安装二进制的promtail那就更简单了
[https://github.com/grafana/loki/releases](https://github.com/grafana/loki/releases)

```
# download promtail binary file
https://github.com/grafana/loki/releases

curl -O -L "https://github.com/grafana/loki/releases/download/v2.2.1/promtail-linux-amd64.zip"
# extract the binary
unzip "promtail-linux-amd64.zip"
# make sure it is executable
chmod a+x "promtail-linux-amd64"

# add promtail-local-config.yaml file
vim promtail-local-config.yaml

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
      job: varlogs
      __path__: /var/log/*log

- job_name: wangao-date
  static_configs:
  - targets:
      - localhost
    labels:
      job: date
      __path__: /var/log/wangao/date

# create test date file
mkdir /var/log/wangao
cd /var/log/wangao
nohup sh -c 'while true; do echo `date` >> date; sleep 1; done;' &

# run promtail
./promtail-linux-amd64 -config.file=promtail-local-config.yaml
nohup ./promtail-linux-amd64 -config.file=promtail-local-config.yaml &
```

也可以通过修改 docker --log-driver=loki 创建测试日志的容器
```
docker run -d \
    --log-driver=loki \
    --log-opt loki-url="http://localhost:3100/loki/api/v1/push" \
    --log-opt loki-external-labels="tags=test-docker-A" \
    --name test-docker-A \
    busybox sh -c 'while true; do echo "This is a log message from container A"; sleep 10; done;'

docker run -d \
    --log-driver=loki \
    --log-opt loki-url="http://localhost:3100/loki/api/v1/push" \
    --log-opt loki-external-labels="tags=test-docker-B" \
    --name test-docker-B \
    busybox sh -c 'while true; do echo "This is a log message from container B"; sleep 10; done;'
```

[![](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201105181327.png)](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201105181327.png)
## 2.4 限流

测试期间遇到的问题，由于开始时promtail采集日志有burst的流量，导致loki被限流。查询相关材料，需要调大loki的相关limit，做了对应的激进调整后，限速消失。

```text
limits_config:
  ingestion_rate_mb: 100
  ingestion_burst_size_mb: 100
  per_stream_rate_limit: 100MB
  per_stream_rate_limit_burst: 200MB
```
