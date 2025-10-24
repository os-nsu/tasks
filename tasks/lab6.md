# Интерфейс библиотеки корутин (в начале файла — код)

```c
/*
 *  coroutine_pool.h
 */

#ifndef COROUTINE_POOL_H
#define COROUTINE_POOL_H

#include <stddef.h>
#include <stdint.h>

#endif



typedef struct coro_t coro_t;
typedef struct coro_sched_t coro_sched_t;
typedef void (*coro_fn_t)(void *arg);

/* Create a scheduler/coroutine pool.
 * max_coroutines — the maximum number of simultaneously existing coroutines.
 * stack_size — default stack size for new coroutines (if 0, uses default value).
 * Returns NULL on failure.
 */
coro_sched_t *coro_sched_create(size_t max_coroutines, size_t stack_size);

/* Free the scheduler and all resources (should be called after cor_sched_shutdown). */
void coro_sched_destroy(coro_sched_t *sched);

/* Initialization/shutdown (alternative to create/destroy). */
int coro_sched_init(coro_sched_t *sched, size_t max_coroutines, size_t stack_size);
void coro_sched_shutdown(coro_sched_t *sched);

/* Create a coroutine and add it to the pool: returns a pointer to the object or NULL.
 * func — coroutine function; arg — argument; flags — reserved.
 */
coro_t *coro_create(coro_sched_t *sched, coro_fn_t func, void *arg, size_t stack_size);

/* Current coroutine (inside the executed function) */
coro_t *coro_current(void);

/* Unique coroutine ID (within the scheduler) */
uint64_t coro_id(coro_t *c);

/* Explicitly yield control to the scheduler:
 * — some other READY coroutine will resume according to the scheduler policy.
 * — if there are no other ready coroutines, control returns to the current one.
 */
void cor_yield(void);

/* Terminate the current coroutine (does not return) */
void cor_exit(void) __attribute__((noreturn));

/* Run the scheduler loop: blocking function that executes coroutines
 * until all coroutines finish or an interruption occurs.
 */
void cor_run(coro_sched_t *sched);

/* Return the number of coroutines in different states (ready/waiting/finished) */
size_t cor_count_ready(coro_sched_t *sched);
size_t cor_count_waiting(coro_sched_t *sched);
size_t cor_count_total(coro_sched_t *sched);

/* -----------------------
   Synchronization primitives
   (cooperative — works within a single thread)
   ----------------------- */

/* Coroutine queue (opaque) — used internally by primitives */
typedef struct cor_queue_t cor_queue_t;

/* Mutex (ownership by coroutine required) */
typedef struct {
    int locked;            /* 0/1 */
    coro_t *owner;         /* owner or NULL */
    cor_queue_t *waiters;  /* queue of waiting coroutines */
} cor_mutex_t;

void cor_mutex_init(cor_mutex_t *m);
void cor_mutex_lock(cor_mutex_t *m);    /* suspends the coroutine if locked */
int  cor_mutex_trylock(cor_mutex_t *m); /* returns 0 on success, !0 on failure immediately */
void cor_mutex_unlock(cor_mutex_t *m);  /* only owner can unlock */
void cor_mutex_destroy(cor_mutex_t *m);

/* Semaphore */
typedef struct {
    int value;
    cor_queue_t *waiters;
} cor_semaphore_t;

void cor_sem_init(cor_semaphore_t *s, int initial);
void cor_sem_wait(cor_semaphore_t *s);
int  cor_sem_trywait(cor_semaphore_t *s);
void cor_sem_post(cor_semaphore_t *s);
void cor_sem_destroy(cor_semaphore_t *s);

/* Read-Write lock (cooperative semantics) */
typedef struct {
    int readers;
    int writer;            /* 0/1 */
    cor_queue_t *r_waiters;
    cor_queue_t *w_waiters;
} cor_rwlock_t;

void cor_rwlock_init(cor_rwlock_t *rw);
void cor_rwlock_rdlock(cor_rwlock_t *rw);
int  cor_rwlock_tryrdlock(cor_rwlock_t *rw);
void cor_rwlock_wrlock(cor_rwlock_t *rw);
int  cor_rwlock_trywrlock(cor_rwlock_t *rw);
void cor_rwlock_unlock(cor_rwlock_t *rw);
void cor_rwlock_destroy(cor_rwlock_t *rw);

/* -----------------------
   Timeouts and event waiting
   ----------------------- */

/* Suspend the current coroutine until timeout (ms) or explicit wake-up.
 * Returns 0 — externally woken, 1 — timeout.
 */
int cor_sleep_ms(unsigned int ms);

/* Wake up one/all coroutines waiting on an object (opaque token).
 * Used to implement wait/notify pattern.
 */
typedef void *cor_wait_token_t;
void cor_wait_on(cor_wait_token_t token);
void cor_notify_one(cor_wait_token_t token);
void cor_notify_all(cor_wait_token_t token);

/* -----------------------
   Diagnostics / debugging
   ----------------------- */
const char *coro_state_name(coro_t *c); /* "READY"/"RUNNING"/"WAITING"/"FINISHED" */
void coro_dump_stats(coro_sched_t *sched); /* print statistics to stderr */

/* -----------------------
   Implementation notes (not API, but useful)
   - coroutines should have separate stacks (heap-allocated)
   - contexts — setcontext/getcontext/makecontext or hand-written asm
   - all blocking operations of a coroutine are cooperative (they call cor_yield)
   ----------------------- */

#ifdef __cplusplus
}
#endif

#endif /* COROUTINE_POOL_H */

```


