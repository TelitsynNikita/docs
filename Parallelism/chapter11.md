# 🛑 Глава 11: Graceful shutdown — корректное завершение

**Что вы узнаете:**
- Почему `os.Exit(0)` и `Ctrl+C` — это **не** корректное завершение.
- Что такое **graceful shutdown** и какие проблемы он решает.
- Как ловить **сигналы ОС** через `signal.Notify` и `signal.NotifyContext`.
- Как **ждать завершения** всех горутин через `WaitGroup` и `errgroup`.
- Как **закрывать** HTTP-сервер без потери запросов.
- **Когда** обработчики должны проверять `r.Context()` — и когда это не нужно.
- Как **дренировать** очереди (Kafka, NATS, RabbitMQ).
- Что такое **shutdown timeout** и зачем он нужен.
- Как корректно завершать **БД-соединения**, **пулы**, **файлы**.
- Как комбинировать **graceful shutdown** с `context` (Глава 5), `errgroup` (Глава 10).
- Как тестировать graceful shutdown.

**После прочтения вы сможете:**
- Написать `main`, который корректно завершается по `SIGTERM`/`SIGINT`.
- Дождаться завершения всех горутин через `WaitGroup` или `errgroup`.
- Закрыть HTTP-сервер без обрыва активных запросов.
- Осознанно решать, **когда** обработчику нужен `r.Context()`.
- Дренировать очереди перед завершением.
- Настроить shutdown timeout, чтобы не висеть вечно.
- Освободить ресурсы: БД, файлы, пулы, соединения.
- Тестировать graceful shutdown через `context` и фейковые сигналы.

---

## Содержание

