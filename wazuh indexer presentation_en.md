# Wazuh Indexer: Comprehensive Technical Presentation

---

## Introduction to Wazuh Indexer

Wazuh Indexer stands as one of the fundamental components within the Wazuh cybersecurity platform, serving as the real-time search and analytics engine for security data. This component receives security data that has been analyzed by the Wazuh server, then indexes and stores it in an organized manner that enables rapid and efficient querying through the Wazuh Dashboard interface.

Wazuh Indexer leverages OpenSearch technologies at its core, providing full-text search capabilities and advanced analytics for large-scale security data. Data is stored in JSON document format, where each document contains a set of keys and values representing security events with complete details.

## Architectural Structure of Wazuh Indexer

The architecture of Wazuh Indexer is characterized by flexibility and scalability, allowing deployment in various configurations based on environment requirements. The system supports single-node configuration for smaller environments, or multi-node cluster configurations for large production environments requiring high availability and advanced processing capabilities.

### Understanding Shards and Replicas

Wazuh Indexer employs a data distribution mechanism using shards, which are independent storage units distributed across cluster nodes. Each shard represents a functional sub-index capable of operating independently, enabling parallel operations and enhancing overall system performance.

In addition to primary shards, the system supports creating replicas of each shard, ensuring operational continuity in case of node failure within the cluster. These replicas are distributed across different nodes to prevent data loss and achieve high availability.

![Wazuh Indexer Cluster Architecture](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/wazuh_indexer_cluster.svg)

## Internal Operating Mechanism of the Indexer

The indexing process begins when the Wazuh server sends analyzed data to Wazuh Indexer through Filebeat. Filebeat reads the alert file stored at `/var/ossec/logs/alerts/alerts.json` on the Wazuh server, then sends these alerts as JSON documents to the Wazuh Indexer API.

Upon receiving data, Wazuh Indexer applies a series of preprocessing operations to the documents. These operations include analyzing geographic data from IP addresses, converting timestamps to unified format, and removing unnecessary fields. These operations are executed through predefined processing pipelines in the `pipeline.json` file.

After processing, documents are distributed to appropriate shards based on a hashing algorithm. The system determines the target shard by applying a hash function to the document identifier, ensuring balanced data distribution across all available shards. The document is then stored in the primary shard and replicated to all associated replica shards.

### Query and Search Operations

When a user submits a query through the Wazuh Dashboard interface, the request is routed to Wazuh Indexer, which coordinates the search operation across all relevant shards. Each shard processes part of the query independently and in parallel, significantly accelerating the search process.

The system collects results from all shards, then merges and sorts them according to specified query criteria. Intelligent caching techniques are employed to improve performance for recurring queries, reducing response time and conserving system resources.

![Data Flow in Wazuh System](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/wazuh_data_flow_en.svg)

## Protocols and Technologies Utilized

Wazuh Indexer relies on a set of standard protocols and technologies that ensure high security and reliable performance across different production environments.

### HTTPS Protocol and Encryption

Wazuh Indexer uses HTTPS protocol on port 9200 by default for all communication operations. The system requires activation of valid TLS certificates to ensure data encryption during transit between various system components. X.509 certificates are used for mutual authentication between clients and servers, preventing man-in-the-middle attacks and ensuring data integrity.

The system supports TLS version 1.2 as minimum requirement, with configuration options for using newer versions. Security certificates are stored in `/etc/wazuh-indexer/certs/` and managed through the main configuration file `opensearch.yml`.

### REST API Interface

Wazuh Indexer provides a comprehensive API based on REST protocol, enabling interaction with all system functions through standard HTTP requests. The API supports multiple operations including creating and deleting indices, searching data, managing cluster settings, and monitoring system status.

The system employs robust authentication mechanisms based on encrypted usernames and passwords, with full support for role and permission management. The API can be accessed through various tools such as cURL or through the Dev Tools utility integrated into the Wazuh Dashboard.

### JSON-Based Indexing Mechanism

