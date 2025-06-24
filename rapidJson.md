---
title: rapidJson
date: 2018-09-22 09:58:02
tags: 技术
---
### 简介
RapidJSON 是一个高效的 C++ JSON 解析／生成器
http://rapidjson.org/zh-cn/md_doc_tutorial_8zh-cn.html

### 序列化
```
rapidjson::StringBuffer buffer1;
rapidjson::Writer<rapidjson::StringBuffer> writer1(buffer1);
    
writer1.StartObject();
writer1.Key("count");
writer1.Int(2);    
writer1.EndObject();
printf("%s\n", buffer1.GetString());

```

### 反序列化
```
#include "rapidjson/document.h"
using namespace rapidjson;
string a="{
    "hello": "world",
    "t": true ,
    "f": false,
    "n": null,
    "i": 123,
    "pi": 3.1416,
    "a": [1, 2, 3, 4]
}"

Document document;
document.Parse(a.c_str());
if(!document.HasMember("hello"))
{
  return -1;
}
printf("hello = %s\n", document["hello"].GetString());
```
