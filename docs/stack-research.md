# mqtt-mcp-bridge: ресерч стека

Данные сверены 21.11.2026: версии и даты релизов взяты с proxy.golang.org (`@latest`), звёзды, лицензии и последний push — из GitHub API. Даты указаны по UTC. Звёзды округлены. Актуальный Go: **go1.27.4** (2026-11-02).

Легенда: ⭐ звёзды на GitHub · «push» — последний коммит в репозитории (не релиз).

---

## 1. MCP SDK

| Кандидат | Версия / релиз | ⭐ / лицензия | Плюсы | Минусы | Когда брать |
|---|---|---|---|---|---|
| **modelcontextprotocol/go-sdk** (официальный, с участием Google) | v1.8.0 / 2026-09-04, push 2026-11-20 | 5.4k / MIT → Apache-2.0 (переходный период) | Официальный. Поддерживает спеку **2026-07-28** и все предыдущие. stdio (`mcp.StdioTransport`), Streamable HTTP (`mcp.NewStreamableHTTPHandler` возвращает обычный `http.Handler`), stateless-режим, проверка bearer-токенов. **JSON Schema выводится из Go-структур** (`mcp.AddTool` + теги `jsonschema:"..."`). Стабильный v1 API | Звёзд меньше, чем у mark3labs. API строже и многословнее | По умолчанию: долгоживущий проект, хочется соответствия спеке |
| **mark3labs/mcp-go** | v1.6.0 / 2026-11-18, push 2026-11-19 | 9.2k / MIT | Самый популярный, большое сообщество и много примеров. stdio, SSE, Streamable HTTP (`StreamableHTTPServer` — тоже `http.Handler`), CORS и защита от DNS rebinding. Удобный builder для инструментов | Спека **2025-11-25** (на одну версию отстаёт от официального). README предупреждает: «under active development». Схемы описываются в основном через builder (`mcp.WithString(...)`), а не выводятся из структур | Нужно быстро начать по множеству примеров из интернета |
| metoro-io/mcp-golang | v0.16.1 / 2026-02-25 | 1.2k / MIT | Схемы из структур | Активность низкая (последний push в феврале), v0 | Не рекомендую |
| ThinkInAIXYZ/go-mcp | v0.2.30 / 2026-11-15 | 0.7k / MIT | Активный | v0, маленькое сообщество | Не рекомендую |

**Если хочешь «правильно и надолго» — go-sdk. Если важнее обилие примеров и туториалов — mark3labs/mcp-go.** Оба отдают `http.Handler`, поэтому роутер им не нужен.

Учиться (go-sdk):
- pkg.go.dev: https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp
- Документация по фичам: https://github.com/modelcontextprotocol/go-sdk/tree/main/docs (`server.md` — инструменты, `protocol.md` — транспорты, stdio/Streamable HTTP/stateless, auth)
- Примеры: https://github.com/modelcontextprotocol/go-sdk/tree/main/examples
- Спека MCP: https://modelcontextprotocol.io/specification
- Для сравнения mcp-go: https://mcp-go.dev , https://pkg.go.dev/github.com/mark3labs/mcp-go

---

## 2. MQTT-клиент

| Кандидат | Версия / релиз | ⭐ / лицензия | Протокол | Плюсы | Минусы | Когда брать |
|---|---|---|---|---|---|---|
| **eclipse/paho.golang** (`autopaho`) | v0.23.0 / 2025-09-06, push 2026-10-28 | 0.45k / EPL-2.0 + EDL-1.0 | **только MQTT 5** | Современный API на `context`. `autopaho` сам переподключается и переподписывается. Полная поддержка QoS1/2 с v0.20. Есть возможности v5: user properties, message expiry, shared subscriptions | Версия v0.x, между минорами бывают breaking changes. Звёзд мало. **MQTT 3.1.1 не поддерживает** | Брокер Mosquitto 2.x (MQTT 5 поддерживается), проект новый |
| **eclipse/paho.mqtt.golang** | v1.5.1 / 2025-09-16, push 2026-11-03 | 3.1k / EPL-2.0 + EDL-1.0 | 3.1 / 3.1.1 | Самый распространённый, проверен годами. Реконнект встроен (`SetAutoReconnect`, `SetConnectRetry`), примеров очень много | API на колбэках, без `context`, местами неочевидные гонки и блокировки в обработчиках. MQTT 5 нет | Нужен старый брокер или устройства только с 3.1.1 |
| (сервер) mochi-mqtt/server/v2 | см. п.5, это брокер, а не клиент | | | | | Только для тестов |

