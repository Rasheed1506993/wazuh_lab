# تقرير شامل: Wazuh Indexer
## آلية التخزين والفهرسة في منظومة Wazuh EDR

---

## 1. المقدمة

### 1.1 ما هو Wazuh Indexer؟

Wazuh Indexer هو محرك بحث وفهرسة قوي مبني على OpenSearch (fork من Elasticsearch 7.10.2). يمثل المكون المسؤول عن تخزين وفهرسة جميع الأحداث الأمنية والتنبيهات التي يولدها Wazuh Manager، مما يتيح البحث السريع والتحليل العميق للبيانات الأمنية.

### 1.2 دوره في منظومة Wazuh

يعمل Indexer كطبقة التخزين الدائم (Persistent Storage Layer) في معمارية Wazuh، حيث:

- **يستقبل** آلاف الأحداث والتنبيهات الأمنية من Wazuh Manager عبر Filebeat
- **يفهرس** البيانات بطريقة منظمة تسمح بالبحث السريع في ملايين السجلات
- **يوفر** واجهة RESTful API للاستعلام عن البيانات وتحليلها
- **يدعم** Dashboard في عرض التنبيهات والإحصائيات الأمنية
- **يحتفظ** بالبيانات التاريخية للتحليل الجنائي والامتثال

### 1.3 لماذا OpenSearch؟

تم اختيار OpenSearch لعدة أسباب تقنية:

1. **البحث بالنص الكامل (Full-Text Search)**: محرك Apache Lucene يوفر بحث فائق السرعة
2. **التوزيع الأفقي (Horizontal Scaling)**: إمكانية توزيع البيانات على عدة عُقد
3. **الموثوقية العالية**: نسخ احتياطية تلقائية (Replicas) لضمان عدم فقدان البيانات
4. **التجميع والإحصاء (Aggregations)**: إمكانيات تحليلية متقدمة
5. **Open Source**: مفتوح المصدر ومدعوم من مجتمع واسع

---

## 2. استقبال البيانات

### 2.1 مسار البيانات من Manager إلى Indexer

```
┌─────────────────┐
│  Wazuh Manager  │
│   (analysisd)   │
└────────┬────────┘
         │ يكتب
         ▼
┌─────────────────┐
│  alerts.json    │
│  archives.json  │
└────────┬────────┘
         │ يقرأ
         ▼
┌─────────────────┐
│    Filebeat     │
│  (Log Shipper)  │
└────────┬────────┘
         │ HTTP/HTTPS
         ▼
┌─────────────────┐
│ Wazuh Indexer   │
│  (OpenSearch)   │
└─────────────────┘
```

### 2.2 دور Filebeat

Filebeat هو أداة خفيفة لنقل السجلات (Log Shipper) تقوم بـ:

**الوظائف الرئيسية:**
- **المراقبة المستمرة**: يراقب ملفات `/var/ossec/logs/alerts/alerts.json`
- **القراءة الذكية**: يتتبع آخر موضع قرأه لتجنب التكرار (registry file)
- **الإرسال الآمن**: يستخدم TLS للتشفير أثناء النقل
- **إعادة المحاولة**: يعيد الإرسال تلقائيًا في حالة الفشل
- **الضغط**: يضغط البيانات قبل الإرسال لتوفير bandwidth

**ملف التكوين الأساسي** (`/etc/filebeat/filebeat.yml`):

```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: false

output.elasticsearch:
  hosts: ["https://indexer:9200"]
  protocol: "https"
  username: "admin"
  password: "${INDEXER_PASSWORD}"
  ssl.certificate_authorities: ["/etc/filebeat/certs/root-ca.pem"]
  
  # Bulk settings
  bulk_max_size: 2048
  worker: 4
```

### 2.3 شكل البيانات (JSON Structure)

**مثال على تنبيه أمني حقيقي:**

```json
{
  "timestamp": "2025-10-19T14:30:45.123+0000",
  "rule": {
    "level": 10,
    "description": "Possible SSH brute force attack",
    "id": "5551",
    "firedtimes": 5,
    "mail": true,
    "groups": ["syslog", "sshd", "authentication_failed"]
  },
  "agent": {
    "id": "001",
    "name": "web-server-01",
    "ip": "192.168.1.100"
  },
  "manager": {
    "name": "wazuh-manager"
  },
  "data": {
    "srcip": "203.0.113.45",
    "srcport": "54321",
    "dstuser": "root"
  },
  "decoder": {
    "name": "sshd"
  },
  "location": "/var/log/auth.log",
  "full_log": "Failed password for root from 203.0.113.45 port 54321 ssh2"
}
```

### 2.4 البروتوكول والتنسيق

- **البروتوكول**: HTTPS (TLS 1.2+)
- **الطريقة**: POST إلى `/_bulk` API
- **التنسيق**: NDJSON (Newline Delimited JSON)
- **المصادقة**: Basic Authentication أو API Key
- **معدل الإرسال**: Batch processing (دفعات من 500-2000 document)

---

## 3. Index Templates

### 3.1 ما هي Index Templates؟

Index Templates هي قوالب تعريفية تحدد كيف يتم إنشاء indices الجديدة تلقائيًا. عندما يصل document جديد، يتحقق Indexer من اسم الـ index، وإذا لم يكن موجودًا، ينشئه باستخدام Template المطابق.

### 3.2 موقع Templates في الكود

في Wazuh Indexer، توجد Templates في:

```
wazuh-indexer/
├── opensearch/
│   └── templates/
│       ├── wazuh-alerts-template.json
│       ├── wazuh-archives-template.json
│       └── wazuh-statistics-template.json
```

### 3.3 مثال: wazuh-alerts Template

```json
{
  "index_patterns": ["wazuh-alerts-4.x-*"],
  "priority": 1,
  "template": {
    "settings": {
      "index": {
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "refresh_interval": "5s",
        "codec": "best_compression",
        "max_result_window": 100000,
        "mapping": {
          "total_fields": {
            "limit": 10000
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "strict_date_optional_time||epoch_millis"
        },
        "rule": {
          "properties": {
            "id": {"type": "keyword"},
            "level": {"type": "long"},
            "description": {"type": "text"}
          }
        }
      }
    }
  }
}
```

### 3.4 شرح الإعدادات المهمة

#### Settings:

| الإعداد | القيمة | الوظيفة |
|---------|--------|---------|
| `number_of_shards` | 3 | عدد الأجزاء التي يُقسم إليها الـ index للتوزيع |
| `number_of_replicas` | 1 | عدد النسخ الاحتياطية لكل shard |
| `refresh_interval` | 5s | معدل تحديث الـ index ليصبح البيانات الجديدة قابلة للبحث |
| `codec` | best_compression | ضغط البيانات لتوفير المساحة |
| `max_result_window` | 100000 | الحد الأقصى لعدد النتائج في استعلام واحد |

#### Index Patterns:

- `wazuh-alerts-4.x-*` يطابق indices مثل:
  - `wazuh-alerts-4.x-2025.10.19`
  - `wazuh-alerts-4.x-2025.10.20`
  
هذا يسمح بإنشاء index جديد يوميًا (Index Rotation) لتحسين الأداء وسهولة الإدارة.

