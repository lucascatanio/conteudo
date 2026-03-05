# Arquitetura Hexagonal (Ports & Adapters)

---

## O que é

Arquitetura Hexagonal (também chamada **Ports & Adapters**) é um padrão arquitetural criado por **Alistair Cockburn em 2005**.

O objetivo central é **isolar o núcleo da aplicação** (regras de negócio) de tudo que é externo: banco de dados, APIs, frameworks, mensageria, UI.

A ideia: seu domínio não deve saber **nada** sobre como os dados chegam ou saem. Ele apenas define **o que pode ser feito**.

```
                    [ HTTP / REST ]
                          |
         [ CLI ]    [ Port (Input) ]    [ Message Queue ]
              \          |              /
               \   +-----------+      /
                --> |           | <---
                    |  DOMÍNIO  |
                --> |  (Core)   | <---
               /    |           |    \
              /     +-----------+     \
         [ Tests ]  [ Port (Output) ]  [ Port (Output) ]
                          |                   |
                    [ PostgreSQL ]         [ Redis ]
```

---

## Problema que Resolve

### Sem Arquitetura Hexagonal (acoplamento direto)

```java
// Controller chama Repository diretamente
@RestController
public class OrderController {

    @Autowired
    private OrderRepository orderRepository; // JPA direto no controller

    @PostMapping("/orders")
    public Order create(@RequestBody OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotal(request.getTotal());
        return orderRepository.save(order); // Entity JPA vazando pra fora
    }
}
```

**Problemas:**
- Trocar PostgreSQL por MongoDB → reescrever tudo
- Testar regra de negócio → precisa subir banco de dados
- Lógica de negócio espalhada em controllers e repositories
- Entity JPA vaza para camadas externas (acoplamento de framework)

---

### Com Arquitetura Hexagonal

```java
// Controller depende apenas de uma interface (Port)
@RestController
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase; // interface, não implementação

    @PostMapping("/orders")
    public OrderResponse create(@RequestBody OrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.getUserId(),
            request.getItems()
        );
        Order order = createOrderUseCase.execute(command);
        return OrderResponse.from(order);
    }
}
```

**Resultado:**
- Trocar banco → apenas troca o Adapter
- Testar regra de negócio → usa mock do Port
- Lógica de negócio protegida no Core
- Zero dependência de JPA, Spring ou qualquer framework no domínio

---

## Componentes Principais

### 1. Domain (Core) - O Coração

O que **nunca** muda independente de tecnologia. Contém:

- **Entities:** Objetos com identidade e comportamento
- **Value Objects:** Objetos sem identidade (imutáveis)
- **Domain Services:** Lógica que não pertence a uma única Entity
- **Domain Events:** O que aconteceu no domínio

```java
// Entity - tem identidade (id) e comportamento
public class Order {
    private final OrderId id;
    private final UserId userId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money total;

    // Comportamento no domínio (regra de negócio aqui!)
    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new OrderAlreadyProcessedException(this.id);
        }
        this.status = OrderStatus.CONFIRMED;
        // Publica domain event
        DomainEvents.publish(new OrderConfirmedEvent(this.id));
    }

    public void addItem(Product product, int quantity) {
        if (quantity <= 0) {
            throw new InvalidQuantityException(quantity);
        }
        this.items.add(new OrderItem(product, quantity));
        this.total = calculateTotal();
    }

    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

```java
// Value Object - sem identidade, imutável
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new NegativeMoneyException(amount);
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.EUR);
}
```

> **Regra de ouro:** Classes do Domain **nunca** importam de `org.springframework`, `jakarta.persistence`, ou qualquer framework externo.

---

### 2. Ports (Interfaces) - Os Contratos

São **interfaces Java** que definem o que o domínio precisa ou oferece.

#### Input Ports (Driving Side) - o que a aplicação pode fazer

```java
// Use Case = Input Port
public interface CreateOrderUseCase {
    Order execute(CreateOrderCommand command);
}

public interface GetOrderUseCase {
    Order execute(OrderId orderId);
}

public interface CancelOrderUseCase {
    void execute(OrderId orderId, CancellationReason reason);
}
```

#### Output Ports (Driven Side) - o que a aplicação precisa

```java
// O domínio precisa persistir orders → define o contrato
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

