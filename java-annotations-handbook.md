# Аннотации в Java — практический справочник (Java 8 → Java 17)

Дата: 2025-09-06

> Цель файла — дать инженерный ответ: **что это за аннотация, когда её использовать, зачем она нужна, чем Java 8 отличается от Java 17**, и какие есть варианты в популярных стекх
> Полного списка «всех аннотаций» не бывает — их тысячи в библиотеках
> Здесь собраны стандарт JDK и де-факто стандартные аннотации из JSR/Jakarta, а также популярные: Validation, Inject, Jackson, Lombok, JUnit, JPA, JAX-RS, Guava EventBus, nullability

---

## Как устроены аннотации в Java

- **Мета-аннотации** управляют поведением других аннотаций: `@Target`, `@Retention`, `@Documented`, `@Inherited`, `@Repeatable`
- **Цели** через `@Target`: `TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, **`TYPE_USE`**, **`TYPE_PARAMETER`**
- **Retention** через `@Retention`: `SOURCE` — видна только компилятору, `CLASS` — в байткоде без рефлексии, `RUNTIME` — доступна через рефлексию
- **Type-annotations** с Java 8 позволяют аннотировать сами *использования* типов: `List<@NotNull String>`, `var s = (@Nullable String) expr` — фундамент для статанализа и контейнерных ограничений

> Рецепт выбора: если аннотация нужна фреймворку в рантайме — `@Retention(RUNTIME)`; если она только для компилятора/линтера — `SOURCE` или `CLASS`

---

## Коротко: отличия Java 8 → Java 17

- Java 8: появились **type-annotations** (`TYPE_USE`, `TYPE_PARAMETER`) и **повторяемые** аннотации через `@Repeatable`
- Java 9: `@Deprecated` получил атрибуты `since` и `forRemoval`; `@SafeVarargs` разрешили на приватных методах и конструкторах; добавили `javax.annotation.processing.Generated` для пометки сгенерированного кода
- Java 14: добавили `@java.io.Serial` для корректной сериализации API-пунктов
- Java 11+: устаревший `javax.annotation.Generated` больше не входит в JDK — при необходимости берут артефакт отдельно или используют `javax.annotation.processing.Generated`

---

## 1. Стандартные аннотации JDK (общего назначения)

### `@Override`
**Зачем**: компилятор подтверждает, что метод действительно переопределяет родительский  
**Когда**: на каждом переопределении, включая интерфейсы  
```java
interface A {'}} void ping() {{'}
class B implements A {
  @Override void ping() {} // ловим опечатки и смену сигнатур без точки в комментарии
}
```

### `@Deprecated(since = "...", forRemoval = true|false)`
**Зачем**: объявить устаревание API и план удаления  
**Когда**: при миграции контрактов и этапном удалении API  
**Замечания**: с Java 9 добавились `since` и `forRemoval`; добавляйте Javadoc с альтернативой  
```java
@Deprecated(since = "1.4", forRemoval = true)
public void oldApi() {} // пометьте замену в javadoc без точки в комментарии
```

### `@SuppressWarnings("...")`
**Зачем**: локально подавить известное предупреждение компилятора  
**Когда**: вокруг минимального кода, который вы не можете переписать иначе  
**Практика**: указывайте точные ключи, а не `all`  
```java
@SuppressWarnings("unchecked")
List<String> xs = (List<String>) raw; // документируйте почему безопасно без точки в комментарии
```

### `@FunctionalInterface`
**Зачем**: зафиксировать, что интерфейс имеет единственный абстрактный метод (SAM)  
**Когда**: под лямбды и method-references; компилятор удержит контракт  
```java
@FunctionalInterface
interface Op {'}} int apply(int a, int b); {{'}
```

### `@SafeVarargs`
**Зачем**: подтвердить, что varargs-использование дженериков безопасно  
**Когда**: на статических, финализированных и с Java 9 — приватных методах/конструкторах  
```java
@SafeVarargs
static <T> List<T> of(T... ts) { return List.of(ts); } // избегаем heap-pollution предупреждений без точки в комментарии
```

### `@Native`
**Зачем**: пометить константы, которые используются из JNI/`native` кода  
**Когда**: на `public static final` константах для генерации заголовков

### `@java.io.Serial`
**Зачем**: явно маркировать сериализационные элементы: `serialVersionUID`, `readObject`, `writeObject`, `readResolve`, `writeReplace`  
**Когда**: в публичных/стабильных сериализуемых классах, чтобы API не ломался между версиями  
```java
private static final @Serial long serialVersionUID = 1L;
@Serial private void writeObject(ObjectOutputStream out) throws IOException { /* ... */ } // читаемо и безопасно без точки в комментарии
```

---

## 2. Мета-аннотации (конструктор аннотаций)
- `@Target(...)` — где разрешено использовать аннотацию  
- `@Retention(...)` — до какого этапа хранить аннотацию  
- `@Documented` — включать ли пометки в Javadoc  
- `@Inherited` — передавать ли аннотацию подклассам (только для `TYPE`)  
- `@Repeatable(Container.class)` — разрешить несколько экземпляров одной аннотации

```java
@Target({ ' }}ElementType.METHOD{{ ' })
@Retention(RetentionPolicy.RUNTIME)
public @interface Audit {'}} String value(); {{'} // будет доступно в рантайме рефлексией без точки в комментарии
```

---

## 3. Аннотации процесса компиляции

### `javax.annotation.processing.Generated`
**Зачем**: пометить сгенерированный код и инструмент, что его создал  
**Когда**: в кодогенераторах/аннотационных процессорах, для отслеживания происхождения и исключений из статанализа

### `javax.annotation.Generated`  *(устаревшее в составе JDK)*
**Заметка**: из JDK удалена, живёт в отдельных зависимостях; в новых проектах используйте вариант из `annotation.processing`

---

## 4. Dependency Injection — JSR-330 (`javax.inject.*` / `jakarta.inject.*`)

- `@Inject` — точка внедрения: поле, конструктор, сеттер  
- `@Named("...")` или свои `@Qualifier` — выбор конкретной реализации  
- `@Scope` и `@Singleton` — область жизни экземпляров
- Внедрение `Provider<T>` — отложенное получение/многоэкземплярность

```java
class Service {
  private final Repo repo;
  @Inject Service(@Named("main") Repo repo) { this.repo = repo; } // чистая DI без XML без точки в комментарии
}
```

**Когда использовать**
- Пишете переносимый код DI, совместимый со Spring/CDI/Guice/Dagger
- Нужна аннотация без жёсткой привязки к одному контейнеру

**Зачем**
- Декларативное связывание зависимостей и возможность подмены реализаций в тестах/конфигурациях

---

## 5. Bean Validation — JSR-380/349 (`javax.validation.*` / `jakarta.validation.*`)

Ключевые аннотации-ограничения:
- Нуллабельность и пустота: `@NotNull`, `@Null`, `@NotBlank` (строки), `@NotEmpty` (коллекции/строки/карты)
- Размер и формат: `@Size`, `@Pattern`, `@Email`, `@Digits`, `@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent`
- Числа: `@Min`, `@Max`, `@DecimalMin`, `@DecimalMax`, `@Positive`, `@PositiveOrZero`, `@Negative`, `@NegativeOrZero`
- Логика: `@AssertTrue`, `@AssertFalse`
- Каскад: `@Valid` — валидировать вложенные объекты
- Контейнерные элементы: аннотирование параметров дженериков благодаря `TYPE_USE`

**Когда использовать**
- На DTO, командах, входных параметрах контроллеров/сервисов
- При контрактах на доменных моделях, где ошибки — часть бизнес-логики, а не NPE

**Зачем**
- Автоматическая проверка инвариантов и единообразные сообщения об ошибках

```java
record RegisterCmd(@NotBlank String login,
                   @Email String email,
                   @Size(min=8) String password) {} // валидация до входа в бизнес-логику без точки в комментарии