---

## 4. Mappings (خرائط البيانات)

### 4.1 ما هي Mappings؟

Mappings هي schema تحدد بنية البيانات في الـ index، مثل:
- نوع كل حقل (field type)
- كيف يتم فهرسته (indexing strategy)
- كيف يتم تحليل النصوص (text analysis)

### 4.2 أهم الحقول في Wazuh Alerts

#### 4.2.1 الحقول الزمنية

```json
"@timestamp": {
  "type": "date",
  "format": "strict_date_optional_time||epoch_millis"
}
```

- **النوع**: `date`
- **الوظيفة**: تخزين وقت حدوث التنبيه
- **التنسيق**: يقبل ISO 8601 أو Unix timestamp
- **الأهمية**: أساسي لتصفية البيانات زمنيًا والرسوم البيانية

#### 4.2.2 حقول القاعدة (Rule Fields)

```json
"rule": {
  "properties": {
    "id": {
      "type": "keyword",
      "ignore_above": 256
    },
    "level": {
      "type": "long"
    },
    "description": {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    },
    "groups": {
      "type": "keyword"
    },
    "mitre": {
      "properties": {
        "id": {"type": "keyword"},
        "tactic": {"type": "keyword"},
        "technique": {"type": "keyword"}
      }
    }
  }
}
```

**الشرح:**

- **`rule.id`**: معرف القاعدة (مثل "5551")
  - نوع `keyword` للبحث الدقيق والتجميع
  
- **`rule.level`**: مستوى الخطورة (0-15)
  - نوع `long` للمقارنات الرقمية
  
- **`rule.description`**: وصف التنبيه
  - نوع `text` للبحث بالنص الكامل
  - حقل `keyword` فرعي للفرز والتجميع
  
- **`rule.groups`**: تصنيفات القاعدة
  - نوع `keyword` array للتصفية

#### 4.2.3 حقول الجهاز (Agent Fields)

```json
"agent": {
  "properties": {
    "id": {"type": "keyword"},
    "name": {"type": "keyword"},
    "ip": {"type": "ip"},
    "version": {"type": "keyword"},
    "os": {
      "properties": {
        "platform": {"type": "keyword"},
        "version": {"type": "keyword"},
        "name": {"type": "text"}
      }
    }
  }
}
```

**الشرح:**

- **`agent.ip`**: نوع خاص `ip` يسمح بـ:
  - البحث بنطاقات CIDR (مثل 192.168.1.0/24)
  - الفرز والتجميع الذكي للعناوين

#### 4.2.4 حقول البيانات الخام (Data Fields)

```json
"data": {
  "type": "object",
  "dynamic": true
}
```

- **Dynamic Mapping**: يسمح بإضافة حقول جديدة تلقائيًا
- **الوظيفة**: تخزين البيانات الخاصة بكل decoder
- **المرونة**: كل نوع حدث له حقول مختلفة

### 4.3 الفرق بين Text و Keyword

| الخاصية | Text | Keyword |
|---------|------|---------|
| **الفهرسة** | يحلل النص إلى tokens | يُخزن كما هو (exact value) |
| **البحث** | Full-text search | Exact match |
| **الاستخدام** | الوصف، الرسائل الطويلة | IDs، التصنيفات، الأسماء |
| **التجميع** | ❌ غير مناسب | ✅ مثالي |
| **الفرز** | ❌ غير مناسب | ✅ سريع |

**مثال:**

```json
"rule.description": "Possible SSH brute force attack"
```

- كـ **text**: يُفهرس كـ ["possible", "ssh", "brute", "force", "attack"]
  - يمكن البحث بـ "brute force" وستظهر النتيجة
  
- كـ **keyword**: يُفهرس كـ "Possible SSH brute force attack"
  - يجب البحث بالنص الكامل تمامًا

### 4.4 Mapping كامل مبسط

```json
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "rule": {
        "properties": {
          "id": {"type": "keyword"},
          "level": {"type": "long"},
          "description": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
          "groups": {"type": "keyword"},
          "firedtimes": {"type": "long"}
        }
      },
      "agent": {
        "properties": {
          "id": {"type": "keyword"},
          "name": {"type": "keyword"},
          "ip": {"type": "ip"}
        }
      },
      "data": {
        "properties": {
          "srcip": {"type": "ip"},
          "dstip": {"type": "ip"},
          "srcport": {"type": "long"},
          "dstport": {"type": "long"},
          "srcuser": {"type": "keyword"},
          "dstuser": {"type": "keyword"},
          "protocol": {"type": "keyword"}
        }
      },
      "location": {"type": "keyword"},
      "full_log": {"type": "text"},
      "decoder": {
        "properties": {
          "name": {"type": "keyword"},
          "parent": {"type": "keyword"}
        }
      }
    }
  }
}
```

---

## 5. عملية الفهرسة (Indexing Process)

### 5.1 خطوات تخزين Document

```
┌─────────────────────────────────────────────────────────┐
│                    Indexing Pipeline                     │
└─────────────────────────────────────────────────────────┘

1. [Document Arrival]
   ↓
2. [Routing] → حساب Shard ID من document ID
   ↓
3. [Template Matching] → اختيار Template المناسب
   ↓
4. [Mapping Application] → تطبيق schema على الحقول
   ↓
5. [Analysis] → تحليل النصوص (tokenization)
   ↓
6. [Indexing] → كتابة إلى Lucene segments
   ↓
7. [Replication] → نسخ إلى Replica shards
   ↓
8. [Response] → إرجاع نتيجة النجاح/الفشل
```

### 5.2 معالجة البيانات (Ingest Pipelines)

Wazuh يستخدم pipelines لتحسين البيانات قبل التخزين:

```json
{
  "description": "Wazuh alerts enrichment pipeline",
  "processors": [
    {
      "set": {
        "field": "event.kind",
        "value": "alert"
      }
    },
    {
      "set": {
        "field": "event.module",
        "value": "wazuh"
      }
    },
    {
      "date": {
        "field": "timestamp",
        "target_field": "@timestamp",
        "formats": ["ISO8601"]
      }
    },
    {
      "geoip": {
        "field": "data.srcip",
        "target_field": "data.srcgeoip",
        "ignore_missing": true
      }
    },
    {
      "script": {
        "source": "ctx.event.severity = ctx.rule.level"
      }
    }
  ]
}
```

**الشرح:**
- **set**: إضافة حقول ثابتة
- **date**: تحويل التاريخ إلى format موحد
- **geoip**: إضافة معلومات جغرافية لعناوين IP
- **script**: تنفيذ كود مخصص لمعالجة البيانات

### 5.3 الدوال المستخدمة (APIs)

#### 5.3.1 Index API (Single Document)

```http
POST /wazuh-alerts-4.x-2025.10.19/_doc
Content-Type: application/json

{
  "@timestamp": "2025-10-19T14:30:45.123Z",
  "rule": {
    "id": "5551",
    "level": 10,
    "description": "SSH brute force"
  },
  "agent": {
    "name": "server-01"
  }
}
```

**Response:**
```json
{
  "_index": "wazuh-alerts-4.x-2025.10.19",
  "_id": "Abc123def456",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  }
}
```