Проверить: на практике Mosquitto 2.x, Zigbee2MQTT и Home Assistant работают с клиентом MQTT 5 без проблем. Мост — отдельный клиент, поэтому версия протокола устройств ему безразлична. Значение имеет только **брокер**.

**Если брокер — Mosquitto 2.x или EMQX (MQTT 5), бери paho.golang/autopaho. Если нужен 3.1.1 или максимум готовых примеров — paho.mqtt.golang.**

Учиться:
- autopaho: https://pkg.go.dev/github.com/eclipse/paho.golang/autopaho
- Примеры (в т.ч. docker + mosquitto и router): https://github.com/eclipse/paho.golang/tree/master/autopaho/examples
- v3-клиент: https://pkg.go.dev/github.com/eclipse/paho.mqtt.golang , примеры: https://github.com/eclipse/paho.mqtt.golang/tree/master/cmd
- Спека MQTT 5: https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- Mosquitto: https://mosquitto.org/documentation/
- HA MQTT discovery: https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery
- Zigbee2MQTT MQTT-топики (`bridge/devices` и др.): https://www.zigbee2mqtt.io/guide/usage/mqtt_topics_and_messages.html

---

## 3. Хранилище истории

Нагрузка: много мелких вставок (timestamp, топик/устройство, значение) и запросы вида «мин/макс/среднее за период с шагом N». В SQL это делается одним `GROUP BY`, в KV-хранилищах — руками.

| Кандидат | Версия / релиз | ⭐ / лицензия | CGO | Плюсы | Минусы | Когда брать |
|---|---|---|---|---|---|---|
| **modernc.org/sqlite** | v1.62.0 / 2026-11-07 | ~0.5k (GitLab cznic/sqlite) / BSD-3 | **нет** | Чистый Go (C-код SQLite транслирован в Go). Статический бинарник, кросс-компиляция под arm/arm64 (Raspberry Pi) одной командой. Обычный `database/sql`. Агрегаты, индексы, WAL | Медленнее mattn примерно в 1.5–2 раза на тяжёлых запросах (для нашей нагрузки некритично). Бинарник крупнее | **По умолчанию** |
| ncruces/go-sqlite3 | v0.38.0 / 2026-11-21 | 1.1k / MIT | нет (SQLite в wasm через wazero) | Без CGO, активно развивается, есть свой API и `database/sql` | Версия v0, wasm-рантайм добавляет память и время старта, на 32-битном ARM ограничения — проверить | Альтернатива modernc, если чем-то не устроит |
| mattn/go-sqlite3 | v1.14.53 / 2026-11-05 | 9.3k / MIT | **да** | Самый быстрый и проверенный | Нужен CGO: кросс-компиляция под Pi требует тулчейн или сборку на самом Pi, статическая линковка сложнее | Если CGO не мешает |
| PostgreSQL (+TimescaleDB), драйвер jackc/pgx/v5 | pgx v5.13.0 / 2026-11-11 | 14.4k / MIT | нет | Ты его знаешь. `date_trunc`/`date_bin` и оконные функции. В TimescaleDB есть hypertables, `time_bucket`, сжатие и retention | **Внешняя зависимость**: ломает принцип «один бинарник», на Pi тяжеловато | Как опциональный backend, если у пользователя уже есть Postgres |
| go.etcd.io/bbolt | v1.5.0 / 2026-06-03 | 9.8k / MIT | нет | Простой встраиваемый KV, B+tree, ключи отсортированы, поэтому скан по диапазону времени эффективен | Агрегаты, retention и индексы пишешь сам. Одна транзакция записи за раз | Если хочется поучиться и написать свой мини-TSDB |
| dgraph-io/badger/v4 | v4.9.6 / 2026-08-05 | 15.8k / Apache-2.0 | нет | LSM, очень быстрые вставки, TTL на ключах (бесплатный retention) | Ест память, нужна фоновая GC значений, агрегаты вручную. Для Pi избыточен | Не для этого проекта |
| DuckDB (duckdb/duckdb-go/v2; marcboeker/go-duckdb заменён официальным) | v2.20000.0 / 2026-10-22 | 0.3k / MIT | **да** | Колоночная OLAP, аналитика по временным рядам очень быстрая | CGO и большая нативная библиотека, OLAP плохо подходит под поток мелких вставок, на Pi тяжёл | Не для MVP |
| In-memory (кольцевой буфер на `map` + slice) | stdlib | — | нет | Ноль зависимостей, быстро | История теряется при рестарте | Для тестов и режима «последние N часов» |

