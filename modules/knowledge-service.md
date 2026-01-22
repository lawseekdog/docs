# Knowledge Service 模块设计

## 概述

Knowledge Service 负责法律知识库的管理和检索，包括法规、案例、要素等内容的存储、索引和语义检索。

## 技术栈

- Java 21 + Spring Boot 3.3
- PostgreSQL（元数据）
- Elasticsearch（全文检索）
- Weaviate（向量检索）

## 核心概念

### KnowledgeBase（知识库）

```java
public class KnowledgeBase {
    private UUID id;
    private String name;
    private String type;           // system | organization | user
    private UUID ownerId;
    private KnowledgeBaseStatus status;
}
```

### KnowledgeDocument（文档）

```java
public class KnowledgeDocument {
    private UUID id;
    private UUID knowledgeBaseId;
    private String title;
    private String docType;        // law | case | regulation | article
    private String sourceUrl;
    private JsonNode metadata;
    private Instant createdAt;
}
```

### KnowledgeChunk（文档块）

```java
public class KnowledgeChunk {
    private UUID id;
    private UUID documentId;
    private String content;
    private Integer chunkIndex;
    private JsonNode metadata;     // section, article_num, etc.
    private float[] embedding;     // 向量（存储在 Weaviate）
}
```

## 检索架构

```
查询请求
    │
    ▼
┌─────────────────────────────────────┐
│        Knowledge Service            │
│                                     │
│  ┌─────────────────────────────┐   │
│  │     Query Preprocessor      │   │
│  │  - 查询改写                  │   │
│  │  - 关键词提取                │   │
│  │  - 法条号解析                │   │
│  └──────────────┬──────────────┘   │
│                 │                   │
│    ┌────────────┴────────────┐     │
│    │                         │     │
│    ▼                         ▼     │
│  ┌──────────┐          ┌──────────┐│
│  │   ES     │          │ Weaviate ││
│  │  BM25    │          │  Vector  ││
│  │ 关键词   │          │  语义    ││
│  └────┬─────┘          └────┬─────┘│
│       │                     │      │
│       └──────────┬──────────┘      │
│                  │                 │
│         ┌───────▼───────┐         │
│         │   Reranker    │         │
│         │  结果重排序    │         │
│         └───────┬───────┘         │
│                 │                 │
└─────────────────┼─────────────────┘
                  │
                  ▼
            检索结果
```

## 核心功能

### 1. 知识库管理

```
POST   /api/v1/knowledge/bases              # 创建知识库
GET    /api/v1/knowledge/bases/{id}         # 获取知识库
DELETE /api/v1/knowledge/bases/{id}         # 删除知识库
```

### 2. 文档管理

```
POST   /api/v1/knowledge/bases/{id}/documents      # 上传文档
GET    /api/v1/knowledge/bases/{id}/documents      # 文档列表
DELETE /api/v1/knowledge/documents/{id}            # 删除文档
```

### 3. 检索接口

```
POST   /api/v1/knowledge/search             # 混合检索
POST   /api/v1/knowledge/search/laws        # 法规检索
POST   /api/v1/knowledge/search/cases       # 案例检索
POST   /api/v1/knowledge/elements/match     # 要素匹配
```

## 混合检索实现

### 检索请求

```json
{
  "query": "劳动合同解除赔偿",
  "doc_types": ["law", "case"],
  "filters": {
    "year_range": [2020, 2024],
    "court_level": "supreme"
  },
  "top_k": 10
}
```

### ES 检索

```java
public List<SearchHit> searchByKeyword(String query, SearchFilters filters) {
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
        .should(QueryBuilders.matchQuery("title", query).boost(2.0f))
        .should(QueryBuilders.matchQuery("content", query))
        .should(QueryBuilders.matchPhraseQuery("content", query).boost(1.5f));

    // 法条号精确匹配
    if (isArticleNumber(query)) {
        boolQuery.should(QueryBuilders.termQuery("article_nums", normalizeArticleNum(query)).boost(3.0f));
    }

    // 案号精确匹配
    if (isCaseNumber(query)) {
        boolQuery.should(QueryBuilders.termQuery("case_number", query).boost(5.0f));
    }

    return esClient.search(boolQuery, filters);
}
```

### Weaviate 向量检索

