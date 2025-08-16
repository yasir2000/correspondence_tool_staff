# Complete System Architecture

This document provides a comprehensive visual overview of the entire Correspondence Tool Staff system architecture.

## Master Architecture Diagram

```mermaid
graph TB
    subgraph "External Layer"
        USERS[Government Staff Users]
        EXTERNAL_APIS[External APIs<br/>- GOV.UK Notify<br/>- SSCL<br/>- Active Directory]
        THIRD_PARTY[Third Party Integrations<br/>- DPS<br/>- Probation Services]
    end
    
    subgraph "Security & Load Balancing"
        WAF[Web Application Firewall]
        LB[Load Balancer]
        SSL[SSL/TLS Termination]
    end
    
    subgraph "Application Infrastructure"
        subgraph "Web Tier"
            WEB1[Web Server 1<br/>Nginx + Rails]
            WEB2[Web Server 2<br/>Nginx + Rails]
            WEB3[Web Server 3<br/>Nginx + Rails]
        end
        
        subgraph "Application Tier"
            APP1[App Instance 1<br/>Rails Application]
            APP2[App Instance 2<br/>Rails Application]
            WORKER1[Background Worker 1<br/>Sidekiq]
            WORKER2[Background Worker 2<br/>Sidekiq]
        end
        
        subgraph "Service Layer"
            CASE_SVC[Case Services]
            WORKFLOW_SVC[Workflow Services]
            NOTIFY_SVC[Notification Services]
            SEARCH_SVC[Search Services]
        end
    end
    
    subgraph "Data Layer"
        subgraph "Primary Storage"
            DB_PRIMARY[(PostgreSQL Primary)]
            DB_REPLICA[(PostgreSQL Replica)]
        end
        
        subgraph "Cache & Queue"
            REDIS[Redis Cache/Queue]
            MEMCACHED[Memcached]
        end
        
        subgraph "File Storage"
            S3[AWS S3<br/>Document Storage]
            LOCAL_FILES[Local File Storage]
        end
    end
    
    subgraph "Monitoring & Logging"
        PROMETHEUS[Prometheus Metrics]
        GRAFANA[Grafana Dashboards]
        SENTRY[Sentry Error Tracking]
        LOGS[Centralized Logging]
        ALERTS[Alert Manager]
    end
    
    subgraph "Backup & DR"
        BACKUP_DB[(Backup Database)]
        BACKUP_FILES[Backup File Storage]
        DR_SITE[Disaster Recovery Site]
    end
    
    %% User Flow
    USERS --> WAF
    WAF --> LB
    LB --> SSL
    SSL --> WEB1
    SSL --> WEB2
    SSL --> WEB3
    
    %% Web to App Tier
    WEB1 --> APP1
    WEB2 --> APP2
    WEB3 --> APP1
    
    %% App to Services
    APP1 --> CASE_SVC
    APP2 --> WORKFLOW_SVC
    APP1 --> NOTIFY_SVC
    APP2 --> SEARCH_SVC
    
    %% Services to Data
    CASE_SVC --> DB_PRIMARY
    WORKFLOW_SVC --> DB_PRIMARY
    SEARCH_SVC --> DB_REPLICA
    NOTIFY_SVC --> REDIS
    
    %% Background Jobs
    REDIS --> WORKER1
    REDIS --> WORKER2
    WORKER1 --> DB_PRIMARY
    WORKER2 --> EXTERNAL_APIS
    
    %% File Storage
    APP1 --> S3
    APP2 --> S3
    APP1 --> LOCAL_FILES
    
    %% Caching
    APP1 --> MEMCACHED
    APP2 --> MEMCACHED
    SEARCH_SVC --> REDIS
    
    %% External Integrations
    NOTIFY_SVC --> EXTERNAL_APIS
    WORKER2 --> THIRD_PARTY
    
    %% Monitoring
    APP1 --> PROMETHEUS
    APP2 --> PROMETHEUS
    WORKER1 --> SENTRY
    WEB1 --> LOGS
    PROMETHEUS --> GRAFANA
    PROMETHEUS --> ALERTS
    
    %% Backup & DR
    DB_PRIMARY -.-> BACKUP_DB
    S3 -.-> BACKUP_FILES
    BACKUP_DB -.-> DR_SITE
    BACKUP_FILES -.-> DR_SITE
    
    %% Database Replication
    DB_PRIMARY -.-> DB_REPLICA
```

## Application Component Architecture

