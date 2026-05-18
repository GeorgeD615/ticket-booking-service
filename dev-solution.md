# Техническое решение проекта «Сервис бронирования билетов»

## Введение
Необходимо спроектировать сервис бронирования билетов на мероприятия, который позволяет пользователю выбрать событие, посмотреть доступные места, забронировать билеты и подтвердить покупку.
Система должна поддерживать базовый сценарий:
- пользователь выбирает мероприятие;
- система показывает доступные места;
- пользователь выбирает одно или несколько мест;
- система временно резервирует выбранные места;
- пользователь подтверждает покупку;
- билеты переходят в состояние проданных либо бронь снимается по таймауту или отмене.

Цель проекта — предложить архитектуру highload-системы, способной корректно работать при высокой конкуренции за ограниченный ресурс, предотвращать двойную продажу одного и того же места и выдерживать всплески нагрузки в момент старта продаж.
 

---

## Глоссарий
| Термин | Определение |
|--------|------------|
| Событие | Концерт, спектакль, матч или другое мероприятие, на которое продаются билеты. |
| Площадка (Venue) | Физическое место проведения события с определённой схемой зала и набором мест. |
| Место | Конкретная позиция в зале, доступная для бронирования в рамках события. |
| Статус места | Состояние места: свободно, временно забронировано (reserved), продано. |
| Бронь (Booking) | Временное резервирование одного или нескольких мест за пользователем до оплаты. |
| Позиция брони | Элемент брони, связанный с конкретным местом события (booking item). |
| Заказ (Order) | Подтверждённая покупка после успешной оплаты, фиксирующая факт продажи билетов. |
| Элемент заказа | Детализация заказа по конкретным местам, включая snapshot данных на момент покупки. |
| Таймаут брони | Ограниченное время удержания мест за пользователем до автоматического освобождения. |
| Virtual Waiting Queue | Механизм управления доступом пользователей к системе при пиковых нагрузках (admin-enabled режим). |
| Oversell | Ситуация, при которой одно и то же место продаётся более одного раза. |

---

## Функциональные требования
Система должна предоставлять следующие функции:

1. Просмотр мероприятий
    - Система должна позволять пользователю:
        - просматривать список доступных мероприятий;
        - получать информацию о выбранном мероприятии;
        - видеть схему зала или список мест;
        - видеть текущую доступность мест.

    - Для каждого мероприятия должны быть доступны как минимум:
        - event_id;
        - название;
        - дата и время;
        - место проведения;
        - схема площадки или список мест.

2. Выбор мест
    - Система должна позволять пользователю:
        - выбрать одно или несколько мест;
        - получить информацию о цене выбранных мест;
        - начать процесс бронирования.

3. Временное резервирование мест
    - Система должна:
        - временно резервировать выбранные места за пользователем;
        - назначать время жизни брони;
        - запрещать одновременное успешное резервирование одного и того же места несколькими пользователями;
        - автоматически освобождать места после истечения таймаута, если покупка не была завершена.

4. Подтверждение покупки
    - Система должна позволять пользователю:
        - подтвердить покупку забронированных мест;
        - получить результат операции;
        - после успешного подтверждения перевести места в состояние sold.

5. Отмена брони
    - Система должна поддерживать снятие брони:
        - по явной отмене со стороны пользователя;
        - по истечении таймаута;
        - при неуспешном завершении покупки.

6. История заказов
    - Система должна позволять пользователю:
        - просматривать список своих заказов;
        - видеть статус заказа;
        - получать базовую информацию о мероприятии, местах и стоимости.

---

## Нефункциональные требования
1. Нагрузка
    - Система должна выдерживать:
        - до 1 000 операций бронирования в секунду в пике;
        - до 10 000 запросов в секунду на чтение доступности мест в момент старта продаж.
    - Основной характер нагрузки:
        - чтение доступности мест;
        - короткие конкурентные операции резервирования;
        - резкие всплески нагрузки на популярных событиях.
2. Производительность
    - Требования к производительности:
        - получение списка мест и их доступности — P95 не более 200 мс;
        - создание временной брони — P95 не более 300 мс;
        - подтверждение покупки — P95 не более 500 мс.
3. Надёжность
    - Система должна обеспечивать:
        - отсутствие потери подтверждённых покупок;
        - корректную работу при повторной отправке запросов;
        - устойчивость к сбоям отдельных экземпляров сервисов;
        - автоматическое освобождение “зависших” броней.