The data storage structure in Wazuh Indexer fundamentally relies on JSON format. Each security event is represented as a separate JSON document containing a set of structured fields. These fields include information such as timestamp, agent identifier, alert severity level, matched rule information, and event-specific data.

The system uses predefined schemas known as templates to ensure data structure consistency across all indices. These templates are automatically applied when creating new indices, ensuring all fields are indexed correctly and data types match expectations.

### Ingest Pipelines for Data Processing

Wazuh Indexer provides a powerful mechanism for processing data before indexing through ingest pipelines. These pipelines are defined in JSON files and contain a series of processors that apply various transformations to incoming data.

Utilized processors include converting geographic data from IP addresses using GeoIP databases, converting timestamps to ISO8601 unified format, creating dynamic index names based on date, and removing unnecessary fields to reduce storage size. These pipelines are configured in the `pipeline.json` file located at `/usr/share/filebeat/module/wazuh/alerts/ingest/`.

### Distribution and Replication Mechanism

Wazuh Indexer uses consistent hashing algorithm to distribute documents across different shards. When adding a new document, the hash value is calculated from the document identifier, then the target shard is determined based on this value. This mechanism ensures balanced data distribution even when adding or removing shards from the system.

Replication is managed through a robust synchronization mechanism ensuring all replicas of a specific shard contain exactly the same data. When writing a new document, it is first written to the primary shard, then synchronously copied to all replica shards before returning success confirmation to the client. This mechanism ensures strong consistency of data across all copies.

![Protocols and Technologies in Wazuh Indexer](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/indexer_protocols_en.svg)

## Index Patterns in Wazuh Indexer

Wazuh Indexer relies on a set of predefined index patterns to organize and store different types of security data. Each pattern serves a specific purpose within the security monitoring ecosystem and contains a specialized data structure suitable for the type of information stored.

### Alert Index Pattern: wazuh-alerts-*

This pattern is considered the most important in the system, storing all security alerts generated when events match defined detection rules. The Wazuh server analyzes incoming events from agents, and upon detecting suspicious activity matching a specific rule, an alert is created and saved first to the alerts.json file, then sent to Wazuh Indexer for indexing.

A new index is created for each day by default, such as wazuh-alerts-4.x-2023.03.17 for March 17th. This time-based partitioning facilitates data management and improves query performance by enabling selection of specific indices based on the required time range.

### Archive Index Pattern: wazuh-archives-*

While the alert index stores only events that generated alerts, the archive index can be enabled to store all events received by the Wazuh server regardless of alert generation. This index proves useful for subsequent analysis and regulatory compliance requirements where maintaining a complete record of all events may be necessary.

It should be noted that enabling this index will significantly increase storage requirements, therefore assessing the actual need before activation is recommended. Activation requires modifying settings in the ossec.conf file on the Wazuh server and the filebeat.yml file on the Filebeat server.

### Monitoring Index Pattern: wazuh-monitoring-*

This index stores a historical record of connection states for all Wazuh agents registered in the system. Possible states include active, disconnected, pending, or never connected. Each agent's status is recorded every fifteen minutes by default, providing clear visibility into connection state evolution over time.

The Wazuh Dashboard uses this data to display important information such as the number of active agents and their historical evolution within specified time frames. A new index is created weekly by default for this data type.

### Statistics Index Pattern: wazuh-statistics-*

This index contains performance and usage information for the Wazuh server, including the number of decoded events, received bytes, and active TCP sessions. The dashboard queries the Wazuh Manager API to collect this information and store it in the statistics index.

Statistics are stored every five minutes by default, with a new index created weekly. Administrators can use this information to monitor system performance and identify any processing or reception issues before they impact security operations.

### State Index Patterns: wazuh-states-*

This group includes several specialized index patterns for storing system inventory and vulnerability information. The wazuh-states-vulnerabilities-* index stores information about security vulnerabilities discovered in monitored systems with details such as severity level, status, and affected software.

