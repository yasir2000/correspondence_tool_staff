# Deployment Architecture

## Infrastructure Overview

```mermaid
graph TB
    subgraph "External Services"
        INTERNET[Internet]
        CDN[CloudFlare CDN]
        DNS[Route 53 DNS]
    end
    
    subgraph "Load Balancing"
        ALB[Application Load Balancer]
        NLB[Network Load Balancer]
    end
    
    subgraph "Government Cloud (AWS)"
        subgraph "Public Subnet"
            NAT[NAT Gateway]
            BASTION[Bastion Host]
        end
        
        subgraph "Private Subnet - Web Tier"
            WEB1[Web Server 1]
            WEB2[Web Server 2]
            WEB3[Web Server 3]
        end
        
        subgraph "Private Subnet - Application Tier"
            APP1[App Server 1]
            APP2[App Server 2]
            WORKER1[Background Worker 1]
            WORKER2[Background Worker 2]
        end
        
        subgraph "Private Subnet - Data Tier"
            RDS_PRIMARY[RDS PostgreSQL Primary]
            RDS_REPLICA[RDS PostgreSQL Replica]
            REDIS[ElastiCache Redis]
            S3[S3 Document Storage]
        end
    end
    
    subgraph "Monitoring & Logging"
        CLOUDWATCH[CloudWatch]
        SENTRY[Sentry.io]
        SPLUNK[Splunk]
    end
    
    INTERNET --> CDN
    CDN --> DNS
    DNS --> ALB
    ALB --> WEB1
    ALB --> WEB2
    ALB --> WEB3
    
    WEB1 --> APP1
    WEB2 --> APP2
    WEB3 --> APP1
    
    APP1 --> RDS_PRIMARY
    APP2 --> RDS_PRIMARY
    APP1 --> RDS_REPLICA
    APP2 --> RDS_REPLICA
    
    APP1 --> REDIS
    APP2 --> REDIS
    WORKER1 --> REDIS
    WORKER2 --> REDIS
    
    APP1 --> S3
    APP2 --> S3
    
    WEB1 --> CLOUDWATCH
    APP1 --> SENTRY
    WORKER1 --> SPLUNK
```

## Container Architecture (Kubernetes)

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "Ingress"
            INGRESS[Nginx Ingress Controller]
            TLS[TLS Termination]
        end
        
        subgraph "Application Namespace"
            subgraph "Web Pods"
                WEB_POD1[Web Pod 1<br/>Rails + Puma]
                WEB_POD2[Web Pod 2<br/>Rails + Puma]
                WEB_POD3[Web Pod 3<br/>Rails + Puma]
            end
            
            subgraph "Worker Pods"
                WORKER_POD1[Worker Pod 1<br/>Sidekiq]
                WORKER_POD2[Worker Pod 2<br/>Sidekiq]
            end
            
            subgraph "Services"
                WEB_SVC[Web Service]
                WORKER_SVC[Worker Service]
            end
        end
        
        subgraph "Database Namespace"
            POSTGRES[PostgreSQL StatefulSet]
            REDIS_CLUSTER[Redis Cluster]
        end
        
        subgraph "Storage"
            PV1[Persistent Volume 1]
            PV2[Persistent Volume 2]
            SECRET_STORE[Secret Store]
        end
        
        subgraph "Monitoring Namespace"
            PROMETHEUS[Prometheus]
            GRAFANA[Grafana]
            ALERTMANAGER[AlertManager]
        end
    end
    
    INGRESS --> WEB_SVC
    WEB_SVC --> WEB_POD1
    WEB_SVC --> WEB_POD2
    WEB_SVC --> WEB_POD3
    
    WEB_POD1 --> POSTGRES
    WEB_POD1 --> REDIS_CLUSTER
    
    WORKER_SVC --> WORKER_POD1
    WORKER_SVC --> WORKER_POD2
    
    POSTGRES --> PV1
    REDIS_CLUSTER --> PV2
    
    WEB_POD1 --> SECRET_STORE
    WORKER_POD1 --> SECRET_STORE
