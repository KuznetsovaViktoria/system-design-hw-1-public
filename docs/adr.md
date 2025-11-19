# ADR: Сервис аренды пауэрбанков

## Контекст

### Исходно
Изначально монолит, храним данные в памяти и синхронно обращается к внешним сервисам без кэшей, фоллбэков, идемпотентности.

### Требования
- **Maintainability**: разделение ответственности, модульность
- **Reliability**: кэширование, фоллбэки, идемпотентность
- **Scalability**: возможность независимого масштабирования компонентов
- **Observability**: структурированные логи, метрики

### Параметры нагрузки
- **X = 10 RPS** — одновременное создание заказов
- **Y = 100** — число просмотров информации об аренде на одну аренду
- **Z = 1 КБ** — размер записи об аренде
- **Хранение**: 2 года

### Внешние зависимости
**Критичные:**
- `stations` — управление станциями и выдача пауэрбанков

**Некритичные:**
- `configs` — конфигурация (кэш 60с)
- `tariffs` — тарифы (LRU кэш, 10 мин)
- `users` — профили пользователей (фоллбэк на жадный прайсинг)
- `payments` — платежи (фоллбэк на долг)

---

## Решение

**Микросервисная архитектура** с разделением на 3 сервиса

### Структура сервисов

```
┌─────────────────────────────────────────────────────────────┐
│                      External Services                       │
│  stations │ tariffs │ users │ configs │ payments            │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
    │ Rentals │◄───┤ Pricing │    │ Billing │
    │  :8000  │    │  :8001  │    │  :8002  │
    └────┬────┘    └─────────┘    └─────────┘
         │
    ┌────▼────────────────────────────┐
    │       SQLite Database           │
    │  offers │ rentals │ debts │... │
    └─────────────────────────────────┘
```

---

## Детальное описание сервисов

### 1. Rentals Service (Core)

**Ответственность:**
- Управление жизненным циклом аренды
- Создание офферов с TTL
- Начало аренды (извлечение пауэрбанка)
- Возврат пауэрбанка
- Предоставление информации об аренде

**Структура:**
```
services/rentals/
├── app.py              # FastAPI приложение, эндпоинты
└── (логика в app.py)
```

**API эндпоинты:**

#### POST /offers
Создание оффера для аренды.

**Request:**
```json
{
  "user_id": "user123",
  "station_id": "station456"
}
```

**Response:**
```json
{
  "id": "offer789",
  "user_id": "user123",
  "station_id": "station456",
  "tariff_id": "tariff18",
  "created_at": "2025-11-12T10:00:00Z",
  "expires_at": "2025-11-12T10:10:00Z",
  "deposit": 300
}
```

**Логика:**
1. Получить данные станции (stations API) -> tariff_id
2. Получить тариф через pricing service
3. Получить профиль пользователя (users API)
4. Рассчитать депозит (0 для trusted, иначе default_deposit)
5. Создать оффер с TTL (по умолчанию 10 мин)
6. Сохранить в БД

**Пример кода:**
```python
@app.post("/offers")
def create_offer(payload: Dict[str, Any] = Body(...)):
    user_id = payload.get("user_id")
    station_id = payload.get("station_id")
    
    station = clients.get_station_data(station_id)
    tariff_id = station["tariff_id"]
    tariff = svc_clients.get_tariff_from_pricing(tariff_id)
    user = clients.get_user_profile(user_id)
    deposit = 0 if user.get("trusted") else tariff["default_deposit"]
    
    offer_id = str(uuid.uuid4())
    created_at = datetime.utcnow()
    expires_at = created_at + timedelta(minutes=_get_offer_ttl_minutes())
    
    with session_scope() as s:
        s.add(Offer(
            id=offer_id, 
            user_id=user_id, 
            station_id=station_id,
            tariff_id=tariff_id,
            created_at=created_at,
            expires_at=expires_at
        ))
    
    metric_offers_created.inc()
    return {"id": offer_id, "expires_at": expires_at.isoformat(), ...}
```

#### GET /offers/{offer_id}/freshness
Проверка актуальности оффера.

**Response:**
```json
{
  "fresh": true,
  "expires_at": "2025-11-12T10:10:00Z"
}
```

