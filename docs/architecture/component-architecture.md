# Component Architecture

## Application Layer Structure

```mermaid
graph TB
    subgraph "Presentation Layer"
        VIEWS[Views/Templates]
        HELPERS[View Helpers]
        ASSETS[Assets/Styles]
    end
    
    subgraph "Controller Layer"
        APP_CTRL[Application Controller]
        CASE_CTRL[Cases Controller]
        ADMIN_CTRL[Admin Controllers]
        API_CTRL[API Controllers]
    end
    
    subgraph "Service Layer"
        CASE_SVC[Case Services]
        NOTIFY_SVC[Notification Services]
        SEARCH_SVC[Search Services]
        FILTER_SVC[Filter Services]
    end
    
    subgraph "Policy Layer"
        AUTH_POLICY[Authorization Policies]
        CASE_POLICY[Case Policies]
        USER_POLICY[User Policies]
    end
    
    subgraph "Model Layer"
        CASE_MODELS[Case Models]
        USER_MODELS[User/Team Models]
        SUPPORT_MODELS[Supporting Models]
    end
    
    subgraph "Infrastructure Layer"
        STATE_MACHINE[State Machines]
        JOBS[Background Jobs]
        MAILERS[Email Mailers]
    end
    
    VIEWS --> CONTROLLER_LAYER
    CONTROLLER_LAYER --> SERVICE_LAYER
    SERVICE_LAYER --> POLICY_LAYER
    SERVICE_LAYER --> MODEL_LAYER
    MODEL_LAYER --> STATE_MACHINE
    SERVICE_LAYER --> JOBS
    JOBS --> MAILERS
```

## Detailed Component Breakdown

### 1. Controllers

```mermaid
classDiagram
    class ApplicationController {
        +authenticate_user()
        +authorize_user()
        +set_current_user()
        +handle_errors()
    }
    
    class CasesController {
        +index()
        +show()
        +new()
        +create()
        +edit()
        +update()
        +destroy()
    }
    
    class AdminController {
        +require_admin()
        +admin_dashboard()
    }
    
    class ApiController {
        +authenticate_api()
        +render_json()
    }
    
    ApplicationController <|-- CasesController
    ApplicationController <|-- AdminController
    ActionController <|-- ApiController
    
    CasesController <|-- FOIController
    CasesController <|-- SARController
    CasesController <|-- OffenderSARController
```

### 2. Service Objects Architecture

```mermaid
graph TB
    subgraph "Case Creation Services"
        CREATE[CaseCreateService]
        STEPPED[SteppedCaseBuilder]
        VALIDATE[CaseValidationService]
    end
    
    subgraph "Assignment Services"
        ASSIGN_TEAM[AssignNewTeamService]
        ASSIGN_USER[CaseAssignResponderService]
        ASSIGN_MEMBER[CaseAssignToTeamMemberService]
    end
    
    subgraph "Workflow Services"
        APPROVAL[CaseApprovalService]
        CLOSURE[CaseClosureService]
        REOPEN[CaseReopenService]
        FLAG[CaseFlagForClearanceService]
    end
    
    subgraph "Notification Services"
        NOTIFY_ASSIGN[NotifyNewAssignmentService]
        NOTIFY_TEAM[NotifyTeamService]
        NOTIFY_RESP[NotifyResponderService]
    end
    
    subgraph "Search & Filter Services"
        SEARCH[CaseSearchService]
        FINDER[CaseFinderService]
        FILTERS[Case Filter Classes]
    end
    
    CREATE --> ASSIGN_TEAM
    ASSIGN_TEAM --> NOTIFY_ASSIGN
    APPROVAL --> NOTIFY_TEAM
    CLOSURE --> NOTIFY_RESP
```

### 3. Model Hierarchy