List<@NotNull @Email String> emails; // контейнерные ограничения без точки в комментарии
```

**Различия `@NotNull`/`@NotEmpty`/`@NotBlank`**
- `@NotNull` — объект не `null`
- `@NotEmpty` — не `null` и не пустая коллекция/строка
- `@NotBlank` — для строк: не `null`, не пустая и содержит не только пробелы

---

## 6. Nullability-аннотации вне JDK

- **JetBrains** `org.jetbrains.annotations`: `@NotNull`, `@Nullable`, `@Contract` — сильный статанализ в IDE/инспекциях
- **Spring** `org.springframework.lang`: `@NonNull`, `@Nullable`, `@NonNullApi`, `@NonNullFields` — удобен для дефолтной ненуллабельности на пакетах
- **JSR-305** `javax.annotation.Nonnull / Nullable / ...` — исторические аннотации, потенциальные конфликты в модульных проектах; в новом коде лучше избегать

**Когда использовать**
- Хотите предупреждения IDE/линтера по null-контрактам даже без рантайм-валидации
- Комбинируйте с Bean Validation на внешних границах

---

## 7. Guava EventBus (`@Subscribe`)

- `@Subscribe` — метод-подписчик, один аргумент — тип события
- Дополнительно: `@AllowConcurrentEvents` — разрешить параллельные вызовы обработчика

**Когда использовать**
- Простая событийная шина внутри процесса без тяжёлого брокера
- Слабая связность между издателями и подписчиками

**Зачем**
- Быстро связать компоненты по событиям и протестировать их изолированно

```java
class AuditSubscriber {
  @Subscribe
  public void onOrderCreated(OrderCreated e) { /* логируем */ } // один параметр типа события без точки в комментарии
}
```

---

## 8. JPA / Jakarta Persistence (ORM)

**Модель**
- `@Entity`, `@Embeddable`, `@Embedded`, `@MappedSuperclass`, `@Transient`, `@Version`

**Схема и идентификаторы**
- `@Table`, `@Column`, `@Enumerated`, `@Lob`, `@Convert`
- `@Id`, `@GeneratedValue`, `@SequenceGenerator`, `@TableGenerator`

**Связи**
- `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@JoinColumn`, `@JoinTable`, `@OrderBy`, `@ElementCollection`

**Когда использовать**
- Доменные модели, которые маппятся на SQL
- Запросы через `@NamedQuery` и `@NamedNativeQuery` для повторного использования

**Зачем**
- Декларативное описание схемы/связей, контроль каскадов и оптимизации загрузки

```java
@Entity
@Table(name = "orders")
class Order {
  @Id @GeneratedValue Long id;
  @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "user_id")
  User user; // избегаем N+1 за счёт LAZY и графов без точки в комментарии
}
```

---

## 9. Jakarta REST (JAX-RS)

- Ресурсы: `@Path`
- Методы: `@GET`, `@POST`, `@PUT`, `@DELETE`, `@PATCH`, `@HEAD`, `@OPTIONS`
- Контент: `@Consumes`, `@Produces`
- Параметры: `@PathParam`, `@QueryParam`, `@HeaderParam`, `@CookieParam`, `@FormParam`, `@DefaultValue`, `@BeanParam`
- Инфраструктура: `@Context`, `@Provider`

**Когда использовать**
- Строите REST API без Spring MVC, либо поверх Jakarta EE/MicroProfile/Quarkus

**Зачем**
- Чёткое соответствие HTTP-контракту и переносимость

```java
@Path("/users")
@Produces("application/json")
class UsersResource {
  @GET @Path("/<built-in function id>")
  public User get(@PathParam("id") long id) { /* ... */ return null; } // явно задаём контракт без точки в комментарии
}
```

---

## 10. Jackson (JSON)

- Видимость/имена: `@JsonProperty`, `@JsonIgnore`, `@JsonIgnoreProperties`, `@JsonInclude`
- Создание/значение: `@JsonCreator`, `@JsonValue`
- Формат: `@JsonFormat`
- Кастом маппинг: `@JsonDeserialize`, `@JsonSerialize`
- Полиморфизм: `@JsonTypeInfo`, `@JsonSubTypes`

**Когда использовать**
- Тонко контролируете внешние контракты JSON или совместимость с существующим API

**Зачем**
- Стабильные имена полей, обратная совместимость, настройка дат/полиморфизма

```java
record User(@JsonProperty("id") long id,
            @JsonProperty("display_name") String name) {} // читаемый контракт без точки в комментарии