4. Консистентность
    - Для критичных операций требуется согласованность:
        - одно место не может быть одновременно успешно забронировано несколькими пользователями;
        - проданное место не может снова стать доступным без отдельной явной операции возврата;
        - подтверждённая покупка не должна приводить к oversell.
    - Для истории заказов допускается eventual consistency.
5. Масштабируемость
    - Система должна горизонтально масштабироваться по следующим контурам:
        - чтение каталога мероприятий;
        - чтение доступности мест;
        - операции бронирования;
        - хранение истории заказов.
---

## Пользовательские сценарии

### Сценарий: просмотр списка мероприятий

1. Пользователь запрашивает список доступных мероприятий. При необходимости пользователь применяет фильтры.
2. Система возвращает отфильтрованный список мероприятий с базовой информацией.

### Сценарий: просмотр деталей мероприятия и схемы зала

1. Пользователь выбирает конкретное мероприятие из списка.
2. Система показывает детальную информацию, включая схему зала с отображением статусов мест.

### Сценарий: выбор свободных мест для бронирования

1. Пользователь на схеме зала выбирает одно или несколько свободных мест для бронирования, и нажимает кнопку "Забронировать"
2. Система проверяет, что все места ещё свободны и не находятся в активной броне у другого пользователя.
3. Система переводит места в статус «забронировано» и назначает время жизни брони.

### Сценарий: подтверждение покупки

1. Пользователь с активной бронью нажимает «Подтвердить покупку».
2. Клиентское приложение отправляет запрос на подтверждение с `booking_id` и платёжными данными.
3. Система проверяет, что бронь активна.
4. Система переводит места из статуса "забронировано" в статус "продано" и создаёт заказ с информацией о мероприятии и местах.
5. Система отправляет пользователю подтверждение с деталями заказа.

### Сценарий: отмена брони пользователем до оплаты

1. Пользователь нажимает кнопку «Отменить бронь» в интерфейсе активной брони.
2. Система проверяет, что бронь существует и принадлежит этому пользователю.
3. Система переводит места из статуса "забронировано" обратно в статус "свободно".
4. Пользователь видит подтверждение, того что бронь отменена.

### Сценарий: автоматическое снятие брони по таймауту

1. Пользователь зарезервировал места, но не подтвердил покупку в течение заданного времени.
2. Для истёкшей брони система переводит связанные места из статуса из статуса "забронировано" обратно в статус "свободно".
4. Освобождённые места снова становятся видны другим пользователям как доступные.
5. Пользователь при попытке оплатить после таймаута получает ошибку: «Время брони истекло. Пожалуйста, выберите места заново».

### Сценарий: просмотр истории заказов пользователя

1. Пользователь запрашивает историю заказов.
3. Система возвращает список заказов пользователя с краткой информацией по каждому.

### Сценарий: просмотр деталей конкретного заказа

1. Пользователь в истории заказов выбирает конкретный заказ.
2. Система возвращает полную информацию о заказе.

---

## API Методы

### Мероприятия

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/events` | Список доступных мероприятий (с фильтрацией) |
| GET | `/events/{event_id}` | Детальная информация о мероприятии, включая схему мест |

### Бронирование

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| POST | `/bookings` | Создать временную бронь выбранных мест |
| POST | `/bookings/{booking_id}/deny` | Отменить бронь |
| POST | `/bookings/{booking_id}/confirm` | Подтвердить покупку и завершить заказ |

### Заказы

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/orders` | История заказов пользователя |
| GET | `/orders/{order_id}` | Детальная информация о заказе |

---

## Модель данных (Data Model)

В основе сервиса бронирования лежит управление состояниями мест и заказов. Ключевое требование — предотвращение double-booking (oversell) при высоком конкурентном доступе. Для этого используется комбинация реляционной БД (PostgreSQL) для гарантий ACID и Redis для распределённых блокировок с автоматическим TTL и кэширования доступности мест.

### Основные сущности (Схема БД)

