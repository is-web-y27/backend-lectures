# Предметно-ориентированное проектирование, часть 2

<!-- .slide: style="text-align: center" -->

---

## DDD и стратегические паттерны

При разработке сложных приложений часто используется подход **DDD (Domain-Driven Design)**, который позволяет отталкиваться от предметной области, а не от особенностей реализации.

На уровне проектирования общей архитектуры DDD вводит несколько понятий (a.k.a. стратегических паттернов):

* Домен и поддомен;
* Единый язык и ограниченный контекст;
* Карта контекстов.

Note:

* **Домен**&nbsp;— вся предметная область, **поддомен**&nbsp;— часть этой предметной области.

* **Единый язык**&nbsp;— общий понятийный аппарат в рамках поддомена, используемый как заказчиками, так и разработчиками.

* **Ограниченный контекст**&nbsp;— область приложения, в которой используется совпадающий единый язык.

* **Карта контекстов** показывает границы контекстов и связи между ними.

---

## DDD и тактические паттерны

На уровне архитектуры приложения DDD также вводит некоторые понятия (тактические паттерны):

* Сущность (Entity);
* Объект-значение (Value Object);
* Агрегат (Aggregate).

Note:

* **Сущность**&nbsp;— объект, который нас интересует сам по себе и определяется идентификатором;

* **Объект-значение**&nbsp;— объект, у которого важно само значение и который не может существовать отдельно;

* **Агрегат**&nbsp;— сборная сущность, которая обновляется в рамках одной транзакции и с которой мы работаем как с одним целым.

---

## Бизнес-логика в DDD

Всё рассмотренное до этого касалось статичного описания доменной области. Для простых сценариев этого может быть достаточно, но для сложных сценариев взаимодействия сущностей нужны дополнительные концепции.

<div class="fragment">

Естественно, предметно-ориентированное проектирование даёт ответ и на этот вопрос в виде ещё одного набора тактических паттернов:

* Доменные события;
* Доменные сервисы;
* Репозитории;
* Фабрики;
* Модули.

</div>

Note:

Естественно, предметно-ориентированное проектирование даёт ответ и на этот вопрос. Можно ввести несколько новых концепций, часть из которых непосредственно отвечает за логику, а часть помогает удобно и логично структурировать код.

---

## Доменные события

Часто мы в рамках приложения хотим знать, что произошло какое-то событие, например, создался новый экземпляр сущности. Можно воспользоваться паттернами ООП, но в дополнение к нему DDD вводит понятие доменного события.

**Доменное событие**&nbsp;— это описание некоторого события, которое произошло в одной части системы и может быть интересно в других частях системы.

Note:

Часто мы в рамках приложения хотим знать, что произошло какое-то событие, например, создался новый экземпляр сущности. Можно воспользоваться стандартным паттерном [*Наблюдатель*](https://refactoring.guru/ru/design-patterns/observer), но в дополнение к нему DDD вводит понятие доменного события.

Основное преимущество доменных событий&nbsp;— расширяемость системы: необязательно изменять существующий код, чтобы добавить новый обработчик события. При этом нет смысла создавать события "на всякий случай", лучше начать с небольшого набора и дополнять его по мере необходимости.

---

## Доменные события

Несмотря на то, что работа с событиями как подходом широко известна, DDD накладывает некоторые правила относительно их структуры и поведения:

* Доменные события **неизменяемы**;
* Содержат временную метку наступления события;
* Не имеют побочных эффектов с точки зрения создателя события;
* Инициируются корнями агрегатов или доменными сервисами и публикуются через слой приложения или инфраструктуры (не в самом домене).

Note:

DDD накладывает некоторые правила относительно их структуры и поведения:

* Доменные события **неизменяемы**, в противном случае мы будем изменять прошедшие события :)

* Содержат временную метку (timestamp) наступления события;

* Не имеют побочных эффектов с точки зрения той части системы, которая создаёт событие;

