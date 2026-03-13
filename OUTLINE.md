# Урок: Введение в Virtual Threads (Java 21+)

> **Уровень:** Middle / Senior
> **Продолжительность:** ~3-4 часа (теория + практика)
> **Предварительные требования:** Java 11+, понимание многопоточности, знакомство с Executor Framework

---

## Цели урока

По завершении урока студент будет уметь:

- Объяснить, чем Virtual Threads отличаются от Platform Threads и почему это важно
- Понять модель планирования: carrier threads, mount/unmount, continuation
- Создавать Virtual Threads через различные API (Thread, Executors, StructuredTaskScope)
- Идентифицировать антипаттерны: thread-local злоупотребления, pinning, thread pools поверх virtual threads
- Мигрировать существующий код с платформенных потоков на виртуальные
- Применять Structured Concurrency для управления жизненным циклом задач
- Профилировать и диагностировать проблемы с Virtual Threads через JFR и JVM-флаги

---

## Диаграммы для урока

| # | Диаграмма | Раздел |
|---|-----------|--------|
| 1 | **Thread per Request vs. Virtual Thread Model** — сравнение потребления памяти и пропускной способности | §1 |
| 2 | **Platform Thread vs. Virtual Thread: стек и структура** — OS thread ↔ carrier thread ↔ virtual thread | §2 |
| 3 | **Mount / Unmount Lifecycle** — состояния виртуального потока: NEW → RUNNABLE → BLOCKED (unmounted) → RUNNABLE (remounted) → TERMINATED | §3 |
| 4 | **Continuation Model** — как JVM сохраняет и восстанавливает стек вызовов при блокировке | §3 |
| 5 | **Pinning Cases** — когда виртуальный поток остаётся «пришпиленным» к carrier: synchronized, JNI | §5 |
| 6 | **Structured Concurrency: дерево задач** — родительская задача → дочерние задачи → область видимости (scope) | §7 |
| 7 | **Thread-Local vs. Scoped Values** — разница в propagation и изоляции данных | §6 |
| 8 | **Throughput Benchmark** — сравнение latency/throughput: blocking IO с Platform Threads vs. Virtual Threads vs. Reactive | §8 |

---

## Разделы урока

---

### §1 — Проблема: почему нас не устраивают платформенные потоки?

**Тема:** Исторический контекст и мотивация появления Virtual Threads

**Ключевые концепции:**
- Модель "thread per request" и её пределы масштабируемости
- Стоимость Platform Thread: ~1 МБ стека, ~1 мс на создание, ограничение ОС
- Callback Hell и Reactive Programming как попытки обойти проблему
- "Little's Law": пропускная способность = λ · W, и при чём тут потоки
- Project Loom как ответ на задачу масштабирования

**Практический пример:**

```java
// Демонстрация потолка платформенных потоков
try {
    List<Thread> threads = new ArrayList<>();
    for (int i = 0; i < 10_000; i++) {
        Thread t = new Thread(() -> {
            try { Thread.sleep(10_000); } catch (InterruptedException e) {}
        });
        t.start();
        threads.add(t);
    }
    System.out.println("Создано потоков: " + threads.size());
} catch (OutOfMemoryError e) {
    System.err.println("OOM: " + e.getMessage());
}
```

**Вопрос для обсуждения:** Почему Reactive-подход решает проблему масштабируемости, но ценой чего?

---

### §2 — Архитектура Virtual Threads: под капотом JVM

**Тема:** Как JVM реализует виртуальные потоки

**Ключевые концепции:**
- **Platform Thread (carrier thread)** — OS thread, управляемый планировщиком ОС
- **Virtual Thread** — сущность JVM, планируемая ForkJoinPool-ом (work-stealing scheduler)
- Стек Virtual Thread хранится в **heap** (growable, начинается с ~200-300 байт)
- Соотношение N:M — M виртуальных потоков работают поверх N carrier threads (N ≈ CPU cores)
- `Thread.ofVirtual()` vs `Thread.ofPlatform()` — новый unified API
- Virtual Thread не имеет приоритета, является daemon-потоком

---

### §3 — Continuation и планировщик: как работает mount/unmount

**Тема:** Механизм парковки и возобновления виртуальных потоков

**Ключевые концепции:**
- **Continuation** — абстракция, позволяющая сохранить и восстановить состояние стека
- **Mount** — виртуальный поток назначается carrier thread и начинает выполнение
- **Unmount** — при блокирующей операции стек сохраняется в heap, carrier thread освобождается
- **Work-Stealing ForkJoinPool** — планировщик виртуальных потоков (по умолчанию)