```mermaid
erDiagram
    direction LR

    USER {
        uuid id PK
    }

    VENUE {
        uuid id PK
        string name
        json address
        json layout
    }

    VENUE_SEAT {
        uuid id PK
        uuid venue_id FK
        string section
        string row_number
        string seat_number
    }

    EVENT {
        uuid id PK
        uuid venue_id FK
        string name
        timestamp event_date
    }

    EVENT_SEAT {
        uuid id PK
        uuid event_id FK
        uuid venue_seat_id FK
        uuid booking_id FK
        uuid order_id FK
        int status
        decimal price
    }

    BOOKING {
        uuid id PK
        uuid user_id FK
        int status
        timestamp created_at
        timestamp expires_at
    }

    BOOKING_ITEM {
        uuid id PK
        uuid booking_id FK
        uuid event_seat_id FK
        decimal price
    }

    ORDER {
        uuid id PK
        uuid user_id FK
        uuid booking_id FK
        uuid event_id FK
        int status
        decimal total_amount
        timestamp created_at
    }

    ORDER_ITEM {
        uuid id PK
        uuid order_id FK
        uuid event_seat_id FK
        string section
        string row_number
        string seat_number
        decimal price
    }

    USER ||--o{ BOOKING : creates
    USER ||--o{ ORDER : purchases

    VENUE ||--o{ VENUE_SEAT : contains
    VENUE ||--o{ EVENT : hosts

    EVENT ||--o{ EVENT_SEAT : includes
    VENUE_SEAT ||--o{ EVENT_SEAT : mapped_to

    BOOKING ||--o{ BOOKING_ITEM : contains
    EVENT_SEAT ||--o{ BOOKING_ITEM : reserved_as

    ORDER ||--o{ ORDER_ITEM : contains
    EVENT_SEAT ||--o{ ORDER_ITEM : sold_as

    BOOKING ||--o{ EVENT_SEAT : temporarily_reserves
    ORDER ||--o{ EVENT_SEAT : permanently_sells
```

### Описание сущностей

1. **`EVENT`**: Хранит информацию о мероприятии (название, дата и время проведения, площадка). Используется в сценариях просмотра каталога мероприятий, получения информации о событии и отображения схемы мест.

2. **`VENUE`**: Содержит информацию о площадке проведения мероприятия (название, адрес). Определяет физическую структуру зала, в рамках которой создаются места (`VENUE_SEAT`).

3. **`VENUE_SEAT`**: Представляет физическое место на площадке. Содержит сектор, ряд и номер места. Используется как шаблон для создания мест конкретного мероприятия (`EVENT_SEAT`).

4. **`EVENT_SEAT`**: Представляет конкретное место на конкретном мероприятии. Является основной сущностью для контроля доступности мест. Содержит статус (`free`, `sold`), цену, ссылки на бронь и заказ.

5. **`BOOKING`**: Фиксирует временное резервирование мест за пользователем. Содержит статус (`active`, `expired`, `cancelled`, `confirmed`), время создания и время истечения брони. Создаётся при выборе мест и завершается подтверждением покупки либо отменой.

6. **`BOOKING_ITEM`**: Связывает бронь с конкретными местами мероприятия (`EVENT_SEAT`). Позволяет одной брони содержать несколько мест. Хранит цену на момент резервирования.

7. **`ORDER`**: Создаётся после успешного подтверждения покупки. Представляет подтверждённый заказ пользователя и используется для хранения истории покупок. Содержит итоговую сумму, статус заказа и время оформления.

8. **`ORDER_ITEM`**: Связывает заказ с конкретными купленными местами. Хранит snapshot данных места (сектор, ряд, номер, цена) на момент покупки.

9. **`USER`**: Хранит идентификатор пользователя системы. Используется для привязки броней, заказов и получения истории покупок пользователя.

---

# Архитектура системы

Архитектура построена на микросервисной модели с использованием Redis для кэширования, распределённых блокировок и управления доступом в периоды высокой нагрузки. Основной фокус системы — поддержка 10 000 RPS на чтение доступности мест и до 1 000 операций бронирования в секунду при пиковых нагрузках.

## Основные компоненты

1. **Load Balancer** — входная точка системы, распределяет трафик между API Gateway инстансами, обеспечивает отказоустойчивость и базовую защиту от перегрузок.

2. **API Gateway** — единая точка входа для клиентов, отвечает за аутентификацию, rate limiting и маршрутизацию запросов к внутренним сервисам. Также участвует в проверке доступа пользователей при использовании Virtual Waiting Queue.

3. **Event Service** — сервис чтения данных о мероприятиях, возвращает информацию о событиях, venue и схеме зала. Использует Redis Cache для ускорения ответов и PostgreSQL как основной источник истины.

4. **Availability Service** — высоконагруженный read-сервис, отвечающий за отображение доступности мест и формирование схемы зала. Объединяет данные из PostgreSQL и Redis (включая временные блокировки).

5. **Search Service** — сервис поиска мероприятий, использующий PostgreSQL Full-Text Search на основе GIN индексов и tsvector.

6. **Booking Service** — основной сервис бронирования, отвечающий за управление временными резервированиями через Redis (SET NX EX), создание бронирований, транзакционные операции в PostgreSQL и финализацию заказов.

