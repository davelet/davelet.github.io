---
layout: post
title: elasticsearch 地理数据查询API （2）
categories: [dev]
tags: [elasticsearch]
---
[上一篇说过](esgeoquery/)，ES 官方给了四种地信查询，分别是
 - geo_shape 查询，主要是检索地图上相交、包含、不相交的文档。
 - geo_bounding-box 检索，检索点落在某矩形中的文档。
 - geo_distance 检索，检索与某点相距特定距离（范围）的点的文档
 - geo_polygon 检索，检索点落在特定多边形内的文档。

第一种过滤器已经介绍了，现在继续下面的。

> 我这里介绍的是6.2版本。

# geo_bounding-box 检索

假设有如下索引：
```
PUT /my_locations
{
    "mappings": {
        "_doc": {
            "properties": {
                "pin": {
                    "properties": {
                        "location": {
                            "type": "geo_point"
                        }
                    }
                }
            }
        }
    }
}

PUT /my_locations/_doc/1
{
    "pin" : {
        "location" : {
            "lat" : 40.12,
            "lon" : -71.34
        }
    }
}
```
下面使用geo_bounding-box来过滤：
```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : {
                            "lat" : 40.73,
                            "lon" : -74.1
                        },
                        "bottom_right" : {
                            "lat" : 40.01,
                            "lon" : -71.12
                        }
                    }
                }
            }
        }
    }
}
```

## 可接受的坐标格式
传入的点可以有多种格式。
### 经纬度属性
上面用到的就是这个格式：
```
"top_left" : {
    "lat" : 40.73,
    "lon" : -74.1
},
"bottom_right" : {
    "lat" : 40.01,
    "lon" : -71.12
}
```
### 数组
两个元素的数组，经度在前：
```
"top_left" : [-74.1, 40.73],
"bottom_right" : [-71.12, 40.01]
```
### 字符串拼接
通过逗号链接，注意要纬度在前：
```
"top_left" : "40.73, -74.1",
"bottom_right" : "40.01, -71.12"
```
### WKT 格式
WKT = Well-Known Text。
```
"wkt" : "BBOX (-74.1, -71.12, 40.73, 40.01)"
```
### geo哈希格式
```
"top_left" : "dr5r9ydj2y73",
"bottom_right" : "drj7teegpus6"
```
可以通过[http://geohash.co/](http://geohash.co/)生成geohash，也是纬度在前哦。
### 顶点
通过四个点当然肯定可以：
```
"top" : 40.73,
"left" : -74.1,
"bottom" : 40.01,
"right" : -71.12
```

## 多点过滤
这个过滤器可以处理有多个位置的文档，只要有一个位置匹配成功就认文档被匹配。

# 过滤类型
有两种类型，一种是memory，指明在内存中计算；一种是indexed，指明在索引上计算。如果点被索引为geo_point了，使用Indexed更快一点；但是注意，indexed 不支持多点过滤。
```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "wkt" : "BBOX (-74.1, -71.12, 40.73, 40.01)"
                    },
                    "type" : "indexed"
                }
            }
        }
    }
}
```