# ConsultClients API – Enterprise Architecture Documentation

**Version:** 1.2  
**Environment:** Production  
**Compliance:** LGPD + Internal Governance  
**Owner:** Flavio D. Lizzardo  

---

## 1. Overview

The ConsultClients API is an enterprise-grade application architecture designed to support:

- High Availability (HA)  
- Network Segmentation  
- Secure External and Internal Access  
- Observability and Monitoring  
- Data Protection Compliance (LGPD)  
- Horizontal Scalability  
- Middleware Resilience  

The infrastructure is segmented into five protected zones:

- Internet Zone  
- DMZ Zone  
- Application Zone  
- Data Zone  
- Monitoring Zone  

Each zone is protected by firewall policies, VIP abstraction, and strict routing controls.

---

## 2. Internet Zone – Public Access Entry Point

| Logical Endpoint              | Public IP   | Port | Protocol             | Firewall Redundancy          |
|--------------------------------|-------------|------|----------------------|-----------------------------|
| consultclients.company.sant    | 200.168.0.20| 443  | HTTPS (TLS 1.2/1.3) | FW-Internet-A / FW-Internet-B Active/Active with VIP |

**Firewall Nodes:**

| Device           | Physical IP     | VIP           | Role              | Description |
|------------------|-----------------|---------------|------------------|------------|
| FW-Internet-A    | 200.168.0.21    | 200.168.0.20  | Active/Active VIP | Internet border firewall, policy enforcement, NAT, logging |
| FW-Internet-B    | 200.168.0.22    | 200.168.0.20  | Active/Active VIP | Redundant Internet border firewall |

**External Traffic Flow:**

1. External Client → Firewall VIP (`200.168.0.20`)  
2. Firewall → DMZ Nginx VIP (`10.0.0.100:443`)  
3. TLS terminated → API VIP (`10.0.0.120:8080`)  
4. API → Data Zone VIPs: Redis (`10.0.0.130:6379`), PostgreSQL (`10.0.0.150:5432`), Kafka (`10.0.0.140:9092`)  
5. Monitoring → Prometheus + Grafana via Monitoring VIP (`10.0.0.160`)  

---

## 3. Internal Access Flow (Intranet)

- Internal users bypass public firewall and DMZ.  
- Connect directly to **API VIP (`10.0.0.120:8080`)**.  
- API servers communicate with Redis, PostgreSQL, and Kafka via VIPs.  
- Monitoring VIP (`10.0.0.160`) collects metrics from all zones.

---

## 4. DMZ Zone – Nginx Reverse Proxy Cluster

| Component      | VIP        | Physical Nodes       | IP Addresses       | Port | Middleware        | Responsibilities                                    |
|----------------|------------|--------------------|-----------------|------|------------------|---------------------------------------------------|
| Nginx Cluster  | 10.0.0.100 | nginx-1 / nginx-2   | 10.0.0.10 / 10.0.0.11 | 443  | Nginx + OpenSSL   | TLS termination, reverse proxy, load balancing, request filtering, rate limiting, security headers, logging |

**Server Details:**

- **nginx-1:** 10.0.0.10, TLS termination, reverse proxy, access logs, WAF rules.  
- **nginx-2:** 10.0.0.11, redundant node, same middleware configuration as nginx-1.

**Operating Mode:** Active/Active with VIP failover  

---

## 5. Application Zone – ConsultClients API Cluster

| Component         | VIP        | Physical Nodes       | IP Addresses       | Port | Middleware                  | Responsibilities |
|------------------|------------|--------------------|-----------------|------|-----------------------------|-----------------|
| API Cluster       | 10.0.0.120 | api-1 / api-2       | 10.0.0.20 / 10.0.0.21 | 8080 | Kafka Client + Spring Boot  | REST API processing, RBAC enforcement, caching, messaging, business logic, metrics exposure |

**Server Details & Middleware Stack per Node:**

- **api-1** (10.0.0.20)  
  - OS: Linux Enterprise  
  - Runtime: OpenJDK 17 LTS (G1GC, heap tuning configured)  
  - Framework: Spring Boot + Spring Security (RBAC)  
  - Kafka Client: Apache Kafka producer/consumer  
  - Redis Client: Lettuce/Jedis for caching  
  - DB Connectivity: PostgreSQL JDBC + HikariCP  
  - Logging: Logback/Log4j2, structured logging  
  - Metrics: Micrometer endpoint for Prometheus  

