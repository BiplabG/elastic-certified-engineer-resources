# Model 3: Exam Questions from ChatGPT

---

### **Question 1: Index Design + Dynamic Templates + Multi-fields**

You are designing an index for an e-commerce platform that stores product data with the following characteristics:

- Fields:
  - `product_id` (unique identifier)
  - `name` (full-text searchable, supports partial matching)
  - `category` (exact match filtering and aggregations)
  - `price` (sortable and range queries)
  - `attributes` (dynamic object with unknown keys; values should be searchable as keywords and full-text)
  - `created_at` (timestamp)

**Tasks:**

1. Create an index with appropriate mappings and settings to support the requirements.
2. Define a **dynamic template** for the `attributes` field so that:
   - All string fields are mapped as both `text` and `keyword`.
3. Configure **multi-fields** for `name` to support:
   - Full-text search
   - Exact match sorting
4. Ensure `category` supports efficient aggregations.

<details>
<summary><strong>Answer</strong></summary>

```
PUT e-commerce-index/
{
  "mappings": {
    "properties": {
      "product_id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "category": {
        "type": "keyword",
        "doc_values": true
      },
      "price": {
        "type": "double",
        "doc_values": true
      },
      "created_at": {
        "type": "date"
      }
    },
    "dynamic_templates": [{
        "attributes_matching": {
            "match_mapping_type": "string",
            "path_match": "attributes.*",
            "mapping": {
                "type": "text",
                "fields": {
                    "keyword": {
                        "type": "keyword"
                    }
                }
            }
        }
    }]
  }
}
```

</details>

---

### **Question 2: Index Lifecycle Management + Data Streams**

You are ingesting time-series logs from an application. Requirements:

- Data arrives continuously and should be stored in a **data stream**.
- Hot phase: data is searchable and indexed normally.
- Warm phase after 7 days: reduce replica count and optimize storage.
- Delete phase after 30 days.

**Tasks:**

1. Define an **ILM policy** matching the requirements.
2. Create an **index template** that:
   - Creates a data stream
   - Applies the ILM policy
   - Defines a mapping for:
     - `@timestamp`
     - `log_level` (keyword)
     - `message` (text)
3. Ensure new data automatically flows into the data stream.

<details>
<summary><strong>Answer</strong></summary>

```
PUT _ilm/policy/my-custom-ilm-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "1gb",
            "max_primary_shard_docs": 1000
          },
          "set_priority": {
            "priority": 100
          }
        },
        "min_age": "0ms"
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "shrink": {
            "number_of_shards": 1,
            "allow_write_after_shrink": false
          },
          "forcemerge": {
            "max_num_segments": 1,
            "index_codec": "best_compression"
          },
          "allocate": {
            "number_of_replicas": 1
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

```
PUT _index_template/my-datastream
{
  "template": {
    "settings": {
      "index.lifecycle.name": "my-custom-ilm-policy",
      "index": {
        "mode": "standard"
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "strict": true
        },
        "log_level": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        }
      }
    }
  },
  "index_patterns": [
    "my-continuous-data*"
  ],
  "data_stream": {}
}
```

```
POST my-continuous-data-now/_doc
{
    "@timestamp": "2026-03-03",
    "log_level": "INFO",
    "message": "Hello world"
}
```

</details>

---

### **Question 3: Complex Search Query + Boolean Logic + Filters**

You have an index `articles` with fields:

- `title` (text)
- `content` (text)
- `tags` (keyword)
- `published_date` (date)
- `views` (integer)

**Tasks:**

Write a query that:

1. Searches for articles where:
   - `title` or `content` contains the phrase “machine learning”
2. Filters results:
   - `published_date` within last 1 year
   - `tags` includes either "ai" or "data-science"
3. Boost results where:
   - `views` > 1000
4. Exclude:
   - Articles tagged with "deprecated"

<details>
<summary><strong>Answer</strong></summary>

```
GET articles/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "fields": [
                  "content",
                  "title"
                ],
                "query": "machine learning",
                "type": "phrase"
              }
            }
          ],
          "must_not": [
            {
                "term": {
                  "tags": {
                    "value": "deprecated"
                  }
                }
            }
          ],
          "filter": [
            {
              "range": {
                "published_date": {
                  "gte": "now-1y"
                }
              }
            },
            {
              "terms": {
                "tags": [
                  "ai",
                  "data-science"
                ]
              }
            }
          ]
        }
      },
      "functions": [
        {
            "filter": {
                "range": {
                  "views": {
                    "gt": 1000
                  }
                }
            },
            "weight": 10
        }
      ],
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

