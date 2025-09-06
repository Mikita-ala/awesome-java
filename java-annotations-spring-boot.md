
# Аннотации Spring Boot — практики и шпаргалка
Дата: 2025-09-06

> Файл сфокусирован на **Spring Boot 3.x** и Java 17+, с акцентом на реальные сценарии использования аннотаций, «когда» и «зачем», плюс рабочие шаблоны

---

## 0. Быстрые ориентиры по версиям

- Spring Boot 3.x требует **Java 17+**, использует **Spring Framework 6** и **Jakarta EE 10** неймспейсы  
- В `@ConfigurationProperties` конструкторная привязка теперь по умолчанию, `@ConstructorBinding` на типе больше не нужен, но можно пометить конкретный конструктор при их нескольких  
- Автоконфигурации помечаются `@AutoConfiguration` и перечисляются в `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

---

## 1. Базовые аннотации запуска

- `@SpringBootApplication` — мета‑аннотация, включает `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`  
  **Когда**: точка входа приложения  
  **Зачем**: автоконфигурация и сканирование компонентов из базового пакета

- `@SpringBootConfiguration` — упрощает поиск основной конфигурации приложением и тестовыми «слайсами»  
  **Когда**: редко ставится вручную, чаще приходит внутри `@SpringBootApplication`

```java
@SpringBootApplication
public class App {
  public static void main(String[] args) { org.springframework.boot.SpringApplication.run(App.class, args); }
}
```

---

## 2. Автоконфигурация и условные аннотации

**Ключевая аннотация**  
- `@AutoConfiguration` — класс автоконфигурации, подключается через `AutoConfiguration.imports`

**Часто используемые условия**  `org.springframework.boot.autoconfigure.condition.*`
- `@ConditionalOnClass` / `@ConditionalOnMissingClass` — наличие классов на classpath  
- `@ConditionalOnBean` / `@ConditionalOnMissingBean` — наличие бинов в контексте  
- `@ConditionalOnProperty` / `@ConditionalOnBooleanProperty` — флаг в конфигурации  
- `@ConditionalOnResource` — ресурс доступен на classpath  
- `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` — тип контекста  
- `@ConditionalOnSingleCandidate` — единственный кандидат нужного типа  
- `@ConditionalOnJava` — минимальная версия JVM  
- `@ConditionalOnCloudPlatform`, `@ConditionalOnExpression`, `@ConditionalOnJndi` — особые среды

**Шаблон автоконфигурации**

```java
@AutoConfiguration
@ConditionalOnClass(Client.class)
@ConditionalOnProperty(prefix = "http", name = "enabled", matchIfMissing = true)
class HttpClientAutoConfiguration {
  @Bean
  @ConditionalOnMissingBean
  Client client(HttpProps props) { return new Client(props.baseUrl(), props.timeout()); } // создаем только если нет пользовательского бина
}
```

Файл `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
com.example.autoconfig.HttpClientAutoConfiguration
```

---

## 3. Свойства и конфигурация

**Аннотации**
- `@ConfigurationProperties(prefix = "app")` — типобезопасное связывание настроек
- `@ConfigurationPropertiesScan` — сканирование классов свойств по пакетам
- `@EnableConfigurationProperties` — явная регистрация конкретных классов свойств
- `@Validated` — валидация `jakarta.validation` при биндинге

**Современный способ: records и конструкторная привязка**

```java
// build.gradle: implementation("org.springframework.boot:spring-boot-configuration-processor") // метаданные для IDE

@ConstructorBinding // опционально, если в типе несколько конструкторов
@ConfigurationProperties(prefix = "http")
@jakarta.validation.constraints.NotNull // можно вешать на сам тип или поля
public record HttpProps(
  @jakarta.validation.constraints.NotNull java.net.URI baseUrl,
  @jakarta.validation.constraints.Positive int timeoutMillis
) {}
```

Регистрация и использование

```java
@SpringBootApplication
@ConfigurationPropertiesScan // сканирует пакет приложения
class App { }

@Configuration
class HttpConfig {
  @Bean Client client(HttpProps p) { return new Client(p.baseUrl(), p.timeoutMillis()); } // типобезопасно, валидируется на старте
}
```

**Когда**
- Набор связанных настроек, нужен контракт и валидация  
**Зачем**
- Fail‑fast при некорректных конфигурациях, автодокументация свойств через configuration‑processor

---

## 4. Профили и включение по флагам

- `@Profile("dev")` — активировать бин в конкретных профилях  
- `@ConditionalOnProperty(prefix="feature.x", name="enabled", havingValue="true")` — простая фича-флагика

```java
@Service
@Profile("prod")
class RealEmailSender implements EmailSender { /* реальная интеграция */ } // включаем только в prod

@Service
@Profile("!prod")
class NoopEmailSender implements EmailSender { /* подмена на стендах */ }
```

---

## 5. Actuator и наблюдаемость

**Пользовательские эндпойнты**
- `@Endpoint(id = "custom")` с методами `@ReadOperation`, `@WriteOperation`, `@DeleteOperation`  
- Подключение через зависимость `spring-boot-starter-actuator` и настройку экспозиции

```java
@Endpoint(id = "build")
class BuildEndpoint {
  @ReadOperation
  Map<String,Object> info() { return Map.of("commit", "abc123", "time", "2025-09-01T12:00:00Z"); } // минимальный health‑like эндпойнт
}
```

**Наблюдаемость Micrometer**
- `@Observed` — помечает метод или класс для сбора метрик и трассировки  
- Для аспектной обработки добавьте `ObservedAspect` или используйте автоконфигурацию стартеров наблюдаемости

```java
import io.micrometer.observation.annotation.Observed;

