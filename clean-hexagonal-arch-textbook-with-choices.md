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

### Decision 5: Testing Strategy

> **Choice required:** How should code that depends on infrastructure (database, external services) be tested?

Each layer has a clear testing approach. The domain, service, and handler layers are straightforward — the repository/storage layer is where the real decision lies.

#### Layer-by-layer (recommended defaults)

**Service tests** — mock the port interfaces (repository, external client). Pure unit tests with no infrastructure:

```go
func TestCreateOrder(t *testing.T) {
    repo := mocks.NewMockOrderRepository(t)
    repo.EXPECT().Create(mock.Anything, mock.Anything).Return(nil)

    svc := service.NewOrderService(repo)
    err := svc.CreateOrder(context.Background(), &domain.Order{ID: uuid.New()})
    assert.NoError(t, err)
}
```

**Handler tests** — mock the service interface. Verify HTTP status codes, response bodies, and request validation:

```go
func TestGetOrderHandler_NotFound(t *testing.T) {
    svc := mocks.NewMockOrderService(t)
    svc.EXPECT().GetOrder(mock.Anything, mock.Anything).Return(nil, ErrNotFound)

    req := httptest.NewRequest("GET", "/orders/123", nil)
    rec := httptest.NewRecorder()
    handler := NewOrderHandler(svc)
    handler.GetOrder(rec, req)

    assert.Equal(t, http.StatusNotFound, rec.Code)
}
```

#### Repository/storage layer — the decision point

This is where infrastructure meets application code. The choice affects test confidence, speed, and complexity.

**Option A — Pure mocks (generated from interfaces).**

A mock generator (mockery, gomock) produces mocks from port interfaces. Service and handler tests already cover this. Storage implementations themselves are not unit-tested.

```go
// Auto-generated mock from ports.OrderRepository
repo := mocks.NewMockOrderRepository(t)
repo.EXPECT().GetByID(mock.Anything, orderID).Return(&order, nil)
```

**Option B — Integration tests with testcontainers.**

A real PostgreSQL container spins up per test suite. Tests run actual SQL/GORM queries against a real database, catching query bugs, migration issues, and GORM behavior that mocks miss.

```go
func TestOrderRepo_Create(t *testing.T) {
    ctx := context.Background()
    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        testcontainers.WithWaitStrategy(
            wait.ForListeningPort("5432/tcp"),
        ),
    )
    t.Cleanup(func() { _ = pg.Terminate(ctx) })

    db := connectGORM(t, pg)
    db.AutoMigrate(&domain.Order{})

    repo := storage.NewOrderRepository(db)
    order := &domain.Order{ID: uuid.New(), Status: "pending"}
    err = repo.Create(ctx, order)
    assert.NoError(t, err)

    found, err := repo.GetByID(ctx, order.ID)
    assert.NoError(t, err)
    assert.Equal(t, order.ID, found.ID)
}
```

**Option C — In-memory fakes.**

Hand-written implementations of port interfaces backed by maps/slices. Faster than testcontainers, more realistic than mocks, but require manual maintenance.

```go
type FakeOrderRepository struct {
    orders map[uuid.UUID]*domain.Order
    mu     sync.RWMutex
}

func (f *FakeOrderRepository) GetByID(_ context.Context, id uuid.UUID) (*domain.Order, error) {
    f.mu.RLock()
    defer f.mu.RUnlock()
    order, ok := f.orders[id]
    if !ok {
        return nil, ErrNotFound
    }
    return order, nil
}
```

| Consideration         | Option A (Mocks)      | Option B (Testcontainers) | Option C (Fakes)        |
|-----------------------|-----------------------|---------------------------|-------------------------|
| Speed                 | Fast                  | Slower (container startup) | Fast                   |
| Confidence            | Low for storage layer | High — real DB behavior    | Medium                  |
| Catches query bugs    | No                    | Yes                        | No                      |
| Catches GORM issues   | No                    | Yes                        | No                      |
| Transaction testing   | No                    | Yes                        | Possible but tricky     |
| Maintenance           | Auto-generated        | Needs Docker in CI         | Manual upkeep           |
| Recommended for       | Small projects, fast CI | Most projects             | Specific integration scenarios |

