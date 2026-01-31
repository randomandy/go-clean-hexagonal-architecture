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
│   │   ├── middleware/            # Authentication, CORS, RBAC, request ID middleware
│   │   └── errors/                # Standardized error responses
│   ├── common/                    # Shared utilities
│   │   ├── errors/                # Domain error types
│   │   └── transaction/           # Transaction manager
│   └── <domain>/                  # One directory per business domain
│       ├── domain/                # Core entities and business rules
│       ├── ports/                 # Interface definitions (service + repository)
│       ├── service/               # Business logic implementation
│       ├── storage/               # Database persistence (GORM)
│       │   └── model/             # Separate persistence models (Decision 1 Option B)
│       ├── dto/                   # Request/response data transfer objects
│       └── api/                   # HTTP handlers
├── external/                      # External service integrations
├── migrations/                    # Database migrations
└── docs/                          # Swagger/OpenAPI documentation
```

> This tree shows the superset of all directories. Some are conditional on design decisions (noted inline).

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

**Domain tests** — if using rich domain models (Decision 2 Option B), domain entities are the easiest and most valuable code to test. No mocks, no setup — pure logic:

```go
func TestOrder_Cancel_WhenPending(t *testing.T) {
    order := &domain.Order{Status: domain.OrderStatusPending}

    err := order.Cancel()

    assert.NoError(t, err)
    assert.Equal(t, domain.OrderStatusCancelled, order.Status)
}

func TestOrder_Cancel_WhenAlreadyConfirmed(t *testing.T) {
    order := &domain.Order{Status: domain.OrderStatusConfirmed}

    err := order.Cancel()

    assert.ErrorIs(t, err, domain.ErrCannotCancel)
    assert.Equal(t, domain.OrderStatusConfirmed, order.Status) // unchanged
}

