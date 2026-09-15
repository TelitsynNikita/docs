# Глава 3: Kafka в Go — библиотеки и базовые паттерны

**Что вы узнаете:**
- Какие библиотеки для работы с Kafka существуют в Go и чем они отличаются.
- Как установить и настроить `sarama` и `kafka-go`.
- Как написать producer на обеих библиотеках: sync, async, с батчингом.
- Как написать consumer на обеих библиотеках: с consumer group, с ручным коммитом.
- Как корректно завершать работу producer'ов и consumer'ов (graceful shutdown).
- Какие базовые паттерны существуют и когда их применять.
- Как согласовать внутренний таймаут приложения с `terminationGracePeriodSeconds` в Kubernetes.

**После прочтения вы сможете:**
- Осознанно выбирать между `sarama` и `kafka-go` под задачу.
- Писать producer'ы и consumer'ы на обеих библиотеках.
- Настраивать батчинг, retries, идемпотентность.
- Делать graceful shutdown без потери сообщений.
- Понимать, какие паттерны (sync, async, worker pool, fan-out) когда применять.

---

## 📑 Содержание главы

- [3.0 Пролог: почему Go и почему две библиотеки](#30-пролог-почему-go-и-почему-две-библиотеки)
- [3.1 Ландшафт библиотек: sarama, kafka-go, confluent-kafka-go](#31-ландшафт-библиотек-sarama-kafka-go-confluent-kafka-go)
- [3.2 Установка и настройка окружения](#32-установка-и-настройка-окружения)
- [3.3 Producer на kafka-go: sync и async](#33-producer-на-kafka-go-sync-и-async)
- [3.4 Producer на sarama: sync и async](#34-producer-на-sarama-sync-и-async)
- [3.5 Сравнение producer API: sarama vs kafka-go](#35-сравнение-producer-api-sarama-vs-kafka-go)
- [3.6 Consumer на kafka-go: consumer group и ручной коммит](#36-consumer-на-kafka-go-consumer-group-и-ручной-коммит)
- [3.7 Consumer на sarama: consumer group и ручной коммит](#37-consumer-на-sarama-consumer-group-и-ручной-коммит)
- [3.8 Сравнение consumer API: sarama vs kafka-go](#38-сравнение-consumer-api-sarama-vs-kafka-go)
- [3.9 Graceful shutdown: как не потерять сообщения](#39-graceful-shutdown-как-не-потерять-сообщения)
- [3.10 Базовые паттерны: обзор](#310-базовые-паттерны-обзор)
- [3.11 Структура Go-проекта с Kafka](#311-структура-go-проекта-с-kafka)
- [3.12 Выводы и типичные ошибки](#312-выводы-и-типичные-ошибки)
- [3.13 Для быстрого повторения](#313-для-быстрого-повторения)
- [3.14 Вопросы для самопроверки](#314-вопросы-для-самопроверки)
- [3.15 Ответы](#315-ответы)
- [3.16 Куда идти дальше?](#316-куда-идти-дальше)
- [3.17 Чек-лист](#317-чек-лист)

---

## 3.0 Пролог: почему Go и почему две библиотеки

В Главе 2 мы написали первый producer и consumer на `kafka-go`. Это был минимальный пример — просто чтобы увидеть сущности Kafka в работе.

Теперь пора **углубиться**. Мы будем изучать **две библиотеки на равных**:

- **`kafka-go`** (Segment) — простой, идиоматичный, context-aware.
- **`sarama`** (IBM) — зрелый, мощный, с низкоуровневым контролем.

**Почему две, а не одну?**

Потому что в реальных проектах выбор зависит от задачи. `kafka-go` хорош для большинства случаев: простой API, минимум boilerplate, нативная поддержка контекста. `sarama` нужен, когда требуется тонкий контроль: кастомные партиционеры, точная настройка батчинга, работа с транзакциями.

**Что мы будем делать:**

Для каждой темы — параллельные примеры на обеих библиотеках. Ты увидишь, как одна и та же задача решается разными API, и сможешь осознанно выбирать.

**Важно:** мы **не** будем использовать `confluent-kafka-go` в качестве основной библиотеки. Она требует `cgo` (C-зависимости), что усложняет сборку и деплой. Мы упомянем её в сравнении, но примеры будут на `kafka-go` и `sarama`.

---

## 3.1 Ландшафт библиотек: sarama, kafka-go, confluent-kafka-go

Прежде чем писать код, разберёмся, какие библиотеки существуют и чем они отличаются.

### Сравнительная таблица

| Критерий | kafka-go | sarama | confluent-kafka-go |
|:---|:---|:---|:---|
| **Разработчик** | Segment (теперь Confluent) | IBM | Confluent |
| **API** | Простой, идиоматичный | Мощный, низкоуровневый | Официальный, как Java-клиент |
| **Context-aware** | ✅ Да | ❌ Нет (нужны обёртки) | ✅ Да |
| **cgo** | ❌ Нет | ❌ Нет | ✅ Да (C-зависимости) |
| **Транзакции** | ✅ Да (с 0.4.x) | ✅ Да (с 1.38) | ✅ Да |
| **Consumer group** | ✅ Да | ✅ Да | ✅ Да |
| **Кастомный партиционер** | ✅ Да | ✅ Да | ✅ Да |
| **Экосистема** | Меньше | Больше | Официальная |
| **Зрелость** | Молодая | Зрелая | Очень зрелая |

### Когда что выбирать

**`kafka-go`:**
- Большинство проектов.
- Нужен простой API и context-aware.
- Не нужны экзотические функции.
- Важна простота сборки (без cgo).

**`sarama`:**
- Нужен тонкий контроль над партиционированием.
- Нужны специфические настройки батчинга.
- Проект уже использует `sarama`.
- Нужна максимальная производительность.

**`confluent-kafka-go`:**
- Нужна официальная поддержка Confluent.
- Нужны транзакции с exactly-once.
- Проект уже использует Confluent-стек.
- Готовы работать с cgo.

### Что мы будем использовать

Мы будем изучать **`kafka-go` и `sarama` параллельно**. Для каждой темы — два примера. Ты сможешь сравнить API и выбрать подходящий.

---

## 3.2 Установка и настройка окружения

### Запуск Kafka

Мы будем использовать тот же `docker-compose.yml`, что и в Главе 2:

```yaml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
```

Запусти:

```bash
docker-compose up -d
```

### Создание топика

Создадим топик `orders` с 3 разделами:

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1
```

Проверим:

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic orders
```

Вывод:

```
Topic: orders   PartitionCount: 3   ReplicationFactor: 1
    Partition: 0   Leader: 1   Replicas: 1   Isr: 1
    Partition: 1   Leader: 1   Replicas: 1   Isr: 1
    Partition: 2   Leader: 1   Replicas: 1   Isr: 1
```

Теперь у нас **3 раздела**. Это позволит увидеть, как producer распределяет сообщения, и как consumer group распределяет разделы между потребителями.

### Go-проект

```bash
mkdir kafka-go-deep
cd kafka-go-deep
go mod init kafka-go-deep

# Устанавливаем обе библиотеки
go get github.com/segmentio/kafka-go
go get github.com/IBM/sarama
```

---

## 3.3 Producer на kafka-go: sync и async

### Sync producer

Самый простой способ — синхронная отправка. Отправили сообщение — дождались подтверждения — пошли дальше.

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/segmentio/kafka-go"
)

func main() {
	w := &kafka.Writer{
		Addr:     kafka.TCP("localhost:9092"),
		Topic:    "orders",
		Balancer: &kafka.LeastBytes{},
	}
	defer w.Close()

	for i := 1; i <= 10; i++ {
		msg := kafka.Message{
			Key:   []byte(fmt.Sprintf("user-%d", i)),
			Value: []byte(fmt.Sprintf(`{"order_id": %d}`, i)),
		}

		err := w.WriteMessages(context.Background(), msg)
		if err != nil {
			log.Printf("Ошибка: %v", err)
			continue
		}

		fmt.Printf("Отправлено: key=%s\n", msg.Key)
	}
}
```

**Что здесь происходит:**

1. `kafka.Writer` — объект для записи.
2. `Addr` — адрес брокера.
3. `Topic` — топик, в который пишем.
4. `Balancer` — стратегия выбора раздела. `LeastBytes` выбирает раздел с наименьшим объёмом данных.
5. `WriteMessages` — синхронная отправка. Блокируется, пока не получит подтверждение.

**Как узнать, в какой раздел попало сообщение?**

`kafka.Writer` не возвращает partition и offset напрямую. Чтобы узнать, нужно использовать `Completion` callback или перейти на `sarama`. Это одно из ограничений `kafka-go` — он скрывает детали.

### Async producer

Асинхронная отправка: сообщение кладётся в буфер, отправляется в фоне.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

func main() {
	w := &kafka.Writer{
		Addr:         kafka.TCP("localhost:9092"),
		Topic:        "orders",
		Balancer:     &kafka.LeastBytes{},
		BatchSize:    100,                    // батч до 100 сообщений
		BatchTimeout: 10 * time.Millisecond,  // или каждые 10 мс
		Async:        true,                   // асинхронная отправка
	}
	defer w.Close()

	for i := 1; i <= 1000; i++ {
		msg := kafka.Message{
			Key:   []byte(fmt.Sprintf("user-%d", i)),
			Value: []byte(fmt.Sprintf(`{"order_id": %d}`, i)),
		}

		err := w.WriteMessages(context.Background(), msg)
		if err != nil {
			log.Printf("Ошибка: %v", err)
		}
	}

	// Ждём завершения всех отправок
	w.Close()
	fmt.Println("Все сообщения отправлены")
}
```

**Что изменилось:**

1. `Async: true` — отправка асинхронная. `WriteMessages` не блокируется.
2. `BatchSize` — сколько сообщений накапливать перед отправкой.
3. `BatchTimeout` — максимальное время ожидания перед отправкой.

**Важно:** при `Async: true` ошибки отправки **не возвращаются** из `WriteMessages`. Нужно использовать `Completion` callback:

```go
w := &kafka.Writer{
    // ...
    Async: true,
    Completion: func(messages []kafka.Message, err error) {
        if err != nil {
            log.Printf("Ошибка отправки %d сообщений: %v", len(messages), err)
        }
    },
}
```

### Сравнение sync vs async

| Аспект | Sync | Async |
|:---|:---|:---|
| **Блокировка** | Да, до подтверждения | Нет |
| **Пропускная способность** | Ниже | Выше (батчинг) |
| **Обработка ошибок** | Через `WriteMessages` | Через `Completion` |
| **Гарантия порядка** | Да (в рамках партиции) | Да (при `Async` + батчинг) |
| **Когда использовать** | Критичные данные | Высокий throughput |

### 💡 Практика: что делать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй `Async: true` для высокого throughput.** Батчинг снижает нагрузку на сеть.
2. **Устанавливай `BatchTimeout`** — иначе сообщения могут задерживаться.
3. **Используй `Completion` callback** для обработки ошибок при `Async`.

**👍 СТОИТ:**

4. **Используй `Sync` для критичных данных**, где важна гарантия доставки каждого сообщения.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `defer w.Close()`** — иначе буфер не сбросится.
6. **Не игнорируй ошибки в `Completion`** — сообщения могут потеряться.

---

## 3.4 Producer на sarama: sync и async

### Sync producer

```go
package main

import (
	"fmt"
	"log"

	"github.com/IBM/sarama"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true // нужен для sync
	config.Producer.RequiredAcks = sarama.WaitForAll

	producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
	if err != nil {
		log.Fatalf("Ошибка создания producer: %v", err)
	}
	defer producer.Close()

	for i := 1; i <= 10; i++ {
		msg := &sarama.ProducerMessage{
			Topic: "orders",
			Key:   sarama.StringEncoder(fmt.Sprintf("user-%d", i)),
			Value: sarama.StringEncoder(fmt.Sprintf(`{"order_id": %d}`, i)),
		}

		partition, offset, err := producer.SendMessage(msg)
		if err != nil {
			log.Printf("Ошибка: %v", err)
			continue
		}

		fmt.Printf("Отправлено: partition=%d offset=%d\n", partition, offset)
	}
}
```

**Что здесь происходит:**

1. `sarama.NewConfig()` — конфигурация.
2. `Producer.Return.Successes = true` — обязательно для sync producer.
3. `Producer.RequiredAcks = sarama.WaitForAll` — ждать подтверждения от всех ISR (аналог `acks=all`).
4. `SendMessage` возвращает **partition** и **offset** — то, чего не хватает в `kafka-go`.

**Ключевое отличие от `kafka-go`:** `sarama` возвращает partition и offset при синхронной отправке. Это важно для отладки и для сценариев, где нужно знать точное местоположение сообщения.

### Async producer

```go
package main

import (
	"fmt"
	"log"
	"sync"
	"time"

	"github.com/IBM/sarama"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	config.Producer.Return.Errors = true
	config.Producer.RequiredAcks = sarama.WaitForAll
	config.Producer.Flush.Frequency = 10 * time.Millisecond
	config.Producer.Flush.Messages = 100

	producer, err := sarama.NewAsyncProducer([]string{"localhost:9092"}, config)
	if err != nil {
		log.Fatalf("Ошибка создания producer: %v", err)
	}

	var wg sync.WaitGroup
	wg.Add(2)

	// Обработка успешных отправок
	go func() {
		defer wg.Done()
		for msg := range producer.Successes() {
			fmt.Printf("Отправлено: partition=%d offset=%d\n", msg.Partition, msg.Offset)
		}
	}()

	// Обработка ошибок
	go func() {
		defer wg.Done()
		for err := range producer.Errors() {
			log.Printf("Ошибка: %v", err)
		}
	}()

	for i := 1; i <= 1000; i++ {
		msg := &sarama.ProducerMessage{
			Topic: "orders",
			Key:   sarama.StringEncoder(fmt.Sprintf("user-%d", i)),
			Value: sarama.StringEncoder(fmt.Sprintf(`{"order_id": %d}`, i)),
		}
		producer.Input() <- msg
	}

	producer.AsyncClose() // закрывает Input(), ждёт обработки
	wg.Wait()
}
```

**Что здесь происходит:**

1. `NewAsyncProducer` — создаёт асинхронный producer.
2. `Successes()` — канал успешных отправок. Возвращает partition и offset.
3. `Errors()` — канал ошибок.
4. `Input()` — канал для отправки сообщений.
5. `AsyncClose()` — закрывает Input() и ждёт, пока все сообщения обработаются.

**Важно:** `sarama` использует каналы для успехов и ошибок. Это мощно, но требует аккуратной обработки — иначе можно заблокироваться.

### Сравнение sync vs async в sarama

| Аспект | Sync | Async |
|:---|:---|:---|
| **API** | `SendMessage` | `Input() <- msg` |
| **Возврат partition/offset** | Да | Через `Successes()` |
| **Обработка ошибок** | Из `SendMessage` | Через `Errors()` |
| **Пропускная способность** | Ниже | Выше |
| **Сложность** | Проще | Сложнее (каналы) |

### 💡 Практика: что делать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Устанавливай `Producer.Return.Successes = true`** для sync producer. Без этого `SendMessage` заблокируется.
2. **Используй `AsyncClose()`** для async producer. `Close()` может потерять сообщения.
3. **Обрабатывай каналы `Successes()` и `Errors()`** — иначе producer заблокируется.

**👍 СТОИТ:**

4. **Настраивай `Flush.Frequency` и `Flush.Messages`** для контроля батчинга.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `Errors()`** — сообщения могут потеряться.
6. **Не используй `Close()` вместо `AsyncClose()`** для async producer.

---

## 3.5 Сравнение producer API: sarama vs kafka-go

| Аспект | kafka-go | sarama |
|:---|:---|:---|
| **Создание** | `&kafka.Writer{...}` | `NewSyncProducer` / `NewAsyncProducer` |
| **Sync отправка** | `WriteMessages` | `SendMessage` |
| **Async отправка** | `Async: true` | `Input() <- msg` |
| **Возврат partition/offset** | Нет | Да |
| **Обработка ошибок** | Из `WriteMessages` | Из `SendMessage` / `Errors()` |
| **Батчинг** | `BatchSize`, `BatchTimeout` | `Flush.Messages`, `Flush.Frequency` |
| **Партиционер** | `Balancer` | `Partitioner` |
| **Context** | ✅ Да | ❌ Нет |
| **Закрытие** | `Close()` | `Close()` / `AsyncClose()` |

**Ключевые различия:**

1. **`kafka-go` проще.** Меньше boilerplate, context-aware, один объект `Writer` для sync и async.
2. **`sarama` мощнее.** Возвращает partition/offset, поддерживает кастомные партиционеры, но требует больше кода.
3. **`kafka-go` скрывает детали.** Ты не знаешь, в какой раздел попало сообщение, без дополнительных усилий.
4. **`sarama` даёт контроль.** Ты видишь partition и offset, можешь настроить партиционер.

**Что выбирать:**

- Если нужно **просто и быстро** — `kafka-go`.
- Если нужен **контроль** — `sarama`.
- Если нужны **транзакции** — обе поддерживают, но `sarama` более зрелая.

---

## 3.6 Consumer на kafka-go: consumer group и ручной коммит

### Consumer group с автоматическим коммитом

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/segmentio/kafka-go"
)

func main() {
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers:  []string{"localhost:9092"},
		GroupID:  "order-processor",
		Topic:    "orders",
		MinBytes: 1,
		MaxBytes: 10e6,
	})
	defer r.Close()

	for {
		msg, err := r.ReadMessage(context.Background())
		if err != nil {
			log.Fatalf("Ошибка чтения: %v", err)
		}

		fmt.Printf("Получено: partition=%d offset=%d key=%s value=%s\n",
			msg.Partition, msg.Offset, msg.Key, msg.Value)

		// Коммит происходит автоматически при следующем ReadMessage
	}
}
```

**Что здесь происходит:**

1. `kafka.NewReader` с `GroupID` — создаёт consumer group.
2. `ReadMessage` — читает следующее сообщение.
3. Offset коммитится **автоматически** при следующем `ReadMessage`.

**Проблема:** если обработка сообщения упадёт после `ReadMessage`, но до коммита, сообщение может быть потеряно. Автокоммит коммитит **предыдущее** сообщение при чтении следующего.

### Ручной коммит

Для контроля над коммитом используем `FetchMessage` + `CommitMessages`:

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/segmentio/kafka-go"
)

func main() {
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers:  []string{"localhost:9092"},
		GroupID:  "order-processor",
		Topic:    "orders",
	})
	defer r.Close()

	for {
		// Читаем сообщение БЕЗ коммита
		msg, err := r.FetchMessage(context.Background())
		if err != nil {
			log.Fatalf("Ошибка чтения: %v", err)
		}

		fmt.Printf("Получено: partition=%d offset=%d\n", msg.Partition, msg.Offset)

		// Обрабатываем сообщение
		if err := processMessage(msg); err != nil {
			log.Printf("Ошибка обработки: %v", err)
			continue // не коммитим — сообщение будет прочитано снова
		}

		// Коммитим ТОЛЬКО после успешной обработки
		if err := r.CommitMessages(context.Background(), msg); err != nil {
			log.Printf("Ошибка коммита: %v", err)
		}
	}
}

func processMessage(msg kafka.Message) error {
	// здесь логика обработки
	return nil
}
```

**Что изменилось:**

1. `FetchMessage` — читает сообщение, но **не коммитит**.
2. `processMessage` — обрабатываем.
3. `CommitMessages` — коммитим **только после успешной обработки**.

**Зачем:** если обработка упадёт, сообщение не будет закоммичено, и при перезапуске consumer прочитает его снова. Это **at-least-once** семантика.

### 💡 Практика: что делать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй ручной коммит для критичных данных.** Автокоммит может потерять сообщения.
2. **Коммить после успешной обработки**, а не до.

**👍 СТОИТ:**

3. **Обрабатывай ошибки коммита** — если коммит не прошёл, сообщение будет прочитано снова.

**❌ НЕ ДЕЛАЙ:**

4. **Не коммить до обработки** — потеряешь сообщения при падении.
5. **Не игнорируй `FetchMessage` ошибки** — consumer может зависнуть.

---

## 3.7 Consumer на sarama: consumer group и ручной коммит

### Consumer group с автоматическим коммитом

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/IBM/sarama"
)

type consumerHandler struct{}

func (h *consumerHandler) Setup(sarama.ConsumerGroupSession) error   { return nil }
func (h *consumerHandler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h *consumerHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		fmt.Printf("Получено: partition=%d offset=%d key=%s value=%s\n",
			msg.Partition, msg.Offset, msg.Key, msg.Value)

		// Помечаем сообщение как обработанное
		session.MarkMessage(msg, "")
	}
	return nil
}

func main() {
	config := sarama.NewConfig()
	config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
	config.Consumer.Offsets.Initial = sarama.OffsetOldest

	group, err := sarama.NewConsumerGroup([]string{"localhost:9092"}, "order-processor", config)
	if err != nil {
		log.Fatalf("Ошибка создания consumer group: %v", err)
	}
	defer group.Close()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Graceful shutdown
	go func() {
		sigterm := make(chan os.Signal, 1)
		signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)
		<-sigterm
		cancel()
	}()

	handler := &consumerHandler{}
	for {
		if err := group.Consume(ctx, []string{"orders"}, handler); err != nil {
			log.Fatalf("Ошибка Consume: %v", err)
		}
		if ctx.Err() != nil {
			return
		}
	}
}
```

**Что здесь происходит:**

1. `consumerHandler` реализует интерфейс `sarama.ConsumerGroupHandler`.
2. `Setup` — вызывается при ребалансировке (начало).
3. `Cleanup` — вызывается при ребалансировке (конец).
4. `ConsumeClaim` — обрабатывает сообщения из назначенных разделов.
5. `session.MarkMessage` — помечает сообщение как обработанное. Коммит происходит периодически.

**Ключевое отличие от `kafka-go`:** `sarama` использует **интерфейс** для обработки сообщений. Ты реализуешь `ConsumeClaim`, и `sarama` вызывает его для каждого назначенного раздела.

### Ручной коммит

```go
func (h *consumerHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		// Обрабатываем
		if err := processMessage(msg); err != nil {
			log.Printf("Ошибка обработки: %v", err)
			continue // не коммитим
		}

		// Коммитим только после успешной обработки
		session.MarkMessage(msg, "")
		session.Commit() // принудительный коммит
	}
	return nil
}
```

**Что изменилось:**

1. `processMessage` — обработка.
2. `session.MarkMessage` — пометить сообщение.
3. `session.Commit()` — принудительно закоммитить.

**Важно:** `session.Commit()` вызывается **синхронно** и может блокировать. Для высокого throughput лучше коммитить периодически, а не после каждого сообщения.

### 💡 Практика: что делать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Реализуй `Setup` и `Cleanup`** — они нужны для корректной работы при ребалансировке.
2. **Используй `MarkMessage` + `Commit`** для ручного коммита.

**👍 СТОИТ:**

3. **Коммить периодически**, а не после каждого сообщения — для производительности.

**❌ НЕ ДЕЛАЙ:**

4. **Не игнорируй ошибки в `ConsumeClaim`** — они приведут к перезапуску.
5. **Не забывай про graceful shutdown** — иначе потеряешь сообщения.

---

## 3.8 Сравнение consumer API: sarama vs kafka-go

| Аспект | kafka-go | sarama |
|:---|:---|:---|
| **Создание** | `kafka.NewReader` | `NewConsumerGroup` |
| **Чтение** | `ReadMessage` / `FetchMessage` | `ConsumeClaim` |
| **Коммит** | `CommitMessages` | `MarkMessage` + `Commit` |
| **Ребалансировка** | Автоматически | Через `Setup`/`Cleanup` |
| **Обработка** | В цикле | Через интерфейс |
| **Context** | ✅ Да | ❌ Нет |
| **Несколько топиков** | ❌ Нет (один `Reader` — один топик) | ✅ Да |

**Ключевые различия:**

1. **`kafka-go` проще.** Ты читаешь в цикле, коммитишь вручную.
2. **`sarama` структурированнее.** Ты реализуешь интерфейс, `sarama` вызывает методы.
3. **`kafka-go` не поддерживает несколько топиков** в одном `Reader`.
4. **`sarama` поддерживает несколько топиков** в одной группе.

**Что выбирать:**

- **`kafka-go`** — для простых сценариев, одного топика, context-aware.
- **`sarama`** — для сложных сценариев, нескольких топиков, тонкого контроля.

---

## 3.9 Graceful shutdown: как не потерять сообщения

### Проблема

Ты запустил consumer. Он читает сообщения. Вдруг приходит `SIGTERM` — Kubernetes останавливает под. Consumer завершается. Но:
- Текущее сообщение не обработано.
- Offset не закоммичен.
- При перезапуске consumer прочитает его снова — **дубликат**.

Или хуже:
- Offset закоммичен.
- Сообщение не обработано.
- При перезапуске consumer **не прочитает** его — **потеря**.

### Как Kubernetes останавливает поды

Kubernetes при остановке пода:

1. Отправляет `SIGTERM` процессу.
2. Ждёт `terminationGracePeriodSeconds` (по умолчанию **30 секунд**).
3. Если процесс не завершился — отправляет `SIGKILL`.

**Важно:** `terminationGracePeriodSeconds` — это **внешний** таймаут. Он не гарантирует, что приложение использует это время правильно. Если приложение зависнет, кубер всё равно убьёт его через 30 секунд.

### Согласование внутреннего и внешнего таймаутов

**Рекомендация из практики:** внутренний таймаут приложения должен быть **меньше** `terminationGracePeriodSeconds` на 10-15 секунд буфера.

```
terminationGracePeriodSeconds = 30s (дефолт)
внутренний context.WithTimeout = 15-20s
буфер для kubelet = 10-15s
```

**Зачем буфер:** kubelet нужно время на cleanup после завершения процесса (закрытие сокетов, удаление сетевых интерфейсов). Если приложение «съест» все 30 секунд, kubelet может не успеть — и прилетит `SIGKILL` прямо во время очистки.

### Аргументы ЗА использование `context.WithTimeout`

**1. Страховка от зависания.** Если `CommitMessages` или внешний API завис, таймаут прервёт операцию.

**2. Предсказуемость.** Ты точно знаешь, сколько времени приложение потратит на shutdown.

**3. Согласованность с кубером.** Внутренний таймаут меньше внешнего — приложение завершится корректно, не дожидаясь `SIGKILL`.

### Аргументы ПРОТИВ использования `context.WithTimeout`

**1. Если куберовский таймаут большой**, а внутренний маленький — приложение завершится раньше, чем могло бы. Если за это время можно было дослать сообщения — они потеряны.

**2. Таймаут может «оборвать» обработку на середине.** Если сообщение обрабатывается 20 секунд, а таймаут — 15 секунд, обработка будет прервана, offset не закоммичен, сообщение перечитано. Это дубликат, но не потеря.

**3. Простота.** Без явного таймаута код проще — кубер сам разберётся.

### Рекомендация

Используй **оба** таймаута, но согласованно:

```go
// main.go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

// Graceful shutdown
go func() {
    sigterm := make(chan os.Signal, 1)
    signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)
    <-sigterm
    cancel()
}()
```

**Для Kubernetes:**

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: consumer
    # ...
```

### Graceful shutdown в kafka-go

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/segmentio/kafka-go"
)

func main() {
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers: []string{"localhost:9092"},
		GroupID: "order-processor",
		Topic:   "orders",
	})
	defer r.Close()

	ctx, cancel := context.WithCancel(context.Background())

	// Graceful shutdown с таймаутом
	go func() {
		sigterm := make(chan os.Signal, 1)
		signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)
		<-sigterm
		log.Println("Получен сигнал, завершаем...")

		// Даём 15 секунд на завершение
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 15*time.Second)
		defer shutdownCancel()

		// Ждём завершения или таймаута
		select {
		case <-shutdownCtx.Done():
			log.Println("Таймаут завершения")
		case <-ctx.Done():
		}
		cancel()
	}()

	for {
		select {
		case <-ctx.Done():
			log.Println("Завершение")
			return
		default:
			msg, err := r.FetchMessage(ctx)
			if err != nil {
				if ctx.Err() != nil {
					return
				}
				log.Printf("Ошибка чтения: %v", err)
				continue
			}

			if err := processMessage(msg); err != nil {
				log.Printf("Ошибка обработки: %v", err)
				continue
			}

			if err := r.CommitMessages(ctx, msg); err != nil {
				log.Printf("Ошибка коммита: %v", err)
			}
		}
	}
}

func processMessage(msg kafka.Message) error {
	// здесь логика обработки
	return nil
}
```

### Graceful shutdown в sarama

```go
func main() {
	config := sarama.NewConfig()
	config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
	config.Consumer.Offsets.Initial = sarama.OffsetOldest

	group, err := sarama.NewConsumerGroup([]string{"localhost:9092"}, "order-processor", config)
	if err != nil {
		log.Fatalf("Ошибка: %v", err)
	}
	defer group.Close()

	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		sigterm := make(chan os.Signal, 1)
		signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)
		<-sigterm
		log.Println("Получен сигнал, завершаем...")

		// Даём 15 секунд на завершение
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 15*time.Second)
		defer shutdownCancel()

		select {
		case <-shutdownCtx.Done():
			log.Println("Таймаут завершения")
		case <-ctx.Done():
		}
		cancel()
	}()

	handler := &consumerHandler{}
	for {
		if err := group.Consume(ctx, []string{"orders"}, handler); err != nil {
			log.Fatalf("Ошибка Consume: %v", err)
		}
		if ctx.Err() != nil {
			log.Println("Завершение")
			return
		}
	}
}
```

### 💡 Практика: что делать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Слушай `SIGTERM` и `SIGINT`.** Kubernetes отправляет `SIGTERM` при остановке пода.
2. **Используй `context.WithTimeout`** — но согласованно с `terminationGracePeriodSeconds`.
3. **Внутренний таймаут ≤ куберовский - 10-15 секунд буфер.**
4. **Коммить offset после обработки** — не до.

**👍 СТОИТ:**

5. **Добавляй логирование** при получении сигнала и при завершении — для отладки.

**❌ НЕ ДЕЛАЙ:**

6. **Не завершай процесс мгновенно** — потеряешь сообщения.
7. **Не коммить offset до обработки** — потеряешь данные при падении.
8. **Не ставь внутренний таймаут = куберовскому** — не оставишь времени на cleanup.

---

## 3.10 Базовые паттерны: обзор

Это **обзорная** подглава. Детально каждый паттерн будет разобран в следующих главах — там, где он решает конкретную задачу.

### Sync producer

**Что:** отправка сообщения с ожиданием подтверждения.

**Когда:** критичные данные, где важна гарантия доставки каждого сообщения.

**Где детально:** Глава 5 (Producer).

```go
// kafka-go
w.WriteMessages(ctx, msg) // блокируется до подтверждения
```

### Async producer

**Что:** отправка сообщения в буфер без ожидания.

**Когда:** высокий throughput, можно пожертвовать задержкой.

**Где детально:** Глава 5 (Producer).

```go
// kafka-go
w := &kafka.Writer{Async: true, BatchSize: 100}
```

### Consumer group

**Что:** группа consumer'ов, совместно читающих топик.

**Когда:** несколько consumer'ов читают один топик параллельно.

**Где детально:** Глава 6 (Consumer).

```go
// kafka-go
r := kafka.NewReader(kafka.ReaderConfig{GroupID: "order-processor", ...})
```

### Ручной коммит

**Что:** коммит offset после успешной обработки.

**Когда:** at-least-once семантика, критичные данные.

**Где детально:** Глава 6 (Consumer).

```go
// kafka-go
msg, _ := r.FetchMessage(ctx)
process(msg)
r.CommitMessages(ctx, msg)
```

### Worker pool

**Что:** несколько горутин обрабатывают сообщения параллельно.

**Когда:** нужно распараллелить обработку внутри одного раздела.

**Где детально:** Глава 6 (Consumer).

**Важно:** worker pool **ломает порядок**. Если порядок важен — не используй.

```go
// Псевдокод
for msg := range messages {
    go process(msg) // fan-out
}
```

### Fan-out

**Что:** одно сообщение обрабатывается несколькими сервисами.

**Когда:** нужно, чтобы разные сервисы реагировали на одно событие.

**Где детально:** Глава 6 (Consumer).

```go
// Сервис 1: GroupID: "email-service"
// Сервис 2: GroupID: "push-service"
// Сервис 3: GroupID: "analytics-service"
// Все читают один топик независимо
```

### Fan-in

**Что:** несколько producer'ов пишут в один топик.

**Когда:** агрегация данных из разных источников.

**Где детально:** Глава 5 (Producer).

```go
// Несколько producer'ов пишут в один топик
```

### Pipeline

**Что:** цепочка обработки: filter → transform → sink.

**Когда:** сложная обработка потока.

**Где детально:** Глава 10 (Streams) или Глава 15 (Паттерны).

### Future/Promise

**Что:** асинхронная отправка с ожиданием результата.

**Когда:** нужно отправить сообщение и получить результат позже.

**Где детально:** Глава 5 (Producer).

**Важно:** в Go нет встроенных future/promise — это делается через каналы.

### Сводная таблица

| Паттерн | Когда использовать | Когда НЕ использовать | Где детально |
|:---|:---|:---|:---|
| **Sync producer** | Критичные данные | Высокий throughput | Глава 5 |
| **Async producer** | Высокий throughput | Критичные данные | Глава 5 |
| **Consumer group** | Параллельное чтение | Один consumer | Глава 6 |
| **Ручной коммит** | At-least-once | At-most-once | Глава 6 |
| **Worker pool** | Параллелизм внутри раздела | Важен порядок | Глава 6 |
| **Fan-out** | Несколько сервисов | Один сервис | Глава 6 |
| **Fan-in** | Агрегация | Строгий порядок | Глава 5 |
| **Pipeline** | Сложная обработка | Простая | Глава 10 |
| **Future** | Асинхронная отправка | Синхронная | Глава 5 |

---

## 3.11 Структура Go-проекта с Kafka

### Рекомендуемая структура

```
project/
├── cmd/
│   ├── producer/
│   │   └── main.go
│   └── consumer/
│       └── main.go
├── internal/
│   ├── kafka/
│   │   ├── producer.go      # обёртка над kafka-go/sarama
│   │   ├── consumer.go      # обёртка над consumer group
│   │   └── config.go        # конфигурация
│   ├── handler/
│   │   └── order.go         # обработка сообщений
│   └── model/
│       └── order.go         # структуры данных
├── pkg/
│   └── ...
├── go.mod
├── go.sum
└── docker-compose.yml
```

### Обёртка над producer

```go
// internal/kafka/producer.go
package kafka

import (
	"context"
	"github.com/segmentio/kafka-go"
)

type Producer struct {
	writer *kafka.Writer
}

func NewProducer(brokers []string, topic string) *Producer {
	return &Producer{
		writer: &kafka.Writer{
			Addr:     kafka.TCP(brokers...),
			Topic:    topic,
			Balancer: &kafka.LeastBytes{},
		},
	}
}

func (p *Producer) Send(ctx context.Context, key, value []byte) error {
	return p.writer.WriteMessages(ctx, kafka.Message{
		Key:   key,
		Value: value,
	})
}

func (p *Producer) Close() error {
	return p.writer.Close()
}
```

### Обёртка над consumer

```go
// internal/kafka/consumer.go
package kafka

import (
	"context"
	"github.com/segmentio/kafka-go"
)

type MessageHandler func(ctx context.Context, msg kafka.Message) error

type Consumer struct {
	reader  *kafka.Reader
	handler MessageHandler
}

func NewConsumer(brokers []string, topic, groupID string, handler MessageHandler) *Consumer {
	return &Consumer{
		reader: kafka.NewReader(kafka.ReaderConfig{
			Brokers: brokers,
			Topic:   topic,
			GroupID: groupID,
		}),
		handler: handler,
	}
}

func (c *Consumer) Run(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
			msg, err := c.reader.FetchMessage(ctx)
			if err != nil {
				if ctx.Err() != nil {
					return nil
				}
				return err
			}

			if err := c.handler(ctx, msg); err != nil {
				continue // не коммитим
			}

			if err := c.reader.CommitMessages(ctx, msg); err != nil {
				return err
			}
		}
	}
}