```

## Environment Architecture

### Development Environment

```mermaid
graph TB
    subgraph "Developer Machine"
        DOCKER[Docker Desktop]
        IDE[VS Code / RubyMine]
        TERMINAL[Terminal]
    end
    
    subgraph "Docker Compose Stack"
        RAILS_DEV[Rails App<br/>Port 3000]
        POSTGRES_DEV[PostgreSQL<br/>Port 5432]
        REDIS_DEV[Redis<br/>Port 6379]
        MAILCATCHER[MailCatcher<br/>Port 1080]
    end
    
    subgraph "External Services (Stubs)"
        NOTIFY_STUB[Gov.UK Notify Stub]
        S3_STUB[LocalStack S3]
        SSCL_STUB[SSCL Mock Server]
    end
    
    IDE --> DOCKER
    DOCKER --> RAILS_DEV
    RAILS_DEV --> POSTGRES_DEV
    RAILS_DEV --> REDIS_DEV
    RAILS_DEV --> MAILCATCHER
    RAILS_DEV --> NOTIFY_STUB
    RAILS_DEV --> S3_STUB
    RAILS_DEV --> SSCL_STUB
```

### Staging Environment

```mermaid
graph TB
    subgraph "GOV.UK PaaS - Staging"
        subgraph "Applications"
            STAGING_APP[Correspondence Tool Staging]
            STAGING_WORKER[Background Worker]
        end
        
        subgraph "Services"
            STAGING_DB[(PostgreSQL Service)]
            STAGING_REDIS[(Redis Service)]
            STAGING_S3[(S3 Service)]
        end
        
        subgraph "Routes"
            STAGING_URL[staging.correspondence-tool.service.gov.uk]
        end
    end
    
    subgraph "External Services"
        NOTIFY_TEST[Gov.UK Notify Test]
        SSCL_TEST[SSCL Test Environment]
    end
    
    STAGING_URL --> STAGING_APP
    STAGING_APP --> STAGING_DB
    STAGING_APP --> STAGING_REDIS
    STAGING_APP --> STAGING_S3
    STAGING_WORKER --> STAGING_REDIS
    
    STAGING_APP --> NOTIFY_TEST
    STAGING_APP --> SSCL_TEST
```

### Production Environment

```mermaid
graph TB
    subgraph "Production Infrastructure"
        subgraph "Load Balancer"
            PROD_LB[Production Load Balancer]
            SSL[SSL/TLS Termination]
        end
        
        subgraph "Application Tier"
            PROD_APP1[App Instance 1]
            PROD_APP2[App Instance 2]
            PROD_APP3[App Instance 3]
            PROD_WORKER1[Worker Instance 1]
            PROD_WORKER2[Worker Instance 2]
        end
        
        subgraph "Database Tier"
            PROD_DB_PRIMARY[(Primary Database)]
            PROD_DB_REPLICA[(Read Replica)]
            PROD_REDIS[(Redis Cluster)]
        end
        
        subgraph "Storage"
            PROD_S3[S3 Document Storage]
            BACKUP_S3[S3 Backup Storage]
        end
        
        subgraph "Monitoring"
            PROD_LOGS[Application Logs]
            PROD_METRICS[Performance Metrics]
            PROD_ALERTS[Alert System]
        end
    end
    
    PROD_LB --> PROD_APP1
    PROD_LB --> PROD_APP2
    PROD_LB --> PROD_APP3
    
    PROD_APP1 --> PROD_DB_PRIMARY
    PROD_APP2 --> PROD_DB_REPLICA
    PROD_APP3 --> PROD_DB_PRIMARY
    
    PROD_WORKER1 --> PROD_REDIS
    PROD_WORKER2 --> PROD_REDIS
    
    PROD_APP1 --> PROD_S3
    PROD_DB_PRIMARY --> BACKUP_S3
    
    PROD_APP1 --> PROD_LOGS
    PROD_WORKER1 --> PROD_METRICS
    PROD_METRICS --> PROD_ALERTS