* Инициируются корнями агрегатов или доменными сервисами и публикуются через слой приложения или инфраструктуры (не в самом домене).

---

<!-- .slide: style="text-align: center" -->

```ts []
class LessonRescheduledEvent {
    constructor(
        public readonly lessonId: LessonId,
        public readonly occurredAt: Date = new Date()
    ) {}
}
```

Пример доменного события. Они не обязаны быть большими, суть именно в том, что событие произошло.

---

<!-- .slide: style="text-align: center" -->

```ts []
class Lesson {
    private domainEvents: LessonRescheduledEvent[] = [];

    // Вызываем метод, когда изменяется, например, день недели
    public changeSchedule(): void {
        // Часть логики опущена...
        this.domainEvents.push(new LessonRescheduledEvent(this.id));
    }

    public pullDomainEvents(): LessonRescheduledEvent[] {
        const events = [...this.domainEvents];
        this.domainEvents = [];
        return events;
    }
}
```

В агрегате мы фиксируем факт доменного события, а публикацию обычно делает сервис приложения.

---

<!-- .slide: style="text-align: center" -->

```ts []
class LessonApplicationService {
    // Инициализация опущена...

    async changeSchedule(lessonId: LessonId) {
        const lesson = await this.lessonRepository.findById(lessonId);
        lesson.changeSchedule();
        await this.lessonRepository.save(lesson);

        for (const event of lesson.pullDomainEvents()) {
            this.eventBus.publish(event);
        }
    }
}

@EventsHandler(LessonRescheduledEvent)
export class LessonRescheduledHandler implements IEventHandler<LessonRescheduledEvent> {
    async handle(event: LessonRescheduledEvent) {
        // Обрабатываем событие, например, делаем рассылку
        console.log('Lesson rescheduled:', event.lessonId, event.newStartAt);
    }
}
```

Где-то (обычно в обработчике вне домена) мы уже обрабатываем опубликованное событие.

---

## Event sourcing

Иногда этот подход используют как основной механизм хранения состояния объектов, и такая идея получила название **event sourcing**. При этом подходе текущее состояние объекта собирается из последовательности событий.

<div class="fragment">

* Плюсы: полная история и состояние на любой момент.
* Минусы: сложнее чтение и инфраструктура.

</div>

Note:

Иногда подход с событиями используется как основной механизм хранения состояния, и появляется идея хранить текущее состояние приложения в виде набора событий. Такая идея получила название **event sourcing**. При таком подходе текущее состояние сущностей и агрегатов собирается путём последовательного применения изменений из событий.

Часто этот паттерн применяется в микросервисных приложениях, поэтому про него мы поговорим в том числе и там.

N.B. Обычно термин "event sourcing" не переводится.

Плюсы такого подхода:

* Можно получить состояние объекта на заданный момент времени;
* Появляется журнал событий.

Минусы такого подхода:

* Если событий много, то сборка объекта с нуля становится дольше;
* Для ускорения чтения данных приходится реализовывать дополнительные паттерны (можно почитать про CQRS).

В подавляющем количестве случаев можно отдельно хранить список произошедших событий в журнале событий (audit log) и работать с данными в БД как обычно.

---

## Доменные сервисы

Есть ситуации, когда те или иные сценарии нельзя строго отнести к какому-то классу с данными. Тогда появляется доменный сервис.

**Доменный сервис**&nbsp;— класс, который содержит бизнес-логику поддомена, которая естественным образом не может быть отнесена к другим классам.

<div class="fragment">

Отличия от других тактических паттернов:

* Каждый доменный сервис отвечает ровно за одну задачу.
* Доменные сервисы не хранят в себе состояние между вызовами.

Удобный способ проверки корректности выделения доменного сервиса: никто на него не ссылается (т.е. нет ассоциаций).

</div>

Note:

Даже в том случае, когда разработчики стараются упаковать бизнес-логику в сущности и агрегаты, есть ситуации, когда те или иные сценарии нельзя строго отнести к какому-то классу с данными. Тогда появляется доменный сервис.