// O domínio precisa enviar notificações → define o contrato
public interface NotificationService {
    void notifyOrderConfirmed(Order order);
}

// O domínio precisa verificar estoque → define o contrato
public interface InventoryService {
    boolean hasStock(ProductId productId, int quantity);
    void reserveStock(ProductId productId, int quantity);
}
```

---

### 3. Application (Use Cases) - Orquestração

Implementa os Input Ports. Coordena o fluxo sem conter regra de negócio.

```java
@Service
@Transactional
public class CreateOrderService implements CreateOrderUseCase {

    private final OrderRepository orderRepository;      // Output Port
    private final InventoryService inventoryService;   // Output Port
    private final NotificationService notificationService; // Output Port

    public CreateOrderService(
        OrderRepository orderRepository,
        InventoryService inventoryService,
        NotificationService notificationService
    ) {
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
    }

    @Override
    public Order execute(CreateOrderCommand command) {
        // 1. Verificar estoque (Output Port)
        command.items().forEach(item -> {
            if (!inventoryService.hasStock(item.productId(), item.quantity())) {
                throw new InsufficientStockException(item.productId());
            }
        });

        // 2. Criar Order (lógica no domínio)
        Order order = Order.create(command.userId(), command.items());

        // 3. Reservar estoque (Output Port)
        command.items().forEach(item ->
            inventoryService.reserveStock(item.productId(), item.quantity())
        );

        // 4. Persistir (Output Port)
        orderRepository.save(order);

        // 5. Notificar (Output Port)
        notificationService.notifyOrderConfirmed(order);

        return order;
    }
}
```

---

### 4. Adapters - As Implementações Externas

Implementam os Output Ports ou adaptam entradas externas para os Input Ports.

#### Input Adapter: REST Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderRestAdapter {

    private final CreateOrderUseCase createOrderUseCase;
    private final GetOrderUseCase getOrderUseCase;
    private final OrderResponseMapper mapper;

    @PostMapping
    public ResponseEntity<OrderResponse> create(
        @Valid @RequestBody CreateOrderRequest request
    ) {
        CreateOrderCommand command = mapper.toCommand(request);
        Order order = createOrderUseCase.execute(command);
        return ResponseEntity.status(201).body(mapper.toResponse(order));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getById(@PathVariable String id) {
        Order order = getOrderUseCase.execute(new OrderId(id));
        return ResponseEntity.ok(mapper.toResponse(order));
    }
}
```

#### Output Adapter: PostgreSQL (JPA)

```java
@Repository
public class OrderJpaAdapter implements OrderRepository {  // implementa o Port

    private final OrderJpaRepository jpaRepository;  // Spring Data JPA
    private final OrderEntityMapper mapper;

    @Override
    public void save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);  // converte Entity JPA → Domain
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return jpaRepository.findByUserId(userId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
}
```

#### Output Adapter: Kafka (Notification)

```java
@Component
public class KafkaNotificationAdapter implements NotificationService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Override
    public void notifyOrderConfirmed(Order order) {
        OrderConfirmedEvent event = new OrderConfirmedEvent(
            order.getId().value(),
            order.getUserId().value(),
            order.getTotal().amount()
        );
        kafkaTemplate.send("order-events", order.getId().value(), event);
    }
}
```

#### Output Adapter: HTTP Client (Inventory)

```java
@Component
public class InventoryHttpAdapter implements InventoryService {

    private final InventoryClient inventoryClient;  // Feign/WebClient

    @Override
    public boolean hasStock(ProductId productId, int quantity) {
        return inventoryClient.checkStock(productId.value(), quantity);
    }

    @Override
    public void reserveStock(ProductId productId, int quantity) {
        inventoryClient.reserve(productId.value(), quantity);
    }
}
```

---

## Estrutura de Pacotes