```

## CI/CD Pipeline

```mermaid
graph LR
    subgraph "Source Control"
        GIT[Git Repository]
        PR[Pull Request]
        MAIN[Main Branch]
    end
    
    subgraph "CI Pipeline"
        TRIGGER[Pipeline Trigger]
        TESTS[Run Tests]
        LINT[Code Linting]
        SECURITY[Security Scan]
        BUILD[Build Image]
    end
    
    subgraph "CD Pipeline"
        STAGING_DEPLOY[Deploy to Staging]
        STAGING_TEST[Staging Tests]
        PROD_DEPLOY[Deploy to Production]
        SMOKE_TEST[Smoke Tests]
    end
    
    subgraph "Deployment Targets"
        STAGING_ENV[Staging Environment]
        PROD_ENV[Production Environment]
    end
    
    GIT --> PR
    PR --> TRIGGER
    TRIGGER --> TESTS
    TESTS --> LINT
    LINT --> SECURITY
    SECURITY --> BUILD
    BUILD --> STAGING_DEPLOY
    
    STAGING_DEPLOY --> STAGING_ENV
    STAGING_ENV --> STAGING_TEST
    STAGING_TEST --> PROD_DEPLOY
    PROD_DEPLOY --> PROD_ENV
    PROD_ENV --> SMOKE_TEST
    
    MAIN --> TRIGGER
```

## Security Architecture

### Network Security

```mermaid
graph TB
    subgraph "Internet"
        USERS[Government Users]
        THREATS[External Threats]
    end
    
    subgraph "Security Perimeter"
        WAF[Web Application Firewall]
        DDOS[DDoS Protection]
        FIREWALL[Network Firewall]
    end
    
    subgraph "DMZ"
        PROXY[Reverse Proxy]
        IDS[Intrusion Detection]
    end
    
    subgraph "Private Network"
        subgraph "Application Layer"
            APP_SERVERS[Application Servers]
            WORKER_SERVERS[Worker Servers]
        end
        
        subgraph "Data Layer"
            DATABASE[Encrypted Database]
            CACHE[Encrypted Cache]
            FILES[Encrypted File Storage]
        end
    end
    
    subgraph "Monitoring"
        SIEM[Security Information<br/>Event Management]
        LOG_ANALYSIS[Log Analysis]
        THREAT_DETECTION[Threat Detection]
    end
    
    USERS --> WAF
    THREATS --> WAF
    WAF --> DDOS
    DDOS --> FIREWALL
    FIREWALL --> PROXY
    PROXY --> IDS
    IDS --> APP_SERVERS
    
    APP_SERVERS --> DATABASE
    APP_SERVERS --> CACHE
    APP_SERVERS --> FILES
    
    WORKER_SERVERS --> DATABASE
    WORKER_SERVERS --> CACHE
    
    APP_SERVERS --> SIEM
    DATABASE --> LOG_ANALYSIS
    IDS --> THREAT_DETECTION
```

### Data Protection

```mermaid
graph TB
    subgraph "Data Classification"
        OFFICIAL[OFFICIAL]
        OFFICIAL_SENSITIVE[OFFICIAL-SENSITIVE]
        SECRET[SECRET (if required)]
    end
    
    subgraph "Encryption"
        TLS[TLS 1.3 in Transit]
        AES[AES-256 at Rest]
        KEY_MANAGEMENT[AWS KMS Key Management]
    end
    
    subgraph "Access Control"
        IAM[Identity & Access Management]
        MFA[Multi-Factor Authentication]
        RBAC[Role-Based Access Control]
        AUDIT[Access Audit Logs]
    end
    
    subgraph "Data Handling"
        BACKUP[Encrypted Backups]
        RETENTION[Data Retention Policies]
        DISPOSAL[Secure Data Disposal]
        GDPR[GDPR Compliance]
    end
    
    OFFICIAL --> TLS
    OFFICIAL_SENSITIVE --> AES
    SECRET --> KEY_MANAGEMENT
    
    IAM --> MFA
    MFA --> RBAC
    RBAC --> AUDIT
    
    AES --> BACKUP
    BACKUP --> RETENTION
    RETENTION --> DISPOSAL
    DISPOSAL --> GDPR
