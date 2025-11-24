# ADR: Сервис аренды пауэрбанков

## Бизнес-задача

Нужен сервис для аренды пауэрбанков на станциях. Пользователь приходит к станции, берёт павер, пользуется, возвращает в любую станцию и платит за время использования.

**Функциональные требования:**
- Создать оффер на аренду (показать цену, депозит, условия)
- Начать аренду - выдать павер со станции
- Вернуть павер на любой станции
- Посчитать стоимость по времени аренды
- Списать деньги с пользователя
- Показать информацию о текущей аренде

### Нефункциональные требования по заданию

**Maintainability:**
- Код разбит на модули по доменам (аренда, цены, платежи)
- Каждый сервис можно разрабатывать и деплоить отдельно
- Изменение логики цен не трогает логику аренды

**Reliability:**
- Если тарифы недоступны -> используем кэш (до 10 минут свежести)
- Если users недоступен -> берём повышенную цену (жадный прайсинг)
- Если payments недоступен -> создаём долг, пускаем в аренду
- Если stations недоступен -> отказываем (это критично, без станции банку не выдать)
- Повторный запрос с тем же ключом идемпотентности -> возвращаем тот же результат

**Scalability:**
- При росте нагрузки на просмотр цен можем поднять больше инстансов pricing
- При росте нагрузки на аренду можем поднять больше инстансов rentals
- Кэши снижают нагрузку на внешние сервисы в 10-20 раз

**Observability:**
- JSON логи с request_id, user_id, rental_id для трейсинга запросов
- Метрики Prometheus: RPS, латентность, hit rate кэшей, число долгов
- По метрикам видно, когда внешний сервис падает (растут ошибки кэша)

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

**Микросервисная архитектура** с разделением на 3 сервиса на Python + FastAPI

### Рассмотренные альтернативы

#### 1. Разделение сервисов

**4 сервиса (Offers, Rentals, Pricing, Billing)**
- ✅ Чёткое разделение: офферы отдельно от заказов
- ❌ Офферы и рентал тесно связаны (один оффер -> одна аренда)
- ❌ Лишняя сложность: два сервиса постоянно общаются
- ❌ Офферы живут 10 минут, не нужен отдельный сервис

**3 сервиса (Rentals+Offers, Pricing, Billing)** <- **выбрали**
- ✅ Офферы и рентал в одном сервисе (связаны жизненным циклом)
- ✅ Pricing отдельно (переиспользуется: оффер, summary, return)
- ✅ Billing отдельно (своя логика долгов, reconciliation)
- ✅ Баланс

- Офферы — это "черновик" аренды, живут 10 минут
- Один оффер создаёт одну аренду (тесная связь)
- Разделять их кажется излишним

#### 2. Выделение Pricing и размещение Offers

**Вариант А: offers+pricing в одном сервисе**
- ✅ Офферы создаются с расчётом цены (логически связаны)
- ✅ Нет HTTP вызова Rentals -> Pricing при создании оффера
- ❌ Расчёт цены нужен не только для офферов (summary, return)
- ❌ Кэш тарифов и фоллбэки смешаются с логикой офферов

**Вариант Б: pricing отдельно, offers в rentals** <- **выбрали**
- ✅ Расчёт цены переиспользуется: оффер, summary, return
- ✅ Pricing изолирован (кэш тарифов, фоллбэки, жадный прайсинг)
- ✅ Можно масштабировать Pricing отдельно
- ❌ Один HTTP вызов при создании оффера

**Обоснование:**
- Расчёт цены нужен в 3 местах, не только для офферов
- Pricing — это переиспользуемый компонент
- Офферы тесно связаны с арендой (один оффер -> одна аренда)

#### 3. База данных

**Одна БД на все сервисы** <- **выбрали**
- ✅ Проще для дз (один SQLite файл)
- ✅ Не нужно настраивать репликацию/синхронизацию
- ✅ ACID транзакции работают
- ❌ Связанность через БД (но таблицы изолированы)
- ❌ Для прода не подходит

**БД на каждый сервис**
- ✅ Полная изоляция сервисов
- ✅ Можно использовать разные типы БД
- ❌ Сложнее для дз поднять локально
- ❌ Нужна eventual consistency
- ❌ Нет транзакций между сервисами

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

**Ответственность:** управление жизненным циклом аренды (офферы, старт, возврат, информация)

**API:**

1. **POST /offers** - создание оффера
2. **POST /rentals** - начало аренды (идемпотентно)
3. **POST /rentals/{id}/return** - возврат (идемпотентно)
4. **GET /rentals/{id}/summary** - текущая информация
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

**Идемпотентность:**
- Не требуется (каждый запрос создаёт новый оффер)
- Оффер живёт 10 минут, потом истекает
- Пользователь может создать несколько офферов, но использовать только один

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

**GET /offers/{id}/freshness** — проверка актуальности (expires_at > now)

**POST /rentals** — начало аренды (идемпотентно через offer_id)
```python
existing = session.query(Rental).filter_by(offer_id=offer_id).first()
if existing: return existing  # идемпотентность

try: clients.hold_deposit(user_id, amount)
except: pass  # некритично, можно записать долг

powerbank = clients.eject_powerbank(station_id)  # fail-fast!
# Создаём rental в БД
```

**POST /rentals/{id}/return** — возврат (идемпотентно через rental.status)
```python
if rental.status == "returned": return cached_result  # идемпотентность

duration = (now - rental.started_at).total_seconds() / 60
price = svc_clients.estimate_price(tariff_id, duration, user_id)
billing = svc_clients.bill_rental(rental_id, price, user_id)
rental.status = "returned"
```

**GET /rentals/{id}/summary** — текущая информация и стоимость

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
- Кэширование тарифов из внешнего сервиса `tariffs`
- Расчет стоимости аренды
- Фоллбэк при недоступности users (жадный прайсинг)