7. **Payment Adapter** — интеграционный слой для работы с платёжным провайдером (например, Stripe), обрабатывает вебхуки и передаёт события в Booking Service для завершения транзакций.

8. **Order History Service** — сервис для получения истории заказов пользователя, построенный по CQRS-подходу и работающий поверх PostgreSQL read-модели.

9. **PostgreSQL Cluster** — основное хранилище системы (source of truth), обеспечивает ACID-гарантии и предотвращение oversell через транзакции и механизмы блокировок.

10. **Redis Cluster** — используется для распределённых блокировок (seat reservations), кэширования часто запрашиваемых данных и хранения состояния виртуальной очереди (admitted users).

## Архитектурная схема

```mermaid
%%{init: {"themeVariables": {"fontSize": "22px"}}}%%
graph LR

  subgraph Client
    ClientApp[Client Application]
  end

  subgraph Edge_Layer
    LB[Load Balancer]
    APIGW[API Gateway]
  end

  subgraph Core_Services
    EventService[Event Service]
    AvailabilityService[Availability Service]
    SearchService[Search Service]
    BookingService[Booking Service]
    PaymentAdapter[Payment Adapter]
    OrderService[Order History Service]
  end

  subgraph Caching_Coordination
    RedisCache[(Redis Cache)]
    RedisLocks[(Redis Locks)]
    Admitted[(Redis Set admitted:eventId)]
    Queue[Virtual Waiting Queue]
  end

  subgraph Persistence
    Postgres[(PostgreSQL Cluster)]
  end

  subgraph External
    Stripe[Stripe]
  end

  %% Entry flow
  ClientApp -->|HTTPS request| LB
  LB -->|route request| APIGW

  %% Event flow
  APIGW -->|GET /events/:id| EventService
  EventService -->|cache lookup| RedisCache
  RedisCache -->|cache miss DB read| Postgres

  %% Availability flow
  APIGW -->|GET /availability| AvailabilityService
  AvailabilityService -->|read bitmap| RedisCache
  AvailabilityService -->|check locks| RedisLocks
  AvailabilityService -->|fallback read| Postgres

  %% Search flow (GIN / FTS)
  APIGW -->|GET /search| SearchService
  SearchService -->|Postgres full-text + GIN| Postgres

  %% Booking flow
  APIGW -->|POST /bookings| BookingService
  BookingService -->|check admitted| Admitted
  BookingService -->|Redis lock SET NX EX| RedisLocks
  BookingService -->|transaction write| Postgres
  BookingService -->|init payment| Stripe

  %% Payment flow
  Stripe -->|webhook payment event| PaymentAdapter
  PaymentAdapter -->|normalized event| BookingService

  %% Queue flow
  APIGW -->|join queue| Queue
  Queue -->|grant access| Admitted

  %% Orders
  APIGW -->|GET /orders| OrderService
  OrderService -->|read model| Postgres
```

### Паттерны и подходы

#### Распределённая блокировка Redis (SET NX EX)

Для временного резервирования мест и уменьшения конкурентных конфликтов используется паттерн **распределённой блокировки** на основе Redis. `Booking Service` при выборе мест выполняет атомарную команду `SET seat:lock:{event_id}:{seat_id} user_id NX EX ttl`, где `NX` позволяет установить ключ только при его отсутствии, а `EX` задаёт TTL блокировки (например, 600 секунд). Это гарантирует, что только один пользователь может временно зарезервировать место в каждый момент времени. Redis используется как механизм краткоживущих блокировок и автоматического освобождения мест по таймауту, тогда как PostgreSQL остаётся основным источником истины и гарантирует отсутствие oversell через транзакции и механизмы блокировок.

#### Кэширование

Для снижения нагрузки на базу данных и ускорения ответа API используется **кэширование часто запрашиваемых и редко изменяемых данных**. В Redis помещаются данные о мероприятиях, площадках и схемах залов. При запросе сначала выполняется чтение из кэша, и только при cache miss происходит обращение к PostgreSQL с последующим обновлением кэша. Для поддержания актуальности используются TTL и/или механизмы инвалидации при изменении данных в базе. Это позволяет существенно снизить нагрузку на БД и уменьшить latency для read-heavy сценариев.

#### Load Balancing

Для равномерного распределения входящего трафика используется **балансировка нагрузки**. Все клиентские запросы проходят через load balancer, который распределяет их между экземплярами API Gateway и микросервисов с использованием алгоритма Round Robin. Это позволяет избежать перегрузки отдельных инстансов, повысить устойчивость системы и обеспечить стабильную работу при пиковых нагрузках, особенно во время старта продаж популярных мероприятий.

