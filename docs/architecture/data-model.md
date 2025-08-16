# Data Model Architecture

## Entity Relationship Overview

```mermaid
erDiagram
    User ||--o{ TeamsUsersRole : "belongs to"
    Team ||--o{ TeamsUsersRole : "has many"
    Team ||--o{ Assignment : "receives"
    User ||--o{ Assignment : "assigned to"
    
    CaseBase ||--o{ Assignment : "has many"
    CaseBase ||--o{ CaseTransition : "has many"
    CaseBase ||--o{ CaseAttachment : "has many"
    CaseBase ||--o{ LinkedCase : "linked to"
    
    User ||--o{ CaseTransition : "performed by"
    Team ||--o{ CaseTransition : "involves"
    
    CaseBase ||--o{ DataRequest : "contains"
    DataRequest ||--o{ DataRequestEmail : "sends"
    
    Team ||--|| Team : "parent/child"
    
    User {
        integer id PK
        string email
        string full_name
        string role
        datetime created_at
        datetime updated_at
    }
    
    Team {
        integer id PK
        string name
        string team_type
        integer parent_id FK
        datetime created_at
        datetime updated_at
    }
    
    CaseBase {
        integer id PK
        string number
        string name
        string email
        string subject
        text message
        date received_date
        date deadline
        string workflow_state
        string type
        datetime created_at
        datetime updated_at
    }
    
    Assignment {
        integer id PK
        integer case_id FK
        integer team_id FK
        integer user_id FK
        string state
        string role
        datetime assigned_at
        datetime created_at
        datetime updated_at
    }
    
    CaseTransition {
        integer id PK
        integer case_id FK
        integer user_id FK
        integer target_team_id FK
        string event
        string from_state
        string to_state
        text metadata
        datetime created_at
    }
```

## Case Type Hierarchy

```mermaid
classDiagram
    class CaseBase {
        +number: string
        +name: string
        +email: string
        +subject: string
        +message: text
        +received_date: date
        +deadline: date
        +workflow_state: string
        +type: string (STI)
        +validate_case_data()
        +transition_to_state()
        +assign_to_team()
    }
    
    class FOIStandard {
        +foi_type: string
        +public_interest_test: boolean
        +exemptions: string[]
        +calculate_deadline()
        +apply_exemptions()
    }
    
    class FOIInternalReview {
        +original_case_id: integer
        +review_reason: string
        +original_deadline: date
        +create_from_original()
    }
    
    class FOIComplianceReview {
        +compliance_issues: text
        +recommendations: text
        +assess_compliance()
    }
    
    class SARStandard {
        +requester_type: string
        +third_party: boolean
        +identity_confirmed: boolean
        +validate_identity()
        +calculate_sar_deadline()
    }
    
    class SAROffender {
        +prisoner_number: string
        +prison: string
        +date_of_birth: date
        +previous_names: string[]
        +validate_prisoner_details()
    }
    
    class SAROffenderComplaint {
        +complaint_type: string
        +complaint_subtype: string
        +priority: string
        +previous_complaint_number: string
        +categorize_complaint()
    }
    
    class ICOBase {
        +ico_reference: string
        +ico_officer: string
        +original_case_id: integer
        +decision_deadline: date
        +link_to_original()
    }
    
    class ICOSAR {
        +ico_decision: string
        +uphold_original: boolean
    }
    
    class ICOFOI {
        +ico_decision: string
        +new_exemptions: string[]
    }
    
    CaseBase <|-- FOIStandard
    CaseBase <|-- SARStandard
    CaseBase <|-- ICOBase
    
    FOIStandard <|-- FOIInternalReview
    FOIInternalReview <|-- FOIComplianceReview
    
    SARStandard <|-- SAROffender
    SAROffender <|-- SAROffenderComplaint
    
    ICOBase <|-- ICOSAR
    ICOBase <|-- ICOFOI
```

## Team Structure & Hierarchy

```mermaid
graph TB
    subgraph "Team Hierarchy"
        DACU[DACU Team<br/>Data Access & Compliance]
        BG1[Business Group 1]
        BG2[Business Group 2]
        BG3[Business Group 3]
        
        DIR1[Directorate 1.1]
        DIR2[Directorate 1.2]
        DIR3[Directorate 2.1]
        DIR4[Directorate 2.2]
        
        BU1[Business Unit 1.1.1]
        BU2[Business Unit 1.1.2]
        BU3[Business Unit 1.2.1]
        BU4[Business Unit 2.1.1]
    end
    
    DACU --> BG1
    DACU --> BG2
    DACU --> BG3
    
    BG1 --> DIR1
    BG1 --> DIR2
    BG2 --> DIR3
    BG2 --> DIR4
    
    DIR1 --> BU1
    DIR1 --> BU2
    DIR2 --> BU3
    DIR3 --> BU4
```