# Задание: библиотека корутин для однопоточного пула

## Краткое описание (цель работы)

Реализовать **библиотеку корутин на С** (далее — *RepaCoro*), которая предоставляет:

* пул корутин в рамках **одного POSIX-потока** (один планировщик на поток);
* кооперативное переключение контекста (корутина уступает управление сама — `cor_yield`, при ожидании примитива и т.п.);
* набор синхронизационных примитивов для корутин: `mutex`, `semaphore`, `rwlock`, `wait/notify`;
* механизмы таймаутов и ожидания событий;
* корректную работу с ресурсами (стеки, объекты примитивов), детекцию утечек;
* детерминированное/обоснованное поведение планировщика (стратегия планирования должна быть описана и обоснована).

Библиотека должна быть пригодна для встраивания в потокобезопасные серверные приложения (например, в следующей лабораторной будет использоваться для реализации lightweight worker pool).


## Функциональные требования (детали)

1. **Корутины и пул**

   * Динамическое создание/удаление корутин.
   * `cor_run()` запускает цикл планировщика; `cor_yield()` — явное уступание.
   * Поддержка `cor_exit()` внутри корутины.
   * Нулевой или малый оверхед на переключение.
   * Корректная обработка ошибок: создание при исчерпании памяти — безопасное сообщение об ошибке, не приводящее к краху.

2. **Планировщик**

   * Ready-queue (FIFO) — по умолчанию.
   * Требуется реализовать и обосновать *одну альтернативную стратегию* (например, LIFO для коротких задач, приоритеты, round-robin с квантованием в количестве yield’ов).
   * Планировщик должен поддерживать приоритеты (опционально) или политики fairness.
   * Планировщик не должен крутиться в пустом цикле при отсутствии готовых корутин — минимальное потребление CPU. (Если библиотека используется в одном потоке, то ожидание внешних событий — ответственность приложения; тем не менее, внутри `cor_run` предусмотреть условные точки ожидания/возврата).

3. **Примитивы синхронизации**

   * `cor_mutex_t` — кооперативный mutex; при попытке захвата занятым — текущая корутина ставится в очередь waiters и вызывается планировщик.
   * `cor_semaphore_t` — стандартная семафорная семантика.
   * `cor_rwlock_t` — допустимы множественные читатели или один писатель; ожидающие писатели не должны страдать starvation (обосновать стратегию).
   * `wait/notify` — объекты- токены: корутина может ждать на `token`, другая корутина — пробудить одну или все ожидания.
   * Все выше — **кооперативные** (не используют syscalls). Они работают только внутри пула корутин в одном потоке.

4. **Ресурсы**

   * Стек корутины — выделяется на куче; при уничтожении корутины освобождается.
   * Планировщик имеет `shutdown` API: аккуратное завершение всех корутин, вызов cleanup-хендлеров.
   * Отчёт о ресурсах: API для сбора статистики (число созданных корутин, пиковое число, текущие waiters и т. п.).


## Стратегия шедулера (обоснование)

В решении требуется выбрать основную стратегию планирования и обосновать выбор..


## Требуемые артефакты (что нужно сдать)

1. **Исходники** библиотеки (`.c`/`.h`) + `Makefile` / `CMakeLists`.
1. **README.md** с:
   * инструкция по сборке,
   * API (коротко),
   * описание стратегии планирования и её обоснование,
   * ограничения/предположения.

1. **Краткий отчёт** (ниже шаблон), включающий схемы и объяснения.


## Метрики и тесты для оценки работоспособности

* **Функциональные тесты**:

  * Создание/завершение N корутин.
  * Mutex: отсутствие гонок (проверка инвариантов).
  * Semaphore: корректное количество одновременно "в исполнении".
  * RWLock: читатели/писатели корректно синхронизированы.

* **Производительность**:

  * Время переключения (микросекунды/наносекунды) — измерение времени `yield` (среднее/пике).
  * Параллельность: пропускная способность для простых задач (например, N корутин, каждая делает M yield’ов)

* **Нагрузка/ресурсы**:

  * Память: пиковый объём, утечки (valgrind/ASAN).
  * CPU в простое: близок к нулю (нет busy-wait).

* **Корректность shutdown**:

  * Симуляция graceful shutdown: запустить, создать N корутин, вызвать shutdown, проверить, что все завершились и ресурсы освобождены.


## Маленький спойлер для следующей лабораторной

> В следующей лабораторной ReCoro будет использована как runtime для исполнения lightweight tasks внутри worker-пула сетевого сервера. Это позволит обрабатывать тысячи логических соединений с минимальными накладными расходами на создание потоков ОС.


