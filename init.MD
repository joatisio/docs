# Project Initiation
```go
// Project Structure
ats/
├── cmd/
│   └── api/
│       └── main.go        // Application entry point using cobra
├── config/
│   ├── config.go          // Viper configuration
│   └── config.yaml        // Configuration file
├── internal/
│   ├── auth/             // Authentication package
│   │   ├── jwt.go        // JWT operations
│   │   ├── middleware.go // Gin auth middleware
│   │   └── service.go    // Auth service
│   ├── model/           // Domain models
│   ├── user/            // User management
│   ├── notification/    // Email & SMS services
│   ├── monitoring/      // Metrics & monitoring
│   └── common/          // Shared utilities
└── pkg/                 // Public packages
```

Let's go through each major component:

1. **Configuration with Viper**:
```go
// config/config.go
package config

import (
    "github.com/spf13/viper"
)

type Config struct {
    Server struct {
        Port int
        Mode string
    }
    Database struct {
        Host     string
        Port     int
        Name     string
        User     string
        Password string
    }
    JWT struct {
        Secret        string
        AccessExpiry  int // in minutes
        RefreshExpiry int // in days
    }
    Email struct {
        SMTPHost     string
        SMTPPort     int
        SMTPUser     string
        SMTPPassword string
    }
    SMS struct {
        Provider string
        APIKey   string
    }
    Monitoring struct {
        PrometheusPort int
    }
}

func LoadConfig() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("./config")
    
    var config Config
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}
```

2. **JWT Authentication**:
```go
// internal/auth/jwt.go
package auth

import (
    "time"
    "github.com/golang-jwt/jwt/v4"
)

type TokenPair struct {
    AccessToken  string
    RefreshToken string
}

type Claims struct {
    UserID    string `json:"user_id"`
    CompanyID string `json:"company_id"`
    Role      string `json:"role"`
    jwt.StandardClaims
}

type JWTService struct {
    config *config.Config
    logger *zap.Logger
}

func (s *JWTService) GenerateTokenPair(userID, companyID, role string) (*TokenPair, error) {
    // Generate access token
    accessToken, err := s.generateAccessToken(userID, companyID, role)
    if err != nil {
        return nil, err
    }

    // Generate refresh token
    refreshToken, err := s.generateRefreshToken(userID)
    if err != nil {
        return nil, err
    }

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

// internal/auth/middleware.go
func AuthMiddleware(jwtService *JWTService) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }

        claims, err := jwtService.ValidateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("company_id", claims.CompanyID)
        c.Set("role", claims.Role)
        
        c.Next()
    }
}
```

3. **Notification Service**:
```go
// internal/notification/service.go
package notification

import (
    "github.com/go-mail/mail"
    "go.uber.org/zap"
)

type NotificationService struct {
    emailClient *mail.Dialer
    smsClient   SMSClient
    logger      *zap.Logger
}

func (s *NotificationService) SendVerificationEmail(email, code string) error {
    // Implementation
}

func (s *NotificationService) SendVerificationSMS(phone, code string) error {
    // Implementation
}

func (s *NotificationService) SendPasswordResetEmail(email, resetLink string) error {
    // Implementation
}
```

4. **Database with GORM**:
```go
// internal/model/base.go
package model

import (
    "time"
    "gorm.io/gorm"
)

type Base struct {
    ID        string         `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// internal/model/user.go
type User struct {
    Base
    Email           string `gorm:"uniqueIndex"`
    Password        string
    Name            string
    Phone           string
    EmailVerified   bool
    PhoneVerified   bool
    Status          string
    CompanyID       string
    Role            string
    LastLogin       *time.Time
}

// internal/database/db.go
func SetupDatabase(config *config.Config) (*gorm.DB, error) {
    dsn := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%d",
        config.Database.Host,
        config.Database.User,
        config.Database.Password,
        config.Database.Name,
        config.Database.Port,
    )

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    if err != nil {
        return nil, err
    }

    // Auto migrate models
    if err := db.AutoMigrate(
        &model.User{},
        // other models...
    ); err != nil {
        return nil, err
    }

    return db, nil
}
```

5. **Monitoring with Prometheus**:
```go
// internal/monitoring/metrics.go
package monitoring

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

type Metrics struct {
    ActiveUsers          prometheus.Gauge
    RequestDuration      prometheus.Histogram
    TotalRequests       *prometheus.CounterVec
    DatabaseErrors      *prometheus.CounterVec
    AuthenticationErrors *prometheus.CounterVec
}

func NewMetrics(reg prometheus.Registerer) *Metrics {
    m := &Metrics{
        ActiveUsers: promauto.NewGauge(prometheus.GaugeOpts{
            Name: "ats_active_users",
            Help: "The total number of active users",
        }),
        RequestDuration: promauto.NewHistogram(prometheus.HistogramOpts{
            Name: "ats_http_request_duration_seconds",
            Help: "Duration of HTTP requests",
        }),
        TotalRequests: promauto.NewCounterVec(
            prometheus.CounterOpts{
                Name: "ats_http_requests_total",
                Help: "Total number of HTTP requests",
            },
            []string{"method", "endpoint", "status"},
        ),
        // ... other metrics
    }

    return m
}

// Middleware for request metrics
func MetricsMiddleware(metrics *Metrics) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        
        c.Next()
        
        duration := time.Since(start)
        metrics.RequestDuration.Observe(duration.Seconds())
        metrics.TotalRequests.WithLabelValues(
            c.Request.Method,
            c.FullPath(),
            strconv.Itoa(c.Writer.Status()),
        ).Inc()
    }
}
```

6. **Application Entry Point with Cobra**:
```go
// cmd/api/main.go
package main

import (
    "github.com/spf13/cobra"
    "go.uber.org/zap"
)

var rootCmd = &cobra.Command{
    Use:   "ats",
    Short: "ATS - Applicant Tracking System",
    Run: func(cmd *cobra.Command, args []string) {
        // Load configuration
        config, err := config.LoadConfig()
        if err != nil {
            log.Fatal(err)
        }

        // Initialize logger
        logger, _ := zap.NewProduction()
        defer logger.Sync()

        // Setup database
        db, err := database.SetupDatabase(config)
        if err != nil {
            logger.Fatal("failed to connect to database", zap.Error(err))
        }

        // Initialize services
        authService := auth.NewService(db, config)
        notificationService := notification.NewService(config)
        metrics := monitoring.NewMetrics(prometheus.DefaultRegisterer)

        // Setup Gin router
        router := gin.New()
        router.Use(
            gin.Recovery(),
            MetricsMiddleware(metrics),
        )

        // Setup routes
        setupRoutes(router, authService, notificationService, metrics)

        // Start server
        logger.Info("starting server", zap.Int("port", config.Server.Port))
        if err := router.Run(fmt.Sprintf(":%d", config.Server.Port)); err != nil {
            logger.Fatal("failed to start server", zap.Error(err))
        }
    },
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```