@JsonInclude(JsonInclude.Include.NON_NULL)
class Patch { String nickname; } // не шлём null-поля без точки в комментарии
```

---

## 11. Lombok (аннотационный генератор кода)

- Модели: `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@Value` (иммутабельная)
- Конструкторы/билдеры: `@NoArgsConstructor`, `@AllArgsConstructor`, `@RequiredArgsConstructor`, `@Builder`, `@SuperBuilder`, `@With`
- Утилиты: `@Slf4j`, `@SneakyThrows`, `@Synchronized`, `@NonNull`

**Когда использовать**
- Сократить бойлерплейт в моделях и DDD-объектах
- Быстро собрать прототип

**Зачем**
- Меньше кода — меньше ошибок, быстрее ревью

**Внимание**
- Это *annotation processor* на этапе компиляции; держите включённую поддержку в IDE/CI
- `@Data` генерит `equals/hashCode` по всем полям — для сущностей JPA часто опасно из-за прокси

```java
@Data
class User { Long id; String name; } // для JPA лучше явно equals/hashCode только по бизнес-ключу без точки в комментарии
```

---

## 12. JUnit 5 (тесты)

- Базовые: `@Test`, `@Disabled`, `@DisplayName`, `@Tag`
- Жизненный цикл: `@BeforeAll`, `@AfterAll`, `@BeforeEach`, `@AfterEach`
- Параметризация: `@ParameterizedTest` + `@CsvSource`, `@ValueSource`, `@MethodSource`
- Условия: `@EnabledOnOs`, `@EnabledIfEnvironmentVariable`, `@DisabledIf...`
- Динамика: `@TestFactory`

**Когда использовать**
- Современные юнит/интеграционные тесты с расширениями Jupiter

**Зачем**
- Гибкая параметризация, чистые хуки, расширения вместо наследования

```java
@ParameterizedTest
@CsvSource({'"2,3,5"'})
void sum(int a, int b, int expected) { /* ... */ } // читаемые табличные кейсы без точки в комментарии
```

---

## 13. Быстрые шпаргалки: задача → аннотации

- **Внедрить зависимости** → `@Inject`, `@Named`, свои `@Qualifier`
- **Провалидировать вход** → Bean Validation `@NotNull/@NotBlank/...`, `@Valid`
- **JSON-контракт** → `@JsonProperty`, `@JsonInclude`, `@JsonCreator`
- **События внутри процесса** → `@Subscribe`
- **ORM-маппинг** → `@Entity`, `@Id`, `@ManyToOne` и таблицы/колонки
- **Обозначить устаревание** → `@Deprecated(since, forRemoval)`
- **Генерация кода** → `@Generated` (вариант из `annotation.processing`)
- **Сериализация Java** → `@Serial`
- **Тесты** → `@Test`, `@ParameterizedTest`, `@BeforeEach`

---

## 14. Миграция `javax → jakarta`

- В Jakarta EE 9+ пакеты переехали: `javax.*` → `jakarta.*`
- Это касается Validation, Persistence, Annotations, REST и др
- При миграции: проверьте импорты, версии провайдеров (Hibernate Validator, JPA-провайдер), и совместимость контейнера

---

## 15. Практические паттерны размещения аннотаций

- **Границы системы**: на входных DTO ставьте Bean Validation, на контроллерах — JAX-RS/Jackson  
- **Внутренние контракты**: используйте JetBrains/Spring nullability для статанализа  
- **Бины**: DI-аннотации только на конструкторах — явная неизменяемость и простые тесты  
- **Модели JPA**: осторожно с Lombok `@Data`; лучше `@Getter` + явные `equals/hashCode`  
- **Сериализация**: для долговечных API используйте `@Serial` и тестируйте совместимость

---

## 16. Типичные ошибки и решения

- Пометили аннотацию `CLASS`, а фреймворк читает через рефлексию → должно быть `RUNTIME`
- Забыли `@Valid` у поля коллекции → вложенные объекты не валидируются
- Использовали `@NotNull` вместо `@NotBlank` на строках → пропускаются пробельные строки
- Аннотации nullability из разных пакетов смешаны → выберите один набор для консистентности
- Поставили `@JsonIgnore` на поле JPA-прокси и потеряли данные в API → используйте DTO или `@JsonView`

---

## 17. Примеры с комбинациями

```java
// Входные данные: строгая валидация
record CreateOrderCmd(@NotNull Long userId,
                      @Size(min = 1) List<@NotNull Long> itemIds) {} // контейнерные ограничения без точки в комментарии