---

### §4 — API Virtual Threads: создание и управление

**Ключевые концепции:**
- `Thread.ofVirtual().start(runnable)` — прямое создание
- `Executors.newVirtualThreadPerTaskExecutor()` — ExecutorService
- Почему **не нужен** пул виртуальных потоков (thread pooling anti-pattern)

---

### §5 — Антипаттерны и подводные камни: Pinning

**Ключевые концепции:**
- **Pinning** — виртуальный поток "пришпилен" к carrier thread и не может unmount
- Причина #1: `synchronized` блок/метод с блокирующей операцией внутри
- Причина #2: вызов нативного (JNI) кода
- Диагностика: `-Djdk.tracePinnedThreads=full` или JFR event `jdk.VirtualThreadPinned`
- Решение: замена `synchronized` на `ReentrantLock`

---

### §6 — ThreadLocal и Scoped Values: управление контекстом

**Ключевые концепции:**
- `ThreadLocal` работает с Virtual Threads, но создаёт проблемы при миллионах VT
- **ScopedValue** (Java 21) — immutable, bounded, propagatable
- Паттерн использования: запрос пришёл → установили контекст → задачи его читают

---

### §7 — Structured Concurrency: порядок в параллельных задачах

**Ключевые концепции:**
- **Structured Concurrency** — дочерние задачи не переживают родительскую область видимости
- `StructuredTaskScope.ShutdownOnFailure` — при первой ошибке отменяем все задачи
- `ShutdownOnSuccess` — берём первый успешный результат (race pattern)

---

### §8 — Производительность и когда применять Virtual Threads

**Ключевые концепции:**
- Virtual Threads отлично подходят для **I/O-bound** задач
- Virtual Threads **не ускоряют** CPU-bound задачи
- Spring Boot 3.2+: `spring.threads.virtual.enabled=true`

---

### §9 — Миграция существующего кода

**Чеклист миграции:**
```
□ Заменить Executors.newFixedThreadPool(N) → newVirtualThreadPerTaskExecutor()
□ Найти synchronized + блокирующий вызов → заменить на ReentrantLock
□ Оценить ThreadLocal — если > 1000 потоков, рассмотреть ScopedValue
□ Запустить с -Djdk.tracePinnedThreads=full
□ Провести нагрузочный тест (k6 / Gatling)
□ Проверить JFR-запись на VirtualThreadPinned events
```

---

### §10 — Отладка и инструментарий

**Ключевые концепции:**
- Stack traces Virtual Threads — читаемые, полные (в отличие от Reactive)
- IntelliJ IDEA 2023.2+: поддержка Virtual Threads в Threads view
- JFR (Java Flight Recorder) — основной инструмент профилирования

---

## Финальное практическое задание

### Задача: "Параллельный агрегатор заказов"

**Контекст:** Параллельно:
1. Проверить инвентарь (latency ~50ms)
2. Проверить кредитный лимит клиента (latency ~80ms)
3. Получить актуальный курс валюты (latency ~30ms)

**Требования:**
1. Реализовать через StructuredTaskScope.ShutdownOnFailure
2. Передавать Request ID через ScopedValue (не ThreadLocal)
3. Все synchronized-блоки с I/O заменить на ReentrantLock
4. Запустить с JFR и убедиться в отсутствии VirtualThreadPinned events

**Скелет:**
```java
static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

AggregatedResult processOrder(String requestId, Order order)
        throws InterruptedException, ExecutionException {
    return ScopedValue.where(REQUEST_ID, requestId).call(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var inventoryTask = scope.fork(() -> checkInventory(order));
            var creditTask    = scope.fork(() -> checkCredit(order));
            var rateTask      = scope.fork(() -> getExchangeRate(order.currency()));
            scope.join().throwIfFailed();
            return new AggregatedResult(
                inventoryTask.get(), creditTask.get(), rateTask.get());
        }
    });
}
```

**Критерии оценки:**
- Нет pinning events в JFR-записи
- Суммарное время обработки ≈ max(50, 80, 30) = ~80ms
- ScopedValue корректно передаётся во все дочерние задачи

---

## Дополнительные материалы

| Ресурс | Ссылка |
|--------|--------|
| JEP 444: Virtual Threads | openjdk.org/jeps/444 |
| JEP 453: Structured Concurrency | openjdk.org/jeps/453 |
| JEP 446: Scoped Values | openjdk.org/jeps/446 |
| Ron Pressler — "Loom: Bringing Lightweight Threads to Java" | YouTube / Devoxx |
| Spring Boot 3.2 Virtual Threads Guide | spring.io/blog |