**Доменный сервис**&nbsp;— класс, который содержит бизнес-логику поддомена, которая естественным образом не может быть отнесена к другим классам.

Что ещё отличает доменный сервис от других классов DDD:

* Каждый доменный сервис отвечает ровно за одну задачу. При этом необязательно это означает, что в классе будет один метод.

* Доменные сервисы не хранят в себе состояние между вызовами. Чаще это обычный класс с методами и зависимостями, внедряемыми через DI.

Доменные сервисы могут взаимодействовать с другими сервисами и с репозиториями, **не храня собственного состояния**. При этом сущности не могут ссылаться на сервис напрямую (т.е. между ними не должно быть ассоциации), в противном случае есть вероятность, что логика в сервисе должна принадлежать сущности.

---

## Сервисы в разных слоях

Важный момент: не путайте между собой доменный сервис, сервис приложения и инфраструктурный сервис.

* Доменный сервис содержит бизнес-правила домена;
* Сервис приложения оркестрирует сценарий использования;
* Инфраструктурный сервис реализует технические детали (доступ к БД, внешние API, шины событий и т.д.).

Note:

Domain Service:

* Живёт в слое домена.
* Содержит бизнес-правила, которые неестественно класть в одну сущность/агрегат.
* Работает в терминах доменных объектов и UL.
* Не знает про HTTP, БД, брокеры, фреймворк.

Пример: `LessonReschedulePolicyService` проверяет, можно ли переносить занятие по бизнес-правилам.

Application Service:

* Живёт в слое application/use-case.
* Оркестрирует сценарий: загрузить агрегат, вызвать доменные методы/сервисы, сохранить, опубликовать событие.
* Управляет транзакционной границей и последовательностью шагов.
* Может использовать репозитории, domain services, event bus (через абстракции).

Пример: `LessonApplicationService.rescheduleLesson(...)`.

Infrastructure Service;

* Живёт в infrastructure.
* Реализует технические детали и адаптеры: SQL/ORM, внешние API, очередь, e-mail, кеш и т.д.
* Реализует контракты, нужные приложению/домену.
* Не должен содержать бизнес-правил домена.

Пример: `PgLessonRepository`, `PrismaLessonRepository`, `KafkaEventPublisher`.

---

<!-- .slide: style="text-align: center" -->

```ts []
class LessonReschedulePolicyService {
    public canReschedule(lesson: Lesson, newSlot: LessonEvent): boolean {
        // Пример доменного правила: нельзя переносить занятие в уже закрытый период
        return !lesson.isArchived() && !lesson.hasExamWeekConflict(newSlot);
    }
}
```

```ts []
class LessonApplicationService {
    constructor(
        private readonly lessonRepository: LessonRepository,
        private readonly lessonReschedulePolicyService: LessonReschedulePolicyService,
    ) {}

    public async rescheduleLesson(id: LessonId, newSlot: LessonEvent): Promise<void> {
        const lesson = await this.lessonRepository.findById(id);
        if (!lesson) throw new Error("Lesson not found");

        if (!this.lessonReschedulePolicyService.canReschedule(lesson, newSlot)) {
            throw new Error("Reschedule is forbidden by domain policy");
        }

        lesson.rescheduleTo(newSlot);
        await this.lessonRepository.save(lesson);
    }
}
```

Note:

Здесь `LessonReschedulePolicyService` содержит бизнес-правило, а `LessonApplicationService` только оркестрирует сценарий.

---

## Фабрики

Для создания объектов со сложной инициализацией удобно использовать фабрику.

**Фабрика** в контексте DDD практически ничем не отличается от порождающего паттерна ООП. Основная идея фабрики&nbsp;— вынести специфичную бизнес-логику создания объекта, которая вовлекает в себя другие объекты.

<div class="fragment">

Варианты реализации фабрики с точки зрения DDD:

