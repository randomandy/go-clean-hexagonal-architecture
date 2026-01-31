# Go Clean Architecture Guide

A reference architecture for building Go REST APIs using Clean Architecture and Hexagonal Architecture principles.

## Project Structure

```
├── cmd/main.go                    # Application entry point
├── internal/
│   ├── bootstrap/                 # Application initialization & dependency injection
│   │   ├── app.go                 # Application struct and Run()
│   │   ├── config.go              # Configuration loading
│   │   └── services.go            # ServiceContainer and DI wiring
│   ├── config/                    # Shared configuration types
│   ├── api/                       # HTTP routing, middleware, route constants
│   │   ├── server.go              # Server setup and route registration
│   │   ├── routes.go              # Route path constants
│   │   ├── middleware/             # Authentication, CORS, RBAC, request ID middleware
│   │   └── errors/                # Standardized error responses
│   ├── common/                    # Shared utilities
│   │   └── transaction/           # Transaction manager
│   └── <domain>/                  # One directory per business domain
│       ├── domain/                # Core entities and business rules
│       ├── ports/                 # Interface definitions (service + repository)
│       ├── service/               # Business logic implementation
│       ├── storage/               # Database persistence (GORM)
│       │   └── model/             # (Option B only) Separate persistence models
│       ├── dto/                   # Request/response data transfer objects
│       └── api/                   # HTTP handlers
├── external/                      # External service integrations
├── migrations/                    # Database migrations
└── docs/                          # Swagger/OpenAPI documentation
```

---

## Design Decisions

The following choices should be made before implementation starts. Each has trade-offs depending on project size and complexity.

### Decision 1: Domain Model Strategy

> **Choice required:** How do domain entities relate to persistence models?

**Option A — Shared structs (pragmatic).** Domain structs double as GORM models via struct tags. Simpler, less code, but couples the domain layer to GORM.

```go
// domain/order.go — single struct used everywhere
type Order struct {
    ID         uuid.UUID   `gorm:"primaryKey;type:uuid"`
    CustomerID uuid.UUID   `gorm:"type:uuid"`
    Status     OrderStatus
    Total      float64
    CreatedAt  time.Time
    UpdatedAt  time.Time
}
```

**Option B — Separate persistence models (purist).** Domain structs are plain Go with no ORM tags. Storage layer has its own model structs with mappers. True domain isolation, but more boilerplate.

```go
// domain/order.go — pure domain, no infrastructure tags
type Order struct {
    ID         uuid.UUID
    CustomerID uuid.UUID
    Status     OrderStatus
    Total      float64
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

// storage/model/order.go — persistence concern
type OrderModel struct {
    ID         uuid.UUID   `gorm:"primaryKey;type:uuid"`
    CustomerID uuid.UUID   `gorm:"type:uuid"`
    Status     string
    Total      float64
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

func (m *OrderModel) ToDomain() *domain.Order { /* ... */ }
func FromDomain(o *domain.Order) *OrderModel  { /* ... */ }
```

| Consideration         | Option A          | Option B               |
|-----------------------|-------------------|------------------------|
| Boilerplate           | Minimal           | Significant            |
| Domain purity         | Compromised       | Full isolation         |
| ORM swap cost         | Touches all domains| Storage layer only     |
| Recommended for       | Small-medium apps  | Large or long-lived apps|

### Decision 2: Domain Richness

> **Choice required:** Where do business rules live?

**Option A — Anemic domain (logic in services).** Domain structs are data carriers. All rules and state transitions live in the service layer.

```go
// service layer owns the rules
func (s *orderService) CancelOrder(ctx context.Context, id uuid.UUID) error {
    order, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }
    if order.Status != domain.OrderStatusPending {
        return domain.ErrCannotCancel
    }
    order.Status = domain.OrderStatusCancelled
    return s.repo.Update(ctx, order)
}
```

**Option B — Rich domain (logic on entities).** Entities enforce their own invariants. Services orchestrate but don't decide business rules.