#### Horizontal Scaling

Для обработки высокой нагрузки сервисы проектируются как **stateless компоненты**, что позволяет масштабировать их горизонтально путём добавления новых экземпляров. API Gateway, Event Service и Search Service могут быть увеличены по количеству инстансов в зависимости от текущей нагрузки. Балансировщик равномерно распределяет трафик между ними. Это обеспечивает возможность системы выдерживать резкие всплески трафика, характерные для популярных событий.

#### Virtual Waiting Queue (Admin-enabled)

Для обеспечения стабильной работы системы при экстремально высокой нагрузке используется **виртуальная очередь ожидания**, которая включается только для отдельных высоконагруженных мероприятий (admin-enabled режим). Перед доступом к странице бронирования пользователи помещаются в очередь, где управляется их порядок доступа к seat map и Booking Service. Очередь реализуется с использованием Redis (например, sorted set с timestamp), а клиент получает обновления через SSE или WebSocket в реальном времени.

После продвижения пользователя из очереди его sessionId добавляется в allowlist (`admitted:{eventId}`) с TTL, и только такие пользователи могут инициировать операции бронирования. Booking Service проверяет наличие пользователя в этом списке перед обработкой запросов. Это позволяет контролировать поток пользователей в систему, предотвращать перегрузку и обеспечивать более стабильный пользовательский опыт во время пиковых продаж.

#### Full-Text Search в PostgreSQL (GIN Index + tsvector)

Для улучшения производительности поиска по событиям в рамках SQL-базы используется встроенный механизм **full-text search в PostgreSQL** на основе `tsvector` и индекса типа **GIN (Generalized Inverted Index)**. Вместо медленного `LIKE '%query%'` применяется преобразование текстовых полей (name, description и др.) в `tsvector`, по которому выполняется индексированный поиск через `tsquery`.

Для ускорения запросов создаётся GIN индекс по полю `tsvector`, что позволяет эффективно выполнять поиск по ключевым словам без полного сканирования таблицы. Это значительно снижает latency по сравнению с `LIKE` и хорошо подходит для умеренных объёмов данных и базового уровня search-функциональности. 

Данный подход не требует отдельной поисковой инфраструктуры и обеспечивает компромисс между производительностью и сложностью системы, однако менее гибок по сравнению с специализированными search engines (например, Elasticsearch) и требует аккуратной настройки индексов и обновления tsvector при изменении данных.

---

## Технические сценарии

### Сценарий: Поиск и просмотр списка мероприятий (PostgreSQL GIN)
1. Клиент отправляет запрос поиска мероприятий через API Gateway (GET /search?query=...).
2. API Gateway перенаправляет запрос в Search Service.
3. Search Service формирует full-text запрос PostgreSQL (tsquery) на основе пользовательского input.
4. Поиск выполняется по предварительно подготовленному tsvector полю (name, description и другие текстовые поля).
5. PostgreSQL использует GIN index, чтобы избежать full table scan и быстро найти релевантные события.
6. Результаты возвращаются в Search Service без необходимости внешних поисковых систем (например, Elasticsearch).
7. Search Service возвращает список событий клиенту.

```mermaid
sequenceDiagram
  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant SearchService
  participant Postgres

  Client->>LB: GET /search?query=music festival
  LB->>APIGW: Route request

  APIGW->>SearchService: Search events

  SearchService->>Postgres: FTS query using tsquery on tsvector

  Note over Postgres: Uses GIN index for fast lookup

  Postgres-->>SearchService: Matching events

  SearchService-->>APIGW: Results
  APIGW-->>Client: 200 OK (events[])
```

### Сценарий: Просмотр деталей мероприятия и схемы зала
1. Клиент запрашивает детали мероприятия через API Gateway (GET /events/:eventId).
2. API Gateway проверяет, не включён ли Virtual Waiting Queue для данного события (admin-enabled режим).
3. Если очередь активна, пользователь должен быть предварительно допущен (admitted list в Redis).
4. Запрос передаётся в Event Service.
5. Event Service сначала выполняет чтение из Redis Cache (event details + venue + layout).
6. При cache miss данные загружаются из PostgreSQL и сохраняются в Redis.
7. Параллельно Availability Service предоставляет актуальный статус мест (bitmap + locks).
8. Клиент получает объединённый ответ: event + venue + seat availability.

