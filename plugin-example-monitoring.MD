# Prometheus Monitoring Plugin
```go
// internal/plugin/monitoring/plugin.go

package monitoring

import (
    "context"
    "net/http"
    "time"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "go.uber.org/zap"
    "your-org/ats/internal/plugin"
)

type MonitoringPlugin struct {
    id       string
    name     string
    version  string
    ctx      plugin.PluginContext
    logger   *zap.Logger
    metrics  *Metrics
    registry *prometheus.Registry
    server   *http.Server
}

func New() plugin.Plugin {
    return &MonitoringPlugin{
        id:      "monitoring",
        name:    "System Monitoring",
        version: "1.0.0",
    }
}

// Metrics holds all Prometheus metrics
type Metrics struct {
    // HTTP metrics
    RequestDuration *prometheus.HistogramVec
    RequestTotal    *prometheus.CounterVec
    RequestActive   prometheus.Gauge

    // Database metrics
    DBConnectionsActive prometheus.Gauge
    DBQueryDuration    *prometheus.HistogramVec
    DBErrors           *prometheus.CounterVec

    // Business metrics
    ActiveUsers         prometheus.Gauge
    JobsCreated        prometheus.Counter
    CandidatesCreated  prometheus.Counter
    InterviewsScheduled prometheus.Counter

    // System metrics
    MemoryUsage    prometheus.Gauge
    GoroutineCount prometheus.Gauge
    CPUUsage       prometheus.Gauge
}

func newMetrics(reg *prometheus.Registry) *Metrics {
    m := &Metrics{
        // HTTP metrics
        RequestDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name: "http_request_duration_seconds",
                Help: "Duration of HTTP requests",
                Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
            },
            []string{"method", "path", "status"},
        ),
        RequestTotal: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "http_requests_total",
                Help: "Total number of HTTP requests",
            },
            []string{"method", "path", "status"},
        ),
        RequestActive: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "http_requests_active",
                Help: "Number of active HTTP requests",
            },
        ),

        // Database metrics
        DBConnectionsActive: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "db_connections_active",
                Help: "Number of active database connections",
            },
        ),
        DBQueryDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name: "db_query_duration_seconds",
                Help: "Duration of database queries",
                Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
            },
            []string{"query_type"},
        ),
        DBErrors: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "db_errors_total",
                Help: "Total number of database errors",
            },
            []string{"operation"},
        ),

        // Business metrics
        ActiveUsers: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "active_users",
                Help: "Number of active users",
            },
        ),
        JobsCreated: prometheus.NewCounter(
            prometheus.CounterOpts{
                Name: "jobs_created_total",
                Help: "Total number of jobs created",
            },
        ),
        CandidatesCreated: prometheus.NewCounter(
            prometheus.CounterOpts{
                Name: "candidates_created_total",
                Help: "Total number of candidates created",
            },
        ),
        InterviewsScheduled: prometheus.NewCounter(
            prometheus.CounterOpts{
                Name: "interviews_scheduled_total",
                Help: "Total number of interviews scheduled",
            },
        ),

        // System metrics
        MemoryUsage: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "memory_usage_bytes",
                Help: "Current memory usage",
            },
        ),
        GoroutineCount: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "goroutine_count",
                Help: "Number of goroutines",
            },
        ),
        CPUUsage: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "cpu_usage_percent",
                Help: "CPU usage percentage",
            },
        ),
    }

    // Register all metrics
    reg.MustRegister(
        m.RequestDuration,
        m.RequestTotal,
        m.RequestActive,
        m.DBConnectionsActive,
        m.DBQueryDuration,
        m.DBErrors,
        m.ActiveUsers,
        m.JobsCreated,
        m.CandidatesCreated,
        m.InterviewsScheduled,
        m.MemoryUsage,
        m.GoroutineCount,
        m.CPUUsage,
    )

    return m
}

// Plugin interface implementation
func (p *MonitoringPlugin) ID() string      { return p.id }
func (p *MonitoringPlugin) Name() string    { return p.name }
func (p *MonitoringPlugin) Version() string { return p.version }

func (p *MonitoringPlugin) Initialize(ctx plugin.PluginContext) error {
    p.ctx = ctx
    p.logger = ctx.Logger()
    p.registry = prometheus.NewRegistry()
    p.metrics = newMetrics(p.registry)

    // Register hooks for monitoring
    if err := p.registerHooks(); err != nil {
        return err
    }

    return nil
}

func (p *MonitoringPlugin) Start() error {
    // Start metrics server
    p.server = &http.Server{
        Addr:    ":9090",
        Handler: promhttp.HandlerFor(p.registry, promhttp.HandlerOpts{}),
    }

    go func() {
        if err := p.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            p.logger.Error("metrics server error", zap.Error(err))
        }
    }()

    // Start system metrics collection
    go p.collectSystemMetrics()

    return nil
}

func (p *MonitoringPlugin) Stop() error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    return p.server.Shutdown(ctx)
}

func (p *MonitoringPlugin) registerHooks() error {
    // Register HTTP request hook
    if err := p.ctx.RegisterHook("http.request", p.httpRequestHook()); err != nil {
        return err
    }

    // Register database query hook
    if err := p.ctx.RegisterHook("db.query", p.dbQueryHook()); err != nil {
        return err
    }

    // Register business event hooks
    hooks := []struct {
        name string
        hook plugin.Hook
    }{
        {"user.login", p.userLoginHook()},
        {"job.created", p.jobCreatedHook()},
        {"candidate.created", p.candidateCreatedHook()},
        {"interview.scheduled", p.interviewScheduledHook()},
    }

    for _, h := range hooks {
        if err := p.ctx.RegisterHook(h.name, h.hook); err != nil {
            return err
        }
    }

    return nil
}

// Hook implementations
func (p *MonitoringPlugin) httpRequestHook() plugin.Hook {
    return plugin.HookFn(func(ctx context.Context, payload interface{}) error {
        data := payload.(HTTPRequestData)
        p.metrics.RequestTotal.WithLabelValues(data.Method, data.Path, data.Status).Inc()
        p.metrics.RequestDuration.WithLabelValues(data.Method, data.Path, data.Status).
            Observe(data.Duration.Seconds())
        return nil
    })
}

func (p *MonitoringPlugin) dbQueryHook() plugin.Hook {
    return plugin.HookFn(func(ctx context.Context, payload interface{}) error {
        data := payload.(DBQueryData)
        p.metrics.DBQueryDuration.WithLabelValues(data.QueryType).
            Observe(data.Duration.Seconds())
        if data.Error != nil {
            p.metrics.DBErrors.WithLabelValues(data.QueryType).Inc()
        }
        return nil
    })
}

// System metrics collection
func (p *MonitoringPlugin) collectSystemMetrics() {
    ticker := time.NewTicker(15 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            
            p.metrics.MemoryUsage.Set(float64(m.Alloc))
            p.metrics.GoroutineCount.Set(float64(runtime.NumGoroutine()))
            
            // CPU usage (simplified)
            if cpu, err := getCPUUsage(); err == nil {
                p.metrics.CPUUsage.Set(cpu)
            }
        }
    }
}

// Helper types
type HTTPRequestData struct {
    Method   string
    Path     string
    Status   string
    Duration time.Duration
}

type DBQueryData struct {
    QueryType string
    Duration  time.Duration
    Error     error
}
```

