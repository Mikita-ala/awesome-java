# Аннотации в Java — экосистема и практики
Дата: 2025-09-06

> Отдельный файл с практическими разделами по Spring MVC/WebFlux/AOP, Spring Boot и Data, транзакциям, MapStruct, OpenAPI/Swagger, Micronaut, Quarkus, а также чек-листами по nullability и валидации

---

## 1. Spring Core: стереотипы и конфигурация

**Стереотипы**
- `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`  
  **Когда**: маркируете класс для сканирования и DI  
  **Зачем**: явная роль класса, авто-обнаружение, спец-обработка `@Repository` исключений

**Конфигурация**
- `@Configuration` + `@Bean` — фабрики бинов в коде  
- `@Primary`, `@Qualifier` — выбрать реализацию при нескольких кандидатах  
- `@Scope("singleton"|"prototype"|web)` и `@Lazy` — управляют жизненным циклом  
- `@Profile("dev","prod")` — альтернативные конфигурации  
- `@Conditional` и семейство Spring Boot условий: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `@ConditionalOnResource`  
  **Когда**: пишете автоконфигурацию или включаете фичи по флагам конфигурации

**Внедрение**
- Предпочтительно **конструкторное внедрение** без поля `@Autowired`  
- Допустимо `@Inject` (JSR-330) вместо `@Autowired` для переносимости

**Частые паттерны**
```java
@Configuration
class HttpConfig {
  @Bean Client client(HttpProperties p) { return new Client(p); } // детерминированное создание бина без точки в комментарии
}

@Service
class UserService {
  private final Repo repo;
  UserService(Repo repo) { this.repo = repo; } // явная зависимость через конструктор без точки в комментарии
}
```

---

## 2. Spring MVC / WebFlux

**Маршрутизация**
- `@RequestMapping` на классе, короткие алиасы на методах: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`

**Параметры и тело**
- `@PathVariable`, `@RequestParam`, `@RequestHeader`, `@CookieValue`, `@RequestBody`  
- `@ResponseBody` и `@ResponseStatus` — управление телом и статусом ответа

**Валидация**
- `@Valid` на параметрах для Bean Validation  
- `@Validated` на классе — включает групповые проверки и валидацию методов через AOP

**Исключения**
- `@ExceptionHandler` и `@ControllerAdvice` — единая обработка ошибок

**WebFlux**
- Те же аннотации, но типы `Mono`/`Flux`, не блокируем I/O

```java
@RestController
@RequestMapping("/users")
class UsersController {
  @GetMapping("/<built-in function id>")
  public UserDto get(@PathVariable long id) { /* ... */ return null; } // контракт понятен из сигнатуры без точки в комментарии

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public UserDto create(@Valid @RequestBody CreateUser cmd) { /* ... */ return null; }
}
```

---

## 3. Транзакции и Spring Data

**Транзакции**
- `@Transactional` из Spring Tx или `jakarta.transaction.Transactional`
  - На уровне **сервиса**, не на репозитории  
  - Настраивайте `readOnly`, `propagation`, `isolation` по необходимости  
  - Учтите прокси: вызов через `this.method()` не применит аспект

**Spring Data JPA**
- `@EnableJpaRepositories` — включить репозитории  
- `@Query`, `@Modifying`, `@Lock`, `@EntityGraph` — тонкая настройка запросов и графов
- В DTO-проекциях используйте явные конструкторы и `@JsonProperty` если нужно стабильное API

```java
@Service
class OrderService {
  private final OrderRepo repo;
  @Transactional
  public void place(OrderCmd cmd) { /* атомарность операций */ } // транзакция живет на уровне сервиса без точки в комментарии
}
```

---

## 4. Spring Boot: автоконфигурация и свойства

- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- `@ConfigurationProperties(prefix="app")` и, при необходимости, `@ConstructorBinding`  
- `@ConfigurationPropertiesScan` — сканирование POJO-свойств
- Условия автоконфигурации: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`  
  **Когда**: пишете стартер или модуль, который должен включаться только при наличии зависимостей или флагов

```java
@ConfigurationProperties(prefix = "app.http")
record HttpProperties(Duration timeout, URI baseUrl) {} // стабильный контракт конфигурации без точки в комментарии
```

---

## 5. AOP в Spring

- `@Aspect` + `@EnableAspectJAutoProxy`  
- Советы: `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`  
- `@Pointcut` — декларация выражений

**Когда**
- Кросс-срезочные задачи: аудит, метрики, ретраи, кеширование, контроль прав

```java
@Aspect
class AuditAspect {
  @Pointcut("within(@org.springframework.stereotype.Service *)")
  void serviceLayer() {} // точка среза по сервисам без точки в комментарии

  @Around("serviceLayer()")
  Object around(ProceedingJoinPoint pjp) throws Throwable { /* логика */ return pjp.proceed(); }
}
```

---

## 6. MapStruct

- `@Mapper` — интерфейс маппера, `componentModel = "spring"` для автосвязывания  
- `@Mapping(source="a", target="b")` — поле к полю, повторяемая  
- `@InheritInverseConfiguration`, `@AfterMapping`, `@BeforeMapping`, `@Context`  
- `@MapperConfig` — общие правила, `unmappedTargetPolicy = ReportingPolicy.ERROR` чтобы ловить пропуски

**Когда**
- Нужна быстрая и явная трансформация DTO ↔ доменные модели без рефлексии в рантайме

```java
@Mapper(componentModel = "spring")
interface UserMapper {
  @Mapping(source = "displayName", target = "name")
  UserDto toDto(User u);
}
```

---

