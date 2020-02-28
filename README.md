## elasticsearch multinode


### 起動

`docker-compose up -d`
最新の7.6.0でelasticsearch, kibana起動


### healthcheck

クラスタ

`curl -X GET "http://localhost:9200/_cat/health?v&pretty"`

```
epoch      timestamp cluster      status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1581911878 03:57:58  test-cluster green           3         3      6   3    0    0        0             0                  -                100.0%
```

ノード

`curl -X GET "http://localhost:9200/_cat/nodes?v&pretty"`

```
ip          heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.0.3           38          97   6    0.52    0.25     0.23 dilm      *      es1
192.168.0.5           48          97   6    0.52    0.25     0.23 dilm      -      es2
192.168.0.4           55          97   6    0.52    0.25     0.23 dilm      -      es3
```


### マスタノードの確認

`curl -X GET "http://localhost:9200/_cat/master?v&pretty"`

３つのうちのどれかがmasterになってると思う
nodeのhealthcheckだけでも確認できる


### indexの確認

`curl -X GET "http://localhost:9200/_cat/indices?v&pretty"`
kibana関連のidexのみがある状態


### indexとmappingを作る
`curl -X PUT "http://localhost:9200/customer" -H 'Content-Type: application/json' -d'
{
  "mappings" : {
        "properties" : {
            "name" : { "type" : "text" },
            "age" : { "type" : "integer" }
        }
    }
}'
`


### mappingの確認
​
`curl -X GET http://localhost:9200/customer/_mapping?pretty`
​
```
{
  "customer" : {
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "integer"
        },
        "name" : {
          "type" : "text"
        }
      }
    }
  }
}
```


### documentの作成
`curl -X POST "http://localhost:9200/customer/_doc" -H 'Content-Type: application/json' -d'
{
  "name" : "Katsuta",
  "age" : 25
}'
`
​

### クラスタにindexが作成されていることを確認

`curl -X GET "http://localhost:9200/_cat/indices?v&pretty"`

settingも確認
`curl -X GET "http://localhost:9200/customer/_settings?pretty"`

elasticsearch６系まで1インデックスに対してプライマリーシャードは5, レプリカは1だった

7系からプライマリーシャードは1, レプリカは1になってる


```
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   customer A5f9xw8RQE65iz2TgC-Kxw   1   1          0            0       460b           230b
```


### idex生成によって作成されたシャードがどのnodeに配置されたかを確認する

`curl -X GET "http://localhost:9200/_cat/shards?v&pretty"`

```
index    shard prirep state   docs store ip            node
customer 0     p      STARTED    0  230b 192.168.128.3 es1
customer 0     r      STARTED    0  230b 192.168.128.4 es2
```


### 各ノードでのシャードのディスク使用状況を確認する

`curl -X GET "http://localhost:9200/_cat/allocation?v&pretty"`


### 全件取得

`curl -X GET http://localhost:9200/customer/_search`

1p1rでどれくらい検索にかかってるか確認


### 作成したindexに対してprimaryShard数を変更してみる

`curl -X PUT 'localhost:9200/customer/_settings' -H 'Content-Type: application/json' -d'
{
  "index" : {
      "number_of_shards" : 3
  }
}'`

Can't update non dynamic settings [[index.number_of_shards]]
変更できないはず
[この前のスライド](https://docs.google.com/presentation/d/11NdyoE-_zxbJJFqljvTWePvAT9N8prw8d2O7cQlvV8I/edit#slide=id.g7df55fa65b_0_395)


### replicaはどうだろう

`curl -X PUT 'localhost:9200/customer/_settings' -H 'Content-Type: application/json' -d'
{
  "index" : {
      "number_of_replicas" : 2
  }
}'`
変更できた

どこに配置された確認する

`curl -X GET "http://localhost:9200/_cat/shards?v&pretty"`


再度全件取得
`curl -X GET http://localhost:9200/customer/_search`

replica増やしたら取得にかかる時間がはやくなる



### primaryshardとreplicashardの数を設定してindexを作ってみよう

`curl -X PUT 'localhost:9200/client' -H 'Content-Type: application/json' -d'
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3,
            "number_of_replicas" : 2
        }
    }
}'`

上記だとindex作成時にshard数replica_shard数を設定しているが共通で設定する場合はelasticsearch.yml
に下記のように設定する

```
index.number_of_shards: 3
index.number_of_replicas: 2
```

### 作成したindexを確認

`curl -X GET "http://localhost:9200/client/_settings?pretty"`

```
{
  "client" : {
    "settings" : {
      "index" : {
        "creation_date" : "1581998846532",
        "number_of_shards" : "3",
        "number_of_replicas" : "2",
        "uuid" : "kvDvavUvTOGnZnjUsWD-yQ",
        "version" : {
          "created" : "7060099"
        },
        "provided_name" : "client"
      }
    }
  }
}
```

指定したシャード数、レプリカ数になってることを確認
どこに配置された確認してみよ
`curl -X GET "http://localhost:9200/_cat/shards?v&pretty"`


### analyzer

docker-composeファイルのコメントアウトしてあるbuildをコメント外して
imageをコメントアウトして再度`docker-compose up -d`

standard
```
curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer" : "standard",
  "text" : "関西国際空港"
}'
```

kuromoji(形態素解析)
```
curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer" : "kuromoji",
  "text" : "関西国際空港"
}'
```

### kibana