**API:**

1. **GET /tariffs/{id}** - кэширование тарифов:
   - Проверяем in-memory кэш (словарь + timestamp)
   - Если есть и свежий (< 10 мин) -> возвращаем из кэша
   - Если устарел (> 10 мин) -> ошибка 503 (данные невалидны)
   - Если нет в кэше -> идём во внешний `tariffs`, кэшируем, возвращаем
   - TTL настраивается через config service

2. **GET /price/estimate** - расчёт стоимости:
   - Пытаемся получить профиль пользователя из `users`
   - Если `users` недоступен -> фоллбэк на жадный прайсинг:
     - Берём коэффициент из config (по умолчанию 1.2)
     - Считаем как для непривилегированного пользователя
     - Умножаем цену на коэффициент (защита от потерь)
   - Получаем тариф через свой GET /tariffs/{id} (из кэша)
   - Считаем: `(duration - free_period) * price_per_hour / 60`
   - Применяем жадный коэффициент если был фоллбэк
   - Возвращаем итоговую сумму

**Структура:**
```
services/pricing/
├── app.py              # FastAPI приложение
└── (кэш в памяти)      # tariff_cache, tariff_cache_ts
```

**Почему не управляем тарифами:**
- Тарифы живут во внешнем `tariffs` (дано в задании)
- Pricing только кэширует, не создаёт/изменяет
- Не дублируем логику

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

## База данных

### Почему SQLite

**Для учебного проекта:**
- Не нужно поднимать отдельный сервер БД
- Простая настройка и есть опыт работы с ней

**Ограничения SQLite:**
- Лимит размера БД ~140 ГБ (у нас будет 630 ГБ за 2 года)
- Нет репликации и шардирования

**Для прода нужна Postgres:**
- Поддерживает терабайты данных
- Нормальная конкурентная запись
- Репликация для отказоустойчивости
- Индексы работают быстрее

---

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
  "user_id": "user123",
  "rental_id": "rental999",
  "offer_id": "offer789",
  "message": "Rental started successfully",
  "duration_ms": 145
}
```

**Prometheus метрики:**
- **Rentals:** `offers_created_total`, `rentals_started_total`, `rentals_returned_total`
- **Pricing:** `tariff_cache_hits_total`, `tariff_cache_misses_total`, `tariff_stale_total`
- **Billing:** `debt_opened_total`, `debt_settled_total`
- **Эндпоинт:** `GET /metrics` (prometheus_client)

---

## Расчет ресурсов

### Исходные данные
- **х = 10 RPS** — одновременное создание заказов
- **Y = 100** — число просмотров информации об аренде на одну аренду
- **Z = 1 КБ** — размер записи об аренде
- **Хранение**: 2 года

### Rentals Service

**Нагрузка:**
- POST /offers: 10 RPS (перед каждой арендой создаём оффер)
- POST /rentals: 10 RPS (начало аренды)
- POST /rentals/{id}/return: 10 RPS (возврат)
- GET /rentals/{id}/summary: ~1000 RPS (Y=100 просмотров на аренду)

**Итого:** ~1030 RPS, из них 97% — чтение

**Данные в БД:**
- Таблица `offers`: 10 офферов/сек х 600 сек TTL = ~6000 активных офферов
  - Размер: 6000 х 0.2 КБ = 1.2 МБ (можно чистить старые)
- Таблица `rentals`: 864000 аренд/день х 730 дней = 630M записей
  - Размер: 630M х 1 КБ = 630 ГБ за 2 года

**Индексы:**
- `rental.id` (PRIMARY KEY) — для быстрого поиска в summary
- `rental.offer_id` (UNIQUE) — для идемпотентности создания аренды

### Pricing Service

**Нагрузка:**
- GET /tariffs/{id}: вызывается при каждом создании оффера = 10 RPS
  - С кэшем hit rate 95% -> нагрузка на внешний tariffs: 0.5 RPS
- GET /price/estimate: вызывается при summary и return = ~1010 RPS
  - Тарифы берутся из кэша

**Итого:** ~1020 RPS, почти всё из кэша

**Кэш тарифов:**
- Размер: ~100 тарифов х 0.5 КБ = 50 КБ
- TTL: 10 минут (настраивается через configs)
- Hit rate: 95%
- Экономия: с 10 RPS до 0.5 RPS на внешний сервис (в 20 раз)

**Config cache:**
- Размер: ~1 КБ
- Обновление: каждые 60 сек
- Нагрузка на configs: 1/60 RPS = 0.017 RPS

**Вывод:** Кэши критичны. Без них убьём внешние сервисы tariffs и configs.

### Billing Service

**Нагрузка:**
- POST /bill/{rental_id}: 10 RPS (при каждом возврате)
- POST /reconcile/{debt_id}: ~0.1 RPS (только когда payments был недоступен)

**Итого:** ~10 RPS

**Данные в БД:**
- Таблица `debts`: зависит от доступности payments
  - Если payments доступен 99.9% времени -> ~860 долгов/день
  - За 2 года: 860 х 730 = 628000 долгов
  - Размер: 628000 х 0.3 КБ = 188 МБ

**Вывод:** Минимальная нагрузка. Долгов мало, если payments работает стабильно.

### База данных (общая)

**Итоговый объём за 2 года:**
- Rentals: 630 ГБ (основное)
- Debts: 188 МБ
- Offers: 1.2 МБ (активные)
- **Итого: ~630 ГБ**

**Проблема SQLite:** лимит ~140 ГБ, нужно 630 ГБ

**Для прода:** Postgres + партиционирование по дате + старые аренды в архив

**Для дз:** SQLite достаточно (реальных 2 лет данных не будет)

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