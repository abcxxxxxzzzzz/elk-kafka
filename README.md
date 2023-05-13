## docker 环境

```bash
# 在 docker 节点执行
# 腾讯云 docker hub 镜像
# export REGISTRY_MIRROR="https://mirror.ccs.tencentyun.com"
# DaoCloud 镜像
# export REGISTRY_MIRROR="http://f1361db2.m.daocloud.io"
# 阿里云 docker hub 镜像
export REGISTRY_MIRROR=https://registry.cn-hangzhou.aliyuncs.com

# 安装 docker
# 参考文档如下
# https://docs.docker.com/install/linux/docker-ce/centos/ 
# https://docs.docker.com/install/linux/linux-postinstall/

# 卸载旧版本
yum remove -y docker \
docker-client \
docker-client-latest \
docker-ce-cli \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# 设置 yum repository
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装并启动 docker
yum install -y docker-ce-19.03.11 docker-ce-cli-19.03.11 containerd.io-1.2.13

mkdir /etc/docker || true

cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["${REGISTRY_MIRROR}"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload
systemctl enable docker
systemctl restart docker

# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab
```

## Docker Compose 环境

```bash
 # 下载 Docker Compose
 sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

 # 修改该文件的权限为可执行
 chmod +x /usr/local/bin/docker-compose

 # 验证信息
docker-compose --version
```

## 环境初始化

```bash
# 需要设置系统内核参数，否则 ES 会因为内存不足无法启动
# 改变设置
sysctl -w vm.max_map_count=262144

# 使之立即生效
sysctl -p



# 创建 logstash 目录，并将 Logstash 的配置文件 logstash.conf 拷贝到该目录
mkdir -p /mydata/logstash

# 需要创建 elasticsearch/data 目录并设置权限，否则 ES 会因为无权限访问而启动失败
mkdir -p /mydata/elasticsearch/data/


chmod 1000:1000 /mydata/elasticsearch/data/
```

## docker-compose 文件

```bash
version: '3'
services:
  elasticsearch:
    image: elasticsearch:7.6.2
    container_name: elasticsearch
    user: root
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      - "discovery.type=single-node" #以单一节点模式启动
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ./elasticsearch.yml:/usr/share/elasticsearch/elasticsearch.yml
      - ./mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins #插件文件挂载
      - ./mydata/elasticsearch/data:/usr/share/elasticsearch/data #数据文件挂载
      - /etc/localtime:/etc/localtime:ro
      - /usr/share/zoneinfo:/usr/share/zoneinfo
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elastic

  logstash:
    image: logstash:7.6.2
    container_name: logstash
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - elasticsearch    # kibana在elasticsearch启动之后再启动
    links:
      - elasticsearch:es # 可以用es这个域名访问elasticsearch服务
    ports:
      - 5044:5044
    networks:
      - elastic

  kibana:
    image: kibana:7.6.2
    container_name: kibana
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    environment:
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml #设置访问elasticsearch的地址
      - /etc/localtime:/etc/localtime:ro
      - /usr/share/zoneinfo:/usr/share/zoneinfo
    ports:
      - 5601:5601
    networks:
      - elastic

  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    volumes:
      - ./mydata/zookeeper/data:/data
      - ./mydata/zookeeper/log:/data/log
      - /etc/localtime:/etc/localtime:ro
      - /usr/share/zoneinfo:/usr/share/zoneinfo
    networks:
      - elastic
    ports:
      - "2181:2181"

  kafka:
    container_name: kafka
    image: wurstmeister/kafka
    depends_on:
      - zookeeper
    volumes:
      - ./mydata/kafka:/kafka
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/localtime:/etc/localtime:ro
      - /usr/share/zoneinfo:/usr/share/zoneinfo
    links:
      - zookeeper
    ports:
      - "9092:9092"
    networks:
      - elastic
    environment:
      - KAFKA_LISTENERS=INTERNAL://kafka:9092, OUT://kafka:29092
      - KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:9092, OUT://kafka:29092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:PLAINTEXT,OUT:PLAINTEXT
      - KAFKA_INTER_BROKER_LISTENER_NAME=OUT
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_MESSAGE_MAX_BYTES=2000000
      - KAFKA_CREATE_TOPICS=logs:1:1

networks:
  elastic:
```

## elasticsearch 配置文件