```mermaid
graph TB
    subgraph "Presentation Layer"
        VIEWS[ERB Templates]
        HELPERS[View Helpers]
        ASSETS[SCSS/JS Assets]
        PARTIALS[Partial Templates]
    end
    
    subgraph "Controller Layer"
        APPLICATION_CTRL[ApplicationController]
        CASES_CTRL[CasesController]
        ASSIGNMENTS_CTRL[AssignmentsController]
        ADMIN_CTRL[AdminController]
        API_CTRL[API Controllers]
    end
    
    subgraph "Authorization Layer"
        PUNDIT[Pundit Policies]
        CASE_POLICIES[Case Policies]
        USER_POLICIES[User Policies]
        TEAM_POLICIES[Team Policies]
    end
    
    subgraph "Service Layer"
        subgraph "Core Services"
            CASE_CREATE[CaseCreateService]
            CASE_ASSIGN[CaseAssignService]
            CASE_APPROVAL[CaseApprovalService]
            CASE_CLOSURE[CaseClosureService]
        end
        
        subgraph "Supporting Services"
            NOTIFICATION[NotificationService]
            SEARCH[SearchService]
            FILTER[FilterService]
            EXPORT[ExportService]
        end
    end
    
    subgraph "Model Layer"
        subgraph "Case Models"
            CASE_BASE[Case::Base]
            FOI_CASES[FOI Cases]
            SAR_CASES[SAR Cases]
            ICO_CASES[ICO Cases]
        end
        
        subgraph "User & Team Models"
            USER[User]
            TEAM[Team]
            ASSIGNMENT[Assignment]
            ROLES[TeamsUsersRole]
        end
        
        subgraph "Supporting Models"
            ATTACHMENT[CaseAttachment]
            TRANSITION[CaseTransition]
            CONTACT[Contact]
            RETENTION[RetentionSchedule]
        end
    end
    
    subgraph "Infrastructure Layer"
        STATE_MACHINES[State Machines]
        BACKGROUND_JOBS[Background Jobs]
        MAILERS[Email Mailers]
        VALIDATORS[Custom Validators]
    end
    
    %% Flow connections
    VIEWS --> CONTROLLER_LAYER
    CONTROLLER_LAYER --> AUTHORIZATION_LAYER
    AUTHORIZATION_LAYER --> SERVICE_LAYER
    SERVICE_LAYER --> MODEL_LAYER
    MODEL_LAYER --> STATE_MACHINES
    SERVICE_LAYER --> BACKGROUND_JOBS
    BACKGROUND_JOBS --> MAILERS
```

## Data Flow Architecture

```mermaid
flowchart TD
    START([User Action]) --> AUTH{Authenticated?}
    AUTH -->|No| LOGIN[Redirect to Login]
    AUTH -->|Yes| AUTHORIZE{Authorized?}
    
    AUTHORIZE -->|No| FORBIDDEN[403 Forbidden]
    AUTHORIZE -->|Yes| CONTROLLER[Controller Action]
    
    CONTROLLER --> VALIDATE{Valid Request?}
    VALIDATE -->|No| ERROR_RESPONSE[Error Response]
    VALIDATE -->|Yes| SERVICE[Service Layer]
    
    SERVICE --> BUSINESS_LOGIC{Business Rules}
    BUSINESS_LOGIC -->|Invalid| BUSINESS_ERROR[Business Logic Error]
    BUSINESS_LOGIC -->|Valid| MODEL[Model Operations]
    
    MODEL --> DATABASE[(Database)]
    MODEL --> STATE_MACHINE[State Machine]
    MODEL --> AUDIT[Audit Log]
    
    STATE_MACHINE --> NOTIFICATIONS[Queue Notifications]
    NOTIFICATIONS --> BACKGROUND_JOB[Background Job]
    BACKGROUND_JOB --> EMAIL[Send Email]
    BACKGROUND_JOB --> EXTERNAL_API[External API Call]
    
    MODEL --> CACHE[Update Cache]
    MODEL --> SEARCH_INDEX[Update Search Index]
    
    DATABASE --> SUCCESS_RESPONSE[Success Response]
    SUCCESS_RESPONSE --> RENDER[Render View]
    RENDER --> END([Response to User])
    
    ERROR_RESPONSE --> END
    BUSINESS_ERROR --> END
    FORBIDDEN --> END
    LOGIN --> END
```

## Security Architecture