A common combination: **Option A for service/handler tests + Option B for storage tests**. This gives fast unit tests where mocks suffice and real database confidence where it matters.

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

**Design choice — Router:**

| Option | Notes |
|--------|-------|
| **net/http (stdlib)** | Go 1.22+ added method routing and path parameters (`{id}`). Zero dependencies, good enough for most APIs. |
| **chi** | Lightweight, idiomatic, fully compatible with `net/http`. Supports middleware groups and subrouters. Actively maintained. |
| **Gorilla Mux** | Feature-rich but the original team archived it. Now community-maintained under github.com/gorilla — evaluate maintenance status before adopting. |

#### Example: chi

```go
func (s *Server) setupRoutes() {
    r := chi.NewRouter()
    r.Get("/health", healthCheck)

    r.Route("/api/v1", func(api chi.Router) {
        api.Use(middleware.CorsMiddleware(s.frontendURL))
        api.Use(middleware.RequestID)

        api.Route("/", func(protected chi.Router) {
            protected.Use(middleware.Authenticate)

            protected.Post("/orders", s.order.CreateOrder)
            protected.Get("/orders/{id}", s.order.GetOrder)
        })
    })
}
```

#### Example: net/http (Go 1.22+)

```go
func (s *Server) setupRoutes() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", healthCheck)

    mux.HandleFunc("POST /api/v1/orders", s.order.CreateOrder)
    mux.HandleFunc("GET /api/v1/orders/{id}", s.order.GetOrder)
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

## Error Handling

### Domain Errors

Define domain-level sentinel errors that express business meaning without leaking infrastructure details:

```go
// internal/common/errors.go
var (
    ErrNotFound      = errors.New("not found")
    ErrConflict      = errors.New("conflict")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
)
```

For errors that carry additional context, use a custom type:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s — %s", e.Field, e.Message)
}
```

### Error Wrapping

Add context as errors propagate up through layers using `%w`. This preserves the original error for `errors.Is`/`errors.As` checks while making logs easier to trace:

```go
// Repository layer
func (r *OrderRepo) GetByID(ctx context.Context, id string) (*Order, error) {
    var order Order
    if err := r.db.WithContext(ctx).First(&order, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("order %s: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("querying order %s: %w", id, err)
    }
    return &order, nil
}

// Service layer
func (s *OrderService) GetOrder(ctx context.Context, id string) (*Order, error) {
    order, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("GetOrder: %w", err)
    }
    return order, nil
}
```

### Error Translation at Boundaries

Adapters (repositories, external clients) translate infrastructure-specific errors into domain errors. The domain and service layers never see `sql.ErrNoRows`, `gorm.ErrRecordNotFound`, or HTTP status codes from third-party APIs:

```go
// External client adapter
func (c *PaymentClient) Charge(ctx context.Context, req ChargeRequest) error {
    resp, err := c.http.Post(ctx, "/charge", req)
    if err != nil {
        return fmt.Errorf("payment charge: %w", err)
    }
    if resp.StatusCode == http.StatusConflict {
        return fmt.Errorf("payment charge %s: %w", req.ID, ErrConflict)
    }
    return nil
}
```

### Centralized HTTP Error Mapping

A single function in the API layer maps domain errors to HTTP responses. Handlers return errors; they don't decide status codes:

```go
// internal/api/errors/errors.go
type ErrorResponse struct {
    Error       string `json:"error"`
    Code        int    `json:"code"`
    Description string `json:"description,omitempty"`
}

func HandleError(w http.ResponseWriter, err error) {
    code, message := mapError(err)

    // Log the full error server-side; return a sanitized message to the client
    slog.Error("request failed", "error", err, "status", code)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(ErrorResponse{Error: message, Code: code})
}

func mapError(err error) (int, string) {
    var validationErr *ValidationError
    switch {
    case errors.Is(err, ErrNotFound):
        return http.StatusNotFound, "not found"
    case errors.Is(err, ErrConflict):
        return http.StatusConflict, "conflict"
    case errors.Is(err, ErrUnauthorized):
        return http.StatusUnauthorized, "unauthorized"
    case errors.Is(err, ErrForbidden):
        return http.StatusForbidden, "forbidden"
    case errors.As(err, &validationErr):
        return http.StatusBadRequest, validationErr.Error()
    default:
        return http.StatusInternalServerError, "internal error"
    }
}
```

