# Model 1: Exam Questions from ChatGPT

---

## **Question 1 — Index Design + Dynamic Templates (Data Management)**

You are onboarding logs from multiple applications where fields may vary dynamically.

**Requirements:**

- Create an index `app-logs-000001` with:
  - `@timestamp` as `date`
  - `service.name` as `keyword`

- Any new string field under `labels.*` must:
  - Be mapped as `keyword`

- Any other string field should:
  - Be mapped as `text` with a `keyword` sub-field

- Disable indexing for any field under `debug.*`

**Task:**

Define the index with appropriate mappings and dynamic templates.

<details>
<summary><strong>Answer</strong></summary>

```
PUT /app-logs-000001
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "service": {
        "properties": {
	        "name": {"type": "keyword"}
        }
      }
    },
    "dynamic_templates": [
      {
        "labels_as_kw": {
          "match_mapping_type": "string",
          "path_match": "labels.*",
          "mapping": {
            "type": "keyword"
          }
        }
      },
      {
        "debug_fields": {
          "path_match": "debug.*",
          "mapping": {
	          "type": "text",
            "index": false
          }
        }
      },
      {
        "other_strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
          }
        }
      }

    ]
  }
}
```

</details>

---

## **Question 2 — ILM + Data Streams (Data Management)**

You are storing time-series metrics data.

**Requirements:**

- Create a data stream named `metrics-system`
- Define an ILM policy:
  - Hot phase: rollover after 50GB or 1 day
  - Warm phase: shrink to 1 shard and force merge to 1 segment
  - Delete phase: after 30 days

- Ensure the template:
  - Applies to `metrics-system*`
  - Uses the ILM policy
  - Has `@timestamp` mapped correctly

**Task:** Create all required components so that new data ingested into the data stream follows this lifecycle.

<details>
<summary><strong>Answer</strong></summary>