```go
// domain/order.go — entity enforces its own invariants
func (o *Order) Cancel() error {
    if o.Status != OrderStatusPending {
        return ErrCannotCancel
    }
    o.Status = OrderStatusCancelled
    return nil
}

// service layer orchestrates
func (s *orderService) CancelOrder(ctx context.Context, id uuid.UUID) error {
    order, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }
    if err := order.Cancel(); err != nil {
        return err
    }
    return s.repo.Update(ctx, order)
}
```

| Consideration         | Option A              | Option B                  |
|-----------------------|-----------------------|---------------------------|
| Simplicity            | Easier to follow      | More upfront design       |
| Invariant safety      | Rules can be bypassed | Co-located with data      |
| Testability           | Need service + mocks  | Test entity in isolation  |
| Recommended for       | CRUD-heavy domains    | Complex business logic    |

Both can coexist: use rich domain for core business entities, anemic for simple CRUD domains.

### Decision 3: Cross-Domain Communication

> **Choice required:** How do domains interact at runtime?

**Option A — Synchronous adapters.** Services call other domains through narrow client adapter interfaces. Simple and predictable.

```go
type PaymentClient interface {
    ChargeCustomer(ctx context.Context, customerID uuid.UUID, amount float64) error
}
```

**Option B — Event-driven (outbox/event bus).** Domains publish events; other domains subscribe. Decoupled but adds infrastructure complexity.

```go
// Domain publishes events
type EventPublisher interface {
    Publish(ctx context.Context, event DomainEvent) error
}

type OrderPlaced struct {
    OrderID    uuid.UUID
    CustomerID uuid.UUID
    Total      float64
    OccurredAt time.Time
}

// Other domain subscribes
type EventHandler interface {
    Handle(ctx context.Context, event DomainEvent) error
}
```

**Option C — Hybrid.** Use synchronous adapters for queries and commands that need immediate consistency. Use events for side effects that can be eventually consistent (notifications, analytics, audit logs).

| Consideration           | Option A          | Option B              | Option C            |
|-------------------------|-------------------|-----------------------|---------------------|
| Complexity              | Low               | High                  | Medium              |
| Consistency             | Immediate         | Eventual              | Mixed               |
| Domain coupling         | Runtime coupling  | Fully decoupled       | Balanced            |
| Infrastructure needs    | None extra        | Message broker/outbox | Message broker      |
| Recommended for         | Most projects     | Microservice path     | Growing monoliths   |

### Decision 4: Repository Interface Granularity

> **Choice required:** How granular should repository interfaces be?

**Option A — Unified CRUD interface.** One interface per domain with all operations.

```go
type OrderRepository interface {
    Create(ctx context.Context, order *domain.Order) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error)
    Update(ctx context.Context, order *domain.Order) error
    List(ctx context.Context, filters OrderFilters) ([]*domain.Order, error)
    Delete(ctx context.Context, id uuid.UUID) error
}
```

**Option B — Segregated read/write interfaces.** Separate reader and writer interfaces. Consumers declare minimal dependencies.

```go
type OrderReader interface {
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error)
    List(ctx context.Context, filters OrderFilters) ([]*domain.Order, error)
}

type OrderWriter interface {
    Create(ctx context.Context, order *domain.Order) error
    Update(ctx context.Context, order *domain.Order) error
    Delete(ctx context.Context, id uuid.UUID) error
}

// Full interface for the storage implementation
type OrderRepository interface {
    OrderReader
    OrderWriter
}
```

| Consideration         | Option A            | Option B                 |
|-----------------------|---------------------|--------------------------|
| Simplicity            | Simpler             | More interfaces          |
| Interface segregation | Broad dependency    | Minimal dependency       |
| Mock complexity       | Larger mocks        | Smaller, focused mocks   |
| Recommended for       | Most projects       | Many consumers with different needs |

---

## Domain Module Layers

Each domain module follows six layers with strict dependency rules. Dependencies point inward: handlers -> service -> ports <- storage.

### 1. Domain (`domain/`)

