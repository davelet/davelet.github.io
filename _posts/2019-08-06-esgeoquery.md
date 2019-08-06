---
layout: post
title: elasticsearch 地理数据查询API （1）
categories: [dev]
tags: [java, elasticsearch]
---
[前面说过](esgeo/)，Elasticsearch 支持两种地理信息数据结构：geo_point 和 geo_shape。
geo_point 就是经纬度组成的数字对，geo_shape 支持点、线、曲线、多边形、多边形组。

ES 官方列出了四种地信查询，分别是
 - geo_shape 查询，主要是检索地图上相交、包含、不想交的文档。
 - geo_bounding-box 检索，检索点落在某矩形中的文档。
 - geo_distance 检索，检索与某点相距特定距离（范围）的点的文档
 - geo_polygon 检索，检索点落在特定多边形内的文档。

第2条和第4条很像，但是它们使用的API不同，应该是内部实现完全不同。

下面我们分部了解一下这四种检索工具。

> 我这里介绍的是6.2版本。

---

# geo_shape 查询
显然这个要求有一个字段索引是 geo_shape 类型的。它支持两种形状的定义，一种当然是直接定义一个完整的形状，另一种是可以引用另一个索引中的形状名。

## 直接定义
假设有如下的索引：
```
PUT /example
{
    "mappings": {
        "_doc": {
            "properties": {
                "location": {
                    "type": "geo_shape"
                }
            }
        }
    }
}

POST /example/_doc?refresh
{
    "name": "Wind & Wetter, Berlin, Germany",
    "location": {
        "type": "point",
        "coordinates": [13.400544, 52.530286]
    }
}
```
查询的时候形状是envelope，envelope由两个点定义一个矩形：左上角和右下角：
```
GET /example/_search
{
    "query":{
        "bool": {
            "must": {
                "match_all": {}
            },
            "filter": {
                "geo_shape": {
                    "location": {
                        "shape": {
                            "type": "envelope",
                            "coordinates" : [[13.0, 53.0], [14.0, 52.0]]
                        },
                        "relation": "within"
                    }
                }
            }
        }
    }
}
```
> es支持的形状有圆circle（定义圆心和半径）、信封envelope、点、多个点、线段、多线段、多边形等。

## 引用形状
如果另一个索引存在已经定义好的地信形状索引，也可以查询。这在现实中场景很多的。
这时候要提供的是：
 - id，要引用的文档id
 - index，要引用字段的索引名，默认shapes
 - type，索引类型
 - path，包含索引字段的路径，默认shape

下面是例子：
```
PUT /shapes
{
    "mappings": {
        "_doc": {
            "properties": {
                "location": {
                    "type": "geo_shape"
                }
            }
        }
    }
}

PUT /shapes/_doc/deu
{
    "location": {
        "type": "envelope",
        "coordinates" : [[13.0, 53.0], [14.0, 52.0]]
    }
}

GET /example/_search
{
    "query": {
        "bool": {
            "filter": {
                "geo_shape": {
                    "location": {
                        "indexed_shape": {
                            "index": "shapes",
                            "type": "_doc",
                            "id": "1234",
                            "path": "location"
                        }
                    }
                }
            }
        }
    }
}
```

## 空间关系

上面用到了within关系。一共有四种空间关系：
 - INTERSECTS，有交集，这个是默认值
 - DISJOINT，无交集
 - WITHIN，完全被覆盖
 - CONTAINS，完全覆盖