#### 5.3.2 Bulk API (Multiple Documents)

Filebeat يستخدم Bulk API لإرسال دفعات:

```http
POST /_bulk
Content-Type: application/x-ndjson

{"index":{"_index":"wazuh-alerts-4.x-2025.10.19"}}
{"@timestamp":"2025-10-19T14:30:45Z","rule":{"id":"5551","level":10}}
{"index":{"_index":"wazuh-alerts-4.x-2025.10.19"}}
{"@timestamp":"2025-10-19T14:30:46Z","rule":{"id":"5502","level":3}}
```

**مزايا Bulk API:**
- تقليل Network overhead (استدعاء واحد بدلًا من آلاف)
- تحسين Throughput بنسبة تصل إلى 10x
- Atomic operations لضمان تناسق البيانات

### 5.4 مثال كود: Python Client

```python
from opensearchpy import OpenSearch

# الاتصال بالـ Indexer
client = OpenSearch(
    hosts=['https://indexer:9200'],
    http_auth=('admin', 'password'),
    use_ssl=True,
    verify_certs=True,
    ca_certs='/path/to/root-ca.pem'
)

# فهرسة document واحد
response = client.index(
    index='wazuh-alerts-4.x-2025.10.19',
    body={
        '@timestamp': '2025-10-19T14:30:45.123Z',
        'rule': {
            'id': '5551',
            'level': 10,
            'description': 'SSH brute force attempt'
        },
        'agent': {
            'name': 'web-server-01',
            'ip': '192.168.1.100'
        },
        'data': {
            'srcip': '203.0.113.45',
            'dstuser': 'root'
        }
    }
)

print(f"Document indexed with ID: {response['_id']}")
```

---

## 6. عملية البحث (Search Process)

### 6.1 كيف يتم البحث؟

عملية البحث تمر بمراحل متعددة:

```
┌─────────────────────────────────────────────────────────┐
│                    Search Pipeline                       │
└─────────────────────────────────────────────────────────┘

1. [Query Reception] → استقبال استعلام من Client
   ↓
2. [Query Parsing] → تحليل Query DSL
   ↓
3. [Shard Selection] → اختيار Shards للبحث فيها
   ↓
4. [Scatter Phase] → إرسال Query لكل Shard
   ↓
5. [Local Search] → كل Shard يبحث محليًا
   ↓
6. [Gather Phase] → جمع النتائج من جميع Shards
   ↓
7. [Sorting & Ranking] → ترتيب النتائج حسب relevance
   ↓
8. [Response] → إرجاع Top results للـ Client
```

### 6.2 Query DSL (Domain Specific Language)

OpenSearch يستخدم JSON-based query language:

#### 6.2.1 Match Query (بحث نصي)

```json
{
  "query": {
    "match": {
      "rule.description": "brute force"
    }
  }
}
```

#### 6.2.2 Term Query (بحث دقيق)

```json
{
  "query": {
    "term": {
      "rule.id": "5551"
    }
  }
}
```

#### 6.2.3 Range Query (نطاق زمني)

```json
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2025-10-19T00:00:00",
        "lt": "2025-10-20T00:00:00"
      }
    }
  }
}
```

#### 6.2.4 Bool Query (استعلام مركب)

```json
{
  "query": {
    "bool": {
      "must": [
        {"range": {"rule.level": {"gte": 7}}}
      ],
      "filter": [
        {"term": {"agent.name": "web-server-01"}},
        {"range": {"@timestamp": {"gte": "now-24h"}}}
      ],
      "must_not": [
        {"term": {"rule.groups": "test"}}
      ]
    }
  },
  "size": 100,
  "sort": [
    {"@timestamp": "desc"}
  ]
}
```

**الشرح:**
- **must**: شروط إلزامية (تؤثر على relevance score)
- **filter**: تصفية (لا تؤثر على scoring، أسرع)
- **must_not**: استبعاد نتائج
- **should**: شروط اختيارية (تحسن الـ score)

### 6.3 أمثلة Queries شائعة

#### مثال 1: تنبيهات عالية الخطورة آخر ساعة

```json
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"rule.level": {"gte": 10}}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "sort": [{"@timestamp": "desc"}],
  "size": 50
}
```

#### مثال 2: هجمات SSH من IP معين

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"rule.groups": "sshd"}},
        {"term": {"data.srcip": "203.0.113.45"}}
      ]
    }
  },
  "aggs": {
    "attacks_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      }
    }
  }
}
```

#### مثال 3: أكثر 10 agents بتنبيهات

```json
{
  "size": 0,
  "aggs": {
    "top_agents": {
      "terms": {
        "field": "agent.name",
        "size": 10,
        "order": {"_count": "desc"}
      },
      "aggs": {
        "avg_severity": {
          "avg": {"field": "rule.level"}
        }
      }
    }
  }
}
```

### 6.4 Aggregations (التجميع والإحصاء)

Aggregations هي ميزة قوية للتحليل:

#### أنواع Aggregations:

1. **Metric Aggregations**: حسابات رقمية
   - `avg`, `sum`, `min`, `max`, `stats`
   
2. **Bucket Aggregations**: تجميع البيانات
   - `terms`, `date_histogram`, `range`
   
3. **Pipeline Aggregations**: معالجة نتائج aggregations أخرى
   - `derivative`, `cumulative_sum`, `moving_avg`

**مثال: إحصائيات يومية للتنبيهات**

```json
{
  "size": 0,
  "aggs": {
    "alerts_per_day": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1d"
      },
      "aggs": {
        "severity_stats": {
          "stats": {"field": "rule.level"}
        },
        "top_rules": {
          "terms": {
            "field": "rule.id",
            "size": 5
          }
        }
      }
    }
  }
}
```

### 6.5 الدوال المستخدمة (Search APIs)

#### Search API

```http
POST /wazuh-alerts-*/_search
Content-Type: application/json

{
  "query": {...},
  "size": 100,
  "from": 0,
  "sort": [{"@timestamp": "desc"}],
  "_source": ["rule.*", "agent.name", "@timestamp"]
}
```

**Parameters:**
- `size`: عدد النتائج (default: 10)
- `from`: offset للـ pagination
- `_source`: الحقول المطلوب إرجاعها
- `sort`: ترتيب النتائج

#### Multi-Search API

للبحث في عدة indices مرة واحدة:

```http
POST /_msearch
Content-Type: application/x-ndjson

{"index":"wazuh-alerts-*"}
{"query":{"match":{"rule.level":10}}}
{"index":"wazuh-archives-*"}
{"query":{"term":{"agent.name":"server-01"}}}
```

---

## 7. المكتبات والأدوات

### 7.1 OpenSearch Version

Wazuh Indexer يستخدم:
- **OpenSearch 2.x** (حاليًا 2.11.x)
- مبني على **Apache Lucene 9.x**
- متوافق مع Elasticsearch 7.10.2 APIs

### 7.2 Filebeat Configuration

**الإعدادات الأساسية:**

```yaml
# /etc/filebeat/filebeat.yml

filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
      var.input: file
      var.paths: ["/var/ossec/logs/alerts/alerts.json"]
    archives:
      enabled: false