Core business entities and business rules. No external dependencies (or GORM-only tags if Decision 1 Option A is chosen).

```go
type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusConfirmed OrderStatus = "confirmed"
    OrderStatusCancelled OrderStatus = "cancelled"
)

type Order struct {
    ID         uuid.UUID
    CustomerID uuid.UUID
    Status     OrderStatus
    Total      float64
    Items      []*Item
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

// Rich domain behavior (if Decision 2 Option B)
func (o *Order) Cancel() error {
    if o.Status != OrderStatusPending {
        return ErrCannotCancel
    }
    o.Status = OrderStatusCancelled
    return nil
}

// Domain errors
var (
    ErrCannotCancel = errors.New("order can only be cancelled while pending")
)
```

Key rules:
- Domain structs are the single source of truth for business entities
- Cross-domain references use UUIDs, not foreign-key joins
- Relationships loaded separately (tagged `gorm:"-"` if using shared structs)

### 2. Ports (`ports/`)

Interface definitions that decouple layers. Two files per domain:

**`service.go`** — defines the contract that handlers depend on:
```go
type OrderService interface {
    CreateOrder(ctx context.Context, order *domain.Order) error
    GetOrder(ctx context.Context, id uuid.UUID) (*domain.Order, error)
    UpdateOrder(ctx context.Context, order *domain.Order) error
    ListOrders(ctx context.Context, filters OrderFilters) ([]*domain.Order, error)
}
```

**`repository.go`** — defines the contract that the service depends on:
```go
type OrderRepository interface {
    Create(ctx context.Context, order *domain.Order) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error)
    Update(ctx context.Context, order *domain.Order) error
    List(ctx context.Context, filters OrderFilters) ([]*domain.Order, error)
}
```

Domain-level sentinel errors also live in ports:
```go
var ErrOrderNotFound = errors.New("order not found")

type ErrOrderExists struct {
    Reference string
}

func (e *ErrOrderExists) Error() string {
    return fmt.Sprintf("order already exists with reference '%s'", e.Reference)
}
```

### 3. Service (`service/`)

Business logic. Depends only on port interfaces, never on concrete implementations.

```go
type orderService struct {
    repo          ports.OrderRepository
    paymentClient PaymentClient
}

func NewOrderService(repo ports.OrderRepository, paymentClient PaymentClient) ports.OrderService {
    return &orderService{repo: repo, paymentClient: paymentClient}
}

func (s *orderService) CreateOrder(ctx context.Context, order *domain.Order) error {
    if order.ID == uuid.Nil {
        order.ID = uuid.New()
    }
    order.Status = domain.OrderStatusPending
    return s.repo.Create(ctx, order)
}
```

**Cross-domain communication** uses narrow client adapter interfaces defined within the consuming service package:

```go
// Defined in order service package — not the full PaymentService interface
type PaymentClient interface {
    ChargeCustomer(ctx context.Context, customerID uuid.UUID, amount float64) error
}

// Adapter implementation wired in bootstrap
type paymentClientAdapter struct {
    svc paymentPorts.PaymentService
}

func NewPaymentClientAdapter(svc paymentPorts.PaymentService) PaymentClient {
    return &paymentClientAdapter{svc: svc}
}

func (a *paymentClientAdapter) ChargeCustomer(ctx context.Context, customerID uuid.UUID, amount float64) error {
    return a.svc.Charge(ctx, customerID, amount)
}
```

### 4. Storage (`storage/`)

Repository implementation using GORM. Implements the repository interface from ports.

```go
type orderRepository struct {
    db        *gorm.DB
    txManager transaction.Manager
}

func NewOrderRepository(db *gorm.DB, txManager transaction.Manager) ports.OrderRepository {
    return &orderRepository{db: db, txManager: txManager}
}

func (r *orderRepository) getDB(ctx context.Context) *gorm.DB {
    if tx := r.txManager.GetTxFromContext(ctx); tx != nil {
        return tx
    }
    return r.db
}

func (r *orderRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    var order domain.Order
    if err := r.getDB(ctx).WithContext(ctx).First(&order, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ports.ErrOrderNotFound
        }
        return nil, err
    }
    return &order, nil
}
```