```mermaid
graph TB
    subgraph "External Threats"
        DDOS[DDoS Attacks]
        MALWARE[Malware]
        INJECTION[SQL Injection]
        XSS[Cross-Site Scripting]
        CSRF[CSRF Attacks]
    end
    
    subgraph "Security Controls"
        subgraph "Network Security"
            WAF_SECURITY[Web Application Firewall]
            RATE_LIMITING[Rate Limiting]
            IP_FILTERING[IP Whitelisting]
        end
        
        subgraph "Application Security"
            AUTHENTICATION[Multi-Factor Authentication]
            AUTHORIZATION[Role-Based Access Control]
            SESSION_MGMT[Session Management]
            INPUT_VALIDATION[Input Validation]
        end
        
        subgraph "Data Security"
            ENCRYPTION_TRANSIT[TLS 1.3 Encryption]
            ENCRYPTION_REST[AES-256 at Rest]
            KEY_MANAGEMENT[Key Management Service]
            DATA_MASKING[Data Masking/Anonymization]
        end
        
        subgraph "Infrastructure Security"
            NETWORK_SEGMENTATION[Network Segmentation]
            VPN_ACCESS[VPN Access Control]
            PRIVILEGE_ESCALATION[Privilege Management]
            PATCH_MANAGEMENT[Security Patching]
        end
    end
    
    subgraph "Monitoring & Response"
        SIEM[Security Information Event Management]
        LOG_ANALYSIS[Security Log Analysis]
        THREAT_DETECTION[Threat Detection]
        INCIDENT_RESPONSE[Incident Response]
        FORENSICS[Digital Forensics]
    end
    
    subgraph "Compliance & Governance"
        GDPR_COMPLIANCE[GDPR Compliance]
        DATA_PROTECTION[Data Protection Act]
        SECURITY_POLICIES[Security Policies]
        AUDIT_TRAILS[Audit Trails]
        RISK_ASSESSMENT[Risk Assessment]
    end
    
    %% Threat mitigation
    DDOS --> WAF_SECURITY
    MALWARE --> IP_FILTERING
    INJECTION --> INPUT_VALIDATION
    XSS --> INPUT_VALIDATION
    CSRF --> SESSION_MGMT
    
    %% Security layers
    WAF_SECURITY --> AUTHENTICATION
    AUTHENTICATION --> AUTHORIZATION
    AUTHORIZATION --> ENCRYPTION_TRANSIT
    ENCRYPTION_TRANSIT --> ENCRYPTION_REST
    
    %% Monitoring connections
    AUTHENTICATION --> SIEM
    AUTHORIZATION --> LOG_ANALYSIS
    ENCRYPTION_REST --> THREAT_DETECTION
    THREAT_DETECTION --> INCIDENT_RESPONSE
    
    %% Compliance connections
    AUDIT_TRAILS --> GDPR_COMPLIANCE
    LOG_ANALYSIS --> DATA_PROTECTION
    RISK_ASSESSMENT --> SECURITY_POLICIES
```

## Integration Architecture

```mermaid
graph TB
    subgraph "Internal Systems"
        CTS[Correspondence Tool Staff]
        WAREHOUSE[Data Warehouse]
        REPORTING[Reporting System]
        ADMIN_TOOLS[Admin Tools]
    end
    
    subgraph "Government Systems"
        GOV_NOTIFY[GOV.UK Notify<br/>Email/SMS Service]
        GOV_SSO[Government Single Sign-On]
        GOV_PAAS[GOV.UK Platform as a Service]
        DIGITAL_MARKETPLACE[Digital Marketplace]
    end
    
    subgraph "Ministry of Justice Systems"
        SSCL[Shared Services Connected Limited]
        DPS[Digital Prison Service]
        PROBATION[National Probation Service]
        HMCTS[HM Courts & Tribunals Service]
    end
    
    subgraph "External Services"
        AWS_S3[AWS S3 Storage]
        SENTRY[Sentry Error Tracking]
        GITHUB[GitHub Source Control]
        SLACK[Slack Notifications]
    end
    
    subgraph "Integration Patterns"
        REST_API[REST API]
        WEBHOOK[Webhooks]
        FILE_TRANSFER[File Transfer]
        MESSAGE_QUEUE[Message Queue]
        DATABASE_SYNC[Database Sync]
    end
    
    %% Internal integrations
    CTS --> WAREHOUSE
    CTS --> REPORTING
    CTS --> ADMIN_TOOLS
    
    %% Government integrations
    CTS --> GOV_NOTIFY
    CTS --> GOV_SSO
    CTS --> GOV_PAAS
    
    %% MoJ integrations
    CTS --> SSCL
    CTS --> DPS
    CTS --> PROBATION
    CTS --> HMCTS
    
    %% External integrations
    CTS --> AWS_S3
    CTS --> SENTRY
    CTS --> GITHUB
    CTS --> SLACK
    
    %% Integration patterns
    WAREHOUSE --> DATABASE_SYNC
    GOV_NOTIFY --> REST_API
    SSCL --> FILE_TRANSFER
    DPS --> WEBHOOK
    SLACK --> MESSAGE_QUEUE
```