- [11.0 Пролог: деплой, который оборвал 10 000 запросов](#110-пролог-деплой-который-оборвал-10-000-запросов)
- [11.1 Почему os.Exit и Ctrl+C — не graceful](#111-почему-osexit-и-ctrlc--не-graceful)
- [11.2 Паттерн graceful shutdown](#112-паттерн-graceful-shutdown)
- [11.3 Сигналы ОС: signal.Notify и NotifyContext](#113-сигналы-ос-signalnotify-и-notifycontext)
- [11.4 Ожидание завершения: WaitGroup и errgroup](#114-ожидание-завершения-waitgroup-и-errgroup)
- [11.5 HTTP-сервер: Shutdown без потери запросов](#115-http-сервер-shutdown-без-потери-запросов)
- [11.6 Когда обработчику нужен r.Context()](#116-когда-обработчику-нужен-rcontext)
- [11.7 Дренаж очередей: Kafka, NATS, RabbitMQ](#117-дренаж-очередей-kafka-nats-rabbitmq)
- [11.8 Shutdown timeout: не висеть вечно](#118-shutdown-timeout-не-висеть-вечно)
- [11.9 Освобождение ресурсов](#119-освобождение-ресурсов)
- [11.10 Тестирование graceful shutdown](#1110-тестирование-graceful-shutdown)
- [11.11 Практика Go: HTTP-сервер с graceful shutdown](#1111-практика-go-http-сервер-с-graceful-shutdown)
- [11.12 Выводы и типичные ошибки](#1112-выводы-и-типичные-ошибки)
- [11.13 Для быстрого повторения](#1113-для-быстрого-повторения)
- [11.14 Вопросы для самопроверки](#1114-вопросы-для-самопроверки)
- [11.15 Ответы](#1115-ответы)
- [11.16 Куда идти дальше?](#1116-куда-идти-дальше)
- [11.17 Чек-лист](#1117-чек-лист)

---

## 11.0 Пролог: деплой, который оборвал 10 000 запросов

Ты пишешь HTTP-сервис на Go. Он работает в Kubernetes. Всё хорошо, пока не приходит время деплоя.

**Что происходит при деплое:**

1. Kubernetes отправляет **`SIGTERM`** процессу.
2. Kubernetes ждёт `terminationGracePeriodSeconds` (по умолчанию 30 секунд).
3. Если процесс не завершился — Kubernetes отправляет **`SIGKILL`**.
4. Процесс **убивается** мгновенно.

**Твой `main`:**

```go
func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Что происходит:**

- `SIGTERM` **игнорируется** — процесс не обрабатывает сигнал.
- `ListenAndServe` продолжает принимать запросы.
- Через 30 секунд — `SIGKILL`.
- **Все активные запросы обрываются.**

**Что видят пользователи:**

- 500-е ошибки.
- Пустые ответы.
- Timeout.

❓ **Как это исправить?**

💡 **Решение:** **graceful shutdown**. Перехватить `SIGTERM`, **перестать** принимать новые запросы, **дождаться** завершения активных, **закрыть** ресурсы, **выйти**.

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(), 
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Println("shutdown error:", err)
    }
}
```

**Что изменилось:**

- `SIGTERM` **перехватывается**.
- `srv.Shutdown` **перестаёт** принимать новые запросы.
- **Ждёт** завершения активных (до 30 секунд).
- Все запросы **завершаются корректно**.

> **Важный мост к будущим главам:** graceful shutdown использует `context` (Глава 5), `errgroup` (Глава 10), каналы (Глава 2). Это **финальная** тема, которая связывает всё вместе.

---

## 11.1 Почему os.Exit и Ctrl+C — не graceful

Прежде чем разбирать graceful shutdown, поймём, **почему** наивные способы плохи.

### `os.Exit(0)`

```go
func main() {
    go worker()
    os.Exit(0)  // ← мгновенный выход
}
```

**Что происходит:**

- **`defer` не выполняются.** Ресурсы не освобождаются.
- **Горутины убиваются.** `worker` не завершится.
- **Файлы не закрываются.** Утечка file descriptors.
- **Транзакции не откатываются.** Несогласованность.
- **HTTP-запросы обрываются.** 500-е ошибки.

### `Ctrl+C` (SIGINT) без обработки

```go
func main() {
    http.ListenAndServe(":8080", nil)
}
```

**Что происходит:**

- `SIGINT` **по умолчанию** завершает процесс.
- **Аналогично `os.Exit`** — горутины убиваются, ресурсы не освобождаются.

### `panic`

```go
func main() {
    panic("oops")
}
```

**Что происходит:**

- `defer` выполняются **в main**.
- Но горутины **убиваются**.
- Ресурсы **частично** освобождаются.

### `SIGKILL`

**`SIGKILL` нельзя перехватить.** Процесс убивается **мгновенно**. `defer`, горутины, ресурсы — всё теряется.

**Что делать:** Kubernetes/система отправляет `SIGTERM` **до** `SIGKILL`. Если процесс успевает завершиться за `terminationGracePeriodSeconds` — `SIGKILL` не придёт.

### Что теряется без graceful shutdown

| Ресурс | Без graceful | С graceful |
|:---|:---|:---|
| Активные HTTP-запросы | Оборваны | Завершены |
| Транзакции | Откатываются | Завершаются |
| Файлы | Не закрыты | Закрыты |
| БД-соединения | Разорваны | Закрыты |
| Очереди | Потеря сообщений | Дренаж |
| Метрики | Не отправлены | Отправлены |
| Логи | Не записаны | Записаны |

### Аналогия: закрытие магазина

**Без graceful shutdown** — как **внезапно выключить свет** в магазине. Покупатели в панике, касса не закрыта, товары не убраны.

**С graceful shutdown** — как **объявить «магазин закрывается»**. Новые покупатели не заходят, текущие спокойно завершают покупки, касса закрывается, свет выключается.

### Аннотация сложности

| Способ | Время завершения | Потери |
|:---|:---|:---|
| `os.Exit` | 0 | Всё |
| `SIGINT` без обработки | 0 | Всё |
| `SIGKILL` | 0 | Всё |
| Graceful shutdown | 1-30 сек | Ничего |

### 💡 Практика: почему graceful shutdown обязателен

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Graceful shutdown для production-сервисов.**
2. **Перехватывай `SIGTERM` и `SIGINT`.**
3. **Дожидайся завершения активных операций.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Shutdown timeout** — чтобы не висеть вечно (см. 11.8).
5. **Логирование** — что происходит при shutdown.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Graceful shutdown для CLI-утилит** — если они не держат ресурсы.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `os.Exit` в main** без причины.
8. **Не игнорируй `SIGTERM`.** Kubernetes убьёт через 30 секунд.
9. **Не рассчитывай на `SIGKILL`** — он не перехватывается.

### Ключевые выводы подглавы 11.1

- **`os.Exit` и `Ctrl+C`** — не graceful. Горутины убиваются, ресурсы не освобождаются.
- **`SIGKILL`** нельзя перехватить.
- **Graceful shutdown** — перехватить сигнал, дождаться завершения, освободить ресурсы.
- **Kubernetes** отправляет `SIGTERM` до `SIGKILL`.

---

## 11.2 Паттерн graceful shutdown

Разберём **паттерн** graceful shutdown.

### Четыре фазы

```
1. Получить сигнал (SIGTERM, SIGINT)
   ↓
2. Перестать принимать новую работу
   ↓
3. Дождаться завершения активной работы
   ↓
4. Освободить ресурсы и выйти
```

### Схема

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  Работа                                         │
│   │                                             │
│   │  ←── SIGTERM                                 │
│   ▼                                             │
│  Фаза 1: Перехват сигнала                       │
│   │                                             │
│   ▼                                             │
│  Фаза 2: Прекратить приём новой работы          │
│   │      (HTTP: Shutdown, Kafka: Pause)         │
│   ▼                                             │
│  Фаза 3: Дождаться завершения активной          │
│   │      (WaitGroup, errgroup)                  │
│   ▼                                             │
│  Фаза 4: Освободить ресурсы                     │
│         (БД, файлы, пулы)                       │
│   │                                             │
│   ▼                                             │
│  Выход                                          │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Шаг 1: перехват сигнала

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()  // ждём сигнал
```

**Что делает:** создаёт `ctx`, который отменяется при получении сигнала.

### Шаг 2: прекратить приём новой работы

**HTTP:**

```go
srv.Shutdown(shutdownCtx)
```

**Kafka:**

```go
consumer.Pause(consumer.Assignment())
```

**Worker pool:**

```go
close(tasksCh)  // закрыть входной канал
```

### Шаг 3: дождаться завершения

**`WaitGroup`:**

```go
wg.Wait()
```

**`errgroup`:**

```go
g.Wait()
```

### Шаг 4: освободить ресурсы

```go
db.Close()
file.Close()
pool.Close()
```

### Полный скелет

```go
func main() {
    // Инициализация
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // Запуск компонентов
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        runServer(ctx)
    }()

    // Ждём сигнал
    <-ctx.Done()
    log.Println("shutdown signal received")

    // Shutdown с таймаутом
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := shutdownComponents(shutdownCtx); err != nil {
        log.Println("shutdown error:", err)
    }

    // Ждём завершения горутин
    wg.Wait()
    log.Println("shutdown complete")
}
```

### Порядок важен

**Неправильный порядок:**

```
1. Закрыть БД
2. Дождаться HTTP-запросов  ← запросы обращаются к БД → ошибки
```

**Правильный порядок:**

```
1. Перестать принимать запросы
2. Дождаться завершения активных (они используют БД)
3. Закрыть БД
```

**Правило:** **сначала** перестань принимать работу, **потом** дождись завершения, **потом** закрывай ресурсы.

### Аннотация сложности

| Фаза | Time |
|:---|:---|
| Перехват сигнала | ~1 мкс |
| Прекратить приём | ~1-100 мс |
| Дождаться завершения | 0-30 сек |
| Освободить ресурсы | ~10-100 мс |

### 💡 Практика: как реализовать graceful shutdown

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Четыре фазы:** перехват → стоп приёма → ожидание → освобождение.
2. **Правильный порядок:** сначала стоп приёма, потом закрытие ресурсов.
3. **`signal.NotifyContext`** — для перехвата.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`WaitGroup` или `errgroup`** — для ожидания.
5. **`context.WithTimeout`** — для shutdown timeout.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Метрики** — сколько времени занял shutdown.

**❌ НЕ ДЕЛАЙ:**

7. **Не закрывай ресурсы до завершения активной работы.**
8. **Не забывай `wg.Wait()`.** Горутины останутся.

### Ключевые выводы подглавы 11.2

- **Четыре фазы:** перехват → стоп → ожидание → освобождение.
- **Правильный порядок** — критичен.
- **`signal.NotifyContext`** — для перехвата.
- **`WaitGroup` / `errgroup`** — для ожидания.

---

## 11.3 Сигналы ОС: signal.Notify и NotifyContext

Разберём **работу с сигналами**.

### signal.Notify

**`signal.Notify`** — регистрирует канал для получения сигналов.

```go
import "os/signal"

sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

sig := <-sigCh  // ждём сигнал
fmt.Println("received:", sig)
```

**Что делает:**

1. Создаёт канал.
2. Регистрирует его для `SIGINT` и `SIGTERM`.
3. Возвращает сигнал при получении.

### signal.NotifyContext (Go 1.16+)

**`signal.NotifyContext`** — обёртка, которая создаёт `ctx`, отменяемый по сигналу.

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

**Что делает:**

1. Создаёт `ctx` с `cancel`.
2. При сигнале — `cancel()` вызывается.
3. `stop()` — отменяет регистрацию (важно для тестов).

**Преимущество:** работает с `context`, легко комбинировать с другими `ctx`.

### Основные сигналы

| Сигнал | Значение | Когда приходит |
|:---|:---|:---|
| **`SIGINT`** | Ctrl+C | Пользователь нажал |
| **`SIGTERM`** | Termination | Kubernetes, systemd, `kill` |
| **`SIGHUP`** | Hangup | Терминал закрыт, reload config |
| **`SIGQUIT`** | Quit | Ctrl+\\, стек-дамп |
| **`SIGUSR1`** | User-defined | Приложение |
| **`SIGUSR2`** | User-defined | Приложение |
| **`SIGKILL`** | Kill | **Не перехватывается** |

### Что перехватывать

**Для HTTP-сервиса:**

```go
signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
```

**Почему:**

- `SIGINT` — Ctrl+C (локальная разработка).
- `SIGTERM` — Kubernetes, systemd (production).

**Для reload config:**

```go
signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)
```

**Что делать с `SIGHUP`:** перечитать конфиг, **не** завершаться.

### Обработка нескольких сигналов

**Первый сигнал** — graceful shutdown.
**Второй сигнал** — force exit.

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()
log.Println("graceful shutdown started, press Ctrl+C again to force exit")

forceCtx, forceStop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer forceStop()

select {
case <-forceCtx.Done():
    log.Println("force exit")
    os.Exit(1)
case <-time.After(30 * time.Second):
    log.Println("shutdown timeout")
}
```

### signal.Notify без NotifyContext

**Если Go < 1.16** — используй `signal.Notify`:

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go func() {
    sig := <-sigCh
    log.Println("received:", sig)
    cancel()
}()

<-ctx.Done()
```

### Ловушка: буферизованный канал

**❌ Плохо:**

```go
sigCh := make(chan os.Signal)  // небуферизованный
signal.Notify(sigCh, syscall.SIGINT)
```

**Что происходит:** если сигнал приходит, когда никто не читает — **теряется**.

**✅ Хорошо:**

```go
sigCh := make(chan os.Signal, 1)  // буфер на 1
signal.Notify(sigCh, syscall.SIGINT)
```

**Почему буфер:** сигнал **гарантированно** попадёт в канал.

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `signal.Notify` | ~1 мкс | ~100 байт |
| `signal.NotifyContext` | ~1 мкс | ~150 байт |
| Получение сигнала | ~1-10 мкс | 0 |

### 💡 Практика: как работать с сигналами

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`signal.NotifyContext`** — для Go 1.16+.
2. **`syscall.SIGINT` + `syscall.SIGTERM`** — минимально.
3. **Буферизованный канал на 1** — если `signal.Notify`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Второй сигнал = force exit.**
5. **`SIGHUP`** — для reload config.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`SIGUSR1` / `SIGUSR2`** — для кастомных действий.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй небуферизованный канал** для сигналов.
8. **Не пытайся перехватить `SIGKILL`.**
9. **Не забывай `stop()`** после `NotifyContext`.

### Ключевые выводы подглавы 11.3

- **`signal.Notify`** — регистрирует канал для сигналов.
- **`signal.NotifyContext`** — создаёт `ctx`, отменяемый по сигналу.
- **`SIGINT` + `SIGTERM`** — основные.
- **Буферизованный канал на 1** — для `Notify`.
- **`stop()`** — обязателен для `NotifyContext`.

---

## 11.4 Ожидание завершения: WaitGroup и errgroup

Разберём **ожидание завершения** всех горутин.

### WaitGroup

**Классический паттерн:**

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        worker(ctx, id)
    }(i)
}

wg.Wait()
```

**Что делает:**

- `wg.Add(1)` — увеличить счётчик.
- `wg.Done()` — уменьшить.
- `wg.Wait()` — ждать нуля.

**Плюсы:**

- Простой.
- Нет ошибок.

**Минусы:**

- Нет ошибок.
- Нет отмены.

### errgroup

**С обработкой ошибок:**

```go
g, ctx := errgroup.WithContext(ctx)

for i := 0; i < 10; i++ {
    i := i
    g.Go(func() error {
        return worker(ctx, i)
    })
}

if err := g.Wait(); err != nil {
    log.Println("error:", err)
}
```

**Что делает:**

- `g.Go` — запуск.
- `g.Wait` — ожидание + первая ошибка.
- `WithContext` — отмена при ошибке.

**Плюсы:**

- Обработка ошибок.
- Отмена.
- `SetLimit`.

**Минусы:**

- Чуть сложнее.

### Что выбрать

| Ситуация | Инструмент |
|:---|:---|
| Воркеры без ошибок | `WaitGroup` |
| Воркеры с ошибками | `errgroup` |
| Нужна отмена | `errgroup.WithContext` |
| Нужен лимит | `errgroup.SetLimit` |

### Ожидание shutdown

**Паттерн:**

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        runServer(ctx)
    }()

    go func() {
        defer wg.Done()
        runWorker(ctx)
    }()

    <-ctx.Done()
    log.Println("shutdown signal received")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    go func() {
        wg.Wait()
        cancel()
    }()

    <-shutdownCtx.Done()
    if shutdownCtx.Err() == context.DeadlineExceeded {
        log.Println("shutdown timeout exceeded")
    }
    log.Println("shutdown complete")
}
```

### Проблема: горутина не завершается

**❌ Плохо:**

```go
go func() {
    for {
        // бесконечный цикл без ctx.Done()
    }
}()
```

**Что происходит:** `wg.Wait()` **висит вечно**. Shutdown timeout спасает, но горутина утекает.

**✅ Хорошо:**

```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        case task := <-tasksCh:
            process(task)
        }
    }
}()
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `wg.Add` | ~5-10 нс | 0 |
| `wg.Done` | ~5-10 нс | 0 |
| `wg.Wait` | ~10-20 нс + ожидание | 0 |
| `g.Go` | ~100-200 нс | ~50 байт |

### 💡 Практика: как ждать завершения

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`WaitGroup`** — для воркеров без ошибок.
2. **`errgroup`** — для воркеров с ошибками.
3. **`ctx.Done()`** — в каждой горутине.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Shutdown timeout** — для защиты от зависания.
5. **Логирование** — что завершилось.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `wg.Done()`.** `Wait` не вернётся.
8. **Не игнорируй `ctx.Done()`.** Горутина утечёт.
9. **Не жди вечно** — используй timeout.

### Ключевые выводы подглавы 11.4

- **`WaitGroup`** — для воркеров без ошибок.
- **`errgroup`** — с обработкой ошибок.
- **`ctx.Done()`** — в каждой горутине.
- **Shutdown timeout** — защита от зависания.

---

## 11.5 HTTP-сервер: Shutdown без потери запросов

Разберём **graceful shutdown** для HTTP-сервера.

### http.Server.Shutdown

**`http.Server.Shutdown`** — метод для graceful shutdown:

1. **Прекращает** принимать новые соединения (закрывает слушающий сокет).
2. **Отменяет** `r.Context()` для всех активных запросов.
3. **Ждёт** завершения активных обработчиков.
4. **Возвращается**, когда все завершены или `ctx` отменён.

**Ключевое:** `srv.Shutdown` **не убивает** обработчики. Он **ждёт**, пока они завершатся **сами**. Отмена `r.Context()` — это **сигнал** обработчику, который он **может** (или не может) использовать.

### Шаг 1: создание сервера

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 30 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

**Что важно:**

- **`Handler`** — роутер.
- **`ReadTimeout`** — таймаут на чтение запроса.
- **`WriteTimeout`** — таймаут на запись ответа.
- **`IdleTimeout`** — таймаут на idle-соединения.

### Шаг 2: запуск сервера

```go
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()
```

**Ключевое:** `ListenAndServe` **блокируется**. Запускаем в горутине.

**Проверка ошибки:** `http.ErrServerClosed` — **нормальная** ошибка при shutdown.

### Шаг 3: ожидание сигнала

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()
log.Println("shutdown signal received")
```

### Шаг 4: graceful shutdown

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Println("shutdown error:", err)
}
```

### Шаг 5: полный пример

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(1 * time.Second)
        w.Write([]byte("hello"))
    })

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    go func() {
        log.Println("server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()

    log.Println("shutdown signal received")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Println("shutdown error:", err)
    } else {
        log.Println("shutdown complete")
    }
}
```

### Что происходит при shutdown

```
t=0:    SIGTERM получен
        ctx.Done() закрыт
        main: "shutdown signal received"

t=0:    srv.Shutdown(shutdownCtx) вызван
        - Слушающий сокет закрыт → новые соединения не принимаются
        - r.Context() для активных запросов отменён
        - srv.Shutdown ждёт завершения обработчиков

t=1s:   Активные запросы завершаются (сами или по отмене)
        srv.Shutdown возвращается

t=1s:   main: "shutdown complete"
        Программа выходит
```

### Что НЕ делает srv.Shutdown

**Важно понимать:**

1. **Не убивает** активные обработчики. Он **ждёт** их.
2. **Не прерывает** внешние вызовы. Только **отменяет** `r.Context()`.
3. **Не закрывает** БД, файлы, пулы. Это делает **твой код**.
4. **Не ждёт** фоновые горутины. Только HTTP-обработчики.

### Три уровня отмены

```
1. srv.Shutdown
   - Прекращает приём новых соединений
   - Отменяет r.Context() для активных
   - Ждёт завершения обработчиков

2. r.Context()
   - Отменяется при shutdown
   - Обработчик ДОЛЖЕН проверить его
   - Если не проверяет — продолжает работу

3. Внешние вызовы (http.Do, db.QueryContext)
   - Отменяются через r.Context()
   - Прерывают работу немедленно
   - Освобождают ресурсы
```

### Проблема: активные запросы долгие

**Что если запрос длится 1 минуту?**

- `srv.Shutdown` ждёт **все** активные запросы.
- Если `shutdownCtx` = 30 секунд — вернёт ошибку через 30 секунд.
- **Активные запросы обрываются.**

**Решение:** `srv.Close` для force close:

```go
if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Println("shutdown error:", err)
    srv.Close()
}
```

**`srv.Close`** — **немедленно** закрывает все соединения.

### Обработка контекста запроса

**Два подхода:**

**Подход 1: не проверять `r.Context()`**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    time.Sleep(1 * time.Second)
    w.Write([]byte("hello"))
}
```

**Подходит, если:** обработчик **быстрый** (< 100 мс) или не имеет внешних вызовов.

**Подход 2: проверять `r.Context()`**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case <-r.Context().Done():
        return
    case <-time.After(1 * time.Second):
        w.Write([]byte("hello"))
    }
}
```

**Подходит, если:** обработчик **долгий** или имеет внешние вызовы.

**Детально — в подглаве 11.6.**

### HTTP/2 и keep-alive

**HTTP/1.1 keep-alive:** соединение остаётся открытым.

**HTTP/2:** мультиплексирование.

**Что делает `srv.Shutdown`:**

- Отправляет **GOAWAY** для HTTP/2.
- Закрывает idle-соединения.
- Ждёт активные.

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `srv.ListenAndServe` | 0 (блокируется) | ~1 КБ |
| `srv.Shutdown` | 0-30 сек | 0 |
| `srv.Close` | ~1-10 мс | 0 |

### 💡 Практика: как делать graceful shutdown HTTP

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`srv.Shutdown`** — для graceful shutdown.
2. **`ListenAndServe` в горутине.**
3. **Проверка `http.ErrServerClosed`.**
4. **Shutdown timeout** — 10-30 секунд.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`ReadTimeout`, `WriteTimeout`, `IdleTimeout`** — для защиты от медленных клиентов.
6. **`srv.Close`** — если таймаут истёк.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **GOAWAY для HTTP/2** — автоматически.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `os.Exit`** при shutdown.
9. **Не забывай `http.ErrServerClosed`.** Иначе `log.Fatal`.
10. **Не игнорируй `r.Context()`** в долгих обработчиках (см. 11.6).

### Ключевые выводы подглавы 11.5

- **`srv.Shutdown`** — перестаёт принимать, **отменяет `r.Context()`**, ждёт активные.
- **`ListenAndServe` в горутине.**
- **`http.ErrServerClosed`** — нормальная ошибка.
- **Shutdown timeout** — 10-30 секунд.
- **`srv.Close`** — force close.
- **Три уровня отмены:** `srv.Shutdown` → `r.Context()` → внешние вызовы.

---

## 11.6 Когда обработчику нужен r.Context()

Ключевой вопрос: **когда** обработчик должен проверять `r.Context()`?

### Ответ: не всегда

**Само по себе** «закрыть приём новых, дать завершиться активным» — **корректное** поведение. `srv.Shutdown` **ждёт** активные обработчики. Если они **быстрые** — shutdown завершится быстро без всякой отмены.

**Отмена через `r.Context()` нужна**, когда:

1. **Обработчик делает долгую работу** (> 1 сек).
2. **Есть внешние вызовы** (HTTP, БД, gRPC).
3. **Shutdown timeout ограничен** (например, 30 сек).
4. **Есть побочные эффекты**, которые лучше не делать при shutdown.

### Сценарий 1: быстрый обработчик — отмена не нужна

```go
func handler(w http.ResponseWriter, r *http.Request) {
    data := cache.Get(r.URL.Path)  // быстро, из памяти
    w.Write(data)
}
```

**Что происходит при shutdown:**

- `r.Context()` отменяется.
- Обработчик **не проверяет** его.
- Обработчик завершается за **микросекунды**.
- `srv.Shutdown` ждёт **микросекунды**.

**Отмена не нужна.** Обработчик и так быстрый.

### Сценарий 2: обработчик с внешним API — отмена нужна

```go
func handler(w http.ResponseWriter, r *http.Request) {
    resp, err := http.Get("https://slow-api.example.com/data")  // 5 сек
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer resp.Body.Close()
    io.Copy(w, resp.Body)
}
```

**Что происходит при shutdown БЕЗ отмены:**

- `r.Context()` отменяется.
- Обработчик **не проверяет** его.
- `http.Get` **продолжает** ждать 5 секунд.
- `srv.Shutdown` ждёт **5 секунд**.
- Если shutdown timeout = 3 сек → **force close** → **обрыв запроса** → клиент получает **500**.

**Что происходит С отменой:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    req, err := http.NewRequestWithContext(r.Context(), "GET",
        "https://slow-api.example.com/data", nil)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        // context.Canceled — shutdown
        http.Error(w, "service shutting down", http.StatusServiceUnavailable)
        return
    }
    defer resp.Body.Close()
    io.Copy(w, resp.Body)
}
```

**Что происходит:**

- `r.Context()` отменяется.
- `http.Do` **видит** отмену и **прерывает** запрос.
- Обработчик возвращает **503** (Service Unavailable).
- `srv.Shutdown` завершается **мгновенно**.

### Валидный пример: сравнение

Покажем **два сервиса**: с отменой и без. Запустим, отправим запросы, сделаем shutdown, посмотрим разницу.

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "log"
    "net/http"
    "os/signal"
    "sync/atomic"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/slow-no-ctx", handlerNoCtx)
    mux.HandleFunc("/slow-with-ctx", handlerWithCtx)

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    go func() {
        log.Println("server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatal(err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    <-ctx.Done()
    log.Println("=== SIGTERM received ===")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    start := time.Now()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Printf("shutdown error: %v (elapsed: %v)", err, time.Since(start))
        srv.Close()
    } else {
        log.Printf("shutdown complete (elapsed: %v)", time.Since(start))
    }
}

// handlerNoCtx НЕ проверяет r.Context() — 5 секунд работы
func handlerNoCtx(w http.ResponseWriter, r *http.Request) {
    log.Println("[no-ctx] handler started")
    time.Sleep(5 * time.Second)  // имитация внешнего API
    log.Println("[no-ctx] handler finished")
    fmt.Fprintln(w, "no-ctx done")
}

// handlerWithCtx проверяет r.Context() — прерывается при shutdown
func handlerWithCtx(w http.ResponseWriter, r *http.Request) {
    log.Println("[with-ctx] handler started")
    select {
    case <-r.Context().Done():
        log.Println("[with-ctx] handler canceled")
        http.Error(w, "service shutting down", http.StatusServiceUnavailable)
        return
    case <-time.After(5 * time.Second):
        log.Println("[with-ctx] handler finished")
        fmt.Fprintln(w, "with-ctx done")
    }
}
```

**Как тестировать:**

```bash
# Терминал 1: запустить сервер
go run main.go

# Терминал 2: запустить медленный запрос без ctx
curl http://localhost:8080/slow-no-ctx &

# Терминал 3: запустить медленный запрос с ctx
curl http://localhost:8080/slow-with-ctx &

# Терминал 1: послать SIGTERM (Ctrl+C)
```

**Что увидим в логах (без ctx):**

```
[no-ctx] handler started
=== SIGTERM received ===
shutdown error: context deadline exceeded (elapsed: 3.000s)
[no-ctx] handler finished
```

**Что видно:**

- Shutdown ждал 3 секунды (timeout).
- Обработчик **не прервался** — ждал 5 секунд.
- `srv.Close()` **оборвал** соединение.
- Клиент получил **500** (или пустой ответ).

**Что увидим в логах (с ctx):**

```
[with-ctx] handler started
=== SIGTERM received ===
[with-ctx] handler canceled
shutdown complete (elapsed: 5ms)
```

**Что видно:**

- Shutdown завершился за **5 мс**.
- Обработчик **прервался**.
- Клиент получил **503** (корректный ответ).

### Когда отмена не нужна

**Пример: быстрый обработчик**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    data := cache.Get(r.URL.Path)  // 10 мкс
    w.Write(data)
}
```

**Что происходит при shutdown:**

- Обработчик завершится за **10 мкс**.
- `srv.Shutdown` подождёт **10 мкс**.
- Отмена **не нужна**.

**Вывод:** если обработчик **быстрый** (< 100 мс) и **локальный** — отмена не критична.

### Когда отмена нужна

**Критерии:**

1. **Долгая работа** (> 1 сек).
2. **Внешние вызовы** — HTTP, БД, gRPC.
3. **Shutdown timeout ограничен.**
4. **Побочные эффекты**, которые лучше не делать.

**Примеры:**

- Вызов внешнего API.
- Запрос в БД с долгим выполнением.
- ML-инференс.
- Отправка email.
- Запись в файл.

### Три уровня отмены

```
1. srv.Shutdown
   - Прекращает приём новых соединений
   - Отменяет r.Context() для активных
   - Ждёт завершения обработчиков

2. r.Context()
   - Отменяется при shutdown
   - Обработчик ДОЛЖЕН проверить его
   - Если не проверяет — продолжает работу

3. Внешние вызовы (http.Do, db.QueryContext)
   - Отменяются через r.Context()
   - Прерывают работу немедленно
   - Освобождают ресурсы
```

### Паттерн: проверка r.Context() в обработчике

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Проверка перед работой
    select {
    case <-ctx.Done():
        http.Error(w, "shutting down", http.StatusServiceUnavailable)
        return
    default:
    }

    // Внешний вызов с ctx
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            http.Error(w, "shutting down", http.StatusServiceUnavailable)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer resp.Body.Close()

    // Проверка после работы
    select {
    case <-ctx.Done():
        return
    default:
    }

    io.Copy(w, resp.Body)
}
```

### Как понять, что отмена сработала

**`context.Canceled`** — отмена через `cancel()` (включая `srv.Shutdown`).

```go
if errors.Is(err, context.Canceled) {
    // shutdown или отмена клиента
}
```

**`context.DeadlineExceeded`** — таймаут.

```go
if errors.Is(err, context.DeadlineExceeded) {
    // таймаут
}
```

### Аннотация сложности

| Сценарий | Время shutdown |
|:---|:---|
| Быстрый обработчик без ctx | ~10 мкс |
| Долгий обработчик без ctx | ~5 сек |
| Долгий обработчик с ctx | ~5 мс |

### 💡 Практика: когда использовать r.Context()

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Проверяй `r.Context()`** в долгих обработчиках (> 1 сек).
2. **Передавай `r.Context()`** во внешние вызовы (`http.NewRequestWithContext`, `db.QueryContext`).
3. **Обрабатывай `context.Canceled`** — возвращай 503.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверяй `ctx.Done()`** до и после долгой работы.
5. **Логируй отмену** — для диагностики.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Проверять `r.Context()` в быстрых обработчиках** — не критично.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй `r.Context()` в долгих обработчиках.** Shutdown будет долгим.
8. **Не используй `context.Background()`** в обработчике — используй `r.Context()`.

### Ключевые выводы подглавы 11.6

- **`srv.Shutdown` не убивает** обработчики — только **отменяет `r.Context()`** и **ждёт**.
- **Отмена нужна**, если обработчик **долгий** или имеет **внешние вызовы**.
- **Отмена не нужна**, если обработчик **быстрый** (< 100 мс) и **локальный**.
- **Три уровня отмены:** `srv.Shutdown` → `r.Context()` → внешние вызовы.
- **`context.Canceled`** — отмена. **503** — корректный ответ.

---

## 11.7 Дренаж очередей: Kafka, NATS, RabbitMQ

Разберём **дренаж очередей** при shutdown.

### Проблема: потеря сообщений

**Без graceful shutdown:**

- Consumer читает сообщение из Kafka.
- `SIGTERM` приходит.
- Процесс умирает.
- **Сообщение потеряно.**

**С graceful shutdown:**

- Consumer перестаёт читать новые сообщения.
- **Дожидается** обработки активных.
- **Коммитит** обработанные.
- Закрывает соединение.

### Паттерн: pause + drain + close

**Kafka (confluent-kafka-go):**

```go
func consume(ctx context.Context, consumer *kafka.Consumer) error {
    for {
        select {
        case <-ctx.Done():
            consumer.Pause(consumer.Assignment())
            log.Println("consumer paused")

            consumer.Close()
            return ctx.Err()
        default:
            msg, err := consumer.ReadMessage(100 * time.Millisecond)
            if err != nil {
                if errors.Is(err, kafka.ErrTimedOut) {
                    continue
                }
                return err
            }
            if err := process(ctx, msg); err != nil {
                log.Println("process error:", err)
                continue
            }
            consumer.CommitMessage(msg)
        }
    }
}
```

### Паттерн: worker pool + очередь

**Общий паттерн:**

```go
func consumeWithWorkers(ctx context.Context, consumer Consumer, numWorkers int) error {
    messagesCh := make(chan Message, 100)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for msg := range messagesCh {
                if err := process(ctx, msg); err != nil {
                    log.Println("process error:", err)
                }
                consumer.Commit(msg)
            }
        }()
    }

    go func() {
        defer close(messagesCh)
        for {
            select {
            case <-ctx.Done():
                return
            default:
                msg, err := consumer.Read(100 * time.Millisecond)
                if err != nil {
                    if errors.Is(err, ErrTimeout) {
                        continue
                    }
                    return
                }
                select {
                case messagesCh <- msg:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()

    wg.Wait()
    consumer.Close()
    return nil
}
```

### NATS

**NATS (nats.go):**

```go
func subscribe(ctx context.Context, nc *nats.Conn, subject string) error {
    sub, err := nc.SubscribeSync(subject)
    if err != nil {
        return err
    }

    for {
        select {
        case <-ctx.Done():
            sub.Unsubscribe()
            nc.Drain()
            return ctx.Err()
        default:
            msg, err := sub.NextMsg(100 * time.Millisecond)
            if err != nil {
                if errors.Is(err, nats.ErrTimeout) {
                    continue
                }
                return err
            }
            if err := process(ctx, msg); err != nil {
                log.Println("process error:", err)
            }
        }
    }
}
```

### RabbitMQ

**RabbitMQ (amqp091-go):**

```go
func consume(ctx context.Context, ch *amqp.Channel, queue string) error {
    msgs, err := ch.Consume(queue, "", false, false, false, false, nil)
    if err != nil {
        return err
    }

    for {
        select {
        case <-ctx.Done():
            ch.Close()
            return ctx.Err()
        case msg, ok := <-msgs:
            if !ok {
                return nil
            }
            if err := process(ctx, msg.Body); err != nil {
                msg.Nack(false, true)
                continue
            }
            msg.Ack(false)
        }
    }
}
```

### Сравнение

| Очередь | Метод остановки | Drain |
|:---|:---|:---|
| **Kafka** | `Pause` | Ручной |
| **NATS** | `Unsubscribe` | `Drain` |
| **RabbitMQ** | `Cancel` | Ручной |

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Pause consumer | ~1-10 мс |
| Drain | ~100 мс - 30 сек |
| Close | ~10-100 мс |

### 💡 Практика: как дренировать очереди

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Остановить приём новых сообщений.**
2. **Дождаться обработки активных.**
3. **Закрыть соединение.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`consumer.Drain()`** — если есть (NATS).
5. **`Pause`** — для Kafka.

**❌ НЕ ДЕЛАЙ:**

6. **Не убивай процесс без drain.**
7. **Не коммить до обработки.**

### Ключевые выводы подглавы 11.7

- **Дренаж очередей** — остановить приём, дождаться обработки, закрыть.
- **Kafka:** `Pause` + ручной drain.
- **NATS:** `Drain`.
- **RabbitMQ:** `Cancel` + ручной drain.

---

## 11.8 Shutdown timeout: не висеть вечно

Разберём **shutdown timeout**.

### Зачем нужен таймаут

**Проблема:** горутина может **зависнуть**. `wg.Wait()` будет ждать вечно.

**Решение:** **shutdown timeout**.

### Паттерн: timeout на shutdown

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Println("shutdown error:", err)
}
```

### Полный паттерн: graceful + force

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    srv := startServer()
    worker := startWorker(ctx)

    <-ctx.Done()
    log.Println("shutdown signal received")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    done := make(chan struct{})
    go func() {
        defer close(done)
        if err := srv.Shutdown(shutdownCtx); err != nil {
            log.Println("server shutdown error:", err)
        }
        worker.Wait()
    }()

    select {
    case <-done:
        log.Println("graceful shutdown complete")
    case <-shutdownCtx.Done():
        log.Println("shutdown timeout exceeded, force closing")
        srv.Close()
    }
}
```

### Размер shutdown timeout

**Рекомендации:**

- **Меньше `terminationGracePeriodSeconds`** в Kubernetes.
- Обычно **10-30 секунд**.
- **Не больше 60 секунд.**

**Пример:** K8s `terminationGracePeriodSeconds: 30`. Shutdown timeout — **25 секунд** (запас 5 секунд).

### Что делать при timeout

**Варианты:**

1. **Force close** — `srv.Close()`.
2. **Логировать** — что не успело.
3. **Дамп горутин** — для диагностики.

```go
case <-shutdownCtx.Done():
    log.Println("force closing")
    srv.Close()

    buf := make([]byte, 1<<20)
    n := runtime.Stack(buf, true)
    log.Printf("active goroutines:\n%s", buf[:n])
```

### Аннотация сложности

| Timeout | Когда |
|:---|:---|
| 5 сек | Быстрый shutdown |
| 10 сек | Стандарт |
| 30 сек | Kubernetes default |
| 60 сек | Максимум |

### 💡 Практика: как настраивать shutdown timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Shutdown timeout** — обязательно.
2. **Меньше `terminationGracePeriodSeconds`.**
3. **Логирование** — что не успело завершиться.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Force close** — при timeout.
5. **Дамп горутин** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

7. **Не жди вечно.** Timeout обязателен.
8. **Не ставь timeout больше `terminationGracePeriodSeconds`.**

### Ключевые выводы подглавы 11.8

- **Shutdown timeout** — обязателен.
- **Меньше `terminationGracePeriodSeconds`.**
- **Force close** при timeout.
- **Дамп горутин** — для диагностики.

---

## 11.9 Освобождение ресурсов

Разберём **освобождение** ресурсов при shutdown.

### Что нужно освободить

| Ресурс | Метод |
|:---|:---|
| **HTTP-сервер** | `srv.Shutdown` |
| **БД** | `db.Close` |
| **БД-pool** | `pool.Close` |
| **Redis** | `redis.Close` |
| **Kafka** | `consumer.Close` |
| **NATS** | `nc.Drain` |
| **Файлы** | `file.Close` |
| **Логгеры** | `logger.Sync` |
| **Метрики** | `exporter.Flush` |
| **Трассировка** | `tracer.Shutdown` |

### Порядок освобождения

**Важно:** сначала освободить **зависимые** ресурсы, потом **базовые**.

```
1. HTTP-сервер (Shutdown)
2. Воркеры (Wait)
3. Kafka consumer (Close)
4. БД (Close)
5. Метрики (Flush)
6. Логгер (Sync)
```

**Почему:** воркеры используют БД. Если закрыть БД первой — воркеры упадут.

### Паттерн: defer + shutdown

```go
func main() {
    db, err := openDB()
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // ... работа ...

    <-ctx.Done()
    // ...
}
```

**Важно:** `log.Fatal` вызывает `os.Exit`. `defer` **не выполнится**.

### Паттерн: явный shutdown

```go
func main() {
    db, err := openDB()
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // ... работа ...

    <-ctx.Done()

    if err := db.Close(); err != nil {
        log.Println("db close error:", err)
    }
    log.Println("shutdown complete")
}
```

### Синхронизация логов

**`logger.Sync()`** — сбросить буферы лога на диск.

```go
defer func() {
    if err := logger.Sync(); err != nil {
        fmt.Fprintln(os.Stderr, "logger sync error:", err)
    }
}()
```

### Метрики и трассировка

**Метрики (pushgateway):**

```go
defer func() {
    if err := pusher.Push(); err != nil {
        log.Println("metrics push error:", err)
    }
}()
```

**Трассировка (OpenTelemetry):**

```go
defer func() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := tracerProvider.Shutdown(ctx); err != nil {
        log.Println("tracer shutdown error:", err)
    }
}()
```

### Аннотация сложности

| Ресурс | Time |
|:---|:---|
| `srv.Shutdown` | 0-30 сек |
| `db.Close` | ~1-100 мс |
| `consumer.Close` | ~10-500 мс |
| `logger.Sync` | ~1-10 мс |
| `tracer.Shutdown` | ~1-100 мс |

### 💡 Практика: как освобождать ресурсы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Правильный порядок:** зависимые → базовые.
2. **`defer`** — для простых случаев.
3. **Явное освобождение** — для сложных.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`logger.Sync`** — для логов.
5. **`tracer.Shutdown`** — для трассировки.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `log.Fatal`** — `defer` не выполнится.
9. **Не закрывай БД до завершения воркеров.**
10. **Не забывай `logger.Sync`.**

### Ключевые выводы подглавы 11.9

- **Порядок:** зависимые → базовые.
- **`defer`** — для простых случаев.
- **`logger.Sync`** — для логов.
- **`log.Fatal`** — `defer` не выполнится.

---

## 11.10 Тестирование graceful shutdown

Разберём **тестирование** graceful shutdown.

### Проблема: сигналы в тестах

**`syscall.SIGINT`** в тестах **нельзя** послать напрямую. **Решение:** использовать `context` + фейковые сигналы.

### Паттерн: shutdown через context

**Вместо `signal.NotifyContext`** в main — передавай `ctx` извне.

```go
func Run(ctx context.Context) error {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Println(err)
        }
    }()

    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    return srv.Shutdown(shutdownCtx)
}

func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    if err := Run(ctx); err != nil {
        log.Fatal(err)
    }
}
```

### Тест graceful shutdown

```go
func TestGracefulShutdown(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())

    done := make(chan error, 1)
    go func() {
        done <- Run(ctx)
    }()

    time.Sleep(100 * time.Millisecond)

    resp, err := http.Get("http://localhost:8080")
    if err != nil {
        t.Fatal(err)
    }
    resp.Body.Close()

    cancel()

    select {
    case err := <-done:
        if err != nil {
            t.Fatal(err)
        }
    case <-time.After(5 * time.Second):
        t.Fatal("shutdown timeout")
    }
}
```

### Тест активного запроса

```go
func TestShutdownWaitsForActiveRequests(t *testing.T) {
    mux := http.NewServeMux()
    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(500 * time.Millisecond)
        w.Write([]byte("done"))
    })

    srv := &http.Server{Addr: ":8081", Handler: mux}
    go srv.ListenAndServe()

    ctx, cancel := context.WithCancel(context.Background())
    shutdownDone := make(chan error, 1)
    go func() {
        <-ctx.Done()
        shutdownCtx, c := context.WithTimeout(context.Background(), 5*time.Second)
        defer c()
        shutdownDone <- srv.Shutdown(shutdownCtx)
    }()

    respCh := make(chan string, 1)
    go func() {
        resp, err := http.Get("http://localhost:8081/slow")
        if err != nil {
            respCh <- ""
            return
        }
        body, _ := io.ReadAll(resp.Body)
        resp.Body.Close()
        respCh <- string(body)
    }()

    time.Sleep(100 * time.Millisecond)
    cancel()

    select {
    case body := <-respCh:
        if body != "done" {
            t.Fatalf("expected 'done', got %q", body)
        }
    case <-time.After(2 * time.Second):
        t.Fatal("request did not complete")
    }

    if err := <-shutdownDone; err != nil {
        t.Fatal(err)
    }
}
```

### Тест отмены обработчика

**Проверяем, что обработчик с `r.Context()` прерывается:**

```go
func TestHandlerCanceledByShutdown(t *testing.T) {
    canceled := make(chan struct{})

    mux := http.NewServeMux()
    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        select {
        case <-r.Context().Done():
            close(canceled)
            http.Error(w, "canceled", http.StatusServiceUnavailable)
        case <-time.After(5 * time.Second):
            w.Write([]byte("done"))
        }
    })

    srv := &http.Server{Addr: ":8082", Handler: mux}
    go srv.ListenAndServe()

    go http.Get("http://localhost:8082/slow")
    time.Sleep(100 * time.Millisecond)

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    srv.Shutdown(shutdownCtx)

    select {
    case <-canceled:
        // ОК: обработчик увидел отмену
    case <-time.After(1 * time.Second):
        t.Fatal("handler was not canceled")
    }
}
```

### Аннотация сложности

| Тест | Time |
|:---|:---|
| TestGracefulShutdown | ~500 мс |
| TestShutdownWaitsForActiveRequests | ~700 мс |
| TestHandlerCanceledByShutdown | ~200 мс |

### 💡 Практика: как тестировать graceful shutdown

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Run(ctx)`** — выноси shutdown в отдельную функцию.
2. **`context.WithCancel`** — для тестов.
3. **Проверка активного запроса.**
4. **Проверка отмены обработчика.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Тест timeout.**
6. **Тест нескольких сигналов.**