## 7. OpenAPI / Swagger (springdoc-openapi)

Современные аннотации из `io.swagger.v3.oas.annotations`
- На уровне метода: `@Operation(summary = "...", description = "...")`  
- Ответы: `@ApiResponses` + `@ApiResponse(responseCode = "200", content = @Content(schema = @Schema(implementation = ...)))`  
- Параметры: `@Parameter`, `@RequestBody` из OpenAPI пакета, `@Schema` на DTO  
- Глобально: `@OpenAPIDefinition`, `@SecurityRequirement`, `@Tag`
- Скрытие: `@Hidden`

**Когда**
- Нужно самоописание API, генерация UI и клиентских SDK

**Заметка**
- Старые аннотации `io.swagger.annotations` из Swagger 2 считаются устаревающими

```java
@Operation(summary = "Получить пользователя")
@ApiResponses({
  @ApiResponse(responseCode = "200", description = "Ок"),
  @ApiResponse(responseCode = "404", description = "Не найден")
})
@GetMapping("/users/{'}}id{{'}")
UserDto get(@Parameter(description = "Идентификатор") @PathVariable long id) { /* ... */ return null; }
```

---

## 8. Micronaut

- DI/скоупы: `@Singleton`, `@Prototype`, `@Factory`, `@EachBean`, `@Bean`  
- Веб: `@Controller`, `@Get`, `@Post`, `@Patch`, `@Body`, `@QueryValue`, `@PathVariable`  
- Клиенты: `@Client`, `@Get`, `@Post` на интерфейсе  
- Конфиг: `@Value`, `@Property`, условия: `@Requires(property="...", value="...")`  
- Валидация: `@Validated` + Jakarta Validation

**Когда**
- Микросервисы с низким overhead и быстрым стартом, анотационное программирование без рефлексии в рантайме

---

## 9. Quarkus

- CDI: `@Inject`, `@ApplicationScoped`, `@Singleton`, `@RequestScoped`  
- RESTEasy/JAX-RS: `@Path`, `@GET`, `@POST`, `@Produces`, `@Consumes`  
- Конфиг: `@ConfigProperty(name="...")` из MicroProfile Config  
- REST-клиенты: `@RegisterRestClient`  
- JPA/Panache: те же JPA-аннотации, доп. утилиты Panache  
- Native-мод: `@RegisterForReflection` — включить классы в рефлексию для native-образа

**Когда**
- Производительные сервисы на GraalVM/native, совместимость с спецификациями MicroProfile

---

## 10. Чек-лист nullability и валидации для ревью кода

**Выберите один набор nullability**
- JetBrains `@NotNull/@Nullable` **или** Spring `@NonNull/@Nullable`  
- Объявляйте package-level поведение: `@NonNullApi` и `@NonNullFields` в Spring

**Контейнеры и дженерики**
- Аннотируйте элементы: `List<@NotNull Foo>`  
- Для карт: `Map<@NotNull Key, @NotNull Value>`

**Границы системы**
- На REST-DTO используйте Bean Validation (`@NotBlank`, `@Size`, `@Email`)  
- Внутри сервиса используйте nullability-аннотации для статанализа

**`@Valid` vs `@Validated`**
- `@Valid` — валидировать аргумент/поле рекурсивно  
- `@Validated` — включить проверку методов и группы валидаторов

**`Optional`**
- Не используйте `Optional` в аргументах публичных методов  
- Возвращать `Optional` допустимо для явно необязательного результата

**Транзакции**
- Только на сервисах, не на контроллерах и не на репозиториях  
- Идём от инварианта домена, а не от механики DAO

**Jackson x JPA**
- Избегайте прямой экспозиции сущностей JPA в API  
- Выделяйте DTO и управляйте JSON через `@JsonProperty`, `@JsonInclude`

---

## 11. Решения типичных проблем

- **Две реализации интерфейса и `NoUniqueBeanDefinitionException`** → добавьте `@Primary` или `@Qualifier("...")`  
- **Валидация коллекции не срабатывает** → добавьте `@Valid` на поле/аргумент контейнера  
- **Метод с `@Transactional` не открывает транзакцию** → вызов через `this`, вынесите в другой бин или используйте self-injection прокси  
- **MapStruct игнорирует поле** → включите `unmappedTargetPolicy = ERROR` через `@MapperConfig`  
- **OpenAPI не видит схему** → поставьте `@Schema(implementation = Type.class)` или добавьте `@Operation`/`@ApiResponses`

---

## 12. Мини-шаблоны для стартовых проектов

**Сервисный слой**
```java
@Service
class AccountService {
  private final AccountRepo repo;
  AccountService(AccountRepo repo) { this.repo = repo; } // внедрение через конструктор без точки в комментарии

  @Transactional(readOnly = true)
  AccountDto get(long id) { /* ... */ return null; }
}
```

**DTO с валидацией и JSON-контрактом**
```java
record CreateAccount(
  @NotBlank @JsonProperty("login") String login,
  @Email    @JsonProperty("email") String email,
  @Size(min = 8) @JsonProperty("password") String password
) {}
```

**MapStruct**
```java
@Mapper(componentModel = "spring")
interface AccountMapper {
  @Mapping(source = "displayName", target = "name")
  AccountDto toDto(Account a);
}
```

---

## 13. Что ещё можно добавить в ваш вариант

- Чек-лист логирования и трассировки: `@Observed`, `@Timed`, `@NewSpan` из Micrometer/Brave  
- Политики сериализации дат и валют через Jackson-модули и `@JsonFormat`  
- Консистентные правила `equals/hashCode` для JPA и Lombok
