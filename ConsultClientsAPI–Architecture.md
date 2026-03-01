# ConsultClients API – Enterprise Architecture Documentation

**Version:** 1.1  
**Environment:** Production  
**Compliance:** Follows LGPD and internal corporate guidelines to ensure all processes are conducted with integrity  
**Owner:** Flavio D. Lizzardo

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

**Note:** Firewall redundancy is implemented at the Internet border to ensure high availability and continuity of external access.



---

## 2. Internet Zone – Public Access Entry Point

| Logical Endpoint              | Public IP   | Port | Protocol             | Firewall Redundancy          |
|--------------------------------|-------------|------|----------------------|-----------------------------|
| consultclients.company.sant    | 200.168.0.20| 443  | HTTPS (TLS 1.2/1.3) | FW-Internet-A / FW-Internet-B Active/Active with VIP |

**Description:**

- This is the only publicly exposed endpoint.  
- All traffic is encrypted using TLS 1.2/1.3.  
- Firewall redundancy ensures high availability and failover without service disruption.
- Firewall Redundancy Strategy
- 
**Design**

- Each zone boundary (Internet→DMZ, DMZ→Application, Application→Data, Data→Monitoring) is protected by a firewall cluster.

- The cluster consists of two firewalls (Active/Passive or Active/Active).

- A Virtual IP (VIP) abstracts the firewalls, ensuring seamless failover.

- If Firewall-1 fails, Firewall-2 immediately takes over, maintaining uninterrupted traffic flow.

**Firewall Configuration:**

| Device           | Physical IP     | VIP           | Role              |
|-----------------|----------------|---------------|-----------------|
| FW-Internet-A    | 200.168.0.21   | 200.168.0.20  | Active/Active VIP |
| FW-Internet-B    | 200.168.0.22   | 200.168.0.20  | Active/Active VIP |

**Traffic Flow:**

1. External Client connects to `consultclients.company.sant` via VIP `200.168.0.20` using HTTPS on port 443.  
2. Traffic passes through the **Internet Border Firewall VIP** (Active/Active).  
3. Traffic is routed to the **DMZ Zone**, reaching the **Nginx VIP** (`10.0.0.100:443`).  
4. Nginx servers terminate TLS/SSL and forward requests via HTTP (`8080`) to the **API VIP** (`10.0.0.120:8080`) in the Application Zone.  
5. The **API servers** process requests and interact with:
   - **Redis VIP** (`10.0.0.130:6379`) for caching.
   - **PostgreSQL VIP** (`10.0.0.150:5432`) for persistent storage.
   - **Kafka VIP** (`10.0.0.140:9092`) for event streaming.  
6. **Prometheus** scrapes metrics; **Grafana** visualizes dashboards via Monitoring VIP (`10.0.0.160`).

---

## 3. Internal Access Flow (Intranet)

1. Internal users bypass the public IP and connect directly to **API VIP** (`10.0.0.120:8080`).  
2. Requests are processed by API servers without traversing the DMZ.  
3. APIs interact with Redis, PostgreSQL, and Kafka via their VIPs.  
4. Monitoring remains identical, with Prometheus collecting metrics and Grafana providing visualization.  

**Traffic Flow:**  
Internal Client → API VIP → Data VIPs  

**Note:** Firewall redundancy is not required for internal traffic.

---

## 4. DMZ Zone – Nginx Reverse Proxy Cluster

| Component      | VIP        | Physical Nodes       | Port | Middleware        |
|----------------|------------|--------------------|------|-----------------|
| Nginx Cluster  | 10.0.0.100 | 10.0.0.10 / 10.0.0.11 | 443 | Nginx + OpenSSL |

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

## 5. Application Zone – Tomcat API Cluster

| Component     | VIP        | Physical Nodes       | Port | Middleware                  |
|---------------|------------|--------------------|------|-----------------------------|
| API Cluster   | 10.0.0.120 | 10.0.0.20 / 10.0.0.21 | 8080 | Tomcat + JVM + Spring Boot |

**Middleware Stack per Node:**

- Linux Enterprise OS  
- OpenJDK 17 LTS  
- Apache Tomcat 9.x / 10.x  
- Spring Boot + Spring Security (RBAC)  
- PostgreSQL JDBC + HikariCP  
- Redis (Lettuce/Jedis)  
- Kafka Java Client  
- Logback / Log4j2  
- Micrometer + Prometheus endpoint  

**Architecture:** Fully stateless for horizontal scaling.

---

## 6. Data Zone

### 6.1 Redis Cluster

| Component      | VIP        | Physical Nodes       | Port | Middleware |
|----------------|------------|--------------------|------|------------|
| Redis Cluster  | 10.0.0.130 | 10.0.0.30 / 10.0.0.31 | 6379 | Redis 6+ |

### 6.2 Kafka Cluster

| Component      | VIP        | Physical Nodes       | Port | Middleware        |
|----------------|------------|--------------------|------|-----------------|
| Kafka Cluster  | 10.0.0.140 | 10.0.0.40 / 10.0.0.41 | 9092 | Apache Kafka 3.x |

### 6.3 PostgreSQL Cluster

| Component          | VIP        | Physical Nodes       | Port | Middleware       |
|--------------------|------------|--------------------|------|-----------------|
| PostgreSQL Cluster | 10.0.0.150 | 10.0.0.50 / 10.0.0.51 | 5432 | PostgreSQL 14+ |

**Replication:** Streaming replication enabled for high availability.

---

## 7. Monitoring Zone

| Component   | VIP        | Physical Nodes       | Port       | Middleware            |
|-------------|------------|--------------------|------------|---------------------|
| Monitoring  | 10.0.0.160 | 10.0.0.60 / 10.0.0.61 | 9090 / 3000 | Prometheus + Grafana |

---

## 8. High Availability & Firewall Strategy

- **Internet Border Firewall:** Active/Active with VIP `200.168.0.20`.  
- VIP abstraction for all clusters (API, DMZ, Data, Monitoring).  
- Redundant nodes in every zone.  
- No single point of failure.  
- Database, Redis, Kafka replication.  
- Monitoring for proactive failure detection.

**Targets:**  
- RTO < 30 minutes  
- RPO < 5 minutes

---

## 9. Security Strategy

- TLS 1.2 / 1.3 encryption externally  
- **Firewall redundancy at Internet border only**  
- Internal-only exposure for Data Zone  
- RBAC enforced at API layer  
- LGPD-aligned data handling  
- Logging and monitoring enabled  

---

## 10. Final Conclusion

The ConsultClients architecture provides:

- Internet border firewall redundancy with VIP `200.168.0.20`  
- Clear VIP-to-physical-node mapping  
- Multi-layer high availability  
- Secure segmentation across five zones  
- Enterprise-grade middleware resilience  
- Observability and operational transparency  
- Compliance with LGPD and internal governance  

Fully prepared for mission-critical production environments requiring **high reliability, scalability, and secure data handling**.
