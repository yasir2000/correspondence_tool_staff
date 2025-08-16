# Service Architecture

## Service Layer Overview

The service layer encapsulates business logic and orchestrates complex operations across multiple models. Services follow consistent patterns and conventions for maintainability and testability.

```mermaid
graph TB
    subgraph "Service Categories"
        CASE_SVC[Case Services]
        WORKFLOW_SVC[Workflow Services]
        NOTIFICATION_SVC[Notification Services]
        SEARCH_SVC[Search & Filter Services]
        INTEGRATION_SVC[Integration Services]
        UTILITY_SVC[Utility Services]
    end
    
    subgraph "Service Patterns"
        COMMAND[Command Pattern]
        FACTORY[Factory Pattern]
        OBSERVER[Observer Pattern]
        STRATEGY[Strategy Pattern]
    end
    
    subgraph "Infrastructure Services"
        QUEUE[Job Queue]
        CACHE[Caching Layer]
        EXTERNAL[External APIs]
    end
    
    CASE_SVC --> COMMAND
    WORKFLOW_SVC --> OBSERVER
    NOTIFICATION_SVC --> STRATEGY
    SEARCH_SVC --> FACTORY
    
    CASE_SVC --> QUEUE
    NOTIFICATION_SVC --> EXTERNAL
    SEARCH_SVC --> CACHE
```

## Core Service Architecture

### 1. Case Management Services

```mermaid
classDiagram
    class CaseServiceBase {
        +user: User
        +case: Case
        +call()
        #validate_permissions()
        #log_transition()
        #send_notifications()
    }
    
    class CaseCreateService {
        +case_type: String
        +params: Hash
        +call()
        #build_case()
        #assign_initial_team()
        #create_initial_assignment()
    }
    
    class CaseAssignService {
        +team: Team
        +role: String
        +call()
        #validate_team_can_handle()
        #create_assignment()
        #transition_state()
    }
    
    class CaseApprovalService {
        +approval_type: String
        +call()
        #validate_can_approve()
        #apply_approval()
        #notify_stakeholders()
    }
    
    class CaseClosureService {
        +outcome: String
        +metadata: Hash
        +call()
        #validate_can_close()
        #apply_closure_metadata()
        #trigger_retention_schedule()
    }
    
    CaseServiceBase <|-- CaseCreateService
    CaseServiceBase <|-- CaseAssignService
    CaseServiceBase <|-- CaseApprovalService
    CaseServiceBase <|-- CaseClosureService
```

### 2. Workflow State Management

```mermaid
sequenceDiagram
    participant Service
    participant StateMachine
    participant Case
    participant AuditLog
    participant NotificationService
    
    Service->>StateMachine: can_transition?(event)
    StateMachine->>Service: validation_result
    Service->>Case: trigger_event!(event)
    Case->>StateMachine: execute_transition
    StateMachine->>AuditLog: log_transition
    StateMachine->>NotificationService: trigger_hooks
    NotificationService->>External: send_notifications
    Case->>Service: updated_case
```

### 3. Assignment Service Flow

```mermaid
flowchart TD
    START[Assignment Request] --> VALIDATE{Validate Request}
    VALIDATE -->|Invalid| ERROR[Return Error]
    VALIDATE -->|Valid| CHECK_TEAM{Team Can Handle?}
    
    CHECK_TEAM -->|No| ERROR
    CHECK_TEAM -->|Yes| CREATE_ASSIGNMENT[Create Assignment]
    
    CREATE_ASSIGNMENT --> UPDATE_STATE[Update Case State]
    UPDATE_STATE --> LOG[Log Transition]
    LOG --> NOTIFY[Send Notifications]
    NOTIFY --> SUCCESS[Return Success]
```

## Service Pattern Implementation

### 1. Command Pattern Services

