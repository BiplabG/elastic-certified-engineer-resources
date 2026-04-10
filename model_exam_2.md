# Model 2: Exam Questions from ChatGPT

## **Question 1 — Index Template + Component Templates (Data Management)**

You are standardizing index creation across teams.

**Requirements:**

- Create a **component template**:
  - Defines:
    - `@timestamp` as `date`
    - `env` as `keyword`
- Create an **index template**:
  - Applies to `logs-*`
  - Uses the component template
  - Sets:
    - `number_of_shards = 1`
    - `number_of_replicas = 1`
- Ensure new indices automatically use these mappings and settings

**Task:**

Define the component template and index template.

<details>
<summary><strong>Answer</strong></summary>

```
PUT _component_template/custom-component-template
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "env": {
          "type": "keyword"
        }
      }
    }
  }
}
```

```
PUT _index_template/my-custom-index-template
{
  "priority": 500,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index": {}
    }
  },
  "index_patterns": [
    "logs-*"
  ],
  "composed_of": [
    "custom-component-template"
  ]
}
```

</details>

---

## **Question 2 — Advanced Dynamic Templates (Data Management)**

You are ingesting JSON documents with unknown fields.

**Requirements:**

- Any field ending with `_id`:
  - Must be mapped as `keyword`
- Any numeric field under `metrics.*`:
  - Must be mapped as `float`
- Any field under `metadata.*`:
  - Must NOT be indexed

**Task:**

Create an index `dynamic-test` with dynamic templates fulfilling these conditions.

<details>
<summary><strong>Answer</strong></summary>

```
PUT dynamic-test/
{
  "mappings": {
    "dynamic_templates": [
      {
        "ending_with_id": {
          "match": "*_id",
          "mapping": {
            "type": "keyword"
          }
        }
      },
      {
        "numeric_field": {
          "path_match": "metrics.*",
          "mapping": {
            "type": "float"
          }
        }
      },
      {
        "metadata_field": {
          "path_match": "metadata.*",
          "mapping": {
            "index": false,
            "type": "keyword"
          }
        }
      }
    ]
  }
}
```

</details>

---

## **Question 3 — Multi-Match + Field Boosting (Searching Data)**

Index `products` contains:

- `title`
- `description`
- `category`

**Requirements:**

- Search for "wireless headphones"
- Boost:
  - `title` 3x
  - `description` 1x
- Only include documents where:
  - `category` is `electronics`
- Sort by `_score` descending

**Task:** Write the query.

<details>
<summary><strong>Answer</strong></summary>

```
"multi_match": {
  "query": "wireless headphones",
  "fields": ["title^3", "description"]
}
```

```
GET products/_search
{
  "query": {
    "function_score": {
      "query": {
        "term": {
          "category": {
            "value": "electronics"
          }
        }
      },
      "functions": [
        {
          "filter": {
            "match": {
              "title": "wireless headphones"
            }
          },
          "weight": 3
        },
        {
          "filter": {
            "match": {
              "description": "wireless headphones"
            }
          },
          "weight": 1
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply"
    }
  },
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ]
}
```

</details>

---

## **Question 4 — Pipeline Aggregations (Searching Data)**

You are analyzing sales data in `sales`.

**Requirements:**

- Aggregate total revenue per `month`
- Calculate:
  - Month-over-month revenue change (difference)
- Only include last 6 months

**Task:**

Write an aggregation query using pipeline aggregations.

<details>
<summary><strong>Answer</strong></summary>

```
GET sales/_search
{
  "size": 0,
  "query": {
    "range": {
      "order_date": {
        "gte": "now-6M"
      }
    }
  },
  "aggs": {
    "month_wise_data": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month"
      },
      "aggs": {
        "revenue_per_month": {
          "sum": {"field": "sales_amount"}
        },
        "revenue_change": {
          "derivative": {"buckets_path": "revenue_per_month"}
        }
      }
    }
  }
}
```

</details>

---

## **Question 15 — Async Search + Runtime Field (Searching Data)**

Index `transactions` contains:

- `amount`
- `tax`

**Requirements:**

- Define runtime field:
  - `total = amount + tax`
- Perform an **asynchronous search**:
  - Return documents where `total > 1000`

**Task:**

Write the async search request.

<details>
<summary><strong>Answer</strong></summary>

```
POST transactions/_async_search?wait_for_completion_timeout=1s
{
  "query": {
    "range": {
      "total": {
        "gt": 1000
      }
    }
  },
  "runtime_mappings": {
    "total": {
      "type": "double",
      "script": {
        "source": "emit(doc['amount'].value + doc['tax'].value)"
      }
    }
  },
  "fields": [
    "total"
  ]
}
```

