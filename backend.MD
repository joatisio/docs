# Backend Architecture
```
ats/
├── cmd/
│   └── api/
│       └── main.go
├── config/
│   ├── config.go
│   └── config.yaml
├── internal/
│   ├── model/              // All domain models in one place
│   │   ├── user.go
│   │   ├── company.go
│   │   ├── job.go
│   │   ├── candidate.go
│   │   ├── interview.go
│   │   ├── application.go
│   │   └── common.go      // Shared types/interfaces
│   ├── user/
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── types.go       // DTOs, request/response types
│   ├── company/
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── types.go
│   ├── job/
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── types.go
│   └── common/
       ├── errors/
       ├── middleware/
       └── utils/
```

Let's see how this looks in code:

```go
// internal/model/user.go
package model

import (
    "time"
    "github.com/google/uuid"
)

type User struct {
    ID        uuid.UUID `json:"id" db:"id"`
    CompanyID uuid.UUID `json:"company_id" db:"company_id"`
    Email     string    `json:"email" db:"email"`
    Name      string    `json:"name" db:"name"`
    Role      string    `json:"role" db:"role"`
    Status    string    `json:"status" db:"status"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// internal/user/types.go
package user

type CreateUserInput struct {
    CompanyID uuid.UUID `json:"company_id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Role      string    `json:"role"`
}

type UpdateUserInput struct {
    Name   *string `json:"name"`
    Role   *string `json:"role"`
    Status *string `json:"status"`
}

type ListUsersInput struct {
    CompanyID uuid.UUID `json:"company_id"`
    Role      *string   `json:"role"`
    Status    *string   `json:"status"`
    Search    *string   `json:"search"`
    Page      int       `json:"page"`
    PerPage   int       `json:"per_page"`
}

// internal/user/repository.go
package user

import (
    "context"
    "github.com/jmoiron/sqlx"
    "github.com/your-org/ats/internal/model"
)

type Repository interface {
    Create(ctx context.Context, user *model.User) error
    GetByID(ctx context.Context, id uuid.UUID) (*model.User, error)
    GetByEmail(ctx context.Context, email string) (*model.User, error)
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id uuid.UUID) error
    List(ctx context.Context, input ListUsersInput) ([]model.User, error)
}

type repository struct {
    db *sqlx.DB
}

func NewRepository(db *sqlx.DB) Repository {
    return &repository{db: db}
}

// internal/user/service.go
package user

import (
    "context"
    "github.com/google/uuid"
    "github.com/your-org/ats/internal/model"
)

type Service interface {
    Create(ctx context.Context, input CreateUserInput) (*model.User, error)
    GetByID(ctx context.Context, id uuid.UUID) (*model.User, error)
    Update(ctx context.Context, id uuid.UUID, input UpdateUserInput) (*model.User, error)
    Delete(ctx context.Context, id uuid.UUID) error
    List(ctx context.Context, input ListUsersInput) ([]model.User, error)
}

type service struct {
    repo Repository
}

func NewService(repo Repository) Service {
    return &service{repo: repo}
}

func (s *service) Create(ctx context.Context, input CreateUserInput) (*model.User, error) {
    user := &model.User{
        ID:        uuid.New(),
        CompanyID: input.CompanyID,
        Email:     input.Email,
        Name:      input.Name,
        Role:      input.Role,
        Status:    "active",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    if err := s.repo.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

The main benefits of this approach:
1. Models are centralized and can be imported by any package
2. Reduces circular dependencies
3. Makes it clear where to find all domain models
4. Easier to maintain model consistency across the application
5. Clearer separation between domain models and DTOs