```mermaid
classDiagram
    class ApplicationRecord {
        +id: integer
        +created_at: datetime
        +updated_at: datetime
    }
    
    class CaseBase {
        +number: string
        +name: string
        +email: string
        +subject: string
        +message: text
        +received_date: date
        +deadline: date
        +workflow_state: string
        +validate_case_data()
        +assign_to_team()
        +close_case()
    }
    
    class User {
        +email: string
        +full_name: string
        +role: string
        +teams: Team[]
        +cases: Case[]
        +authenticate()
        +authorize()
    }
    
    class Team {
        +name: string
        +team_type: string
        +users: User[]
        +parent_team: Team
        +child_teams: Team[]
        +can_handle_case_type()
    }
    
    class Assignment {
        +state: string
        +role: string
        +assigned_at: datetime
        +user: User
        +team: Team
        +case: Case
    }
    
    ApplicationRecord <|-- CaseBase
    ApplicationRecord <|-- User
    ApplicationRecord <|-- Team
    ApplicationRecord <|-- Assignment
    
    CaseBase <|-- FOIStandard
    CaseBase <|-- SARStandard
    CaseBase <|-- OffenderSAR
    CaseBase <|-- ICOCase
    
    User ||--o{ Assignment
    Team ||--o{ Assignment
    CaseBase ||--o{ Assignment
```

### 4. State Machine Architecture

```mermaid
stateDiagram-v2
    [*] --> unassigned: Create Case
    
    unassigned --> awaiting_responder: Assign to Team
    awaiting_responder --> accepted: Accept Assignment
    accepted --> drafting: Start Drafting
    
    drafting --> pending_dacu_clearance: Flag for Clearance
    pending_dacu_clearance --> awaiting_dispatch: Approve
    pending_dacu_clearance --> drafting: Request Changes
    
    drafting --> awaiting_dispatch: Ready to Send
    awaiting_dispatch --> responded: Send Response
    responded --> closed: Close Case
    
    drafting --> closed: Close Case
    accepted --> closed: Close Case
    
    closed --> [*]
    
    note right of pending_dacu_clearance
        DACU = Data Access and 
        Compliance Unit
    end note
```

### 5. Policy Architecture

```mermaid
graph TB
    subgraph "Authorization Flow"
        REQUEST[HTTP Request]
        CONTROLLER[Controller Action]
        POLICY[Policy Check]
        ACTION[Execute Action]
        RESPONSE[HTTP Response]
    end
    
    subgraph "Policy Hierarchy"
        APP_POLICY[ApplicationPolicy]
        CASE_POLICY[Case::BasePolicy]
        FOI_POLICY[Case::FOI::StandardPolicy]
        SAR_POLICY[Case::SAR::StandardPolicy]
        USER_POLICY[UserPolicy]
        TEAM_POLICY[TeamPolicy]
    end
    
    subgraph "Policy Methods"
        SHOW[show?]
        CREATE[create?]
        UPDATE[update?]
        DESTROY[destroy?]
        SCOPE[Scope.resolve]
    end
    
    REQUEST --> CONTROLLER
    CONTROLLER --> POLICY
    POLICY --> ACTION
    ACTION --> RESPONSE
    
    APP_POLICY --> CASE_POLICY
    CASE_POLICY --> FOI_POLICY
    CASE_POLICY --> SAR_POLICY
    APP_POLICY --> USER_POLICY
    APP_POLICY --> TEAM_POLICY
    
    POLICY --> SHOW
    POLICY --> CREATE
    POLICY --> UPDATE
    POLICY --> DESTROY
    POLICY --> SCOPE
```

## Service Layer Patterns

### 1. Command Pattern
Services follow the command pattern with a single `call` method:

```ruby
class CaseCreateService
  def self.call(user:, case_type:, params:)
    new(user: user, case_type: case_type, params: params).call
  end
  
  def call
    validate_params
    create_case
    assign_initial_team
    send_notifications
    case
  end
end
```

### 2. Factory Pattern
Case creation uses factory pattern for different case types:

```mermaid
graph TB
    FACTORY[CaseFactory] --> FOI_FACTORY[FOI::CaseFactory]
    FACTORY --> SAR_FACTORY[SAR::CaseFactory]
    FACTORY --> ICO_FACTORY[ICO::CaseFactory]
    
    FOI_FACTORY --> FOI_STANDARD[FOI::Standard]
    FOI_FACTORY --> FOI_REVIEW[FOI::InternalReview]
    
    SAR_FACTORY --> SAR_STANDARD[SAR::Standard]
    SAR_FACTORY --> SAR_OFFENDER[SAR::Offender]
```

### 3. Observer Pattern
State changes trigger notifications:

```mermaid
sequenceDiagram
    participant Service
    participant StateMachine
    participant Observer
    participant NotificationService
    
    Service->>StateMachine: trigger_event!
    StateMachine->>Observer: state_changed
    Observer->>NotificationService: send_notification
    NotificationService->>External: email/sms
```

## Component Interaction Patterns

### 1. Case Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Controller
    participant Service
    participant Model
    participant StateMachine
    participant NotificationService
    
    User->>Controller: POST /cases
    Controller->>Service: CaseCreateService.call
    Service->>Model: Case.create
    Model->>StateMachine: set_initial_state
    Service->>NotificationService: notify_team
    NotificationService->>External: send_email
    Service->>Controller: return case
    Controller->>User: redirect_to case
```

### 2. Assignment Flow

```mermaid
sequenceDiagram
    participant Manager
    participant Controller
    participant AssignmentService
    participant Case
    participant Team
    participant User
    participant NotificationService
    
    Manager->>Controller: assign case
    Controller->>AssignmentService: call
    AssignmentService->>Case: create_assignment
    Case->>Team: validate_can_handle
    AssignmentService->>User: notify_assignment
    AssignmentService->>NotificationService: send_notification
    NotificationService->>User: email notification
```

### 3. Approval Workflow

```mermaid
graph TB
    DRAFT[Drafting State] --> FLAG{Flag for Clearance?}
    FLAG -->|Yes| PENDING[Pending DACU Clearance]
    FLAG -->|No| DISPATCH[Awaiting Dispatch]
    
    PENDING --> APPROVE{Approved?}
    APPROVE -->|Yes| DISPATCH
    APPROVE -->|No| AMEND[Request Amendments]
    AMEND --> DRAFT
    
    DISPATCH --> SEND[Send Response]
    SEND --> CLOSE[Closed]
```

## Background Job Architecture

```mermaid
graph TB
    subgraph "Job Types"
        EMAIL_JOBS[Email Jobs]
        REPORT_JOBS[Report Generation]
        CLEANUP_JOBS[Data Cleanup]
        INTEGRATION_JOBS[External API Jobs]
    end
    
    subgraph "Queue Management"
        REDIS[Redis Queue]
        SIDEKIQ[Sidekiq Workers]
        SCHEDULER[Cron Jobs]
    end
    
    subgraph "Job Processing"
        WORKER1[Worker 1]
        WORKER2[Worker 2]
        WORKER3[Worker 3]
    end
    
    EMAIL_JOBS --> REDIS
    REPORT_JOBS --> REDIS
    CLEANUP_JOBS --> REDIS
    INTEGRATION_JOBS --> REDIS
    
    REDIS --> SIDEKIQ
    SCHEDULER --> SIDEKIQ
    
    SIDEKIQ --> WORKER1
    SIDEKIQ --> WORKER2
    SIDEKIQ --> WORKER3
```

## Integration Points

### External Service Integration

```mermaid
graph TB
    subgraph "Application"
        RAILS[Rails App]
        JOBS[Background Jobs]
    end
    
    subgraph "Government Services"
        NOTIFY[GOV.UK Notify]
        SSO[Government SSO]
        S3[AWS S3]
    end
    
    subgraph "Internal Services"
        SSCL[SSCL Systems]
        AD[Active Directory]
        WAREHOUSE[Data Warehouse]
    end
    
    RAILS --> NOTIFY
    RAILS --> SSO
    RAILS --> S3
    JOBS --> SSCL
    JOBS --> AD
    JOBS --> WAREHOUSE
```

## Error Handling Strategy

```mermaid
graph TB
    ERROR[Error Occurs] --> TYPE{Error Type}
    
    TYPE -->|Validation| VALIDATION[ValidationError]
    TYPE -->|Authorization| AUTH_ERROR[AuthorizationError]
    TYPE -->|Business Logic| BUSINESS[BusinessLogicError]
    TYPE -->|System| SYSTEM[SystemError]
    
    VALIDATION --> USER_MSG[User Friendly Message]
    AUTH_ERROR --> REDIRECT[Redirect to Login]
    BUSINESS --> RETRY[Retry Logic]
    SYSTEM --> SENTRY[Log to Sentry]
    
    SENTRY --> ALERT[Alert DevOps]
    USER_MSG --> RESPONSE[Error Response]
    REDIRECT --> RESPONSE
    RETRY --> RESPONSE
```