**❌ НЕ ДЕЛАЙ:**

7. **Не посылай `SIGINT` в тестах.** Используй `context`.

### Ключевые выводы подглавы 11.10

- **`Run(ctx)`** — для тестируемости.
- **`context.WithCancel`** — в тестах.
- **Проверка активных запросов.**
- **Проверка отмены обработчика.**

---

## 11.11 Практика Go: HTTP-сервер с graceful shutdown

Напишем **полный HTTP-сервер** с graceful shutdown, показывающий **разницу** между обработчиками с `r.Context()` и без.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    if err := Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }

    slog.Info("shutdown complete")
}

func Run(ctx context.Context) error {
    mux := http.NewServeMux()
    mux.HandleFunc("/fast", handleFast)
    mux.HandleFunc("/slow-with-ctx", handleSlowWithCtx)
    mux.HandleFunc("/slow-without-ctx", handleSlowWithoutCtx)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    serverErr := make(chan error, 1)
    go func() {
        slog.Info("server starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            serverErr <- err
        }
        close(serverErr)
    }()

    select {
    case <-ctx.Done():
        slog.Info("shutdown signal received")
    case err := <-serverErr:
        if err != nil {
            return fmt.Errorf("server error: %w", err)
        }
    }

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        start := time.Now()
        if err := srv.Shutdown(shutdownCtx); err != nil {
            slog.Error("server shutdown error", "error", err)
            srv.Close()
        }
        slog.Info("shutdown complete", "elapsed", time.Since(start))
    }()

    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        return nil
    case <-shutdownCtx.Done():
        slog.Warn("shutdown timeout exceeded, forcing close")
        srv.Close()
        return shutdownCtx.Err()
    }
}