```bash
# elasticsearch.yml 
# 名称，网络, 在集群上，除了各自名称 name 不一样，其他配置都是一样
cluster.name: "docker-cluster"
network.host: 0.0.0.0

discovery.seed_hosts: ["es:9300"]
cluster.initial_master_nodes: ["es:9300"]

# 数据，日志
path.data: /usr/share/elasticsearch/data
path.logs: /usr/share/elasticsearch/log

# 跨域
http.cors.enabled: true
http.cors.allow-origin: "*"

# 开启权限认证后, es-head-master 访问 es 需要的配置
http.cors.allow-headers: Authorization, X-Requested-With,Content-Length,Content-Type


# 开启权限认证
xpack.security.enabled: true

# 传输通信
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
```

## logstash 配置文件

```bash
# cat logstash.conf 
input { 
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "nginx_access"
     topics => ["nginx_access"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "nginx_error"
     topics => ["nginx_error"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "mysql_error"
     topics => ["mysql_error"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "mysql_slow"
     topics => ["mysql_slow"]
     codec => json
  }



}

output {
  elasticsearch {
    hosts => ["es:9200"]
    #index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    #index => "logs-%{+YYYY.MM.dd}"
    #index => "kafka-%{+YYYY.MM.dd}"
    index => "kafka-%{[fields][kafka_topic]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "elastic"
  }
}
```

## kiana 配置文件

```bash
# cat kibana.yml 
i18n.locale: "zh-CN"
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://es:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
```



## 启动

```bash
[root@es2 data]# docker-compose up -d
[root@es2 data]# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED        STATUS         PORTS                                                                                  NAMES
541e4f5421b9   logstash:7.6.2           "/usr/local/bin/dock…"   23 hours ago   Up 2 minutes   0.0.0.0:5044->5044/tcp, :::5044->5044/tcp, 9600/tcp                                    logstash
573f14a5750e   wurstmeister/kafka       "start-kafka.sh"         23 hours ago   Up 2 minutes   0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                              kafka
af7405513781   kibana:7.6.2             "/usr/local/bin/dumb…"   23 hours ago   Up 2 minutes   0.0.0.0:5601->5601/tcp, :::5601->5601/tcp                                              kibana
693ad300c571   elasticsearch:7.6.2      "/usr/local/bin/dock…"   23 hours ago   Up 2 minutes   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp   elasticsearch
327580f65c76   wurstmeister/zookeeper   "/bin/sh -c '/usr/sb…"   23 hours ago   Up 2 minutes   22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp                  zookeeper
```

## 插件安装

```bash
# elasticsearch 需要安装中文分词器 IKAnalyzer，并重新启动。
#docker exec -it elasticsearch /bin/bash

#此命令需要在容器中运行
#/usr/share/elasticsearch/bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip

# 生产证书
#/usr/share/elasticsearch/bin/elasticsearch-certutil ca
#/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
#mv elastic-* /usr/share/elasticsearch/data/

# 初始化密码： elastic
#/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive


#docker restart elasticsearch

# logstas h需要安装 json_lines 插件，并重新启动。
docker exec -it logstash /bin/bash
logstash-plugin install logstash-codec-json_lines
docker restart logstash
```

## filebeat 客户端安装

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.2-linux-x86_64.tar.gz

tar xzvf filebeat-7.4.2-linux-x86_64.tar.gz
cd filebeat-7.4.2-linux-x86_64
```

## filebeat.yaml 配置

```bash
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /usr/local/nginx/logs/access*.log
  json.keys_under_root: true
  json.overwrite_keys: true
  fields:
    kafka_topic: nginx_access       # 自定义字段

- type: log
  enabled: true
  paths:
    - /usr/local/nginx/logs/error*.log
  json.keys_under_root: true
  json.overwrite_keys: true
  fields:
    kafka_topic: nginx_error       # 自定义字段

- type: log
  enabled: true
  paths:
    - /var/log/mariadb/mariadb.log
  fields:
    kafka_topic: mysql_error       # 自定义字段

- type: log
  enabled: true
  paths:
    - /var/log/mariadb/slow.log
  json.keys_under_root: true
  json.overwrite_keys: true
  #tags: ["mysql_slow"]
  fields:
    kafka_topic: mysql_slow       # 自定义字段

#filebeat.config.modules:
#  path: ${path.config}/modules.d/*.yml
#  reload.enabled: false
#  reload.period: 1s
setup.kibana:
  host: "http://kafka:5601"