* Отдельный класс-фабрика, ответственный за создание одного класса агрегатов или объектов-значений;
* Абстрактная фабрика в привычном понимании ООП;
* Фабричные методы в корнях агрегатов.

Например, если мы хотим иметь возможность добавлять в дисциплины контрольные точки, это должно относиться к агрегату `Subject`...

</div>

Note:

Ещё одним тактическим паттерном DDD является фабрика. В случае DDD понятие **фабрики** практически совпадает с соответствующими порождающими шаблонами проектирования из ООП: это метод класса (фабричный метод) или отдельный класс (абстрактная фабрика), создающий объекты другого класса.

Отличие в том, что нам не нужно иметь какие-то иерархии классов, чтобы применять фабрики в DDD, основная суть&nbsp;— вынести логику создания объекта.

Фабрики нужно писать только тогда, когда создание объекта требует специфичной бизнес-логики, и эта логика не относится исключительно к объекту.

Есть несколько вариантов реализации фабрики с точки зрения DDD:

* Отдельный класс-фабрика, ответственный за создание одного типа объектов (какой-то агрегат или value object);
* Абстрактная фабрика в привычном понимании ООП;
* Фабричные методы в понимании ООП, которые реализованы в корнях агрегатов.

Основной интерес с точки зрения разделения ответственности представляет реализация фабричных методов, поскольку здесь всё так же нужно правильно отнести логику к тому или иному объекту. Например, если мы хотим иметь возможность добавлять в дисциплины контрольные точки, это должно относиться к агрегату `Subject`...

---

<!-- .slide: style="text-align: center" -->

```ts
class Checkpoint {
    constructor(
        private id: CheckpointId,
        private name: CheckpointName,
        private points: Points,
        private subjectId: SubjectId,
    ) {}
}
```

```ts
class Subject {
    constructor(
        private id: SubjectId,
        private checkpoints: Checkpoint[],
        // Остальные поля опущены...
    ) {}

    public addCheckpoint(name: CheckpointName, points: Points) {
        const checkpointId = new CheckpointId(generateCheckpointId());
        const checkpoint = new Checkpoint(checkpointId, name, points, this.id);
        this.checkpoints.push(checkpoint);
    }
}
```

... например, вот так.

---

<!-- .slide: style="text-align: center" -->

```ts
class SubjectFactory {
    public createSubject(
        name: SubjectName,
        semester: Semester,
        maxStudents: PositiveInt,
    ): Subject {
        const subject = Subject.create(
            new SubjectId(generateSubjectId()),
            name,
            semester,
            maxStudents,
        );

        // Бизнес-правило: у дисциплины сразу есть контрольная точка
        // для промежуточной аттестации
        subject.addCheckpoint(new CheckpointName("Экзамен"), new Points(20));
        return subject;
    }
}
```

Отдельная фабрика уместна, когда создание агрегата требует согласованного набора доменных правил.

---

## Репозитории

Для описания работы с хранилищем данных в терминах домена часто используется **репозиторий**. Чаще всего репозиторий выглядит как связка интерфейса на уровне домена и реализации на уровне инфраструктуры.

* Может быть collection-oriented и persistence-oriented:
  * Первый случай имитирует работу с коллекцией;
  * Второй случай имитирует работу с хранилищем;
* На каждый агрегат свой репозиторий;
* Ответственен за получение и сохранение агрегатов.

---

<!-- .slide: style="text-align: center" -->

```ts []
interface LessonRepository {
    findById(id: LessonId): Promise<Lesson | null>;
    findBySubject(subjectId: SubjectId): Promise<Lesson[]>;
    save(lesson: Lesson): Promise<void>;
}
```

```ts []
class InMemoryLessonRepository implements LessonRepository {
    private readonly storage = new Map<string, Lesson>();

    async findById(id: LessonId): Promise<Lesson | null> {
        return this.storage.get(id.value) ?? null;
    }

    async findBySubject(subjectId: SubjectId): Promise<Lesson[]> {
        return [...this.storage.values()].filter(x => x.subjectIdEquals(subjectId));
    }

    async save(lesson: Lesson): Promise<void> {
        this.storage.set(lesson.id.value, lesson);
    }
}
```

