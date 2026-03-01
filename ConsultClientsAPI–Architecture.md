# ConsultClients API – Enterprise Architecture Documentation

**Version:** 1.0  
**Environment:** Production  
**Compliance:** Follows the rules of the LGPC (General Personal Data Protection Law) and the company's internal guidelines to ensure that all processes are conducted with integrity
Onwer: Flavio D. Lizzardo

---

## 1. Executive Overview

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

Each zone is protected by firewall policies and strict routing controls.

---

## 2. Internet Zone

**Public Access Entry Point**

| Logical Endpoint              | Public IP   | Port | Protocol             |
|--------------------------------|-------------|------|----------------------|
| consultclients.company.sant    | 200.168.0.20| 443  | HTTPS (TLS 1.2/1.3)  |

**Description:**

- This is the only publicly exposed endpoint.  
- All traffic is encrypted using TLS 1.2/1.3.  
- No internal services are directly accessible from the Internet.  

1. **External Client** connects to `consultclients.company.sant` via public IP `200.168.0.20` using HTTPS on port 443.
2. Traffic is routed through the **DMZ Zone**, reaching the **Nginx VIP** (`10.0.0.100:443`).
3. Nginx servers terminate TLS/SSL and forward requests via HTTP (`8080`) to the **API VIP** (`10.0.0.120:8080`) in the Application Zone.
4. The **API servers** process requests and interact with:
   - **Redis VIP** (`10.0.0.130:6379`) for caching.
   - **PostgreSQL VIP** (`10.0.0.150:5432`) for persistent storage.
   - **Kafka VIP** (`10.0.0.140:9092`) for event streaming.
5. **Prometheus** scrapes metrics from APIs, Redis, PostgreSQL, and Kafka.
6. **Grafana** connects to Prometheus via the Monitoring VIP (`10.0.0.160`) to visualize dashboards.

- **Descriptive:** Entry point for users who access the application securely and is exposed with TLS1.2 and 1.3 level certificates throughout the cycle and follows the same policy internally.

---

## Internal Access Flow (Intranet)

1. **Internal users** bypass the public IP and connect directly to the **API VIP** (`10.0.0.120:8080`).
2. Requests are processed by API servers without traversing the DMZ.
3. APIs interact with the same Redis, PostgreSQL, and Kafka clusters via their VIPs.
4. Monitoring remains identical, with Prometheus collecting metrics and Grafana providing visualization.

**Describe:** Internal access is faster and restricted to the corporate network, reducing exposure to external threats.

**Traffic Flow:**  
External Client → Firewall → DMZ (Nginx VIP)

---

## 3. DMZ Zone – Nginx Reverse Proxy Cluster

**Cluster Overview (VIP + Physical Nodes)**

| Component      | VIP        | Physical Nodes       | Port | Middleware        |
|----------------|------------|----------------------|------|-------------------|
| Nginx Cluster  | 10.0.0.100 | 10.0.0.10 / 10.0.0.11| 443  | Nginx + OpenSSL   |

**Logical Mapping:**

- nginx-vip.dmz → 10.0.0.100  
- nginx-1.dmz → 10.0.0.10  
- nginx-2.dmz → 10.0.0.11  

**Responsibilities:**

- TLS termination  
- Reverse proxy  
- Load balancing  
- Request filtering  
- Rate limiting  
- Security header enforcement  
- Logging  

**Operating Mode:** Active/Active with VIP failover.

---

## 4. Application Zone – Tomcat API Cluster

**Cluster Overview (VIP + Physical Nodes)**

| Component     | VIP        | Physical Nodes       | Port | Middleware                  |
|---------------|------------|----------------------|------|-----------------------------|
| API Cluster   | 10.0.0.120 | 10.0.0.20 / 10.0.0.21| 8080 | Tomcat + JVM + Spring Boot  |

**Logical Mapping:**

- api-vip → 10.0.0.120  
- api-1 → 10.0.0.20  
- api-2 → 10.0.0.21  

**Middleware Stack per API Node:**

- **OS:** Linux Enterprise  
- **Runtime:** OpenJDK 17 LTS (G1GC enabled, heap tuning configured)  
- **Container:** Apache Tomcat 9.x / 10.x  
- **Framework:** Spring Boot + Spring Security (RBAC)  
- **Database Connectivity:** PostgreSQL JDBC Driver + HikariCP pool  
- **Cache Client:** Redis (Lettuce/Jedis)  
- **Messaging Client:** Kafka Java Client  
- **Logging:** Logback/Log4j2 with structured logging  
- **Metrics:** Micrometer + Prometheus endpoint  