```mermaid
sequenceDiagram
  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant Queue as Virtual Waiting Queue
  participant Admitted as Redis (Admitted Set)
  participant EventService
  participant AvailabilityService
  participant RedisCache
  participant RedisLocks
  participant Postgres

  Client->>LB: GET /events/{eventId}
  LB->>APIGW: Route request

  alt Queue enabled for event
    APIGW->>Admitted: Check session access

    alt Not admitted
      APIGW-->>Client: 403 / Redirect to Queue
    else Admitted
      APIGW->>EventService: Fetch event details
    end

  else Queue disabled
    APIGW->>EventService: Fetch event details
  end

  EventService->>RedisCache: GET event:{eventId}

  alt Cache Hit
    RedisCache-->>EventService: Event data
  else Cache Miss
    EventService->>Postgres: Read event + venue + layout
    Postgres-->>EventService: Data
    EventService->>RedisCache: SET event:{eventId} (TTL)
  end

  EventService-->>APIGW: Event + venue

  APIGW->>AvailabilityService: Get seat availability
  AvailabilityService->>RedisCache: GET bitmap:{eventId}
  AvailabilityService->>RedisLocks: GET active locks

  AvailabilityService-->>APIGW: Seat map status
  APIGW-->>Client: 200 OK (event + seat map)
```

### Сценарий: выбор свободных мест для бронирования

1. Пользователь выбирает места на seat map и отправляет запрос `POST /bookings/select-seats`.
2. Запрос проходит через Load Balancer и API Gateway и направляется в `Booking Service`.
3. `Booking Service` пытается атомарно установить Redis-блокировки (`SET NX EX`) для каждого выбранного места.
4. Если все блокировки успешно установлены, создаётся запись `BOOKING` со статусом `IN_PROGRESS`, а выбранные места сохраняются в `BOOKING_SEAT`.
5. Пользователь получает `bookingId` и время жизни резервирования (TTL), в течение которого должен завершить оплату.
6. Если хотя бы одно место уже заблокировано другим пользователем, сервис возвращает ошибку `409 Conflict`, а успешно захваченные блокировки освобождаются.
7. Если пользователь не завершает оплату вовремя, Redis автоматически удаляет блокировки по TTL, и места снова становятся доступными для бронирования.
8. Финальная защита от oversell обеспечивается PostgreSQL через транзакции и механизмы блокировок при подтверждении покупки.

```mermaid
%%{init: {"themeVariables": {"fontSize": "22px"}}}%%
sequenceDiagram

  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant Booking as Booking Service
  participant RedisLocks as Redis Locks
  participant Postgres as PostgreSQL

  Client->>LB: POST /bookings/select-seats
  LB->>APIGW: Route request
  APIGW->>Booking: Reserve selected seats

  loop For each selected seat
    Booking->>RedisLocks: SET seat:lock:{eventId}:{seatId} userId NX EX 600
  end

  alt All locks acquired
    RedisLocks-->>Booking: OK

    Booking->>Postgres: INSERT booking (status=IN_PROGRESS)
    Booking->>Postgres: INSERT booking_seats

    Postgres-->>Booking: Booking created

    Booking-->>APIGW: bookingId + reservation TTL
    APIGW-->>Client: 200 OK (Seats Reserved)

    Note over RedisLocks: Locks automatically expire after TTL

  else Some seat already locked
    RedisLocks-->>Booking: LOCK_FAILED

    Booking-->>APIGW: Seat unavailable
    APIGW-->>Client: 409 Conflict
  end
```

### Сценарий: Подтверждение покупки

1. После успешного резервирования мест пользователь отправляет запрос `POST /bookings/{bookingId}/confirm`.
2. Запрос проходит через Load Balancer и API Gateway и направляется в `Booking Service`.
3. `Booking Service` проверяет существование активного бронирования (`BOOKING.status = IN_PROGRESS`) и валидность Redis-блокировок для всех мест.
4. Сервис инициирует оплату через Stripe и создаёт `PaymentIntent`.
5. Пользователь завершает оплату на стороне Stripe.
6. После успешной оплаты Stripe отправляет webhook в `Payment Adapter`.
7. `Payment Adapter` нормализует webhook и передаёт событие в `Booking Service`.
8. `Booking Service` открывает транзакцию в PostgreSQL и повторно проверяет статус мест.
9. В рамках транзакции:
   - `BOOKING.status` обновляется на `CONFIRMED`
   - создаётся запись `ORDER`
   - места в `BOOKING_SEAT` переводятся в состояние `SOLD`
10. После успешного commit Redis-блокировки удаляются вручную, чтобы освободить ресурсы раньше TTL.
11. Пользователь получает подтверждение покупки и может увидеть заказ через `Order History Service`.
12. Если во время подтверждения возникает конфликт или блокировки истекли, транзакция откатывается, а платёж компенсируется через refund.