#### POST /rentals
Начало аренды (идемпотентное).

**Headers:**
```
Idempotency-Key: unique-key-123
```

**Request:**
```json
{
  "offer_id": "offer789"
}
```

**Response:**
```json
{
  "id": "rental999",
  "status": "active",
  "powerbank_id": "powerbank_638"
}
```

**Логика:**
1. Проверить существование и свежесть оффера
2. Проверить идемпотентность (если уже есть rental с этим offer_id)
3. Захолдировать депозит (payments API)
4. Извлечь пауэрбанк на станции (stations API - критичный!)
5. Создать запись rental в БД
6. Вернуть результат

**Обработка отказов:**
- Если stations недоступен -> ошибка 503
- Если пауэрбанков нет -> откат холда, ошибка 409
- Если payments недоступен -> создаем долг, продолжаем

#### POST /rentals/{rental_id}/return
Возврат пауэрбанка (идемпотентное).

**Headers:**
```
Idempotency-Key: unique-key-456
```

**Response:**
```json
{
  "id": "rental999",
  "status": "returned",
  "billing": {
    "status": "charged",
    "amount_cents": 150
  }
}
```

**Логика:**
1. Получить rental из БД
2. Проверить идемпотентность (если уже returned)
3. Рассчитать длительность и стоимость через pricing
4. Вызвать billing service для списания
5. Обновить статус rental
6. Вернуть результат

#### GET /rentals/{rental_id}/summary
Информация об аренде и текущая стоимость.

**Response:**
```json
{
  "id": "rental999",
  "status": "active",
  "duration_minutes": 45,
  "estimated_amount": 150
}
```

**Интеграции:**
```python
# Критичные (fail-fast)
clients.get_station_data(station_id)      # stations
clients.eject_powerbank(station_id)       # stations

# Некритичные (с обработкой)
clients.get_user_profile(user_id)         # users -> fallback
clients.get_configs()                     # configs -> cache
svc_clients.get_tariff_from_pricing(...)  # pricing -> cache
svc_clients.bill_rental(...)              # billing -> debt
```

---

### 2. Pricing Service

**Ответственность:**
- Управление тарифами с кэшированием
- Расчет стоимости аренды
- Фоллбэк при недоступности users

**Структура:**
```
services/pricing/
├── app.py              # FastAPI приложение
└── (кэш в памяти)
```

**API эндпоинты:**

#### GET /tariffs/{tariff_id}
Получение тарифа с LRU кэшем.

**Response:**
```json
{
  "id": "tariff18",
  "price_per_hour": 50,
  "free_period_min": 5,
  "default_deposit": 300
}
```

**Кэширование:**
```python
# In-memory LRU cache
tariff_cache: Dict[str, Dict[str, Any]] = {}
tariff_cache_ts: Dict[str, datetime] = {}

@app.get("/tariffs/{tariff_id}")
def get_tariff(tariff_id: str):
    now = datetime.utcnow()
    
    # Проверяем кэш
    if tariff_id in tariff_cache:
        age = (now - tariff_cache_ts[tariff_id]).total_seconds()
        ttl = get_tariff_ttl_seconds()  # 600s по умолчанию
        
        if age <= ttl:
            metric_tariff_hits.inc()
            return tariff_cache[tariff_id]
        else:
            # Данные устарели
            metric_tariff_stale.inc()
            raise HTTPException(status_code=503, detail="Tariff is stale")
    
    # Cache miss - загружаем
    metric_tariff_misses.inc()
    data = clients.get_tariff(tariff_id)
    tariff_cache[tariff_id] = data
    tariff_cache_ts[tariff_id] = now
    return data
```

**Конфигурация TTL:**
```python
def get_tariff_ttl_seconds() -> int:
    cfg = config_cache.get()
    try:
        return int(cfg.get("tariffs", {}).get("valid_seconds", 600))
    except Exception:
        return 600  # 10 минут по умолчанию
```

#### GET /price/estimate
Оценка стоимости с фоллбэком для users.

**Query params:**
- `tariff_id` (required)
- `duration_minutes` (required)
- `user_id` (optional)

