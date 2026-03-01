# ConsultClients API – Network Flow Diagram with External and Internal Flow
# Version: 1.2
# Owner: Flavio D. Lizzardo

```mermaid

flowchart LR
%% ConsultClients API – External + Internal Flow Enhanced
%% Color classes for zones
classDef internet fill:#cce5ff,stroke:#3399ff,stroke-width:2px;
classDef firewall fill:#ffcccc,stroke:#ff0000,stroke-width:2px;
classDef dmz fill:#fff2cc,stroke:#ffaa00,stroke-width:2px;
classDef app fill:#d5f5e3,stroke:#27ae60,stroke-width:2px;
classDef data fill:#f5e6ff,stroke:#8e44ad,stroke-width:2px;
classDef monitoring fill:#ffe6f0,stroke:#ff33a6,stroke-width:2px;
classDef internal fill:#d1ecf1,stroke:#17a2b8,stroke-width:2px,stroke-dasharray: 5 5;
classDef flowExternal stroke:#0056b3,stroke-width:2px;
classDef flowInternal stroke:#17a2b8,stroke-width:2px,stroke-dasharray: 5 5;

%% Internet Zone
subgraph INTERNET_ZONE["Internet Zone"]
    A1["1. External Client\nconsultclients.company.sant\n200.168.0.20:443\nHTTPS"]:::internet
    subgraph FIREWALL_BORDER["Internet Border Firewalls"]
        B1["2. FW-Internet-A\n200.168.0.21\nActive/Active VIP 200.168.0.20"]:::firewall
        B2["3. FW-Internet-B\n200.168.0.22\nActive/Active VIP 200.168.0.20"]:::firewall
        B1 <--> B2
    end
end

%% DMZ Zone
subgraph DMZ_ZONE["DMZ Zone"]
    C1["4. Nginx VIP\n10.0.0.100:443"]:::dmz
    C2["5. nginx-1\n10.0.0.10:443"]:::dmz
    C3["6. nginx-2\n10.0.0.11:443"]:::dmz
end

%% Application Zone
subgraph APP_ZONE["Application Zone"]
    D1["7. API VIP\n10.0.0.120:8080"]:::app
    D2["8. api-1\n10.0.0.20:8080"]:::app
    D3["9. api-2\n10.0.0.21:8080"]:::app
end

%% Data Zone
subgraph DATA_ZONE["Data Zone"]
    subgraph REDIS["Redis Cluster"]
        E1["10. Redis VIP\n10.0.0.130:6379"]:::data
        E2["11. redis-1\n10.0.0.30:6379"]:::data
        E3["12. redis-2\n10.0.0.31:6379"]:::data
    end

    subgraph KAFKA["Kafka Cluster"]
        F1["13. Kafka VIP\n10.0.0.140:9092"]:::data
        F2["14. kafka-1\n10.0.0.40:9092"]:::data
        F3["15. kafka-2\n10.0.0.41:9092"]:::data
    end

    subgraph POSTGRES["PostgreSQL Cluster"]
        G1["16. Postgres VIP\n10.0.0.150:5432"]:::data
        G2["17. postgres-primary\n10.0.0.50:5432\nDatabases: db_user, db_clients"]:::data
        G3["18. postgres-replica\n10.0.0.51:5432\nStreaming Replication"]:::data
    end
end

%% Monitoring Zone with redundancy
subgraph MONITORING_ZONE["Monitoring Zone"]
    H1["19. Monitor VIP\n10.0.0.160:9090/3000\nActive/Active"]:::monitoring
    H2["20. Prometheus\n10.0.0.60"]:::monitoring
    H3["21. Prometheus-2\n10.0.0.61"]:::monitoring
    H4["22. Grafana\n10.0.0.61"]:::monitoring
    H5["23. Grafana-2\n10.0.0.62"]:::monitoring
end

%% Internal Corporate Network
subgraph INTERNAL_NETWORK["Corporate Internal Network"]
    I1["24. Internal User"]:::internal
end

%% External Flow (blue arrows)
A1 -. flowExternal .-> B1
A1 -. flowExternal .-> B2
B1 -. flowExternal .-> C1
B2 -. flowExternal .-> C1
C1 -. flowExternal .-> D1

%% Internal Flow (cyan dashed arrows)
I1 -. flowInternal .-> D1

%% API to Nodes
D1 --> D2
D1 --> D3

%% API to Data Zone
D1 --> E1
D1 --> F1
D1 --> G1

E1 --> E2
E1 --> E3

F1 --> F2
F1 --> F3

G1 --> G2
G1 --> G3

%% Monitoring connections via VIP
H1 --> H2
H1 --> H3
H1 --> H4
H1 --> H5
H2 --> D2
H3 --> D3
H4 --> D2
H5 --> D3

%% Legend
subgraph LEGEND["Legend"]
    LE1["Blue arrows: External Flow"]:::flowExternal
    LE2["Cyan dashed arrows: Internal Flow"]:::flowInternal
end
