## Глава 20: Video streaming: CDN, транскодинг, адаптация

### Оглавление главы

- **20.1** Уточнение требований: что такое video streaming и кому он нужен
- **20.2** Оценка нагрузки: загрузки, просмотры, хранение
- **20.3** API: загрузка, воспроизведение, метаданные
- **20.4** Проектирование данных: видео, метаданные, сегменты
- **20.5** Транскодинг: кодеки, битрейт-ладдер, per-shot encoding
- **20.6** Адаптивный битрейт: HLS, DASH, манифесты
- **20.7** CDN: edge, origin, cache invalidation
- **20.8** Отказоустойчивость, мониторинг и безопасность

---

### Введение

В предыдущей главе мы спроектировали ride-sharing сервис — от требований до продакшена. Ты узнал, что такое геопространственные индексы, поиск ближайших, matching, ETA, surge pricing. Ты увидел, как те же строительные блоки применяются к геопространственной задаче. Но ride-sharing — это про координацию в реальном мире. А что если задача — это доставка контента?

Представь, что тебя просят спроектировать video streaming платформу — как YouTube. Пользователи загружают видео. Пользователи смотрят видео. Видео нужно транскодировать в разные форматы, адаптировать под разные устройства и скорости интернета, кешировать в CDN, доставлять миллионам пользователей одновременно. И всё это — с минимальной задержкой и максимальным качеством.

На первый взгляд, всё просто: пользователь загружает видео, ты его сохраняешь, другие пользователи его смотрят. Но что если видео весит гигабайты? Что если пользователей миллиард? Что если у пользователя медленный интернет? Что если видео смотрят одновременно миллион человек? Наивный подход ломается на масштабе.

В этой главе ты узнаешь, как строить video streaming платформу. Ты разберёшь транскодинг, адаптивный битрейт (HLS, DASH), CDN. Ты узнаешь, как хранить и доставлять петабайты видео. Ты узнаешь, что такое per-shot encoding и почему он экономит bandwidth. И ты поймёшь, что video streaming — это задача о хранении, обработке и доставке контента, которая требует понимания кодеков, CDN и масштабирования.

---

### 20.1 Уточнение требований: что такое video streaming и кому он нужен

Прежде чем рисовать диаграмму, нужно понять, что мы строим. И первый шаг — уточнить требования.

**Функциональные требования.**
- Пользователь может загружать видео (upload).
- Пользователь может смотреть видео (playback).
- Пользователь может искать видео по названию, описанию, тегам (опционально).
- Пользователь может ставить лайки, комментарии, подписываться на каналы (опционально).
- Система генерирует рекомендации (опционально).
- Система обрабатывает видео: транскодинг, thumbnails, модерация.
- Система поддерживает разные разрешения (360p, 720p, 1080p, 4K).
- Система поддерживает разные устройства (Web, iOS, Android, Smart TV).

**Нефункциональные требования.**
- Высокая доступность: 99.99% для просмотра.
- Низкая задержка: video start time (VST) меньше 2 секунд.
- Масштабируемость: система должна выдерживать миллиарды просмотров.
- Долговечность: видео не должны теряться.
- Адаптивность: качество должно адаптироваться под скорость интернета.
- Экономичность: bandwidth и storage должны быть оптимизированы.
- Безопасность: защита от пиратства, модерация контента.

**Вопросы, которые нужно задать.**
- Сколько видео загружается в день? 500 часов в минуту? 1000 часов в минуту?
- Сколько просмотров в день? 1 миллиард? 10 миллиардов?
- Какие разрешения поддерживать? 360p, 720p, 1080p, 4K, 8K?
- Какие кодеки использовать? H.264, H.265, VP9, AV1?
- Нужна ли live streaming? Или только VOD (video on demand)?
- Нужна ли монетизация (реклама)? Это меняет архитектуру.
- Нужна ли модерация контента? Автоматическая или ручная?
- Нужна ли поддержка DRM (Digital Rights Management)?

Ответы на эти вопросы определяют архитектуру. Например, если загружается 500 часов видео в минуту, это петабайты в день. Нужно объектное хранилище (S3) и распределённый транскодинг. Если просмотров миллиард в день, нужен CDN с кешированием на edge. Если нужна поддержка 4K, нужны мощные транскодеры и эффективные кодеки (HEVC, AV1).

Клеппман в «Высоконагруженных приложениях» объясняет, что требования определяют архитектуру. Без требований ты проектируешь в вакууме. И выбор зависит от того, какие требования у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, что первый шаг на любом интервью — уточнить требования. Без этого ты не можешь оценить нагрузку, выбрать компоненты, обосновать решения.

