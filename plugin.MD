# Plugin System Documentation
## ATS Plugin Architecture Guide

### Overview
This document describes the plugin architecture of our ATS (Applicant Tracking System). The plugin system allows for extending the core functionality of the ATS without modifying the base code, enabling both internal teams and third-party developers to add new features safely.

### Core Components

#### 1. Plugin Manager
The Plugin Manager is the central component responsible for:
- Managing plugin lifecycle (registration, initialization, start, stop)
- Coordinating hook executions
- Ensuring system stability and plugin isolation
- Managing plugin resources

```go
type PluginManager struct {
    mu      sync.RWMutex
    plugins map[string]Plugin
    hooks   map[string][]Hook
    ctx     PluginContext
    logger  Logger
}
```

#### 2. Plugin Interface
Each plugin must implement the following interface:

```go
type Plugin interface {
    ID() string        // Unique identifier for the plugin
    Name() string      // Human-readable name
    Version() string   // Plugin version
    
    Initialize(ctx PluginContext) error  // Setup plugin
    Start() error                        // Start plugin operations
    Stop() error                         // Cleanup and shutdown
}
```

#### 3. Plugin Context
Plugins receive a context that provides safe access to core services:

```go
type PluginContext interface {
    Logger() Logger
    DB() Database
    Config() Config
    
    RegisterHook(name string, hook Hook) error
    UnregisterHook(name string, hook Hook) error
}
```

#### 4. Hooks
Hooks are extension points where plugins can inject custom behavior:

```go
type Hook interface {
    Execute(ctx context.Context, payload interface{}) error
}
```

### Creating a Plugin

Here's an example of how to create a plugin:

```go
type ExamplePlugin struct {
    id      string
    name    string
    version string
    ctx     PluginContext
}

func (p *ExamplePlugin) ID() string      { return p.id }
func (p *ExamplePlugin) Name() string    { return p.name }
func (p *ExamplePlugin) Version() string { return p.version }

func (p *ExamplePlugin) Initialize(ctx PluginContext) error {
    p.ctx = ctx
    // Register hooks, setup resources
    return nil
}

func (p *ExamplePlugin) Start() error {
    // Start plugin operations
    return nil
}

func (p *ExamplePlugin) Stop() error {
    // Cleanup resources
    return nil
}
```

### Hook Points
Hooks can be registered at various points in the application. Common hook points include:

```go
const (
    // Candidate lifecycle hooks
    HookCandidateCreated = "candidate.created"
    HookCandidateUpdated = "candidate.updated"
    
    // Job lifecycle hooks
    HookJobCreated = "job.created"
    HookJobUpdated = "job.updated"
    
    // Application lifecycle hooks
    HookApplicationStatusChanged = "application.status_changed"
    
    // More hook points can be added as needed
)
```

### Safety Features

1. **Timeout Protection**
```go
// Hooks have a 5-second timeout
select {
case err := <-done:
    return err
case <-time.After(5 * time.Second):
    return ErrHookTimeout
}
```

2. **Panic Recovery**
```go
defer func() {
    if r := recover(); r != nil {
        err = fmt.Errorf("hook panicked: %v", r)
    }
}()
```

3. **Thread Safety**
```go
// All operations are protected by mutex
pm.mu.Lock()
defer pm.mu.Unlock()
```

### Usage Example

1. **Registering a Plugin**
```go
func main() {
    // Create plugin manager
    pm := NewPluginManager(ctx)

    // Create and register plugin
    plugin := &ExamplePlugin{
        id:      "example-plugin",
        name:    "Example Plugin",
        version: "1.0.0",
    }
    
    err := pm.RegisterPlugin(plugin)
    if err != nil {
        log.Fatal(err)
    }
}
```

2. **Creating a Hook**
```go
type CandidateCreatedHook struct{}

func (h *CandidateCreatedHook) Execute(ctx context.Context, payload interface{}) error {
    candidate := payload.(*CandidateData)
    // Implement hook logic
    return nil
}
```

### Best Practices

1. **Plugin Development**
   - Keep plugins focused and single-purpose
   - Handle errors gracefully
   - Clean up resources in Stop()
   - Use timeouts for long-running operations
   - Document hook payloads and requirements

2. **Hook Implementation**
   - Keep hook execution fast
   - Don't block unnecessarily
   - Handle errors appropriately
   - Validate payload data
   - Log important operations

3. **Resource Management**
   - Clean up resources when plugin stops
   - Use context cancellation
   - Implement proper error handling
   - Monitor resource usage

### Error Handling

Plugins should handle errors appropriately:
```go
// Example error handling in a hook
func (h *MyHook) Execute(ctx context.Context, payload interface{}) error {
    // Validate payload
    data, ok := payload.(*MyData)
    if !ok {
        return fmt.Errorf("invalid payload type")
    }

    // Handle errors
    if err := doSomething(data); err != nil {
        return fmt.Errorf("hook operation failed: %w", err)
    }

    return nil
}
```

### Testing Plugins

Example of testing a plugin:
```go
func TestExamplePlugin(t *testing.T) {
    // Create test context
    ctx := NewTestPluginContext()
    
    // Create plugin
    plugin := &ExamplePlugin{
        id:      "test-plugin",
        name:    "Test Plugin",
        version: "1.0.0",
    }
    
    // Test initialization
    if err := plugin.Initialize(ctx); err != nil {
        t.Fatalf("Failed to initialize plugin: %v", err)
    }
    
    // Test hook execution
    if err := plugin.ExecuteHook(context.Background(), testPayload); err != nil {
        t.Errorf("Hook execution failed: %v", err)
    }
}
```
