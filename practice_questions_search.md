# Practice Questions for Search

---

# 🔹 Task 1 — Correct field targeting (text vs keyword)

Search for orders where:

- `category` contains “Men's Clothing”
- but only return exact matches (no tokenization issues)

👉 Constraint:

- You must **fix the common mistake** of using `category` instead of `category.keyword`.

#### Answer

```
GET kibana_sample_data_ecommerce/_search
{
    "size": 10,
    "query": {
        "term": {
            "category.keyword": "Men's Clothing"
        }
    }
}
```

---

# 🔹 Task 2 — Filtering vs scoring (bool query)

Extend Task 1:

- Only include orders where:
  - `currency = "EUR"`
  - `total_quantity > 2`

👉 Requirements:

- Use `filter` for structured data
- Keep text search in `must`

#### Answer

```
GET kibana_sample_data_ecommerce/_search
{
    "query": {
        "bool": {
            "must": [
              {
                "term": {
                  "currency": "EUR"
                }
              }
            ],
            "filter": [
              {
                "range": {
                  "total_quantity": {
                    "gt": 2
                  }
                }
              }
            ]
        }
    }
}
```

---

# 🔹 Task 3 — Working with nested-like arrays (products.\*)

Find orders where:

- At least one product has:
  - `products.product_name` matching “shirt”
  - AND `products.price > 50`

⚠️ Trick (important for exam):

- `products` is **NOT nested**
- You must deal with **cross-object matching issue**

👉 Question:

- How would you _detect incorrect matches_ caused by this mapping?

#### Answer

It was not easy to deal with cross-object matching issue. It needs the mapping to be nested by design. The existing kibana sample ecommerce dataset does not have nested data type there.
For more information, see nested field type [reference](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested).

---

# 🔹 Task 4 — Sorting + pagination (search_after)

From Task 3 results:

- Sort by:
  1. `order_date` descending
  2. `total_quantity` descending

Then:

- Retrieve page 2 using `search_after`

👉 Constraint:

- Do NOT use `from/size`

#### Answer

In search_after, you keep the value returned by the hits from previous results in “sort”. The sort consists of list of values specified on which to sort.

```
GET kibana_sample_data_ecommerce/_search
{
    "query": {
        "match": {
          "category": "clothing"
        }
    },
    "sort": [
      {
        "order_date": {
          "order": "desc"
        }
      },
      {
        "total_quantity": {
            "order": "desc"
        }
      },
      {
        "_score": {
            "order": "desc"
        }
      }
    ],
    "_source": false,
    "fields": [
      "category", "products.product_name", "order_date", "total_quantity"
    ],
      **"search_after": [ # Look HERE
          1774124467000,
          4,
          0.12776625
        ]**
}
```

To retrieve the next page of results, repeat the request, take the `sort` values from the last hit, and insert those into the `search_after` array:

    			`GET twitter/_search
    				{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    },
    "search_after": [1463538857, "654323"],
    "sort": [
        {"date": "asc"},
        {"tie_breaker_id": "asc"}
    ]

}`

Repeat this process by updating the `search_after` array every time you retrieve a new page of results. If a [refresh](https://www.elastic.co/docs/manage-data/data-store/near-real-time-search)
occurs between these requests, the order of your results may change,
causing inconsistent results across pages. To prevent this, you can
create a [point in time (PIT)](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-open-point-in-time) to preserve the current index state over your searches.

```jsx
GET /_search
{
  "size": 10000,
  "query": {
    "match" : {
      "user.id" : "elkbee"
    }
  },
  "pit": {
    "id":  "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4BXV1aWQxAgZub2RlXzEAAAAAAAAAAAEBYQADaWR5BXV1aWQyKgZub2RlXzIAAAAAAAAAAAwBYgACBXV1aWQyAAAFdXVpZDEAAQltYXRjaF9hbGw_gAAAAA==",
    "keep_alive": "1m"
  },
  "sort": [
    {"@timestamp": {"order": "asc", "format": "strict_date_optional_time_nanos"}},
    {"_shard_doc": "desc"}
  ]
}
```

---

# 🔹 Task 5 — Aggregation fundamentals

From filtered results:

- Get:
  - Total revenue (`sum` of `taxful_total_price`)
  - Average order value
  - Total number of orders

👉 Twist:

- Ensure missing values don’t break your aggregation

#### Answer

```
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "total_taxful_price": {
      "sum": {
        "field": "taxful_total_price",
        "missing": 0
      }
    },
    "average_taxful_price": {
      "avg": {
        "field": "taxful_total_price",
        "missing": 0
      }
    },
    "count_of_orders": {
      "value_count": {
        "field": "taxful_total_price"
      }
    }
  }
}
```

---

# 🔹 Task 6 — Bucket + sub-aggregation

Group results by:

- `geoip.country_iso_code`

Inside each country:

- Get:
  - Average `taxful_total_price`
  - Top 3 `manufacturer.keyword`

👉 Key skill:

- Multi-level aggregation
- Correct `.keyword` usage

---

Now analyze products:

- Group by:
  - `products.category.keyword`

#### Answer

```
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "group_by_country": {
      "terms": {
        "field": "geoip.country_iso_code"
      },
      "aggs": {
        "average_taxful_total": {
          "avg": {"field": "taxful_total_price"}
        },
        "top_manufacturer": {
          "terms": {
            "field": "manufacturer.keyword",
            "size": 3
          }
        }
      }
    }
  }
}

```

---

# 🔹 Task 7 — Runtime field (exam favorite)

Create a runtime field:

```
order_size:
- "small" if total_quantity <= 2
- "medium" if 3–5
- "large" if >5
```

Then:

- Aggregate orders by this runtime field

👉 Key skill:

- `runtime_mappings`
- Using scripted logic in aggregations

#### Answer

```
GET /kibana_sample_data_ecommerce/_search
{
  "runtime_mappings": {
    "order_size": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['total_quantity'].value <= 2) {emit('small');}
          else if ((doc['total_quantity'].value <= 3 )&& (doc['total_quantity'].value <= 5)) { emit('medium');}
          else {emit('large');}
          """
      }
    }
  },
  "query": {
    "term": {
      "order_size": {
        "value": "small"
      }
    }
  },
  "_source": false,
  "fields": [
    "order_size"
  ]
}
```

---

# 🔹 Task 8 — Advanced filtering + date math

Find orders:

- From last 30 days
- Where:
  - `customer_gender = "F"`
  - AND at least one product has discount > 20%

Then:

- Aggregate by `day_of_week`

👉 Concepts tested:

- date math (`now-30d`)
- combining root + array conditions

#### Answer

```
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "customer_gender": {
              "value": "FEMALE"
            }
          }
        },
        {
          "range": {
            "products.min_price": {
              "gte": 20
            }
          }
        }
      ],
      "filter": [
        {
          "range": {
            "order_date": {
              "gte": "now-30d"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "agg_by_day_of_week": {
      "terms": {"field":"day_of_week"}
    }
  }
}
```
