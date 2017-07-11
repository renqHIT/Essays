# ElasticSearch 父子查询

标签（空格分隔）： ElasticSearch

---

下文中，user是父表，tweet是子表；

```
# 查看子表mapping
GET weibo_panel_123/_mapping/tweet

# 查询tweet
GET weibo_panel_123/tweet/_search
{
  "query": {
    "term": {
      "twSource": "iPhone 6s"
    }
  }
}

# 支持指定父文档id查询子文档
GET weibo_panel_all/tweet/3998995262949430?parent=1203744932

# 不支持直接用id查询子文档，报错：routing_missing_exception
GET weibo_panel_all/tweet/3998995262949430

# 直接用文档id查询
GET weibo_panel_all/user/1203744932

# 不支持查询父文档是指定id的所有子文档，报错：illegal_argument_exception  [1]
GET weibo_panel_all/tweet/_search?parent=1203744932

# 使用has_parent查询，获取指定父文档id的所有子文档(例如，获取用户id=1203744932的所有微博) [2]
GET weibo_panel_123/tweet/_search
{
  "query": {
    "has_parent": {
      "type": "user",
      "query": {
        "match": {
          "userUid": "1203744932"
        }
      }
    }
  }
}

# 使用has_child查询，获取指定子文档id的所有父文档(例如，获取微博id=3998995262949430的用户)
GET weibo_panel_123/user/_search
{
  "query": {
    "has_child": {
      "type": "tweet",
      "query": {
        "match": {
          "id": "4011744945038157"
        }
      }
    }
  }
}
```

[1] 这个导入数据的逻辑是一样的，父文档不需要知道子文档的id，但子文档必须指定父文档的id。这是因为：ES文档都保存在分片中。默认情况，分片由文档id决定；但在父子文档中，ES要保证父子文档在同一个分片中存储，所以，父文档id决定了子文档在哪个分片保存（父文档和子文档拥有相同的routing值）。
分片公式为：shard = hash(routing) % number_of_primary_shards
[2] 注意，如果子文档个数较多，查询会很慢...
