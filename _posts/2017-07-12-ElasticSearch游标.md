# ElasticSearch 游标

标签（空格分隔）： ElasticSearch

---

为了重建索引，我们需要批量地获取数据，然后向新的索引批量插入这些数据，其中，批量地从旧所有获取数据使用scroll, 批量地向新索引插入数据使用bulk API。

```
POST _reindex
{
    "source": {
        "index": "tweet"
    },
    "dest": {
        "index": "new_tweet"
    }
}

```

## Scroll

Scroll查询用于高效获取大量文档，不需要花费深度分页的代价；Scroll允许我们执行一条初始搜索，持续从ES获取结果，直到查询到所有结果。这有点像传统数据库的游标（cursor）。

一次Scroll查询会制作一个快照，查询请求发出后，不会看到建立快照之后的索引更新。通过保留旧数据文件，我们可以留住查询当时的视图。

深度分页的耗时操作主要是对查询结果进行全量排序，如果我们禁止排序，就可以快速返回结果。为了禁止全局排序，我们指定按_doc排序，这样即使其他分片也有数据需要返回，ES也会立刻返回一批结果。

为了遍历结果，我们执行搜索请求，设置scroll值为保持scroll窗口开启的时长。每次执行一条新的scroll请求，scroll过期时间就会更新。因此窗口时长需要设置为大于一次scroll请求的处理时长，而不是全量数据的处理时长。超时时长的选取很关键，因为我们希望尽快释放维护scroll窗口所需要的资源。

```
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
```

返回的结果包含一个\_scroll\_id, 它是一个Base64编码的字符串；然后我们把这个\_scroll\_id传给\_search/scroll去获取下一批数据；

```
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```


## Bulk


