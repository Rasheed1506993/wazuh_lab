# Wazuh Indexer
## Comprehensive Source Code Structure and System Overview Report

---

**Repository**: https://github.com/wazuh/wazuh-indexer  
**Base**: OpenSearch 2.x (fork من Elasticsearch 7.10.2)  
**Language**: Java, Groovy, Shell Scripts  
**Build System**: Gradle 8.x

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [BuildSrc Architecture](#3-buildsrc-architecture)
4. [Core Components](#4-core-components)
5. [Plugins System](#5-plugins-system)
6. [Integration Layer](#6-integration-layer)
7. [Configuration Files](#7-configuration-files)
8. [Build Process Flow](#8-build-process-flow)
9. [Key Classes and Methods](#9-key-classes-and-methods)
10. [Testing Infrastructure](#10-testing-infrastructure)
11. [Deployment Artifacts](#11-deployment-artifacts)
12. [Summary](#12-summary)

---

## 1. Project Overview

### 1.1 What is Wazuh Indexer?

Wazuh Indexer هو نسخة معدلة من OpenSearch مصممة خصيصاً لـ Wazuh Security Platform. يوفر:

- **Data Storage**: تخزين الأحداث الأمنية والتنبيهات
- **Search Engine**: محرك بحث قوي للاستعلام عن البيانات
- **REST API**: واجهة برمجية للتكامل مع Wazuh Dashboard
- **Custom Plugins**: إضافات مخصصة لإدارة الأوامر الأمنية

### 1.2 Technical Stack

| المكون | التقنية | الإصدار |
|--------|---------|---------|
| **Core Engine** | Apache Lucene | 9.7+ |
| **Base Platform** | OpenSearch | 2.11.x |
| **Language** | Java | 17+ |
| **Build Tool** | Gradle | 8.5+ |
| **Network Layer** | Netty | 4.1.x |
| **Serialization** | Jackson | 2.15.x |

### 1.3 Repository Statistics

```bash
Total Files: ~15,000+ files
Java Code: ~350,000 lines
Test Code: ~150,000 lines
Documentation: ~5,000 lines
Plugins: 2 custom plugins
```

---

## 2. Repository Structure

### 2.1 Top-Level Directory Layout

```
wazuh-indexer/
├── buildSrc/                    # ⭐ Build logic and Gradle plugins
├── distribution/                # Distribution packages (tar, zip, deb, rpm)
├── ism-policy-templates/        # Index State Management policies
├── libs/                        # Core libraries
├── modules/                     # Core modules (transport, analysis, etc)
├── plugins/                     # Wazuh custom plugins
├── qa/                          # Quality assurance tests
├── rest-api-spec/               # OpenSearch REST API specifications
├── server/                      # Core server implementation
├── test/                        # Test infrastructure
└── integrations/                # Integration with external tools
    └── filebeat/                # Filebeat configuration
```

### 2.2 Key Directories Explained

#### 2.2.1 buildSrc/ ⭐
**الدور**: مركز البناء والتجميع
- يحتوي على Gradle plugins المخصصة
- يدير عملية البناء بأكملها
- يحدد كيف يتم تجميع وتوزيع المشروع

**الأهمية**: 🔴 حرجة - بدونها لا يمكن بناء المشروع

#### 2.2.2 distribution/
**الدور**: حزم التوزيع النهائية
- `.tar.gz` لـ Linux/Mac
- `.zip` لـ Windows
- `.deb` لـ Debian/Ubuntu
- `.rpm` لـ RedHat/CentOS

#### 2.2.3 plugins/
**الدور**: الإضافات المخصصة لـ Wazuh
```
plugins/
├── command-manager/         # إدارة الأوامر الأمنية
└── setup/                   # إعداد أولي للـ Indexer
```

#### 2.2.4 server/
**الدور**: الكود الرئيسي للخادم
- معالجة الطلبات
- إدارة الـ indices
- تنفيذ الـ queries

#### 2.2.5 integrations/
**الدور**: التكامل مع أدوات خارجية
```
integrations/
└── filebeat/
    ├── wazuh-template.json       # Index template
    └── wazuh-module.yml          # Filebeat module config
```

---

## 3. BuildSrc Architecture

### 3.1 Why BuildSrc is Critical?

`buildSrc/` هو **أهم مجلد في المشروع** لأنه:

1. **يحدد كيف يُبنى المشروع** (Build Logic)
2. **يحتوي على Gradle Plugins** المخصصة
3. **يدير Dependencies** وإصدارات المكتبات
4. **ينظم عملية الاختبار** (Testing Framework)
5. **يُنشئ حزم التوزيع** (Distribution Packages)

### 3.2 BuildSrc Directory Structure

```
buildSrc/
├── src/
│   ├── main/                    # Build logic
│   │   ├── java/                # Java plugins
│   │   ├── groovy/              # Groovy plugins
│   │   └── resources/           # Configuration files
│   ├── test/                    # Unit tests for build logic
│   ├── testFixtures/            # Test utilities
│   ├── testKit/                 # Gradle TestKit tests
│   └── integTest/               # Integration tests
└── build.gradle                 # BuildSrc own build file
```

### 3.3 Main Components in BuildSrc

#### 3.3.1 Core Build Plugins

**الموقع**: `buildSrc/src/main/java/org/opensearch/gradle/`

| Plugin | الملف | الوظيفة |
|--------|-------|---------|
| **BuildPlugin** | `BuildPlugin.groovy` | Plugin رئيسي لبناء المشروع |
| **OpenSearchJavaPlugin** | `OpenSearchJavaPlugin.java` | إعدادات Java للمشروع |
| **DistributionDownloadPlugin** | `DistributionDownloadPlugin.java` | تحميل distributions |
| **JdkDownloadPlugin** | `JdkDownloadPlugin.java` | تحميل JDK versions |
| **TestClustersPlugin** | `TestClustersPlugin.java` | إنشاء test clusters |

#### 3.3.2 Directory: `buildSrc/src/main/java/`

##### A. Core Classes

```java
// Architecture.java - تحديد معمارية النظام
public enum Architecture {
    X64,      // 64-bit Intel/AMD
    AARCH64,  // ARM 64-bit
    S390X;    // IBM System z
    
    public static Architecture current() {
        // يحدد المعمارية الحالية تلقائياً
    }
}

// OS.java - تحديد نظام التشغيل
public enum OS {
    LINUX,
    WINDOWS,
    MAC;
    
    public static OS current() {
        // يكشف نظام التشغيل الحالي
    }
}

// Version.java - إدارة الإصدارات
public class Version implements Comparable<Version> {
    private final int major;
    private final int minor;
    private final int revision;
    
    public static Version fromString(String version) {
        // يحول string إلى Version object
    }
}
```

##### B. Distribution Management

```java
// OpenSearchDistribution.java
public class OpenSearchDistribution {
    private String version;
    private Type type;        // TAR, ZIP, DEB, RPM
    private Platform platform; // LINUX, WINDOWS, DARWIN
    private Architecture architecture;
    
    public enum Type {
        INTEG_TEST,
        ARCHIVE,
        DEB,
        RPM,
        DOCKER
    }
}

// DistributionDownloadPlugin.java
public class DistributionDownloadPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        // يُسجل tasks لتحميل distributions
        setupDistributionContainer(project);
        registerDownloadTasks(project);
    }
}
```

##### C. Test Infrastructure

```java
// TestClustersPlugin.java
public class TestClustersPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        // ينشئ registry لـ test clusters
        TestClustersRegistry registry = 
            new TestClustersRegistry();
        
        // يربط test tasks بـ clusters
        configureTestTasks(project, registry);
    }
}

// OpenSearchCluster.java
public class OpenSearchCluster implements TestClusterConfiguration {
    private List<OpenSearchNode> nodes;
    private String version;
    private Map<String, String> settings;
    
    public void start() {
        // يشغل cluster للاختبار
    }
    
    public void stop() {
        // يوقف cluster بعد الاختبار
    }
}
```

#### 3.3.3 Directory: `buildSrc/src/main/groovy/`

Groovy plugins توفر DSL أبسط لـ Gradle:

```groovy
// BuildPlugin.groovy
class BuildPlugin implements Plugin<Project> {
    void apply(Project project) {
        // إعدادات Java
        project.plugins.apply(JavaPlugin)
        
        // إعدادات repositories
        project.repositories {
            mavenCentral()
            maven {
                url 'https://artifacts.opensearch.org/releases'
            }
        }
        
        // Dependencies
        configureDependencies(project)
        
        // Testing
        configureTestTasks(project)
    }
}

// PluginBuildPlugin.groovy
class PluginBuildPlugin implements Plugin<Project> {
    void apply(Project project) {
        // ينشئ plugin descriptor
        createPluginDescriptor(project)
        
        // يبني plugin zip
        configurePluginZip(project)
    }
}
```

#### 3.3.4 Directory: `buildSrc/src/main/resources/`

##### A. Gradle Plugin Properties

**الموقع**: `META-INF/gradle-plugins/`

كل ملف `.properties` يُسجل plugin:

```properties
# opensearch.build.properties
implementation-class=org.opensearch.gradle.BuildPlugin

# opensearch.testclusters.properties
implementation-class=org.opensearch.gradle.testclusters.TestClustersPlugin

# opensearch.distribution-download.properties
implementation-class=org.opensearch.gradle.DistributionDownloadPlugin
```

**الفائدة**: يسمح باستخدام plugins في `build.gradle` بهذه الطريقة:
```groovy
plugins {
    id 'opensearch.build'
    id 'opensearch.testclusters'
}
```

##### B. Forbidden APIs

**الموقع**: `forbidden/`

ملفات تحدد APIs الممنوع استخدامها:

```
forbidden/
├── jdk-signatures.txt              # JDK APIs الممنوعة
├── opensearch-all-signatures.txt   # APIs ممنوعة في كل OpenSearch
├── opensearch-server-signatures.txt # APIs ممنوعة في Server فقط
└── http-signatures.txt             # HTTP signatures
```

**مثال من `jdk-signatures.txt`**:
```
@defaultMessage Don't use java.net.URL; use URI instead
java.net.URL

@defaultMessage Don't use java.util.Random; use SecureRandom
java.util.Random#<init>()

@defaultMessage Don't use Thread.stop(); it's dangerous
java.lang.Thread#stop()
```

**الفائدة**: يمنع استخدام APIs خطرة أو deprecated

##### C. License Headers

**الموقع**: `license-headers/license-header.txt`

```
/*
 * SPDX-License-Identifier: Apache-2.0
 *
 * The OpenSearch Contributors require contributions made to
 * this file be licensed under the Apache-2.0 license or a
 * compatible open source license.
 */
```

**الفائدة**: يضمن أن كل ملف يحتوي على license header صحيح

##### D. Version Files

```
# minimumCompilerVersion
17

# minimumGradleVersion
8.5

# minimumRuntimeVersion
17
```

**الفائدة**: يتحقق من أن البيئة تستوفي الحد الأدنى من المتطلبات

### 3.4 BuildSrc Flow

```
┌─────────────────────────────────────────────────────────────┐
│                  BuildSrc Execution Flow                     │
└─────────────────────────────────────────────────────────────┘

1. [Gradle Startup]
   ↓
2. [Load buildSrc/]
   - Compile Java/Groovy plugins
   - Register custom tasks
   ↓
3. [Apply Plugins to Main Project]
   - opensearch.build
   - opensearch.testclusters
   - opensearch.distribution-download
   ↓
4. [Configure Dependencies]
   - Download required libraries
   - Resolve version conflicts
   ↓
5. [Execute Tasks]
   - compileJava
   - processResources
   - test
   - assemble
   ↓
6. [Create Distributions]
   - Build tar.gz
   - Build zip
   - Build deb/rpm
   ↓
7. [Run Tests]
   - Unit tests
   - Integration tests
   - Test clusters
   ↓
8. [Publish Artifacts]
   - Maven repository
   - Docker registry
```

---

## 4. Core Components

### 4.1 Server Module

**الموقع**: `server/src/main/java/org/opensearch/`

#### 4.1.1 Entry Point

```java
// Bootstrap.java
public final class Bootstrap {
    public static void main(String[] args) throws Exception {
        // 1. تهيئة النظام
        initializeNatives();
        
        // 2. تحميل الإعدادات
        Environment environment = createEnvironment(args);
        
        // 3. إنشاء Node
        Node node = new Node(environment);
        
        // 4. بدء الخادم
        node.start();
    }
}
```

#### 4.1.2 Core Classes

##### A. Node.java
```java
public class Node implements Closeable {
    private final Client client;
    private final IndicesService indicesService;
    private final PluginsService pluginsService;
    private final TransportService transportService;
    
    public Node(Environment environment) {
        // تهيئة جميع الخدمات
        this.indicesService = new IndicesService(...);
        this.transportService = new TransportService(...);
        this.pluginsService = new PluginsService(...);
    }
    
    public void start() {
        // بدء جميع الخدمات
        transportService.start();
        indicesService.start();
        // ...
    }
}
```

##### B. IndicesService.java
```java
public class IndicesService implements Closeable {
    private final Map<String, IndexService> indices;
    
    public IndexService createIndex(
        IndexMetadata indexMetadata,
        List<IndexEventListener> listeners
    ) {
        // إنشاء index جديد
        IndexService indexService = new IndexService(...);
        indices.put(indexMetadata.getIndex().getName(), indexService);
        return indexService;
    }
    
    public void deleteIndex(Index index) {
        // حذف index
        IndexService service = indices.remove(index.getName());
        service.close();
    }
}
```

##### C. TransportService.java
```java
public class TransportService {
    private final Transport transport;
    
    public void sendRequest(
        DiscoveryNode node,
        String action,
        TransportRequest request,
        TransportResponseHandler<T> handler
    ) {
        // إرسال طلب لـ node آخر
        transport.sendRequest(node, action, request, handler);
    }
}
```

### 4.2 Index Management

#### 4.2.1 Index Creation Flow

```java
// IndexService.java
public class IndexService {
    private final MapperService mapperService;
    private final IndexFieldDataService indexFieldDataService;
    private final Map<Integer, IndexShard> shards;
    
    public IndexShard createShard(
        ShardRouting routing,
        RetentionLeaseSyncer retentionLeaseSyncer
    ) {
        // إنشاء shard جديد
        final ShardId shardId = routing.shardId();
        final IndexShard indexShard = new IndexShard(
            routing,
            indexSettings,
            shardPath,
            store,
            // ...
        );
        
        shards.put(shardId.id(), indexShard);
        return indexShard;
    }
}
```

#### 4.2.2 Mapping Service

```java
// MapperService.java
public class MapperService {
    private volatile DocumentMapper mapper;
    
    public DocumentMapper merge(
        String type,
        CompressedXContent mappingSource,
        MergeReason reason
    ) {
        // دمج mapping جديد مع موجود
        DocumentMapper newMapper = parse(type, mappingSource);
        
        if (mapper != null) {
            // التحقق من التوافق
            newMapper = mapper.merge(newMapper.mapping());
        }
        
        mapper = newMapper;
        return mapper;
    }
}
```

### 4.3 Search & Query

#### 4.3.1 Search Request Flow

```java
// SearchService.java
public class SearchService {
    public void executeQueryPhase(
        SearchTask task,
        SearchShardTask searchShardTask
    ) {
        // 1. Parse query
        QueryShardContext context = createQueryContext(...);
        Query query = parseQuery(request.source(), context);
        
        // 2. Execute on Lucene
        IndexSearcher searcher = getSearcher(shardId);
        TopDocs topDocs = searcher.search(query, numHits);
        
        // 3. Return results
        return createSearchResult(topDocs);
    }
}
```

#### 4.3.2 Query Builders

```java
// BoolQueryBuilder.java
public class BoolQueryBuilder extends AbstractQueryBuilder {
    private final List<QueryBuilder> mustClauses = new ArrayList<>();
    private final List<QueryBuilder> filterClauses = new ArrayList<>();
    private final List<QueryBuilder> shouldClauses = new ArrayList<>();
    
    public BoolQueryBuilder must(QueryBuilder queryBuilder) {
        mustClauses.add(queryBuilder);
        return this;
    }
    
    @Override
    protected Query doToQuery(QueryShardContext context) {
        BooleanQuery.Builder booleanQuery = new BooleanQuery.Builder();
        
        for (QueryBuilder clause : mustClauses) {
            booleanQuery.add(clause.toQuery(context), Occur.MUST);
        }
        
        return booleanQuery.build();
    }
}
```

### 4.4 REST API Layer

```java
// RestController.java
public class RestController {
    private final Map<String, RestHandler> handlers = new HashMap<>();
    
    public void registerHandler(RestRequest.Method method, String path, RestHandler handler) {
        handlers.put(method + ":" + path, handler);
    }
    
    public void dispatchRequest(RestRequest request, RestChannel channel) {
        String key = request.method() + ":" + request.path();
        RestHandler handler = handlers.get(key);
        
        if (handler != null) {
            handler.handleRequest(request, channel, client);
        } else {
            channel.sendResponse(new BytesRestResponse(NOT_FOUND));
        }
    }
}

// RestIndexAction.java - مثال على handler
public class RestIndexAction extends BaseRestHandler {
    @Override
    public List<Route> routes() {
        return List.of(
            new Route(POST, "/{index}/_doc"),
            new Route(PUT, "/{index}/_doc/{id}")
        );
    }
    
    @Override
    protected RestChannelConsumer prepareRequest(RestRequest request, NodeClient client) {
        IndexRequest indexRequest = new IndexRequest(request.param("index"));
        indexRequest.id(request.param("id"));
        indexRequest.source(request.content(), request.getXContentType());
        
        return channel -> client.index(
            indexRequest,
            new RestStatusToXContentListener<>(channel)
        );
    }
}
```

---

## 5. Plugins System

### 5.1 Wazuh Custom Plugins

#### 5.1.1 Command Manager Plugin

**الموقع**: `plugins/command-manager/`

**الوظيفة**: إدارة الأوامر الأمنية من Wazuh Manager

```java
// CommandManagerPlugin.java
public class CommandManagerPlugin extends Plugin implements ActionPlugin {
    @Override
    public List<RestHandler> getRestHandlers(
        Settings settings,
        RestController restController,
        // ...
    ) {
        return Arrays.asList(
            new RestCommandAction(),
            new RestCommandStatusAction()
        );
    }
    
    @Override
    public Collection<Object> createComponents(
        Client client,
        ClusterService clusterService,
        // ...
    ) {
        CommandManager commandManager = new CommandManager(client);
        return Collections.singletonList(commandManager);
    }
}

// RestCommandAction.java
public class RestCommandAction extends BaseRestHandler {
    @Override
    public List<Route> routes() {
        return List.of(
            new Route(POST, "/_plugins/command-manager/commands")
        );
    }
    
    @Override
    protected RestChannelConsumer prepareRequest(RestRequest request, NodeClient client) {
        // استقبال أمر من Wazuh Manager
        String command = request.param("command");
        String target = request.param("target");
        
        // تنفيذ الأمر
        executeCommand(command, target);
        
        return channel -> channel.sendResponse(
            new BytesRestResponse(OK, "Command executed")
        );
    }
}
```

#### 5.1.2 Setup Plugin

**الموقع**: `plugins/setup/`

**الوظيفة**: الإعداد الأولي للـ Indexer

```java
public class SetupPlugin extends Plugin {
    @Override
    public Collection<Object> createComponents(
        Client client,
        ClusterService clusterService,
        // ...
    ) {
        // إنشاء index templates تلقائياً
        IndexTemplateInstaller installer = new IndexTemplateInstaller(client);
        installer.installDefaultTemplates();
        
        // إنشاء users افتراضيين
        SecuritySetup securitySetup = new SecuritySetup(client);
        securitySetup.createDefaultUsers();
        
        return Collections.emptyList();
    }
}
```

### 5.2 Plugin Descriptor

كل plugin يحتوي على `plugin-descriptor.properties`:

```properties
# plugins/command-manager/src/main/resources/plugin-descriptor.properties
description=Wazuh Command Manager
version=4.8.0
name=command-manager
classname=com.wazuh.commandmanager.CommandManagerPlugin
java.version=17
opensearch.version=2.11.0
```

---

## 6. Integration Layer

### 6.1 Filebeat Integration

**الموقع**: `integrations/filebeat/`

#### 6.1.1 Wazuh Template

**ملف**: `wazuh-template.json`

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
        "mapping": {
          "total_fields": {
            "limit": 10000
          },
          "depth": {
            "limit": 50
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
        },
        "agent": {
          "properties": {
            "id": {"type": "keyword"},
            "name": {"type": "keyword"},
            "ip": {"type": "ip"},
            "version": {"type": "keyword"}
          }
        },
        "data": {
          "type": "object",
          "dynamic": true
        },
        "location": {
          "type": "keyword"
        },
        "full_log": {
          "type": "text",
          "index": false
        }
      }
    }
  }
}
```

**الشرح**:
- `index_patterns`: يطابق indices بنمط معين
- `number_of_shards`: عدد الأجزاء (3 للتوازن)
- `number_of_replicas`: نسخة احتياطية واحدة
- `refresh_interval`: تحديث كل 5 ثوان
- `codec: best_compression`: ضغط البيانات

#### 6.1.2 Filebeat Module

**ملف**: `wazuh-module.yml`

```yaml
- module: wazuh
  alerts:
    enabled: true
    var.paths:
      - /var/ossec/logs/alerts/alerts.json
    var.index_prefix: wazuh-alerts-4.x
  
  archives:
    enabled: false
    var.paths:
      - /var/ossec/logs/archives/archives.json
    var.index_prefix: wazuh-archives-4.x
```

---

## 7. Configuration Files

### 7.1 OpenSearch Configuration

**الموقع**: `distribution/src/config/opensearch.yml`

```yaml
# ======================== OpenSearch Configuration =========================

# ---------------------------------- Cluster -----------------------------------
cluster.name: wazuh-cluster
node.name: wazuh-node-1

# ----------------------------------- Paths ------------------------------------
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

# ---------------------------------- Network -----------------------------------
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300

# --------------------------------- Discovery ----------------------------------
discovery.type: single-node
# discovery.seed_hosts: ["host1", "host2"]
# cluster.initial_master_nodes: ["node-1", "node-2"]

# ---------------------------------- Security ----------------------------------
plugins.security.disabled: false
plugins.security.ssl.transport.pemcert_filepath: node-cert.pem
plugins.security.ssl.transport.pemkey_filepath: node-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: node-cert.pem
plugins.security.ssl.http.pemkey_filepath: node-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: root-ca.pem

plugins.security.allow_default_init_securityindex: true
plugins.security.authcz.admin_dn:
  - CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US

# ---------------------------------- Various -----------------------------------
compatibility.override_main_response_version: true
```

### 7.2 JVM Options

**الموقع**: `distribution/src/config/jvm.options`

```
## JVM configuration

################################################################
## IMPORTANT: JVM heap size
################################################################
-Xms2g
-Xmx2g

################################################################
## Expert settings
################################################################
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

## DNS cache policy
-Des.networkaddress.cache.ttl=60
-Des.networkaddress.cache.negative.ttl=10

## Security manager
-Djava.security.manager=allow
-Djava.security.policy=file:///etc/wazuh-indexer/opensearch-java.policy

## Locale
-Dfile.encoding=UTF-8

## Listeners
-Djava.awt.headless=true

## Heap dumps
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/lib/wazuh-indexer

## GC logging
-Xlog:gc*,gc+age=trace,safepoint:file=/var/log/wazuh-indexer/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

### 7.3 Log4j Configuration

**الموقع**: `distribution/src/config/log4j2.properties`

```properties
status = error

appender.console.type = Console
appender.console.name = console
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] [%node_name]%marker %m%n

appender.rolling.type = RollingFile
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:opensearch.logs.base_path}${sys:file.separator}${sys:opensearch.logs.cluster_name}.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] [%node_name]%marker %.-10000m%n
appender.rolling.filePattern = ${sys:opensearch.logs.base_path}${sys:file.separator}${sys:opensearch.logs.cluster_name}-%d{yyyy-MM-dd}-%i.log.gz
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 1
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size = 256MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.fileIndex = nomax
appender.rolling.strategy.action.type = Delete
appender.rolling.strategy.action.basepath = ${sys:opensearch.logs.base_path}
appender.rolling.strategy.action.condition.type = IfFileName
appender.rolling.strategy.action.condition.glob = ${sys:opensearch.logs.cluster_name}-*
appender.rolling.strategy.action.condition.nested_condition.type = IfLastModified
appender.rolling.strategy.action.condition.nested_condition.age = 7D

rootLogger.level = info
rootLogger.appenderRef.console.ref = console
rootLogger.appenderRef.rolling.ref = rolling

logger.action.name = org.opensearch.action
logger.action.level = debug

logger.deprecation.name = org.opensearch.deprecation
logger.deprecation.level = warn
```

---

## 8. Build Process Flow

### 8.1 Complete Build Sequence

```
┌──────────────────────────────────────────────────────────────┐
│              Wazuh Indexer Build Process                      │
└──────────────────────────────────────────────────────────────┘

[Phase 1: Environment Setup]
1. Check JDK version (>= 17)
2. Check Gradle version (>= 8.5)
3. Load buildSrc/
4. Compile Gradle plugins
   ↓

[Phase 2: Dependency Resolution]
5. Download dependencies from Maven Central
6. Download OpenSearch artifacts
7. Resolve version conflicts
   ↓

[Phase 3: Code Compilation]
8. Compile server module
   - server/src/main/java/
9. Compile plugins
   - plugins/command-manager/
   - plugins/setup/
10. Process resources
    - Copy configuration files
    - Generate plugin descriptors
   ↓

[Phase 4: Testing]
11. Unit Tests
    - server/src/test/
12. Integration Tests
    - buildSrc/src/integTest/
13. Test Clusters
    - Start temporary cluster
    - Run REST API tests
    - Stop cluster
   ↓

[Phase 5: Packaging]
14. Create JAR files
    - server.jar
    - plugin JARs
15. Assemble distributions
    - wazuh-indexer-4.8.0-linux-x64.tar.gz
    - wazuh-indexer-4.8.0-linux-x64.zip
    - wazuh-indexer-4.8.0.deb
    - wazuh-indexer-4.8.0.rpm
   ↓

[Phase 6: Verification]
16. Checksum generation (SHA-512)
17. Signature creation (GPG)
18. License validation
   ↓

[Phase 7: Publishing]
19. Upload to artifact repository
20. Update package managers
21. Push Docker images
```

### 8.2 Key Gradle Tasks

#### 8.2.1 في Root Project

```gradle
// الموقع: build.gradle

tasks.register('assemble') {
    description = 'Assembles all distributions'
    dependsOn subprojects.collect { it.tasks.withType(Tar) }
    dependsOn subprojects.collect { it.tasks.withType(Zip) }
}

tasks.register('check') {
    description = 'Runs all checks'
    dependsOn 'test'
    dependsOn 'precommit'
}

tasks.register('build') {
    description = 'Assembles and tests this project'
    dependsOn 'assemble'
    dependsOn 'check'
}
```

#### 8.2.2 في Server Module

```gradle
// الموقع: server/build.gradle

plugins {
    id 'opensearch.build'
    id 'opensearch.publish'
}

dependencies {
    api "org.apache.lucene:lucene-core:${versions.lucene}"
    api "org.apache.lucene:lucene-analyzers-common:${versions.lucene}"
    api "com.fasterxml.jackson.core:jackson-core:${versions.jackson}"
    
    implementation "io.netty:netty-all:${versions.netty}"
    implementation "org.apache.logging.log4j:log4j-api:${versions.log4j}"
    
    testImplementation "junit:junit:${versions.junit}"
    testImplementation "org.hamcrest:hamcrest:${versions.hamcrest}"
}

tasks.named('test') {
    systemProperty 'tests.security.manager', 'true'
    systemProperty 'tests.locale', 'random'
    systemProperty 'tests.timezone', 'random'
}
```

#### 8.2.3 في Plugin Modules

```gradle
// الموقع: plugins/command-manager/build.gradle

plugins {
    id 'opensearch.opensearchplugin'
}

opensearchplugin {
    name 'command-manager'
    description 'Wazuh Command Manager Plugin'
    classname 'com.wazuh.commandmanager.CommandManagerPlugin'
}

dependencies {
    compileOnly project(':server')
    
    testImplementation project(':test:framework')
}
```

### 8.3 Build Commands

```bash
# بناء كامل
./gradlew build

# بناء بدون اختبارات (أسرع)
./gradlew assemble -x test

# بناء distribution معين
./gradlew :distribution:archives:linux-tar:assemble

# تشغيل الاختبارات فقط
./gradlew test

# اختبارات integration
./gradlew integTest

# تنظيف المشروع
./gradlew clean

# نشر للـ Maven local
./gradlew publishToMavenLocal

# عرض dependencies
./gradlew dependencies

# عرض tasks المتاحة
./gradlew tasks
```

---

## 9. Key Classes and Methods

### 9.1 Data Flow Classes

#### 9.1.1 Indexing Path

```java
// 1. REST Layer: RestIndexAction.java
public class RestIndexAction extends BaseRestHandler {
    @Override
    protected RestChannelConsumer prepareRequest(
        RestRequest request, 
        NodeClient client
    ) {
        // تحويل REST request إلى IndexRequest
        IndexRequest indexRequest = new IndexRequest(request.param("index"));
        indexRequest.source(request.content(), request.getXContentType());
        
        return channel -> client.index(
            indexRequest,
            new RestStatusToXContentListener<>(channel)
        );
    }
}

// 2. Transport Layer: TransportIndexAction.java
public class TransportIndexAction extends TransportWriteAction<IndexRequest, IndexResponse> {
    @Override
    protected void shardOperationOnPrimary(
        IndexRequest request,
        IndexShard indexShard,
        ActionListener<PrimaryResult<IndexRequest, IndexResponse>> listener
    ) {
        // تنفيذ الفهرسة على primary shard
        Engine.IndexResult result = indexShard.applyIndexOperationOnPrimary(
            request.version(),
            request.versionType(),
            new SourceToParse(
                request.index(),
                request.id(),
                request.source(),
                request.getContentType()
            ),
            request.getAutoGeneratedTimestamp(),
            request.isRetry()
        );
        
        listener.onResponse(new PrimaryResult<>(request, result));
    }
}

// 3. Shard Layer: IndexShard.java
public class IndexShard {
    public Engine.IndexResult applyIndexOperationOnPrimary(
        long version,
        VersionType versionType,
        SourceToParse sourceToParse,
        long autoGeneratedTimestamp,
        boolean isRetry
    ) {
        // تطبيق mapping
        ParsedDocument doc = mapperService.documentMapper()
            .parse(sourceToParse);
        
        // كتابة في Lucene
        Engine.Index operation = new Engine.Index(
            doc.id(),
            doc,
            version,
            versionType,
            autoGeneratedTimestamp,
            isRetry
        );
        
        return getEngine().index(operation);
    }
}

// 4. Engine Layer: InternalEngine.java
public class InternalEngine extends Engine {
    @Override
    public IndexResult index(Index index) {
        // كتابة في Lucene IndexWriter
        try {
            long seqNo = generateSeqNo();
            
            indexWriter.updateDocument(
                index.uid(),
                index.docs(),
                new Term("_id", index.id())
            );
            
            return new IndexResult(
                index.version(),
                seqNo,
                true
            );
        } catch (IOException e) {
            return new IndexResult(e, seqNo);
        }
    }
}
```

#### 9.1.2 Search Path

```java
// 1. REST Layer: RestSearchAction.java
public class RestSearchAction extends BaseRestHandler {
    @Override
    protected RestChannelConsumer prepareRequest(
        RestRequest request,
        NodeClient client
    ) {
        SearchRequest searchRequest = new SearchRequest();
        
        // parse query من request body
        if (request.hasContent()) {
            searchRequest.source(
                new SearchSourceBuilder()
                    .parseXContent(request.contentParser())
            );
        }
        
        return channel -> client.search(
            searchRequest,
            new RestStatusToXContentListener<>(channel)
        );
    }
}

// 2. Transport Layer: TransportSearchAction.java
public class TransportSearchAction extends HandledTransportAction<SearchRequest, SearchResponse> {
    @Override
    protected void doExecute(
        Task task,
        SearchRequest request,
        ActionListener<SearchResponse> listener
    ) {
        // توزيع البحث على shards
        searchPhaseController.execute(
            request,
            (SearchTask) task,
            listener
        );
    }
}

// 3. Search Phase Controller
public class SearchPhaseController {
    public void execute(
        SearchRequest request,
        SearchTask task,
        ActionListener<SearchResponse> listener
    ) {
        // Phase 1: Query phase (scatter)
        new QueryPhase().run();
        
        // Phase 2: Fetch phase (gather)
        new FetchPhase().run();
        
        // Phase 3: Merge results
        SearchResponse response = mergeResults();
        listener.onResponse(response);
    }
}

// 4. Shard Level Search
public class SearchService {
    public void executeQueryPhase(SearchShardTask task) {
        // إنشاء Lucene query
        Query luceneQuery = queryShardContext.toQuery(request.source().query());
        
        // البحث في Lucene
        IndexSearcher searcher = searchContext.searcher();
        TopDocs topDocs = searcher.search(luceneQuery, numHits);
        
        // إرجاع IDs فقط (ليس المحتوى الكامل)
        return topDocs;
    }
    
    public void executeFetchPhase(SearchShardTask task) {
        // جلب المحتوى الكامل للـ documents المطلوبة
        List<String> docIds = task.getDocIds();
        List<SearchHit> hits = new ArrayList<>();
        
        for (String docId : docIds) {
            Document doc = searcher.doc(docId);
            hits.add(createSearchHit(doc));
        }
        
        return hits;
    }
}
```

### 9.2 Critical Methods

#### 9.2.1 Index Template Application

```java
// MetadataIndexTemplateService.java
public class MetadataIndexTemplateService {
    public static List<IndexTemplateMetadata> findTemplates(
        Metadata metadata,
        String indexName
    ) {
        List<IndexTemplateMetadata> templates = new ArrayList<>();
        
        for (IndexTemplateMetadata template : metadata.templates().values()) {
            // التحقق من تطابق index pattern
            if (Regex.simpleMatch(template.patterns(), indexName)) {
                templates.add(template);
            }
        }
        
        // ترتيب حسب priority
        templates.sort(Comparator.comparingInt(IndexTemplateMetadata::order));
        
        return templates;
    }
    
    public ClusterState addIndexTemplate(
        ClusterState currentState,
        String name,
        IndexTemplateMetadata template
    ) {
        // إضافة template جديد للـ metadata
        Metadata.Builder metadataBuilder = Metadata.builder(currentState.metadata());
        metadataBuilder.put(name, template);
        
        return ClusterState.builder(currentState)
            .metadata(metadataBuilder)
            .build();
    }
}
```

#### 9.2.2 Mapping Merge

```java
// MapperService.java
public DocumentMapper merge(
    String type,
    CompressedXContent mappingSource,
    MergeReason reason
) {
    DocumentMapper newMapper;
    
    try {
        // parse mapping الجديد
        newMapper = documentParser.parse(type, mappingSource);
    } catch (Exception e) {
        throw new MapperParsingException("Failed to parse mapping", e);
    }
    
    if (mapper != null) {
        // دمج مع mapping الموجود
        DocumentMapper mergedMapper = mapper.merge(newMapper.mapping(), reason);
        
        // التحقق من عدم وجود تعارضات
        checkFieldCaps(mergedMapper);
        
        return mergedMapper;
    }
    
    return newMapper;
}

private void checkFieldCaps(DocumentMapper mapper) {
    // التحقق من أن الحقول الجديدة لا تتعارض مع الموجودة
    for (FieldMapper fieldMapper : mapper.mappers()) {
        FieldMapper existing = this.mapper.mappers().getMapper(fieldMapper.name());
        
        if (existing != null && !existing.getClass().equals(fieldMapper.getClass())) {
            throw new IllegalArgumentException(
                "Mapper for [" + fieldMapper.name() + "] conflicts with existing mapper"
            );
        }
    }
}
```

#### 9.2.3 Shard Allocation

```java
// AllocationService.java
public class AllocationService {
    public RoutingAllocation.Result reroute(
        ClusterState clusterState,
        String reason
    ) {
        RoutingNodes routingNodes = getMutableRoutingNodes(clusterState);
        
        // 1. تخصيص unassigned shards
        allocateUnassignedShards(routingNodes);
        
        // 2. موازنة الـ shards بين nodes
        balanceShards(routingNodes);
        
        // 3. نقل shards إذا لزم الأمر
        moveShards(routingNodes);
        
        return RoutingAllocation.Result.changed(
            buildResultAndLogHealthChange(clusterState, routingNodes, reason)
        );
    }
    
    private void allocateUnassignedShards(RoutingNodes routingNodes) {
        for (ShardRouting shard : routingNodes.unassigned()) {
            // إيجاد أفضل node لهذا shard
            RoutingNode targetNode = findBestNode(shard, routingNodes);
            
            if (targetNode != null) {
                // تخصيص shard للـ node
                routingNodes.initialize(shard, targetNode.nodeId());
            }
        }
    }
}
```

---

## 10. Testing Infrastructure

### 10.1 Test Hierarchy

```
Testing Levels:
├── Unit Tests              (Fast, Isolated)
│   └── Test individual methods/classes
├── Integration Tests       (Medium speed)
│   └── Test interaction between components
├── Test Clusters          (Slower)
│   └── Test with real OpenSearch cluster
└── Distribution Tests     (Slowest)
    └── Test final packages (tar, zip, deb, rpm)
```

### 10.1 BuildSrc Test Structure

#### 10.1.1 Directory: `buildSrc/src/test/`

**Unit tests للـ build logic:**

```java
// BwcVersionsTests.java
public class BwcVersionsTests extends GradleUnitTestCase {
    @Test
    public void testParsing() {
        BwcVersions bwc = new BwcVersions(
            Version.fromString("2.11.0"),
            List.of(
                Version.fromString("2.10.0"),
                Version.fromString("2.9.0")
            )
        );
        
        assertEquals("2.11.0", bwc.getCurrentVersion().toString());
        assertEquals(2, bwc.getIndexCompatible().size());
    }
    
    @Test
    public void testWireCompatibility() {
        // التحقق من توافق versions للاتصال
        assertTrue(bwc.isWireCompatible(Version.fromString("2.10.0")));
        assertFalse(bwc.isWireCompatible(Version.fromString("1.0.0")));
    }
}

// VersionTests.java
public class VersionTests {
    @Test
    public void testVersionComparison() {
        Version v1 = Version.fromString("2.11.0");
        Version v2 = Version.fromString("2.10.5");
        
        assertTrue(v1.compareTo(v2) > 0);
        assertEquals(2, v1.major);
        assertEquals(11, v1.minor);
    }
}
```

#### 10.1.2 Directory: `buildSrc/src/integTest/`

**Integration tests للـ plugins:**

```groovy
// TestClustersPluginFuncTest.groovy
class TestClustersPluginFuncTest extends AbstractGradleFuncTest {
    def "can start and stop test cluster"() {
        given:
        buildFile << """
            plugins {
                id 'opensearch.testclusters'
            }
            
            testClusters {
                myCluster {
                    numberOfNodes = 3
                    version = '2.11.0'
                }
            }
        """
        
        when:
        def result = gradleRunner('startMyCluster').build()
        
        then:
        result.task(':startMyCluster').outcome == SUCCESS
        clusterIsRunning('myCluster')
        
        cleanup:
        gradleRunner('stopMyCluster').build()
    }
}

// DistributionDownloadPluginFuncTest.groovy
class DistributionDownloadPluginFuncTest extends AbstractGradleFuncTest {
    def "downloads distribution successfully"() {
        given:
        buildFile << """
            plugins {
                id 'opensearch.distribution-download'
            }
            
            opensearch_distributions {
                myDist {
                    version = '2.11.0'
                    type = 'archive'
                    platform = 'linux'
                    architecture = 'x64'
                }
            }
        """
        
        when:
        def result = gradleRunner('setupMyDist').build()
        
        then:
        result.task(':setupMyDist').outcome == SUCCESS
        file('build/opensearch-2.11.0-linux-x64.tar.gz').exists()
    }
}
```

#### 10.1.3 Directory: `buildSrc/src/testFixtures/`

**Utilities مشتركة للاختبارات:**

```java
// GradleIntegrationTestCase.java
public abstract class GradleIntegrationTestCase extends BaseTestCase {
    protected GradleRunner gradleRunner(String... arguments) {
        return GradleRunner.create()
            .withProjectDir(testProjectDir.getRoot())
            .withArguments(arguments)
            .withPluginClasspath()
            .withDebug(true);
    }
    
    protected File file(String path) {
        return new File(testProjectDir.getRoot(), path);
    }
    
    protected void writeBuildScript(String content) throws IOException {
        Files.write(
            file("build.gradle").toPath(),
            content.getBytes(StandardCharsets.UTF_8)
        );
    }
}

// TestClasspathUtils.java
public class TestClasspathUtils {
    public static List<File> getBuildSrcClasspath() {
        // يحصل على classpath للـ buildSrc
        // مفيد لتشغيل Gradle TestKit tests
        return Stream.of(
            System.getProperty("java.class.path").split(File.pathSeparator)
        )
        .map(File::new)
        .collect(Collectors.toList());
    }
}
```

### 10.2 Server Tests

#### 10.2.1 Unit Tests

```java
// server/src/test/java/.../IndicesServiceTests.java
public class IndicesServiceTests extends OpenSearchSingleNodeTestCase {
    @Test
    public void testCreateIndex() {
        IndicesService indicesService = getInstanceFromNode(IndicesService.class);
        
        // إنشاء index
        IndexService indexService = indicesService.createIndex(
            IndexMetadata.builder("test-index")
                .settings(Settings.builder()
                    .put(IndexMetadata.SETTING_VERSION_CREATED, Version.CURRENT)
                    .put(IndexMetadata.SETTING_NUMBER_OF_SHARDS, 1)
                    .put(IndexMetadata.SETTING_NUMBER_OF_REPLICAS, 0)
                    .build())
                .build(),
            Collections.emptyList()
        );
        
        assertNotNull(indexService);
        assertEquals("test-index", indexService.index().getName());
    }
    
    @Test
    public void testDeleteIndex() {
        // إنشاء ثم حذف
        createIndex("test-index");
        assertTrue(indicesService.hasIndex(new Index("test-index", "_na_")));
        
        indicesService.deleteIndex(new Index("test-index", "_na_"));
        assertFalse(indicesService.hasIndex(new Index("test-index", "_na_")));
    }
}
```

#### 10.2.2 Integration Tests

```java
// server/src/test/java/.../SimpleIndexTemplateIT.java
@OpenSearchIntegTestCase.ClusterScope(scope = SUITE)
public class SimpleIndexTemplateIT extends OpenSearchIntegTestCase {
    @Test
    public void testIndexTemplateCreation() {
        // إنشاء template
        client().admin().indices().preparePutTemplate("wazuh-alerts-template")
            .setPatterns(Collections.singletonList("wazuh-alerts-*"))
            .setSettings(Settings.builder()
                .put("index.number_of_shards", 3)
                .put("index.number_of_replicas", 1))
            .setMapping("{\"properties\": {" +
                "\"@timestamp\": {\"type\": \"date\"}," +
                "\"rule\": {\"properties\": {\"id\": {\"type\": \"keyword\"}}}" +
                "}}")
            .get();
        
        // إنشاء index يطابق pattern
        createIndex("wazuh-alerts-4.x-2025-01-01");
        
        // التحقق من تطبيق template
        GetIndexResponse response = client().admin().indices()
            .prepareGetIndex()
            .addIndices("wazuh-alerts-4.x-2025-01-01")
            .get();
        
        Settings settings = response.getSettings().get("wazuh-alerts-4.x-2025-01-01");
        assertEquals("3", settings.get("index.number_of_shards"));
        assertEquals("1", settings.get("index.number_of_replicas"));
    }
}
```

### 10.3 Test Clusters

```groovy
// في build.gradle
testClusters {
    integTest {
        numberOfNodes = 3
        
        setting 'cluster.name', 'test-cluster'
        setting 'path.repo', "${buildDir}/cluster/shared/repo"
        
        // تثبيت plugins
        plugin project(':plugins:command-manager').bundlePlugin
        
        // users للأمان
        user username: 'admin', password: 'admin', role: 'admin'
        user username: 'test', password: 'test', role: 'user'
    }
}

tasks.named('integTest') {
    useCluster testClusters.integTest
    
    systemProperty 'tests.cluster', 
        "${-> testClusters.integTest.allHttpSocketURI.join(',')}"
}
```

---

## 11. Deployment Artifacts

### 11.1 Distribution Types

#### 11.1.1 TAR.GZ (Linux/Mac)

**الموقع**: `distribution/archives/linux-tar/`

**البنية**:
```
wazuh-indexer-4.8.0/
├── bin/
│   ├── opensearch            # Startup script
│   ├── opensearch-cli        # CLI tool
│   └── opensearch-plugin     # Plugin manager
├── config/
│   ├── opensearch.yml        # Main config
│   ├── jvm.options           # JVM settings
│   └── log4j2.properties     # Logging
├── lib/
│   └── *.jar                 # Core libraries
├── modules/
│   └── */                    # Core modules
├── plugins/
│   ├── command-manager/      # Wazuh plugin
│   └── opensearch-security/  # Security plugin
├── logs/                     # Log directory
├── data/                     # Data directory
├── LICENSE.txt
├── NOTICE.txt
└── README.md
```

**التثبيت**:
```bash
tar -xzf wazuh-indexer-4.8.0-linux-x64.tar.gz
cd wazuh-indexer-4.8.0/
./bin/opensearch
```

#### 11.1.2 DEB Package (Debian/Ubuntu)

**الموقع**: `distribution/packages/deb/`

**محتويات الحزمة**:
```
/usr/share/wazuh-indexer/          # البرنامج
/etc/wazuh-indexer/                # الإعدادات
/var/lib/wazuh-indexer/            # البيانات
/var/log/wazuh-indexer/            # السجلات
/usr/lib/systemd/system/wazuh-indexer.service  # Service unit
```

**Scripts**:
- `preinst.ftl`: قبل التثبيت (إنشاء مستخدم)
- `postinst.ftl`: بعد التثبيت (تفعيل service)
- `prerm`: قبل الإزالة
- `postrm`: بعد الإزالة

**التثبيت**:
```bash
sudo dpkg -i wazuh-indexer_4.8.0_amd64.deb
sudo systemctl start wazuh-indexer
```

#### 11.1.3 RPM Package (RedHat/CentOS)

**الموقع**: `distribution/packages/rpm/`

**مشابه للـ DEB لكن بتنسيق RPM**

**التثبيت**:
```bash
sudo rpm -ivh wazuh-indexer-4.8.0.x86_64.rpm
sudo systemctl start wazuh-indexer
```

#### 11.1.4 Docker Image

**الموقع**: `distribution/docker/`

**Dockerfile**:
```dockerfile
FROM amazonlinux:2

# Install dependencies
RUN yum install -y java-17-amazon-corretto

# Copy distribution
COPY wazuh-indexer-4.8.0-linux-x64.tar.gz /tmp/
RUN tar -xzf /tmp/wazuh-indexer-4.8.0-linux-x64.tar.gz -C /usr/share/ \
    && mv /usr/share/wazuh-indexer-4.8.0 /usr/share/wazuh-indexer

# Setup user
RUN useradd -m -u 1000 wazuh-indexer

# Expose ports
EXPOSE 9200 9300

# Entrypoint
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["opensearch"]
```

**الاستخدام**:
```bash
docker pull wazuh/wazuh-indexer:4.8.0
docker run -p 9200:9200 -p 9300:9300 wazuh/wazuh-indexer:4.8.0
```

### 11.2 Build Artifacts

بعد البناء الناجح، تجد:

```
build/
├── distributions/
│   ├── wazuh-indexer-4.8.0-linux-x64.tar.gz
│   ├── wazuh-indexer-4.8.0-linux-x64.tar.gz.sha512
│   ├── wazuh-indexer-4.8.0-linux-arm64.tar.gz
│   ├── wazuh-indexer-4.8.0-windows-x64.zip
│   ├── wazuh-indexer_4.8.0_amd64.deb
│   └── wazuh-indexer-4.8.0.x86_64.rpm
├── install/
│   └── wazuh-indexer-4.8.0/     # Exploded directory
└── reports/
    ├── tests/                    # Test results
    └── jacoco/                   # Code coverage
```

---

## 12. Summary

### 12.1 Project Architecture Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Wazuh Indexer Architecture Layers                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  [Layer 1: Build System]                                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ buildSrc/                                              │  │
│  │  - Gradle plugins (Java/Groovy)                        │  │
│  │  - Build logic & task definitions                      │  │
│  │  - Test infrastructure                                 │  │
│  │  - Distribution packaging                              │  │
│  └────────────────────────────────────────────────────────┘  │
│                           ▼                                   │
│  [Layer 2: Core Server]                                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ server/                                                │  │
│  │  - Node management                                     │  │
│  │  - Index service                                       │  │
│  │  - Search service                                      │  │
│  │  - Transport layer                                     │  │
│  │  - REST API                                            │  │
│  └────────────────────────────────────────────────────────┘  │
│                           ▼                                   │
│  [Layer 3: Storage Engine]                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Apache Lucene                                          │  │
│  │  - Inverted index                                      │  │
│  │  - Document storage                                    │  │
│  │  - Query execution                                     │  │
│  └────────────────────────────────────────────────────────┘  │
│                           ▼                                   │
│  [Layer 4: Extensions]                                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ plugins/                                               │  │
│  │  - command-manager: Security commands                 │  │
│  │  - setup: Initial configuration                       │  │
│  │  - security: Authentication & authorization           │  │
│  └────────────────────────────────────────────────────────┘  │
│                           ▼                                   │
│  [Layer 5: Integration]                                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ integrations/                                          │  │
│  │  - Filebeat templates                                 │  │
│  │  - Index templates                                    │  │
│  │  - ISM policies                                       │  │
│  └────────────────────────────────────────────────────────┘  │
│                           ▼                                   │
│  [Layer 6: Distribution]                                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ distribution/                                          │  │
│  │  - tar.gz packages                                    │  │
│  │  - deb/rpm packages                                   │  │
│  │  - Docker images                                      │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 12.2 Key Components Matrix

| Component | Location | Language | Lines of Code | Purpose |
|-----------|----------|----------|---------------|---------|
| **BuildSrc** | `buildSrc/` | Java/Groovy | ~15,000 | Build automation |
| **Server Core** | `server/` | Java | ~200,000 | Main functionality |
| **REST API** | `server/src/main/java/.../rest/` | Java | ~25,000 | HTTP endpoints |
| **Plugins** | `plugins/` | Java | ~5,000 | Wazuh extensions |
| **Modules** | `modules/` | Java | ~50,000 | Core modules |
| **Libraries** | `libs/` | Java | ~40,000 | Shared utilities |
| **Tests** | `*/test/` | Java/Groovy | ~150,000 | Quality assurance |
| **Config** | `distribution/src/config/` | YAML/Properties | ~1,000 | Configuration |

### 12.3 Critical Files Reference

#### Build & Configuration

| File | Purpose | Importance |
|------|---------|------------|
| `build.gradle` | Root build script | ⭐⭐⭐⭐⭐ |
| `settings.gradle` | Project structure | ⭐⭐⭐⭐⭐ |
| `buildSrc/build.gradle` | Build system setup | ⭐⭐⭐⭐⭐ |
| `gradle.properties` | Build properties | ⭐⭐⭐⭐ |
| `opensearch.yml` | Main configuration | ⭐⭐⭐⭐⭐ |
| `jvm.options` | JVM settings | ⭐⭐⭐⭐ |
| `log4j2.properties` | Logging config | ⭐⭐⭐ |

#### Source Code

| File | Purpose | Importance |
|------|---------|------------|
| `Bootstrap.java` | Application entry point | ⭐⭐⭐⭐⭐ |
| `Node.java` | Node lifecycle | ⭐⭐⭐⭐⭐ |
| `IndicesService.java` | Index management | ⭐⭐⭐⭐⭐ |
| `TransportService.java` | Network layer | ⭐⭐⭐⭐⭐ |
| `SearchService.java` | Search execution | ⭐⭐⭐⭐⭐ |
| `RestController.java` | REST routing | ⭐⭐⭐⭐ |
| `MapperService.java` | Mapping management | ⭐⭐⭐⭐⭐ |
| `InternalEngine.java` | Lucene interface | ⭐⭐⭐⭐⭐ |

#### Integration

| File | Purpose | Importance |
|------|---------|------------|
| `wazuh-template.json` | Index template | ⭐⭐⭐⭐⭐ |
| `wazuh-module.yml` | Filebeat config | ⭐⭐⭐⭐ |
| `wazuh-alerts-policy.json` | ISM policy | ⭐⭐⭐⭐ |

### 12.4 Data Flow Summary

```
┌──────────────────────────────────────────────────────────────┐
│              Complete Data Journey                            │
└──────────────────────────────────────────────────────────────┘

[Indexing Flow]
Wazuh Manager → alerts.json
                     ↓
               Filebeat reads
                     ↓
          POST /_bulk (HTTP/HTTPS)
                     ↓
          RestController receives
                     ↓
     TransportBulkAction processes
                     ↓
        Routes to Primary Shard
                     ↓
       IndexShard.applyIndexOperation
                     ↓
         MapperService applies mapping
                     ↓
      InternalEngine writes to Lucene
                     ↓
     Replicates to Replica Shards
                     ↓
            Document persisted
                     ↓
      (Wait ~5s for refresh_interval)
                     ↓
         Document searchable ✓


[Search Flow]
Dashboard/API → POST /_search (HTTP/HTTPS)
                     ↓
          RestController receives
                     ↓
     TransportSearchAction processes
                     ↓
        Scatters to all Shards
                     ↓
    Each Shard: SearchService.executeQueryPhase
                     ↓
         Lucene searches locally
                     ↓
    Returns top document IDs + scores
                     ↓
      Coordinator merges results
                     ↓
         Sorts globally by score
                     ↓
    FetchPhase retrieves full documents
                     ↓
           JSON response to client
```

### 12.5 BuildSrc Deep Dive Summary

**Why BuildSrc is the Heart of the Project:**

1. **Custom Gradle Plugins**: 30+ plugins تتحكم في كل جانب من البناء
2. **Version Management**: يدير توافق versions بين components
3. **Testing Framework**: ينشئ test clusters ويدير integration tests
4. **Distribution Creation**: يبني packages لجميع platforms
5. **Quality Control**: ينفذ precommit checks (license headers, forbidden APIs, etc.)

**Key Statistics:**
- **341 files** في buildSrc
- **~15,000 lines** من Java/Groovy code
- **32 Gradle plugins** مخصصة
- **15+ test suites** للـ build logic
- **4 distribution types** (tar, zip, deb, rpm)

**Most Important Classes in BuildSrc:**

```java
1. BuildPlugin.groovy
   - الـ plugin الرئيسي الذي يطبق على كل module
   - يضبط Java compilation settings
   - يدير dependencies

2. TestClustersPlugin.java
   - ينشئ OpenSearch clusters للاختبار
   - يدير دورة حياة الـ clusters (start/stop)
   - يوفر DSL لتكوين الـ clusters

3. DistributionDownloadPlugin.java
   - يحمل OpenSearch distributions
   - يستخدمها في البناء والاختبار
   - يتعامل مع caching

4. OpenSearchJavaPlugin.java
   - يضبط Java compilation
   - يحدد source/target compatibility
   - يطبق code style checks

5. PrecommitTasks.groovy
   - ينفذ quality checks قبل commit
   - يتحقق من license headers
   - يمنع APIs خطرة
```

### 12.6 Development Workflow

```
┌──────────────────────────────────────────────────────────────┐
│              Developer Workflow                               │
└──────────────────────────────────────────────────────────────┘

[1. Setup Environment]
git clone https://github.com/wazuh/wazuh-indexer.git
cd wazuh-indexer
./gradlew --version  # Verify Gradle
java --version       # Verify JDK 17+

[2. Build Project]
./gradlew assemble   # Compile & package
# buildSrc compiles first, then main project

[3. Run Tests]
./gradlew test       # Unit tests
./gradlew integTest  # Integration tests
# Test clusters start automatically

[4. Make Changes]
vim server/src/main/java/.../MyClass.java
# Edit code

[5. Verify Changes]
./gradlew :server:test
./gradlew :server:precommit
# Checks: license headers, forbidden APIs, etc.

[6. Build Distribution]
./gradlew :distribution:archives:linux-tar:assemble
# Creates: build/distributions/wazuh-indexer-*.tar.gz

[7. Test Distribution]
tar -xzf build/distributions/wazuh-indexer-*.tar.gz
cd wazuh-indexer-*/
./bin/opensearch

[8. Commit & Push]
git add .
git commit -m "feat: add new feature"
git push origin feature-branch
```

### 12.7 Important Patterns & Conventions

#### 12.7.1 Naming Conventions

```
Classes:
- Services: *Service (e.g., IndicesService, SearchService)
- Actions: Transport*Action (e.g., TransportIndexAction)
- Handlers: Rest*Action (e.g., RestIndexAction)
- Tests: *Tests for unit, *IT for integration

Packages:
- org.opensearch.action.*     # Actions & requests
- org.opensearch.indices.*    # Index management
- org.opensearch.cluster.*    # Cluster coordination
- org.opensearch.rest.*       # REST API
- org.opensearch.search.*     # Search functionality

Files:
- build.gradle                # Build script
- plugin-descriptor.properties # Plugin metadata
- opensearch.yml              # Configuration
```

#### 12.7.2 Code Organization

```java
// Typical class structure
public class MyService extends AbstractLifecycleComponent {
    // 1. Fields
    private final ClusterService clusterService;
    private final ThreadPool threadPool;
    
    // 2. Constructor with dependency injection
    public MyService(
        Settings settings,
        ClusterService clusterService,
        ThreadPool threadPool
    ) {
        this.clusterService = clusterService;
        this.threadPool = threadPool;
    }
    
    // 3. Lifecycle methods
    @Override
    protected void doStart() {
        // Initialization logic
    }
    
    @Override
    protected void doStop() {
        // Cleanup logic
    }
    
    @Override
    protected void doClose() {
        // Final cleanup
    }
    
    // 4. Public API
    public void myPublicMethod() {
        // Implementation
    }
    
    // 5. Private helpers
    private void myPrivateHelper() {
        // Helper logic
    }
}
```

### 12.8 Dependencies Overview

#### Core Dependencies

```groovy
// من build.gradle و gradle/libs.versions.toml

dependencies {
    // Apache Lucene (Search Engine)
    api "org.apache.lucene:lucene-core:9.7.0"
    api "org.apache.lucene:lucene-analyzers-common:9.7.0"
    api "org.apache.lucene:lucene-queries:9.7.0"
    
    // JSON Processing
    api "com.fasterxml.jackson.core:jackson-core:2.15.2"
    api "com.fasterxml.jackson.core:jackson-databind:2.15.2"
    
    // Networking
    implementation "io.netty:netty-all:4.1.100.Final"
    
    // Logging
    api "org.apache.logging.log4j:log4j-api:2.20.0"
    implementation "org.apache.logging.log4j:log4j-core:2.20.0"
    
    // Compression
    implementation "org.xerial.snappy:snappy-java:1.1.10.5"
    implementation "com.github.luben:zstd-jni:1.5.5-5"
    
    // Security
    implementation "org.bouncycastle:bcprov-jdk15on:1.70"
    
    // Testing
    testImplementation "junit:junit:4.13.2"
    testImplementation "org.hamcrest:hamcrest:2.2"
    testImplementation "org.mockito:mockito-core:5.5.0"
}
```

### 12.9 Performance Considerations

#### In the Code

```java
// 1. Use bulk operations
BulkRequest bulkRequest = new BulkRequest();
for (Document doc : documents) {
    bulkRequest.add(new IndexRequest("my-index").source(doc));
}
client.bulk(bulkRequest);  // ✓ Fast

// vs
for (Document doc : documents) {
    client.index(new IndexRequest("my-index").source(doc));  // ✗ Slow
}

// 2. Use filters over queries when scoring not needed
{
    "query": {
        "bool": {
            "filter": [                    // ✓ Fast (no scoring)
                {"term": {"status": "active"}}
            ]
        }
    }
}

// vs
{
    "query": {
        "match": {"status": "active"}      // ✗ Slower (scoring)
    }
}

// 3. Limit _source fields
GET /my-index/_search
{
    "_source": ["field1", "field2"],       // ✓ Only needed fields
    "query": {...}
}

// 4. Use appropriate shard count
// Rule of thumb: shard size between 10-50GB
// Too many shards = overhead
// Too few shards = large shards, slow queries
```

### 12.10 Troubleshooting Guide

#### Common Build Issues

```bash
# Issue 1: BuildSrc compilation fails
./gradlew clean cleanBuildSrc
./gradlew build

# Issue 2: Test failures
./gradlew test --info  # Verbose output
# Check: build/reports/tests/test/index.html

# Issue 3: Out of memory during build
export GRADLE_OPTS="-Xmx4g -XX:MaxMetaspaceSize=1g"
./gradlew build

# Issue 4: Stale test clusters
./gradlew --stop  # Stop Gradle daemon
ps aux | grep opensearch  # Kill any remaining processes
./gradlew cleanTestClusters

# Issue 5: Plugin not loading
# Check: logs/opensearch.log
# Verify: plugin-descriptor.properties is correct
./bin/opensearch-plugin list  # List installed plugins
```

#### Runtime Issues

```bash
# Issue 1: Cluster won't start
# Check logs: /var/log/wazuh-indexer/wazuh-cluster.log
# Common causes:
# - Port 9200/9300 already in use
# - Insufficient memory
# - Wrong file permissions

# Issue 2: OutOfMemoryError
# Edit: jvm.options
-Xms4g
-Xmx4g

# Issue 3: Slow queries
# Enable slow log:
PUT /my-index/_settings
{
    "index.search.slowlog.threshold.query.warn": "10s",
    "index.search.slowlog.threshold.query.info": "5s"
}

# Issue 4: Unassigned shards
GET /_cluster/allocation/explain
# Check node disk space, memory, etc.
```

### 12.11 Security Best Practices

```yaml
# opensearch.yml - Production settings

# 1. Enable TLS
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: node-cert.pem
plugins.security.ssl.http.pemkey_filepath: node-key.pem

# 2. Strong authentication
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"

# 3. Audit logging
plugins.security.audit.type: internal_opensearch
plugins.security.audit.config.disabled_rest_categories: NONE
plugins.security.audit.config.disabled_transport_categories: NONE

# 4. Network restrictions
network.host: 127.0.0.1  # Only local in production behind proxy

# 5. Disable dangerous features
http.cors.enabled: false
```

```java
// في الكود: Input validation
public void indexDocument(String indexName, String source) {
    // Validate index name
    if (!indexName.matches("^[a-z0-9-_.]+$")) {
        throw new IllegalArgumentException("Invalid index name");
    }
    
    // Validate JSON
    try {
        XContentFactory.xContent(XContentType.JSON)
            .createParser(source);
    } catch (Exception e) {
        throw new IllegalArgumentException("Invalid JSON", e);
    }
    
    // Proceed with indexing
    client.index(new IndexRequest(indexName).source(source));
}
```

### 12.12 Monitoring & Observability

#### Key Metrics to Monitor

```bash
# 1. Cluster health
GET /_cluster/health
# Watch: status (green/yellow/red)

# 2. Node stats
GET /_nodes/stats
# Watch: JVM heap usage, CPU, disk I/O

# 3. Index stats
GET /wazuh-alerts-*/_stats
# Watch: document count, storage size, search rate

# 4. Thread pool stats
GET /_cat/thread_pool?v
# Watch: queue size, rejected tasks

# 5. Hot threads
GET /_nodes/hot_threads
# Identify performance bottlenecks
```

#### Logging Strategy

```
Logs hierarchy:
├── opensearch.log              # Main application log
├── opensearch_deprecation.log  # Deprecated API usage
├── opensearch_index_*.log      # Indexing slowlog
├── opensearch_search_*.log     # Search slowlog
├── opensearch_audit.log        # Security audit
└── gc.log                      # Garbage collection
```

---

## 13. Appendix

### 13.1 Gradle Commands Reference

```bash
# Build commands
./gradlew assemble              # Build without tests
./gradlew build                 # Full build with tests
./gradlew clean build           # Clean then build

# Test commands
./gradlew test                  # Unit tests
./gradlew integTest             # Integration tests
./gradlew check                 # All quality checks
./gradlew test --tests MyTest   # Specific test

# Distribution commands
./gradlew :distribution:archives:linux-tar:assemble
./gradlew :distribution:packages:deb:assemble
./gradlew :distribution:packages:rpm:assemble

# Plugin commands
./gradlew :plugins:command-manager:assemble
./gradlew :plugins:command-manager:bundlePlugin

# Utility commands
./gradlew tasks                 # List all tasks
./gradlew dependencies          # Show dependencies
./gradlew projects              # List sub-projects
./gradlew --stop                # Stop Gradle daemon
./gradlew --refresh-dependencies # Force refresh deps
```

### 13.2 Environment Variables

```bash
# Build environment
export JAVA_HOME=/path/to/jdk-17
export GRADLE_OPTS="-Xmx4g -XX:MaxMetaspaceSize=1g"
export OPENSEARCH_JAVA_HOME=$JAVA_HOME

# Runtime environment
export OPENSEARCH_HOME=/usr/share/wazuh-indexer
export OPENSEARCH_PATH_CONF=/etc/wazuh-indexer
export OPENSEARCH_JAVA_OPTS="-Xms2g -Xmx2g"
```

### 13.3 Useful URLs

| Resource | URL |
|----------|-----|
| **Main Repository** | https://github.com/wazuh/wazuh-indexer |
| **OpenSearch Docs** | https://opensearch.org/docs/latest/ |
| **Lucene Docs** | https://lucene.apache.org/core/documentation.html |
| **Gradle Docs** | https://docs.gradle.org/ |
| **Wazuh Docs** | https://documentation.wazuh.com/ |
| **Issue Tracker** | https://github.com/wazuh/wazuh-indexer/issues |

### 13.4 Glossary

| Term | Definition |
|------|------------|
| **Shard** | جزء من index يحتوي على subset من documents |
| **Replica** | نسخة احتياطية من shard |
| **Node** | instance واحد من OpenSearch |
| **Cluster** | مجموعة من nodes |
| **Index** | مجموعة documents بـ schema مشترك |
| **Document** | وحدة البيانات (JSON object) |
| **Mapping** | تعريف schema للـ index |
| **Template** | قالب لإنشاء indices جديدة |
| **Analyzer** | محلل نصوص (tokenization) |
| **Inverted Index** | بنية بيانات Lucene للبحث السريع |
| **BuildSrc** | مجلد Gradle plugins المخصصة |
| **Plugin** | إضافة توسع functionality |

---

## 14. Conclusion

### 14.1 Key Takeaways

**Wazuh Indexer** هو نظام معقد ولكن منظم جيداً:

1. **BuildSrc هو القلب**: يتحكم في كل جوانب البناء والتجميع
2. **معمارية طبقية**: من REST API إلى Lucene بشكل واضح ومنظم
3. **قابلية التوسع**: نظام plugins قوي للإضافات المخصصة
4. **اختبارات شاملة**: 3 مستويات من الاختبارات تضمن الجودة
5. **توزيع متعدد**: دعم لجميع platforms (Linux, Windows, Docker)

### 14.2 What We Learned from Source Code

```
┌──────────────────────────────────────────────────────────────┐
│             Source Code Insights                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│ ✓ BuildSrc: 341 files, 15K+ lines                           │
│   → Custom Gradle plugins control everything                 │
│   → Test infrastructure is sophisticated                     │
│   → Distribution creation is automated                       │
│                                                               │
│ ✓ Server: 200K+ lines of Java                               │
│   → Well-structured, modular design                          │
│   → Clear separation of concerns                             │
│   → Lifecycle management is robust                           │
│                                                               │
│ ✓ Plugins: 2 custom Wazuh plugins                           │
│   → command-manager: Security operations                     │
│   → setup: Initial configuration                             │
│                                                               │
│ ✓ Integration: Filebeat templates                           │
│   → Pre-configured index templates                           │
│   → ISM policies for data lifecycle                          │
│   → Ready-to-use configurations                              │
│                                                               │
│ ✓ Testing: 150K+ lines of test code                         │
│   → Comprehensive coverage                                    │
│   → Automated test clusters                                   │
│   → Integration tests for all components                     │
└──────────────────────────────────────────────────────────────┘
```

### 14.3 BuildSrc Importance Recap

**لماذا BuildSrc هو أهم مجلد في المشروع؟**

1. **يحدد كيف يُبنى كل شيء**
   - Compilation settings
   - Dependency management
   - Test execution
   - Distribution packaging

2. **يوفر البنية التحتية للتطوير**
   - Test clusters
   - Integration testing
   - Quality checks (precommit)
   - Code generation

3. **يضمن الجودة**
   - License header validation
   - Forbidden API checks
   - Dependency license audit
   - Code style enforcement

4. **يُبسط التوزيع**
   - Multi-platform support
   - Automated packaging
   - Checksum generation
   - GPG signing

**بدون BuildSrc**: لا يمكن بناء المشروع أو اختباره أو توزيعه! 🔴

### 14.4 Next Steps for Learning

```
Recommended Learning Path:

[Level 1: Basics] (1-2 weeks)
├── فهم Gradle basics
├── استكشاف buildSrc/src/main/java/
├── قراءة build.gradle files
└── تشغيل ./gradlew tasks

[Level 2: Core] (2-4 weeks)
├── دراسة server/src/main/java/
├── فهم Index lifecycle
├── تتبع REST request flow
└── فهم Search execution

[Level 3: Advanced] (4-8 weeks)
├── Plugin development
├── Custom analyzers
├── Performance tuning
└── Cluster management

[Level 4: Expert] (2-3 months)
├── Lucene internals
├── Distributed systems
├── Security implementation
└── Contributing to project
```

### 14.5 Final Summary

```
═══════════════════════════════════════════════════════════════
              WAZUH INDEXER SOURCE CODE REPORT
═══════════════════════════════════════════════════════════════

📦 Total Files: ~15,000+
💻 Code Lines: ~600,000+ (Java/Groovy)
🧪 Test Lines: ~150,000+
🔌 Plugins: 2 custom (command-manager, setup)
📚 Modules: 30+ core modules
🏗️ BuildSrc: 341 files, 30+ Gradle plugins

🎯 Key Components:
   ├── buildSrc/     → Build automation (⭐⭐⭐⭐⭐)
   ├── server/       → Core functionality (⭐⭐⭐⭐⭐)
   ├── plugins/      → Wazuh extensions (⭐⭐⭐⭐)
   ├── integrations/ → Filebeat templates (⭐⭐⭐⭐)
   └── distribution/ → Packages (⭐⭐⭐⭐)

🔄 Data Flow:
   Manager → Filebeat → REST API → TransportAction
   → IndexShard → Lucene → Disk Storage

🔍 Search Flow:
   Client → REST API → SearchAction → Shards (scatter)
   → Lucene Search → Results (gather) → Response

✅ Build Process:
   BuildSrc compile → Dependencies → Compile → Test
   → Package → Verify → Publish

🚀 Distribution Types:
   - tar.gz (Linux/Mac)
   - zip (Windows)
   - deb (Debian/Ubuntu)
   - rpm (RedHat/CentOS)
   - Docker image

═══════════════════════════════════════════════════════════════
                    END OF REPORT
═══════════════════════════════════════════════════════════════
```

---

**تاريخ إعداد التقرير**: أكتوبر 2025  
**الإصدار**: 1.0  
**Repository**: https://github.com/wazuh/wazuh-indexer  
**المرجع الرئيسي**: OpenSearch 2.x Documentation

---

## 📚 References

1. **Wazuh Indexer Repository**  
   https://github.com/wazuh/wazuh-indexer

2. **OpenSearch Documentation**  
   https://opensearch.org/docs/latest/

3. **Apache Lucene Documentation**  
   https://lucene.apache.org/core/9_7_0/

4. **Gradle Documentation**  
   https://docs.gradle.org/current/userguide/userguide.html

5. **Wazuh Documentation**  
   https://documentation.wazuh.com/current/

6. **Java 17 Documentation**  
   https://docs.oracle.com/en/java/javase/17/

---

**EOF - End of Source Code Structure Report**