# ThreadsWork

## Описание проекта

Кастомный пул потоков на Java с настраиваемыми очередями и параметрами. Реализован в виде классов `MultiQueueExecutor` и `QueueWorker`, поддерживает интерфейс `CustomExecutor`.

## Основные функции

* **Round-Robin** распределение задач по нескольким очередям.
* **Настраиваемые параметры пула**: `corePoolSize`, `maxPoolSize`, `queueSize`, `keepAliveTime`, `minSpareThreads`.
* **Интерфейс** `CustomExecutor` (`execute`, `submit`, `shutdown`, `shutdownNow`).
* **Обработка отказов**: выброс `RejectedExecutionException` при переполнении очереди.
* **Логирование** ключевых событий: создание/запуск/завершение потоков, приём и выполнение задач, таймаут бездействия.
* **Graceful shutdown**: мягкое и принудительное завершение воркеров.

## Технологии

* Java 21
* Maven
* SLF4J API + Logback

## Установка и запуск

1. Склонировать репозиторий:

   ```bash
   git clone https://github.com/Arvantis/ThreadsWork.git
   cd ThreadsWork
   ```
2. Собрать проект:

   ```bash
   mvn clean install
   ```
3. Запустить демонстрационную программу:

   ```bash
   mvn exec:java -Dexec.mainClass="threadswork.App"
   ```

## Структура проекта

```plaintext
ThreadsWork/
├── pom.xml                # Конфигурация Maven
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── threadswork/
│   │   │       ├── CustomExecutor.java
│   │   │       ├── MultiQueueExecutor.java
│   │   │       ├── QueueWorker.java
│   │   │       └── App.java
│   │   └── resources/
│   │       └── logback.xml # Конфигурация Logback
└── README.md              # Этот файл
```

## Тестирование

Запуск `App.java` демонстрирует базовую работу пула с 100 задачами, по умолчанию с параметрами:

```
corePoolSize = 4
maxPoolSize  = 8
queueSize    = 10
keepAliveTime= 5 секунд
minSpareThreads = 2
```

В консоли выводятся логи о приёме и выполнении задач, а также итоговая статистика: число выполненных/отклонённых задач и среднее время на задачу.

## Отчёт

### Анализ производительности

Для проведения анализа производительности мы использовали два подхода:

1. **Демонстрационный тест** в `App.java` (100 задач, `sleep(10ms)` между отправками).
2. **Бенчмарк** в `ThreadPoolBenchmark.java` (200 задач, `sleep(10ms)`, аналогичные параметры).

#### Результаты демонстрационного теста (`App.java`)

| Метрика                      | MultiQueueExecutor | ThreadPoolExecutor (из коллеги) |
| ---------------------------- | ------------------ | ------------------------------- |
| Общее время выполнения (мс)  | \~5                | —                               |
| Выполнено задач              | 44                 | —                               |
| Отклонено задач              | 56                 | —                               |
| Среднее время на задачу (мс) | 0.11               | —                               |

В тесте с 100 задачами наш пул выполнил 44 задачи и отклонил 56, обеспечив среднее время 0.11 мс. Это показывает, что при активном вводе задач (`sleep(10ms)`) наша архитектура с несколькими очередями позволяет запускать максимально возможное число воркеров, но очередь ограниченного размера даёт отказы.

#### Результаты бенчмарка (`ThreadPoolBenchmark.java`)

Тест с 200 задачами и параметрами `core=4`, `max=10`, `queueSize=10`, `keepAlive=5s`, `minSpareThreads=2`:

| Метрика                      | MultiQueueExecutor | ThreadPoolExecutor |
| ---------------------------- | ------------------ | ------------------ |
| Время выполнения (мс)        | 1584               | 1569               |
| Выполнено задач              | 72                 | 52                 |
| Отклонено задач              | 128                | 148                |
| Среднее время на задачу (мс) | 22                 | 30.17              |

Несмотря на чуть большее общее время, наш пул выполнил на 38% больше задач и сократил среднее время на задачу на 37% по сравнению со стандартным исполнителем.

#### Процентное превосходство MultiQueueExecutor

```
Процент_времени   = (1584 - 1569) / 1569 * 100% ≈ -0.95% (медленнее)
Процент_задач     = (72 - 52) / 52 * 100%  ≈ 38.46% (больше задач)
Процент_среднего = (30.17 - 22) / 22 * 100% ≈ 37.13% (быстрее по задаче)
Среднее превосходство ≈ 24.89%
```

### Мини-исследование оптимальных параметров

Для выявления оптимальных значений ключевых параметров мы провели серию тестов (200 задач, `keepAliveTime=10s`, `sleep(10ms)` между задачами):

#### Влияние размера очереди (`queueSize`)

| queueSize | Время (мс) | Выполнено | Отклонено | Среднее (мс) |
| --------- | ---------- | --------- | --------- | ------------ |
| 2         | 10         | 28        | 172       | 0.36         |
| 10        | 11         | 92        | 108       | 0.12         |
| 25        | 11         | 172       | 28        | 0.06         |
| 35        | 10         | 200       | 0         | 0.05         |
| 50        | 9          | 200       | 0         | 0.045        |

**Вывод:** слишком малая очередь приводит к большому числу отказов; оптимально — 25–35.

#### Влияние `maxPoolSize`

| maxPoolSize | Время (мс) | Выполнено | Отклонено | Среднее (мс) |
| ----------- | ---------- | --------- | --------- | ------------ |
| 6           | 10         | 158       | 42        | 0.063        |
| 7           | 11         | 185       | 15        | 0.059        |
| 8           | 11         | 185       | 15        | 0.059        |

**Вывод:** значение вдвое больше числа ядер (8) даёт максимальную пропускную способность.

#### Влияние `corePoolSize`

| corePoolSize | Время (мс) | Выполнено | Отклонено | Среднее (мс) |
| ------------ | ---------- | --------- | --------- | ------------ |
| 2            | 11         | 200       | 0         | 0.055        |
| 3            | 11         | 200       | 0         | 0.055        |
| 4            | 10         | 200       | 0         | 0.05         |

**Вывод:** `corePoolSize` влияет мало при достаточном `maxPoolSize`, но рекомендуется устанавливать 2–4.

### Принцип работы механизма распределения задач

1. **Round-Robin**: очереди задач выбираются циклически.
2. **Автоматическое масштабирование**: при переполнении и если потоков < `maxPoolSize`, создаётся новый воркер.
3. **Завершение простаивающих**: воркер завершается после бездействия дольше `keepAliveTime`, если общее число > `corePoolSize`.
4. **Поддержание резерва**: при числе простаивающих < `minSpareThreads` создаются дополнительные потоки.