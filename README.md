# Virtual Threads in Java 21+ — Интерактивная презентация

> Урок для разработчиков уровня **Middle / Senior** · Project Loom · JEP 444
> 26 слайдов · ~3–4 часа · живые анимации и диаграммы

## Живая демо

**[sorokin-school.github.io/order-payment-system-JavaShifu](https://sorokin-school.github.io/order-payment-system-JavaShifu/)**

Презентация автоматически деплоится на GitHub Pages при каждом push в `main`.

---

## Темы презентации

| Раздел | Что разбираем |
|--------|---------------|
| §1 Проблема Platform Threads | Thread per Request, Little's Law, стоимость OS-потока (~1 МБ, ~1 мс на создание) |
| §2 Архитектура Virtual Threads | Carrier threads, ForkJoinPool, N:M маппинг, heap-стек (~300 Б) |
| §3 Continuation & Mount/Unmount | Как JVM сохраняет и восстанавливает стек, work-stealing scheduler |
| §4 API создания | `Thread.ofVirtual()`, `Executors.newVirtualThreadPerTaskExecutor()`, ThreadFactory |
| §5 Антипаттерн: Pinning | `synchronized` + I/O, JNI, диагностика через `-Djdk.tracePinnedThreads` и JFR |
| §6 ThreadLocal vs ScopedValue | Масштабируемость контекста при миллионах VT, immutable propagation |
| §7 Structured Concurrency | `StructuredTaskScope`, `ShutdownOnFailure`, `ShutdownOnSuccess`, auto-cancel |
| §8 Производительность | I/O-bound vs CPU-bound, benchmark Platform vs Virtual vs Reactive, Spring Boot 3.2+ |
| §9 Миграция кода | Чеклист: замена пулов, `synchronized` → `ReentrantLock`, аудит ThreadLocal |
| §10 Отладка и инструментарий | JFR события, `jcmd`, IntelliJ IDEA 2023.2+, читаемые стектрейсы |

**Финальное задание:** Параллельный агрегатор заказов — `StructuredTaskScope` + `ScopedValue` + JFR-верификация отсутствия Pinning-событий.

---

## Запуск локально

### Открыть напрямую в браузере

```bash
git clone https://github.com/sorokin-school/order-payment-system-JavaShifu.git
cd order-payment-system-JavaShifu

# Windows
start index.html

# macOS
open index.html

# Linux
xdg-open index.html
```

> Reveal.js подключается через CDN — нужно интернет-соединение.

### Локальный HTTP-сервер (рекомендуется)

```bash
# Python
python3 -m http.server 8000
# → http://localhost:8000

# Node.js
npx serve .
# → http://localhost:3000
```

### Навигация

| Действие | Клавиша |
|----------|---------|
| Следующий слайд | `→` или `Space` |
| Предыдущий слайд | `←` |
| Полноэкранный режим | `F` |
| Обзор всех слайдов | `Esc` |
| Заметки докладчика | `S` |

---

## Структура

```
.
├── index.html               # Reveal.js презентация (26 слайдов)
├── OUTLINE.md               # Детальный план урока с примерами кода
└── .github/
    └── workflows/
        └── deploy.yml       # CI/CD → GitHub Pages
```

---

## CI/CD

При каждом push в `main` GitHub Actions копирует `index.html` и `OUTLINE.md` в артефакт и деплоит на GitHub Pages. Время деплоя ~30–60 секунд.

**Первоначальная настройка GitHub Pages:**
Settings → Pages → Source → выбрать **GitHub Actions**