```

## Monitoring and Observability

### Application Monitoring

```mermaid
graph TB
    subgraph "Application Layer"
        RAILS_APP[Rails Application]
        SIDEKIQ[Sidekiq Workers]
        NGINX[Nginx Web Server]
    end
    
    subgraph "Metrics Collection"
        APP_METRICS[Application Metrics]
        SYSTEM_METRICS[System Metrics]
        BUSINESS_METRICS[Business Metrics]
    end
    
    subgraph "Monitoring Stack"
        PROMETHEUS[Prometheus]
        GRAFANA[Grafana Dashboards]
        ALERTMANAGER[Alert Manager]
    end
    
    subgraph "Logging"
        APP_LOGS[Application Logs]
        ACCESS_LOGS[Access Logs]
        ERROR_LOGS[Error Logs]
        AUDIT_LOGS[Audit Logs]
    end
    
    subgraph "Alerting"
        SLACK[Slack Notifications]
        EMAIL[Email Alerts]
        PAGERDUTY[PagerDuty]
    end
    
    RAILS_APP --> APP_METRICS
    SIDEKIQ --> SYSTEM_METRICS
    NGINX --> BUSINESS_METRICS
    
    APP_METRICS --> PROMETHEUS
    SYSTEM_METRICS --> PROMETHEUS
    BUSINESS_METRICS --> PROMETHEUS
    
    PROMETHEUS --> GRAFANA
    PROMETHEUS --> ALERTMANAGER
    
    RAILS_APP --> APP_LOGS
    NGINX --> ACCESS_LOGS
    RAILS_APP --> ERROR_LOGS
    RAILS_APP --> AUDIT_LOGS
    
    ALERTMANAGER --> SLACK
    ALERTMANAGER --> EMAIL
    ALERTMANAGER --> PAGERDUTY
```

### Performance Monitoring

```mermaid
graph TB
    subgraph "Frontend Monitoring"
        PAGE_LOAD[Page Load Times]
        USER_INTERACTION[User Interactions]
        ERROR_TRACKING[Frontend Errors]
    end
    
    subgraph "Backend Monitoring"
        RESPONSE_TIME[API Response Times]
        THROUGHPUT[Request Throughput]
        ERROR_RATE[Error Rates]
        QUEUE_DEPTH[Job Queue Depth]
    end
    
    subgraph "Infrastructure Monitoring"
        CPU_USAGE[CPU Usage]
        MEMORY_USAGE[Memory Usage]
        DISK_IO[Disk I/O]
        NETWORK_IO[Network I/O]
    end
    
    subgraph "Database Monitoring"
        QUERY_PERFORMANCE[Query Performance]
        CONNECTION_POOL[Connection Pool]
        REPLICATION_LAG[Replication Lag]
        DEADLOCKS[Deadlock Detection]
    end
    
    subgraph "Alerting Thresholds"
        CRITICAL[Critical Alerts<br/>> 95th percentile]
        WARNING[Warning Alerts<br/>> 90th percentile]
        INFO[Info Alerts<br/>> 80th percentile]
    end
    
    PAGE_LOAD --> CRITICAL
    RESPONSE_TIME --> WARNING
    CPU_USAGE --> INFO
    QUERY_PERFORMANCE --> CRITICAL
```

## Disaster Recovery

### Backup Strategy

```mermaid
graph TB
    subgraph "Production Data"
        PROD_DB[(Production Database)]
        PROD_FILES[Production Files]
        PROD_CONFIG[Configuration]
    end
    
    subgraph "Backup Types"
        FULL_BACKUP[Full Backup<br/>Weekly]
        INCREMENTAL[Incremental Backup<br/>Daily]
        TRANSACTION_LOG[Transaction Log<br/>Every 15 minutes]
    end
    
    subgraph "Backup Storage"
        LOCAL_BACKUP[Local Backup Storage]
        OFFSITE_BACKUP[Offsite Backup Storage]
        ARCHIVE[Long-term Archive]
    end
    
    subgraph "Recovery Options"
        POINT_IN_TIME[Point-in-Time Recovery]
        FULL_RESTORE[Full System Restore]
        PARTIAL_RESTORE[Partial Data Restore]
    end
    
    PROD_DB --> FULL_BACKUP
    PROD_DB --> INCREMENTAL
    PROD_DB --> TRANSACTION_LOG
    
    PROD_FILES --> FULL_BACKUP
    PROD_CONFIG --> FULL_BACKUP
    
    FULL_BACKUP --> LOCAL_BACKUP
    INCREMENTAL --> LOCAL_BACKUP
    TRANSACTION_LOG --> LOCAL_BACKUP
    
    LOCAL_BACKUP --> OFFSITE_BACKUP
    OFFSITE_BACKUP --> ARCHIVE
    
    LOCAL_BACKUP --> POINT_IN_TIME
    OFFSITE_BACKUP --> FULL_RESTORE
    LOCAL_BACKUP --> PARTIAL_RESTORE
