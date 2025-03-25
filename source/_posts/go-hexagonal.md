---
title: Go 六边形架构实战项目分析
catalog: true
date: 2025-03-20 05:13:11
subtitle: Go Hexagonal In-Depth 
author: Rancho
header-img:
tags:
    - 六边形架构
    - 领域驱动设计
---


# Go六边形架构实战详解

## 一、架构详解

### 1. 六边形架构核心概念

[六边形架构](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))，也称为"端口与适配器"架构，由Alistair Cockburn于2005年提出，其核心理念是：

- **内部与外部分离**：业务逻辑位于六边形中心，外部系统在边界外
- **依赖方向**：依赖指向领域核心，外部依赖内部
- **端口定义边界**：通过端口(接口)定义内外部交互方式
- **适配器连接外部**：适配器实现端口接口，连接外部系统

![Hexagonal Architecture](https://raw.githubusercontent.com/Sairyss/domain-driven-hexagon/master/assets/images/DomainDrivenHexagon.png)

### 2. 项目分层结构

本项目采用四层架构实现六边形架构：

1. **领域层**：核心业务逻辑
2. **应用层**：用例协调、业务流程编排
3. **端口层**：定义内外交互的接口
4. **适配器层**：连接外部技术实现

**目录结构**：

```
go-hexagonal/
├── domain/             # 领域层：业务核心
│   ├── model/          # 领域模型
│   ├── service/        # 领域服务
│   ├── repo/           # 仓储接口(出站端口)
│   ├── event/          # 领域事件
│   ├── aggregate/      # 聚合
│   └── vo/             # 值对象
├── application/        # 应用层：用例协调
│   ├── example/        # 具体用例
│   └── factory.go      # 应用工厂
├── api/                # 入站适配器：API接口
│   ├── http/           # HTTP处理器
│   ├── grpc/           # gRPC处理器
│   ├── dto/            # 数据传输对象
│   └── error_code/     # 错误码定义
├── adapter/            # 出站适配器：基础设施
│   ├── repository/     # 数据存储实现
│   ├── amqp/           # 消息队列实现
│   ├── converter/      # 数据转换器
│   ├── dependency/     # 依赖注入
│   └── job/            # 后台任务
├── config/             # 配置管理
└── tests/              # 测试
```

### 3. 各层详解

#### 领域层

包含业务核心概念和规则：

```go
// domain/model/example.go - 领域模型
type Example struct {
    Id        int       `json:"id"`
    Name      string    `json:"name"`
    Alias     string    `json:"alias"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// domain/repo/example.go - 出站端口(仓储接口)
type IExampleRepo interface {
    Create(ctx context.Context, tx Transaction, example *model.Example) (*model.Example, error)
    GetByID(ctx context.Context, tx Transaction, id int) (*model.Example, error)
    Update(ctx context.Context, tx Transaction, example *model.Example) error
    Delete(ctx context.Context, tx Transaction, id int) error
    FindByName(ctx context.Context, tx Transaction, name string) (*model.Example, error)
}

// domain/service/example.go - 领域服务
type ExampleService struct {
    repo         repo.IExampleRepo
    EventBus     event.EventBus
    exampleCache repo.IExampleCacheRepo
}
```

#### 应用层

协调领域对象完成业务用例：

```go
// application/example/create.go - 用例实现
type CreateUseCase struct {
    exampleService service.IExampleService
    txFactory      repo.TransactionFactory
}

func (u *CreateUseCase) Execute(ctx context.Context, example *model.Example) (*model.Example, error) {
    tx, err := u.txFactory.NewTransaction(ctx)
    if err != nil {
        return nil, err
    }
    
    if err := tx.Begin(); err != nil {
        return nil, err
    }
    
    defer func() {
        if err != nil {
            tx.Rollback()
            return
        }
        tx.Commit()
    }()
    
    return u.exampleService.Create(ctx, tx, example)
}
```

#### 入站适配器

将外部请求转换为应用层调用：

```go
// api/http/example.go - 入站适配器
func CreateExample(c *gin.Context) {
    var req dto.CreateExampleRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        handle.BadRequest(c, err)
        return
    }
    
    example := &model.Example{
        Name:  req.Name,
        Alias: req.Alias,
    }
    
    result, err := appFactory.Example().Create().Execute(c.Request.Context(), example)
    if err != nil {
        handle.Error(c, err)
        return
    }
    
    handle.Success(c, dto.ToExampleResponse(result))
}
```

#### 出站适配器

实现领域层定义的出站端口：

```go
// adapter/repository/mysql/entity/example.go - 出站适配器
type Example struct {
    client *repository.MySQL
}