**Response:**
```json
{
  "tariff_id": "tariff18",
  "duration_minutes": 45,
  "price_per_hour": 50,
  "free_period_min": 5,
  "deposit": 300,
  "estimated_amount": 34
}
```

**Фоллбэк для users:**
```python
@app.get("/price/estimate")
def price_estimate(tariff_id: str, duration_minutes: int, user_id: str = None):
    # Пытаемся получить профиль пользователя
    try:
        user = clients.get_user_profile(user_id) if user_id else None
    except Exception:
        # FALLBACK: жадный прайсинг
        cfg = config_cache.get().get("pricing", {})
        greedy_coeff = float(cfg.get("greedy_coeff", 1.2))
        user = {
            "has_subscribtion": False,
            "trusted": False,
            "_greedy_coeff": greedy_coeff
        }
    
    tariff = get_tariff(tariff_id)
    
    # Расчет стоимости
    billable_minutes = max(0, duration_minutes - tariff["free_period_min"])
    amount = int((billable_minutes / 60.0) * tariff["price_per_hour"])
    
    # Применяем жадный коэффициент если был фоллбэк
    greedy_coeff = float(user.get("_greedy_coeff", 1.0))
    amount = int(amount * greedy_coeff)
    
    return {"estimated_amount": amount, ...}
```

**Config cache:**
```python
# packages/common/config_cache.py
class ConfigCache:
    def __init__(self, refresh_seconds: int = 60):
        self._refresh_seconds = refresh_seconds
        self._data: Dict[str, Any] = {}
        self._lock = threading.RLock()
    
    def start(self):
        # Фоновый поток обновления
        threading.Thread(target=self._run, daemon=True).start()
    
    def _run(self):
        while not self._stop:
            try:
                new_data = clients.get_configs()
                with self._lock:
                    self._data = new_data
            except Exception:
                # Используем старые данные
                pass
            time.sleep(self._refresh_seconds)
    
    def get(self) -> Dict[str, Any]:
        with self._lock:
            return dict(self._data)
```

---

### 3. Billing Service

**Ответственность:**
- Списание средств через payments
- Управление долгами при недоступности payments
- Сверка и погашение долгов

**Структура:**
```
services/billing/
├── app.py              # FastAPI приложение
└── (работа с БД)
```

**API эндпоинты:**

#### POST /bill/{rental_id}
Списание средств за аренду.

**Request:**
```json
{
  "amount_cents": 150,
  "user_id": "user123"
}
```

**Response (успех):**
```json
{
  "status": "charged",
  "amount_cents": 150
}
```

**Response (payments недоступен):**
```json
{
  "status": "debt_recorded",
  "amount_cents": 150
}
```

**Логика с фоллбэком:**
```python
@app.post("/bill/{rental_id}")
def bill_rental(rental_id: str, payload: Dict[str, Any]):
    amount_cents = int(payload.get("amount_cents", 0))
    user_id = payload.get("user_id")
    
    try:
        # Пытаемся списать через payments
        clients.clear_money_for_order(
            user_id=user_id,
            order_id=rental_id,
            amount=amount_cents
        )
        return {"status": "charged", "amount_cents": amount_cents}
    
    except Exception:
        # FALLBACK: создаем долг
        with session_scope() as s:
            debt = Debt(
                id=f"debt_{rental_id}",
                rental_id=rental_id,
                amount_cents=amount_cents,
                status="open",
                created_at=datetime.utcnow(),
                settled_at=None
            )
            s.merge(debt)
        
        metric_debt_opened.inc()
        return {"status": "debt_recorded", "amount_cents": amount_cents}
```

#### POST /reconcile/{debt_id}
Повторная попытка списания долга.

**Response:**
```json
{
  "status": "settled"
}
```

**Логика:**
```python
@app.post("/reconcile/{debt_id}")
def reconcile(debt_id: str):
    with session_scope() as s:
        debt = s.get(Debt, debt_id)
        if not debt or debt.status != "open":
            raise HTTPException(404, "Debt not found or already settled")
        
        try:
            clients.clear_money_for_order(
                user_id="unknown",  # можно хранить в debt
                order_id=debt.rental_id,
                amount=debt.amount_cents
            )
            debt.status = "settled"
            debt.settled_at = datetime.utcnow()
            metric_debt_settled.inc()
            return {"status": "settled"}
        except Exception as exc:
            raise HTTPException(503, str(exc))
```