**Если «один бинарник, Pi, без CGO» — бери modernc.org/sqlite (+ in-memory для тестов). Если уже есть Postgres и хочешь использовать опыт — pgx (+TimescaleDB) как второй backend, но не единственный.**

Учиться:
- modernc: https://pkg.go.dev/modernc.org/sqlite (имя драйвера `"sqlite"`)
- SQLite: агрегаты https://sqlite.org/lang_aggfunc.html , даты https://sqlite.org/lang_datefunc.html , WAL https://sqlite.org/wal.html
- `database/sql`: https://go.dev/doc/database/ , https://pkg.go.dev/database/sql
- pgx: https://pkg.go.dev/github.com/jackc/pgx/v5 , TimescaleDB `time_bucket`: https://docs.timescale.com/api/latest/hyperfunctions/time_bucket/
- ncruces: https://pkg.go.dev/github.com/ncruces/go-sqlite3

---

## 4. HTTP-роутер

HTTP нужен для трёх маршрутов: `/mcp` (handler из SDK), опционально `/healthz` и `/metrics`. Плюс middleware для bearer-токена.

| Кандидат | Версия / релиз | ⭐ / лицензия | Плюсы | Минусы | Когда брать |
|---|---|---|---|---|---|
| **stdlib `net/http`** (`ServeMux` с паттернами Go 1.22+: `GET /healthz`, `{id}`) | вместе с Go | BSD | Ноль зависимостей. Методы и wildcard в паттернах. Middleware — обычная обёртка `func(http.Handler) http.Handler` | Нет готового набора middleware и группировки маршрутов | **Для этого проекта достаточно** |
| **go-chi/chi/v5** | v5.5.0 / 2026-10-14 | 23k / MIT | Ты его знаешь. 100% совместим с `net/http`. Группы маршрутов, готовые middleware (`Logger`, `Recoverer`, `Timeout`, `RealIP`) | Лишняя зависимость при 2–3 маршрутах | Если планируется REST API или веб-интерфейс сверх MCP |
| labstack/echo/v4 | v4.16.6 / 2026-11-20 | 33k / MIT | Мощный фреймворк | Свой `Context`, не `net/http`-native | Не нужно |
| gin-gonic/gin | v1.12.0 / 2026-02-28 | 90k / MIT | Популярный | Свой контекст, тянет много зависимостей | Не нужно |

**Если только `/mcp` + `/healthz` — stdlib `net/http`. Если хочешь привычный chi или будет REST — chi (handler SDK монтируется как `r.Handle("/mcp", h)`).**