func (c *Consumer) Close() error {
	return c.reader.Close()
}
```

### Преимущества такой структуры

1. **Изоляция.** Логика Kafka отделена от бизнес-логики.
2. **Тестируемость.** Можно подменить producer/consumer на mock.
3. **Переиспользование.** Обёртки можно использовать в разных сервисах.
4. **Гибкость.** Легко переключиться с `kafka-go` на `sarama`.

---

## 3.12 Выводы и типичные ошибки

**Что мы узнали?**

В Go есть две основные библиотеки для Kafka: `kafka-go` (простая, context-aware) и `sarama` (мощная, низкоуровневая). Producer может быть sync (надёжно, медленно) или async (быстро, сложнее). Consumer работает через consumer group, offset можно коммитить автоматически или вручную. Graceful shutdown обязателен для продакшена, а внутренний таймаут должен быть согласован с `terminationGracePeriodSeconds` в Kubernetes. Базовые паттерны (sync/async producer, consumer group, ручной коммит, worker pool, fan-out, fan-in) будут детально разобраны в главах 5 и 6.

**Типичные ошибки:**

- ❌ **Использовать автокоммит для критичных данных.** Автокоммит коммитит предыдущее сообщение при чтении следующего. При падении можно потерять сообщение.
- ❌ **Не делать graceful shutdown.** При `SIGTERM` процесс завершается мгновенно, текущие сообщения не обрабатываются, offset не коммитится.
- ❌ **Ставить внутренний таймаут = `terminationGracePeriodSeconds`.** Не оставишь времени kubelet на cleanup — прилетит `SIGKILL`.
- ❌ **Забывать `defer w.Close()`.** Буфер producer'а не сбрасывается, сообщения теряются.
- ❌ **Игнорировать каналы `Successes()` и `Errors()` в sarama async producer.** Producer заблокируется.
- ❌ **Использовать `Close()` вместо `AsyncClose()` в sarama async producer.** Сообщения могут потеряться.
- ❌ **Коммитить offset до обработки.** При падении обработки сообщение будет потеряно.
- ❌ **Не обрабатывать ошибки коммита.** Если коммит не прошёл, сообщение будет прочитано снова — дубликат.
- ❌ **Использовать worker pool, когда важен порядок.** Worker pool ломает порядок обработки.
- ❌ **Читать несколько топиков одним `kafka.Reader`.** В `kafka-go` один `Reader` — один топик.

---

## 3.13 Для быстрого повторения

- **Две библиотеки:** `kafka-go` (простая, context-aware) и `sarama` (мощная, низкоуровневая).
- **Sync producer:** `WriteMessages` (kafka-go) / `SendMessage` (sarama). Блокируется до подтверждения.
- **Async producer:** `Async: true` (kafka-go) / `Input() <- msg` (sarama). Батчинг для throughput.
- **Consumer group:** `GroupID` (kafka-go) / `NewConsumerGroup` (sarama). Один раздел — один consumer в группе.
- **Ручной коммит:** `FetchMessage` + `CommitMessages` (kafka-go) / `MarkMessage` + `Commit` (sarama).
- **Graceful shutdown:** `context.WithCancel` + `signal.Notify` + `context.WithTimeout`.
- **Kubernetes:** `terminationGracePeriodSeconds` по умолчанию 30 секунд. Внутренний таймаут ≤ 30 - 10-15 = 15-20 секунд.
- **Паттерны:** sync/async producer, consumer group, ручной коммит, worker pool, fan-out, fan-in — обзор в 3.10, детали в главах 5 и 6.
- **Структура проекта:** `cmd/`, `internal/kafka/`, `internal/handler/`, `internal/model/`.

---

## 3.14 Вопросы для самопроверки

1. Чем `kafka-go` отличается от `sarama`? Когда что выбирать?
2. Что такое sync producer и async producer? Когда что использовать?
3. Как в `kafka-go` настроить батчинг?
4. Как в `sarama` обрабатывать успехи и ошибки async producer?
5. Что такое consumer group? Сколько consumer'ов могут читать один раздел в одной группе?
6. Чем автокоммит отличается от ручного коммита? Когда что использовать?
7. Как сделать graceful shutdown в `kafka-go`? В `sarama`?
8. Что такое `terminationGracePeriodSeconds`? Почему внутренний таймаут должен быть меньше?
9. Что такое worker pool? Почему он ломает порядок?
10. Что такое fan-out и fan-in? Где они будут разобраны детально?

---

## 3.15 Ответы

**1.** `kafka-go` — простая, context-aware, минимум boilerplate. `sarama` — мощная, низкоуровневая, возвращает partition/offset, поддерживает кастомные партиционеры. `kafka-go` для большинства проектов, `sarama` — когда нужен контроль.

**2.** Sync producer блокируется до подтверждения, надёжен, но медленный. Async producer кладёт сообщение в буфер и продолжает, быстрый за счёт батчинга, но сложнее обработка ошибок.

**3.** В `kafka-go`: `BatchSize` (сколько сообщений) и `BatchTimeout` (максимальное время ожидания) в `kafka.Writer`.

**4.** В `sarama` async producer нужно читать каналы `Successes()` и `Errors()`. Если не читать — producer заблокируется.

**5.** Consumer group — группа потребителей, совместно читающих топик. Один раздел читается только одним consumer'ом в группе. Больше consumer'ов, чем разделов — лишние простаивают.

**6.** Автокоммит коммитит предыдущее сообщение при чтении следующего. Ручной коммит — после успешной обработки. Автокоммит проще, но может потерять сообщения. Ручной коммит надёжнее, но сложнее.

**7.** В `kafka-go`: `context.WithCancel` + `signal.Notify` + `context.WithTimeout`, цикл проверяет `ctx.Done()`. В `sarama`: то же самое, но `group.Consume(ctx, ...)` возвращает при отмене контекста.

**8.** `terminationGracePeriodSeconds` — время, которое Kubernetes даёт поду на завершение после `SIGTERM` (по умолчанию 30 секунд). Внутренний таймаут должен быть меньше, потому что kubelet нужно время на cleanup после завершения процесса. Если приложение «съест» все 30 секунд, прилетит `SIGKILL`.

**9.** Worker pool — несколько горутин обрабатывают сообщения параллельно. Ломает порядок, потому что сообщения обрабатываются не по очереди.

**10.** Fan-out — одно сообщение обрабатывается несколькими сервисами (разные consumer groups). Fan-in — несколько producer'ов пишут в один топик. Детально: fan-out в Главе 6 (Consumer), fan-in в Главе 5 (Producer).

---

## 3.16 Куда идти дальше?

Мы разобрались с **библиотеками и базовыми паттернами**. Теперь мы готовы углубляться в каждую тему.

В **Главе 4** мы разберём **топики, партиции и репликацию** детально:

- Как устроены сегменты лога.
- Как работает retention и compaction.
- Как реплицируются данные между брокерами.
- Что такое HW и LEO.
- Как происходят выборы лидера.
- Как добавить брокер и перераспределить разделы.

**Глава 4: Топики, партиции и репликация.**

---

## 3.17 Чек-лист

| Тема | kafka-go | sarama |
|:---|:---|:---|
| **Sync producer** | `WriteMessages` | `SendMessage` |
| **Async producer** | `Async: true` | `Input() <- msg` |
| **Батчинг** | `BatchSize`, `BatchTimeout` | `Flush.Messages`, `Flush.Frequency` |
| **Partition/offset** | ❌ Не возвращает | ✅ Возвращает |
| **Consumer group** | `GroupID` | `NewConsumerGroup` |
| **Ручной коммит** | `FetchMessage` + `CommitMessages` | `MarkMessage` + `Commit` |
| **Graceful shutdown** | `context.WithCancel` + `context.WithTimeout` | `context.WithCancel` + `context.WithTimeout` |
| **Несколько топиков** | ❌ Нет | ✅ Да |
| **Context-aware** | ✅ Да | ❌ Нет |

🔐 **Ключевая идея:** выбирай `kafka-go` для простоты и context-aware, `sarama` для контроля и partition/offset. Всегда делай graceful shutdown. Внутренний таймаут ≤ `terminationGracePeriodSeconds` - 10-15 секунд. Коммить offset после обработки. Используй паттерны осознанно — детали в главах 5 и 6.