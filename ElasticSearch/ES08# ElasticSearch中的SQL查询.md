---
title: ES08# ElasticSearch中的SQL查询
categories: Elasticsearch
tags: Elasticsearch
date: 2022-05-29 11:55:01
---



# 引言

通过SQL进行检索ElasticSearch的文档，在一些复杂场景更为灵活。由于DSL需要熟悉其语法，自建的日志平台可能将DSL屏蔽和封装，暴露SQL的查询更易上手。本文顺着官方指南实操一把，文章内容有。

* Kibana执行SQL查询
* Post请求执行SQL分页查询
* SQL中使用DSL过滤
* 使用复杂查询条件
* 其他查询方式（运行时字段与异步SQL）



# 一、Kibana执行SQL查询

请求示例：

```sql
POST /_sql?format=txt
{
  "query": """
      SELECT "pid","span_id","trace_id","user_id" FROM "prd_detail-xxx_*" LIMIT 10
   """
}
```

返回结果：

```
      pid      |    span_id     |    trace_id    |    user_id    
---------------+----------------+----------------+---------------
833037         |a481fcd11b5e7ef3|0ffc42e668901b86|null           
2631155        |44273ff566fc9634|2a770bf4a65425e6|null           
1397839        |691f7a77caf21a4c|ebc60684c13a2af3|null           
3984591        |638c9eda5973bcd3|e36218668bcac321|null   
```



