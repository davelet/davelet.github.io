---
layout: post
title: elasticsearch 地理数据查询API （3）
categories: [dev]
tags: [elasticsearch]
---
[上上篇说过](esgeoquery/)，ES 官方给了四种地信查询过滤器，分别是
 - geo_shape 查询，主要是检索地图上相交、包含、不相交的文档。
 - geo_bounding-box 检索，检索点落在某矩形中的文档。
 - geo_distance 检索，检索与某点相距特定距离（范围）的点的文档
 - geo_polygon 检索，检索点落在特定多边形内的文档。

现在继续介绍剩下两种。

> 我这里介绍的是6.2版本。

# geo_distance 检索
geo_distance过滤的是点间距离小于指定长度的文档，[《elasticsearch 地理信息处理（java）》](esgeo/)中的查询就是用的这个。假设有如下索引：
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
通过geo_distance过滤：
```
GET /my_locations/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "200km",
                    "pin.location" : {
                        "lat" : 40,
                        "lon" : -70
                    }
                }
            }
        }
    }
}
```

# geo_polygon 检索
这个过滤器过滤出来被指定多边形包含的文档。下面是检索在三角形区域内的人员：
```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_polygon" : {
                    "person.location" : {
                        "points" : [
                        {"lat" : 40, "lon" : -70},
                        {"lat" : 30, "lon" : -80},
                        {"lat" : 20, "lon" : -90}
                        ]
                    }
                }
            }
        }
    }
}
```