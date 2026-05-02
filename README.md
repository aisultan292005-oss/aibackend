# AI-Driven Backend — Spring Boot + LLM Orchestrator

Бэкенд на Spring Boot 3 для оркестрации LLM-запросов с кэшированием, очередями, историей и ролевыми лимитами.

## Стек

| Технология | Назначение |
|---|---|
| Spring Boot 3.2 + Java 21 | Основа |
| Spring Security + JWT | Аутентификация / авторизация |
| WebClient (WebFlux) | Реактивные вызовы к LLM API |
| PostgreSQL + Flyway | Хранение истории и пользователей |
| Redis | Кэш ответов LLM |
| Kafka | Async очередь запросов |
| Lombok + MapStruct | Бойлерплейт |

## Быстрый старт

### 1. Инфраструктура (Docker)

```bash
docker-compose up -d
```

Поднимет: PostgreSQL, Redis, Kafka + Zookeeper, Kafka UI (http://localhost:8090)

### 2. Переменные окружения

```bash
export OPENAI_API_KEY=sk-...
export JWT_SECRET=your-super-secret-at-least-32-chars
# Опционально:
export DB_USERNAME=postgres
export DB_PASSWORD=postgres
export REDIS_HOST=localhost
export KAFKA_SERVERS=localhost:9092
```

### 3. Запуск

```bash
# В IntelliJ: Run AiBackendApplication
# Или через Maven:
./mvnw spring-boot:run
```

Приложение стартует на http://localhost:8080

---

## API

### Аутентификация

```http
# Регистрация
POST /api/v1/auth/register
{
  "username": "user1",
  "email": "user1@example.com",
  "password": "Password123"
}

# Логин
POST /api/v1/auth/login
{
  "username": "user1",
  "password": "Password123"
}
# Ответ: { "accessToken": "eyJ...", "tokenType": "Bearer", ... }
```

### LLM — синхронный запрос

```http
POST /api/v1/llm/complete
Authorization: Bearer <token>
{
  "prompt": "Объясни принцип работы трансформеров в ML",
  "model": "gpt-4o-mini",  // опционально
  "temperature": 0.7,       // опционально
  "maxTokens": 1024         // опционально
}
```

### LLM — асинхронный (через Kafka)

```http
POST /api/v1/llm/complete/async
Authorization: Bearer <token>
{ "prompt": "Напиши тесты для сортировки пузырьком" }

# Ответ 202:
{ "requestId": "uuid", "status": "PENDING", "pollUrl": "/api/v1/llm/status/uuid" }

# Опрашиваем результат:
GET /api/v1/llm/status/{id}
```

### Prompt Chain — цепочка промптов

```http
POST /api/v1/llm/chain
Authorization: Bearer <token>
[
  "Придумай бизнес-идею для стартапа в сфере AI",
  "Напиши MVP план для этой идеи",
  "Оцени риски и составь список первых 5 задач"
]
# Каждый шаг получает результат предыдущего как контекст
```

### История запросов

```http
GET /api/v1/llm/history?page=0&size=20
GET /api/v1/llm/history/{requestId}
```

### Админ

```http
GET  /api/v1/admin/requests      # Все запросы всех пользователей
DELETE /api/v1/admin/cache        # Очистить кэш Redis
```

---

## Роли и лимиты

| Роль | Лимит запросов/час |
|---|---|
| USER | 100 |
| PREMIUM | 1 000 |
| ADMIN | 10 000 |

Лимиты сбрасываются по расписанию (каждый час).

---

## Добавить новый LLM провайдер

Реализуйте интерфейс `LlmProvider`:

```java
@Component
public class AnthropicProvider implements LlmProvider {

    @Override
    public Mono<LlmProviderResponse> complete(LlmProviderRequest request) {
        // вызов Anthropic API
    }

    @Override public String getProviderName() { return "anthropic"; }
    @Override public String getDefaultModel()  { return "claude-3-haiku-20240307"; }
}
```

Затем в `application.yml`:
```yaml
app.llm.provider: anthropic
```

---

## Структура проекта

```
src/main/java/com/aibackend/
├── AiBackendApplication.java
├── config/
│   ├── AppProperties.java       # Typed config (@ConfigurationProperties)
│   ├── SecurityConfig.java      # Spring Security + JWT
│   ├── RedisConfig.java         # Redis + CacheManager
│   ├── WebClientConfig.java     # WebClient для OpenAI и Local LLM
│   └── KafkaConfig.java         # Topics, Producer, Consumer factory
├── controller/
│   ├── AuthController.java      # /api/v1/auth/*
│   ├── LlmController.java       # /api/v1/llm/*
│   └── AdminController.java     # /api/v1/admin/*
├── service/
│   ├── LlmOrchestrationService.java  # Основной оркестратор
│   ├── AuthService.java
│   ├── HistoryService.java
│   ├── SchedulerService.java    # Сброс лимитов
│   ├── llm/
│   │   ├── LlmProvider.java     # Strategy interface
│   │   ├── OpenAiProvider.java
│   │   └── LocalLlmProvider.java
│   ├── cache/
│   │   └── LlmCacheService.java
│   └── kafka/
│       ├── LlmKafkaProducer.java
│       └── LlmKafkaConsumer.java
├── entity/
│   ├── User.java
│   └── LlmRequest.java
├── repository/
│   ├── UserRepository.java
│   └── LlmRequestRepository.java
├── dto/
│   ├── AuthDtos.java
│   ├── LlmCompletionRequest.java
│   ├── LlmCompletionResponse.java
│   └── LlmRequestEvent.java
├── security/
│   ├── JwtService.java
│   └── JwtAuthenticationFilter.java
└── exception/
    ├── GlobalExceptionHandler.java
    ├── RateLimitExceededException.java
    └── ResourceAlreadyExistsException.java
```