```ruby
class CaseCreateService
  include ServiceResult
  
  def self.call(**args)
    new(**args).call
  end
  
  def initialize(user:, case_type:, params:)
    @user = user
    @case_type = case_type
    @params = params
  end
  
  def call
    validate_permissions!
    build_case
    assign_initial_team
    send_notifications
    
    ServiceResult.success(case: @case)
  rescue => error
    ServiceResult.failure(error: error)
  end
  
  private
  
  def validate_permissions!
    raise UnauthorizedError unless can_create_case?
  end
  
  def build_case
    @case = @case_type.constantize.create!(@params)
  end
  
  def assign_initial_team
    DefaultTeamService.call(case: @case, user: @user)
  end
  
  def send_notifications
    NotifyNewCaseService.call(case: @case)
  end
end
```

### 2. Strategy Pattern for Notifications

```mermaid
classDiagram
    class NotificationStrategy {
        <<interface>>
        +send(recipient, message)
    }
    
    class EmailNotification {
        +send(recipient, message)
        +template: String
        +delivery_method: Symbol
    }
    
    class SMSNotification {
        +send(recipient, message)
        +phone_number: String
        +service_provider: String
    }
    
    class InAppNotification {
        +send(recipient, message)
        +user_id: Integer
        +notification_type: String
    }
    
    class NotificationService {
        +strategies: Array[NotificationStrategy]
        +send_notification(type, recipient, message)
        #select_strategy(type)
    }
    
    NotificationStrategy <|.. EmailNotification
    NotificationStrategy <|.. SMSNotification
    NotificationStrategy <|.. InAppNotification
    NotificationService --> NotificationStrategy
```

### 3. Factory Pattern for Case Creation

```mermaid
graph TB
    REQUEST[Case Creation Request] --> FACTORY[CaseFactory]
    FACTORY --> RESOLVER[TypeResolver]
    RESOLVER --> VALIDATOR[ParamsValidator]
    VALIDATOR --> BUILDER[CaseBuilder]
    
    BUILDER --> FOI_BUILDER[FOI::CaseBuilder]
    BUILDER --> SAR_BUILDER[SAR::CaseBuilder]
    BUILDER --> ICO_BUILDER[ICO::CaseBuilder]
    
    FOI_BUILDER --> FOI_CASE[FOI::Standard]
    SAR_BUILDER --> SAR_CASE[SAR::Standard]
    ICO_BUILDER --> ICO_CASE[ICO::Base]
```

## Search and Filter Service Architecture

### 1. Filter Chain Pattern

```mermaid
graph LR
    QUERY[Search Query] --> PARSER[Query Parser]
    PARSER --> FILTER1[Date Filter]
    FILTER1 --> FILTER2[Status Filter]
    FILTER2 --> FILTER3[Team Filter]
    FILTER3 --> FILTER4[Type Filter]
    FILTER4 --> RESULTS[Filtered Results]
    
    subgraph "Filter Chain"
        FILTER1
        FILTER2
        FILTER3
        FILTER4
    end
```

### 2. Search Service Implementation

```mermaid
classDiagram
    class CaseSearchService {
        +query_params: Hash
        +user: User
        +call()
        #parse_query()
        #apply_filters()
        #apply_permissions()
        #execute_search()
    }
    
    class FilterChain {
        +filters: Array[Filter]
        +apply(scope)
        +add_filter(filter)
    }
    
    class CaseFilterBase {
        <<abstract>>
        +params: Hash
        +apply(scope)
        #validate_params()
    }
    
    class DateRangeFilter {
        +start_date: Date
        +end_date: Date
        +apply(scope)
    }
    
    class StatusFilter {
        +statuses: Array[String]
        +apply(scope)
    }
    
    class TeamFilter {
        +team_ids: Array[Integer]
        +apply(scope)
    }
    
    CaseSearchService --> FilterChain
    FilterChain --> CaseFilterBase
    CaseFilterBase <|-- DateRangeFilter
    CaseFilterBase <|-- StatusFilter
    CaseFilterBase <|-- TeamFilter
```