output.elasticsearch:
  hosts: ["https://wazuh-indexer:9200"]
  protocol: "https"
  username: "admin"
  password: "${ELASTIC_PASSWORD}"
  
  # TLS
  ssl.certificate_authorities: ["/etc/filebeat/certs/root-ca.pem"]
  ssl.certificate: "/etc/filebeat/certs/filebeat.pem"
  ssl.key: "/etc/filebeat/certs/filebeat-key.pem"
  
  # Performance
  bulk_max_size: 2048
  worker: 4
  compression_level: 3
  
  # Index settings
  index: "wazuh-alerts-4.x-%{+yyyy.MM.dd}"
  
  # Retry
  max_retries: 3
  backoff.init: 1s
  backoff.max: 60s

# Processors
processors:
  - drop_fields:
      fields: ["ecs.version", "host.name", "input.type"]
  - rename:
      fields:
        - {from: "timestamp", to: "@timestamp"}
  - timestamp:
      field: "@timestamp"
      layouts:
        - "2006-01-02T15:04:05.999Z"
```

### 7.3 Security Features

#### Authentication Methods:

1. **Basic Authentication**
```bash
curl -u admin:password https://indexer:9200/_cluster/health
```

2. **API Keys**
```bash
# إنشاء API key
POST /_security/api_key
{
  "name": "filebeat-key",
  "role_descriptors": {
    "filebeat_writer": {
      "cluster": ["monitor"],
      "index": [
        {
          "names": ["wazuh-alerts-*"],
          "privileges": ["create_index", "write", "create"]
        }
      ]
    }
  }
}
```

#### TLS/SSL Configuration:

```yaml
# opensearch.yml
plugins.security.ssl.transport.pemcert_filepath: node-cert.pem
plugins.security.ssl.transport.pemkey_filepath: node-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: node-cert.pem
plugins.security.ssl.http.pemkey_filepath: node-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: root-ca.pem
```

#### Role-Based Access Control (RBAC):

```json
{
  "wazuh_user": {
    "cluster_permissions": ["cluster_monitor"],
    "index_permissions": [
      {
        "index_patterns": ["wazuh-alerts-*", "wazuh-archives-*"],
        "allowed_actions": ["read", "search"]
      }
    ]
  }
}
```

### 7.4 أدوات إضافية

| الأداة | الوظيفة | المستودع |
|--------|---------|----------|
| **opensearch-cli** | أداة سطر أوامر للإدارة | github.com/opensearch-project/opensearch-cli |
| **Index State Management (ISM)** | إدارة دورة حياة الـ indices | مدمج في OpenSearch |
| **Performance Analyzer** | تحليل أداء Cluster | مدمج في OpenSearch |
| **Anomaly Detection** | كشف الشذوذ بالذكاء الاصطناعي | Plugin |

---

## 8. تدفق البيانات الكامل

### 8.1 من Manager إلى التخزين النهائي

```
════════════════════════════════════════════════════════════════
                    COMPLETE DATA FLOW
════════════════════════════════════════════════════════════════

[Step 1: Event Generation]
┌─────────────────────┐
│   Wazuh Agent       │
│  (File Monitor)     │
└──────────┬──────────┘
           │ TCP/TLS 1514
           ▼
┌─────────────────────┐
│  Wazuh Manager      │
│   - remoted         │ ← استقبال الأحداث
│   - analysisd       │ ← تحليل وتطبيق القواعد
│   - alertd          │ ← توليد التنبيهات
└──────────┬──────────┘
           │ writes to disk
           ▼

[Step 2: File Writing]
┌─────────────────────────────────────┐
│  /var/ossec/logs/alerts/            │
│  ├── alerts.json                    │
│  └── archives.json                  │
└──────────┬──────────────────────────┘
           │ monitored by
           ▼

[Step 3: Log Shipping]
┌─────────────────────┐
│    Filebeat         │
│  ┌──────────────┐   │
│  │ Harvester    │   │ ← قراءة الملفات
│  └──────┬───────┘   │
│  ┌──────▼───────┐   │
│  │ Processor    │   │ ← معالجة البيانات
│  └──────┬───────┘   │
│  ┌──────▼───────┐   │
│  │ Publisher    │   │ ← إرسال دفعات
│  └──────────────┘   │
└──────────┬──────────┘
           │ HTTPS Bulk API
           ▼

[Step 4: Indexing]
┌─────────────────────────────────────┐
│      Wazuh Indexer                  │
│  ┌──────────────────────────────┐   │
│  │  Coordinator Node            │   │
│  │  - Receives bulk request     │   │
│  │  - Routes to primary shards  │   │
│  └──────────┬───────────────────┘   │
│             ▼                        │
│  ┌──────────────────────────────┐   │
│  │  Primary Shard               │   │
│  │  - Applies template          │   │
│  │  - Applies mapping           │   │
│  │  - Runs ingest pipeline      │   │
│  │  - Indexes document          │   │
│  └──────────┬───────────────────┘   │
│             │ replication            │
│             ▼                        │
│  ┌──────────────────────────────┐   │
│  │  Replica Shard               │   │
│  │  - Stores copy               │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
           │
           ▼
     [Data Persisted]

[Step 5: Querying]
┌─────────────────────┐
│  Wazuh Dashboard    │
│  or API Client      │
└──────────┬──────────┘
           │ Search Request
           ▼
┌─────────────────────────────────────┐
│      Wazuh Indexer                  │
│  ┌──────────────────────────────┐   │
│  │  Coordinator Node            │   │
│  │  - Parses query              │   │
│  │  - Scatters to shards        │   │
│  └──────────┬───────────────────┘   │
│             ▼                        │
│  ┌──────────────────────────────┐   │
│  │  Shards (parallel search)    │   │
│  │  - Execute query locally     │   │
│  │  - Return top results        │   │
│  └──────────┬───────────────────┘   │
│             │ gather results         │
│             ▼                        │
│  ┌──────────────────────────────┐   │
│  │  Coordinator Node            │   │
│  │  - Merges results            │   │
│  │  - Sorts globally            │   │
│  │  - Returns final response    │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│   Response to       │
│   Dashboard/Client  │
└─────────────────────┘
```

### 8.2 شرح تفصيلي لكل مرحلة

#### المرحلة 1: توليد التنبيه في Manager

```
1. analysisd يقرأ حدث من Agent
2. يطابقه مع القواعد (rules)
3. إذا تطابق → يولد alert
4. يُثري بمعلومات إضافية:
   - معلومات Agent
   - تصنيف MITRE ATT&CK
   - GeoIP lookup
5. يكتب في alerts.json كـ JSON line
```

**توقيت**: ~1-5ms لكل حدث

#### المرحلة 2: القراءة بواسطة Filebeat

```
1. Harvester يراقب alerts.json
2. يقرأ سطر جديد (JSON)
3. يحفظ موضعه في registry file
4. يرسل للـ Processor
```

**توقيت**: ~10-100ms latency

#### المرحلة 3: المعالجة والإرسال

```
1. Processor يطبق filters:
   - حذف حقول غير مهمة
   - إعادة تسمية حقول
   - إضافة metadata