func TestOrder_AddItem_CalculatesTotal(t *testing.T) {
    order := &domain.Order{}

    order.AddItem(&domain.Item{Price: 100, Quantity: 2})
    order.AddItem(&domain.Item{Price: 50, Quantity: 1})

    assert.Equal(t, 250.0, order.Total)
}
```

This is why rich domain models improve testability — invariant logic is tested without infrastructure.

**Service tests** — mock the port interfaces (repository, external client). Pure unit tests with no infrastructure:

```go
func TestCreateOrder(t *testing.T) {
    repo := mocks.NewMockOrderRepository(t)
    repo.EXPECT().Create(mock.Anything, mock.Anything).Return(nil)

    svc := service.NewOrderService(repo)
    err := svc.CreateOrder(context.Background(), &domain.Order{ID: uuid.New()})
    require.NoError(t, err)
}
```

**Handler tests** — mock the service interface. Verify HTTP status codes, response bodies, and request validation:

```go
func TestGetOrderHandler_NotFound(t *testing.T) {
    svc := mocks.NewMockOrderService(t)
    svc.EXPECT().GetOrder(mock.Anything, mock.Anything).Return(nil, commonerrors.ErrNotFound)

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
    require.NoError(t, err)
    t.Cleanup(func() { _ = pg.Terminate(ctx) })

    db := connectGORM(t, pg)

    // Note: AutoMigrate with domain types assumes Decision 1 Option A (shared structs).
    // For Option B, migrate with storage/model types instead.
    db.AutoMigrate(&domain.Order{})

    repo := storage.NewOrderRepository(db, transaction.NewGormManager(db))
    order := &domain.Order{ID: uuid.New(), Status: domain.OrderStatusPending}

    err = repo.Create(ctx, order)
    require.NoError(t, err)

    found, err := repo.GetByID(ctx, order.ID)
    require.NoError(t, err)
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
        return nil, commonerrors.ErrNotFound
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

// Domain-specific errors (business rule violations)
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

**`errors.go`** — domain-specific error types that carry business context:
```go
// Use for errors that carry domain-specific context beyond simple not-found/conflict
type ErrInvalidStateTransition struct {
    From, To OrderStatus
}

func (e *ErrInvalidStateTransition) Error() string {
    return fmt.Sprintf("cannot transition order from %s to %s", e.From, e.To)
}

// For simple not-found cases, repositories wrap common.ErrNotFound with context:
//   return nil, fmt.Errorf("order %s: %w", id, commonerrors.ErrNotFound)
```

### 3. Service (`service/`)

Business logic. Depends only on port interfaces, never on concrete implementations.

```go
type orderService struct {
    repo          ports.OrderRepository
    paymentClient PaymentClient
    txManager     transaction.Manager
}

func NewOrderService(
    repo ports.OrderRepository,
    paymentClient PaymentClient,
    txManager transaction.Manager,
) ports.OrderService {
    return &orderService{repo: repo, paymentClient: paymentClient, txManager: txManager}
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
    db *gorm.DB
}

func NewOrderRepository(db *gorm.DB) ports.OrderRepository {
    return &orderRepository{db: db}
}

func (r *orderRepository) getDB(ctx context.Context) *gorm.DB {
    if tx := transaction.GetTx(ctx); tx != nil {
        return tx
    }
    return r.db
}

func (r *orderRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    var order domain.Order
    if err := r.getDB(ctx).WithContext(ctx).First(&order, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("order %s: %w", id, commonerrors.ErrNotFound)
        }
        return nil, fmt.Errorf("querying order %s: %w", id, err)
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

// ToDomain converts request data only — business defaults (ID, status, timestamps)
// are set by the service layer.
func (r *CreateOrderRequest) ToDomain() *domain.Order {
    return &domain.Order{
        CustomerID: r.CustomerID,
        Items:      toItemsDomain(r.Items),
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

#### Batch-Friendly Endpoints

For operations that benefit from bulk processing, define batch request/response types alongside the single-item versions. The handler accepts an array, validates each item, and delegates to a service method that operates on the full batch:

```go
// dto/order.go
type CreateOrdersBatchRequest struct {
    Orders []CreateOrderRequest `json:"orders"`
}

func (r *CreateOrdersBatchRequest) Validate() error {
    if len(r.Orders) == 0 {
        return errors.New("at least one order is required")
    }
    for i, o := range r.Orders {
        if err := o.Validate(); err != nil {
            return fmt.Errorf("orders[%d]: %w", i, err)
        }
    }
    return nil
}

func (r *CreateOrdersBatchRequest) ToDomain() []*domain.Order {
    orders := make([]*domain.Order, len(r.Orders))
    for i, o := range r.Orders {
        orders[i] = o.ToDomain()
    }
    return orders
}

type CreateOrdersBatchResponse struct {
    Orders []*OrderResponse `json:"orders"`
}
```

The service and repository layers follow the same pattern — a `CreateBatch` method that operates on a slice:

```go
// ports/service.go
type OrderService interface {
    CreateOrder(ctx context.Context, order *domain.Order) error
    CreateOrders(ctx context.Context, orders []*domain.Order) error
    // ...
}

// ports/repository.go
type OrderRepository interface {
    Create(ctx context.Context, order *domain.Order) error
    CreateBatch(ctx context.Context, orders []*domain.Order) error
    // ...
}
```

This avoids N+1 round-trips for bulk operations while keeping the single-item path simple.

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
// @Failure 400 {object} apierrors.ErrorResponse
// @Router /orders [post]
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req dto.CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        apierrors.HandleError(w, r, &commonerrors.ValidationError{Field: "body", Message: err.Error()})
        return
    }

    if err := req.Validate(); err != nil {
        apierrors.HandleError(w, r, &commonerrors.ValidationError{Field: "body", Message: err.Error()})
        return
    }

    order := req.ToDomain()
    if err := h.service.CreateOrder(r.Context(), order); err != nil {
        apierrors.HandleError(w, r, err)
        return
    }

    writeJSON(w, http.StatusCreated, dto.ToResponse(order))
}
```

#### Reducing Handler Boilerplate

Handlers that accept a request body follow a repetitive pattern: decode, validate, call service, handle errors, write response. A generic helper eliminates this duplication for POST/PUT endpoints:

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
        apierrors.HandleError(w, r, &commonerrors.ValidationError{Field: "body", Message: err.Error()})
        return
    }
    if err := req.Validate(); err != nil {
        apierrors.HandleError(w, r, &commonerrors.ValidationError{Field: "body", Message: err.Error()})
        return
    }
    resp, err := execute(r.Context(), req)
    if err != nil {
        apierrors.HandleError(w, r, err)
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

// GET endpoints that read path/query parameters don't use Handle — they remain explicit
// (see the GetOrder handler example in the Error Handling section).
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
    c.OrderRepo = orderStorage.NewOrderRepository(c.DB)

    // 3. Services (with cross-domain adapters)
    paymentAdapter := orderService.NewPaymentClientAdapter(c.PaymentSvc)
    c.OrderSvc = orderService.NewOrderService(c.OrderRepo, paymentAdapter, c.TxManager)

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

Context-based transactions that propagate transparently. The interface hides the ORM implementation from consumers.

```go
// internal/common/transaction/manager.go
package transaction

type Manager interface {
    // ExecuteTx runs fn within a transaction. If fn returns an error, the
    // transaction is rolled back; otherwise it commits.
    ExecuteTx(ctx context.Context, fn func(ctx context.Context) error) error
}
```

Implementation (GORM-specific, lives in infrastructure):

```go
// internal/common/transaction/gorm.go
package transaction

type ctxKey struct{}

type gormManager struct {
    db *gorm.DB
}

func NewGormManager(db *gorm.DB) Manager {
    return &gormManager{db: db}
}

func (m *gormManager) ExecuteTx(ctx context.Context, fn func(ctx context.Context) error) error {
    return m.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        txCtx := context.WithValue(ctx, ctxKey{}, tx)
        return fn(txCtx)
    })
}

// GetTx retrieves the transaction from context. Called by repositories.
func GetTx(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(ctxKey{}).(*gorm.DB); ok {
        return tx
    }
    return nil
}
```

Repositories use a helper to get the active connection:

```go
func (r *orderRepository) getDB(ctx context.Context) *gorm.DB {
    if tx := transaction.GetTx(ctx); tx != nil {
        return tx
    }
    return r.db
}
```

Usage in a service — no GORM types visible:

```go
func (s *orderService) PlaceOrder(ctx context.Context, order *domain.Order) error {
    return s.txManager.ExecuteTx(ctx, func(txCtx context.Context) error {
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

### Handling Partial Failures

When a transaction fails, the error should indicate which operation failed. Repositories wrap errors with context (see Error Handling), so the service receives errors like:

- `"reserving inventory for order abc-123: insufficient stock for SKU xyz"`
- `"creating order abc-123: conflict"`

The service can inspect these with `errors.Is()` / `errors.As()` if it needs to handle specific failures differently, or simply propagate to the handler for logging and response.

## HTTP Layer

### Routing

**Design choice — Router:**

| Option | Notes |
|--------|-------|
| **net/http (stdlib)** | Go 1.22+ added method routing and path parameters (`{id}`). Zero dependencies, good enough for most APIs. |
| **chi** | Lightweight, idiomatic, fully compatible with `net/http`. Supports middleware groups and subrouters. Actively maintained. |

> **Note:** Gorilla Mux was a popular choice historically but was archived by the original team. It is now community-maintained under github.com/gorilla. For new projects, chi or the stdlib are safer bets.

#### Example: chi

```go
func (s *Server) setupRoutes() {
    r := chi.NewRouter()

    // Global middleware
    r.Use(middleware.RequestID)
    r.Use(middleware.Recoverer)

    r.Get("/health", healthCheck)

    r.Route("/api/v1", func(api chi.Router) {
        api.Use(middleware.CorsMiddleware(s.frontendURL))

        // Public routes
        api.Post("/auth/login", s.auth.Login)

        // Protected routes
        api.Group(func(protected chi.Router) {
            protected.Use(middleware.Authenticate(s.tokenValidator))

            protected.Post("/orders", s.order.CreateOrder)
            protected.Get("/orders/{id}", s.order.GetOrder)
            protected.Put("/orders/{id}", s.order.UpdateOrder)

            // Admin-only routes
            protected.Group(func(admin chi.Router) {
                admin.Use(middleware.RequireRole("admin"))
                admin.Delete("/orders/{id}", s.order.DeleteOrder)
            })
        })
    })

    s.router = r
}
```

#### Example: net/http (Go 1.22+)

The stdlib router doesn't have built-in middleware chaining. Compose middleware manually or use a helper:

```go
func (s *Server) setupRoutes() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", healthCheck)

    // Protected endpoints need manual middleware composition
    mux.Handle("POST /api/v1/orders",
        middleware.RequestID(
            middleware.Authenticate(s.tokenValidator)(
                http.HandlerFunc(s.order.CreateOrder),
            ),
        ),
    )
    mux.Handle("GET /api/v1/orders/{id}",
        middleware.RequestID(
            middleware.Authenticate(s.tokenValidator)(
                http.HandlerFunc(s.order.GetOrder),
            ),
        ),
    )

    s.handler = mux
}
```

For APIs with many routes needing the same middleware stack, chi or a similar router is more practical.

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

**JWT Authentication** — extracts and validates Bearer token, stores claims in context. The validator is injected to avoid coupling middleware to a concrete service package:
```go
type TokenValidator interface {
    ValidateToken(token string) (*Claims, error)
}