func (e *Example) GetByID(ctx context.Context, tx repo.Transaction, id int) (*model.Example, error) {
    db := e.getDB(ctx, tx)
    var entity ExampleEntity
    
    if err := db.Where("id = ?", id).First(&entity).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, repo.ErrNotFound
        }
        return nil, err
    }
    
    return mapToModel(&entity), nil
}
```

## 二、依赖倒置原则实现示例

[依赖倒置原则(DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle)是SOLID原则之一，包含两个核心点：

1. 高层模块不应依赖低层模块，两者都应依赖抽象
2. 抽象不应依赖细节，细节应依赖抽象

本项目通过以下方式实现依赖倒置：

### 1. 仓储接口定义在领域层

```go
// domain/repo/example.go - 在领域层定义接口
type IExampleRepo interface {
    Create(ctx context.Context, tx Transaction, example *model.Example) (*model.Example, error)
    // 其他方法...
}

// domain/service/example.go - 领域服务依赖抽象接口
type ExampleService struct {
    repo repo.IExampleRepo  // 依赖接口而非具体实现
    // 其他字段...
}

func NewExampleService(repo repo.IExampleRepo, cache repo.IExampleCacheRepo) *ExampleService {
    return &ExampleService{
        repo:         repo,
        exampleCache: cache,
    }
}
```

### 2. 适配器实现领域层接口

```go
// adapter/repository/mysql/entity/example.go - 实现领域层定义的接口
type Example struct {
    client *repository.MySQL
}

// 确保Example实现IExampleRepo接口
var _ repo.IExampleRepo = (*Example)(nil)

func NewExampleRepository(client *repository.MySQL) repo.IExampleRepo {
    return &Example{client: client}
}

// 实现接口方法
func (e *Example) Create(ctx context.Context, tx repo.Transaction, example *model.Example) (*model.Example, error) {
    // 实现代码...
}
```

### 3. 依赖注入连接层次

```go
// adapter/dependency/wire.go - 依赖注入将实现绑定到接口
func provideExampleRepo(mysqlClient *repository.MySQL) repo.IExampleRepo {
    return mysql.NewExampleRepository(mysqlClient)
}

func provideExampleService(repo repo.IExampleRepo, cache repo.IExampleCacheRepo, eventBus event.EventBus) *service.ExampleService {
    svc := service.NewExampleService(repo, cache)
    svc.EventBus = eventBus
    return svc
}
```

这种设计使**依赖方向从外向内**，领域层不依赖适配器实现，而适配器层依赖领域层接口，从而实现了依赖倒置。

## 三、单一职责与开放封闭原则的体现

### 1. 单一职责原则实现

[单一职责原则(SRP)](https://en.wikipedia.org/wiki/Single-responsibility_principle)：一个类应该只有一个改变的理由。

```go
// 1. 仓储接口专注于数据访问
type IExampleRepo interface {
    Create(ctx context.Context, tx Transaction, example *model.Example) (*model.Example, error)
    // 其他数据访问方法...
}

// 2. 领域服务专注于业务规则
type ExampleService struct {
    // 字段...
}

func (s *ExampleService) Create(ctx context.Context, tx repo.Transaction, example *model.Example) (*model.Example, error) {
    // 业务规则验证
    if example.Name == "" {
        return nil, errors.New("name is required")
    }
    
    // 委托数据访问
    return s.repo.Create(ctx, tx, example)
}