Key pattern: `getDB()` checks context for an active transaction, enabling transparent transaction propagation.

### 5. DTO (`dto/`)

Data transfer objects for API layer. Decouple API contracts from domain entities. Every request DTO implements a `Validate()` method for explicit input validation at the API boundary.

```go
type CreateOrderRequest struct {
    CustomerID uuid.UUID `json:"customer_id"`
    Items      []ItemDTO `json:"items"`
}

func (r *CreateOrderRequest) Validate() error {
    if r.CustomerID == uuid.Nil {
        return errors.New("customer_id is required")
    }
    if len(r.Items) == 0 {
        return errors.New("at least one item is required")
    }
    for i, item := range r.Items {
        if err := item.Validate(); err != nil {
            return fmt.Errorf("item[%d]: %w", i, err)
        }
    }
    return nil
}

func (r *CreateOrderRequest) ToDomain() *domain.Order {
    return &domain.Order{
        ID:         uuid.New(),
        CustomerID: r.CustomerID,
        Status:     domain.OrderStatusPending,
        CreatedAt:  time.Now(),
        UpdatedAt:  time.Now(),
    }
}

type OrderResponse struct {
    ID         uuid.UUID          `json:"id"`
    CustomerID uuid.UUID          `json:"customer_id"`
    Status     domain.OrderStatus `json:"status"`
    Total      float64            `json:"total"`
    CreatedAt  time.Time          `json:"created_at"`
}

func ToResponse(order *domain.Order) *OrderResponse {
    return &OrderResponse{
        ID:         order.ID,
        CustomerID: order.CustomerID,
        Status:     order.Status,
        Total:      order.Total,
        CreatedAt:  order.CreatedAt,
    }
}
```

### 6. API Handlers (`api/`)

HTTP handlers. Parse requests, validate, call services, write responses.

```go
type OrderHandler struct {
    service ports.OrderService
}

func NewOrderHandler(service ports.OrderService) *OrderHandler {
    return &OrderHandler{service: service}
}

// CreateOrder godoc
// @Summary Create a new order
// @Tags Orders
// @Accept json
// @Produce json
// @Param request body dto.CreateOrderRequest true "Order to create"
// @Success 201 {object} dto.OrderResponse
// @Failure 400 {object} errors.ErrorResponse
// @Router /orders [post]
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req dto.CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        errors.WriteErrorResponse(w, http.StatusBadRequest, "Invalid request", err.Error())
        return
    }

    if err := req.Validate(); err != nil {
        errors.WriteErrorResponse(w, http.StatusBadRequest, "Validation failed", err.Error())
        return
    }

    order := req.ToDomain()
    if err := h.service.CreateOrder(r.Context(), order); err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusCreated, dto.ToResponse(order))
}
```

#### Reducing Handler Boilerplate

Handlers follow a repetitive pattern: decode, validate, call service, handle errors, write response. A generic helper eliminates this duplication:

```go
// common/api/handler.go
func Handle[Req Validatable, Resp any](
    w http.ResponseWriter,
    r *http.Request,
    execute func(ctx context.Context, req Req) (Resp, error),
    successCode int,
) {
    var req Req
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        errors.WriteErrorResponse(w, http.StatusBadRequest, "Invalid request", err.Error())
        return
    }
    if err := req.Validate(); err != nil {
        errors.WriteErrorResponse(w, http.StatusBadRequest, "Validation failed", err.Error())
        return
    }
    resp, err := execute(r.Context(), req)
    if err != nil {
        handleServiceError(w, err)
        return
    }
    writeJSON(w, successCode, resp)
}

type Validatable interface {
    Validate() error
}

// Usage in handler — one-liner per endpoint
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    Handle(w, r, func(ctx context.Context, req dto.CreateOrderRequest) (*dto.OrderResponse, error) {
        order := req.ToDomain()
        if err := h.service.CreateOrder(ctx, order); err != nil {
            return nil, err
        }
        return dto.ToResponse(order), nil
    }, http.StatusCreated)
}
```