// handleFast — быстрый обработчик, не нужен r.Context()
func handleFast(w http.ResponseWriter, r *http.Request) {
    slog.Info("fast handler")
    fmt.Fprintln(w, "fast done")
}

// handleSlowWithCtx — долгий обработчик с проверкой r.Context()
func handleSlowWithCtx(w http.ResponseWriter, r *http.Request) {
    slog.Info("slow-with-ctx handler started")

    select {
    case <-r.Context().Done():
        slog.Info("slow-with-ctx handler canceled")
        http.Error(w, "service shutting down", http.StatusServiceUnavailable)
        return
    case <-time.After(10 * time.Second):
        slog.Info("slow-with-ctx handler finished")
        fmt.Fprintln(w, "slow-with-ctx done")
    }
}

// handleSlowWithoutCtx — долгий обработчик БЕЗ проверки r.Context()
func handleSlowWithoutCtx(w http.ResponseWriter, r *http.Request) {
    slog.Info("slow-without-ctx handler started")
    time.Sleep(10 * time.Second)  // ← не проверяет ctx
    slog.Info("slow-without-ctx handler finished")
    fmt.Fprintln(w, "slow-without-ctx done")
}
```

### Что демонстрирует

1. **`Run(ctx)`** — тестируемость.
2. **`signal.NotifyContext`** — перехват сигналов.
3. **`srv.Shutdown`** — graceful.
4. **Shutdown timeout** — 5 секунд.
5. **`slog`** — structured logging.
6. **Три обработчика:**
   - `/fast` — мгновенный, ctx не нужен.
   - `/slow-with-ctx` — отменяется при shutdown.
   - `/slow-without-ctx` — НЕ отменяется, ждёт 10 секунд.

### Как тестировать

**Сценарий 1: только `/fast`**

```bash
# Запускаем сервер
go run main.go