Inventory indices include several specialized patterns such as wazuh-states-inventory-packages-* for installed software, wazuh-states-inventory-processes-* for running processes, and wazuh-states-inventory-ports-* for open ports. These indices provide comprehensive visibility into each monitored system's state and help detect unauthorized changes and deviations from established baselines.

![Index Patterns and Functions](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/index_patterns_structure_en.svg)

## Interaction with Other System Components

Wazuh Indexer represents a vital convergence point in the Wazuh cybersecurity platform, interacting closely with all other major system components. This organized interaction ensures smooth data flow from collection points on monitored systems through to display and analysis on the dashboard.

### Interaction with Wazuh Manager

Interaction begins when the Wazuh server processes incoming events from agents and applies detection rules to them. When an event matches a specific rule, a security alert is created and recorded in the alerts.json file located in the local path on the server. The server also maintains a copy in the alerts.log file in text format for historical and backup purposes.

The server formats timestamps and adds additional contextual information to alerts before saving them, facilitating subsequent indexing operations. This information includes the agent identifier that sent the event, alert severity level, matched rule information, and event-specific data. This unified formatting ensures all alerts follow the same structure upon reaching Wazuh Indexer.

### Interaction with Filebeat

Filebeat functions as a data transport bridge between the Wazuh server and Wazuh Indexer. Filebeat continuously monitors the alerts.json file, and when new alerts appear, it reads and converts them to an appropriate format for transmission. Filebeat ensures no alert is sent twice through an intelligent tracking mechanism that preserves the last read position in the file.

Before sending data, Filebeat applies a series of transformations defined in configuration files and templates. These transformations include adding additional fields such as source system name, adding classification tags, and applying specified filters. Filebeat sends data via encrypted HTTPS protocol to the Wazuh Indexer API using configured authentication mechanisms.

Filebeat also supports automatic retry mechanisms in case of connection failure to Wazuh Indexer, ensuring no data loss during temporary outages. It maintains data in temporary memory and retries according to a gradual backoff policy until the operation succeeds.

### Interaction with Wazuh Dashboard

The Wazuh Dashboard relies entirely on Wazuh Indexer to obtain data displayed to users. When a user opens one of the analysis or report pages, the dashboard sends complex queries to the Wazuh Indexer API to fetch required data. These queries use an advanced query language supporting filtering, grouping, sorting, and statistical aggregation.

The dashboard employs intelligent caching mechanisms for recurring queries, reducing load on Wazuh Indexer and improving user response time. It also supports automatic periodic refresh of displayed data without requiring manual page refresh, providing real-time visibility into the security posture.

The dashboard also enables users to create custom dashboards and various visualizations, all based on data indexed in Wazuh Indexer. Users can define complex queries and save them for later reuse or share them with other team members.

![Component Interaction Diagram](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/component_interaction_en.svg)

## Performance Tuning and Optimization

Operating Wazuh Indexer in large production environments requires several optimizations to default settings to ensure optimal performance and high reliability. These optimizations include memory management and proper configuration of shards and replicas.

### Memory Locking

Memory locking is considered one of the most important performance settings in Wazuh Indexer. When the operating system begins using swap file for memory, Wazuh Indexer performance may be significantly affected. Therefore, it is essential to prevent the operating system from moving Java Virtual Machine memory to hard disk by enabling memory locking.

This feature is activated by adding the line bootstrap.memory_lock: true to the main configuration file opensearch.yml. Additionally, the operating system must be configured to allow the Wazuh Indexer process to lock sufficient memory by modifying resource limits in systemd configuration files.

After applying these settings, success should be verified by querying node status and confirming that the mlockall value is set to true. In case of failure, an error message will appear in the Wazuh Indexer log file indicating inability to lock JVM memory.

### JVM Heap Size Configuration

An appropriate size for JVM Heap memory must be determined based on available RAM in the system. The general rule is to allocate half of system memory to JVM Heap while not exceeding a maximum of 32 gigabytes. The Xms minimum and Xmx maximum values should be identical to prevent memory reallocation during runtime, which could affect performance.