## Deployment Pipeline

```mermaid
graph TB
    subgraph "Development"
        DEV_CODE[Developer Code]
        LOCAL_TEST[Local Testing]
        GIT_COMMIT[Git Commit]
    end
    
    subgraph "Continuous Integration"
        TRIGGER[Pipeline Trigger]
        CODE_CHECKOUT[Code Checkout]
        DEPENDENCIES[Install Dependencies]
        UNIT_TESTS[Unit Tests]
        INTEGRATION_TESTS[Integration Tests]
        CODE_QUALITY[Code Quality Checks]
        SECURITY_SCAN[Security Scanning]
        BUILD_IMAGE[Build Docker Image]
    end
    
    subgraph "Staging Deployment"
        DEPLOY_STAGING[Deploy to Staging]
        STAGING_ENV[Staging Environment]
        SMOKE_TESTS[Smoke Tests]
        REGRESSION_TESTS[Regression Tests]
        MANUAL_TESTING[Manual Testing]
        APPROVAL[Approval Gate]
    end
    
    subgraph "Production Deployment"
        DEPLOY_PROD[Deploy to Production]
        BLUE_GREEN[Blue-Green Deployment]
        HEALTH_CHECK[Health Checks]
        ROLLBACK[Rollback Capability]
        MONITORING[Post-Deploy Monitoring]
    end
    
    subgraph "Post-Deployment"
        PERF_MONITORING[Performance Monitoring]
        ERROR_TRACKING[Error Tracking]
        USER_FEEDBACK[User Feedback]
        METRICS[Business Metrics]
    end
    
    %% Development flow
    DEV_CODE --> LOCAL_TEST
    LOCAL_TEST --> GIT_COMMIT
    GIT_COMMIT --> TRIGGER
    
    %% CI flow
    TRIGGER --> CODE_CHECKOUT
    CODE_CHECKOUT --> DEPENDENCIES
    DEPENDENCIES --> UNIT_TESTS
    UNIT_TESTS --> INTEGRATION_TESTS
    INTEGRATION_TESTS --> CODE_QUALITY
    CODE_QUALITY --> SECURITY_SCAN
    SECURITY_SCAN --> BUILD_IMAGE
    
    %% Staging flow
    BUILD_IMAGE --> DEPLOY_STAGING
    DEPLOY_STAGING --> STAGING_ENV
    STAGING_ENV --> SMOKE_TESTS
    SMOKE_TESTS --> REGRESSION_TESTS
    REGRESSION_TESTS --> MANUAL_TESTING
    MANUAL_TESTING --> APPROVAL
    
    %% Production flow
    APPROVAL --> DEPLOY_PROD
    DEPLOY_PROD --> BLUE_GREEN
    BLUE_GREEN --> HEALTH_CHECK
    HEALTH_CHECK --> MONITORING
    
    %% Rollback capability
    HEALTH_CHECK -.-> ROLLBACK
    MONITORING -.-> ROLLBACK
    
    %% Post-deployment
    MONITORING --> PERF_MONITORING
    MONITORING --> ERROR_TRACKING
    MONITORING --> USER_FEEDBACK
    MONITORING --> METRICS
```

## Technology Stack Overview

```mermaid
mindmap
  root((Correspondence Tool Staff))
    Backend
      Ruby on Rails 7
      PostgreSQL
      Redis
      Sidekiq
    Frontend
      ERB Templates
      SCSS/Sass
      JavaScript ES6+
      Stimulus Framework
    Infrastructure
      Docker
      Kubernetes
      AWS/GOV.UK PaaS
      Nginx
    Development
      RSpec Testing
      GitHub Actions CI/CD
      Rubocop Linting
      Brakeman Security
    Monitoring
      Sentry Error Tracking
      Prometheus Metrics
      Grafana Dashboards
      Splunk Logging
    Security
      TLS 1.3
      AES-256 Encryption
      Multi-Factor Auth
      RBAC Authorization
```

This comprehensive architecture documentation provides:

1. **Complete system overview** with all major components
2. **Detailed component interactions** and data flows
3. **Security architecture** showing protection layers
4. **Integration patterns** with external systems
5. **Deployment pipeline** from development to production
6. **Technology stack** overview

All diagrams are created using Mermaid syntax and can be viewed in any Markdown viewer that supports Mermaid, including GitHub, VS Code, and dedicated Mermaid viewers.