```

### Failover Architecture

```mermaid
graph TB
    subgraph "Primary Site"
        PRIMARY_LB[Primary Load Balancer]
        PRIMARY_APP[Primary Application Servers]
        PRIMARY_DB[(Primary Database)]
        PRIMARY_STORAGE[Primary File Storage]
    end
    
    subgraph "Secondary Site"
        SECONDARY_LB[Secondary Load Balancer]
        SECONDARY_APP[Secondary Application Servers]
        SECONDARY_DB[(Secondary Database)]
        SECONDARY_STORAGE[Secondary File Storage]
    end
    
    subgraph "Failover Control"
        HEALTH_CHECK[Health Monitoring]
        FAILOVER_TRIGGER[Automatic Failover]
        DNS_SWITCH[DNS Switching]
        MANUAL_OVERRIDE[Manual Override]
    end
    
    PRIMARY_APP --> PRIMARY_DB
    PRIMARY_APP --> PRIMARY_STORAGE
    
    SECONDARY_APP --> SECONDARY_DB
    SECONDARY_APP --> SECONDARY_STORAGE
    
    PRIMARY_DB -.->|Replication| SECONDARY_DB
    PRIMARY_STORAGE -.->|Sync| SECONDARY_STORAGE
    
    HEALTH_CHECK --> PRIMARY_LB
    HEALTH_CHECK --> FAILOVER_TRIGGER
    FAILOVER_TRIGGER --> DNS_SWITCH
    DNS_SWITCH --> SECONDARY_LB
    MANUAL_OVERRIDE --> DNS_SWITCH
```

## Scalability Architecture

### Horizontal Scaling

```mermaid
graph TB
    subgraph "Load Distribution"
        LOAD_BALANCER[Application Load Balancer]
        AUTO_SCALING[Auto Scaling Group]
    end
    
    subgraph "Application Instances"
        APP1[App Instance 1]
        APP2[App Instance 2]
        APP3[App Instance 3]
        APP4[App Instance 4 - On Demand]
        APP5[App Instance 5 - On Demand]
    end
    
    subgraph "Worker Scaling"
        WORKER_POOL[Worker Pool]
        QUEUE_MONITOR[Queue Depth Monitor]
        WORKER_SCALING[Worker Auto Scaling]
    end
    
    subgraph "Database Scaling"
        WRITE_DB[(Write Database)]
        READ_DB1[(Read Replica 1)]
        READ_DB2[(Read Replica 2)]
        CONNECTION_POOL[Connection Pool]
    end
    
    LOAD_BALANCER --> APP1
    LOAD_BALANCER --> APP2
    LOAD_BALANCER --> APP3
    AUTO_SCALING --> APP4
    AUTO_SCALING --> APP5
    
    QUEUE_MONITOR --> WORKER_SCALING
    WORKER_SCALING --> WORKER_POOL
    
    APP1 --> WRITE_DB
    APP2 --> READ_DB1
    APP3 --> READ_DB2
    
    CONNECTION_POOL --> WRITE_DB
    CONNECTION_POOL --> READ_DB1
    CONNECTION_POOL --> READ_DB2
```

### Performance Optimization

```mermaid
graph TB
    subgraph "Caching Strategy"
        CDN[Content Delivery Network]
        APP_CACHE[Application Cache]
        DB_CACHE[Database Query Cache]
        SESSION_CACHE[Session Cache]
    end
    
    subgraph "Database Optimization"
        INDEX_OPTIMIZATION[Index Optimization]
        QUERY_OPTIMIZATION[Query Optimization]
        PARTITIONING[Table Partitioning]
        ARCHIVING[Data Archiving]
    end
    
    subgraph "Application Optimization"
        CODE_OPTIMIZATION[Code Optimization]
        ASYNC_PROCESSING[Async Processing]
        BACKGROUND_JOBS[Background Jobs]
        API_OPTIMIZATION[API Optimization]
    end
    
    CDN --> APP_CACHE
    APP_CACHE --> DB_CACHE
    DB_CACHE --> SESSION_CACHE
    
    INDEX_OPTIMIZATION --> QUERY_OPTIMIZATION
    QUERY_OPTIMIZATION --> PARTITIONING
    PARTITIONING --> ARCHIVING
    
    CODE_OPTIMIZATION --> ASYNC_PROCESSING
    ASYNC_PROCESSING --> BACKGROUND_JOBS
    BACKGROUND_JOBS --> API_OPTIMIZATION
```