Collection-oriented репозиторий выглядит как коллекция доменных объектов, а не как SQL API.

---

<!-- .slide: style="text-align: center" -->

```ts []
type LessonRow = { id: string; subject_id: string; starts_at: Date; ends_at: Date };
type LessonFilter = { subject_id?: string; starts_from?: Date; starts_to?: Date };
type LessonPage = { items: LessonRow[]; total: number };

interface LessonPersistenceRepository {
    findPage(filter: LessonFilter, options: { limit: number; offset: number }): Promise<LessonPage>;
}
```

```ts []
class PgLessonPersistenceRepository implements LessonPersistenceRepository {
    constructor(private readonly db: DbClient) {}

    async findPage(filter: LessonFilter, options: { limit: number; offset: number }): Promise<LessonPage> {
        const items = await this.db.query<LessonRow>(
            `select id, subject_id, starts_at, ends_at from lessons
             where ($1::uuid is null or subject_id = $1) and ($2::timestamptz is null or starts_at >= $2)
               and ($3::timestamptz is null or starts_at <= $3)
             order by starts_at limit $4 offset $5`,
            [filter.subject_id ?? null, filter.starts_from ?? null, filter.starts_to ?? null, options.limit, options.offset],
        );
        const totalRow = await this.db.queryOne<{ total: number }>(
            "select count(*)::int as total from lessons where ($1::uuid is null or subject_id = $1)",
            [filter.subject_id ?? null],
        );
        return { items, total: totalRow?.total ?? 0 };
    }
}
```

Note:

Обратите внимание, что тут в интерфейс просачивается больше деталей реализации хранилища: пагинация, фильтры и т.п.

---

## Модули

**Модули** в DDD помогают зафиксировать границы концептов на уровне кода и удерживать бизнес-структуру в архитектуре:

* Группируют классы, описывающие один концепт с точки зрения бизнеса;
* Не всегда концепт один-в-один отражает контекст, контекст может содержать несколько концептов;
* Можно представить как пространство имён или пакет.

Не надо разделять приложение на модули, исходя из технического назначения классов (отдельно сущности, отдельно сервисы и пр.).

Note:

Модуль в DDD важен как механизм архитектурной дисциплины: он не только группирует код, но и ограничивает область языка. Через модули проще контролировать зависимости и не допускать утечек терминов между разными концептами.

---

<!-- .slide: style="text-align: center" -->

```text
scheduling/
├── domain/
│   ├── lesson.ts
│   ├── lesson-event.ts
│   ├── lesson-reschedule-domain.service.ts
│   └── lesson.repository.ts
├── application/
│   └── lesson-application.service.ts
└── infrastructure/
    ├── pg-lesson.repository.ts
    └── lesson-event-bus.publisher.ts
```

Модуль группирует всё вокруг бизнес-концепта `scheduling`, а не по техническим слоям всего приложения.

---

<!-- .slide: style="text-align: center" -->

## Объектно-реляционное преобразование (ORM)

Note:

Даже хорошо спроектированная доменная модель должна сохраняться в хранилище.

Нужен механизм отображения объектной модели на реляционные структуры, чтобы бизнес-логика не деградировала до работы напрямую с таблицами.

---

## Паттерны взаимодействия с БД

* Table Data Gateway;
* Active Record;
* Data Mapper;
* Object-Relational Mapping.

Они различаются по степени проникновения деталей хранения в доменную модель.

---

## Table Data Gateway

Самая простая реализация репозитория:

* Работает с одной таблицей;
* Содержит в себе все CRUD-запросы для работы с таблицей;
* Ничего не знает про сущности.

Note:

Table Data Gateway полезен, когда нужен простой и прозрачный доступ к таблице без сложной доменной модели. Ещё один плюс в простоте его написания. Ограничение подхода в том, что бизнес-логика остаётся снаружи, а сам gateway работает на уровне строк и SQL-операций.