---

## Идемпотентность

### Механизм
Использование заголовка `Idempotency-Key` для критичных операций.

**Таблица в БД:**
```sql
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    endpoint TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    response JSON,
    status_code INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

**Реализация:**
```python
# packages/common/idempotency.py
def _hash_request(payload: Dict[str, Any]) -> str:
    raw = json.dumps(payload, sort_keys=True).encode("utf-8")
    return hashlib.sha256(raw).hexdigest()

def idempotent(endpoint_name: str):
    def decorator(func):
        async def wrapper(*args, idempotency_key: str = Header(None), **kwargs):
            if not idempotency_key:
                return await func(*args, **kwargs)
            
            # Хэшируем запрос
            body = kwargs.get("body_payload") or {}
            request_hash = _hash_request({"endpoint": endpoint_name, "body": body})
            
            # Проверяем существующий ключ
            with session_scope() as s:
                existing = s.get(IdempotencyKey, idempotency_key)
                if existing and existing.request_hash == request_hash:
                    # Возвращаем сохраненный ответ
                    return JSONResponse(
                        status_code=existing.status_code,
                        content=json.loads(existing.response)
                    )
            
            # Выполняем запрос
            response = await func(*args, **kwargs)
            
            # Сохраняем результат
            with session_scope() as s:
                s.merge(IdempotencyKey(
                    key=idempotency_key,
                    endpoint=endpoint_name,
                    request_hash=request_hash,
                    response=json.dumps(response.body),
                    status_code=response.status_code
                ))
            
            return response
        return wrapper
    return decorator
```

**Использование:**
```python
@app.post("/rentals")
@idempotent("create_rental")
async def start_rental(payload: Dict, idempotency_key: str = Header(None)):
    # Логика создания аренды
    ...
```

---

## База данных

### Схема

```sql
-- Офферы с TTL
CREATE TABLE offers (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    station_id TEXT NOT NULL,
    tariff_id TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL
);

-- Аренды
CREATE TABLE rentals (
    id TEXT PRIMARY KEY,
    offer_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    station_id TEXT NOT NULL,
    powerbank_id TEXT,
    started_at TIMESTAMP NOT NULL,
    returned_at TIMESTAMP,
    status TEXT NOT NULL CHECK (status IN ('active','returned','overdue'))
);

-- События аренды (для аудита)
CREATE TABLE rental_events (
    id TEXT PRIMARY KEY,
    rental_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    event_at TIMESTAMP NOT NULL,
    payload JSON
);

-- Долги
CREATE TABLE debts (
    id TEXT PRIMARY KEY,
    rental_id TEXT NOT NULL,
    amount_cents INTEGER NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('open','settled')),
    created_at TIMESTAMP NOT NULL,
    settled_at TIMESTAMP
);

-- Идемпотентность
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    endpoint TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    response JSON,
    status_code INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL
);

-- Миграции
CREATE TABLE schema_migrations (
    version TEXT PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL
);
```

### Миграции
```python
# packages/common/migrate.py
def apply_migrations():
    with ENGINE.begin() as conn:
        # Создаем таблицу миграций
        conn.execute(text("""
            CREATE TABLE IF NOT EXISTS schema_migrations (
                version TEXT PRIMARY KEY,
                applied_at TIMESTAMP NOT NULL
            )
        """))
        
        # Применяем миграции
        for version, sql in MIGRATIONS:
            result = conn.execute(
                text("SELECT 1 FROM schema_migrations WHERE version = :v"),
                {"v": version}
            )
            if not result.fetchone():
                conn.execute(text(sql))
                conn.execute(
                    text("INSERT INTO schema_migrations VALUES (:v, :t)"),
                    {"v": version, "t": datetime.utcnow()}
                )
```

---

## Логи

**Формат:**
```json
{
  "timestamp": "2025-11-12T10:00:00.123Z",
  "level": "INFO",
  "service": "rentals",
  "request_id": "req-abc123",
  "idempotency_key": "idem-xyz789",
  "user_id": "user123",
  "rental_id": "rental999",
  "message": "Rental started successfully",
  "duration_ms": 145
}
```

**Реализация:**
```python
# packages/common/logging_utils.py
import logging
import json
from datetime import datetime