// 3. 用例专注于业务流程协调
type CreateUseCase struct {
    exampleService service.IExampleService
    txFactory      repo.TransactionFactory
}

func (u *CreateUseCase) Execute(ctx context.Context, example *model.Example) (*model.Example, error) {
    // 事务管理和流程协调
    // ...
}

// 4. 控制器专注于HTTP请求处理
func CreateExample(c *gin.Context) {
    // 参数解析和响应处理
    // ...
}
```

### 2. 开放封闭原则实现

[开放封闭原则(OCP)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)：软件实体应对扩展开放，对修改封闭。

```go
// 1. 通过接口扩展存储实现
type IExampleRepo interface {
    // ...定义方法
}

// MySQL实现
type MySQLExampleRepo struct{}

// PostgreSQL实现
type PostgreSQLExampleRepo struct{}

// 2. 事件处理器扩展
type EventHandler interface {
    HandleEvent(ctx context.Context, event Event) error
    InterestedIn(event Event) bool
}

// 可以添加新的处理器而无需修改事件总线
type LoggingEventHandler struct{}
type NotificationEventHandler struct{}

// 3. 工厂方法支持扩展
func (f *RepositoryFactory) NewExampleRepo() repo.IExampleRepo {
    // 可以基于配置选择不同实现，无需修改调用方代码
    switch config.GlobalConfig.DefaultStore {
    case "mysql":
        return mysql.NewExampleRepository(f.clients.MySQL)
    case "postgres":
        return postgre.NewExampleRepository(f.clients.PostgreSQL)
    default:
        return mysql.NewExampleRepository(f.clients.MySQL)
    }
}
```

## 四、依赖注入实现方式

项目使用[Google Wire](https://github.com/google/wire)实现依赖注入，自动生成依赖图：

### 1. Wire定义依赖提供者

```go
// adapter/dependency/wire.go
func InitializeServices() (*service.Services, error) {
    wire.Build(
        provideEventBus,
        provideRedis,
        provideMySQL,
        provideTransactionFactory,
        provideExampleRepo,
        provideExampleCache,
        provideExampleService,
        provideServices,
    )
    return nil, nil
}

// 提供者函数
func provideMySQL() (*repository.MySQL, error) {
    if config.GlobalConfig.MySQL == nil {
        return nil, repository.ErrMissingMySQLConfig
    }

    db, err := repository.OpenGormDB()
    if err != nil {
        return nil, err
    }

    return &repository.MySQL{DB: db}, nil
}

func provideExampleRepo(mysqlClient *repository.MySQL) repo.IExampleRepo {
    return mysql.NewExampleRepository(mysqlClient)
}
```

### 2. 生成依赖注入代码

执行`wire`命令生成依赖初始化代码：

```go
// adapter/dependency/wire_gen.go (自动生成)
func InitializeServices() (*service.Services, error) {
    eventBus := provideEventBus()
    
    mysqlClient, err := provideMySQL()
    if err != nil {
        return nil, err
    }
    
    transactionFactory := provideTransactionFactory(mysqlClient)
    iExampleRepo := provideExampleRepo(mysqlClient)
    
    redisClient, err := provideRedis()
    if err != nil {
        return nil, err
    }
    
    iExampleCacheRepo := provideExampleCache(redisClient)
    exampleService := provideExampleService(iExampleRepo, iExampleCacheRepo, eventBus)
    services := provideServices(exampleService, eventBus)
    
    return services, nil
}
```

### 3. 注入使用示例

```go
// cmd/http_server/main.go
func main() {
    // 初始化配置...
    
    // 使用依赖注入初始化服务
    services, err := dependency.InitializeServices()
    if err != nil {
        log.Logger.Fatal("failed to initialize services", zap.Error(err))
    }
    
    // 创建HTTP服务器并注入依赖
    router := gin.Default()
    api.RegisterRoutes(router, services)
    
    // 启动服务...
}
```

## 五、工厂模式实现

### 1. 工厂模式介绍

[工厂模式](https://en.wikipedia.org/wiki/Factory_method_pattern)封装对象创建过程，降低系统耦合度。本项目实现多级工厂：

### 2. 应用层工厂

```go
// application/factory.go
type ExampleFactory struct {
    exampleService service.IExampleService
    txFactory      repo.TransactionFactory
}