2. Publisher يجمع events في batch
3. يضغط البيانات (gzip)
4. يرسل POST /_bulk
```

**حجم Batch**: 500-2000 document
**توقيت**: ~100-500ms لكل batch

#### المرحلة 4: الفهرسة في Indexer

```
1. Coordinator يستقبل bulk request
2. يحلل كل document:
   - يحدد target index
   - يحسب shard ID
3. يرسل لـ Primary shard المناسب
4. Primary shard:
   - يطبق template (إذا index جديد)
   - يطبق mapping
   - ينفذ ingest pipeline
   - يكتب في Lucene segments
   - يرسل للـ replicas
5. ينتظر acknowledgment من replicas
6. يرد بـ success/failure
```

**توقيت**: ~50-200ms لكل batch

#### المرحلة 5: البحث والاستعلام

```
1. Client يرسل search query
2. Coordinator:
   - يحدد indices المستهدفة
   - يختار shard من كل index
3. يرسل query بالتوازي لكل shard
4. كل shard:
   - ينفذ query محليًا
   - يرتب النتائج
   - يرجع top N results
5. Coordinator:
   - يدمج النتائج
   - يرتب عالميًا
   - يرجع final response
```

**توقيت**: ~10-1000ms حسب complexity

### 8.3 مثال Timeline كامل

```
T+0ms     : Agent يكشف تغيير في ملف
T+5ms     : Manager يستقبل الحدث
T+10ms    : analysisd يطابق القاعدة 5551
T+15ms    : يكتب alert في alerts.json
T+115ms   : Filebeat يقرأ السطر الجديد
T+120ms   : يضيفه لـ batch
T+620ms   : Batch ممتلئ (2000 doc) → إرسال
T+720ms   : Indexer يستقبل ويبدأ الفهرسة
T+820ms   : Indexer يكمل الكتابة
T+825ms   : Filebeat يستقبل acknowledgment
T+5000ms  : Refresh interval → Document قابل للبحث
T+5100ms  : Dashboard تبحث وتعرض التنبيه
```

**إجمالي Latency**: ~5 ثواني من الحدث إلى الظهور

---

## 9. المخططات التوضيحية

### 9.1 Sequence Diagram: تدفق التخزين

```
┌────────┐   ┌─────────┐   ┌──────────┐   ┌─────────┐
│ Manager│   │Filebeat │   │ Indexer  │   │ Replica │
└───┬────┘   └────┬────┘   └────┬─────┘   └────┬────┘
    │             │              │              │
    │ Write Alert │              │              │
    │─────────────>              │              │
    │             │              │              │
    │             │ Read File    │              │
    │             │<─────────────│              │
    │             │              │              │
    │             │ POST /_bulk  │              │
    │             │──────────────>              │
    │             │              │              │
    │             │        Apply Template       │
    │             │              │──┐           │
    │             │              │  │           │
    │             │              │<─┘           │
    │             │              │              │
    │             │         Apply Mapping       │
    │             │              │──┐           │
    │             │              │  │           │
    │             │              │<─┘           │
    │             │              │              │
    │             │           Index Doc         │
    │             │              │──┐           │
    │             │              │  │           │
    │             │              │<─┘           │
    │             │              │              │
    │             │              │  Replicate   │
    │             │              │──────────────>
    │             │              │              │
    │             │              │     ACK      │
    │             │              │<──────────────
    │             │              │              │
    │             │    200 OK    │              │
    │             │<──────────────              │
    │             │              │              │
```

### 9.2 Sequence Diagram: تدفق البحث

```
┌──────────┐   ┌────────────┐   ┌─────────┐   ┌────────┐
│Dashboard │   │Wazuh API   │   │ Indexer │   │ Shards │
└────┬─────┘   └─────┬──────┘   └────┬────┘   └───┬────┘
     │               │               │            │
     │ GET /alerts   │               │            │
     │───────────────>               │            │
     │               │               │            │
     │               │ POST /_search │            │
     │               │───────────────>            │
     │               │               │            │
     │               │         Parse Query        │
     │               │               │──┐         │
     │               │               │  │         │
     │               │               │<─┘         │
     │               │               │            │
     │               │               │ Scatter    │
     │               │               │───────────>│
     │               │               │            │
     │               │               │  Search    │
     │               │               │            │──┐
     │               │               │            │  │
     │               │               │            │<─┘
     │               │               │            │
     │               │               │  Results   │
     │               │               │<───────────│
     │               │               │            │
     │               │          Merge & Sort      │
     │               │               │──┐         │
     │               │               │  │         │
     │               │               │<─┘         │
     │               │               │            │
     │               │   Response    │            │
     │               │<───────────────            │
     │               │               │            │
     │  JSON Results │               │            │
     │<───────────────               │            │
     │               │               │            │
     │  Render UI    │               │            │
     │──┐            │               │            │
     │  │            │               │            │
     │<─┘            │               │            │
```

### 9.3 Component/Architecture Diagram

```
╔════════════════════════════════════════════════════════════════╗
║                    WAZUH INDEXER ARCHITECTURE                   ║
╚════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────┐
│                        DATA PRODUCERS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │ Wazuh Agent  │────────>│Wazuh Manager │                      │
│  │   (Hosts)    │  TCP    │  (analysisd) │                      │
│  └──────────────┘  1514   └──────┬───────┘                      │
│                                   │                              │
│                                   │ writes                       │
│                                   ▼                              │
│                          ┌─────────────────┐                     │
│                          │  alerts.json    │                     │
│                          │  archives.json  │                     │
│                          └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   │ monitors
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                         LOG SHIPPING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Filebeat                             │  │
│  │  ┌────────────┐  ┌───────────┐  ┌──────────────────┐     │  │
│  │  │ Harvester  │─>│ Processor │─>│ Bulk Publisher   │     │  │
│  │  └────────────┘  └───────────┘  └──────────────────┘     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   │ HTTPS /_bulk
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                       WAZUH INDEXER CLUSTER                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │               Coordinator Node (Master)                    │ │
│  │  - Cluster Management                                      │ │
│  │  - Request Routing                                         │ │
│  │  - Index/Template Management                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                    │                          │                  │
│         ┌──────────┴──────────┐      ┌───────┴──────────┐       │
│         ▼                     ▼      ▼                  ▼       │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐   │
│  │  Data Node  │       │  Data Node  │       │  Data Node  │   │
│  │             │       │             │       │             │   │
│  │ ┌─────────┐ │       │ ┌─────────┐ │       │ ┌─────────┐ │   │
│  │ │Shard 0  │ │       │ │Shard 1  │ │       │ │Shard 2  │ │   │
│  │ │(Primary)│ │       │ │(Primary)│ │       │ │(Primary)│ │   │
│  │ └─────────┘ │       │ └─────────┘ │       │ └─────────┘ │   │
│  │ ┌─────────┐ │       │ ┌─────────┐ │       │ ┌─────────┐ │   │
│  │ │Shard 1  │ │       │ │Shard 2  │ │       │ │Shard 0  │ │   │
│  │ │(Replica)│ │       │ │(Replica)│ │       │ │(Replica)│ │   │
│  │ └─────────┘ │       │ └─────────┘ │       │ └─────────┘ │   │
│  └─────────────┘       └─────────────┘       └─────────────┘   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Index Templates                         │ │
│  │  - wazuh-alerts-template                                   │ │
│  │  - wazuh-archives-template                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                       Indices                              │ │
│  │  - wazuh-alerts-4.x-2025.10.19                             │ │
│  │  - wazuh-alerts-4.x-2025.10.18                             │ │
│  │  - wazuh-archives-4.x-2025.10.19                           │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   │ REST API
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                          DATA CONSUMERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │  Wazuh API       │              │ Wazuh Dashboard  │         │
│  │  (Backend)       │◄─────────────│    (Frontend)    │         │
│  └──────────────────┘   queries    └──────────────────┘         │
│           │                                                      │
│           └────────> GET /_search                                │
│                      POST /_msearch                              │
│                      GET /_count                                 │
└─────────────────────────────────────────────────────────────────┘