```java
public List<WeaviateResult> searchByVector(String query, int topK) {
    float[] embedding = embeddingService.embed(query);

    return weaviateClient.query()
        .className("DocChunk")
        .withNearVector(embedding)
        .withLimit(topK)
        .run();
}
```

### 结果融合

```java
public List<SearchResult> hybridSearch(SearchRequest request) {
    // 1. ES 关键词检索
    List<SearchHit> esResults = searchByKeyword(request.getQuery(), request.getFilters());

    // 2. Weaviate 向量检索
    List<WeaviateResult> vectorResults = searchByVector(request.getQuery(), request.getTopK() * 2);

    // 3. RRF 融合
    Map<String, Double> scores = new HashMap<>();
    int k = 60; // RRF 参数

    for (int i = 0; i < esResults.size(); i++) {
        String id = esResults.get(i).getId();
        scores.merge(id, 1.0 / (k + i + 1), Double::sum);
    }

    for (int i = 0; i < vectorResults.size(); i++) {
        String id = vectorResults.get(i).getId();
        scores.merge(id, 1.0 / (k + i + 1), Double::sum);
    }

    // 4. 按融合分数排序
    return scores.entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .limit(request.getTopK())
        .map(e -> buildResult(e.getKey()))
        .toList();
}
```

## 要素匹配

### Element（要素）

```java
public class Element {
    private String id;
    private String causeCode;      // 案由代码
    private String name;           // 要素名称
    private String description;
    private ElementType type;      // fact | evidence | claim
    private List<String> keywords;
}
```

### 要素匹配接口

```
POST /api/v1/knowledge/elements/match
{
  "cause_code": "tort_personal_injury",
  "facts": "原告在被告经营的超市购物时滑倒受伤...",
  "evidence_list": ["医疗费发票", "监控录像"]
}

Response:
{
  "matched_elements": [
    {
      "element_id": "e001",
      "name": "侵权行为",
      "match_score": 0.92,
      "evidence_support": ["监控录像"]
    }
  ],
  "missing_elements": [
    {
      "element_id": "e002",
      "name": "因果关系证明",
      "suggestion": "建议补充医疗诊断证明"
    }
  ]
}
```

## 文档入库流程

```
上传文档
    │
    ▼
┌─────────────────────────────────────┐
│     UnifiedKnowledgeIngestService   │
│                                     │
│  1. 解析文档（PDF/Word/Markdown）   │
│  2. 提取元数据                      │
│  3. 文档分块                        │
│  4. 生成向量                        │
│  5. 存储到 PostgreSQL              │
│  6. 索引到 Elasticsearch           │
│  7. 存储向量到 Weaviate            │
│                                     │
└─────────────────────────────────────┘
```

## 数据模型

```
┌─────────────────┐       ┌─────────────────┐
│  KnowledgeBase  │       │KnowledgeDocument│
├─────────────────┤       ├─────────────────┤
│ id              │───┐   │ id              │
│ name            │   │   │ knowledge_base_id│──┐
│ type            │   └──▶│ title           │  │
│ owner_id        │       │ doc_type        │  │
│ status          │       │ source_url      │  │
└─────────────────┘       │ metadata        │  │
                          └─────────────────┘  │
                                               │
                          ┌─────────────────┐  │
                          │ KnowledgeChunk  │  │
                          ├─────────────────┤  │
                          │ id              │  │
                          │ document_id     │──┘
                          │ content         │
                          │ chunk_index     │
                          │ metadata        │
                          └─────────────────┘
```

## 目录结构

```
knowledge-service/
├── src/main/java/com/lawseekdog/knowledge/
│   ├── api/
│   │   ├── controller/
│   │   │   ├── KnowledgeBaseController.java
│   │   │   ├── DocumentController.java
│   │   │   └── SearchController.java
│   │   └── dto/
│   ├── application/
│   │   └── service/
│   │       ├── KnowledgeQueryService.java
│   │       ├── SearchIndexService.java
│   │       ├── ElementService.java
│   │       └── UnifiedKnowledgeIngestService.java
│   ├── domain/
│   │   ├── entity/
│   │   ├── repository/
│   │   └── search/
│   │       └── KnowledgeSearchBackend.java
│   └── infrastructure/
│       ├── external/
│       │   ├── search/
│       │   │   └── ElasticsearchKnowledgeSearchBackend.java
│       │   └── vector/
│       │       └── WeaviateClient.java
│       └── persistence/
└── src/main/resources/
```