---

```ts []
class LessonTableGateway {
    constructor(private readonly db: DbClient) {}

    async findById(id: string): Promise<{ id: string; starts_at: Date; ends_at: Date } | null> {
        return this.db.queryOne(
            "select id, starts_at, ends_at from lessons where id = $1",
            [id],
        );
    }

    async insert(id: string, startsAt: Date, endsAt: Date): Promise<void> {
        await this.db.execute(
            "insert into lessons(id, starts_at, ends_at) values ($1, $2, $3)",
            [id, startsAt, endsAt],
        );
    }
}
```

---

## Active Record

* Обычно одна модель отображается на одну таблицу (или представление);
* Объект содержит данные и операции сохранения/удаления (`save`, `delete`);
* Поисковые операции часто оформляются как `class`/`static`-методы.

Не самый лучший подход, поскольку нарушается принцип единственной ответственности и в целом детали реализации хранилища проникают в домен.

---

```ts
class LessonRecord {
    constructor(
        public id: string,
        public startsAt: Date,
        public endsAt: Date,
    ) {}

    static async findById(db: DbClient, id: string): Promise<LessonRecord | null> {
        const row = await db.queryOne<{ id: string; starts_at: Date; ends_at: Date }>(
            "select id, starts_at, ends_at from lessons where id = $1",
            [id],
        );
        return row ? new LessonRecord(row.id, row.starts_at, row.ends_at) : null;
    }

    async save(db: DbClient): Promise<void> {
        await db.execute(
            "update lessons set starts_at = $2, ends_at = $3 where id = $1",
            [this.id, this.startsAt, this.endsAt],
        );
    }
}
```

---

## Data Mapper

**Data Mapper** отделяет доменную модель от хранения: доменные объекты не обязаны знать SQL, таблицы и формат строк.

* Может работать с несколькими таблицами;
* Отдельный класс для работы с хранилищем;
* Знает про сущности;
* Изолирует домен от хранилища.

Note:

Data Mapper отделяет доменную модель от хранения: доменные объекты не обязаны знать SQL, таблицы и формат строк. Это повышает тестируемость домена и упрощает эволюцию хранилища, но добавляет слой маппинга и его стоимость поддержки.

---

```ts
class LessonMapper {
    toDomain(row: LessonRow): Lesson {
        return Lesson.restore(
            new LessonId(row.id), 
            new SubjectId(row.subject_id), 
            row.starts_at, 
            row.ends_at
        );
    }

    toRow(lesson: Lesson): LessonRow {
        return {
            id: lesson.id.value,
            subject_id: lesson.subjectId.value,
            starts_at: lesson.startsAt,
            ends_at: lesson.endsAt,
        };
    }
}

class LessonDataMapperRepository {
    constructor(private readonly db: DbClient, private readonly mapper: LessonMapper) {}

    async findById(id: LessonId): Promise<Lesson | null> {
        const row = await this.db.queryOne<LessonRow>(
            "select * from lessons where id = $1", [id.value]
        );
        return row ? this.mapper.toDomain(row) : null;
    }
}
```

---

## ORM

**Object-Relational Mapping** (ORM) автоматизирует маппинг и часто включает паттерны Identity Map или Unit of Work.

* Объектно-реляционное преобразование;
* Может работать с графами объектов (т.е. агрегатами);
* Связывает объектную модель приложения с реляционной моделью хранения;
* Может работать с несколькими таблицами.

Никогда не пишите ORM сами :)

---

<div style="display: grid; grid-template-columns: 1fr 2fr"><div>

```ts []
@Entity("lessons")
class LessonOrmModel {
    @PrimaryColumn("uuid")
    id!: string;

    @Column("uuid")
    subjectId!: string;

    @Column("timestamptz")
    startsAt!: Date;

    @Column("timestamptz")
    endsAt!: Date;
}
```

</div><div>