### 3. Filter Configuration

```yaml
# config/case_filters.yml
filters:
  date_received:
    class_name: CaseFilter::ReceivedDateFilter
    template: date_range
    default_params:
      start_date: -30.days
      end_date: today
      
  case_status:
    class_name: CaseFilter::CaseStatusFilter
    template: multi_select
    options:
      - unassigned
      - awaiting_responder
      - drafting
      - pending_clearance
      - closed
      
  assigned_team:
    class_name: CaseFilter::TeamFilter
    template: team_select
    scope: user_teams
```

## Background Job Services

### 1. Job Categories and Queues

```mermaid
graph TB
    subgraph "Job Queues"
        CRITICAL[Critical Queue]
        HIGH[High Priority Queue]
        DEFAULT[Default Queue]
        LOW[Low Priority Queue]
        CLEANUP[Cleanup Queue]
    end
    
    subgraph "Job Types"
        EMAIL[Email Jobs] --> HIGH
        NOTIFICATION[Notification Jobs] --> HIGH
        REPORT[Report Generation] --> DEFAULT
        DATA_EXPORT[Data Export] --> DEFAULT
        ARCHIVE[Archive Jobs] --> LOW
        CLEANUP_JOBS[Cleanup Jobs] --> CLEANUP
        INTEGRATION[External API] --> CRITICAL
    end
    
    subgraph "Workers"
        WORKER1[Worker 1] --> CRITICAL
        WORKER2[Worker 2] --> HIGH
        WORKER3[Worker 3] --> DEFAULT
        WORKER4[Worker 4] --> LOW
    end
```

### 2. Email Service Architecture

```mermaid
sequenceDiagram
    participant Service
    participant EmailJob
    participant Template
    participant GovNotify
    participant AuditLog
    
    Service->>EmailJob: queue_email
    EmailJob->>Template: render_content
    Template->>EmailJob: html_content
    EmailJob->>GovNotify: send_email
    GovNotify->>EmailJob: delivery_status
    EmailJob->>AuditLog: log_delivery
    EmailJob->>Service: completion_status
```

### 3. Report Generation Service

```mermaid
flowchart TD
    REQUEST[Report Request] --> VALIDATE[Validate Parameters]
    VALIDATE --> QUEUE[Queue Background Job]
    QUEUE --> FETCH[Fetch Data]
    FETCH --> PROCESS[Process Data]
    PROCESS --> GENERATE[Generate Report]
    GENERATE --> STORE[Store File]
    STORE --> NOTIFY[Notify User]
    NOTIFY --> CLEANUP[Schedule Cleanup]
```

## Integration Service Patterns

### 1. External API Integration

```mermaid
classDiagram
    class ExternalServiceBase {
        +endpoint: String
        +timeout: Integer
        +retry_attempts: Integer
        +call()
        #authenticate()
        #make_request()
        #handle_response()
        #handle_errors()
    }
    
    class GovNotifyService {
        +api_key: String
        +send_email(template, recipient, params)
        +send_sms(template, recipient, params)
        +get_delivery_status(notification_id)
    }
    
    class SSCLIntegrationService {
        +base_url: String
        +sync_user_data()
        +fetch_prison_data()
        +update_offender_records()
    }
    
    class S3StorageService {
        +bucket: String
        +region: String
        +upload_file(file, key)
        +download_file(key)
        +generate_presigned_url(key)
    }
    
    ExternalServiceBase <|-- GovNotifyService
    ExternalServiceBase <|-- SSCLIntegrationService
    ExternalServiceBase <|-- S3StorageService
```

### 2. Circuit Breaker Pattern

```mermaid
stateDiagram-v2
    [*] --> Closed: Normal Operation
    
    Closed --> Open: Failure Threshold Reached
    Open --> HalfOpen: Timeout Elapsed
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    note right of Open
        All requests fail fast
        No external calls made
    end note
    
    note right of HalfOpen
        Limited requests allowed
        Testing if service recovered
    end note
```