func Authenticate(validator TokenValidator) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractBearerToken(r)
            claims, err := validator.ValidateToken(token)
            if err != nil {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            ctx := context.WithValue(r.Context(), UserClaimsKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

**Role-Based Access Control** — checks role from JWT claims:
```go
func RequireRole(role string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := r.Context().Value(UserClaimsKey).(*Claims)
            if !ok || claims == nil {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
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

Define generic sentinel errors in a shared package. Domains add context through wrapping rather than defining domain-specific variants like `ErrOrderNotFound`.

```go
// internal/common/errors/errors.go
package commonerrors

import "errors"

// Sentinel errors for HTTP-mappable categories
var (
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
)

// ValidationError carries field-level detail
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// ExternalServiceError wraps failures from third-party integrations
type ExternalServiceError struct {
    Service   string // "payment", "identity", "shipping"
    Operation string // "charge", "verify", "get_status"
    Retryable bool
    Cause     error
}

func (e *ExternalServiceError) Error() string {
    return fmt.Sprintf("%s.%s: %v", e.Service, e.Operation, e.Cause)
}

func (e *ExternalServiceError) Unwrap() error {
    return e.Cause
}
```

### Error Wrapping Guidelines

Wrap errors only when adding **meaningful context** the caller doesn't have. Function names alone add noise — they're already in stack traces.

```go
// ❌ Redundant — function name adds nothing
func (s *OrderService) GetOrder(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    order, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("GetOrder: %w", err)  // Don't do this
    }
    return order, nil
}

// ✅ No wrap needed — repo already added context
func (s *OrderService) GetOrder(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    return s.repo.GetByID(ctx, id)
}

// ✅ Meaningful context — adds info the repo didn't have
func (s *OrderService) GetOrderForCustomer(ctx context.Context, customerID, orderID uuid.UUID) (*domain.Order, error) {
    order, err := s.repo.GetByID(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("customer %s: %w", customerID, err)
    }
    if order.CustomerID != customerID {
        return nil, fmt.Errorf("order %s: %w", orderID, commonerrors.ErrNotFound)
    }
    return order, nil
}
```

### Error Translation at Boundaries

Adapters (repositories, external clients) translate infrastructure-specific errors into domain errors. The domain and service layers never see `gorm.ErrRecordNotFound`, `sql.ErrNoRows`, or HTTP status codes from third-party APIs.

```go
// Repository translates GORM errors
func (r *orderRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    var order domain.Order
    if err := r.getDB(ctx).WithContext(ctx).First(&order, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("order %s: %w", id, commonerrors.ErrNotFound)
        }
        return nil, fmt.Errorf("querying order %s: %w", id, err)
    }
    return &order, nil
}