def setup_json_logging(service_name: str):
    class JsonFormatter(logging.Formatter):
        def format(self, record):
            log_obj = {
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "level": record.levelname,
                "service": service_name,
                "message": record.getMessage(),
            }
            if hasattr(record, "request_id"):
                log_obj["request_id"] = record.request_id
            if hasattr(record, "user_id"):
                log_obj["user_id"] = record.user_id
            return json.dumps(log_obj)
    
    handler = logging.StreamHandler()
    handler.setFormatter(JsonFormatter())
    logging.root.addHandler(handler)
    logging.root.setLevel(logging.INFO)
```

### Метрики Prometheus

**Rentals:**
```python
from prometheus_client import Counter, Histogram

metric_offers_created = Counter(
    "rentals_offers_created_total",
    "Total offers created"
)

metric_rentals_started = Counter(
    "rentals_started_total",
    "Total rentals started"
)

metric_rentals_returned = Counter(
    "rentals_returned_total",
    "Total rentals returned"
)

metric_request_duration = Histogram(
    "rentals_request_duration_seconds",
    "Request duration",
    ["endpoint"]
)
```

**Pricing:**
```python
metric_tariff_hits = Counter(
    "pricing_tariff_cache_hits_total",
    "Tariff cache hits"
)

metric_tariff_misses = Counter(
    "pricing_tariff_cache_misses_total",
    "Tariff cache misses"
)

metric_tariff_stale = Counter(
    "pricing_tariff_stale_total",
    "Tariff entries considered stale"
)
```

**Billing:**
```python
metric_debt_opened = Counter(
    "billing_debt_opened_total",
    "Debts created"
)

metric_debt_settled = Counter(
    "billing_debt_settled_total",
    "Debts settled"
)
```

**Эндпоинт метрик:**
```python
from prometheus_client import make_asgi_app

app = FastAPI()
app.mount("/metrics", make_asgi_app())
```

---

## Расчет ресурсов

### Нагрузка
- **X = 10 RPS** создание заказов
- **Y = 100** просмотров на аренду
- **Период хранения**: 2 года

### Оценка данных

**Аренды в день:**
- 10 RPS × 86400 сек = 864,000 аренд/день
- 864,000 × 365 × 2 = 630,720,000 аренд за 2 года

**Размер данных:**
- 1 аренда = 1 КБ
- 630M аренд × 1 КБ ≈ 630 ГБ за 2 года

**Просмотры (summary):**
- 864,000 аренд/день × 100 просмотров = 86,400,000 запросов/день
- ≈ 1000 RPS на GET /rentals/{id}/summary

**Вывод:** Необходимо кэширование для summary или индексы в БД.

### Кэши

**Config cache:**
- Размер: ~1 КБ
- Обновление: каждые 60 сек
- Нагрузка на configs: 1/60 RPS ≈ 0.017 RPS

**Tariff cache:**
- Размер: ~100 тарифов × 0.5 КБ = 50 КБ
- TTL: 10 минут
- Hit rate: ~95% (при стабильных тарифах)
- Нагрузка на tariffs: 10 RPS × 0.05 = 0.5 RPS

---

## Верхнеуровневый план

1. **Подготовить БД (SQLite) и миграции**
   - Создать схему таблиц
   - Реализовать миграционный скрипт

2. **Выделить 3 FastAPI-сервиса**
   - Разделить код по доменам
   - Настроить HTTP-клиенты между сервисами

3. **Реализовать кэши/фоллбэки/идемпотентность**
   - Config cache с фоновым обновлением
   - Tariff LRU cache
   - Идемпотентность через заголовки
   - Фоллбэки для users и payments

4. **Тесты и наблюдаемость**
   - Unit-тесты для бизнес-логики
   - Integration-тесты для сценариев
   - Load-тесты (X=10 RPS, Y=100)
   - JSON-логи и Prometheus метрики

5. **Документация запуска**
   - Инструкции по установке
   - Примеры API-вызовов
   - Описание метрик