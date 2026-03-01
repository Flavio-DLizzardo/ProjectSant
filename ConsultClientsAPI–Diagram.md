# ConsultClients API – Enterprise Architecture Diagram
# Onwer Flavio D. Lizzardo

```mermaid
flowchart TB

subgraph INTERNET_ZONE["Internet Zone"]
    InternetClient["consultclients.company.sant\n200.168.0.20:443\nHTTPS (TLS 1.2/1.3)"]
end

subgraph DMZ_ZONE["DMZ Zone"]
    NginxVIP["nginx-vip.dmz\n10.0.0.100:443"]
    Nginx1["nginx-1.dmz\n10.0.0.10:443"]
    Nginx2["nginx-2.dmz\n10.0.0.11:443"]
end

subgraph APP_ZONE["Application Zone"]
    APIVIP["api-vip\n10.0.0.120:8080"]
    API1["api-1\n10.0.0.20:8080"]
    API2["api-2\n10.0.0.21:8080"]
end

subgraph DATA_ZONE["Data Zone"]
    subgraph REDIS["Redis Cluster"]
        RedisVIP["redis-vip\n10.0.0.130:6379"]
        Redis1["redis-1\n10.0.0.30:6379"]
        Redis2["redis-2\n10.0.0.31:6379"]
    end

    subgraph KAFKA["Kafka Cluster"]
        KafkaVIP["kafka-vip\n10.0.0.140:9092"]
        Kafka1["kafka-1\n10.0.0.40:9092"]
        Kafka2["kafka-2\n10.0.0.41:9092"]
    end

    subgraph POSTGRES["PostgreSQL Cluster"]
        PostgresVIP["postgres-vip\n10.0.0.150:5432"]
        PostgresPrimary["postgres-primary\n10.0.0.50:5432\nDatabases: db_user, db_clients"]
        PostgresReplica["postgres-replica\n10.0.0.51:5432\nStreaming Replication"]
    end
end

subgraph MONITORING_ZONE["Monitoring Zone"]
    MonitorVIP["monitor-vip\n10.0.0.160:9090/3000"]
    Prometheus["prometheus.monitor\n10.0.0.60:9090"]
    Grafana["grafana.monitor\n10.0.0.61:3000"]
end

%% Flow connections
InternetClient --> NginxVIP
NginxVIP --> APIVIP
APIVIP --> RedisVIP
APIVIP --> KafkaVIP
APIVIP --> PostgresVIP

RedisVIP --> Redis1
RedisVIP --> Redis2

KafkaVIP --> Kafka1
KafkaVIP --> Kafka2

PostgresVIP --> PostgresPrimary
PostgresVIP --> PostgresReplica

Prometheus --> API1
Prometheus --> Redis1
Prometheus --> Kafka1
Prometheus --> PostgresPrimary
Grafana --> Prometheus