// External client translates HTTP errors
func (c *PaymentClient) Charge(ctx context.Context, customerID uuid.UUID, amount float64) error {
    resp, err := c.http.Post(ctx, "/charge", chargeRequest{CustomerID: customerID, Amount: amount})
    if err != nil {
        return &commonerrors.ExternalServiceError{
            Service:   "payment",
            Operation: "charge",
            Retryable: isRetryable(err),
            Cause:     err,
        }
    }
    switch resp.StatusCode {
    case http.StatusConflict:
        return fmt.Errorf("charge customer %s: %w", customerID, commonerrors.ErrConflict)
    case http.StatusUnprocessableEntity:
        return &commonerrors.ValidationError{Field: "amount", Message: "insufficient funds"}
    }
    return nil
}
```

### Centralized HTTP Error Mapping

A single function maps domain errors to HTTP responses. Handlers delegate to this function rather than deciding status codes themselves.

```go
// internal/api/errors/handler.go
package apierrors

type ErrorResponse struct {
    Code      string `json:"code"`                 // Stable code for integrators
    Message   string `json:"message"`              // Human-readable
    RequestID string `json:"request_id,omitempty"` // For support correlation
}

func HandleError(w http.ResponseWriter, r *http.Request, err error) {
    code, status, message := mapError(err)
    requestID, _ := r.Context().Value(middleware.RequestIDKey).(string)

    // Log full error chain server-side
    slog.ErrorContext(r.Context(), "request failed",
        "error", err,
        "status", status,
        "code", code,
        "request_id", requestID,
    )

    // Return sanitized response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Code:      code,
        Message:   message,
        RequestID: requestID,
    })
}