func NewExampleFactory(exampleService service.IExampleService, txFactory repo.TransactionFactory) *ExampleFactory {
    return &ExampleFactory{
        exampleService: exampleService,
        txFactory:      txFactory,
    }
}

// 创建用例实例
func (f *ExampleFactory) Create() *example.CreateUseCase {
    return example.NewCreateUseCase(f.exampleService, f.txFactory)
}

func (f *ExampleFactory) Get() *example.GetUseCase {
    return example.NewGetUseCase(f.exampleService, f.txFactory)
}

// 应用层总工厂
type Factory struct {
    example *ExampleFactory
}

func NewFactory(services *service.Services, txFactory repo.TransactionFactory) *Factory {
    return &Factory{
        example: NewExampleFactory(services.Example, txFactory),
    }
}

func (f *Factory) Example() *ExampleFactory {
    return f.example
}
```

### 3. 仓储工厂

```go
// adapter/repository/factory.go
type RepositoryFactory struct {
    clients *ClientContainer
}

func NewRepositoryFactory(clients *ClientContainer) *RepositoryFactory {
    return &RepositoryFactory{clients: clients}
}

// 创建仓储实例
func (f *RepositoryFactory) NewExampleRepo() repo.IExampleRepo {
    // 根据配置选择合适的存储实现
    switch config.GlobalConfig.DefaultStore {
    case "mysql":
        return mysql.NewExampleRepository(f.clients.MySQL)
    case "postgres":
        return postgre.NewExampleRepository(f.clients.PostgreSQL)
    default:
        return mysql.NewExampleRepository(f.clients.MySQL)
    }
}

// 创建缓存实例
func (f *RepositoryFactory) NewExampleCache() repo.IExampleCacheRepo {
    return redis.NewExampleCache(f.clients.Redis)
}
```

### 4. 使用示例

```go
// api/http/example.go
func CreateExample(c *gin.Context) {
    // 使用工厂获取用例
    createUseCase := appFactory.Example().Create()
    
    // 执行用例
    result, err := createUseCase.Execute(c.Request.Context(), example)
    // ...
}
```

## 六、事件驱动实现

### 1. 领域事件系统

基于发布-订阅模式实现领域事件：

```go
// domain/event/event.go
type Event interface {
    EventName() string
    AggregateID() string
    EventID() string
    OccurredAt() time.Time
}

// 基础事件实现
type BaseEvent struct {
    eventName   string
    aggregateID string
    eventID     string
    occurredAt  time.Time
}

// domain/event/event_bus.go
type EventHandler interface {
    HandleEvent(ctx context.Context, event Event) error
    InterestedIn(event Event) bool
}

type EventBus interface {
    Publish(ctx context.Context, event Event) error
    Subscribe(handler EventHandler)
    Unsubscribe(handler EventHandler)
}

// 内存实现
type InMemoryEventBus struct {
    handlers []EventHandler
    mu       sync.RWMutex
}