Usage example:

```go
// In your main application
func main() {
    // Initialize plugin manager
    pluginManager := plugin.NewPluginManager(pluginContext)

    // Register monitoring plugin
    monitoringPlugin := monitoring.New()
    if err := pluginManager.RegisterPlugin(monitoringPlugin); err != nil {
        log.Fatal(err)
    }

    // Use the monitoring middleware
    router.Use(monitoringMiddleware(monitoringPlugin.Metrics()))
}

// Middleware to collect HTTP metrics
func monitoringMiddleware(metrics *monitoring.Metrics) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        metrics.RequestActive.Inc()
        
        c.Next()
        
        metrics.RequestActive.Dec()
        duration := time.Since(start)
        
        metrics.RequestDuration.WithLabelValues(
            c.Request.Method,
            c.FullPath(),
            strconv.Itoa(c.Writer.Status()),
        ).Observe(duration.Seconds())
    }
}
```

This monitoring plugin provides:

1. HTTP metrics:
   - Request duration
   - Request count
   - Active requests

2. Database metrics:
   - Query duration
   - Error count
   - Active connections

3. Business metrics:
   - Active users
   - Jobs created
   - Candidates created
   - Interviews scheduled

4. System metrics:
   - Memory usage
   - Goroutine count
   - CPU usage

The metrics are exposed on `:9090/metrics` in Prometheus format.