A shared `handleServiceError` function maps domain errors to HTTP status codes consistently:

```go
func handleServiceError(w http.ResponseWriter, err error) {
    var existsErr interface{ Error() string }
    switch {
    case errors.As(err, &existsErr) && strings.Contains(err.Error(), "already exists"):
        errors.WriteErrorResponse(w, http.StatusConflict, "Resource already exists", err.Error())
    case errors.Is(err, ErrNotFound):
        errors.WriteErrorResponse(w, http.StatusNotFound, "Not found", err.Error())
    default:
        errors.WriteErrorResponse(w, http.StatusInternalServerError, "Internal server error", "")
    }
}
```

## Dependency Injection

All wiring happens in `internal/bootstrap/` — no DI framework, just constructors.

### ServiceContainer

```go
type ServiceContainer struct {
    DB        *gorm.DB
    TxManager transaction.Manager

    // Repositories (stored as interfaces)
    OrderRepo  orderPorts.OrderRepository
    // ...

    // Services (stored as interfaces)
    OrderSvc   orderPorts.OrderService
    // ...
}

func InitializeServices(cfg *AppConfig) (*ServiceContainer, error) {
    c := &ServiceContainer{}

    // 1. Database
    c.DB, _ = config.NewDatabase(cfg.Database)
    c.TxManager = transaction.NewGormManager(c.DB)

    // 2. Repositories
    c.OrderRepo = orderStorage.NewOrderRepository(c.DB, c.TxManager)

    // 3. Services (with cross-domain adapters)
    paymentAdapter := orderService.NewPaymentClientAdapter(c.PaymentSvc)
    c.OrderSvc = orderService.NewOrderService(c.OrderRepo, paymentAdapter)

    return c, nil
}
```

### Application Lifecycle

```go
// cmd/main.go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go handleShutdown(cancel)

    app, err := bootstrap.New()
    if err != nil {
        log.Fatalf("failed to initialize: %v", err)
    }

    if err := app.Run(ctx); err != nil {
        log.Fatalf("application error: %v", err)
    }
}

func handleShutdown(cancel context.CancelFunc) {
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    <-sigChan
    cancel()
}
```

## Transaction Management

Context-based transactions that propagate transparently through repositories.

```go
type Manager interface {
    ExecuteTx(ctx context.Context, fn func(ctx context.Context, tx *gorm.DB) error) error
    GetTxFromContext(ctx context.Context) *gorm.DB
}
```

Usage in a service:

```go
func (s *orderService) PlaceOrder(ctx context.Context, order *domain.Order) error {
    return s.txManager.ExecuteTx(ctx, func(txCtx context.Context, tx *gorm.DB) error {
        if err := s.orderRepo.Create(txCtx, order); err != nil {
            return err // automatic rollback
        }
        if err := s.inventoryRepo.Reserve(txCtx, order.Items); err != nil {
            return err // automatic rollback
        }
        return nil // automatic commit
    })
}
```

Any repository receiving `txCtx` automatically participates in the transaction via `getDB(ctx)`.

## HTTP Layer

### Routing

Gorilla Mux with subrouters for middleware scoping.

```go
func (s *Server) setupRoutes() {
    s.base.HandleFunc("/health", healthCheck)

    api := s.base.PathPrefix("/api/v1").Subrouter()
    api.Use(middleware.CorsMiddleware(s.frontendURL))
    api.Use(middleware.RequestID)

    protected := api.PathPrefix("").Subrouter()
    protected.Use(middleware.Authenticate)

    protected.HandleFunc("/orders", s.order.CreateOrder).Methods("POST")
    protected.HandleFunc("/orders/{id}", s.order.GetOrder).Methods("GET")
}
```

Route paths are defined as constants:
```go
const (
    RouteOrders    = "/orders"
    RouteOrderByID = "/orders/{id}"
)
```

### Middleware