**Схема 20.1 — Общая картина системы (высокоуровневая)**

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Content      │         │ Viewer       │         │ Admin        │
│ Creator      │         │ (Web, iOS,   │         │ (moderation) │
│ (upload)     │         │  Android)    │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       └────────────────────────┼────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API Gateway / L7 LB                              │
│         маршрутизация, TLS termination, rate limiting               │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│   Upload Service        │   │   Watch Service         │
│   (Go)                  │   │   (Go)                  │
│   - загрузка видео      │   │   - воспроизведение     │
│   - валидация           │   │   - метаданные          │
│   - метаданные          │   │   - рекомендации        │
└───────────┬─────────────┘   └───────────┬─────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  S3 (raw)     │  │  PostgreSQL   │  │    Kafka      │
│  (raw videos) │  │  (metadata)   │  │  (transcode   │
│               │  │               │  │   jobs,       │
│               │  │               │  │   events)     │
└───────────────┘  └───────────────┘  └───────┬───────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  Transcoder   │
                                       │  Workers      │
                                       │  (chunk-      │
                                       │   parallel)   │
                                       └───────┬───────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  S3 (encoded) │
                                       │  (segments)   │
                                       └───────┬───────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  CDN          │
                                       │  (edge cache) │
                                       └───────────────┘
```

---

### 20.2 Оценка нагрузки: загрузки, просмотры, хранение

Теперь у нас есть требования. Давай оценим нагрузку.

**Загрузки.**
- 500 часов видео загружается каждую минуту. Это 500 * 60 = 30 000 часов в час = 720 000 часов в день.
- Средний битрейт raw видео: 16 Mbps (смесь 1080p, 4K, mobile).
- Ежедневный объём загрузок: 720 000 часов * 3600 секунд * 16 Mbps / 8 = 5.2 петабайт в день.

**Хранение.**
- Encoded варианты: примерно 5x от оригинала (8 rungs * 3 codecs с per-shot bit budgets). Ежедневный рост encoded: 25 петабайт в день. Годовой рост: 9 эксабайт в год (до lifecycle deletes) .
- Метаданные: 720 000 видео в день * 1 КБ (title, description, tags) = 720 МБ в день. Это ничтожно по сравнению с видео.

**Просмотры.**
- 1 миллиард часов просмотра в день.
- Средний битрейт при просмотре: 3 Mbps.
- Ежедневный egress: 1 миллиард часов * 3600 секунд * 3 Mbps / 8 = 1.35 эксабайт в день.
- Пиковое количество одновременных просмотров: ~250 миллионов (оценка) .

**CDN.**
- CDN с 95% byte-level offload от origin: ~67 петабайт в день от origin вместо 1.35 эксабайт в день .
- Это означает, что 95% трафика обслуживается CDN, и только 5% идёт в origin.

**Latency.**
- Video start time (VST): меньше 2 секунд (P95).
- Rebuffering ratio: меньше 1%.
- Cache hit ratio: 98.5–99% для VOD .

Вывод: система storage-heavy (петабайты в день) и bandwidth-heavy (эксабайты в день). Нужно объектное хранилище (S3) для raw и encoded видео, распределённый транскодинг (chunk-parallel), CDN с высоким cache hit ratio. Без CDN origin не справится с трафиком.

Клеппман в «Высоконагруженных приложениях» объясняет, что оценки определяют, где bottleneck. Без оценок ты не знаешь, нужен ли CDN, шардинг, распределённый транскодинг. И выбор зависит от того, какие числа у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как оценки влияют на архитектуру. Если у тебя bandwidth-heavy, CDN обязателен. Если storage-heavy, объектное хранилище.

**Схема 20.2 — Потоки данных: загрузка, транскодинг, просмотр**

```
UPLOAD PATH (загрузка видео):

Content Creator ──POST /upload──▶ Upload Service
                                      │
                                      ├──1. Валидация (формат, размер)
                                      ├──2. Сохранить raw video в S3 (multipart upload)
                                      ├──3. INSERT metadata в PostgreSQL (status: uploading)
                                      ├──4. PUBLISH в Kafka (topic: video_uploaded)
                                      └──5. Вернуть video_id

TRANSCODE PATH (транскодинг, асинхронно):

Kafka (video_uploaded) ──▶ Transcoder Orchestrator
                                │
                                ├──1. Demux raw video
                                ├──2. Segment на chunks (2-4 сек, IDR boundaries)
                                ├──3. Analyze каждый shot (per-shot bit budget)
                                ├──4. Encode cross-product (chunk × codec × rung) параллельно
                                ├──5. QC каждого chunk (VMAF)
                                ├──6. Package chunks as CMAF, HLS, DASH
                                ├──7. Загрузить encoded segments в S3
                                ├──8. Обновить metadata (status: ready)
                                └──9. PUBLISH в Kafka (topic: video_ready)

WATCH PATH (просмотр):

Viewer ──GET /watch/{video_id}──▶ Watch Service
                                      │
                                      ├──1. Получить metadata из PostgreSQL (или Redis cache)
                                      ├──2. Вернуть manifest URL (HLS .m3u8 или DASH .mpd)
                                      └──3. Viewer загружает manifest и сегменты с CDN