```ts []
class LessonOrmRepository implements LessonRepository {
    constructor(
        private readonly ormRepo: Repository<LessonOrmModel>
    ) {}

    async findById(id: LessonId): Promise<Lesson | null> {
        const model = await this.ormRepo.findOneBy({ 
            id: id.value 
        });
        return model ? Lesson.restore(
            new LessonId(model.id), 
            new SubjectId(model.subjectId), 
            model.startsAt, 
            model.endsAt
        ) : null;
    }
}
```

</div></div>

---

## Identity Map

Основная идея: один экземпляр сущности на один `id` в рамках единицы работы.

* Кэш в рамках единицы работы: один и тот же объект загружается один раз;
* Предотвращает дублирование экземпляров одной и той же сущности в памяти;
* Упрощает отслеживание изменений.

Note:

Identity Map нужен для консистентности в памяти: если в одном use-case один и тот же объект загружается несколько раз, вы получаете один экземпляр, а не копии. Это снижает риск расхождения изменений и упрощает работу Unit of Work.

---

```ts []
class IdentityMap<T> {
    private readonly storage = new Map<string, T>();

    get(id: string): T | undefined {
        return this.storage.get(id);
    }

    set(id: string, entity: T): void {
        this.storage.set(id, entity);
    }
}
```

```ts []
class LessonRepositoryWithIdentityMap {
    constructor(private readonly db: DbClient, private readonly map: IdentityMap<Lesson>) {}

    async findById(id: LessonId): Promise<Lesson | null> {
        const cached = this.map.get(id.value);
        if (cached) return cached;

        const row = await this.db.queryOne<LessonRow>("select * from lessons where id = $1", [id.value]);
        if (!row) return null;

        const lesson = Lesson.restore(new LessonId(row.id), new SubjectId(row.subject_id), row.starts_at, row.ends_at);
        this.map.set(id.value, lesson);
        return lesson;
    }
}
```

---

## Unit of Work

Основная идея: собираем изменения по объектам и фиксируем их одним коммитом.

* Отслеживает новые/изменённые/удалённые объекты;
* Фиксирует изменения пакетом в конце use-case;
* Часто работает совместно с Identity Map.

Note:

Unit of Work концентрирует запись изменений в конец сценария: сначала бизнес-логика работает с объектами, затем изменения фиксируются пакетом. Это помогает обеспечить атомарность и согласованность при сохранении нескольких связанных модификаций.

---

```ts []
class UnitOfWork {
    private readonly newLessons: Lesson[] = [];
    private readonly dirtyLessons: Lesson[] = [];

    constructor(private readonly lessonRepository: LessonRepository) {}

    registerNew(lesson: Lesson): void {
        this.newLessons.push(lesson);
    }

    registerDirty(lesson: Lesson): void {
        this.dirtyLessons.push(lesson);
    }

    async commit(): Promise<void> {
        for (const lesson of this.newLessons) {
            await this.lessonRepository.save(lesson);
        }
        for (const lesson of this.dirtyLessons) {
            await this.lessonRepository.save(lesson);
        }
        this.newLessons.length = 0;
        this.dirtyLessons.length = 0;
    }
}
```

---

## Веб-сервисы

Альтернативный источник данных, который иногда используют вместо работы с БД напрямую.

Note:

Напоминалка: кроме реляционной БД, источником данных может быть внешний сервис. В таком случае инфраструктурный слой выполняет адаптацию ответа сервиса в доменную модель, сохраняя контракт репозитория в терминах домена.

---

## Работа с данными в экосистеме TypeScript

* Sequelize (проблемы с новыми фичами);
* TypeORM (есть типизация, но при запросах она исчезает, code-first);
* Prisma (есть продвинутая типизация за счёт кодогенерации, schema-first).

---

## TypeORM

TypeORM обычно используют как инфраструктурный адаптер к БД.

Для DDD обычно выбирают Data Mapper-подход и не тянут ORM-декораторы в домен.

