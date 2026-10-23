# Spring Boot Code Patterns

Copy these shapes and replace `Product` with the feature's entity.

## BaseEntity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Long version;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    public Long getId() { return id; }
    public Long getVersion() { return version; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

```java
@Configuration
@EnableJpaAuditing
class JpaAuditingConfig {}
```

## DTOs

```java
public record CreateProductRequest(
        @NotBlank @Size(max = 64) String sku,
        @NotBlank @Size(max = 200) String name,
        @NotNull @DecimalMin("0.00") @Digits(integer = 17, fraction = 2) BigDecimal unitPrice,
        @NotNull Long supplierId) {}

public record ProductResponse(
        Long id, String sku, String name, BigDecimal unitPrice,
        int quantity, Long supplierId, Instant createdAt, Instant updatedAt) {}
```

Serialize `BigDecimal` as a string globally with
`spring.jackson.generator.write-bigdecimal-as-plain: true` plus a
`Jackson2ObjectMapperBuilderCustomizer` that registers `ToStringSerializer` for `BigDecimal`.

## PageResponse

```java
public record PageResponse<T>(List<T> content, int page, int size, long totalElements, int totalPages) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(page.getContent(), page.getNumber(), page.getSize(),
                page.getTotalElements(), page.getTotalPages());
    }
}
```

## Controller

```java
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products")
class ProductController {

    private final ProductService service;

    ProductController(ProductService service) {
        this.service = service;
    }

    @GetMapping
    PageResponse<ProductResponse> list(@ParameterObject @PageableDefault(size = 20, sort = "createdAt",
            direction = Sort.Direction.DESC) Pageable pageable) {
        return PageResponse.from(service.list(pageable));
    }

    @GetMapping("/{id}")
    ProductResponse get(@PathVariable Long id) {
        return service.get(id);
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest request,
                                           UriComponentsBuilder uri) {
        ProductResponse created = service.create(request);
        URI location = uri.path("/api/v1/products/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    void delete(@PathVariable Long id) {
        service.delete(id);
    }
}
```

## Service

```java
@Service
@Transactional(readOnly = true)
public class ProductService {

    private static final Logger log = LoggerFactory.getLogger(ProductService.class);

    private final ProductRepository repository;
    private final SupplierService supplierService;
    private final ProductMapper mapper;

    public ProductService(ProductRepository repository, SupplierService supplierService, ProductMapper mapper) {
        this.repository = repository;
        this.supplierService = supplierService;
        this.mapper = mapper;
    }

    public Page<ProductResponse> list(Pageable pageable) {
        return repository.findAll(pageable).map(mapper::toResponse);
    }

    public ProductResponse get(Long id) {
        return mapper.toResponse(find(id));
    }

    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        if (repository.existsBySku(request.sku())) {
            throw new ConflictException("Product with SKU '%s' already exists".formatted(request.sku()));
        }
        Supplier supplier = supplierService.getReference(request.supplierId());
        Product saved = repository.save(mapper.toEntity(request, supplier));
        log.info("Product created id={} sku={}", saved.getId(), saved.getSku());
        return mapper.toResponse(saved);
    }

    @Transactional
    public void delete(Long id) {
        repository.delete(find(id));
        log.info("Product deleted id={}", id);
    }

    Product find(Long id) {
        return repository.findById(id).orElseThrow(() -> new NotFoundException("Product", id));
    }
}
```

## Exceptions

```java
public class NotFoundException extends RuntimeException {
    public NotFoundException(String resource, Object id) {
        super("%s with id %s was not found".formatted(resource, id));
    }
}

public class ConflictException extends RuntimeException {
    public ConflictException(String message) { super(message); }
}

public class BusinessRuleException extends RuntimeException {
    public BusinessRuleException(String message) { super(message); }
}
```

## GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(NotFoundException.class)
    ProblemDetail handleNotFound(NotFoundException ex) {
        return problem(HttpStatus.NOT_FOUND, "Resource not found", ex.getMessage());
    }

    @ExceptionHandler({ConflictException.class, ObjectOptimisticLockingFailureException.class})
    ProblemDetail handleConflict(RuntimeException ex) {
        String detail = ex instanceof ConflictException ? ex.getMessage()
                : "The resource was modified by someone else. Reload and try again.";
        return problem(HttpStatus.CONFLICT, "Conflict", detail);
    }

    @ExceptionHandler(BusinessRuleException.class)
    ProblemDetail handleBusinessRule(BusinessRuleException ex) {
        return problem(HttpStatus.UNPROCESSABLE_ENTITY, "Business rule violated", ex.getMessage());
    }

    // Method security throws this; without a handler the catch-all below would turn it into a 500.
    @ExceptionHandler(AccessDeniedException.class)
    ProblemDetail handleAccessDenied(AccessDeniedException ex) {
        return problem(HttpStatus.FORBIDDEN, "Forbidden", "You do not have permission to perform this action");
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        return problem(HttpStatus.INTERNAL_SERVER_ERROR, "Internal error", "An unexpected error occurred");
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        ProblemDetail body = problem(HttpStatus.BAD_REQUEST, "Validation failed", "One or more fields are invalid");
        body.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(e -> Map.of("field", e.getField(), "message", String.valueOf(e.getDefaultMessage())))
                .toList());
        return ResponseEntity.badRequest().body(body);
    }

    private static ProblemDetail problem(HttpStatus status, String title, String detail) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, detail);
        pd.setTitle(title);
        return pd;
    }
}
```

## application.yml

```yaml
spring:
  application:
    name: <app>-api
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 5        # free-tier databases allow few connections
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
  data:
    web:
      pageable:
        max-page-size: 100
server:
  port: ${PORT:8080}
  shutdown: graceful
  forward-headers-strategy: framework
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true
app:
  security:
    jwt-secret: ${JWT_SECRET}
    access-token-ttl: 30m
  cors:
    allowed-origins: ${FRONTEND_ORIGIN:http://localhost:5173}
```

`application-local.yml` adds `spring.flyway.locations: classpath:db/migration,classpath:db/seed`
and DEBUG logging for the app's own package.