## User Roles & Permissions

```mermaid
graph TB
    subgraph "Role Hierarchy"
        ADMIN[System Admin]
        MANAGER[Team Manager]
        RESPONDER[Responder]
        APPROVER[Approver]
        VIEWER[Viewer]
    end
    
    subgraph "Permissions"
        CREATE_CASE[Create Cases]
        ASSIGN_CASE[Assign Cases]
        APPROVE_CASE[Approve Cases]
        CLOSE_CASE[Close Cases]
        VIEW_ALL[View All Cases]
        EDIT_TEAMS[Edit Teams]
        MANAGE_USERS[Manage Users]
    end
    
    ADMIN --> MANAGE_USERS
    ADMIN --> EDIT_TEAMS
    ADMIN --> VIEW_ALL
    
    MANAGER --> ASSIGN_CASE
    MANAGER --> VIEW_ALL
    MANAGER --> CLOSE_CASE
    
    APPROVER --> APPROVE_CASE
    APPROVER --> VIEW_ALL
    
    RESPONDER --> CREATE_CASE
    RESPONDER --> CLOSE_CASE
    
    VIEWER --> VIEW_ALL
```

## Case State Transitions

```mermaid
stateDiagram-v2
    [*] --> unassigned: create
    
    unassigned --> awaiting_responder: assign_responder
    unassigned --> rejected: reject
    
    awaiting_responder --> accepted: accept_responder_assignment
    awaiting_responder --> rejected: reject_responder_assignment
    
    accepted --> drafting: start_drafting
    accepted --> awaiting_responder: reassign_user
    
    drafting --> pending_dacu_clearance: flag_for_clearance
    drafting --> awaiting_dispatch: ready_for_dispatch
    drafting --> closed: close
    
    pending_dacu_clearance --> awaiting_dispatch: approve
    pending_dacu_clearance --> drafting: request_amends
    pending_dacu_clearance --> closed: close
    
    awaiting_dispatch --> responded: respond
    awaiting_dispatch --> drafting: request_further_action
    awaiting_dispatch --> closed: close
    
    responded --> closed: close
    
    rejected --> [*]
    closed --> [*]
    
    note right of pending_dacu_clearance
        DACU approval required for
        sensitive cases
    end note
```

## Document Management Schema

```mermaid
erDiagram
    CaseBase ||--o{ CaseAttachment : "has many"
    CaseAttachment ||--|| CaseAttachmentUploadGroup : "belongs to"
    CaseAttachmentUploadGroup ||--o{ CaseAttachment : "contains"
    
    User ||--o{ CaseAttachment : "uploaded by"
    
    CaseBase ||--o{ CommissioningDocument : "generates"
    CommissioningDocument ||--|| CommissioningDocumentTemplate : "uses"
    
    CaseAttachment {
        integer id PK
        integer case_id FK
        integer user_id FK
        integer upload_group_id FK
        string filename
        string content_type
        integer file_size
        string upload_path
        boolean preview_converted
        datetime created_at
        datetime updated_at
    }
    
    CaseAttachmentUploadGroup {
        integer id PK
        string upload_comment
        datetime created_at
        datetime updated_at
    }
    
    CommissioningDocument {
        integer id PK
        integer case_id FK
        string template_name
        json data_request_fields
        datetime created_at
        datetime updated_at
    }
    
    CommissioningDocumentTemplate {
        string name PK
        string template_type
        json required_fields
        text template_content
    }
```

## Data Request System

```mermaid
erDiagram
    CaseBase ||--o{ DataRequest : "contains"
    DataRequest ||--o{ DataRequestEmail : "sends"
    DataRequest ||--|| DataRequestArea : "targets"
    
    DataRequest {
        integer id PK
        integer case_id FK
        integer data_request_area_id FK
        date date_requested
        date date_from
        date date_to
        text request_details
        string status
        datetime created_at
        datetime updated_at
    }
    
    DataRequestArea {
        integer id PK
        string name
        string email
        boolean active
        datetime created_at
        datetime updated_at
    }
    
    DataRequestEmail {
        integer id PK
        integer data_request_id FK
        string email_type
        datetime sent_at
        json email_data
        datetime created_at
        datetime updated_at
    }
```