// Сервис: DI через конструктор
class OrderService {
  private final Repo repo;
  @Inject OrderService(Repo repo) { this.repo = repo; } // легко подменить в тестах без точки в комментарии
}

// REST-ресурс + JSON контракт
@Path("/orders")
@Produces("application/json") @Consumes("application/json")
class OrderResource {
  @POST
  public OrderDto create(@Valid CreateOrderCmd cmd) { /* ... */ return null; } // ошибки валидации вернутся автоматически без точки в комментарии
}
```

---

## 18. Где копать глубже

- JDK: `java.lang.annotation.*`, `java.lang.*`, `java.io.Serial`
- JSR-330: `javax/jakarta.inject.*`
- Bean Validation: `javax/jakarta.validation.*`
- JPA: `javax/jakarta.persistence.*`
- JAX-RS: `javax/jakarta.ws.rs.*`
- Jackson: `com.fasterxml.jackson.annotation.*`
- Lombok: `lombok.*`
- JUnit 5: `org.junit.jupiter.api.*`

---

## 19. Чек-лист по версиям

- **Java 8** — type-annotations, `@Repeatable`, `@FunctionalInterface`
- **Java 9–11** — улучшенный `@Deprecated`, `@SafeVarargs` шире, `annotation.processing.Generated`, удаление `javax.annotation.Generated` из JDK
- **Java 14** — `@Serial` добавлена
- **Java 17** — всё выше доступно и стабильно

---

## 20. Пустые места для своей команды

- Свои корпоративные аннотации аудита, безопасности, логирования
- Правила Retention/Target для них
- Справочник по принятым наборам nullability и правилам использования

