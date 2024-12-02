# REST API structure

```typescript
// Base API URL: /api/v1

// Types for Request/Response
interface PaginationParams {
  page: number;
  per_page: number;
  sort_by?: string;
  sort_order?: 'asc' | 'desc';
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  per_page: number;
  total_pages: number;
}

interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: any;
  };
}
```

### 1. Authentication & User Management
```typescript
// Authentication
POST /auth/login
Request:
{
  email: string;
  password: string;
}
Response:
{
  token: string;
  user: User;
}

POST /auth/logout
Response: 200 OK

POST /auth/refresh-token
Response:
{
  token: string;
}

// User Management
GET /users
Query Params: PaginationParams & {
  role?: string;
  status?: string;
  search?: string;
}
Response: PaginatedResponse<User>

POST /users
Request:
{
  email: string;
  first_name: string;
  last_name: string;
  role: string;
  // other user fields
}
Response: User

GET /users/:id
Response: User

PUT /users/:id
Request: Partial<User>
Response: User

DELETE /users/:id
Response: 204 No Content
```

### 2. Job Management
```typescript
GET /jobs
Query Params: PaginationParams & {
  status?: string;
  department?: string;
  location?: string;
  search?: string;
}
Response: PaginatedResponse<Job>

POST /jobs
Request:
{
  title: string;
  description: string;
  requirements: string;
  location: string;
  job_type: string;
  department: string;
  salary_min?: number;
  salary_max?: number;
  salary_currency?: string;
  metadata?: Record<string, any>;
}
Response: Job

GET /jobs/:id
Response: Job

PUT /jobs/:id
Request: Partial<Job>
Response: Job

DELETE /jobs/:id
Response: 204 No Content

// Job Pipeline Management
GET /jobs/:id/pipeline
Response: Pipeline

PUT /jobs/:id/pipeline
Request: {
  stages: PipelineStage[];
}
Response: Pipeline

// Job Statistics
GET /jobs/:id/statistics
Response: {
  total_applications: number;
  applications_by_stage: Record<string, number>;
  time_to_fill: number;
  // other statistics
}
```

### 3. Candidate Management
```typescript
GET /candidates
Query Params: PaginationParams & {
  status?: string;
  tags?: string[];
  source?: string;
  search?: string;
}
Response: PaginatedResponse<Candidate>

POST /candidates
Request:
{
  first_name: string;
  last_name: string;
  email: string;
  phone?: string;
  location?: string;
  current_company?: string;
  current_title?: string;
  source?: string;
  tags?: string[];
  metadata?: Record<string, any>;
}
Response: Candidate

GET /candidates/:id
Response: Candidate

PUT /candidates/:id
Request: Partial<Candidate>
Response: Candidate

DELETE /candidates/:id
Response: 204 No Content

// Candidate Documents
POST /candidates/:id/documents
Request: FormData // multipart/form-data
Response: Document

GET /candidates/:id/documents
Response: Document[]

DELETE /candidates/:id/documents/:documentId
Response: 204 No Content
```

### 4. Application Management
```typescript
GET /applications
Query Params: PaginationParams & {
  job_id?: string;
  candidate_id?: string;
  status?: string;
  stage?: string;
}
Response: PaginatedResponse<Application>

POST /applications
Request:
{
  job_id: string;
  candidate_id: string;
  // other application fields
}
Response: Application

GET /applications/:id
Response: Application

PUT /applications/:id/stage
Request:
{
  stage: string;
  notes?: string;
}
Response: Application

PUT /applications/:id/status
Request:
{
  status: string;
  reason?: string;
}
Response: Application
```

### 5. Interview Management
```typescript
GET /interviews
Query Params: PaginationParams & {
  application_id?: string;
  status?: string;
  from_date?: string;
  to_date?: string;
}
Response: PaginatedResponse<Interview>

POST /interviews
Request:
{
  application_id: string;
  stage_name: string;
  scheduled_at: string;
  duration_minutes: number;
  location: string;
  participants: {
    user_id: string;
    role: string;
  }[];
}
Response: Interview

GET /interviews/:id
Response: Interview

PUT /interviews/:id
Request: Partial<Interview>
Response: Interview

DELETE /interviews/:id
Response: 204 No Content

// Interview Feedback
POST /interviews/:id/feedback
Request:
{
  overall_rating: number;
  feedback_data: Record<string, any>;
  recommendation: string;
  notes?: string;
}
Response: InterviewFeedback

GET /interviews/:id/feedback
Response: InterviewFeedback[]
```

### 6. Analytics & Reporting
```typescript
GET /analytics/overview
Response: {
  active_jobs: number;
  total_candidates: number;
  ongoing_interviews: number;
  // other overview metrics
}

GET /analytics/recruitment-funnel
Query Params: {
  job_id?: string;
  date_from?: string;
  date_to?: string;
}
Response: {
  stages: {
    name: string;
    count: number;
    conversion_rate: number;
  }[];
}

GET /analytics/time-to-hire
Response: {
  average_days: number;
  by_department: Record<string, number>;
  by_position_type: Record<string, number>;
}
```

### 7. System Configuration
```typescript
GET /config/pipeline-stages
Response: {
  available_stages: {
    id: string;
    name: string;
    description: string;
  }[];
}

GET /config/user-roles
Response: {
  available_roles: {
    id: string;
    name: string;
    permissions: string[];
  }[];
}
```

Authentication & Authorization:
```typescript
// All endpoints except /auth/* require:
headers: {
  Authorization: 'Bearer <token>'
}

// Common Error Responses:
401 Unauthorized:
{
  error: {
    code: 'unauthorized',
    message: 'Invalid or expired token'
  }
}

403 Forbidden:
{
  error: {
    code: 'forbidden',
    message: 'Insufficient permissions'
  }
}

404 Not Found:
{
  error: {
    code: 'not_found',
    message: 'Resource not found'
  }
}
```
