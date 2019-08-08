---
layout: post
title:  "3장 데이터 색인, 변경, 삭제"
date:   2019-08-07 11:00:00
author: Dark
categories: elasticsearch
tags: elasticsearch
---

# 3장 데이터 색인, 변경, 삭제

데이터 색인, 변경과 삭제를 알아보기 이전에 기본적인 Concept 에 대해 이해하는 것이 중요합니다.  
이번 장에서 살펴볼 내용들에 언급되는 용어들의 이해를 돕기 위해 아래와 같은 테이블을 만들어 보았습니다.        

## Concept

|   Index   | Type |    Mapping     | Field  | Field Type | Document |
|-----------|------|----------------|--------|------------|----------|
| Database  |Table | Schema + Index | Column |  Data Type |    Row   |

- 테이블의 상단은 Elasticsearch 이고, 하단은 RDBMS 용어입니다. 
- RDBMS 에서 Table 은 Column 들로 이루어져 있고, 개별 Column 들은 특정 Data Type 이 지정되어 있습니다.
- Elasticsearch 에서 Type 은 Field 들로 이루어져 있고, 개별 Field 들은 특정 Field Type 이 지정되어 있습니다. 
- RDBMS 에서는 Table 의 Schema 가 무엇인지를 확인하여 Data Type 을 살펴봅니다.
- Elasticsearch 에서는 Type 의 Mapping 정보가 무엇인지를 확인하여 Field Type 과 어떻게 색인될 수 있는지 정보를 확인합니다.

## 데이터 색인 ( Data Indexing )

### Mapping

Elasticsearch 에서 Mapping 이란 Data 를 어떤 방식으로 Indexing 할 지에 대한 메타 정보를 제공해주는 개념입니다.   
이에 따라, Field 의 Type 을 정의하고 어떤 Analyzer 를 통해 각 Field 들을 Indexing 할지에 대한 정보가 저장되어 있습니다.   

#### Static Mapping

Static Mapping 은 Elasticsearch 에게 해당 Type 은 어떤 Data 들이 유입될 예정이고, 이들에 대해서는 미리 정의한 정보를 토대로 Indexing 을 처리하길 원하는 경우에 사용하는 방법입니다.  
이에 대한 설명을 위하여 실제 색인을 하기에 앞서 Index 와 Document 를 보관할 Type 을 생성해보겠습니다.

```
PUT {index}
PUT get-together
```
- get-together 라는 Index 를 생성하였습니다.

```
PUT {index}/_mapping/{type}
PUT get-together/_mapping/new-events
{
    "new-events": {
        "properties": {
            "host": {
                "type": "string" 
            }
        }
    }
}
```
- get-together 라는 Index 내부에 new-events 라는 Type 을 생성하였습니다. 
- new-events Type 의 mapping 정보를 설정하였습니다.
- new-events type 은 host 라는 field 를 가지게 되고 이는 string 이라는 field type 을 갖도록 설정하였습니다.

```
PUT {index}/{type}/{documentId}
PUT get-together/new-events/1
{
    "host" : "localhost"
}
```
- document 의 id 를 1 로 지정하였습니다.
- document 의 host field 의 값으로 localhost 를 지정하여 indexing 하였습니다.

#### Dynamic Mapping

Dynamic Mapping 은 Elasticsearch 에게 어떠한 Data 가 유입될지 모르는 경우 사용하는 방법입니다.

```
PUT {index}/{type}/{documentId}
PUT dynamic-index/dynamic-type/1
{
    "host" : "localhost"
}
```
- document 의 id 를 1 로 지정하였습니다.
- document 의 host field 의 값으로 localhost 를 지정하여 indexing 하였습니다.

##### Static vs Dynamic Mapping
```
PUT dynamic-index/dynamic-type/1
{
    "int" : 1
}
```

```
PUT dynamic-index/dynamic-type/2
{
    "int" : "localhost"
}
```
- 최초에 추론하여 설정되었던 Field type 이 달라진 경우 Indexing 시 문제가 발생할 수 있다.
- Document 의 type 이 제대로 관리 되지 않아 불필요한 field 들에 대한 mapping 정보들이 쌓일 수 있다.

### Field

#### Meta Field

#### Field


### Indexing

#### analyzed, not_analyzed, no

아래와 같이 Index 의 종류에 따라 어떻게 검색을 하게 되는지 확인하기 위하여 테스트용 Index 와 Type 을 생성해보도록 하겠습니다.
```
PUT static-index/_mapping/static-type
{
    "static-type": {
        "properties": {
            "notAnalyzedText": {
                "type": "string",
                "index": "not_analyzed" 
            },
            "analyzedText": {
                "type": "string",
                "index": "analyzed" 
            },
            "no": {
                "type": "string",
                "index": "no" 
            }
        }
    }
}
```

##### not_analyzed

notAnalyzedText field 에 소문자로 검색을 시도해보도록 하겠습니다.
```
POST static-index/static-type/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": {
          "notAnalyzedText": "localhost"
        }
      }
    }
  }
}
```

```
{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 0,
    "max_score": null,
    "hits": [

    ]
  }
}
```

##### analyzed

```
POST static-index/static-type/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": {
          "analyzedText": "localhost"
        }
      }
    }
  }
}
```

##### no
```
POST static-index/static-type/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": {
          "no": "Localhost"
        }
      }
    }
  }
}
```

### Reference
[Elasticsearch In Action]
[Standard Analyzer]

[Elasticsearch In Action]:      http://www.yes24.com/Product/Goods/33029174?Acode=101
[Standard Analyzer]:            https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html