```jsx
PUT _ilm/policy/metrics-system-custom-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": {
            "priority": 100
          }
        },
        "min_age": "0ms"
      },
      "warm": {
        "min_age": "0m",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "shrink": {
            "number_of_shards": 1,
            "allow_write_after_shrink": false
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

```jsx
PUT _index_template/metrics-system-custom-template
{
  "template": {
    "settings": {
      "index.lifecycle.name": "metrics-system-custom-policy",
      "index": {
        "mode": "standard"
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "index": true,
          "ignore_malformed": false,
          "doc_values": true,
          "store": false
        }
      }
    }
  },
  "index_patterns": [
    "metrics-system*"
  ],
  "data_stream": {}
}
```

```jsx
POST metrics-system/_doc{  "@timestamp": "2026-02-02",  "name": "Biplab",  "salary": 231231}
```

</details>

---

## **Question 3 — Complex Query + Boolean Logic (Searching Data)**

You have an index `ecommerce`.

**Requirements:**

Write a query that:

- Searches for products where:
  - `name` contains “laptop” OR “notebook”

- Filters:
  - `price` between 800 and 2000
  - `availability` is `in_stock`

- Must NOT include:
  - Products with `brand` = "BrandX"

- Boost products where:
  - `rating` >= 4.5

**Task:**

Construct the full query.

<details>
<summary><strong>Answer</strong></summary>

```
GET ecommerce/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "should": [
            {
              "match": {
                "name": "laptop"
              }
            },
            {
	            "match": {
		            "name": "notebook"
	            }
            }
          ],
          "filter": [
            {
              "range": {
                "price": {
                  "gte": 800,
                  "lte": 2000
                }
              }
            },
            {
              "term": {
                "availability": "in_stock"
              }
            }
          ],
          "must_not": [
            {
              "term": {
                "brand": {
                  "value": "BrandX"
                }
              }
            }
          ]
        }
      },
      "functions": [
        {
          "filter": {
            "range": {
              "rating": {
                "gte": 4.5
              }
            }
          },
          "weight": 10
        }
      ],
      "boost_mode": "multiply"
    }
  }
}
```

</details>

---

## **Question 4 — Aggregations + Sub-Aggregations (Searching Data)**

You are analyzing logs in `web-logs`.

**Requirements:**

- Group logs by `response.status_code`
- For each status code:
  - Calculate average `response_time`
  - Find top 3 `url.path` values (by count)

- Only include logs from the last 24 hours

**Task:**

Write the aggregation query with appropriate filtering and sub-aggregations.

<details>
<summary><strong>Answer</strong></summary>

```
GET web-logs/_search
{
  "size": 0,
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-1d"
      }
    }
  },
  "aggs": {
    "logs_by_status_code": {
      "terms": {
        "field": "response.status_code"
      },
      "aggs": {
        "average_response_time": {
          "avg": {
            "field": "response_time"
          }
        },
        "top_url_paths": {
          "terms": {
            "field": "url.path",
            "size": 3,
            "order": {
              "_count": "desc"
            }
          }
        }
      }
    }
  }
}
```

</details>

---

## **Question 5 — Runtime Fields + Search (Searching Data)**

<details>
<summary><strong>Answer</strong></summary>

```
PUT orders/
{
  "mappings": {
    "properties": {
      "price": {
        "type": "double"
      },
      "quantity": {
        "type": "integer"
      }
    },
    "runtime": {
      "total_price": {
        "type": "double",
        "script": {
          "source": """
            emit(doc['price'].value * doc['quantity'].value)
            """
        }
      }
    }
  }
}
```

```
GET orders/_search
{
  "query": {
    "range": {
      "total_price": {
        "gt": 500
      }
    }
  },
  "sort": [
    {
      "total_price": {
        "order": "desc"
      }
    }
  ]
}
```

</details>

---

## **Question 6 — Pagination + Sorting + Aliases (Developing Search Applications)**

### 1. Create the alias

<details>
<summary><strong>Answer</strong></summary>

```
PUT articles-v1/
{
  "aliases": {
    "articles-current": {
      "is_write_index": true
    }
  }
}
```

</details>

### 2. Paginated search query

<details>
<summary><strong>Answer</strong></summary>

```
GET articles-current/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch",
      "fields": ["title", "author"]
    }
  },
  "sort": [
    {
      "publish_date": {
        "order": "desc"
      }
    },
    {
      "_score": {
        "order": "desc"
      }
    }
  ],
  "from": 40,
  "size": 20
}
```

</details>

---

## **Question 7 — Reindex + Update By Query (Data Processing)**

<details>
<summary><strong>Answer</strong></summary>

```
POST _reindex/
{
  "source": {
    "index": "users-old"
  },
  "dest": {
    "index": "users-new"
  },
  "script": {
    "source": "ctx._source.name = ctx._source.remove('fullname')"
  }
}
```

```
POST users-new/_update_by_query
{
  "query": {
    "term": {
      "status.keyword": {
        "value": "inactive"
      }
    }
  },
  "script": {
    "source": "ctx._source.archived=true",
    "lang": "painless"
  }
}
```

</details>

---

## **Question 8 — Ingest Pipeline + Multi-fields (Data Processing)**

### 1. Ingest pipeline

<details>
<summary><strong>Answer</strong></summary>

```
PUT _ingest/pipeline/test-ingest-pipeline
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{IP:source_ip} %{WORD:action}"
        ]
      }
    },
    {
      "user_agent": {
        "field": "user_agent"
      }
    }
  ]
}
```

</details>

### 2. Mapping

<details>
<summary><strong>Answer</strong></summary>

```
PUT test-index/
{
  "mappings": {
    "properties": {
      "source_ip": {
        "type": "ip"
      },
      "action": {
        "type": "keyword"
      },
      "user_agent": {
   	      "properties": {
   		        "type": "text",
   		        "fields": {
   		          "keyword": {
   		            "type": "keyword"
   		          }
   		      }
   	      }
         }
       }
     }
```

</details>

---

## **Question 9 — Cluster Health + Snapshot Restore (Cluster Management)**

### 1. Diagnose

<details>
<summary><strong>Answer</strong></summary>

```
GET _cluster/health
GET _cat/indices?v&health=yellow

GET _cat/shards/test-index?v

GET _cluster/allocation/explain
{
  "index": "test-index",
  "shard": 0,
  "primary": false
}

GET test-index/_settings
```

</details>

### 2. Fix allocation

<details>
<summary><strong>Answer</strong></summary>

```
PUT test-index/_settings
{
  "number_of_replicas": 1
}
```

</details>

### 3. Restore

<details>
<summary><strong>Answer</strong></summary>

```
GET _snapshot/backup-repo/*?verbose=false
POST orders/_close
POST _snapshot/backup-repo/snaphost_01/_restore
{
  "indices": "orders",
  "include_aliases": false
}
```

</details>

---

## **Question 10 — Cross-Cluster Search + Replication + SLM (Cluster Management)**

### 1. Remote cluster config

<details>
<summary><strong>Answer</strong></summary>

```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_b": {
          "skip_unavailable": false,
          "mode": "sniff",
          "proxy_address": null,
          "proxy_socket_connections": null,
          "server_name": null,
          "seeds": [
            "10.100.22.200:9200"
          ],
          "node_connections": 3
        }
      }
    }
  }
}
```

</details>

### 2. Cross-cluster search

<details>
<summary><strong>Answer</strong></summary>

```
GET /cluster_b:logs/_search
{
  "size": 1,
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  },
  "_source": ["user.id", "message", "http.response.status_code"]
}
```

</details>

### 3. CCR setup

<details>
<summary><strong>Answer</strong></summary>

In Kibana
Stack Management >> Cross Cluster Replication >> Choose follower indices tab >> Create a follower Index.

Choose leader index “logs” from cluster b.

Enter the name of follower index “follower-logs” in cluster a.

</details>

### 4. SLM policy

<details>
<summary><strong>Answer</strong></summary>

```
PUT _slm/policy/nightly-snapshots
{
  "schedule": "0 0 2 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "backup-repo",
  "config": {
    "indices": "*",
    "include_global_state": true
  },
  "retention": {
    "expire_after": "7d",
    "min_count": 1,
    "max_count": 7
  }
}
```

</details>