╔════════════════════════════════════════════════════════════════╗
║                      KEY COMPONENTS                             ║
╠════════════════════════════════════════════════════════════════╣
║ Shards: Data partitions for horizontal scaling                 ║
║ Replicas: Backup copies for high availability                  ║
║ Templates: Schemas for automatic index creation                ║
║ Coordinator: Routes requests and aggregates results            ║
╚════════════════════════════════════════════════════════════════╝
```

---

## 10. أمثلة عملية

### 10.1 مثال كامل: من Alert حقيقي إلى تخزينه

#### السيناريو: محاولة هجوم SSH Brute Force

**1. الحدث الأصلي (في /var/log/auth.log)**

```
Oct 19 14:30:45 web-server-01 sshd[12345]: Failed password for root from 203.0.113.45 port 54321 ssh2
Oct 19 14:30:47 web-server-01 sshd[12346]: Failed password for root from 203.0.113.45 port 54322 ssh2
Oct 19 14:30:49 web-server-01 sshd[12347]: Failed password for root from 203.0.113.45 port 54323 ssh2
Oct 19 14:30:51 web-server-01 sshd[12348]: Failed password for root from 203.0.113.45 port 54324 ssh2
Oct 19 14:30:53 web-server-01 sshd[12349]: Failed password for root from 203.0.113.45 port 54325 ssh2
```

**2. Wazuh Agent يقرأ الحدث**

```xml
<!-- في ossec.conf -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

**3. Manager يحلل ويطبق القاعدة**

```xml
<!-- Rule 5551 -->
<rule id="5551" level="10" frequency="5" timeframe="120">
  <if_matched_sid>5503</if_matched_sid>
  <description>Multiple authentication failures</description>
  <mitre>
    <id>T1110</id>
  </mitre>
  <group>authentication_failures,</group>
</rule>
```

**4. Manager يكتب Alert**

```json
{
  "timestamp": "2025-10-19T14:30:53.456+0000",
  "rule": {
    "level": 10,
    "description": "Multiple authentication failures",
    "id": "5551",
    "mitre": {
      "id": ["T1110"],
      "tactic": ["Credential Access"],
      "technique": ["Brute Force"]
    },
    "firedtimes": 5,
    "mail": true,
    "groups": ["authentication_failures", "syslog", "sshd"]
  },
  "agent": {
    "id": "001",
    "name": "web-server-01",
    "ip": "192.168.1.100",
    "version": "Wazuh v4.8.0"
  },
  "manager": {
    "name": "wazuh-manager"
  },
  "id": "1697724653.45678",
  "decoder": {
    "parent": "sshd",
    "name": "sshd"
  },
  "data": {
    "srcip": "203.0.113.45",
    "srcport": "54325",
    "dstuser": "root"
  },
  "location": "/var/log/auth.log",
  "full_log": "Failed password for root from 203.0.113.45 port 54325 ssh2"
}
```

**5. Filebeat يقرأ ويرسل**

```
[2025-10-19T14:30:54] INFO Harvester started for file: /var/ossec/logs/alerts/alerts.json
[2025-10-19T14:30:54] INFO Read 1 lines from /var/ossec/logs/alerts/alerts.json
[2025-10-19T14:30:54] INFO Added event to batch
[2025-10-19T14:30:55] INFO Batch full (2000 events), publishing...
[2025-10-19T14:30:55] INFO POST https://indexer:9200/_bulk (428 KB)
```

**6. Indexer يفهرس**

```json
POST /_bulk
Content-Type: application/x-ndjson

{"index":{"_index":"wazuh-alerts-4.x-2025.10.19","_id":"1697724653.45678"}}
{"@timestamp":"2025-10-19T14:30:53.456Z","rule":{"level":10,"description":"Multiple authentication failures","id":"5551","mitre":{"id":["T1110"],"tactic":["Credential Access"],"technique":["Brute Force"]}},"agent":{"id":"001","name":"web-server-01","ip":"192.168.1.100"},"data":{"srcip":"203.0.113.45","dstuser":"root"},"location":"/var/log/auth.log"}
```

**7. Indexer Response**

```json
{
  "took": 45,
  "errors": false,
  "items": [
    {
      "index": {
        "_index": "wazuh-alerts-4.x-2025.10.19",
        "_id": "1697724653.45678",
        "_version": 1,
        "result": "created",
        "_shards": {
          "total": 2,
          "successful": 2,
          "failed": 0
        },
        "_seq_no": 12345,
        "_primary_term": 1,
        "status": 201
      }
    }
  ]
}
```

**8. الآن التنبيه مخزن ويمكن البحث عنه**

### 10.2 مثال كامل: بحث واستعلام

#### السيناريو: البحث عن جميع هجمات SSH آخر 24 ساعة

**1. طلب من Dashboard**

```http
POST /wazuh-alerts-*/_search
Content-Type: application/json
Authorization: Basic YWRtaW46cGFzc3dvcmQ=

{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "rule.groups": "sshd"
          }
        },
        {
          "range": {
            "rule.level": {
              "gte": 7
            }
          }
        },
        {
          "range": {
            "@timestamp": {
              "gte": "now-24h"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "attacks_by_source": {
      "terms": {
        "field": "data.srcip",
        "size": 10
      },
      "aggs": {
        "targeted_users": {
          "terms": {
            "field": "data.dstuser",
            "size": 5
          }
        }
      }
    },
    "timeline": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      }
    }
  },
  "size": 100,
  "sort": [
    {
      "@timestamp": "desc"
    }
  ],
  "_source": [
    "rule.id",
    "rule.description",
    "rule.level",
    "@timestamp",
    "agent.name",
    "data.srcip",
    "data.dstuser"
  ]
}
```

**2. Indexer Response**

