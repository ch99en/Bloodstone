

### Links

[`Github`](https://github.com/RediSearch/RediSearch)

[`Getting-Started`](https://github.com/RediSearch/redisearch-getting-started)



### 关键指令

#### 创建索引

```
> FT.CREATE idx:movie ON hash PREFIX 1 "movie:" SCHEMA title TEXT SORTABLE release_year NUMERIC SORTABLE rating NUMERIC SORTABLE genre TAG SORTABLE
```

- [`FT.CREATE`](https://oss.redislabs.com/redisearch/master/Commands/#ftcreate) : creates an index with the given spec. The index name will be used in all the key names so keep it short.
- `idx:movie` : the name of the index
- `ON hash` : the type of structure to be indexed. *Note that in RediSearch 2.0 only hash structures are supported, this parameter will accept other Redis data types in future as RediSearch is updated to index them*
- `PREFIX 1 "movie:"` : the prefix of the keys that should be indexed. This is a list, so since we want to only index movie:* keys the number is 1. Suppose you want to index movies and tv_show that have the same fields, you can use: `PREFIX 2 "movie:" "tv_show:"`
- `SCHEMA ...`: defines the schema, the fields and their type, to index, as you can see in the command, we are using [TEXT](https://oss.redislabs.com/redisearch/Query_Syntax/#a_few_query_examples), [NUMERIC](https://oss.redislabs.com/redisearch/Query_Syntax/#numeric_filters_in_query) and [TAG](https://oss.redislabs.com/redisearch/Query_Syntax/#tag_filters), and [SORTABLE](https://oss.redislabs.com/redisearch/Sorting/) parameters.



#### 查询全量字段

```
> FT.SEARCH idx:movie "war"

1) (integer) 2
2) "movie:11005"
3)  1) "title"
    2) "Star Wars: Episode VI - Return of the Jedi"
    ...
   14) "tt0086190"
4) "movie:11002"
5)  1) "title"
    2) "Star Wars: Episode V - The Empire Strikes Back"
    ...
   13) "imdb_id"
   14) "tt0080684"
```



#### 查询给定字段

```
> FT.SEARCH idx:movie "war" RETURN 2 title release_year

1) (integer) 2
2) "movie:11005"
3) 1) "title"
   2) "Star Wars: Episode VI - Return of the Jedi"
   3) "release_year"
   4) "1983"
4) "movie:11002"
5) 1) "title"
   2) "Star Wars: Episode V - The Empire Strikes Back"
   3) "release_year"
   4) "1980"
```



#### 用例

```
FT.CREATE SMARTX_VM SCHEMA title TEXT WEIGHT 5.0 desc TEXT
FT.ADD SMARTX_VM vm-2019082911110001 1.0 LANGUAGE "chinese" FIELDS title "人工智能" desc "我在北京昌平学习人工智能"
FT.SEARCH SMARTX_VM "人工智能" LANGUAGE "chinese"
```



### 可行性分析

线上 redis 实例如何支持 redisearch？



如何同步全量数据到 redis 中？



日语是否支持？日语的模糊搜索规则是怎样的？

不支持



包含多语言（中日文）是否支持索引？如果不支持，区分环境处理（开发服中文，正式服日文），是否满足要求？



### 落地

#### 模型

```
第一版先简单，完成基本目标需求
- document model
	key
	field[]
- field model
	key
	value
	type

- indexer
	- base-indexer
	- user-info-indexer(name, id, obfuse_id)
	build(data: dict)

- query
	- base-query
	- user-info-query
	get()
```