```mermaid
%%{init: {"themeVariables": {"fontSize": "22px"}}}%%
sequenceDiagram

  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant Booking as Booking Service
  participant RedisLocks as Redis Locks
  participant Stripe
  participant Adapter as Payment Adapter
  participant Postgres as PostgreSQL

  Client->>LB: POST /bookings/{bookingId}/confirm
  LB->>APIGW: Route request
  APIGW->>Booking: Confirm booking

  Booking->>Postgres: Validate BOOKING (IN_PROGRESS)
  Booking->>RedisLocks: Check seat locks ownership

  alt Locks valid
    RedisLocks-->>Booking: Locks OK

    Booking->>Stripe: Create PaymentIntent
    Stripe-->>Client: Checkout / Payment Form

    Note over Client, Stripe: User completes payment

    Stripe->>Adapter: payment_succeeded webhook
    Adapter->>Booking: Normalized payment event

    Booking->>Postgres: BEGIN TRANSACTION
    Booking->>Postgres: UPDATE BOOKING status=CONFIRMED
    Booking->>Postgres: INSERT ORDER
    Booking->>Postgres: UPDATE BOOKING_SEAT status=SOLD
    Postgres-->>Booking: COMMIT

    Booking->>RedisLocks: DEL seat locks

    Booking-->>APIGW: Purchase confirmed
    APIGW-->>Client: 200 OK (order confirmed)

  else Lock expired / invalid
    RedisLocks-->>Booking: LOCK_INVALID

    Booking->>Stripe: Refund payment (if needed)

    Booking-->>APIGW: Booking failed
    APIGW-->>Client: 409 Conflict
  end
```

### Сценарий: отмена брони пользователем до оплаты

1. Пользователь отправляет запрос `DELETE /bookings/{bookingId}` или `POST /bookings/{bookingId}/cancel`.
2. Запрос проходит через Load Balancer и API Gateway и направляется в `Booking Service`.
3. `Booking Service` проверяет существование бронирования и его статус (`IN_PROGRESS`).
4. Сервис валидирует, что отмена выполняется владельцем бронирования и что оплата ещё не была подтверждена.
5. В PostgreSQL открывается транзакция:
   - `BOOKING.status` обновляется на `CANCELLED`
   - связанные записи `BOOKING_SEAT` переводятся в состояние `FREE`
6. После успешного commit сервис вручную удаляет Redis-блокировки (`DEL seat:lock:{eventId}:{seatId}`), не дожидаясь TTL.
7. Освобождённые места снова становятся доступны для других пользователей.
8. Пользователь получает подтверждение успешной отмены бронирования.

```mermaid
%%{init: {"themeVariables": {"fontSize": "22px"}}}%%
sequenceDiagram

  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant Booking as Booking Service
  participant RedisLocks as Redis Locks
  participant Postgres as PostgreSQL

  Client->>LB: DELETE /bookings/{bookingId}
  LB->>APIGW: Route request
  APIGW->>Booking: Cancel booking

  Booking->>Postgres: Validate BOOKING status=IN_PROGRESS

  alt Booking active
    Booking->>Postgres: BEGIN TRANSACTION
    Booking->>Postgres: UPDATE BOOKING status=CANCELLED
    Booking->>Postgres: UPDATE BOOKING_SEAT status=FREE
    Postgres-->>Booking: COMMIT

    Booking->>RedisLocks: DEL seat locks

    Booking-->>APIGW: Booking cancelled
    APIGW-->>Client: 200 OK

  else Booking already confirmed / expired
    Booking-->>APIGW: Cancellation rejected
    APIGW-->>Client: 409 Conflict
  end
```

### Сценарий: автоматическое снятие брони по таймауту

1. После выбора мест `Booking Service` создаёт Redis-блокировки (`SET NX EX`) с TTL, например 10 минут.
2. Одновременно в PostgreSQL создаётся запись `BOOKING` со статусом `IN_PROGRESS`.
3. Если пользователь не завершает оплату до истечения TTL, Redis автоматически удаляет блокировки мест.
4. После исчезновения Redis lock места перестают считаться временно зарезервированными и снова отображаются как доступные через `Availability Service`.
5. Фоновый процесс (`Booking Cleanup Job`) периодически сканирует PostgreSQL на наличие просроченных бронирований со статусом `IN_PROGRESS`.
6. Для всех истёкших бронирований сервис обновляет статус `BOOKING` на `EXPIRED`.
7. Связанные записи `BOOKING_SEAT` переводятся обратно в состояние `FREE`.
8. После завершения cleanup места полностью возвращаются в пул доступных для покупки.