```json
{
  "took": 127,
  "timed_out": false,
  "_shards": {
    "total": 9,
    "successful": 9,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 342,
      "relation": "eq"
    },
    "max_score": null,
    "hits": [
      {
        "_index": "wazuh-alerts-4.x-2025.10.19",
        "_id": "1697724653.45678",
        "_score": null,
        "_source": {
          "@timestamp": "2025-10-19T14:30:53.456Z",
          "rule": {
            "id": "5551",
            "description": "Multiple authentication failures",
            "level": 10
          },
          "agent": {
            "name": "web-server-01"
          },
          "data": {
            "srcip": "203.0.113.45",
            "dstuser": "root"
          }
        },
        "sort": [1697724653456]
      },
      {
        "_index": "wazuh-alerts-4.x-2025.10.19",
        "_id": "1697720123.12345",
        "_score": null,
        "_source": {
          "@timestamp": "2025-10-19T13:15:23.123Z",
          "rule": {
            "id": "5551",
            "description": "Multiple authentication failures",
            "level": 10
          },
          "agent": {
            "name": "db-server-02"
          },
          "data": {
            "srcip": "198.51.100.87",
            "dstuser": "admin"
          }
        },
        "sort": [1697720123123]
      }
    ]
  },
  "aggregations": {
    "attacks_by_source": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 12,
      "buckets": [
        {
          "key": "203.0.113.45",
          "doc_count": 45,
          "targeted_users": {
            "buckets": [
              {"key": "root", "doc_count": 30},
              {"key": "admin", "doc_count": 10},
              {"key": "ubuntu", "doc_count": 5}
            ]
          }
        },
        {
          "key": "198.51.100.87",
          "doc_count": 28,
          "targeted_users": {
            "buckets": [
              {"key": "admin", "doc_count": 20},
              {"key": "postgres", "doc_count": 8}
            ]
          }
        }
      ]
    },
    "timeline": {
      "buckets": [
        {"key_as_string": "2025-10-19T10:00:00.000Z", "key": 1697709600000, "doc_count": 23},
        {"key_as_string": "2025-10-19T11:00:00.000Z", "key": 1697713200000, "doc_count": 45},
        {"key_as_string": "2025-10-19T12:00:00.000Z", "key": 1697716800000, "doc_count": 67},
        {"key_as_string": "2025-10-19T13:00:00.000Z", "key": 1697720400000, "doc_count": 89},
        {"key_as_string": "2025-10-19T14:00:00.000Z", "key": 1697724000000, "doc_count": 118}
      ]
    }
  }
}
```

**3. Dashboard يعرض النتائج**

- **إجمالي الهجمات**: 342 خلال 24 ساعة
- **أخطر IP**: 203.0.113.45 (45 محاولة)
- **أكثر مستخدم مستهدف**: root (30 محاولة)
- **ذروة الهجمات**: الساعة 14:00 (118 محاولة)

---

## 11. التحديات والحلول

### 11.1 التحديات الشائعة

#### 11.1.1 أداء الفهرسة البطيء

**المشكلة**: Indexing throughput منخفض

**الأسباب**:
- عدد Shards غير مناسب
- Refresh interval قصير جدًا
- نقص الموارد (CPU/Memory/Disk)

**الحلول**:
```json
{
  "settings": {
    "index": {
      "number_of_shards": 5,
      "refresh_interval": "30s",
      "translog.durability": "async",
      "translog.sync_interval": "30s"
    }
  }
}
```

#### 11.1.2 استهلاك مساحة التخزين

**المشكلة**: Indices تستهلك مساحة كبيرة

**الحلول**:

**1. تفعيل الضغط:**
```json
{
  "settings": {
    "index.codec": "best_compression"
  }
}
```

**2. استخدام Index State Management (ISM):**
```json
{
  "policy": {
    "description": "Wazuh alerts retention policy",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "warm",
            "conditions": {
              "min_index_age": "7d"
            }
          }
        ]
      },
      {
        "name": "warm",
        "actions": [
          {
            "replica_count": {"number_of_replicas": 0}
          },
          {
            "force_merge": {"max_num_segments": 1}
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "90d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          {"delete": {}}
        ]
      }
    ]
  }
}
```

#### 11.1.3 بطء البحث

**المشكلة**: Queries تستغرق وقتًا طويلاً

**الحلول**:

**1. استخدام Index Patterns الذكية:**
```json
// بدلاً من البحث في جميع الـ indices
GET /wazuh-alerts-*/_search

// ابحث فقط في النطاق الزمني المحدد
GET /wazuh-alerts-4.x-2025.10.{19,18,17}/_search
```

**2. تحسين Queries:**
```json
// بطيء
{
  "query": {
    "wildcard": {
      "rule.description": "*brute*force*"
    }
  }
}

// سريع
{
  "query": {
    "match": {
      "rule.description": "brute force"
    }
  }
}
```

**3. استخدام Filters بدلاً من Queries عندما لا تحتاج scoring:**
```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"rule.level": 10}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  }
}
```

### 11.2 Best Practices

#### 11.2.1 تصميم Index Strategy

```
✅ DO:
- استخدم indices يومية أو أسبوعية
- سمّ indices بنمط واضح: wazuh-alerts-4.x-YYYY.MM.DD
- استخدم aliases للإشارة إلى مجموعات indices

❌ DON'T:
- لا تستخدم index واحد كبير
- لا تستخدم أسماء عشوائية للـ indices
```

#### 11.2.2 تحسين Mappings

```json
{
  "mappings": {
    "properties": {
      // ✅ استخدم الأنواع المناسبة
      "timestamp": {"type": "date"},
      "count": {"type": "integer"},
      "severity": {"type": "byte"},
      
      // ✅ عطّل indexing للحقول غير المستخدمة في البحث
      "full_log": {
        "type": "text",
        "index": false
      },
      
      // ✅ استخدم ignore_above للـ keywords الطويلة
      "url": {
        "type": "keyword",
        "ignore_above": 512
      }
    }
  }
}
```

#### 11.2.3 Cluster Sizing

**للـ Production Environment:**

| المكون | الحد الأدنى | المُوصى به |
|--------|------------|-------------|
| **Coordinator Nodes** | 2 | 3 |
| **Data Nodes** | 3 | 5+ |
| **CPU per Node** | 4 cores | 8+ cores |
| **RAM per Node** | 8 GB | 32+ GB |
| **Disk** | SSD | NVMe SSD |
| **Network** | 1 Gbps | 10 Gbps |

**JVM Heap Size:**
```bash
# في opensearch.yml أو jvm.options
# اجعل heap = 50% من RAM (لكن لا تتجاوز 32GB)
-Xms16g
-Xmx16g
```

---

## 12. Monitoring & Troubleshooting

### 12.1 مراقبة صحة Cluster

```bash
# صحة الـ Cluster
GET /_cluster/health

{
  "cluster_name": "wazuh-cluster",
  "status": "green",  # green = جيد، yellow = تحذير، red = خطأ
  "timed_out": false,
  "number_of_nodes": 5,
  "number_of_data_nodes": 3,
  "active_primary_shards": 15,
  "active_shards": 30,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0
}
```

### 12.2 إحصائيات الـ Indices

```bash
# حجم كل index
GET /_cat/indices/wazuh-alerts-*?v&h=index,docs.count,store.size&s=index:desc

index                          docs.count store.size
wazuh-alerts-4.x-2025.10.19      1234567   2.3gb
wazuh-alerts-4.x-2025.10.18      1198234   2.1gb
wazuh-alerts-4.x-2025.10.17      1156789   2.0gb
```

### 12.3 أداء Queries

```bash
# تفعيل slow query logging
PUT /wazuh-alerts-*/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.fetch.warn": "1s"
}
```

### 12.4 استكشاف الأخطاء

#### مشكلة: Unassigned Shards