Handlers stay clean:

```go
func (h *OrderHandler) GetOrder(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    order, err := h.service.GetOrder(r.Context(), id)
    if err != nil {
        apierrors.HandleError(w, err)
        return
    }
    writeJSON(w, http.StatusOK, order)
}
```

**Design choice — Error response detail level:**

| Option | Notes |
|--------|-------|
| **Minimal (recommended)** | Return generic messages (`"not found"`, `"internal error"`). Safest default — never leaks internals. |
| **Descriptive** | Include a `description` field with more context for client developers. Useful during development but requires careful review to avoid leaking sensitive details. |

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

### Distributed Tracing

In a multi-service architecture, a single user request may span several services. Use OpenTelemetry to propagate trace context across service boundaries so you can follow a request end-to-end.

#### Tracer Setup

Initialize a tracer provider at application startup in the bootstrap layer:

```go
// internal/bootstrap/tracing.go
func InitTracer(ctx context.Context, serviceName string) (*sdktrace.TracerProvider, error) {
    exporter, err := newExporter(ctx)
    if err != nil {
        return nil, fmt.Errorf("creating trace exporter: %w", err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
        )),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    return tp, nil
}
```

Shut it down gracefully on application exit to flush pending spans:

```go
defer func() { _ = tp.Shutdown(context.Background()) }()
```

#### HTTP Middleware

Extract incoming trace context (or start a new trace) and create a span per request:

```go
func Tracing(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))
        tracer := otel.Tracer("api")
        ctx, span := tracer.Start(ctx, fmt.Sprintf("%s %s", r.Method, r.URL.Path))
        defer span.End()

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

#### Propagating Context to Outgoing Calls

When calling other services, inject the trace context into outgoing HTTP headers so the downstream service continues the same trace:

```go
func (c *Client) do(ctx context.Context, req *http.Request) (*http.Response, error) {
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
    return c.http.Do(req)
}
```

#### Connecting Logs to Traces

Include the trace ID in structured log entries so logs and traces can be correlated:

```go
func LoggerFromContext(ctx context.Context) *slog.Logger {
    logger := slog.Default()
    if spanCtx := trace.SpanContextFromContext(ctx); spanCtx.HasTraceID() {
        logger = logger.With("trace_id", spanCtx.TraceID().String())
    }
    if requestID, ok := ctx.Value(RequestIDKey).(string); ok {
        logger = logger.With("request_id", requestID)
    }
    return logger
}
```

#### Async / Queue Propagation

For messages published to queues (e.g., RabbitMQ, SQS), inject trace context into message headers on the producer side and extract it on the consumer side, using the same `TextMapPropagator`:

```go
// Producer
carrier := propagation.MapCarrier{}
otel.GetTextMapPropagator().Inject(ctx, carrier)
// attach carrier as message headers

// Consumer
ctx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
```

**Design choice — Tracing backend:**

| Option | Notes |
|--------|-------|
| **Jaeger** | Open source, self-hosted, well-established. Good for teams that want full control. |
| **Grafana Tempo** | Pairs naturally with Grafana/Loki stacks. Cost-efficient object-storage backend. |
| **Cloud-native (Datadog, AWS X-Ray, GCP Cloud Trace)** | Managed, minimal ops overhead. Best when already invested in a cloud platform. |

All options work with the same OpenTelemetry SDK — only the exporter changes.

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
- Repository/storage test approach depends on Decision 5 (Testing Strategy)
- File naming: `*_test.go` for unit tests, `*_handler_test.go` / `*_handler_integration_test.go` for API tests

## Key Technologies

| Component       | Technology       |
|-----------------|------------------|
| Language        | Go               |
| Web Framework   | chi / net/http (stdlib) / Gorilla Mux |
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