### 3. Retry Strategy

```mermaid
graph TB
    REQUEST[API Request] --> ATTEMPT{Attempt}
    ATTEMPT -->|Success| SUCCESS[Return Response]
    ATTEMPT -->|Failure| CHECK{Retries Left?}
    
    CHECK -->|Yes| BACKOFF[Exponential Backoff]
    CHECK -->|No| CIRCUIT[Update Circuit Breaker]
    
    BACKOFF --> WAIT[Wait Period]
    WAIT --> ATTEMPT
    
    CIRCUIT --> FAILURE[Return Failure]
```

## Service Testing Patterns

### 1. Service Test Structure

```ruby
RSpec.describe CaseCreateService do
  describe '.call' do
    let(:user) { create(:user) }
    let(:case_type) { 'Case::FOI::Standard' }
    let(:params) { build(:foi_case_params) }
    
    subject(:result) { described_class.call(user: user, case_type: case_type, params: params) }
    
    context 'with valid parameters' do
      it 'creates a new case' do
        expect { result }.to change { Case::FOI::Standard.count }.by(1)
      end
      
      it 'assigns the case to default team' do
        expect(result.case.assignments).to be_present
      end
      
      it 'sends notifications' do
        expect(NotifyNewCaseService).to receive(:call)
        result
      end
    end
    
    context 'with invalid parameters' do
      let(:params) { { subject: nil } }
      
      it 'returns failure result' do
        expect(result).to be_failure
      end
      
      it 'does not create a case' do
        expect { result }.not_to change { Case::FOI::Standard.count }
      end
    end
  end
end
```

### 2. Mock External Services

```ruby
class MockGovNotifyService
  def self.send_email(template:, recipient:, personalisation:)
    Rails.logger.info "Mock email sent to #{recipient}"
    OpenStruct.new(id: SecureRandom.uuid, status: 'delivered')
  end
end

# In test configuration
Rails.application.configure do
  config.gov_notify_service = MockGovNotifyService
end
```

## Performance Optimization

### 1. Service Caching Strategy

```mermaid
graph TB
    REQUEST[Service Request] --> CACHE_CHECK{Cache Hit?}
    CACHE_CHECK -->|Yes| RETURN_CACHED[Return Cached Result]
    CACHE_CHECK -->|No| EXECUTE[Execute Service]
    EXECUTE --> CACHE_STORE[Store in Cache]
    CACHE_STORE --> RETURN_RESULT[Return Result]
    
    subgraph "Cache Layers"
        MEMORY[Memory Cache]
        REDIS[Redis Cache]
        DB_CACHE[Database Cache]
    end
    
    CACHE_CHECK --> MEMORY
    CACHE_CHECK --> REDIS
    CACHE_CHECK --> DB_CACHE
```

### 2. Async Processing

```mermaid
sequenceDiagram
    participant Controller
    participant Service
    participant Job
    participant Worker
    participant User
    
    Controller->>Service: call_async
    Service->>Job: enqueue
    Job->>Worker: execute
    Service->>Controller: job_id
    Controller->>User: processing_message
    
    Worker->>User: completion_notification
```

### 3. Database Optimization

```ruby
# Service optimization example
class CaseSearchService
  def call
    # Use includes to avoid N+1 queries
    scope = Case.includes(:assignments, :transitions, :attachments)
    
    # Apply filters efficiently
    scope = apply_filters(scope)
    
    # Use counter cache for counts
    scope = scope.select('cases.*, assignments_count, transitions_count')
    
    # Paginate results
    scope.page(page).per(per_page)
  end
  
  private
  
  def apply_filters(scope)
    filters.reduce(scope) do |current_scope, filter|
      filter.apply(current_scope)
    end
  end
end
```