```bash
# تحديد السبب
GET /_cluster/allocation/explain

# الحل المحتمل: إعادة محاولة التخصيص
POST /_cluster/reroute?retry_failed=true
```

#### مشكلة: Circuit Breaker

```bash
# زيادة حد memory circuit breaker
PUT /_cluster/settings
{
  "persistent": {
    "indices.breaker.total.limit": "80%"
  }
}
```

---

## 13. الخلاصة

### 13.1 ملخص سريع

**Wazuh Indexer** هو العمود الفقري لتخزين واسترجاع البيانات في منظومة Wazuh EDR:

1. **الاستقبال**: يستقبل التنبيهات من Manager عبر Filebeat
2. **الفهرسة**: يطبق Templates و Mappings لتنظيم البيانات
3. **التخزين**: يوزع البيانات على Shards للأداء والموثوقية
4. **البحث**: يوفر queries سريعة وقوية مع aggregations متقدمة
5. **التكامل**: يتصل مع Dashboard و API لعرض البيانات

### 13.2 المكونات الرئيسية

| المكون | الوظيفة | الأهمية |
|--------|---------|---------|
| **Index Templates** | قوالب إنشاء الـ indices | ⭐⭐⭐⭐⭐ |
| **Mappings** | تعريف schema البيانات | ⭐⭐⭐⭐⭐ |
| **Shards** | توزيع البيانات | ⭐⭐⭐⭐ |
| **Replicas** | النسخ الاحتياطي | ⭐⭐⭐⭐ |
| **Query DSL** | لغة الاستعلام | ⭐⭐⭐⭐⭐ |
| **Aggregations** | التحليل الإحصائي | ⭐⭐⭐⭐ |

### 13.3 أهمية Indexer في النظام

```
بدون Indexer:
❌ لا يمكن البحث في الأحداث السابقة
❌ لا يمكن تحليل الاتجاهات الأمنية
❌ لا يمكن إنشاء تقارير الامتثال
❌ لا يمكن التحقيق الجنائي

مع Indexer:
✅ بحث سريع في ملايين الأحداث (< 100ms)
✅ تحليل في الوقت الفعلي
✅ الاحتفاظ بالبيانات لشهور/سنوات
✅ تقارير تفصيلية وإحصائيات
✅ استعلامات معقدة ومرنة
```

### 13.4 الخطوات التالية (للتعلم المتقدم)

1. **تعلم Query DSL بعمق**: 
   - https://opensearch.org/docs/latest/query-dsl/

2. **إتقان Aggregations**:
   - https://opensearch.org/docs/latest/aggregations/

3. **تحسين الأداء**:
   - https://opensearch.org/docs/latest/tuning-your-cluster/

4. **Index State Management**:
   - https://opensearch.org/docs/latest/im-plugin/ism/index/

5. **Security Configuration**:
   - https://opensearch.org/docs/latest/security/

---

## 14. المراجع والمصادر

### 14.1 التوثيق الرسمي

**Wazuh:**
- البنية المعمارية: https://documentation.wazuh.com/current/getting-started/architecture.html
- Indexer Setup: https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/index.html
- Index Management: https://documentation.wazuh.com/current/user-manual/elasticsearch/elastic-tuning.html

**OpenSearch:**
- Getting Started: https://opensearch.org/docs/latest/getting-started/
- Index APIs: https://opensearch.org/docs/latest/api-reference/index-apis/
- Search APIs: https://opensearch.org/docs/latest/api-reference/search/
- Mappings: https://opensearch.org/docs/latest/field-types/mappings/
- Query DSL: https://opensearch.org/docs/latest/query-dsl/

### 14.2 الكود المصدري

- Wazuh Indexer: https://github.com/wazuh/wazuh-indexer
- Wazuh Main: https://github.com/wazuh/wazuh
- OpenSearch: https://github.com/opensearch-project/OpenSearch

### 14.3 أدوات مفيدة

- Wazuh Documentation: https://documentation.wazuh.com/
- OpenSearch Playground: https://playground.opensearch.org/
- Elastic Common Schema: https://www.elastic.co/guide/en/ecs/current/index.html

---

## 15. Appendix: مصطلحات تقنية

| المصطلح | الشرح |
|---------|--------|
| **Index** | مجموعة من documents مخزنة معًا |
| **Document** | وحدة البيانات الأساسية (مثل JSON object) |
| **Shard** | جزء من index موزع على node |
| **Replica** | نسخة احتياطية من shard |
| **Node** | خادم واحد في الـ cluster |
| **Cluster** | مجموعة من nodes تعمل معًا |
| **Mapping** | تعريف schema للـ documents |
| **Template** | قالب لإنشاء indices جديدة |
| **Analyzer** | محلل نصوص (tokenization) |
| **Aggregation** | عملية تجميع وإحصاء |
| **Query DSL** | لغة استعلام JSON |
| **Inverted Index** | هيكل بيانات Lucene للبحث السريع |
| **Refresh** | جعل documents جديدة قابلة للبحث |
| **Flush** | كتابة البيانات من memory إلى disk |

---

## 16. شكر وتقدير

هذا التقرير تم إعداده كجزء من مشروع فهم وتوثيق **Wazuh EDR**، بهدف تحويل الكود المعقد إلى معرفة واضحة ومبسطة.

**الفريق المسؤول عن المشروع:**
- Server/Manager: عُمر
- Agent: صالح + زياد
- Dashboard: عبدالحميد
- Indexer: [تم إخفاء الاسم]
- الورقة العلمية: عبدالخالق

**تاريخ الإعداد**: أكتوبر 2025
**الإصدار**: 1.0

---

## 17. ملاحظات نهائية

### لمن يقرأ هذا التقرير:

هذا التقرير يركز على **Wazuh Indexer** فقط، وهو جزء من منظومة أكبر. لفهم الصورة الكاملة، يُنصح بقراءة التقارير الأخرى عن:
- **Manager**: كيف يتم تحليل الأحداث وتوليد التنبيهات
- **Agent**: كيف يتم جمع الأحداث من الأجهزة
- **Dashboard**: كيف يتم عرض البيانات للمستخدم

### نقاط القوة في هذا التقرير:

✅ شرح مفصل لـ Index Templates و Mappings  
✅ أمثلة عملية حقيقية من البداية للنهاية  
✅ مخططات توضيحية شاملة  
✅ تغطية كاملة لعملية الفهرسة والبحث  
✅ Best practices ونصائح تحسين الأداء  
✅ روابط مباشرة للمصادر الرسمية  

### للاستفسارات أو التحسينات:

هذا التقرير قابل للتحديث والتطوير. إذا وجدت أي أخطاء أو لديك اقتراحات، يُرجى:
1. مراجعة التوثيق الرسمي للتأكد
2. التواصل مع الفريق
3. المساهمة في تحسين التقرير

---

**EOF - نهاية التقرير**

```
╔══════════════════════════════════════════════════════════╗
║    تم بحمد الله - Wazuh Indexer Report Completed       ║
║                                                          ║
║    "المعرفة قوة، والتوثيق الجيد يجعلها متاحة للجميع"   ║
╚══════════════════════════════════════════════════════════╝
```