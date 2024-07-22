
# 1 例子1

https://wsgzao.github.io/post/loki/

vim promtail-nginx.yml
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

client:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: nginx
  static_configs:
  - targets:
      - localhost
    labels:
      job: nginx
      env: test
      app: nginx
      project: wangao-test-nginx
      host: wangao-test-nginx
      __path__: /var/log/nginx/*.log
```


vim promtail-nginx-pipeline.yml
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

- job_name: nginx-pipeline
  static_configs:
  - targets:
      - localhost
    labels:
      job: nginx-pipeline
      env: live
      app: nginx
      project: wangao-live-nginx
      host: wangao-live-nginx
      __path__: /var/log/nginx/*.log
  pipeline_stages:
  - match:
      selector: '{job="nginx-pipeline"}'
      stages:
      - regex:
          # logline example: 127.0.0.1 - - [21/Apr/2020:13:59:45 +0000] "GET /?foo=bar HTTP/1.1" 200 612 "http://example.com/lekkebot.html" "curl/7.58.0"
          expression: '^(?P<remote_addr>[\w\.]+) - (?P<remote_user>[^ ]*) \[(?P<time_local>.*)\] "(?P<method>[^ ]*) (?P<request>[^ ]*) (?P<protocol>[^ ]*)" (?P<status>[\d]+) (?P<body_bytes_sent>[\d]+) "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)"?'
      - labels:
          remote_addr:
          remote_user:
          time_local:
          method:
          request:
          protocol:
          status:
          body_bytes_sent:
          http_referer:
          http_user_agent:
```


Querying Logs with LogQL
```
{app="nginx"} |= "GET"
{app="nginx"} |~ "200|201|202"
{app="nginx"} != "GET"
{app="nginx"} !~ "200|201|202"

```


To search for a certain string in the results, you can use a search expression. This can be just text matching by using |= or a regex expression by using |~. And by using a ! instead of the pipe, the expression can be negated. Here are some examples:

```
{app="nginx"} |= "GET"
{app="nginx"} |~ "200|201|202"
{app="nginx"} != "GET"
{app="nginx"} !~ "200|201|202"
{app="nginx"} |= "GET" |= "error"
{app="nginx"} |= "GET" |= "error" |~ "192.168.32."

#Requests for last 60 Seconds:
count_over_time( {job="nginx"} [60s])
# Rate over 60s
rate( ( {env="production", job="nginx"} ) [60s])
# Show metrics with filter patterns:
rate( ( {env="production", job="nginx"} |~ "GET (/er|/ax)" ) [10s])

# try more metrics
sum by (method,path)
(
  rate({compose_service="nginx"}
    | regexp "\"(?P<method>\\w+ )(?P<path>.*) HTTP"
  [1m])
)

{compose_service="nginx"}
    | regexp
    "\"(?P<method>\\w+ )(?P<path>.*) HTTP\\/\\d\\.\\d\" (?P<status_code>\\d{3}) \\d{1,} (?P<value>\\d{1,}.\\d{1,})"
	
max by (path)
  (
    avg_over_time(
    {compose_service="nginx"}
    | regexp
   "\"(?P<method>\\w+ )(?P<path>.*) HTTP\\/\\d\\.\\d\" (?P<status_code>\\d{3}) \\d{1,} (?P<value>\\d{1,}.\\d{1,})"    [1m]
  )
)

quantile_over_time(0.99,
  {compose_service="nginx"}| regexp
    "\"\\w+ .* HTTP\\/\\d\\.\\d\" \\d{3} \\d{1,} (?P<value>\\d{1,}.\\d{1,})"
    [1m]
  )

```

# 2 例子2

修改 promtail 配置文件
url： 指定 Loki 地址

```
$ vim /data/soft/promtail/config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.0.11:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: nginxlogs # job 名称
      __path__: /var/log/nginx/*log  # 收集日志路径

```

挂载 nginx 日志文件目录和 promtail 配置文件到容器
```
$ docker run -ti --name promtail \
-v /var/log/nginx/:/var/log/nginx/ \
-v /data/soft/promtail/config.yml:/etc/promtail/config.yml \
grafana/promtail:2.4.1 -config.file=/etc/promtail/config.yml

```