Учиться:
- Паттерны ServeMux: https://go.dev/blog/routing-enhancements , https://pkg.go.dev/net/http#ServeMux
- chi: https://go-chi.io , https://pkg.go.dev/github.com/go-chi/chi/v5

---

## 5. Остальное

### Конфиг (YAML + переопределение через env)

| Кандидат | Версия / релиз | ⭐ / лицензия | Заметка |
|---|---|---|---|
| **go.yaml.in/yaml/v3** (v4) | v3.0.6 / 2026-10-13; v4.0.0 / 2026-11-04 | yaml/go-yaml 0.6k / Apache-2.0 | Официальный преемник `gopkg.in/yaml.v3`. **Старый `gopkg.in/yaml.v3` (v3.0.1, 2022) — репозиторий go-yaml/yaml в архиве.** API тот же |
| goccy/go-yaml | v1.19.2 / 2026-01-08 | 2.2k / MIT | Хорошие сообщения об ошибках с номером строки, валидация |
| **caarlos0/env/v11** | v11.4.2 / 2026-11-02 | 6.4k / MIT | Env → struct по тегам. Сочетается с YAML |
| knadh/koanf/v2 | v2.4.1 / 2026-10-18 | 4.2k / MIT | Слои: файл + env + флаги, модульный, мало зависимостей |
| spf13/viper | v1.21.0 / 2025-09-08 | 31k / MIT | Всё в одном, но тяжёлый и с неявной магией |

**Если конфиг простой — yaml (go.yaml.in) + caarlos0/env. Если хочешь слои file/env/flags из коробки — koanf.** Viper для такого бинарника избыточен.

### Логирование

| Кандидат | Версия | Заметка |
|---|---|---|
| **log/slog** (stdlib, Go 1.21+) | вместе с Go | Структурированные логи, JSON/Text handler, без зависимостей |
| rs/zerolog | v1.35.1 / 2026-04-20 (12.5k, MIT) | Zero-alloc, самый быстрый |
| go.uber.org/zap | v1.28.0 / 2026-04-28 (24.7k, MIT) | Быстрый, многословный API |

**Бери slog.** Нагрузка моста не та, чтобы скорость логгера имела значение. Важно: при stdio-транспорте MCP логи писать **только в stderr**, stdout занят протоколом.
Учиться: https://go.dev/blog/slog , https://pkg.go.dev/log/slog

### CLI-флаги

| Кандидат | Версия | Заметка |
|---|---|---|
| **flag** (stdlib) | — | Хватает для `--config`, `--check`, `--transport` |
| spf13/cobra | v1.10.2 / 2025-12-03 (45k, Apache-2.0) | Подкоманды, автодополнение. Избыточен, пока подкоманд нет |
| urfave/cli/v3 | v3.14.0 / 2026-10-02 (24k, MIT) | Средний вариант |
| alecthomas/kong | v1.17.0 / 2026-10-30 (3.2k, MIT) | Флаги из struct-тегов, лаконично |

**Если 3–5 флагов — `flag`. Если появятся подкоманды (`serve`, `check`, `devices`) — kong или cobra.**
Учиться: https://pkg.go.dev/flag

### Разбор JSON-путей (payload → значение)

| Кандидат | Версия | Заметка |
|---|---|---|
| **tidwall/gjson** | v1.20.5 / 2026-11-20 (15.6k, MIT) | Свой синтаксис `a.b.0.c`, очень быстрый, без анмаршалинга |
| ohler55/ojg | v1.28.9 / 2026-10-27 (1k, MIT) | Настоящий JSONPath (`$.a.b[0]`), стандартный синтаксис |
| PaesslerAG/jsonpath | v0.1.1 / 2019 | Заброшен, не брать |
| `encoding/json` + `map[string]any` | stdlib | Достаточно, если пути простые (`temperature`, `state`) |

**Если пользователи будут писать пути в конфиге — gjson (проще синтаксис). Если нужен стандартный JSONPath — ojg.**
Учиться: https://github.com/tidwall/gjson/blob/master/SYNTAX.md

