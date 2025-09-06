
# Spring Boot сервисы и Minecraft — архитектуры, практики, примеры
Дата: 2025-09-06

> Цель — показать как подключать Spring Boot сервисы к серверам Minecraft (Paper/Spigot/Velocity), когда что использовать и зачем
> Внутри: архитектуры, docker-compose без ключа version, API примеры, плагин-шаблон, безопасность, наблюдаемость, чек-лист

---

## 1. Архитектуры интеграции

**A. Внешний сервис + плагин по HTTP/WebSocket**
- Плагин на Paper/Spigot отправляет события в Spring Boot REST API и запрашивает команды
- Подходит для большинства задач: чат‑модерация, экономика, whitelisting, ачивки, социальные фичи
- Преимущества: простота, совместимость, легко масштабировать

**B. Внешний сервис + RCON**
- Spring Boot управляет сервером через RCON (консольные команды)
- Плюсы: не нужен свой плагин для базовых команд
- Минусы: однонаправленность, безопасность и аудит слабее, нет типизированного контракта

**C. Событийная шина: плагин ↔ брокер ↔ сервис**
- Плагин публикует события в Redis Streams / RabbitMQ / Kafka, Spring Boot подписывается
- Плюсы: надёжная доставка, ретраи, decoupling
- Минусы: усложнение инфраструктуры

**D. Встраивание Spring в плагин**
- Технически возможно поднимать ApplicationContext в Bukkit‑плагине, но чаще не стоит
- Минусы: тяжёлый старт, конкуренция контекстов, сложнее дебажить и обновлять плагины отдельно

Рекомендация: начинать с A, добавлять C при росте нагрузки и требований к надёжности

---

## 2. Быстрый старт через docker‑compose

- Один Compose поднимает Spring Boot API, Postgres, Redis и Paper‑сервер
- Папка `./mc/plugins` монтируется в контейнер для ваших плагинов
- API общается с плагином по HTTP, альтернативно слушает Redis Streams

```yaml
services:
  api:
    build:
      context: ./api
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/app
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: app
      SPRING_REDIS_HOST: redis
      APP_API_KEY: dev-key-please-change
    ports:
      - "8080:8080"
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - db-data:/var/lib/postgresql/data

  redis:
    image: redis:7

  mc:
    image: itzg/minecraft-server:latest
    environment:
      EULA: "TRUE"
      TYPE: "PAPER"
      ONLINE_MODE: "TRUE"
      # при необходимости включите rcon: ENABLE_RCON: "true", RCON_PASSWORD: "pass"
    ports:
      - "25565:25565"
    volumes:
      - ./mc/plugins:/plugins
      - mc-data:/data
    depends_on:
      - api

volumes:
  db-data:
  mc-data:
```

Папки проекта
```
api/               # код Spring Boot сервиса
mc/plugins/        # сюда кладете jar плагина
docker-compose.yml # без ключа version
```

---

## 3. Spring Boot API — минимальный контракт

```java
// build.gradle: implementation "org.springframework.boot:spring-boot-starter-web"
// build.gradle: implementation "org.springframework.boot:spring-boot-starter-validation"
// build.gradle: implementation "org.springframework.boot:spring-boot-starter-actuator"
// комментарии без точки

@RestController
@RequestMapping("/api")
class GameController {

  private final String apiKey;

  GameController(@Value("${APP_API_KEY}") String apiKey) {
    this.apiKey = apiKey;
  }

  private void checkKey(String key) {
    if (!apiKey.equals(key)) throw new ResponseStatusException(HttpStatus.FORBIDDEN, "bad key"); // краткая проверка ключа
  }

  @GetMapping("/ping")
  Map<String, Object> ping(@RequestHeader("X-Api-Key") String key) {
    checkKey(key);
    return Map.of("ok", true, "ts", System.currentTimeMillis());
  }

  record ChatMsg(@NotBlank String player, @NotBlank String msg) {}

  @PostMapping("/events/chat")
  void chat(@RequestHeader("X-Api-Key") String key, @Valid @RequestBody ChatMsg evt) {
    checkKey(key);
    // здесь можно публиковать событие в Redis Streams или сохранять в БД
  }

  record Tell(@NotBlank String player, @NotBlank String msg) {}

  @PostMapping("/commands/tell")
  void tell(@RequestHeader("X-Api-Key") String key, @Valid @RequestBody Tell cmd) {
    checkKey(key);
    // вариант A: отправить в канал команд для плагина
    // вариант B: через RCON выполнить tellraw или msg
  }
}
```

Когда
- Нужен простой и явный контракт для плагина и внешних сервисов
Зачем
- Минимизируем связность и упрощаем тестирование

---

## 4. Плагин Paper/Spigot — скелет и связка с API

**Maven `pom.xml` ключевые части**
```xml
<properties>
  <maven.compiler.source>17</maven.compiler.source>
  <maven.compiler.target>17</maven.compiler.target>
</properties>
<dependencies>
  <dependency>
    <groupId>io.papermc.paper</groupId>
    <artifactId>paper-api</artifactId>
    <version>1.21.1-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.12.0</version>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.1</version>
  </dependency>
</dependencies>
```