output.kafka:
    hosts: ["kafka:9092"]
    topic: "%{[fields.kafka_topic]}"  # 发送到 kafka 的那个 topic,用于生产事件
    version: 1.0.0          # kafka版本 
    partition.round_robin:  # 向 kafka 分区 发送的方式(这里是轮询)
      reachable_only: false
    required_acks: 1          # kafka 应答方式
    compression: gzip         # 设置输出压缩编解码器。必须是none，snappy，lz4和gzip其中一个。默认为gzip
    compression_level: 4      # 设置 gzip 使用的压缩级别。将此值设置为 0 将禁用压缩。压缩级别必须在 1（最佳速度）到 9（最佳压缩）的范围内。提高压缩级别会降低网络使用率，但会增加 CPU 使用率。
    max_message_bytes: 100000 # 规定大于 max_message_bytes 的事件将丢弃
    codec.json:
      pretty: false
```

## filebeat 参数说明

- type: 输入类型。
- enabled: 设置配置是否生效。 true 表示生效， false 表示不生效
- paths:   需要监控的日志文件路径。多个日志路径另起一行
- hosts:   消息队列 Kafka 实例接入点
- topic:   日志输出到消息队列 Kafka 的 Topic。 请指定已经创建的 Topic

## 指定 Host

```bash
[root@vm-1# cat /etc/hosts
172.16.62.179 kafka

# 客户端启动服务
[root@vm-1#./filebeat &
```

## 更多配置

- 有关 filebeat 的 log input 的配置介绍见官网文档：https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html

- 有关 filebeat output 到 kafka 的配置介绍见官方文档：https://www.elastic.co/guide/en/beats/filebeat/current/kafka-output.html

## 配置 Logstash 管道

```bash
input { 
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "nginx_access"
     topics => ["nginx_access"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "nginx_error"
     topics => ["nginx_error"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "mysql_error"
     topics => ["mysql_error"]
     codec => json
  }
   kafka {
     bootstrap_servers => ["kafka:29092"]
     group_id => "mysql_slow"
     topics => ["mysql_slow"]
     codec => json
  }



}


# 分析、过滤插件，可以多个
# filter {
#    grok {
#        match => { "message" => "%{COMBINEDAPACHELOG}"}
#    }
#    geoip {
#        source => "clientip"
#    }
# }


output {
  elasticsearch {
    hosts => ["es:9200"]
    #index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    #index => "kafka-%{+YYYY.MM.dd}"
    index => "kafka-%{[fields][kafka_topic]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "elastic"
  }
}
```

## logstash 参数说明

**input**

- bootstrap_servers: 消息队列 Kafka 实例接入点
- group_id: 指定已创建的 Consumer Group 名称
- topics:   指定为已创建的 Topic 名称(不存在自动创建)， 需要与 Filebeat 中配置的 Topic 名称保持一致
- codec:    设置未 Json, 表示解析 JSON 格式字段

**output**

- hosts: ES 访问地址

- user:  ES 用户名

- password: ES密码

- index: 索引名称

- 有关 logstash 中 kafka-input 的配置介绍见官方文档：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html

- 有关 logstash 中 grok-filter 的配置介绍见官方文档：https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html

- 有关 logstash 中 output-elasticsearch 的配置介绍见官方文档：https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html

## 查看 kafka 日志消费状

```bash
# 进入容器
docker exec -it kafka bash


# kafka 默认安装在 /opt/kafka
cd opt/kafka


# 要想查询消费数据，必须要指定组
bash-5.1# bin/kafka-consumer-groups.sh --bootstrap-server 172.16.62.179:9092 --list
logstash


# 查看 topic
bash-5.1# bin/kafka-topics.sh --list --zookeeper 172.16.62.179:2181
__consumer_offsets
logs

# 查看消费情况
bash-5.1# bin/kafka-consumer-groupsdescribe --bootstrap-server 172.16.62.179:9092 --group logstash

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
logstash        logs            0          107335          107335          0               logstash-0-c6d82a1c-0f14-4372-b49f-8cd476f54d90 /172.19.0.2     logstash-0

#参数解释：
#--describe  显示详细信息
#--bootstrap-server 指定kafka连接地址
#--group 指定组。


# TOPCI: topic 名称
# PARTITION: 分区ID
# CURRENT-OFFSET: 当前已消费的条数
# LOG-END-OFFSET: 总条数
# LAG: 未消费的条数
# CONSUMER-ID: 消费ID
# HOST: 主机IP
# CLIENT-ID: 客户端ID

# 从上面的信息可以看出，topic 为 logs 总共消费了 107335 条信息， 未消费的条数为 0。也就是说，消费数据没有积压的情况.
```

## kibana 索引

```bash
Create index pattern

The index pattern you've entered doesn't match any indices. You can match any of your 4 indices, below.
kafka-mysql_slow-2021.05.13
kafka-nginx_access-2021.05.13

Rows per page: 10
```