### Миграции (если SQL)

| Кандидат | Версия | Заметка |
|---|---|---|
| **pressly/goose/v3** | v3.30.1 / 2026-11-12 (11.7k, MIT) | Миграции через `embed.FS` прямо в бинарнике, есть SQLite (modernc) и Postgres, удобная библиотечная API |
| golang-migrate/migrate/v4 | v4.20.1 / 2026-09-09 (19k, MIT) | Популярный. Драйвер `sqlite3` использует mattn (CGO), драйвер `sqlite` — modernc (без CGO) |
| Свой мини-раннер (`PRAGMA user_version` + embed) | — | Для 2–3 таблиц достаточно, заодно хорошая практика |

**Если хочется готового — goose с `embed`. Если таблиц 1–3 — свой раннер на `user_version`.**
Учиться: https://pressly.github.io/goose/ , https://pkg.go.dev/embed

### Тесты

| Кандидат | Версия | Заметка |
|---|---|---|
| **testcontainers-go** + модуль `mosquitto` | v0.45.0 / 2026-11-05 (5k, MIT) | Настоящий Mosquitto в Docker. Нужен Docker в CI |
| **mochi-mqtt/server/v2** (встроенный брокер) | v2.7.9 / 2025-03-01 (2k, MIT) | Брокер in-process, MQTT 3.1.1 и 5, тесты без Docker. ⚠️ Последний коммит в main — март 2025, то есть проект фактически заморожен. Для тестов приемлемо |
| stretchr/testify | v1.12.1 / 2026-08-17 (26k, MIT) | `require`/`assert`. По желанию: можно обойтись stdlib `testing` |
| go-sdk: in-memory транспорт | — | `mcp.NewInMemoryTransports()` — тест MCP-инструментов без сети |

**Unit-тесты — mochi-mqtt in-process (быстро, без Docker). Интеграционные — testcontainers с Mosquitto (как в проде).**
Учиться: https://golang.testcontainers.org/modules/mosquitto/ , https://pkg.go.dev/github.com/mochi-mqtt/server/v2 , https://go.dev/doc/tutorial/add-a-test

### Генерация JSON Schema (если SDK не делает сам)
- go-sdk использует **google/jsonschema-go** (v0.4.3 / 2026-04-17) — вывод схем уже встроен.
- invopop/jsonschema (v0.14.0 / 2026-04-23) — если нужна отдельная генерация.

---

## Сводка: «если X — бери Y»

| Категория | Рекомендация |
|---|---|
| MCP | go-sdk v1.8.0 (официальный, спека 2026-07-28, схемы из структур). mark3labs/mcp-go — если важны примеры |
| MQTT | paho.golang/autopaho, если брокер MQTT 5 (Mosquitto 2.x). paho.mqtt.golang — если нужен 3.1.1 |
| БД | modernc.org/sqlite (без CGO, один бинарник, агрегаты в SQL) + in-memory для тестов. Postgres/pgx — опционально позже |
| Роутер | stdlib `net/http` (1.22+ паттерны). chi — если будет REST/UI или хочется привычного |
| Конфиг | go.yaml.in/yaml + caarlos0/env. koanf — если нужны слои |
| Логи | log/slog (в stderr при stdio) |
| CLI | stdlib flag. kong/cobra — при подкомандах |
| JSON-пути | gjson |
| Миграции | goose + embed, или свой на `PRAGMA user_version` |
| Тесты | mochi-mqtt in-process + testcontainers/mosquitto для интеграции |

### Что не удалось подтвердить точно
- Скорость modernc и mattn («в 1.5–2 раза») — это оценка по известным бенчмаркам, сегодня не перемерял.
- golang-migrate `database/sqlite` и `mcp.NewInMemoryTransports` проверены по исходникам и pkg.go.dev.
- Работа ncruces/wazero на 32-битном Raspberry Pi OS (armv7) не проверена.