```mermaid
%%{init: {"themeVariables": {"fontSize": "22px"}}}%%
sequenceDiagram

  participant Client
  participant Booking as Booking Service
  participant RedisLocks as Redis Locks
  participant Availability as Availability Service
  participant Cleanup as Booking Cleanup Job
  participant Postgres as PostgreSQL

  Client->>Booking: Select seats
  Booking->>RedisLocks: SET seat lock NX EX 600
  Booking->>Postgres: INSERT BOOKING (IN_PROGRESS)

  Note over RedisLocks: TTL countdown starts

  alt User does not complete payment
    RedisLocks-->>RedisLocks: Auto expire locks

    Availability->>RedisLocks: Check active locks
    RedisLocks-->>Availability: Lock not found

    Availability-->>Client: Seats visible as FREE

    Cleanup->>Postgres: Find expired IN_PROGRESS bookings
    Cleanup->>Postgres: UPDATE BOOKING status=EXPIRED
    Cleanup->>Postgres: UPDATE BOOKING_SEAT status=FREE

    Postgres-->>Cleanup: Cleanup completed
  end
```

### Сценарий: просмотр истории заказов пользователя

1. Пользователь отправляет запрос `GET /orders?limit=20&cursor=...` для получения своей истории заказов.
2. Запрос проходит через `Load Balancer` и `API Gateway`.
3. `API Gateway` выполняет аутентификацию пользователя и маршрутизирует запрос в `Order History Service`.
4. `Order History Service` выполняет запрос к PostgreSQL read-модели (`ORDER`, `BOOKING`, `EVENT`) с использованием cursor-based пагинации.
5. PostgreSQL возвращает список заказов пользователя.
6. `Order History Service` формирует DTO-ответ и возвращает его клиенту.
7. Клиент сохраняет полученные данные локально (например, in-memory cache / browser storage) с коротким TTL.
8. При повторном открытии страницы клиент сначала отображает локально сохранённые данные и параллельно может выполнять background refresh для актуализации информации.

```mermaid
sequenceDiagram
  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant OrderService as Order History Service
  participant Postgres as PostgreSQL

  Client->>Client: Check local cache

  alt Local Cache Hit
    Client-->>Client: Show cached orders
    Client->>LB: Background refresh request
  else Cache Miss
    Client->>LB: GET /orders?limit=20&cursor=...
  end

  LB->>APIGW: Route request
  APIGW->>OrderService: Get user orders

  OrderService->>Postgres: SELECT user orders (cursor pagination)
  Postgres-->>OrderService: Orders list

  OrderService-->>APIGW: Orders response
  APIGW-->>Client: 200 OK (orders)

  Client->>Client: Update local cache
```

### Сценарий: просмотр деталей конкретного заказа

1. Пользователь открывает страницу заказа (`GET /orders/{orderId}`).
2. Клиент сначала проверяет наличие деталей заказа в локальном кэше.
3. При наличии данных (`cache hit`) клиент сразу отображает сохранённую информацию и параллельно может выполнять background refresh.
4. При отсутствии данных (`cache miss`) запрос отправляется через `Load Balancer` и `API Gateway`.
5. `API Gateway` выполняет аутентификацию пользователя и маршрутизирует запрос в `Order History Service`.
6. `Order History Service` выполняет запрос к PostgreSQL read-модели (`ORDER`, `BOOKING`, `EVENT`, `SEAT`).
7. PostgreSQL возвращает детали заказа, включая статус оплаты, информацию о мероприятии и список мест.
8. `Order History Service` формирует DTO-ответ и возвращает его клиенту.
9. Клиент сохраняет результат в локальном кэше с коротким TTL для ускорения последующих просмотров.

```mermaid
sequenceDiagram
  participant Client
  participant LB as Load Balancer
  participant APIGW as API Gateway
  participant OrderService as Order History Service
  participant Postgres as PostgreSQL

  Client->>Client: Check local cache (order details)

  alt Local Cache Hit
    Client-->>Client: Show cached order details
    Client->>LB: Background refresh request
  else Cache Miss
    Client->>LB: GET /orders/{orderId}
  end

  LB->>APIGW: Route request
  APIGW->>OrderService: Get order details

  OrderService->>Postgres: SELECT order + booking + seats + event
  Postgres-->>OrderService: Order details

  OrderService-->>APIGW: Order DTO response
  APIGW-->>Client: 200 OK (order details)

  Client->>Client: Update local cache
```