```
src/main/java/com/company/orders/
│
├── domain/                          # Core - ZERO dependências externas
│   ├── model/
│   │   ├── Order.java               # Entity
│   │   ├── OrderItem.java           # Entity
│   │   ├── OrderId.java             # Value Object
│   │   ├── OrderStatus.java         # Enum
│   │   └── Money.java               # Value Object
│   ├── event/
│   │   ├── OrderConfirmedEvent.java # Domain Event
│   │   └── OrderCancelledEvent.java # Domain Event
│   └── exception/
│       ├── OrderNotFoundException.java
│       └── InsufficientStockException.java
│
├── application/                     # Use Cases + Ports
│   ├── port/
│   │   ├── input/                   # Input Ports (interfaces)
│   │   │   ├── CreateOrderUseCase.java
│   │   │   ├── GetOrderUseCase.java
│   │   │   └── CancelOrderUseCase.java
│   │   └── output/                  # Output Ports (interfaces)
│   │       ├── OrderRepository.java
│   │       ├── NotificationService.java
│   │       └── InventoryService.java
│   ├── service/                     # Implementações dos Input Ports
│   │   ├── CreateOrderService.java
│   │   ├── GetOrderService.java
│   │   └── CancelOrderService.java
│   └── command/                     # DTOs de entrada do domínio
│       ├── CreateOrderCommand.java
│       └── CancelOrderCommand.java
│
└── adapter/                         # Adapters - implementam os Output Ports
    ├── input/
    │   ├── rest/
    │   │   ├── OrderRestAdapter.java
    │   │   ├── dto/
    │   │   │   ├── CreateOrderRequest.java
    │   │   │   └── OrderResponse.java
    │   │   └── mapper/
    │   │       └── OrderRestMapper.java
    │   └── messaging/
    │       └── OrderEventConsumer.java   # Kafka consumer → chama Use Case
    └── output/
        ├── persistence/
        │   ├── OrderJpaAdapter.java       # implementa OrderRepository
        │   ├── entity/
        │   │   └── OrderEntity.java       # JPA Entity (separada do domínio!)
        │   ├── repository/
        │   │   └── OrderJpaRepository.java # Spring Data
        │   └── mapper/
        │       └── OrderPersistenceMapper.java
        ├── notification/
        │   └── KafkaNotificationAdapter.java  # implementa NotificationService
        └── inventory/
            └── InventoryHttpAdapter.java       # implementa InventoryService
```

---

## Testando com Hexagonal

O maior benefício prático: **testar o domínio sem infraestrutura**.

### Teste Unitário Puro do Use Case

```java
class CreateOrderServiceTest {

    // Mocks dos Output Ports (não precisa de banco, Kafka, HTTP!)
    private OrderRepository orderRepository = mock(OrderRepository.class);
    private InventoryService inventoryService = mock(InventoryService.class);
    private NotificationService notificationService = mock(NotificationService.class);

    private CreateOrderUseCase createOrderUseCase = new CreateOrderService(
        orderRepository, inventoryService, notificationService
    );

    @Test
    void shouldCreateOrderSuccessfully() {
        // Given
        var command = new CreateOrderCommand(
            new UserId("user-1"),
            List.of(new OrderItemCommand(new ProductId("prod-1"), 2))
        );
        given(inventoryService.hasStock(new ProductId("prod-1"), 2))
            .willReturn(true);

        // When
        Order order = createOrderUseCase.execute(command);

        // Then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.getItems()).hasSize(1);
        verify(orderRepository).save(order);
        verify(notificationService).notifyOrderConfirmed(order);
    }

    @Test
    void shouldThrowWhenInsufficientStock() {
        // Given
        var command = new CreateOrderCommand(
            new UserId("user-1"),
            List.of(new OrderItemCommand(new ProductId("prod-1"), 100))
        );
        given(inventoryService.hasStock(new ProductId("prod-1"), 100))
            .willReturn(false);

        // When / Then
        assertThatThrownBy(() -> createOrderUseCase.execute(command))
            .isInstanceOf(InsufficientStockException.class);

        verifyNoInteractions(orderRepository); // garantia: não persistiu
        verifyNoInteractions(notificationService); // não notificou
    }
}
```

### Teste de Integração do Adapter

```java
@SpringBootTest
@Testcontainers  // sobe PostgreSQL real em Docker
class OrderJpaAdapterTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired
    private OrderJpaAdapter orderJpaAdapter;

    @Test
    void shouldPersistAndRetrieveOrder() {
        Order order = Order.create(new UserId("user-1"), List.of(...));

        orderJpaAdapter.save(order);

        Optional<Order> found = orderJpaAdapter.findById(order.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getUserId()).isEqualTo(new UserId("user-1"));
    }
}
```

---

## Trade-offs