**`plugin.yml`**
```yaml
name: Bridge
main: com.example.BridgePlugin
api-version: 1.20
commands:
  bridgeping: { description: ping api }
```

**Код плагина**
```java
// комментарии без точки
public final class BridgePlugin extends JavaPlugin implements Listener {

  private OkHttpClient http;
  private String apiUrl;
  private String apiKey;
  private ObjectMapper json;

  @Override
  public void onEnable() {
    saveDefaultConfig() // создаем config.yml если нет
    this.apiUrl = getConfig().getString("api.url", "http://api:8080");
    this.apiKey = getConfig().getString("api.key", "dev-key-please-change");
    this.http = new OkHttpClient();
    this.json = new ObjectMapper();
    getServer().getPluginManager().registerEvents(this, this);
    getLogger().info("Bridge enabled") // логи помогут при отладке
  }

  @EventHandler
  public void onChat(AsyncPlayerChatEvent e) {
    // не блокируем главный тред
    Bukkit.getScheduler().runTaskAsynchronously(this, () -> {
      try {
        var body = json.writeValueAsString(Map.of("player", e.getPlayer().getName(), "msg", e.getMessage()));
        var req = new Request.Builder()
          .url(apiUrl + "/api/events/chat")
          .header("X-Api-Key", apiKey)
          .post(RequestBody.create(body, MediaType.parse("application/json")))
          .build();
        try (var resp = http.newCall(req).execute()) { /* можно проверить код */ }
      } catch(Exception ex) { getLogger().warning("chat event post failed"); }
    });
  }

  @Override
  public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
    if (cmd.getName().equalsIgnoreCase("bridgeping")) {
      Bukkit.getScheduler().runTaskAsynchronously(this, () -> {
        try {
          var req = new Request.Builder().url(apiUrl + "/api/ping").header("X-Api-Key", apiKey).get().build();
          try (var resp = http.newCall(req).execute()) {
            sender.sendMessage("api: " + resp.code());
          }
        } catch(Exception ex) { sender.sendMessage("api error"); }
      });
      return true;
    }
    return false;
  }
}
```

Когда
- Нужна минимальная связка сервера и вашего сервиса без брокеров
Зачем
- Быстрый путь к продакшену и простые отказы

---

## 5. Безопасность и ограничения

- Используйте ключи доступа по заголовку `X-Api-Key`, храните их в `config.yml` плагина и в переменных окружения Spring  
- Для публичных эндпойнтов добавляйте rate limit на стороне API и проверяйте UUID/никнейм в белых списках  
- Не блокируйте основной тред сервера — сетевые вызовы только асинхронно

Шаблон фильтра безопасности
```java
// комментарии без точки
@Configuration
class SecurityConfig {
  @Bean SecurityFilterChain http(HttpSecurity h) throws Exception {
    return h.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(reg -> reg.requestMatchers("/actuator/**").permitAll()
                                             .anyRequest().authenticated())
            .addFilterBefore(new ApiKeyFilter(), UsernamePasswordAuthenticationFilter.class)
            .build();
  }
}

class ApiKeyFilter extends OncePerRequestFilter {
  @Value("${APP_API_KEY}") private String apiKey;
  @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    var key = req.getHeader("X-Api-Key");
    if (!apiKey.equals(key)) { res.sendError(HttpServletResponse.SC_FORBIDDEN, "bad key"); return; }
    chain.doFilter(req, res);
  }
}
```

---

## 6. Вариант с RCON

- Включите RCON в контейнере itzg/minecraft-server
- В Spring Boot добавьте клиент RCON и выполняйте команды tellraw, give, whitelist add
- Плюс: не нужно писать плагин для простых сценариев
- Минус: контроль обратной связи и типизация слабее, лучше для админских задач

---

## 7. Наблюдаемость и DevOps

- Подключите Actuator: /actuator/health, /actuator/metrics, /actuator/prometheus
- Логируйте корреляционные traceId/spanId при обработке событий от плагина
- В плагине логируйте тайминги вызовов к API и количество ошибок
- Для очередей используйте Redis Streams или RabbitMQ, чтобы не терять события при перезапусках

---

## 8. Тестирование

- API: @WebMvcTest для контроллеров, WireMock для интеграции с внешними системами
- Плагин: библиотека MockBukkit для юнит‑тестов без реального сервера
- Контракты: генерируйте OpenAPI из Spring и используйте его при разработке плагина

---

## 9. Чек‑лист перед продом

- Асинхронность в плагине соблюдена, главный тред не блокируется
- API ключи и секреты в переменных среды и config.yml, не в репозитории
- Ограничения по скорости и защита от повторных запросов
- Метрики и алерты на ошибки сети и время ответа
- Бэкапы БД и снапшоты мира
- Нагрузочные тесты критичных маршрутов: вход игрока, чат, экономические операции

---

## 10. Куда развивать

- Перейти на шину событий для устойчивой доставки
- Добавить WebSocket канал обратных команд в плагин
- Вынести команды с побочными эффектами в очередь с идемпотентностью
- Сбор ачивок и статистики в ClickHouse/TimescaleDB и дашборды Grafana