备注：在使用kibana console会使用三引号(""")，format格式支持csv、json、txt、yaml等众多格式，查询支持*号。



# 二、Post请求执行SQL分页查询

**1.添加测试数据** 

先造点测试数据，方便测试，请求URL：

```
PUT /library/_bulk?refresh
```

输入参数：

```
{"index":{"_id": "Leviathan Wakes"}}
{"name": "Leviathan Wakes", "author": "James S.A. Corey", "release_date": "2011-06-02", "page_count": 561}
{"index":{"_id": "Hyperion"}}
{"name": "Hyperion", "author": "Dan Simmons", "release_date": "1989-05-26", "page_count": 482}
{"index":{"_id": "Dune"}}
{"name": "Dune", "author": "Frank Herbert", "release_date": "1965-06-01", "page_count": 604}
```



备注：上面命令通过kibana将结果注入。



**2.查询数据** 

请求URL：

```
http://127.0.0.1:9200/_sql?format=json
```

输入参数：

```
{
  "query": "SELECT * FROM library ORDER BY page_count DESC"
}
```

返回结果：

```
{
    "columns": [
        {
            "name": "author",
            "type": "text"
        },
        {
            "name": "name",
            "type": "text"
        },
        {
            "name": "page_count",
            "type": "long"
        },
        {
            "name": "release_date",
            "type": "datetime"
        }
    ],
    "rows": [
        [
            "Frank Herbert",
            "Dune",
            604,
            "1965-06-01T00:00:00.000Z"
        ],
        [
            "James S.A. Corey",
            "Leviathan Wakes",
            561,
            "2011-06-02T00:00:00.000Z"
        ],
        [
            "Dan Simmons",
            "Hyperion",
            482,
            "1989-05-26T00:00:00.000Z"
        ]
    ]
}
```

备注：Postman中通过SQL查询导入的共计3条数据。



**2.分页首次查询** 

输入参数：

```
{
  "query": "SELECT * FROM library ORDER BY page_count DESC",
  "fetch_size": 2
}
```

返回结果：

```json
{
    "columns": [
        {
            "name": "author",
            "type": "text"
        },
        {
            "name": "name",
            "type": "text"
        },
        {
            "name": "page_count",
            "type": "long"
        },
        {
            "name": "release_date",
            "type": "datetime"
        }
    ],
    "rows": [
        [
            "Frank Herbert",
            "Dune",
            604,
            "1965-06-01T00:00:00.000Z"
        ],
        [
            "James S.A. Corey",
            "Leviathan Wakes",
            561,
            "2011-06-02T00:00:00.000Z"
        ]
    ],
    "cursor": "i6+xAwFaAXN4RkdsdVkyeDFaR1ZmWTI5dWRHVjRkRjkxZFdsa0RYRjFaWEo1UVc1a1JtVjBZMmdCRm1kbFJGRTBZa2wzVkdwdGN6ZG1RMVo1WlZWSFgyY0FBQUFBQUFBQ0R4WXRNMnhRTm5CTk9GTkljVEV0ZFd4d1IxRjVZbGRS/////w8EAWYGYXV0aG9yAQZhdXRob3IBBHRleHQAAAABZgRuYW1lAQRuYW1lAQR0ZXh0AAAAAWYKcGFnZV9jb3VudAEKcGFnZV9jb3VudAEEbG9uZwAAAAFmDHJlbGVhc2VfZGF0ZQEMcmVsZWFzZV9kYXRlAQhkYXRldGltZQEAAAEP"
}
```

备注：Postman中执行，总共3条数据，查询一页2条，返回的最后一行cursor，下一页用它来查。



**3.分页第二次查询** 

输入参数：

```json
{
  "cursor": "i6+xAwFaAXN4RkdsdVkyeDFaR1ZmWTI5dWRHVjRkRjkxZFdsa0RYRjFaWEo1UVc1a1JtVjBZMmdCRm5WSmRrWktRM0pIVVhOeGVFNTRVRGsyVFhGNFdIY0FBQUFBQUFBQ2xCWTJTRU5VTUhSTVVGRTNkVXR2U2xCdVdWQnNUekZC/////w8EAWYGYXV0aG9yAQZhdXRob3IBBHRleHQAAAABZgRuYW1lAQRuYW1lAQR0ZXh0AAAAAWYKcGFnZV9jb3VudAEKcGFnZV9jb3VudAEEbG9uZwAAAAFmDHJlbGVhc2VfZGF0ZQEMcmVsZWFzZV9kYXRlAQhkYXRldGltZQEAAAEP"
}

```

返回结果：

```json
{
    "rows": [
        [
            "Dan Simmons",
            "Hyperion",
            482,
            "1989-05-26T00:00:00.000Z"
        ]
    ],
    "cursor": "i6+xAwFaAXN4RkdsdVkyeDFaR1ZmWTI5dWRHVjRkRjkxZFdsa0RYRjFaWEo1UVc1a1JtVjBZMmdCRm5WSmRrWktRM0pIVVhOeGVFNTRVRGsyVFhGNFdIY0FBQUFBQUFBQ2xCWTJTRU5VTUhSTVVGRTNkVXR2U2xCdVdWQnNUekZC/////w8EAWYGYXV0aG9yAQZhdXRob3IBBHRleHQAAAABZgRuYW1lAQRuYW1lAQR0ZXh0AAAAAWYKcGFnZV9jb3VudAEKcGFnZV9jb3VudAEEbG9uZwAAAAFmDHJlbGVhc2VfZGF0ZQEMcmVsZWFzZV9kYXRlAQhkYXRldGltZQEAAAEP"
}
```

 备注：当继续分页查询时，需要使用上次查询返回的cursor来查，第二次查询依旧一页2条数据，总共3条，返回了1条数据。



**4.分页第三次查询** 

输入参数：

```json
{
  "cursor": "i6+xAwFaAXN4RkdsdVkyeDFaR1ZmWTI5dWRHVjRkRjkxZFdsa0RYRjFaWEo1UVc1a1JtVjBZMmdCRm1kbFJGRTBZa2wzVkdwdGN6ZG1RMVo1WlZWSFgyY0FBQUFBQUFBQ2NSWXRNMnhRTm5CTk9GTkljVEV0ZFd4d1IxRjVZbGRS/////w8EAWYGYXV0aG9yAQZhdXRob3IBBHRleHQAAAABZgRuYW1lAQRuYW1lAQR0ZXh0AAAAAWYKcGFnZV9jb3VudAEKcGFnZV9jb3VudAEEbG9uZwAAAAFmDHJlbGVhc2VfZGF0ZQEMcmVsZWFzZV9kYXRlAQhkYXRldGltZQEAAAEP" 
}

```

返回结果：

```json
{
    "rows": []
}
```

备注：当再次输入cursor查询时，返回记录为空，分页结束。



# 三、SQL中使用DSL过滤

请求URL：

```
http://127.0.0.1:9200/_sql?format=json
```

输入参数：

```json
{
  "query": "SELECT * FROM library ORDER BY page_count DESC",
  "filter": {
    "range": {
      "page_count": {
        "gte" : 400,
        "lte" : 500
      }
    }
  },
  "fetch_size": 2
}
```

返回结果：

```json
{
    "columns": [
        {
            "name": "author",
            "type": "text"
        },
        {
            "name": "name",
            "type": "text"
        },
        {
            "name": "page_count",
            "type": "long"
        },
        {
            "name": "release_date",
            "type": "datetime"
        }
    ],
    "rows": [
        [
            "Dan Simmons",
            "Hyperion",
            482,
            "1989-05-26T00:00:00.000Z"
        ]
    ]
}
```

备注：可以通过ElasticSearch DSL来过滤结果。



# 四、柱状显示查询结果

请求参数：

```
http://127.0.0.1:9200/_sql?format=json
```

输入参数：

```json
{
  "query": "SELECT * FROM library ORDER BY page_count DESC",
  "fetch_size": 2,
  "columnar": true
}
```

返回结果：

```json
{
    "columns": [
        {
            "name": "author",
            "type": "text"
        },
        {
            "name": "name",
            "type": "text"
        },
        {
            "name": "page_count",
            "type": "long"
        },
        {
            "name": "release_date",
            "type": "datetime"
        }
    ],
    "values": [
        [
            "Frank Herbert",
            "James S.A. Corey"
        ],
        [
            "Dune",
            "Leviathan Wakes"
        ],
        [
            604,
            561
        ],
        [
            "1965-06-01T00:00:00.000Z",
            "2011-06-02T00:00:00.000Z"
        ]
    ],
    "cursor": "i6+xAwFaAXN4RkdsdVkyeDFaR1ZmWTI5dWRHVjRkRjkxZFdsa0RYRjFaWEo1UVc1a1JtVjBZMmdCRm1kbFJGRTBZa2wzVkdwdGN6ZG1RMVo1WlZWSFgyY0FBQUFBQUFBZkdoWXRNMnhRTm5CTk9GTkljVEV0ZFd4d1IxRjVZbGRS/////w8EAWYGYXV0aG9yAQZhdXRob3IBBHRleHQAAAABZgRuYW1lAQRuYW1lAQR0ZXh0AAAAAWYKcGFnZV9jb3VudAEKcGFnZV9jb3VudAEEbG9uZwAAAAFmDHJlbGVhc2VfZGF0ZQEMcmVsZWFzZV9kYXRlAQhkYXRldGltZQEAAAEP"
}
```



备注：通过参数columnar来设置显示样式，默认为false。



# 五、使用复杂查询条件

请求url：

```
http://127.0.0.1:9200/_sql?format=json
```

输入参数：

```json
{
	"query": "SELECT YEAR(release_date) AS year FROM library WHERE page_count > 300 AND author = 'Frank Herbert' GROUP BY year HAVING COUNT(*) > 0"
}
```

返回结果：

```json
{
    "columns": [
        {
            "name": "year",
            "type": "integer"
        }
    ],
    "rows": [
        [
            1965
        ]
    ],
    "cursor": "i6+xAwFaAWMBB2xpYnJhcnn+AgEBCWNvbXBvc2l0ZQdncm91cGJ5AAEPYnVja2V0X3NlbGVjdG9yD2hhdmluZy4zZTc4ZDhjNgEGX2NvdW50/wECYTAGX2NvdW50AAEIcGFpbmxlc3NTSW50ZXJuYWxRbFNjcmlwdFV0aWxzLm51bGxTYWZlRmlsdGVyKEludGVybmFsUWxTY3JpcHRVdGlscy5ndChwYXJhbXMuYTAscGFyYW1zLnYwKSkKAAoBAnYwAQAAAAAB/wEBCDYzOTU0MjMzAQxyZWxlYXNlX2RhdGUAAAEAAAECMXkCAQFaAAAAAAAAAADoBwEKAQg2Mzk1NDIzMwL////bRGPIAAACAQAAAAABAP////8PAAAAAAEEYm9vbD+AAAAAAgVyYW5nZT+AAAAACnBhZ2VfY291bnQBAAABLP8AAAAAAAR0ZXJtP4AAAAAOYXV0aG9yLmtleXdvcmQVDUZyYW5rIEhlcmJlcnQAAAAAAQAAAAAAAAAAAVoDAAICAAAAAAAAAAD/////DwIBcAEuAWEBawg2Mzk1NDIzMwABAmR0CQABawg2Mzk1NDIzMwEAAQEA"
}
```

备注：可通过SQL92查询、分组等复杂条件来执行。





# 六、其他查询方式



* 可利用运行时字段(runtime fields)对查询结果聚合，过滤和排序，需要es 7.11版本以上，本文使用7.10 不再演示
* 通常使用同步SQL查询，elasticsearch也支持异步SQL查询