* Плюсы: быстрый старт, репозитории, транзакции и query builder.
* Риски: протекание ORM в домен и необходимость строгой архитектурной дисциплины.

Note:

TypeORM обычно используют как инфраструктурный адаптер к реляционной БД:

* Контракт репозитория остаётся в домене;
* Маппинг entity-классов и SQL скрывается в infrastructure;
* Вариант с Data Mapper лучше подходит для DDD, чем Active Record.

Практически это значит: доменные агрегаты не должны зависеть от `@Entity`, `@Column` и других ORM-декораторов.

Плюсы:

* Быстрый старт с сущностями и миграциями;
* Знакомый паттерн `Repository`;
* Поддержка транзакций и query builder.

Ограничения:

* Легко «протащить» ORM-типы в домен;
* Сложные выборки часто теряют удобную строгую типизацию;
* Требует дисциплины в разделении domain/application/infrastructure.

---

<div style="display: grid; grid-template-columns: 1fr 2fr"><div>

```ts []
@Entity("lessons")
class LessonOrmEntity {
    @PrimaryColumn("uuid")
    id!: string;

    @Column("uuid")
    subjectId!: string;

    @Column("timestamptz")
    startsAt!: Date;

    @Column("timestamptz")
    endsAt!: Date;
}
```

</div><div>

```ts []
class TypeOrmLessonRepository implements LessonRepository {
    constructor(
        private readonly ormRepo: Repository<LessonOrmEntity>
    ) {}

    async findById(id: LessonId): Promise<Lesson | null> {
        const row = await this.ormRepo.findOneBy({ 
            id: id.value 
        });
        return row ? Lesson.restore(
            new LessonId(row.id), 
            new SubjectId(row.subjectId), 
            row.startsAt, 
            row.endsAt
        ) : null;
    }

    async save(lesson: Lesson): Promise<void> {
        await this.ormRepo.save({
            id: lesson.id.value,
            subjectId: lesson.subjectId.value,
            startsAt: lesson.startsAt,
            endsAt: lesson.endsAt,
        });
    }
}
```

</div></div>

---

## Prisma

Prisma чаще выбирают за schema-first подход и сильную типизацию client API.

В DDD Prisma обычно используется как инфраструктурный слой, а доменные объекты собираются вручную или через mapper.

* Сильные стороны: строгая типизация и удобный schema-first workflow.
* Ограничения: в сложных доменных сценариях всё равно нужен ручной маппинг и архитектурная дисциплина.

Note:

Prisma хорошо подходит, когда нужен:

* schema-first подход;
* строго типизированный клиент;
* предсказуемый SQL через декларативную схему.

В DDD Prisma обычно используется как инфраструктурный слой, а доменные объекты собираются вручную или через mapper.

Плюсы:

* Сильная типизация запросов на этапе компиляции;
* Удобные миграции и прозрачная schema-first модель;
* Хорошая предсказуемость для CRUD-сценариев.

Ограничения:

* В сложных доменных запросах всё равно нужен явный mapper;
* Иногда не хватает гибкости query builder уровня SQL;
* Как и с любой ORM, без архитектурной дисциплины домен можно «загрязнить» инфраструктурой.

---

```prisma []
model Lesson {
  id        String   @id @db.Uuid
  subjectId String   @db.Uuid
  startsAt  DateTime @db.Timestamptz(6)
  endsAt    DateTime @db.Timestamptz(6)

  @@index([subjectId])
}
```

```ts []
class PrismaLessonRepository implements LessonRepository {
    constructor(private readonly prisma: PrismaClient) {}

    async findById(id: LessonId): Promise<Lesson | null> {
        const row = await this.prisma.lesson.findUnique({ 
            where: { id: id.value } 
        });
        return row ? Lesson.restore(
            new LessonId(row.id), 
            new SubjectId(row.subjectId), 
            row.startsAt, 
            row.endsAt
        ) : null;
    }
}
```

---

## Вопросы?

<!-- .slide: style="text-align: center" -->