**Functional Responsibilities:**

- REST API processing  
- Authentication via `db_user`  
- Business operations via `db_clients`  
- Redis caching  
- Kafka event publishing  
- Metrics exposure  

**Architecture:** Fully stateless to allow horizontal scaling.

---

## 5. Data Zone

### 5.1 Redis Cluster

| Component      | VIP        | Physical Nodes       | Port | Middleware |
|----------------|------------|----------------------|------|------------|
| Redis Cluster  | 10.0.0.130 | 10.0.0.30 / 10.0.0.31| 6379 | Redis 6+   |

**Logical Mapping:**

- redis-vip → 10.0.0.130  
- redis-1→ 10.0.0.30  
- redis-2 → 10.0.0.31  

**Purpose:**  
- Read cache  
- Token/session caching  
- Performance optimization  
- Replication enabled  

---

### 5.2 Kafka Cluster

| Component      | VIP        | Physical Nodes       | Port | Middleware        |
|----------------|------------|----------------------|------|-------------------|
| Kafka Cluster  | 10.0.0.140 | 10.0.0.40 / 10.0.0.41| 9092 | Apache Kafka 3.x  |

**Logical Mapping:**

- kafka-vip → 10.0.0.140  
- kafka-1 → 10.0.0.40  
- kafka-2 → 10.0.0.41  

**Purpose:**  
- Event streaming  
- Asynchronous integration  
- Message durability  
- Replication factor ≥ 2  

---

### 5.3 PostgreSQL Cluster

| Component          | VIP        | Physical Nodes       | Port | Middleware       |
|--------------------|------------|----------------------|------|------------------|
| PostgreSQL Cluster | 10.0.0.150 | 10.0.0.50 / 10.0.0.51| 5432 | PostgreSQL 14+   |

**Logical Mapping:**

- postgres-vip → 10.0.0.150  
- postgres-primary → 10.0.0.50  
- postgres-replica → 10.0.0.51  

**Logical Databases:**

- **db_user**  
  - Authentication credentials  
  - Role mapping  
  - Permission mapping  
  - Audit logs  

- **db_clients**  
  - Client records  
  - Business transactions  
  - Operational data  

**Replication:** Streaming replication enabled.

---

## 6. Monitoring Zone

**Cluster Overview (VIP + Nodes)**

| Component   | VIP        | Physical Nodes       | Port       | Middleware            |
|-------------|------------|----------------------|------------|-----------------------|
| Monitoring  | 10.0.0.160 | 10.0.0.60 / 10.0.0.61| 9090 / 3000| Prometheus + Grafana  |

**Logical Mapping:**

- prometheus.monitor → 10.0.0.60  
- grafana.monitor → 10.0.0.61  
- monitor-vip → 10.0.0.160  

**Purpose:**

- Metrics collection  
- Health checks  
- Alerting  
- Performance dashboards  

**Prometheus scrapes:**

- API metrics endpoint  
- Redis exporter  
- PostgreSQL exporter  
- Kafka exporter  

---

## 7. External vs Internal Access Model

**External Flow:**

Internet (200.168.0.20)  
→ Firewall  
→ nginx-vip (10.0.0.100)  
→ api-vip (10.0.0.120)  
→ Data VIPs  

**Internal Flow:**

Corporate Network  
→ api-vip (10.0.0.120)  
→ Data VIPs  

Internal access bypasses DMZ but maintains authentication and RBAC enforcement.

---

## 8. High Availability Strategy

- VIP abstraction for all clusters  
- Redundant nodes in every zone  
- No single point of failure  
- Database replication  
- Kafka broker replication  
- Redis replication  
- Monitoring for proactive failure detection  

**Targets:** RTO (Recovery Time Objective)

- RTO < 30 minutes  
- RPO < 5 minutes  

---

## 9. Security Strategy

- TLS 1.2 / 1.3 encryption externally  
- Firewall segmentation between zones  
- Internal-only exposure for Data Zone  
- RBAC enforced at API layer  
- LGPD-aligned data handling  
- Logging and monitoring enabled  

---

## 10. Final Conclusion

The ConsultClients architecture provides:

- Clear VIP-to-physical-node mapping  
- Multi-layer high availability  
- Secure segmentation across five zones  
- Enterprise-grade middleware resilience  
- Observability and operational transparency  
- Compliance with LGPD and internal governance  

This infrastructure is fully prepared for mission-critical production environments requiring **high reliability, scalability, and secure data handling**.