</details>

---

## **Question 16 — Search After + Deep Pagination (Developing Search Applications)**

Index `logs` contains millions of documents.

**Requirements:**

- Retrieve results sorted by:
  - `@timestamp` ascending
- Implement deep pagination using `search_after`
- Return next page after:
  - `@timestamp = 2026-01-01T00:00:00`
  - `_id = "abc123"`

**Task:**

Write the search request.

<details>
<summary><strong>Answer</strong></summary>

```
GET logs/_search
{
  "search_after": [
    1767225600,
    "v0__uZwB800tdfnSDlO7"
  ],
  "sort": [
    {
      "@timestamp": {
        "order": "asc"
      }
    },
    {
      "_id": {
        "order": "asc"
      }
    }
  ]
}
```

</details>

---

## **Question 17 — Index Aliases with Write Index (Developing Search Applications)**

You are implementing zero-downtime reindexing.

**Requirements:**

- Alias: `orders`
- Current index: `orders-v1`
- New index: `orders-v2`
- After migration:
  - Writes go to `orders-v2`
  - Reads go to both indices

**Task:**

Update aliases to reflect this setup.

<details>
<summary><strong>Answer</strong></summary>

First adding: current index as write index to alias orders.

```
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "orders-v1",
        "alias": "orders",
        "is_write_index": true
      }
    }
  ]
}
```

Migration:

```
PUT orders-v2/

POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "orders-v2",
        "alias": "orders",
        "is_write_index": true
      }
    },
    {
      "add": {
        "index": "orders-v1",
        "alias": "orders",
        "is_write_index": false
      }
    }
  ]
}
```

</details>

---

## **Question 18 — Ingest Pipeline with Conditional Logic (Data Processing)**

Incoming logs contain:

- `status_code`
- `message`

**Requirements:**

- If `status_code >= 500`:
  - Add field `severity = "error"`
- Otherwise:
  - Add field `severity = "info"`
- Remove field `message` after processing

**Task:**

Define the ingest pipeline.

<details>
<summary><strong>Answer</strong></summary>

```
PUT _ingest/pipeline/my-custom-ingest-pipeline
{
  "processors": [
    {
      "script": {
        "lang": "painless",
        "source": "if(ctx.status_code >= 500){\r\n  ctx.severity = \"error\"\r\n} else {\r\n  ctx.severity=\"info\"\r\n}"
      }
    },
    {
      "remove": {
        "field": "message"
      }
    }
  ]
}
```

</details>

---

## **Question 19 — Update By Query with Script Logic (Data Processing)**

Index `employees` contains:

- `salary`
- `bonus`

**Requirements:**

- For all documents:
  - Add field `total_compensation = salary + bonus`
- Only update documents where:
  - `bonus` exists

**Task:**

Write the update by query request.

<details>
<summary><strong>Answer</strong></summary>

```
POST employees/_update_by_query
{
  "query": {
    "exists": {
      "field": "bonus"
    }
  },
  "script": {
    "source": "ctx._source.total_compensation = ctx._source.salary + ctx._source.bonus",
    "lang": "painless"
  }
}
```

</details>

---

## **Question 20 — Cross-Cluster + Searchable Snapshot (Cluster Management)**

You have:

- Remote cluster: `archive_cluster`
- Snapshot repository: `cold-repo`

**Requirements:**

- Configure remote cluster
- Mount a **searchable snapshot**:
  - Snapshot: `logs-snap-01`
  - Index: `logs-2025`
- Perform a search on mounted index from local cluster

**Task:**

Write:

1. Remote cluster configuration

      <details>
   <summary><strong>Answer</strong></summary>

   ```
   PUT _cluster/settings
   {
     "persistent": {
       "cluster": {
         "remote": {
           "archive_cluster": {
             "skip_unavailable": true,
             "mode": "sniff",
             "proxy_address": null,
             "proxy_socket_connections": null,
             "server_name": null,
             "seeds": [
               "10.22.23.24:9200"
             ],
             "node_connections": 3
           }
         }
       }
     }
   }
   ```

   </details>

2. Mount searchable snapshot request

      <details>
   <summary><strong>Answer</strong></summary>

   ```
   POST /_snapshot/cold-repo/logs-snap-01/_mount?wait_for_completion=true
   {
     "index": "logs-2025",
     "renamed_index": "mounted-logs-2025",
     "index_settings": {
       "index.number_of_replicas": 0
     },
     "ignore_index_settings": [ "index.refresh_interval" ]
   }
   ```

   </details>

3. Cross-cluster search query

      <details>
   <summary><strong>Answer</strong></summary>

   ```
   POST archive_cluster:mounted-logs-2025/_search
   ```

</details>

---