## Audit and Compliance Schema

```mermaid
erDiagram
    CaseBase ||--o{ CaseTransition : "has many"
    CaseBase ||--o{ RetentionSchedule : "follows"
    
    User ||--o{ CaseTransition : "performed by"
    Team ||--o{ CaseTransition : "acted on"
    
    CaseTransition {
        integer id PK
        integer case_id FK
        integer user_id FK
        integer acting_team_id FK
        integer target_team_id FK
        string event
        string from_state
        string to_state
        json metadata
        text message
        datetime created_at
    }
    
    RetentionSchedule {
        integer id PK
        integer case_id FK
        date planned_destruction_date
        string state
        text reason
        datetime created_at
        datetime updated_at
    }
    
    CasesUsersTransitionsTracker {
        integer id PK
        integer case_id FK
        integer user_id FK
        datetime last_transition_date
        integer transition_count
    }
```

## Reporting and Analytics Schema

```mermaid
erDiagram
    Report ||--|| ReportType : "of type"
    User ||--o{ Report : "generates"
    
    Report {
        integer id PK
        integer user_id FK
        integer report_type_id FK
        string status
        json report_data
        string filename
        datetime created_at
        datetime updated_at
    }
    
    ReportType {
        integer id PK
        string abbr
        string full_name
        string class_name
        boolean active
        json default_reporting_period
        datetime created_at
        datetime updated_at
    }
    
    SearchQuery {
        integer id PK
        integer user_id FK
        string query_type
        json filter_params
        integer num_results
        datetime created_at
    }
```

## Database Performance Considerations

### Indexes

```sql
-- Core case lookup indexes
CREATE INDEX idx_cases_number ON cases(number);
CREATE INDEX idx_cases_workflow_state ON cases(workflow_state);
CREATE INDEX idx_cases_received_date ON cases(received_date);
CREATE INDEX idx_cases_deadline ON cases(deadline);
CREATE INDEX idx_cases_type ON cases(type);

-- Assignment indexes
CREATE INDEX idx_assignments_case_team ON assignments(case_id, team_id);
CREATE INDEX idx_assignments_user_state ON assignments(user_id, state);
CREATE INDEX idx_assignments_team_state ON assignments(team_id, state);

-- Transition tracking
CREATE INDEX idx_transitions_case_date ON case_transitions(case_id, created_at);
CREATE INDEX idx_transitions_user_date ON case_transitions(user_id, created_at);

-- Full-text search
CREATE INDEX idx_cases_search ON cases USING gin(to_tsvector('english', subject || ' ' || message));
```

### Data Partitioning Strategy

```mermaid
graph TB
    subgraph "Partitioning Strategy"
        CASES[Cases Table] --> CURRENT[Current Year]
        CASES --> ARCHIVE[Previous Years]
        
        TRANSITIONS[Case Transitions] --> T_CURRENT[Current Year]
        TRANSITIONS --> T_ARCHIVE[Previous Years]
        
        ATTACHMENTS[Case Attachments] --> A_ACTIVE[Active Cases]
        ATTACHMENTS --> A_CLOSED[Closed Cases]
    end
    
    subgraph "Archive Strategy"
        A_ACTIVE --> COLD_STORAGE[Cold Storage]
        T_ARCHIVE --> COLD_STORAGE
        ARCHIVE --> COLD_STORAGE
    end
```

## Data Migration Patterns

### Case Type Migration

```mermaid
sequenceDiagram
    participant Migration
    participant OldCase
    participant NewCase
    participant Validator
    participant AuditLog
    
    Migration->>OldCase: fetch_batch
    Migration->>Validator: validate_data
    Validator->>Migration: validation_result
    Migration->>NewCase: create_with_data
    NewCase->>AuditLog: log_migration
    Migration->>OldCase: mark_migrated
```

### Data Cleanup Patterns

```mermaid
graph TB
    CLEANUP[Data Cleanup Job] --> IDENTIFY[Identify Candidates]
    IDENTIFY --> VALIDATE[Validate Safety]
    VALIDATE --> BACKUP[Create Backup]
    BACKUP --> DELETE[Soft Delete]
    DELETE --> AUDIT[Audit Log]
    AUDIT --> REPORT[Generate Report]
```