func (b *InMemoryEventBus) Publish(ctx context.Context, event Event) error {
    b.mu.RLock()
    defer b.mu.RUnlock()
    
    var wg sync.WaitGroup
    errs := make(chan error, len(b.handlers))
    
    for _, handler := range b.handlers {
        if handler.InterestedIn(event) {
            wg.Add(1)
            go func(h EventHandler) {
                defer wg.Done()
                if err := h.HandleEvent(ctx, event); err != nil {
                    errs <- err
                }
            }(handler)
        }
    }
    
    go func() {
        wg.Wait()
        close(errs)
    }()
    
    for err := range errs {
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

### 2. 定义具体事件

```go
// domain/event/example_events.go
type ExampleCreatedEvent struct {
    BaseEvent
    Example *model.Example `json:"example"`
}

func NewExampleCreatedEvent(aggregateID string, example *model.Example) *ExampleCreatedEvent {
    return &ExampleCreatedEvent{
        BaseEvent: BaseEvent{
            eventName:   "example.created",
            aggregateID: aggregateID,
            eventID:     uuid.New().String(),
            occurredAt:  time.Now(),
        },
        Example: example,
    }
}
```

### 3. 事件处理器

```go
// domain/event/example_handlers.go
type ExampleEventHandler struct{}

func (h *ExampleEventHandler) HandleEvent(ctx context.Context, e Event) error {
    switch evt := e.(type) {
    case *ExampleCreatedEvent:
        log.Logger.Info("example created", zap.Any("example", evt.Example))
        // 执行后续操作，如发送通知等
    case *ExampleUpdatedEvent:
        log.Logger.Info("example updated", zap.Any("example", evt.Example))
    }
    return nil
}

func (h *ExampleEventHandler) InterestedIn(e Event) bool {
    switch e.(type) {
    case *ExampleCreatedEvent, *ExampleUpdatedEvent:
        return true
    default:
        return false
    }
}
```

### 4. 事件发布示例

```go
// domain/service/example.go
func (s *ExampleService) Create(ctx context.Context, tx repo.Transaction, example *model.Example) (*model.Example, error) {
    // 业务逻辑处理
    created, err := s.repo.Create(ctx, tx, example)
    if err != nil {
        return nil, err
    }
    
    // 发布领域事件
    if s.EventBus != nil {
        event := event.NewExampleCreatedEvent(strconv.Itoa(created.Id), created)
        if err := s.EventBus.Publish(ctx, event); err != nil {
            log.Logger.Error("failed to publish example created event", zap.Error(err))
        }
    }
    
    return created, nil
}
```

## 七、配置管理与热更新

### 1. Viper组件介绍

项目使用[Viper](https://github.com/spf13/viper)管理配置，支持多种配置源和热更新：

```go
// config/config.go
type Config struct {
    Env      string      `mapstructure:"env" yaml:"env"`
    App      *App        `mapstructure:"app" yaml:"app"`
    MySQL    *MySQL      `mapstructure:"mysql" yaml:"mysql"`
    Redis    *Redis      `mapstructure:"redis" yaml:"redis"`
    Postgre  *PostgreSQL `mapstructure:"postgre" yaml:"postgre"`
    // 更多配置...
}

// 全局配置实例
var GlobalConfig = &Config{}

// 初始化配置
func InitConfig(configFile string) error {
    v := viper.New()
    v.SetConfigFile(configFile)
    v.AutomaticEnv()
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    
    if err := v.ReadInConfig(); err != nil {
        return fmt.Errorf("failed to read config file: %w", err)
    }
    
    if err := v.Unmarshal(&GlobalConfig); err != nil {
        return fmt.Errorf("failed to unmarshal config: %w", err)
    }
    
    // 配置热更新
    v.WatchConfig()
    v.OnConfigChange(func(e fsnotify.Event) {
        log.Logger.Info("config file changed", zap.String("file", e.Name))
        if err := v.Unmarshal(&GlobalConfig); err != nil {
            log.Logger.Error("failed to unmarshal config on change", zap.Error(err))
            return
        }
        // 触发配置变更事件
        notifyConfigChange()
    })
    
    return nil
}
```

### 2. 配置热更新实现

配置变更通知机制：

```go
// config/config.go
var (
    configChangeSubscribers []func()
    subscriberMu            sync.RWMutex
)

// 订阅配置变更
func SubscribeConfigChange(subscriber func()) {
    subscriberMu.Lock()
    defer subscriberMu.Unlock()
    configChangeSubscribers = append(configChangeSubscribers, subscriber)
}

// 通知所有订阅者
func notifyConfigChange() {
    subscriberMu.RLock()
    defer subscriberMu.RUnlock()
    for _, subscriber := range configChangeSubscribers {
        go subscriber()
    }
}
```

### 3. 配置变更监听示例

```go
// adapter/repository/redis/client.go
func InitRedisClient() {
    // 初始化Redis客户端...
    
    // 订阅配置变更
    config.SubscribeConfigChange(func() {
        // 检查Redis配置是否变更
        newConfig := config.GlobalConfig.Redis
        if reflect.DeepEqual(currentConfig, newConfig) {
            return
        }
        
        log.Logger.Info("redis config changed, reconnecting...")
        
        // 关闭旧连接
        if err := client.Close(); err != nil {
            log.Logger.Error("failed to close redis connection", zap.Error(err))
        }
        
        // 创建新连接
        newClient, err := NewRedisClient(
            fmt.Sprintf("%s:%d", newConfig.Host, newConfig.Port),
            newConfig.Password,
            newConfig.DB,
        )
        if err != nil {
            log.Logger.Error("failed to reconnect redis", zap.Error(err))
            return
        }
        
        // 更新连接和配置
        client = newClient
        currentConfig = newConfig
        log.Logger.Info("redis reconnected successfully")
    })
}
```

## 八、服务启动流程与优雅退出

### 1. 服务启动流程

```go
// cmd/http_server/main.go
func main() {
    // 1. 初始化配置
    if err := config.InitConfig("config/config.yaml"); err != nil {
        panic(err)
    }
    
    // 2. 初始化日志
    if err := log.InitLogger(config.GlobalConfig.Log); err != nil {
        panic(err)
    }
    
    // 3. 初始化存储
    repository.Initialize()
    defer repository.Close(context.Background())
    
    // 4. 初始化依赖
    services, err := dependency.InitializeServices()
    if err != nil {
        log.Logger.Fatal("failed to initialize services", zap.Error(err))
    }
    
    // 5. 构建应用工厂
    appFactory := application.NewFactory(
        services,
        repository.NewTransactionFactory(repository.Clients),
    )
    
    // 6. 创建HTTP服务器
    router := gin.Default()
    api.RegisterRoutes(router, services, appFactory)
    
    server := &http.Server{
        Addr:         config.GlobalConfig.HTTPServer.Addr,
        Handler:      router,
        ReadTimeout:  config.GetDuration(config.GlobalConfig.HTTPServer.ReadTimeout),
        WriteTimeout: config.GetDuration(config.GlobalConfig.HTTPServer.WriteTimeout),
    }
    
    // 7. 启动HTTP服务器与优雅退出
    startServerWithGracefulShutdown(server)
}
```

### 2. 优雅退出实现

```go
// cmd/http_server/main.go
func startServerWithGracefulShutdown(server *http.Server) {
    // 在独立协程中启动服务器
    go func() {
        log.Logger.Info("server starting", zap.String("addr", server.Addr))
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Logger.Fatal("server failed to start", zap.Error(err))
        }
    }()
    
    // 等待中断信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Logger.Info("shutting down server...")
    
    // 创建超时上下文进行关闭
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // 关闭所有连接
    if err := server.Shutdown(ctx); err != nil {
        log.Logger.Fatal("server forced to shutdown", zap.Error(err))
    }
    
    // 关闭存储连接
    repository.Close(ctx)
    
    log.Logger.Info("server exited gracefully")
}
```

## 九、接口分离原则实现

[接口分离原则(ISP)](https://en.wikipedia.org/wiki/Interface_segregation_principle)：客户端不应依赖它不需要的接口。

```go
// 1. 细粒度的仓储接口
type IExampleRepo interface {
    Create(ctx context.Context, tx Transaction, example *model.Example) (*model.Example, error)
    GetByID(ctx context.Context, tx Transaction, id int) (*model.Example, error)
    Update(ctx context.Context, tx Transaction, example *model.Example) error
    Delete(ctx context.Context, tx Transaction, id int) error
    FindByName(ctx context.Context, tx Transaction, name string) (*model.Example, error)
}

// 专用缓存接口
type IExampleCacheRepo interface {
    Get(ctx context.Context, id int) (*model.Example, error)
    Set(ctx context.Context, example *model.Example) error
    Delete(ctx context.Context, id int) error
}

// 2. 事务接口
type Transaction interface {
    Begin() error
    Commit() error
    Rollback() error
    GetDB(ctx context.Context) interface{}
}

// 3. 事件处理接口
type EventHandler interface {
    HandleEvent(ctx context.Context, event Event) error
    InterestedIn(event Event) bool
}

// 4. 专用应用服务接口
type IExampleService interface {
    Create(ctx context.Context, tx repo.Transaction, example *model.Example) (*model.Example, error)
    GetByID(ctx context.Context, tx repo.Transaction, id int) (*model.Example, error)
    Update(ctx context.Context, tx repo.Transaction, example *model.Example) error
    Delete(ctx context.Context, tx repo.Transaction, id int) error
    FindByName(ctx context.Context, tx repo.Transaction, name string) (*model.Example, error)
}
```

## 十、最佳实践总结

### 1. 关注点分离

- **领域层**专注业务规则，不依赖技术实现
- **应用层**协调业务流程，处理事务
- **适配器层**处理外部交互，隔离技术细节

### 2. 接口设计原则

- 小而专注的接口，遵循接口分离原则
- 按领域概念划分接口，而非技术概念
- 接口定义在依赖方（领域层），实现在被依赖方（适配器层）

### 3. 错误处理策略

- 领域错误与技术错误分离
- 使用自定义错误类型表达业务规则违反
- 适配器层转换底层错误为领域错误

```go
// domain/repo/error.go
var (
    ErrNotFound          = errors.New("entity not found")
    ErrDuplicate         = errors.New("entity already exists")
    ErrInvalidEntity     = errors.New("invalid entity")
    ErrTransactionFailed = errors.New("transaction failed")
)

// 错误转换示例
func (e *Example) GetByID(ctx context.Context, tx repo.Transaction, id int) (*model.Example, error) {
    if err := db.Where("id = ?", id).First(&entity).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, repo.ErrNotFound // 转换为领域错误
        }
        return nil, err
    }
    // ...
}
```

### 4. 测试策略

- **单元测试**：使用模拟对象测试每一层
- **集成测试**：使用[testcontainers-go](https://github.com/testcontainers/testcontainers-go)创建隔离测试环境
- **端到端测试**：验证完整功能流程

```go
// tests/mysql.go - 容器化测试环境
func SetupMySQLContainer(t *testing.T) *MySQLConfig {
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "mysql:8.0",
        ExposedPorts: []string{"3306/tcp"},
        Env: map[string]string{
            "MYSQL_USER":          "test",
            "MYSQL_PASSWORD":      "test",
            "MYSQL_ROOT_PASSWORD": "test",
            "MYSQL_DATABASE":      "test_db",
        },
        WaitingFor: wait.ForLog("port: 3306  MySQL Community Server - GPL"),
    }
    
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    
    // 配置连接...
    return &MySQLConfig{Container: container, DSN: dsn}
}
```

### 5. 配置管理最佳实践

- 使用结构化配置，便于类型检查
- 支持多种配置源：文件、环境变量、命令行
- 实现配置热更新与变更通知机制

### 6. 依赖注入原则

- 显式声明依赖，便于测试和替换
- 使用Wire自动化依赖图构建
- 将初始化逻辑与运行时逻辑分离

## 总结

Go六边形架构实现了业务逻辑与技术细节的有效分离，具有以下优势：

1. **高内聚低耦合**：各层通过接口交互，降低依赖
2. **可测试性**：业务逻辑可独立测试，不依赖外部系统
3. **灵活性**：易于切换技术实现，如数据库或消息队列
4. **可维护性**：清晰的责任边界，易于理解和扩展

该架构特别适合复杂业务应用，尤其是需要长期维护和演进的系统。通过采用这种模式，项目可以更好地应对业务变化，同时保持技术实现的灵活性。


> 本项目已在GitHub开源[Go-Hexagonal](https://github.com/RanchoCooper/go-hexagonal)，欢迎社区贡献和改进建议。