```

---

### 20.3 API: загрузка, воспроизведение, метаданные

API — это контракт между клиентом и сервером. Давай определим endpoints.

**Инициализация загрузки.**
```
POST /api/v1/videos/upload
Body: {
  "title": "My Video",
  "description": "Description",
  "tags": ["tag1", "tag2"],
  "file_size": 1073741824,  // bytes
  "content_type": "video/mp4"
}
Response: {
  "video_id": "vid_123",
  "upload_url": "https://s3.amazonaws.com/raw-bucket/vid_123?presigned",
  "expires_at": "2024-01-15T11:30:45Z"
}
```

**Завершение загрузки.**
```
POST /api/v1/videos/{video_id}/complete
Body: {
  "upload_id": "upload_456"
}
Response: {
  "video_id": "vid_123",
  "status": "processing"
}
```

**Получение метаданных.**
```
GET /api/v1/videos/{video_id}
Response: {
  "video_id": "vid_123",
  "title": "My Video",
  "description": "Description",
  "duration": 600,  // seconds
  "status": "ready",
  "thumbnails": ["https://cdn.example.com/thumb1.jpg", ...],
  "manifest_url": "https://cdn.example.com/vid_123/manifest.m3u8"
}
```

**Воспроизведение (HLS manifest).**
```
GET /cdn/{video_id}/manifest.m3u8
Response: (HLS manifest)
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/index.m3u8
```

**Воспроизведение (DASH manifest).**
```
GET /cdn/{video_id}/manifest.mpd
Response: (DASH MPD)
```

**Поиск видео (опционально).**
```
GET /api/v1/search?q=query&limit=20&cursor=abc
Response: {
  "videos": [...],
  "next_cursor": "def"
}
```

Важно: загрузка должна быть multipart (для больших файлов) и resumable (если загрузка прервалась, можно продолжить). Воспроизведение должно использовать adaptive bitrate (HLS или DASH), чтобы плеер мог выбирать качество в зависимости от скорости интернета.

Клеппман в «Высоконагруженных приложениях» объясняет, что API — это контракт. Он определяет, как клиент и сервер взаимодействуют. И выбор стиля API (REST, gRPC) зависит от того, кто клиенты и какие требования к производительности.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как API влияет на архитектуру. Если у тебя большие файлы, multipart upload. Если adaptive streaming, HLS/DASH.

**Схема 20.3 — API endpoints и их взаимодействие с компонентами**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway / LB                            │
│                                                                     │
│  POST /api/v1/videos/upload ──▶ Upload Service ──▶ S3 (presigned URL)
│                                        │                            │
│                                        └──▶ PostgreSQL (metadata)   │
│                                                                     │
│  POST /api/v1/videos/{id}/complete ▶ Upload Service ──▶ Kafka      │
│                                        │                            │
│                                        └──▶ PostgreSQL (status)     │
│                                                                     │
│  GET /api/v1/videos/{id} ──────▶ Watch Service ──▶ PostgreSQL       │
│                                        │              │             │
│                                        │              └──▶ Redis    │
│                                        └──▶ вернуть metadata        │
│                                                                     │
│  GET /cdn/{id}/manifest.m3u8 ───▶ CDN Edge ──▶ S3 (origin)         │
│                                        │                            │
│                                        └──▶ HLS manifest           │
│                                                                     │
│  GET /cdn/{id}/360p/segment.ts ▶ CDN Edge ──▶ S3 (origin)          │
│                                        │                            │
│                                        └──▶ video segment          │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 20.4 Проектирование данных: видео, метаданные, сегменты

Теперь давай спроектируем данные. Нам нужны таблицы для видео и метаданных.

**Схема.**
```sql
CREATE TABLE videos (
  id BIGSERIAL PRIMARY KEY,
  video_id VARCHAR(20) UNIQUE NOT NULL,
  creator_id BIGINT NOT NULL,
  title VARCHAR(200) NOT NULL,
  description TEXT,
  tags JSONB,
  duration INT,  -- seconds
  status VARCHAR(20) DEFAULT 'uploading',  -- 'uploading', 'processing', 'ready', 'failed'
  raw_s3_key VARCHAR(500),
  thumbnail_url VARCHAR(500),
  manifest_url VARCHAR(500),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMP,
  is_deleted BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_videos_creator ON videos(creator_id, created_at DESC);
CREATE INDEX idx_videos_status ON videos(status) WHERE status IN ('uploading', 'processing');
CREATE INDEX idx_videos_created ON videos(created_at DESC);

CREATE TABLE video_variants (
  id BIGSERIAL PRIMARY KEY,
  video_id VARCHAR(20) NOT NULL,
  resolution VARCHAR(10) NOT NULL,  -- '360p', '720p', '1080p', '4k'
  codec VARCHAR(10) NOT NULL,  -- 'h264', 'h265', 'vp9', 'av1'
  bitrate INT NOT NULL,  -- kbps
  s3_key VARCHAR(500) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE (video_id, resolution, codec)
);

CREATE INDEX idx_variants_video ON video_variants(video_id);
```

**Шардирование.**
- Videos: шардируем по creator_id (hash-based). Это позволяет быстро получать видео создателя. Но для просмотра (по video_id) нужен глобальный индекс или отдельная таблица videos_by_id (шардированная по video_id).
- Video_variants: шардируем по video_id (hash-based). Это позволяет быстро получать варианты видео.

**Хранение.**
- Raw видео: S3 (raw-bucket). Хранится, пока не завершится транскодинг, потом можно удалить (или архивировать).
- Encoded сегменты: S3 (encoded-bucket). Хранятся в течение retention period (например, 1 год).
- Thumbnails: S3 (thumbnail-bucket).
- Метаданные: PostgreSQL.

**CDN.**
- CDN кеширует encoded сегменты на edge. TTL зависит от популярности: популярные видео кешируются дольше, непопулярные — короче.
- Cache invalidation: если видео удалено или обновлено, нужно инвалидировать CDN кеш (через API CDN).

Клеппман в «Высоконагруженных приложениях» объясняет, что проектирование данных — это компромисс между нормализацией и денормализацией, между шардированием и cross-shard запросами. И выбор зависит от того, какие запросы у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как проектирование данных влияет на архитектуру. Если у тебя storage-heavy, объектное хранилище. Если bandwidth-heavy, CDN.

**Схема 20.4 — Схема данных и хранение**

```
PostgreSQL (videos) — шардирован по creator_id
┌─────────────────────────────────────────────────────────────┐
│ videos                                                      │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ video_id VARCHAR(20) UNIQUE                                 │
│ creator_id BIGINT (shard key)                               │
│ title VARCHAR(200)                                          │
│ description TEXT                                            │
│ tags JSONB                                                  │
│ duration INT                                                │
│ status VARCHAR(20)                                          │
│ raw_s3_key VARCHAR(500)                                     │
│ thumbnail_url VARCHAR(500)                                  │
│ manifest_url VARCHAR(500)                                   │
│ created_at TIMESTAMP                                        │
│ processed_at TIMESTAMP                                      │
│ is_deleted BOOLEAN                                          │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (video_variants) — шардирован по video_id
┌─────────────────────────────────────────────────────────────┐
│ video_variants                                              │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ video_id VARCHAR(20) (shard key)                            │
│ resolution VARCHAR(10)                                      │
│ codec VARCHAR(10)                                           │
│ bitrate INT                                                 │
│ s3_key VARCHAR(500)                                         │
│ created_at TIMESTAMP                                        │
│ UNIQUE (video_id, resolution, codec)                        │
└─────────────────────────────────────────────────────────────┘

S3 Buckets:
┌─────────────────────────────────────────────────────────────┐
│ raw-bucket/                                                 │
│   ├── vid_123/original.mp4                                  │
│   └── vid_456/original.mp4                                  │
│                                                             │
│ encoded-bucket/                                             │
│   ├── vid_123/                                              │
│   │   ├── 360p/index.m3u8                                   │
│   │   ├── 360p/segment_001.ts                               │
│   │   ├── 720p/index.m3u8                                   │
│   │   ├── 720p/segment_001.ts                               │
│   │   ├── 1080p/index.m3u8                                  │
│   │   └── 1080p/segment_001.ts                              │
│   └── vid_456/...                                           │
│                                                             │
│ thumbnail-bucket/                                           │
│   ├── vid_123/thumb_001.jpg                                 │
│   └── vid_456/thumb_001.jpg                                 │
└─────────────────────────────────────────────────────────────┘

CDN Edge Cache:
┌─────────────────────────────────────────────────────────────┐
│ /cdn/vid_123/360p/segment_001.ts (TTL: 7 days)              │
│ /cdn/vid_123/720p/segment_001.ts (TTL: 7 days)              │
│ /cdn/vid_123/manifest.m3u8 (TTL: 1 hour)                    │
└─────────────────────────────────────────────────────────────┘
```

---

### 20.5 Транскодинг: кодеки, битрейт-ладдер, per-shot encoding

Теперь давай разберём транскодинг — процесс преобразования видео в разные форматы.

**Кодеки.**
- **H.264 (AVC):** универсальный, поддерживается везде. Обязательный fallback.
- **H.265 (HEVC):** ~50% лучше сжатие, чем H.264. Поддерживается в Safari, iOS, Android (с royalties). Encode cost: 2-4x от H.264.
- **VP9:** ~50% лучше сжатие, чем H.264. Поддерживается в Chrome, Edge, Firefox, Android. Encode cost: 2-3x от H.264.
- **AV1:** ~30% лучше сжатие, чем VP9. Поддерживается в Chrome, Firefox, Edge, Safari 17+. Encode cost: 5-10x от H.264 (software), но ниже с hardware.

**Битрейт-ладдер.**
Типичный ладдер для VP9 (general-purpose UGC) :
- 4K (2160p): 12–20 Mbps
- 1440p: 6–10 Mbps
- 1080p: 3–6 Mbps
- 720p: 1.5–3 Mbps
- 480p: 0.5–1.5 Mbps
- 360p: 0.3–0.7 Mbps

**Per-shot encoding.**
Вместо одного битрейта для всего видео, per-shot encoding анализирует каждый shot (сцену) и выбирает оптимальный битрейт для него. Это экономит bandwidth без потери качества. Netflix Dynamic Optimizer (per-shot encoding) дал ~30% дополнительной экономии битрейта по сравнению с per-title и >65% снижения rebuffering .

**Chunk-parallel transcoding.**
Вместо транскодирования всего видео одним job, видео разбивается на chunks (2-4 секунды, IDR boundaries). Каждый chunk транскодируется параллельно на разных workers. Это позволяет:
- Масштабировать горизонтально (добавлять workers).
- Уменьшить время транскодирования (O(chunk_count / workers) вместо O(video_duration)).
- Изолировать failures (один chunk failed — перекодировать только его).

Клеппман в «Высоконагруженных приложениях» объясняет, что транскодинг — это компромисс между качеством, bandwidth и вычислительной стоимостью. Более эффективные кодеки (AV1) экономят bandwidth, но дороже в кодировании. Per-shot encoding экономит bandwidth, но требует анализа. Chunk-parallel transcoding ускоряет, но добавляет сложность оркестрации. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как транскодинг влияет на архитектуру. Если у тебя много загрузок, chunk-parallel transcoding обязателен.

**Схема 20.5 — Транскодинг: кодеки, битрейт-ладдер, per-shot encoding**

```
TRANSCODING PIPELINE:

Raw video (vid_123/original.mp4)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Demux                                                      │
│  ├── video stream                                           │
│  ├── audio stream                                           │
│  └── subtitles                                              │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Segment (IDR boundaries, 2-4 sec chunks)                   │
│  ├── chunk_001 (0-2s)                                       │
│  ├── chunk_002 (2-4s)                                       │
│  ├── chunk_003 (4-6s)                                       │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Analyze each shot (per-shot bit budget)                    │
│  ├── shot_001 (low motion) → bitrate: 2 Mbps                │
│  ├── shot_002 (high motion) → bitrate: 4 Mbps               │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Encode (parallel)                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ chunk_001 × codec × rung                            │   │
│  │ ├── h264 360p (0.5 Mbps)                            │   │
│  │ ├── h264 720p (2 Mbps)                              │   │
│  │ ├── vp9 360p (0.4 Mbps)                             │   │
│  │ ├── vp9 720p (1.5 Mbps)                             │   │
│  │ └── ...                                             │   │
│  └─────────────────────────────────────────────────────┘   │
│  (each chunk × codec × rung is a separate task)             │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  QC (VMAF)                                                  │
│  ├── если quality < threshold: re-encode at higher rung     │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Package                                                    │
│  ├── CMAF (fragmented MP4)                                  │
│  ├── HLS (.m3u8 manifest)                                   │
│  └── DASH (.mpd manifest)                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
S3 (encoded-bucket)
```

---

### 20.6 Адаптивный битрейт: HLS, DASH, манифесты

Теперь давай разберём, как доставлять видео с адаптивным битрейтом.

**HLS (HTTP Live Streaming).** Протокол от Apple. Видео разбивается на сегменты (обычно 2-10 секунд), и каждый сегмент доступен в разных качествах. Манифест (.m3u8) указывает, где находятся сегменты и какие качества доступны. Плеер выбирает качество в зависимости от скорости интернета.

**DASH (Dynamic Adaptive Streaming over HTTP).** Открытый стандарт, похожий на HLS. Использует MPD (Media Presentation Description) манифест.

**Adaptive bitrate (ABR).** Плеер измеряет скорость загрузки сегментов и выбирает следующее качество. Если скорость высокая — выбирает высокое качество, если низкая — низкое. Это позволяет избежать rebuffering (остановки видео) и обеспечить максимальное качество.

**Манифесты.**
HLS manifest (.m3u8):
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/index.m3u8
```

DASH manifest (.mpd):
```xml
<MPD>
  <Period>
    <AdaptationSet mimeType="video/mp4">
      <Representation id="360p" bandwidth="800000" width="640" height="360">
        <SegmentTemplate media="360p/segment_$Number$.ts" />
      </Representation>
      <Representation id="720p" bandwidth="2500000" width="1280" height="720">
        <SegmentTemplate media="720p/segment_$Number$.ts" />
      </Representation>
      <Representation id="1080p" bandwidth="5000000" width="1920" height="1080">
        <SegmentTemplate media="1080p/segment_$Number$.ts" />
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

Клеппман в «Высоконагруженных приложениях» объясняет, что адаптивный битрейт — это компромисс между качеством и стабильностью. Более высокое качество лучше, но может вызвать rebuffering. ABR выбирает баланс. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как адаптивный битрейт влияет на архитектуру. Если у тебя video streaming, HLS или DASH обязательны.

**Схема 20.6 — Адаптивный битрейт: HLS, DASH, манифесты**

```
HLS MANIFEST:

manifest.m3u8
┌─────────────────────────────────────────────────────────────┐
│ #EXTM3U                                                     │
│ #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360       │
│ 360p/index.m3u8                                             │
│ #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720     │
│ 720p/index.m3u8                                             │
│ #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080    │
│ 1080p/index.m3u8                                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  360p/index.m3u8                                            │
│  #EXTM3U                                                    │
│  #EXTINF:4.0,                                               │
│  segment_001.ts                                             │
│  #EXTINF:4.0,                                               │
│  segment_002.ts                                             │
│  #EXTINF:4.0,                                               │
│  segment_003.ts                                             │
│  ...                                                        │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  CDN Edge                                                   │
│  ├── 360p/segment_001.ts (TTL: 7 days)                      │
│  ├── 360p/segment_002.ts (TTL: 7 days)                      │
│  ├── 720p/segment_001.ts (TTL: 7 days)                      │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘

ABR (Adaptive Bitrate):

Плеер измеряет скорость загрузки:
- Скорость 5 Mbps → выбирает 1080p
- Скорость падает до 2 Mbps → переключается на 720p
- Скорость падает до 0.8 Mbps → переключается на 360p
- Скорость растёт до 4 Mbps → переключается на 1080p

Цель: избежать rebuffering, максимизировать качество.
```

---

### 20.7 CDN: edge, origin, cache invalidation

Теперь давай разберём CDN — сеть доставки контента.

**CDN (Content Delivery Network)** — это распределённая система серверов, которая доставляет контент пользователям. CDN состоит из:
- **Edge servers:** серверы, которые находятся близко к пользователям и кешируют контент.
- **Origin servers:** серверы, где хранится оригинальный контент (S3).
- **Content router:** компонент, который направляет пользователя к оптимальному edge server .

**Как работает CDN.**
1. Пользователь запрашивает видео.
2. DNS направляет запрос к ближайшему edge server (через GeoDNS или Anycast).
3. Edge server проверяет, есть ли сегмент в кеше.
4. Если есть (cache hit) — возвращает сегмент.
5. Если нет (cache miss) — запрашивает у origin (S3), кеширует и возвращает.

**Cache hit ratio.**
YouTube поддерживает ~98.5–99% cache hit ratio для VOD . Это означает, что 98.5–99% сегментов обслуживаются из CDN, и только 1–1.5% идут в origin. Без CDN origin не справился бы с трафиком.

**Cache invalidation.**
Если видео удалено или обновлено, нужно инвалидировать CDN кеш. Это делается через API CDN (например, Cloudflare purge, AWS CloudFront invalidation). Инвалидация может быть по URL или по тегу.

**TTL (Time To Live).**
TTL определяет, как долго сегмент хранится в CDN. Для популярных видео TTL может быть 7 дней. Для непопулярных — 1 день. Для манифестов — 1 час (потому что они могут меняться).

Клеппман в «Высоконагруженных приложениях» объясняет, что CDN — это компромисс между задержкой, bandwidth и стоимостью. CDN снижает задержку и bandwidth на origin, но добавляет стоимость. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как CDN влияет на архитектуру. Если у тебя video streaming, CDN обязателен.

**Схема 20.7 — CDN: edge, origin, cache invalidation**

```
CDN ARCHITECTURE:

Viewer (Europe) ──▶ DNS ──▶ CDN Edge (Frankfurt)
                                │
                                ├── Cache HIT (98.5%) ──▶ вернуть сегмент
                                │
                                └── Cache MISS (1.5%) ──▶ CDN Origin Shield
                                                              │
                                                              └──▶ S3 (origin)
                                                                     │
                                                                     └── вернуть сегмент
                                                                     └── кешировать на Edge

Viewer (US) ──▶ DNS ──▶ CDN Edge (New York)
                          │
                          └── ... (то же самое)

CDN EDGE CACHE:

┌─────────────────────────────────────────────────────────────┐
│  Edge Server (Frankfurt)                                    │
│  ├── vid_123/360p/segment_001.ts (TTL: 7 days)              │
│  ├── vid_123/720p/segment_001.ts (TTL: 7 days)              │
│  ├── vid_123/1080p/segment_001.ts (TTL: 7 days)             │
│  ├── vid_456/360p/segment_001.ts (TTL: 1 day)               │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘

CACHE INVALIDATION:

Видео vid_123 удалено:
CDN API ──▶ Purge /cdn/vid_123/*
         ──▶ Edge Servers (все) удаляют сегменты vid_123

TTL Strategy:
- Popular videos: TTL 7 days
- Unpopular videos: TTL 1 day
- Manifests: TTL 1 hour (могут меняться)
```

---

### 20.8 Отказоустойчивость, мониторинг и безопасность

Теперь давай разберём, как обеспечить отказоустойчивость, мониторинг и безопасность.

**Отказоустойчивость.**
- Failover: если Upload Service или Watch Service упал, клиенты переподключаются к другому экземпляру (через load balancer).
- Circuit breaker: если база данных или S3 недоступны, сервис быстро возвращает ошибку.
- Graceful degradation: если транскодинг недоступен, видео остаётся в статусе processing (пользователь видит «обрабатывается»).
- Retry: с exponential backoff для временных сбоев (S3 upload, Kafka publish).
- Timeout: все запросы должны иметь timeout (2 секунды для VST).

**Мониторинг.**
- Client-side QoE: Video Start Time (VST) < 2s (P95), Rebuffering ratio < 1%, Bitrate distribution, Frame drop rate, ABR switches .
- Server-side: Cache hit ratio (98.5–99%), TTFB < 200ms (P95), Origin request rate, Bandwidth per PoP .
- Transcoding pipeline: Queue depth, Processing latency (P95, P99), Failure rate < 0.1%, Cost per minute .
- Databases: Query latency (P95) < 10ms, Shard QPS balance, Replication lag < 1s .
- Трейсинг: OpenTelemetry, Jaeger (trace ID across API gateway → metadata service → CDN edge → player).
- Алерты: VST P95 > 3s, Rebuffering > 2%, Cache hit ratio < 95%, Transcoding queue depth > 10k jobs, Database P99 latency > 100ms .

**Безопасность.**
- Аутентификация: JWT для загрузки и просмотра (опционально для публичных видео).
- Авторизация: создатель может удалять только свои видео.
- Rate limiting: ограничение загрузок (например, 10 в день), просмотров (например, 1000 в час).
- Валидация: проверка формата видео, размера, кодека.
- Модерация: автоматическая (ML-классификаторы) и ручная.
- DRM (опционально): Widevine, FairPlay, PlayReady для защиты от пиратства.
- Секреты: хранение в Vault, ротация.
- Аудит: логирование загрузок, удалений, просмотров.

Клеппман в «Высоконагруженных приложениях» объясняет, что отказоустойчивость, мониторинг и безопасность — это не фичи, а свойства системы. Без них система не может эксплуатироваться. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как отказоустойчивость, мониторинг и безопасность влияют на архитектуру. Если у тебя публичный video streaming, они обязательны.

**Схема 20.8 — Полная архитектура с отказоустойчивостью и мониторингом**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Observability Stack                         │
│  Prometheus ──▶ Grafana (dashboards)                                │
│  Loki ──▶ Grafana (logs)                                            │
│  Jaeger/Tempo ──▶ Grafana (traces)                                  │
│  Alertmanager ──▶ PagerDuty/Slack (alerts)                          │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                    Upload / Watch Service                           │
│  ┌──────────────────────────────┼──────────────────────────────┐   │
│  │  Circuit Breaker             │                              │   │
│  │  ├── PostgreSQL: CLOSED/OPEN/HALF-OPEN                      │   │
│  │  ├── S3: CLOSED/OPEN/HALF-OPEN                              │   │
│  │  └── Kafka: CLOSED/OPEN/HALF-OPEN                           │   │
│  │                                                             │   │
│  │  Graceful Degradation:                                      │   │
│  │  ├── S3 недоступен: 503 для upload                          │   │
│  │  ├── Kafka недоступен: транскодинг отложен                  │   │
│  │  └── CDN недоступен: fallback на origin (медленнее)         │   │
│  │                                                             │   │
│  │  Timeout: 2s для VST, 500ms для metadata                    │   │
│  │  Retry: exponential backoff + jitter                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ PostgreSQL    │         │ S3            │         │ Kafka Cluster │
│ Master        │         │ (raw, encoded,│         │ (3 brokers)   │
│      │        │         │  thumbnails)  │         │               │
│      ▼        │         │               │         │ topic:        │
│ Replica 1     │         │               │         │ video_uploaded│
│ Replica 2     │         │               │         │ video_ready   │
└───────────────┘         └───────────────┘         └───────┬───────┘
                                                            │
                                                            ▼
                                                    ┌───────────────┐
                                                    │ Transcoder    │
                                                    │ Workers       │
                                                    │ (chunk-       │
                                                    │  parallel)    │
                                                    │               │
                                                    │ - demux       │
                                                    │ - segment     │
                                                    │ - encode      │
                                                    │ - package     │
                                                    └───────────────┘

CDN:
┌─────────────────────────────────────────────────────────────────────┐
│  Edge Servers (global)                                              │
│  ├── Cache HIT: 98.5–99%                                            │
│  ├── TTFB: < 200ms (P95)                                            │
│  └── Origin Shield: reduce origin load                              │
└─────────────────────────────────────────────────────────────────────┘

Security Layer:
┌─────────────────────────────────────────────────────────────────────┐
│  WAF (Cloudflare) ──▶ Rate Limiter (Redis) ──▶ JWT Auth            │
│  Vault (secrets) ──▶ Content Moderation (ML) ──▶ DRM (optional)    │
│  Audit Log (append-only)                                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Выводы

В этой главе ты узнал, как строить video streaming платформу. Ты разобрал транскодинг, адаптивный битрейт (HLS, DASH), CDN. Ты узнал, как хранить и доставлять петабайты видео. Ты узнал, что такое per-shot encoding и почему он экономит bandwidth. Ты увидел, как те же строительные блоки (S3, CDN, Kafka) применяются к video streaming задаче.

Главное, что ты должен вынести из этой главы: **video streaming — это задача о хранении, обработке и доставке контента.** Хорошая архитектура — не та, которая использует самый модный кодек. Хорошая архитектура — та, которая обеспечивает низкую задержку, высокое качество, экономию bandwidth и масштабируемость.

---

## Выжимка для быстрого повторения

- **Требования:** загрузка, просмотр, адаптивное качество, модерация. Доступность 99.99%, VST < 2s, rebuffering < 1%.
- **Оценки:** 500 часов/мин загрузок = 5.2 ПБ/день raw, 25 ПБ/день encoded. 1 млрд часов/день просмотров = 1.35 ЭБ/день egress. CDN offload 95%.
- **API:** POST /upload (multipart), GET /watch/{id} (metadata), GET /cdn/{id}/manifest.m3u8 (HLS), GET /cdn/{id}/manifest.mpd (DASH).
- **Данные:** videos (шардирован по creator_id), video_variants (шардирован по video_id). S3 для raw, encoded, thumbnails.
- **Транскодинг:** H.264 (fallback), H.265, VP9, AV1. Битрейт-ладдер от 360p до 4K. Per-shot encoding экономит ~30% bandwidth. Chunk-parallel transcoding для масштабирования.
- **Адаптивный битрейт:** HLS (.m3u8), DASH (.mpd). ABR выбирает качество в зависимости от скорости.
- **CDN:** edge cache, origin shield, cache hit ratio 98.5–99%. TTL для популярных видео 7 дней, для непопулярных — 1 день. Cache invalidation через API.
- **Мониторинг:** VST < 2s, rebuffering < 1%, cache hit ratio > 95%, transcoding failure rate < 0.1%.
- **Безопасность:** JWT, rate limiting, валидация, модерация, DRM (опционально), аудит.

---

## Вопросы для самопроверки

1. Почему video streaming требует CDN? Как работает cache hit ratio и почему он важен?
2. Что такое адаптивный битрейт? Чем HLS отличается от DASH?
3. Что такое per-shot encoding и почему он экономит bandwidth?
4. Почему chunk-parallel transcoding лучше centralized transcoding для больших объёмов?
5. Как обеспечить отказоустойчивость video streaming платформы?

---

## Эталонные ответы

**1. Почему video streaming требует CDN? Как работает cache hit ratio и почему он важен?**

Video streaming генерирует эксабайты трафика в день (1.35 ЭБ/день egress для YouTube). Origin (S3) не может обслужить такой трафик: это дорого и медленно. CDN кеширует видео на edge-серверах, близких к пользователям. Cache hit ratio — это процент запросов, которые обслуживаются из CDN без обращения к origin. YouTube поддерживает 98.5–99% для VOD. Это означает, что только 1–1.5% трафика идёт в origin, что снижает нагрузку и стоимость. Без CDN origin не справился бы.

**2. Что такое адаптивный битрейт? Чем HLS отличается от DASH?**

Адаптивный битрейт (ABR) — это техника, при которой видео доступно в нескольких качествах (битрейтах), и плеер выбирает качество в зависимости от скорости интернета. Это позволяет избежать rebuffering и максимизировать качество. HLS (HTTP Live Streaming) — протокол от Apple, использует .m3u8 манифест. DASH (Dynamic Adaptive Streaming over HTTP) — открытый стандарт, использует .mpd манифест. HLS лучше поддерживается в Apple-устройствах, DASH — в остальных. Оба используют HTTP-сегменты, но разные форматы манифестов.

**3. Что такое per-shot encoding и почему он экономит bandwidth?**

Per-shot encoding — это техника, при которой каждый shot (сцена) в видео анализируется, и для него выбирается оптимальный битрейт. Вместо одного битрейта для всего видео (per-title), per-shot encoding учитывает сложность каждой сцены: низкомоционные сцены (говорящая голова) требуют меньше битрейта, высокомоционные (спорт) — больше. Netflix Dynamic Optimizer (per-shot encoding) дал ~30% дополнительной экономии битрейта по сравнению с per-title и >65% снижения rebuffering. Это экономит bandwidth без потери воспринимаемого качества.

**4. Почему chunk-parallel transcoding лучше centralized transcoding для больших объёмов?**

Centralized transcoding: один job на всё видео, время транскодирования O(video_duration). Для 4-часового видео это часы. Если job failed, нужно перекодировать всё. Chunk-parallel transcoding: видео разбивается на chunks (2-4 секунды), каждый chunk транскодируется параллельно на разных workers. Время транскодирования O(chunk_count / workers). Если chunk failed, перекодируется только он. Это масштабируется горизонтально (добавлять workers) и изолирует failures. Для больших объёмов (500 часов/мин) chunk-parallel обязателен.

**5. Как обеспечить отказоустойчивость video streaming платформы?**

Failover: load balancer направляет трафик на здоровые экземпляры. Circuit breaker: если S3 или база данных недоступны, сервис быстро возвращает ошибку. Graceful degradation: если транскодинг недоступен, видео остаётся в статусе processing. Retry: exponential backoff для временных сбоев. Timeout: VST < 2s. CDN: если edge недоступен, запрос идёт в origin (медленнее, но работает). Мониторинг: алерты на VST > 3s, rebuffering > 2%, cache hit ratio < 95%. Безопасность: JWT, rate limiting, валидация, модерация.

---

## Мост к следующей главе

Ты спроектировал video streaming платформу — от требований до продакшена. Ты узнал, что такое транскодинг, адаптивный битрейт, CDN, per-shot encoding. Ты увидел, как те же строительные блоки применяются к video streaming задаче.

Но video streaming — это про доставку контента. А что если задача — это деньги? Что если нужно проводить платежи, которые не должны теряться, дублироваться или быть мошенническими? Что если каждая транзакция должна быть атомарной, идемпотентной и аудируемой?

В следующей главе — **Payment system: транзакции, идемпотентность, аудит**. Ты узнаешь, как работают платёжные системы, почему ACID критичен, как обеспечить идемпотентность, как обрабатывать сбои, как проводить аудит. Ты узнаешь, что такое PCI DSS и почему безопасность в платежах — это не опция. И ты поймёшь, что payment system — это задача о строгой согласованности и надёжности, где ошибка стоит денег.