### Quando Usar

- Sistemas com **regras de negócio complexas** (e-commerce, fintech, ERP)
- Aplicações que **precisam trocar infraestrutura** (ex: migrar Oracle → PostgreSQL)
- Times que praticam **TDD** (testes rápidos sem infraestrutura)
- Microsserviços com **domínio rico**

### Quando NÃO Usar

- **CRUDs simples:** Uma API que só salva e busca dados não precisa dessa complexidade
- **Projetos pequenos/curtos:** Overhead de criar Ports/Adapters para pouca lógica
- **Protótipos e MVPs:** Comece simples, refatore quando necessário

> "Make it work, make it right, make it fast." – Kent Beck

### Comparação com Layered Architecture

| Aspecto | Layered (MVC) | Hexagonal |
|---|---|---|
| **Complexidade inicial** | Baixa | Média-Alta |
| **Testabilidade** | Difícil (banco acoplado) | Excelente (mocks de ports) |
| **Troca de tecnologia** | Difícil | Fácil |
| **Lógica de negócio** | Espalhada | Centralizada no core |
| **Curva de aprendizado** | Baixa | Média |
| **Ideal para** | CRUDs, MVPs | Domínios complexos |

---

## Armadilhas Comuns

### 1. Anemic Domain Model (Anti-Pattern)

```java
// ❌ Domínio sem comportamento - só getters/setters
public class Order {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}

// Lógica de negócio vazou pro Service...
public class OrderService {
    public void confirm(Order order) {
        if (!order.getStatus().equals("PENDING")) throw ...;
        order.setStatus("CONFIRMED");  // domínio anêmico!
    }
}
```

```java
// ✅ Domínio rico - comportamento encapsulado
public class Order {
    public void confirm() {
        if (this.status != OrderStatus.PENDING) throw ...;
        this.status = OrderStatus.CONFIRMED;
    }
}
```

### 2. JPA Entity como Domain Entity

```java
// ❌ Entity JPA no domínio
@Entity
@Table(name = "orders")
public class Order {  // domínio dependendo de JPA!
    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderItem> items;
}
```

```java
// ✅ Separação: Domain Entity vs JPA Entity
// Domain (sem anotações)
public class Order {
    private OrderId id;
    private List<OrderItem> items;
}

// Adapter (com anotações JPA)
@Entity
@Table(name = "orders")
public class OrderEntity {
    @Id
    private String id;

    @OneToMany(...)
    private List<OrderItemEntity> items;
}
```

### 3. Use Case com Lógica de Negócio

```java
// ❌ Regra de negócio no Use Case (orquestrador)
public class CreateOrderService implements CreateOrderUseCase {
    public Order execute(CreateOrderCommand command) {
        // Isso é lógica de negócio, não deve estar aqui!
        if (command.total().compareTo(BigDecimal.valueOf(10000)) > 0) {
            throw new OrderLimitExceededException();
        }
        ...
    }
}

// ✅ Lógica de negócio no domínio
public class Order {
    private static final Money MAX_ORDER_VALUE = Money.of(10000, Currency.EUR);

    public static Order create(UserId userId, List<OrderItem> items) {
        Money total = calculateTotal(items);
        if (total.isGreaterThan(MAX_ORDER_VALUE)) {
            throw new OrderLimitExceededException(total);
        }
        return new Order(OrderId.generate(), userId, items, total);
    }
}
```

---

## Validação de Conhecimento

Antes de avançar, você deve conseguir responder:

1. Qual a diferença entre **Port** e **Adapter**?
2. Por que a Entity JPA deve ser **separada** da Domain Entity?
3. O que é **Anemic Domain Model** e por que é um anti-pattern?
4. Como testar um Use Case **sem subir o Spring**?
5. Em que situação você **NÃO** usaria Hexagonal?

---

## Recursos para Aprofundar

- **Artigo original:** [Hexagonal Architecture - Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- **Livro:** *Clean Architecture* - Robert Martin (Uncle Bob)
- **Livro:** *Domain-Driven Design* - Eric Evans (cap. sobre isolamento do domínio)
- **Livro:** *Implementing Domain-Driven Design* - Vaughn Vernon
- **Projeto de exemplo:** [Spring Hexagonal Sample](https://github.com/thombergs/buckpal)