func mapError(err error) (code string, status int, message string) {
    var validationErr *commonerrors.ValidationError
    var externalErr *commonerrors.ExternalServiceError

    switch {
    case errors.Is(err, commonerrors.ErrNotFound):
        return "NOT_FOUND", http.StatusNotFound, "Resource not found"
    case errors.Is(err, commonerrors.ErrConflict):
        return "CONFLICT", http.StatusConflict, "Resource already exists"
    case errors.Is(err, commonerrors.ErrUnauthorized):
        return "UNAUTHORIZED", http.StatusUnauthorized, "Authentication required"
    case errors.Is(err, commonerrors.ErrForbidden):
        return "FORBIDDEN", http.StatusForbidden, "Access denied"
    case errors.As(err, &validationErr):
        return "VALIDATION_ERROR", http.StatusBadRequest, validationErr.Error()
    case errors.As(err, &externalErr):
        // Don't leak external service details to clients
        return "SERVICE_UNAVAILABLE", http.StatusServiceUnavailable, "Downstream service error"
    default:
        return "INTERNAL_ERROR", http.StatusInternalServerError, "Internal error"
    }
}
```

Handlers stay clean:

```go
func (h *OrderHandler) GetOrder(w http.ResponseWriter, r *http.Request) {
    id, err := uuid.Parse(chi.URLParam(r, "id"))
    if err != nil {
        apierrors.HandleError(w, r, &commonerrors.ValidationError{Field: "id", Message: "invalid UUID"})
        return
    }

    order, err := h.service.GetOrder(r.Context(), id)
    if err != nil {
        apierrors.HandleError(w, r, err)
        return
    }

    writeJSON(w, http.StatusOK, dto.ToResponse(order))
}
```

## Observability

### Structured Logging

Use structured logging (slog, zerolog, or zap) with the request ID and trace ID from context for log correlation:

```go
func LoggerFromContext(ctx context.Context) *slog.Logger {
    logger := slog.Default()
    if spanCtx := trace.SpanContextFromContext(ctx); spanCtx.HasTraceID() {
        logger = logger.With("trace_id", spanCtx.TraceID().String())
    }
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

// newExporter returns a span exporter for the chosen backend.
// Swap the implementation depending on the tracing backend (see design choice below).
// Examples:
//   Jaeger:       jaeger.New(jaeger.WithCollectorEndpoint(jaeger.WithEndpoint(url)))
//   OTLP/gRPC:    otlptracegrpc.New(ctx, otlptracegrpc.WithEndpoint(url))
//   Stdout (dev): stdouttrace.New(stdouttrace.WithPrettyPrint())
func newExporter(ctx context.Context) (sdktrace.SpanExporter, error) { /* ... */ }

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

Extract incoming trace context (or start a new trace), create a span per request, and record errors:

```go
func Tracing(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))
        tracer := otel.Tracer("api")
        ctx, span := tracer.Start(ctx, fmt.Sprintf("%s %s", r.Method, r.URL.Path))
        defer span.End()

        // Wrap response writer to capture status code
        wrapped := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
        next.ServeHTTP(wrapped, r.WithContext(ctx))

        // Record error status on span
        if wrapped.status >= 400 {
            span.SetStatus(codes.Error, http.StatusText(wrapped.status))
        }
    })
}

type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(code int) {
    r.status = code
    r.ResponseWriter.WriteHeader(code)
}
```

For errors caught in handlers or services, record them explicitly:

```go
import "go.opentelemetry.io/otel/codes"

func (s *orderService) CreateOrder(ctx context.Context, order *domain.Order) error {
    span := trace.SpanFromContext(ctx)

    if err := s.repo.Create(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to create order")
        return err
    }
    return nil
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
- Domain tests for rich domain models (pure logic, no dependencies)
- Service tests mock repository and client adapter interfaces
- Handler tests mock service interfaces
- Repository/storage test approach depends on Decision 5 (Testing Strategy)
- File naming: `*_test.go` for unit tests, `*_handler_test.go` / `*_handler_integration_test.go` for API tests

## Key Technologies

| Component       | Technology       |
|-----------------|------------------|
| Language        | Go               |
| Web Framework   | chi / net/http (stdlib) |
| ORM             | GORM             |
| Database        | PostgreSQL       |
| Authentication  | JWT              |
| API Docs        | Swagger/OpenAPI  |
| Testing         | testify, testcontainers |
| Logging         | slog / zerolog   |
| Tracing         | OpenTelemetry    |
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
10. **Structured observability** — request IDs and trace IDs propagated via context, structured logging across all layers
11. **Error translation at boundaries** — adapters translate infrastructure errors to domain errors; handlers map domain errors to HTTP responses