For example, in a system with 8 gigabytes of memory, allocating 4 gigabytes to JVM Heap is recommended by adding the lines -Xms4g and -Xmx4g to the jvm.options file. The remaining half of memory is used by the operating system and other processes, and for file system level caching operations, improving read and write performance.

### Optimizing Shard and Replica Count

Determining the correct number of shards and replicas is one of the most important design decisions in Wazuh Indexer. The general rule is that the number of primary shards should equal the number of nodes in the cluster, ensuring balanced load distribution and maximum utilization of all node resources.

Regarding replicas, the optimal number depends on the required availability level and available storage space. In a three-node cluster, using one replica provides good balance between availability and storage requirements, allowing the system to continue operating even if one node fails. Using two replicas provides greater protection but triples storage requirements.

It should be noted that the number of shards cannot be changed after index creation without performing a re-indexing operation. Therefore, careful planning of shard count before beginning system use in production environment is important. Conversely, the number of replicas can be modified at any time without needing to re-index data.

## Re-indexing Operations

The re-indexing process is used when changes occur to the data schema or when needing to copy data from an old index to a new index. These changes may be necessary during system updates, when changing shard configuration, or when applying new templates requiring different data formats.

Re-indexing operations can be performed through the Dev Tools interface in Wazuh Dashboard or through command line using cURL. The operation uses a special API called _reindex that copies all documents from the source index to the destination index while applying any necessary transformations.

During re-indexing, data is copied in batches sequentially, reducing impact on system performance. The output displays detailed information about the operation including number of copied documents, time consumed, any version conflicts, and any errors that occurred during the operation. Progress for large indices can be monitored through separate queries using the _tasks API.

After successful completion of re-indexing, the old index can be deleted if no longer needed, or retained as backup for a specified period. It is important to ensure that the dashboard and other applications are configured to use the new index pattern before deleting the old index to avoid losing data access.

![Performance Tuning Guide](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/performance_tuning_en.svg)

## Security and Protection in Wazuh Indexer

Security forms a fundamental element in Wazuh Indexer design, particularly as it handles sensitive security data requiring strong protection from unauthorized access and tampering. The system relies on multiple integrated security layers to ensure data confidentiality, integrity, and availability.

### Encryption and Certificate Management

Wazuh Indexer mandates HTTPS protocol usage for all incoming and outgoing connections with support for TLS versions 1.2 or newer. Digital certificates are stored in `/etc/wazuh-indexer/certs/` and include server certificate, private key, and certificate authority certificates. The system requires all certificates to be valid and trusted to ensure establishment of secure connections.

The system also supports mutual authentication between client and server through client certificates, providing an additional security layer for highly sensitive environments. This feature can be configured through the opensearch.yml file with specification of required certificate paths and verification policies.

### Identity and Permission Management

Wazuh Indexer provides an advanced system for managing users, roles, and permissions based on Role-Based Access Control (RBAC) model. Custom roles can be defined for each user group with precise permission specification at index, document, or field level.

Authentication data is stored securely with password encryption using strong hashing algorithms such as bcrypt. The system also supports integration with external identity providers through protocols such as LDAP, SAML, and OpenID Connect, facilitating integration into existing organizational security infrastructures.

### Data Protection During Transit and Storage

All data transmitted between different system components is protected through TLS encryption. This includes connections between agents and Wazuh server, between Filebeat and Wazuh Indexer, and between dashboard and Wazuh Indexer. The system also supports encrypting internal communications between cluster nodes to protect exchanged data during replication and balancing operations.

Regarding stored data, Wazuh Indexer provides the ability to enable disk-level encryption as an optional feature. Operating system level disk encryption technologies such as LUKS in Linux or BitLocker in Windows can be used to protect data stored in `/var/lib/wazuh-indexer/` from unauthorized physical access.

### Security Monitoring and Auditing