**Request ID** — injects a unique trace ID into every request context for log correlation:
```go
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
        w.Header().Set("X-Request-ID", requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**JWT Authentication** — extracts and validates Bearer token, stores claims in context:
```go
func Authenticate(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractBearerToken(r)
        claims, err := service.ValidateToken(token)
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), UserClaimsKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**Role-Based Access Control** — checks role from JWT claims:
```go
func RequireRole(role string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims := r.Context().Value(UserClaimsKey).(*service.Claims)
            if claims.Role != role && claims.Role != "admin" {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

**CORS** — configurable allowed origin with preflight support.

### Error Responses

Standardized JSON error format:
```go
type ErrorResponse struct {
    Error       string `json:"error"`
    Code        int    `json:"code"`
    Description string `json:"description,omitempty"`
}

func WriteErrorResponse(w http.ResponseWriter, code int, message, description string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(ErrorResponse{Error: message, Code: code, Description: description})
}
```

## Observability

### Structured Logging

Use structured logging (slog, zerolog, or zap) with the request ID from context for log correlation:

```go
func LoggerFromContext(ctx context.Context) *slog.Logger {
    logger := slog.Default()
    if requestID, ok := ctx.Value(RequestIDKey).(string); ok {
        logger = logger.With("request_id", requestID)
    }
    if claims, ok := ctx.Value(UserClaimsKey).(*service.Claims); ok {
        logger = logger.With("user_id", claims.UserID)
    }
    return logger
}

// Usage in service
func (s *orderService) CreateOrder(ctx context.Context, order *domain.Order) error {
    logger := LoggerFromContext(ctx)
    logger.Info("creating order", "order_id", order.ID)
    // ...
}
```

This enables filtering all logs for a single request or user across all layers.

## Configuration

Two-tier: environment variables for local dev, AWS Secrets Manager for production. A `LOCAL_DEV=true` flag bypasses AWS lookups.

```go
func LoadConfig() (*AppConfig, error) {
    cfg := &AppConfig{
        Database: config.DatabaseConfig{
            Host:     getSecretOrEnv("DB_HOST", "localhost"),
            Port:     getSecretOrEnv("DB_PORT", "5432"),
            // ...
        },
    }
    return cfg, nil
}
```

## Database

- PostgreSQL with GORM
- UUID primary keys (`uuid.UUID` with `gorm:"primaryKey;type:uuid"`)
- Connection pooling configured at startup
- Migrations in `migrations/` directory

## API Documentation

Swagger annotations on handlers, served at `/swagger/` endpoint. Generated with `swag init`.

## Testing

- Standard Go testing with testify for assertions
- Service tests mock repository and client adapter interfaces
- Handler tests mock service interfaces
- File naming: `*_test.go` for unit tests, `*_handler_test.go` / `*_handler_integration_test.go` for API tests

## Key Technologies

| Component       | Technology       |
|-----------------|------------------|
| Language        | Go               |
| Web Framework   | Gorilla Mux      |
| ORM             | GORM             |
| Database        | PostgreSQL       |
| Authentication  | JWT              |
| API Docs        | Swagger/OpenAPI  |
| Testing         | testify          |
| Logging         | slog / zerolog   |
| Containerization| Docker           |

## Architectural Principles

1. **Dependency inversion** — all layers depend on interfaces (ports), not implementations
2. **Domain isolation** — domain entities have zero (or minimal) external dependencies
3. **Narrow cross-domain contracts** — services consume small client adapter interfaces, not full service interfaces from other domains
4. **Transparent transactions** — context-based transaction propagation; repositories are unaware of transaction boundaries
5. **DTO separation** — API request/response types are separate from domain entities with explicit conversion and validation
6. **Constructor-based DI** — no framework; all wiring in bootstrap package with explicit initialization order
7. **UUID primary keys** — better for distributed systems
8. **Batch-friendly endpoints** — handlers accept both single items and arrays where appropriate
9. **Explicit validation at the boundary** — every request DTO validates input before conversion to domain
10. **Structured observability** — request IDs propagated via context, structured logging across all layers