- **api-2** (10.0.0.21)  
  - Redundant node, identical configuration as api-1 for horizontal scaling  

**API Endpoints Handled:**  

- `/api/v1/clients` – Client data retrieval  
- `/api/v1/users` – User data and authentication  
- `/api/v1/transactions` – Business transactions  
- `/api/v1/metrics` – Exposed metrics endpoint for Prometheus  

---

## 6. Data Zone

### 6.1 Redis Cluster

| Component      | VIP        | Physical Nodes       | IP Addresses       | Port | Middleware | Responsibilities                       |
|----------------|------------|--------------------|-----------------|------|------------|----------------------------------------|
| Redis Cluster  | 10.0.0.130 | redis-1 / redis-2   | 10.0.0.30 / 10.0.0.31 | 6379 | Redis 6+   | Session caching, read cache, performance optimization, replication |

- **redis-1** (10.0.0.30) – Primary, handles write/read, HA replication.  
- **redis-2** (10.0.0.31) – Replica, read-only, failover for HA.

---

### 6.2 Kafka Cluster

| Component      | VIP        | Physical Nodes       | IP Addresses       | Port | Middleware        | Responsibilities                        |
|----------------|------------|--------------------|-----------------|------|------------------|----------------------------------------|
| Kafka Cluster  | 10.0.0.140 | kafka-1 / kafka-2   | 10.0.0.40 / 10.0.0.41 | 9092 | Apache Kafka 3.x | Event streaming, asynchronous integration, message durability, replication factor ≥ 2 |

- **kafka-1** (10.0.0.40) – Primary broker, topic partitions leader.  
- **kafka-2** (10.0.0.41) – Replica broker, partition follower.

---

### 6.3 PostgreSQL Cluster

| Component          | VIP        | Physical Nodes       | IP Addresses       | Port | Middleware       | Responsibilities                        |
|--------------------|------------|--------------------|-----------------|------|-----------------|----------------------------------------|
| PostgreSQL Cluster | 10.0.0.150 | postgres-primary / postgres-replica | 10.0.0.50 / 10.0.0.51 | 5432 | PostgreSQL 14+  | Persistent storage, business data, authentication, auditing |

**Databases:**

- **db_user** – Authentication, role mapping, audit logs  
- **db_clients** – Client records, business transactions, operational data  

- **postgres-primary** (10.0.0.50) – Main write node  
- **postgres-replica** (10.0.0.51) – Streaming replication for HA  

---

## 7. Monitoring Zone – Prometheus + Grafana

| Component   | VIP        | Physical Nodes       | IP Addresses       | Port       | Middleware            | Responsibilities                   |
|-------------|------------|--------------------|-----------------|------------|---------------------|----------------------------------|
| Monitoring  | 10.0.0.160 | prometheus / grafana | 10.0.0.60 / 10.0.0.61 | 9090 / 3000| Prometheus + Grafana | Metrics collection, dashboards, alerting, health checks |

**Server Details:**

- **prometheus** (10.0.0.60) – Scrapes API, Redis, PostgreSQL, Kafka metrics  
- **grafana** (10.0.0.61) – Visualizes dashboards and alerts  
- **monitor-vip** (10.0.0.160) – Abstracts Prometheus/Grafana nodes for high availability  

---

## 8. High Availability & Security Strategy

- VIP abstraction for all clusters (API, Redis, PostgreSQL, Kafka, Monitoring)  
- Redundant nodes in every zone  
- No single point of failure  
- TLS 1.2/1.3 for external traffic  
- RBAC enforced at API layer  
- Firewall segmentation across zones  
- Continuous monitoring with Prometheus + Grafana dashboards  

**Targets:**  
- RTO < 30 minutes  
- RPO < 5 minutes  

---

## 9. Final Conclusion

The ConsultClients API architecture now ensures:

- Detailed mapping of **VIPs → physical nodes → IPs**  
- Multi-layer high availability and failover  
- Secure segmentation across Internet, DMZ, Application, Data, and Monitoring zones  
- Middleware resilience with **stateless API nodes**, Redis, Kafka, PostgreSQL, and monitoring clusters  
- Observability with dashboards always available via Monitoring VIP (`10.0.0.160`)  
- Compliance with LGPD and internal governance  

This infrastructure is fully prepared for **mission-critical production environments** requiring **high availability, horizontal scalability, and secure data handling**.