### **Question 4: Aggregations + Sub-aggregations**

You are analyzing sales data stored in `sales_index` with fields:

- `region` (keyword)
- `product` (keyword)
- `price` (double)
- `quantity` (integer)
- `timestamp` (date)

**Tasks:**

1. Create an aggregation that:
   - Groups by `region`
2. Within each region:
   - Compute total revenue (`price * quantity`)
   - Identify top 3 products by total revenue
3. For each top product:
   - Calculate average quantity sold

<details>
<summary><strong>Answer</strong></summary>

```
GET sales_index/_search
{
  "runtime_mappings": {
    "total_revenue": {
      "type": "double",
      "script": {
        "source": "emit(doc['price'].value * doc['quantity'].value)"
      }
    }
  },
  "size": 0,
  "aggs": {
    "group_by_region": {
      "terms": {
        "field": "region"
      },
      "aggs": {
        "total_revenue": {
          "sum": {
            "field": "total_revenue"
          }
        },
        "top_three_products": {
          "terms": {
            "field": "product",
            "order": {
              "total_revenue.value": "desc"
            },
            "size": 3
          },
          "aggs": {
            "total_revenue": {
              "sum": {
                "field": "total_revenue"
              }
            },
            "average_quantity": {
              "avg": {
                "field": "quantity"
              }
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

### **Question 5: Runtime Fields + Painless Scripting**

You have an index `orders` with fields:

- `unit_price` (double)
- `quantity` (integer)
- `discount` (double, optional)

**Tasks:**

1. Define a **runtime field** called `total_price` that:
   - Computes `(unit_price * quantity) - discount`
   - Handles missing `discount` values
2. Execute a search query that:
   - Filters orders where `total_price > 500`
3. Sort results by `total_price` descending

<details>
<summary><strong>Answer</strong></summary>

```
GET orders/_search
{
    "runtime_mappings": {
      "total_price": {
        "type": "double",
        "script": {
            "source": "emit((doc['unit_price'].value * doc['quantity'].value) - field('discount').get(0))"
        }
      }
    },
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

### **Question 6: Pagination + Sorting + Aliases**

You manage an index `customers_v1` and plan to migrate to `customers_v2`.

**Tasks:**

1. Create an **alias** `customers_current` pointing to `customers_v1`
2. Write a query using the alias that:
   - Sorts customers by `last_purchase_date` descending
   - Implements pagination (page size = 20, page 3)
3. Switch the alias to `customers_v2` without downtime

<details>
<summary><strong>Answer</strong></summary>

```
POST _aliases
{
    "actions": [
      {
        "add": {
          "index": "customers_v1",
          "alias": "customers_current",
          "is_write_index": true
        }
      }
    ]
}
```

```
GET customers_current/_search
{
    "size": 20,
    "from": 40,
    "sort": [
      {
        "last_purchase_date": {
          "order": "desc"
        }
      }
    ]
}
```

```
PUT customers_v2/
POST _aliases
{
    "actions": [
      {
        "remove": {
          "index": "customers_v1",
          "alias": "customers_current"
        }
      },
      {
        "add": {
          "index": "customers_v2",
          "alias": "customers_current",
          "is_write_index": true
        }
      }
    ]
}
```

</details>

---

### **Question 7: Ingest Pipeline + Data Processing**

Incoming documents contain:

- `full_name` (e.g., "John Doe")
- `email`
- `timestamp`

**Tasks:**

1. Create an **ingest pipeline** that:
   - Splits `full_name` into `first_name` and `last_name`
   - Converts `email` to lowercase
   - Adds a new field `ingest_time` with current timestamp
2. Apply the pipeline when indexing documents
3. Ensure mapping supports efficient querying of new fields

<details>
<summary><strong>Answer</strong></summary>

```
PUT _ingest/pipeline/my-custom-exam-ingest-pipeline
{
  "processors": [
    {
      "grok": {
        "field": "full_name",
        "patterns": [
          "%{WORD:first_name} %{WORD:last_name}"
        ]
      }
    },
    {
      "lowercase": {
        "field": "email"
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": [
          "ISO8601",
          "UNIX"
        ],
        "target_field": "ingest_time"
      }
    }
  ]
}
```

```
PUT my-ingesting-index/
{
    "settings": {
        "index.default_pipeline": "my-custom-exam-ingest-pipeline"
    },
    "mappings": {
        "properties": {
            "full_name": {
                "type": "text"
            },
            "first_name":{
                "type": "keyword"
            },
            "last_name": {
                "type": "keyword"
            },
            "email": {
                "type": "keyword"
            },
            "timestamp": {
                "type": "date"
            },
            "ingest_time": {
                "type": "date"
            }
        }
    }
}
```

</details>

---

### **Question 8: Reindex + Update By Query**

You have an index `products_old` with:

- `price` stored as a string
- Field `category` inconsistently capitalized

**Tasks:**

1. Create a new index `products_new` with:
   - Correct mapping for `price` as numeric
2. Use **Reindex API** to:
   - Convert `price` to numeric
3. Use **Update By Query API** to:
   - Normalize `category` to lowercase in the new index

<details>
<summary><strong>Answer</strong></summary>

```
PUT products_new/
{
    "mappings": {
        "properties": {
            "price": {
                "type": "double"
            },
            "category": {
                "type": "keyword"
            }
        }
    }
}
```

```
POST _reindex/
{
    "source": {
        "index": "products_old"
    },
    "dest": {
        "index": "products_new"
    }
}
```

```
PUT _ingest/pipeline/lowercase-custom
{
  "processors": [
    {
      "lowercase": {
        "field": "category"
      }
    }
  ]
}

POST products_new/_update_by_query?pipeline=lowercase-custom
```

</details>

---

### **Question 9: Cross-Cluster Search + Asynchronous Search**

You have two clusters:

- `cluster_a` (local)
- `cluster_b` (remote)

Both contain an index `logs`.

**Tasks:**

1. Configure cross-cluster search so `cluster_b` is accessible
2. Write a query that:
   - Searches across both clusters
   - Finds logs containing "error"
3. Execute the search as an **asynchronous search**
4. Retrieve partial results before completion

<details>
<summary><strong>Answer</strong></summary>

```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_b": {
          "skip_unavailable": true,
          "mode": "sniff",
          "proxy_address": null,
          "proxy_socket_connections": null,
          "server_name": null,
          "seeds": [
            "10.22.22.12:9200"
          ],
          "node_connections": 3
        }
      }
    }
  }
}
```

```
POST /logs,cluster_b:logs/_async_search?wait_for_completion_timeout=1s
{
    "query": {
        "match": {
          "category": "clothing"
        }
    }
}
```

```
GET /_async_search/FlZ3R2dPczVfVFNxOHpkZlJHTHFkakEeREVGZ2JTQ0tUUWViWVgyN0R6LXUtQToxNjMxMDI4?
      return_intermediate_results=true
```

</details>

---

### **Question 10: Cluster Management + Snapshots + Shard Issues**

A cluster is in **yellow/red state** due to shard allocation issues.

**Tasks:**

1. Diagnose the cause of unassigned shards
2. Provide steps to restore cluster health
3. Configure a **snapshot repository**
4. Create a **snapshot policy (SLM)** to:
   - Run daily backups
5. Restore a specific index from snapshot
6. Make the snapshot **searchable without full restore**

<details>
<summary><strong>Answer</strong></summary>

GET \_cluster/health

GET \_cat/indices?v&health=yellow

GET \_cat/shards/broken-index?v

```
GET _cluster/allocation/explain
{
    "index": "broken-index",
    "shard": 0,
    "primary": true
}
```

Solve the problem based on the reason in the cluster allocation or index issue. Can be in the number of replicas, allocation settings, disk watermarks among many of the issues.

```
PUT /_snapshot/my_repository
{
  "type": "fs",
  "settings": {
    "location": "my_backup_location"
  }
}
```

```bash
PUT /_slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "my_repository",
  "config": {
    "indices": "*",
    "ignore_unavailable": false,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

```
POST /_snapshot/my_repository/snapshot_2/_restore?wait_for_completion=true
{
  "indices": "index_2",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "index_(.+)",
  "rename_replacement": "restored_index_$1",
  "include_aliases": false
}
```

Mount a searchable snapshot

```
POST /_snapshot/my_repository/my_snapshot/_mount?wait_for_completion=true
{
  "index": "my_docs",
  "renamed_index": "docs",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [
    "index.refresh_interval"
  ]
}
```

</details>

---
