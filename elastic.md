---
title: elastic
date: 2018-07-31 21:28:54
tags: 技术
---
# 安装

## 创建非root用户
adduser elastic
passwd elastic

## 使用新用户下载
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.10.tar.gz

## 启动：
./bin/elasticsearch

打开另一个终端进行测试：

curl 'http://localhost:9200/?pretty'
你能看到以下返回信息：
{
   "status": 200,
   "name": "Shrunken Bones",
   "version": {
      "number": "1.4.0",
      "lucene_version": "4.10"
   },
   "tagline": "You Know, for Search"
}

报错：
[root@localhost ~]# curl -XGET 'http://localhost:9200/?pretty'
<HTML>
<HEAD><TITLE>Redirection</TITLE></HEAD>
<BODY><H1>Redirect</H1></BODY>

关闭防火墙和代理
unset $http_proxy
unset $https_proxy

## 修改绑定ip
尝试修改配置文件 elasticsearch.yml
中network.host(注意配置文件格式不是以#开头的要空一格， ：后要空一格) 为network.host: 0.0.0.0


## 安装kibana sense
下载解压kibana5.6.10
Sense was renamed to Console and is built into Kibana 5.
https://github.com/elastic/sense/blob/master/README.md

直接启动./bin/kibana
会连接上默认配置的elastic

# restful API命令格式
命令格式：
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_talking_to_elasticsearch.html
<!--more-->
### -i 选项
curl -i -XGET 'localhost:9200/'
-i指定请求返还响应的头
### 简单搜索
GET /megacorp/employee/_search?q=last_name:Smith
直接在url后边添加参数，但是这种方式不支持单词的部分匹配
GET /megacorp/employee/_search?q=last_name:Sm是搜索不到Smith的

### 指定请求体
#### count返回符合条件的文档数量
curl -XGET localhost:9200/_count?pretty -d '
{
    "query": {
        "match_all": {}
    }
}'

#### count返回符合条件的文档数量
curl -XGET 'localhost:9200/megacorp/employee/_search' -d '{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}'

#### 单查询与组合多查询
* 单查询
'{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}'
query和filter的区别
filter用于精确字段的精确查找和过滤，没有scoce，只返回精确匹配查找到的doc
query用于

* filtered用于组合query和filter
'{
  "query":{
    "filtered":{
      "query" : {
          "match" : {
              "last_name" : "Smith"
          }
      },
      "filter":{
        term:{
          "age":12
        }
      }
    }

  }
}'
组合多查询:使用bool
bool下面可以包含4个子句: must,must_not,should,filter
'{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith"
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 }
                }
            }
        }
    }
}'
constant_score可以用来代替bool查询语句
https://www.elastic.co/guide/cn/elasticsearch/guide/current/combining-queries-together.html

#### 创建索引或者添加文档
添加文档时，若索引不存在会自动创建索引，可以指定自动创建的索引所使用的**模板**，这样对于自动创建的索引进行控制
curl -XPUT 'localhost:9200/intel/employee/3' -d '{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}'
创建索引时有三个配置项，shards,replication,以及analysis
analysis的使用：
https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html
定制化分析器：
可以配置type为某个内置的analyzer然后配置这个analyzer的选项，也可以直接指定type为custom，然后配置tokenizer，filter以及char_filter
https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-custom-analyzer.html

关闭和打开索引
http://cwiki.apachecn.org/pages/viewpage.action?pageId=4882797
curl -XPOST 'localhost:9200/my_index/_close?pretty'
curl -XPOST 'localhost:9200/my_index/_open?pretty'

## 查询语句的区别:
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_most_important_queries.html
match: 会用对应查询的字段的分析器去分词并匹配计算score，与term的区别
match_all: 匹配所有对应文档
  {
      "query": {
          "match_all": {}
      }
  }
match_phrase: 精确完全匹配上整串短语,保留那些包含 全部 搜索词项，且 位置 与搜索词项相同的文档
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_phrase_search.html
term:查询被用于精确值 匹配,注意term区分大小写，所以一个字段field如果是text类型，在倒排索引中会被状态成小写，直接term查找会找不到，需要term查找这个字段的field.keyword
terms:terms 查询和 term 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件
prefix: 前缀查询，不会对输入的查询信息进行analyze
wildcard:使用标准的 shell 通配符查询
regexp:使用正则表达式通配符查询
match_phrase_prefix:
range：
multi_match: 在多个字段上搜索同一个文本
https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-match-query.html
有三种评分方式：
best_field,most_field,cross_field
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cross_fields_queries.html

scroll：游标查询，可以用于批量查询数据
https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html
fuzzy:模糊匹配
https://www.elastic.co/guide/cn/elasticsearch/guide/current/fuzzy-query.html
range和term常用于filter

## 查看字段映射信息
GET /_mapping?pretty
GET /index/_mapping?pretty
GET /index/_mapping/type?pretty

mapping中text和keyword的区别
"about": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    }
  }
}
about字段类型是text，默认会被analyze和倒排索引
about.keyword字段则不会被analyze

"fielddata": true
fielddata主要用于聚类，
只有text字段要配置fielddata用于倒排索引，text字段的fielddata默认是关闭的，需要对字段聚类时要配置它打开:
https://blog.csdn.net/baristas/article/details/78974090
其它类型的字段用doc_values用于倒排索引

**doc_values与fielddata的区别**
doc_values用于非text字段聚合，默认是打开的，作为倒排索引的转置存放在硬盘上，可以在创建索引时关闭
fielddata用于text字段，默认是关闭的，可以创建索引时打开，第一次使用字段的聚类时生成，并存放在内存中


## 查看分析器
curl -XGET http://10.239.182.93:9200/_analyze?pretty -d '{"analyzer":"standard","text":"Text to analyze"}'
用于查看standard分析器是如何分析Text to analyze这句话的

## 获得具体搜索时分析的信息
GET /my_index/my_type/_validate/query?explain
{
    "query": {
        "match": {
            "name": "brown fo"
        }
    }
}
GET /my_index/doc/_search?explain
{
  "query": {
    "term": {
      "text": "fox"
    }
  }
}

## search的时候评分的规则
https://www.elastic.co/guide/cn/elasticsearch/guide/current/scoring-theory.html

## 排序：
sort

## 聚类
理解两个概念，桶和度量
分桶之后，根据指定的度量计算每个桶的度量值

## 地理位置
地理坐标点不能被动态映射 （dynamic mapping）自动检测，而是需要显式声明对应字段类型为 geo-point
```
PUT /attractions
{
  "mappings": {
    "restaurant": {
      "properties": {
        "name": {
          "type": "string"
        },
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
}
```
## 集群状态监测
GET _cluster/health
GET _nodes/stats
GET _cluster/stats
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cluster_stats.html
GET /_cat 列出所有可用的cat API
GET /_cat/health
GET /_cat/health?v
?v 参数： 显示表头

#集群修改
vim config/elasticsearch.yml
cluster.name: leslie-elastic  #多个instance同一个cluster name
node.name: node-1  #独立的node name
network.host: 10.239.182.240 #外网访问的IP
discovery.zen.ping.unicast.hosts: ["10.239.182.240", "10.239.182.51"]
默认是单播模式，只会在一台机器上去搜索同一个名字的elastic，希望搜索其它host需要在discovery.zen.ping.unicast.hosts添加上搜索的IP

vim config/kibana.yml
server.host: 10.239.182.240
elasticsearch.url: "http://10.239.182.240:9200"
