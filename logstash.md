---
title: logstash
date: 2018-08-05 20:27:38
tags: 技术
---
# 安装

# 配置
在配置文件logstash.conf中修改
input指定log文件的位置
filter对日志进行处理和格式化
output指定输出到redis或者elastic

**配置文件中一定需要显示指定一个input和output**
启动logstash
bin/logstash -f logstash.conf

报错
Logstash could not be started because there is already another instance using the configured data directory
解决
./bin/logstash -f test.conf --path.data=/home/elastic

## filter 介绍
date filter的作用，原来输出字段中的@timestamp是读取系统时间，通过配置filter date可以从日志的message中简析出时间并格式化时间到@timestamp字段当中
先用grok filter从mesage里面解析并转换生成新的字段

看看grok的用法，基本表达式%{SYNTAX:SEMANTIC}
SYNTAX是指定的一个正则表达式，上面这句话的意思就是从message字段中，将匹配到SYNTAX的内容放到新生成的SEMANTIC字段当中

**所以学好logstash一定要熟悉grok的正则匹配**
# 配置模板
<!--more-->
### 监听ssh用户登录
input{
        file{
                path => "/var/log/secure"
        }
}
output{
        stdout{}
}

### 输出到elasticsearch
input{
        file{
                path => "/var/log/secure"
        }
}
output{
        elasticsearch{
                hosts => "http://10.239.182.93:9200"
                index => "logstash-test"
                document_type => "test"
        }
}

### 简析系统日志elasticsearch配置
input{
        file{
                path => "/var/log/secure"
                type => "system-ssh-log"
        }

}
filter{
        grok{
                match=>{"message"=>"%{SYSLOGBASE}"}
        }
}
output{
        elasticsearch{
                hosts => "http://10.239.182.93:9200"
                index => "logstash-%{type}-%{+YYYY.MM.dd}"
                document_type => "%{type}"

        }
}

**SYSLOGBASE是很重要的一个系统日志匹配的正则表达式**
