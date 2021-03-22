---
title: Golang23-操作Elasticsearch
date: 2021-03-21 22:34:54
tags: golang
---
上一篇文章介绍了[Elasticsearch集群](/2021/03/21/service-elasticsearch/)相关的知识，本文介绍如何通过golang去处理elasticsearch数据。

## 安装
```
go get github.com/olivere/elastic/v7
```

## 连接es
```go
import (
	"fmt"
	"github.com/olivere/elastic/v7"
	"log"
)

func main() {
	// 连接
	var host  = "http://127.0.0.1:9200"
	client, err := elastic.NewClient(elastic.SetURL(host), elastic.SetSniff(false))
	if err != nil {
		log.Fatalf("connect elasticsearch: %v", err)
	}
	log.Println("connect success")
	fmt.Println(client)
}
```

## 创建索引
```go
func CreateIndex(client *elastic.Client) {
    ctx := context.Background()
	result, err := client.CreateIndex("blog").Do(ctx)
	if err != nil {
		log.Fatalf("create index: %v", err)
	}
	if !result.Acknowledged {
		log.Fatalf("create index failed")
	}
	log.Println("create index success")
}
```

## 插入数据
```go
func InsertDoc(client *elastic.Client)  {
	indexName := "blog"
	response, err := client.Index().Index(indexName).BodyJson(`{"title":"hello","content":"world","created_at":1616381976}`).Do(ctx)
	if err != nil {
		log.Fatalf("insert document: %v", err)
	}
	fmt.Printf("insert document success,id: %s", response.Id)
}
```

## 查询
```go
func QueryDoc(client *elastic.Client)  {
	indexName := "blog"
	boolQuery := elastic.NewTermQuery("title", "hello")

	searchResult, err := client.Search(indexName).Query(boolQuery).Do(context.Background())
	if err != nil {
		log.Fatalf("query document: %v", err)
	}
	fmt.Printf("Query took %d milliseconds\n", searchResult.TookInMillis)
	// Number of hits
	if searchResult.Hits != nil {
		fmt.Printf("Found a total of %d tweets\n", searchResult.Hits.TotalHits.Value)

		// Iterate through results
		for _, hit := range searchResult.Hits.Hits {
			// hit.Index contains the name of the index

			// Deserialize hit.Source into a Tweet (could also be just a map[string]interface{}).
			var t map[string]interface{}
			err := json.Unmarshal(hit.Source, &t)
			if err != nil {
				// Deserialization failed
			}

			// Work with tweet
			fmt.Printf("id:%s, title: %s, content: %s\n", hit.Id, t["title"], t["content"])
		}
	} else {
		// No hits
		fmt.Print("Found no tweets\n")
	}
}
```
更多示例请查看：`https://pkg.go.dev/github.com/olivere/elastic`