# В другом терминале:
curl http://localhost:8080/fast
# fast done

# Ctrl+C в первом терминале
# shutdown complete (elapsed: ~1ms)
```

**Что видно:** shutdown мгновенный.

**Сценарий 2: `/slow-with-ctx`**

```bash
# В другом терминале:
curl http://localhost:8080/slow-with-ctx &

# Ctrl+C в первом терминале
# slow-with-ctx handler canceled
# shutdown complete (elapsed: ~2ms)
```

**Что видно:** shutdown мгновенный, обработчик отменён.

**Сценарий 3: `/slow-without-ctx`**

```bash
# В другом терминале:
curl http://localhost:8080/slow-without-ctx &

# Ctrl+C в первом терминале
# shutdown error: context deadline exceeded (elapsed: 5.000s)
# shutdown timeout exceeded, forcing close
# slow-without-ctx handler finished
```

**Что видно:**

- Shutdown ждал 5 секунд (timeout).
- Обработчик **не прервался** — ждал 10 секунд.
- `srv.Close()` **оборвал** соединение.
- Клиент получил **пустой ответ** или **500**.

### Ключевой вывод

**Без `r.Context()`:** shutdown долгий, force close, клиент получает ошибку.

**С `r.Context()`:** shutdown мгновенный, корректный ответ.

**Отмена нужна**, если обработчик **долгий**.

### Аннотация сложности

| Обработчик | Время shutdown | Ответ клиенту |
|:---|:---|:---|
| `/fast` без ctx | ~1 мс | 200 OK |
| `/slow-with-ctx` | ~2 мс | 503 |
| `/slow-without-ctx` | 5 сек (timeout) | Обрыв/500 |

### 💡 Практика: как писать production HTTP-сервер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`signal.NotifyContext`** — для сигналов.
2. **`Run(ctx)`** — для тестируемости.
3. **Shutdown timeout** — 10-30 секунд.
4. **Force close** — при timeout.
5. **`slog`** — structured logging.
6. **`r.Context()`** — в долгих обработчиках.

**👍 СТОИТ СДЕЛАТЬ:**

7. **`ReadTimeout`, `WriteTimeout`, `IdleTimeout`.**
8. **Метрики** — количество активных запросов.

**❌ НЕ ДЕЛАЙ:**

9. **Не используй `log.Fatal`** в Run.
10. **Не игнорируй `http.ErrServerClosed`.**
11. **Не забывай `srv.Close`** при timeout.
12. **Не игнорируй `r.Context()`** в долгих обработчиках.

### Ключевые выводы подглавы 11.11

- **`Run(ctx)`** — для тестируемости.
- **Shutdown timeout** — 5-30 секунд.
- **Force close** — при timeout.
- **`r.Context()`** — критично для долгих обработчиков.
- **`slog`** — structured logging.

---

## 11.12 Выводы и типичные ошибки

**Что мы узнали?**

Graceful shutdown — четыре фазы: перехват сигнала → стоп приёма → ожидание → освобождение. `signal.NotifyContext` для перехвата `SIGINT`/`SIGTERM`. `WaitGroup` и `errgroup` для ожидания. `srv.Shutdown` для HTTP — **закрывает сокет**, **отменяет `r.Context()`**, **ждёт** активные обработчики. **`r.Context()` нужен**, если обработчик долгий или имеет внешние вызовы. Дренаж очередей (Kafka: `Pause`, NATS: `Drain`). Shutdown timeout — обязательно. Освобождение ресурсов в правильном порядке. Тестирование через `context`.

**Типичные ошибки:**

- ❌ **Использовать `os.Exit`** в main. `defer` не выполняется.
- ❌ **Игнорировать `SIGTERM`.** K8s убьёт через 30 секунд.
- ❌ **Не перехватывать `SIGINT`.** Ctrl+C убьёт процесс.
- ❌ **Забыть `wg.Wait()`.** Горутины останутся.
- ❌ **Не использовать `ctx.Done()`** в горутинах. Утечка.
- ❌ **Закрыть БД до завершения воркеров.** Ошибки.
- ❌ **Не использовать shutdown timeout.** Висеть вечно.
- ❌ **Не логировать shutdown.** Непонятно, что произошло.
- ❌ **Не использовать `logger.Sync`.** Потеря логов.
- ❌ **Не проверять `http.ErrServerClosed`.** `log.Fatal` при shutdown.
- ❌ **Не проверять `r.Context()`** в долгих обработчиках. Shutdown долгий.
- ❌ **Использовать `context.Background()`** в обработчике. Потеря отмены.
- ❌ **Не тестировать graceful shutdown.**
- ❌ **Не логировать force close.**

---

## 11.13 Для быстрого повторения

- **Graceful shutdown** — четыре фазы: перехват → стоп → ожидание → освобождение.
- **`os.Exit`** — не graceful. `defer` не выполняется.
- **`signal.NotifyContext`** — для перехвата `SIGINT`/`SIGTERM`.
- **Буферизованный канал на 1** — для `signal.Notify`.
- **`WaitGroup`** — для воркеров без ошибок.
- **`errgroup`** — с обработкой ошибок.
- **`srv.Shutdown`** — закрывает сокет, **отменяет `r.Context()`**, ждёт активные.
- **`r.Context()`** — отменяется при shutdown. Нужен для долгих обработчиков.
- **Отмена не нужна**, если обработчик быстрый (< 100 мс).
- **Отмена нужна**, если обработчик долгий или имеет внешние вызовы.
- **Три уровня отмены:** `srv.Shutdown` → `r.Context()` → внешние вызовы.
- **`http.ErrServerClosed`** — нормальная ошибка.
- **`srv.Close`** — force close.
- **Дренаж очередей:** Kafka — `Pause`, NATS — `Drain`, RabbitMQ — `Cancel`.
- **Shutdown timeout** — обязательно. Меньше `terminationGracePeriodSeconds`.
- **Порядок освобождения:** зависимые → базовые.
- **`logger.Sync`** — для логов.
- **`log.Fatal`** — `defer` не выполнится.
- **Тестирование:** `Run(ctx)` + `context.WithCancel`.

---

## 11.14 Вопросы для самопроверки

1. Почему `os.Exit` и `Ctrl+C` — не graceful?
2. Что такое graceful shutdown? Четыре фазы?
3. Почему `SIGKILL` нельзя перехватить?
4. Как использовать `signal.NotifyContext`?
5. Какой сигнал посылает Kubernetes?
6. Что происходит при втором `SIGTERM`?
7. Почему буферизованный канал на 1 для сигналов?
8. Что выбрать: `WaitGroup` или `errgroup`?
9. Что делает `srv.Shutdown`? Три вещи.
10. Почему `http.ErrServerClosed` — нормальная ошибка?
11. Что делает `srv.Close`?
12. **Когда обработчику нужен `r.Context()`? Когда не нужен?**
13. **Что происходит, если обработчик не проверяет `r.Context()`?**
14. Как дренировать Kafka? NATS? RabbitMQ?
15. Что такое shutdown timeout? Зачем?
16. Почему порядок освобождения ресурсов важен?
17. Почему `log.Fatal` не выполняет `defer`?
18. Что делает `logger.Sync`?
19. Как тестировать graceful shutdown?
20. Что такое три уровня отмены?

---

## 11.15 Ответы

### Ответ 1

**`os.Exit`** — мгновенный выход. `defer` не выполняются, горутины убиваются, ресурсы не освобождаются.

**`Ctrl+C` (SIGINT) без обработки** — аналогично.

### Ответ 2

**Graceful shutdown** — корректное завершение.

**Четыре фазы:**
1. Перехват сигнала.
2. Прекратить приём новой работы.
3. Дождаться завершения активной работы.
4. Освободить ресурсы.

### Ответ 3

**`SIGKILL` нельзя перехватить**, потому что он **не доставляется** процессу. Ядро убивает процесс напрямую.

### Ответ 4

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

### Ответ 5

**Kubernetes** посылает `SIGTERM`. Ждёт `terminationGracePeriodSeconds` (30 сек). Потом `SIGKILL`.

### Ответ 6

**Второй `SIGTERM`** — обычно **force exit**. Можно ловить и делать `os.Exit(1)`.

### Ответ 7

**Буферизованный канал на 1** для сигналов, потому что сигнал может прийти, когда никто не читает. Буфер **гарантирует** доставку.

### Ответ 8

- **`WaitGroup`** — для воркеров без ошибок.
- **`errgroup`** — с обработкой ошибок и отменой.

### Ответ 9

**`srv.Shutdown` делает три вещи:**

1. **Закрывает слушающий сокет** — новые соединения не принимаются.
2. **Отменяет `r.Context()`** для всех активных запросов.
3. **Ждёт** завершения активных обработчиков.

**Важно:** `srv.Shutdown` **не убивает** обработчики. Он **ждёт**, пока они завершатся **сами**.

### Ответ 10

**`http.ErrServerClosed`** — нормальная ошибка при shutdown. `ListenAndServe` возвращает её, когда `Shutdown` был вызван.

### Ответ 11

**`srv.Close`** — немедленно закрывает все соединения.

### Ответ 12

**`r.Context()` нужен**, если:
- Обработчик **долгий** (> 1 сек).
- Есть **внешние вызовы** (HTTP, БД, gRPC).
- **Shutdown timeout ограничен.**
- Есть **побочные эффекты**, которые лучше не делать.

**Не нужен**, если:
- Обработчик **быстрый** (< 100 мс).
- Работа **локальная** (кэш, память).

### Ответ 13

**Если обработчик не проверяет `r.Context()`:**

- `srv.Shutdown` **отменяет** `r.Context()`.
- Но обработчик **не видит** отмену.
- Обработчик **продолжает** работу.
- `srv.Shutdown` **ждёт** его завершения.
- Если обработчик долгий — shutdown timeout истечёт → **force close** → **обрыв запроса** → клиент получит **500**.

### Ответ 14

- **Kafka:** `consumer.Pause(consumer.Assignment())` + ручной drain.
- **NATS:** `nc.Drain()`.
- **RabbitMQ:** `ch.Cancel()` + ручной drain.

### Ответ 15

**Shutdown timeout** — таймаут на graceful shutdown. Защита от зависания. Меньше `terminationGracePeriodSeconds`.

### Ответ 16

**Порядок освобождения важен**, потому что зависимые ресурсы используют базовые. Если закрыть БД до воркеров — воркеры упадут.

**Правильный порядок:** зависимые → базовые.

### Ответ 17

**`log.Fatal`** вызывает `os.Exit`. `os.Exit` **не выполняет** `defer`.

### Ответ 18

**`logger.Sync`** — сбросить буферы лога на диск. Без него последние логи могут потеряться.

### Ответ 19

**Тестирование graceful shutdown:**

1. **`Run(ctx)`** — вынести в функцию.
2. **`context.WithCancel`** — в тесте.
3. **Проверка активного запроса.**
4. **Проверка отмены обработчика.**

### Ответ 20

**Три уровня отмены:**

1. **`srv.Shutdown`** — прекращает приём, отменяет `r.Context()`.
2. **`r.Context()`** — обработчик должен **проверить** его.
3. **Внешние вызовы** — отменяются через `r.Context()`.

---

## 11.16 Куда идти дальше?

Мы разобрали graceful shutdown: сигналы, ожидание, HTTP, `r.Context()`, очереди, timeout, освобождение ресурсов, тестирование. Теперь мы умеем **корректно завершать** сервисы.

Но остаётся **важный вопрос**: как **тестировать** и **профилировать** конкурентный код? Как найти утечки? Как измерить производительность?

- **Как тестировать конкурентный код?** Race detector, stress-тесты, `pprof`. → **Глава 12: Тестирование и профилирование конкурентного кода.**
- **Какие анти-паттерны существуют?** Утечки горутин, deadlock, livelock, гонки. → **Глава 13: Анти-паттерны.**
- **Реальные сценарии:** HTTP-сервер, очереди, ETL, scraping. → **Глава 14: Реальные сценарии.**

---

## 11.17 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Graceful shutdown** | Корректное завершение | Четыре фазы |
| **`os.Exit`** | Не graceful | `defer` не выполняется |
| **`SIGTERM`** | Termination | Kubernetes |
| **`SIGINT`** | Ctrl+C | Пользователь |
| **`SIGKILL`** | Kill | Не перехватывается |
| **`signal.NotifyContext`** | Перехват | Go 1.16+ |
| **`WaitGroup`** | Ожидание | Без ошибок |
| **`errgroup`** | Ожидание | С ошибками |
| **`srv.Shutdown`** | HTTP graceful | Закрыть сокет + отменить ctx + ждать |
| **`r.Context()`** | Контекст запроса | Отменяется при shutdown |
| **Отмена не нужна** | Быстрый обработчик | < 100 мс |
| **Отмена нужна** | Долгий обработчик | > 1 сек, внешние вызовы |
| **Три уровня отмены** | Shutdown → ctx → вызовы | Критично |
| **`http.ErrServerClosed`** | Нормальная ошибка | При shutdown |
| **`srv.Close`** | HTTP force | Немедленно |
| **Дренаж Kafka** | `Pause` | + ручной |
| **Дренаж NATS** | `Drain` | Встроенный |
| **Дренаж RabbitMQ** | `Cancel` | + ручной |
| **Shutdown timeout** | 10-30 сек | Меньше K8s |
| **Порядок освобождения** | Зависимые → базовые | Критично |
| **`logger.Sync`** | Сброс логов | Обязательно |
| **`log.Fatal`** | Не выполняет `defer` | Осторожно |
| **Тестирование** | `Run(ctx)` | + `context.WithCancel` |

🛑 **Ключевая идея:** Graceful shutdown — четыре фазы: перехват сигнала → стоп приёма → ожидание → освобождение. `srv.Shutdown` **закрывает сокет**, **отменяет `r.Context()`**, **ждёт** активные обработчики. **`r.Context()` нужен**, если обработчик долгий или имеет внешние вызовы; **не нужен**, если обработчик быстрый (< 100 мс) и локальный. Без `r.Context()` shutdown может затянуться, timeout истечёт, force close оборвёт запросы. Три уровня отмены: `srv.Shutdown` → `r.Context()` → внешние вызовы. Shutdown timeout обязательно, меньше `terminationGracePeriodSeconds`. Порядок освобождения ресурсов: зависимые → базовые. `logger.Sync` для логов. `log.Fatal` не выполняет `defer`. Тестирование через `Run(ctx)` + `context.WithCancel`.