@Service
class PaymentService {
  @Observed(name = "payment.process", contextualName = "payment")
  PaymentResult process(Order o) { /* бизнес‑логика */ return PaymentResult.success(); }
}
```

---

## 6. Тестирование в Spring Boot

**Интеграционные**
- `@SpringBootTest` — поднимает полный контекст  
  **Когда**: e2e сценарии, контракты интеграций

**Быстрые «слайсы»** — поднимают часть контекста
- Веб: `@WebMvcTest`, `@WebFluxTest`  
- Данные: `@DataJpaTest`, `@JdbcTest`, `@DataJdbcTest`, `@DataR2dbcTest`, `@DataMongoTest`, `@DataRedisTest`, `@DataCassandraTest`  
- Клиенты и JSON: `@RestClientTest`, `@JsonTest`  
- Прочее: `@GraphQlTest`

Полезные дополнения
- `@MockBean` — замена зависимостей на моки
- `@AutoConfigureTestDatabase` — встраиваемая БД для JPA/ JDBC
- `@TestConfiguration` — локальные тестовые бины

```java
@WebMvcTest(controllers = UsersController.class)
class UsersControllerTest {
  @Autowired MockMvc mvc;
  @MockBean UserService service;

  @org.junit.jupiter.api.Test
  void returns200() throws Exception {
    mvc.perform(get("/users/42")).andExpect(status().isOk()); // тест веб‑слоя быстрый и изолированный
  }
}
```

---

## 7. Кеширование, планирование и асинхронность

- `@EnableCaching`, `@Cacheable`, `@CachePut`, `@CacheEvict`  
  **Когда**: повторяющиеся чтения без строгой консистентности

- `@EnableScheduling`, `@Scheduled(cron="...")`  
  **Когда**: периодические задачи без отдельного оркестратора

- `@EnableAsync`, `@Async`  
  **Когда**: неблокирующие фоновые операции, не требующие отслеживания потока запроса

```java
@Service
class CatalogService {
  @Cacheable("products")
  Product get(long id) { /* обращение к БД или внешней системе */ return null; } // кеш по ключу id

  @Scheduled(fixedDelayString = "PT5M")
  void warmup() { /* подгрев кеша */ }
}
```

---

## 8. Паттерны включения/переопределения бинов

**`@Primary` и `@Qualifier`**
- Разруливают многозначность кандидатов для DI

**Override автоконфигурации**
- Свой бин того же типа + `@ConditionalOnMissingBean` в авто‑конфигурации стороннего стартера обеспечивает переопределение по умолчанию

```java
@Configuration
class CustomHttpClient {
  @Bean @Primary Client client() { return Client.builder().withRetry(3).build(); } // приоритет над авто‑сконфигурированным
}
```

---

## 9. Частые ошибки и как их избежать

- `@Transactional` не срабатывает при `this.method()` вызовах → вынесите в другой бин или используйте self‑инъекцию прокси  
- `@ConfigurationProperties` не биндинится → добавьте `@ConfigurationPropertiesScan` или `@EnableConfigurationProperties` и проверьте префикс  
- Несколько реализаций интерфейса → добавьте `@Primary` или `@Qualifier`  
- Тесты медленные → используйте слайсы, а не всегда `@SpringBootTest`  
- `@Observed` не пишет метрики → проверьте зависимость наблюдаемости и наличие аспекта

---

## 10. Мини‑шаблоны

**Фича‑флаг через конфиг**

```java
@AutoConfiguration
@ConditionalOnProperty(prefix = "feature.xyz", name = "enabled", havingValue = "true")
class XyzAutoConfig {
  @Bean XyzService xyz() { return new XyzService(); } // включаем функциональность флагом
}
```

**Типобезопасные свойства для HTTP‑клиента**

```java
@Validated
@ConfigurationProperties(prefix = "client.http")
public record ClientHttpProps(
  @jakarta.validation.constraints.NotBlank String baseUrl,
  @jakarta.validation.constraints.Positive int connectTimeoutMillis,
  @jakarta.validation.constraints.Positive int readTimeoutMillis
) {}
```

**Тест JPA‑слоя**

```java
@DataJpaTest
class RepoTest {
  @Autowired EntityManager em;
  @Autowired UserRepo repo;

  @org.junit.jupiter.api.Test
  void savesAndFinds() {
    var u = new User("neo"); em.persist(u); em.flush();
    var found = repo.findByName("neo"); // быстрый тест репозитория
  }
}
```

---

## 11. Что добавить под ваш стек

- Аннотации безопасности и метод‑секьюрити `@EnableMethodSecurity`, `@PreAuthorize`  
- Наблюдаемость на HTTP‑клиентах через `RestClient.Builder` и теги метрик  
- Примеры автоконфигурации для собственных стартеров и зависимостей
