---
title: MySQL同步至ElasticSearch?
categories: [运维]
tags: [MySQL,ElasticSearch]
date: 2024-12-27
media_subpath: '/posts/2024/12/27'
---

## 1. 使用 Docker Compose 启动 Elasticsearch 和 Kibana

```yaml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0
    container_name: elasticsearch
    environment:
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=true
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.10.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana
      - ELASTICSEARCH_PASSWORD=<示例密码>
    depends_on:
      - elasticsearch

volumes:
  esdata:
    driver: local
```

### 运行命令

```bash
docker-compose up -d
```

## 2. 初始化密码

进入容器：

```bash
docker exec -it <elasticsearch-container-id> /bin/bash
```

执行密码初始化：

```bash
bin/elasticsearch-setup-passwords interactive
```

按照提示配置 `kibana`、`elastic` 等用户的密码。

## 3. 下载并安装 Logstash

从 [Elastic 官网](https://www.elastic.co/downloads/past-releases#logstash) 下载与 Elasticsearch 版本对应的 Logstash。本示例使用 7.10.0 版本。

## 4. 配置 pipelines

在 `logstash` 目录下的 `config/pipelines.yml` 添加多个管道：

```yaml


# ...existing yaml content...
- pipeline.id: pipeline_one.conf
  pipeline.workers: 1
  path.config: "/path/to/logstash/config/pipeline_one.conf"

- pipeline.id: pipeline_two.conf
  pipeline.workers: 1
  path.config: "/path/to/logstash/config/pipeline_two.conf"

- pipeline.id: pipeline_three.conf
  pipeline.workers: 1
  path.config: "/path/to/logstash/config/pipeline_three.conf"
```

## 5. 以 pipeline_three.conf 为例

```conf


input {
  jdbc {
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://<示例IP>:3306/<示例数据库>?useSSL=false&serverTimezone=UTC"
    jdbc_driver_library => "/path/to/logstash/mysql-connector-java.jar"
    jdbc_user => "<示例用户>"
    jdbc_password => "<示例密码>"
    use_column_value => true
    tracking_column => "update_time"
    tracking_column_type => "timestamp"
    last_run_metadata_path => "/path/to/logstash/last_run_metadata/pipeline_three"
    statement => "SELECT * FROM <示例表> WHERE update_time > :sql_last_value ORDER BY update_time ASC"
    schedule => "0,10,20,30,40,50 * * * * *"
  }
}

filter {
  mutate { remove_field => ["@timestamp","update_time"] }
}

output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "<示例索引>"
    document_id => "%{id}"
    action => "update"
    doc_as_upsert => true
    user => "elastic"
    password => "<示例密码>"
  }
}
```

## 6. 启动数据同步

在 Logstash 根目录下执行：

```bash
bin/logstash -f config/pipelines.yml
```

以上即可完成示例配置。