Wazuh Indexer maintains detailed logs of all operations and security events in `/var/log/wazuh-indexer/wazuh-indexer.log`. These logs include successful and failed authentication attempts, operations performed on indices, configuration changes, and various errors and warnings. Log detail level can be configured as needed while considering balance between security and performance.

Regular monitoring of these logs is recommended, along with integration with Security Information and Event Management (SIEM) systems for early detection of any suspicious activities or intrusion attempts. Automatic alerting features can also be enabled when critical security events occur, such as repeated failed authentication attempts or unauthorized resource access attempts.

## Daily Maintenance and Management

Managing Wazuh Indexer in production environments requires following regular maintenance practices to ensure operational continuity and optimal performance. These practices include monitoring system health, managing storage space, and backup and recovery operations.

### System Health and Performance Monitoring

Wazuh Indexer provides several APIs for monitoring cluster state and performance. The `/_cluster/health` interface can be used to obtain a quick overview of cluster state, which may be green (all shards active), yellow (some replicas unavailable), or red (some primary shards unavailable).

The `/_nodes/stats` interface provides detailed information about resource usage in each node, including memory usage, processor, disk space, and network statistics. These indicators should be monitored regularly with alerts configured when specific thresholds are exceeded, such as disk usage exceeding eighty percent or JVM memory usage approaching maximum limit.

### Index Lifecycle Management

Over time, large amounts of data accumulate in Wazuh Indexer, consuming significant storage space and affecting performance. Therefore, it is necessary to implement Index Lifecycle Management policies that include automatic deletion or archiving of old indices.

Different retention periods can be specified for each index type based on compliance and usage requirements. For example, alert indices may be retained for ninety days while inventory indices are retained for only thirty days. These policies can be implemented through scheduled tasks using APIs to delete indices based on creation date.

### Backup and Recovery

Regular data backup is critically important for protecting data from loss due to failures, human errors, or cyber attacks. Wazuh Indexer supports snapshot mechanisms that allow creating incremental backups of indices and storing them in external repositories.

A snapshot repository must be configured in advance, which can be a shared file system or cloud storage service such as Amazon S3. Subsequently, snapshot creation operations can be scheduled periodically, daily or weekly depending on data change rate and importance. The snapshot mechanism allows restoring specific data or the entire system to a previous state when needed.

![Security and Maintenance Guide](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/security_maintenance_en.svg)

## Summary and Key Points

Wazuh Indexer represents a pivotal component in the Wazuh cybersecurity platform, providing the fundamental infrastructure for storing, indexing, and retrieving security data with high efficiency. The system's capabilities rely on advanced OpenSearch technologies that provide a full-text search engine with sophisticated analytical capabilities supporting real-time security monitoring operations.

The system's architectural structure is characterized by flexibility and scalability through the shards and replicas mechanism that ensures balanced data distribution across cluster nodes and provides high levels of availability and parallel processing capabilities. This design enables the system to handle large volumes of security data without impacting performance or stability.

Wazuh Indexer interacts closely with other platform components, receiving analyzed data from the Wazuh server through Filebeat and providing it for querying and analysis through the Wazuh Dashboard. This interaction relies on standard security protocols including HTTPS, TLS, and strong authentication, ensuring data protection during transit and storage.

Multiple index patterns in the system provide comprehensive organization of different security data, from alerts and events to inventory information and vulnerabilities. This organization enables effective data lifecycle management and application of customized retention policies for each data type based on compliance and usage requirements.

Maintaining optimal system performance in production environments requires applying specialized tuning and optimization practices including proper memory management, configuring appropriate numbers of shards and replicas, and regular monitoring of system health and resources. Implementing comprehensive backup and recovery strategies is also critically important for ensuring operational continuity and protecting data from loss.

Ultimately, understanding the internal operating mechanism of Wazuh Indexer and the technologies used within it is fundamental for system administrators and cybersecurity professionals who rely on Wazuh for security monitoring operations. This deep understanding enables them to leverage the system's full capabilities, optimize its performance, and ensure its stability across different production environments.
