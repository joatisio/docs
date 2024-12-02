# DATABASE SCHEMA  

  
```sql
-- Core Authentication & User Management
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255),
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    role VARCHAR(50) NOT NULL, -- admin, hiring_manager, recruiter, interviewer
    status VARCHAR(50) NOT NULL DEFAULT 'active', -- active, inactive, invited
    settings JSONB DEFAULT '{}',
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Job Management
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    created_by UUID NOT NULL REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    requirements TEXT,
    location VARCHAR(255),
    job_type VARCHAR(50), -- full-time, part-time, contract
    status VARCHAR(50) NOT NULL, -- draft, published, closed
    department VARCHAR(255),
    salary_min INTEGER,
    salary_max INTEGER,
    salary_currency VARCHAR(3),
    published_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Candidate Management
CREATE TABLE candidates (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    location VARCHAR(255),
    current_company VARCHAR(255),
    current_title VARCHAR(255),
    source VARCHAR(100), -- referral, job board, direct
    status VARCHAR(50) NOT NULL, -- active, archived, blacklisted
    tags TEXT[],
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (company_id, email)
);

-- Application Pipeline
CREATE TABLE job_pipelines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_id UUID NOT NULL REFERENCES jobs(id),
    name VARCHAR(255) NOT NULL,
    stages JSONB NOT NULL, -- Array of stage configurations
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    job_id UUID NOT NULL REFERENCES jobs(id),
    candidate_id UUID NOT NULL REFERENCES candidates(id),
    pipeline_id UUID NOT NULL REFERENCES job_pipelines(id),
    current_stage VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL, -- active, rejected, withdrawn, hired
    applied_at TIMESTAMP WITH TIME ZONE NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (job_id, candidate_id)
);

-- Interview Management
CREATE TABLE interviews (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    application_id UUID NOT NULL REFERENCES applications(id),
    stage_name VARCHAR(100) NOT NULL,
    scheduled_at TIMESTAMP WITH TIME ZONE NOT NULL,
    duration_minutes INTEGER NOT NULL,
    location VARCHAR(255), -- can be physical location or video call link
    status VARCHAR(50) NOT NULL, -- scheduled, completed, cancelled, no_show
    feedback_submitted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE interview_participants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    interview_id UUID NOT NULL REFERENCES interviews(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role VARCHAR(50) NOT NULL, -- interviewer, coordinator
    status VARCHAR(50) NOT NULL, -- pending, accepted, declined
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (interview_id, user_id)
);

-- Feedback & Evaluation
CREATE TABLE interview_feedbacks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    interview_id UUID NOT NULL REFERENCES interviews(id),
    user_id UUID NOT NULL REFERENCES users(id),
    overall_rating INTEGER CHECK (overall_rating BETWEEN 1 AND 5),
    feedback_data JSONB NOT NULL, -- Structured feedback data
    recommendation VARCHAR(50) NOT NULL, -- strong_yes, yes, maybe, no, strong_no
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (interview_id, user_id)
);

-- Document Management
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    candidate_id UUID NOT NULL REFERENCES candidates(id),
    type VARCHAR(50) NOT NULL, -- resume, cover_letter, portfolio, other
    file_name VARCHAR(255) NOT NULL,
    file_size INTEGER NOT NULL,
    file_type VARCHAR(100) NOT NULL,
    storage_path TEXT NOT NULL,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Activity Logging
CREATE TABLE activity_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID NOT NULL REFERENCES companies(id),
    user_id UUID REFERENCES users(id),
    entity_type VARCHAR(50) NOT NULL, -- candidate, job, application, interview
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    details JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

Important Indexes to Add:
```sql
-- Performance Indexes
CREATE INDEX idx_users_company_id ON users(company_id);
CREATE INDEX idx_jobs_company_id ON jobs(company_id);
CREATE INDEX idx_candidates_company_id ON candidates(company_id);
CREATE INDEX idx_applications_job_id ON applications(job_id);
CREATE INDEX idx_applications_candidate_id ON applications(candidate_id);
CREATE INDEX idx_interviews_application_id ON interviews(application_id);
CREATE INDEX idx_documents_candidate_id ON documents(candidate_id);
CREATE INDEX idx_activity_logs_entity ON activity_logs(entity_type, entity_id);

-- Search Indexes
CREATE INDEX idx_candidates_email ON candidates(email);
CREATE INDEX idx_candidates_tags ON candidates USING gin(tags);
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_applications_status ON applications(status);
```

Key Features of this Schema:
1. Multi-tenant by design (company_id everywhere)
2. Comprehensive audit logging
3. Flexible metadata storage using JSONB
4. Proper relationships and constraints
5. Efficient indexing for common queries
6. Support for complex workflows
7. Document management
8. Interview scheduling and feedback
