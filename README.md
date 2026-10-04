# FeedService — архитектурная документация

## 1. Назначение

**FeedService** — микросервис централизованной работы с товарными данными.

Его задача — получать товарные данные из разных источников, сохранять исходные данные, приводить их к единой канонической модели, нормализовать, валидировать, хранить и предоставлять потребителям в необходимых форматах.

Основной поток:

```text
Внешний источник
      ↓
Connector
      ↓
Parser
      ↓
Source DTO
      ↓
Mapper
      ↓
RawProduct
      ↓
TypeResolver
      ↓
Type-specific Normalizer
      ↓
NormalizedProduct
      ↓
ProductAssembler
      ↓
Canonical Product
      ↓
Validator
      ↓
Repository
      ↓
PostgreSQL
      ↓
Consumers / Exports
```

FeedService является владельцем **канонической товарной модели**, но не должен забирать на себя бизнес-логику потребителей.

---

# 2. Зачем нужен FeedService

Сейчас разные системы могут работать с разными представлениями одного и того же товара:

```text
1С
 └── одна структура

Web
 └── другая структура

CRM / Mindbox
 └── третья структура

Analytics
 └── четвёртая структура
```

При прямых интеграциях:

```text
1С ─────────→ Web
1С ─────────→ CRM
Web ────────→ CRM
1С ─────────→ Analytics
...
```

возникают:

* дублирование логики;
* разные правила нормализации;
* разные названия и категории;
* сложное изменение интеграций;
* отсутствие единой модели товара;
* невозможность централизованно исправлять ошибки качества данных.

FeedService создаёт промежуточный канонический слой:

```text
             ┌──→ Web
             │
1С ──────┐   ├──→ CRM / Mindbox
         ↓   │
FeedService ─┼──→ Analytics
         ↑   │
Web ─────┘   └──→ Другие потребители
```

---

# 3. Основные задачи FeedService

FeedService отвечает за:

1. получение данных из источников;
2. сохранение исходных данных;
3. преобразование формата источника;
4. определение типа товара;
5. нормализацию данных;
6. объединение данных из нескольких источников;
7. формирование канонического товара;
8. валидацию;
9. хранение канонических данных;
10. отслеживание изменений;
11. публикацию событий;
12. формирование документов;
13. формирование экспортов для потребителей;
14. обработку ошибок и невалидных товаров;
15. повторную обработку ранее полученных данных.

FeedService **не отвечает** за бизнес-логику конечного потребителя.

Например:

* CRM сама определяет правила коммуникаций;
* Web сам отвечает за отображение;
* Analytics сам определяет свою аналитическую модель.

FeedService предоставляет качественные и согласованные данные.

---

# 4. Архитектурные принципы

Основные принципы:

### 4.1. Canonical Model

Внутри FeedService существует единая каноническая модель товара.

### 4.2. Source Independence

Источник данных не должен определять внутреннюю модель системы.

```text
1С ≠ Product
Web ≠ Product
```

Источник преобразуется в каноническую модель.

### 4.3. Raw First

Исходные данные сохраняются до нормализации.

Это позволяет:

* анализировать ошибки;
* повторно запускать нормализацию;
* менять правила без повторной загрузки из источника;
* сравнивать версии обработки.

### 4.4. Single Responsibility

Каждый этап отвечает только за свою задачу:

```text
Connector      → получить данные
Parser         → разобрать формат
Mapper         → преобразовать DTO в RawProduct
TypeResolver   → определить тип
Normalizer     → нормализовать
Assembler      → объединить источники
Validator      → проверить
Repository     → сохранить
Exporter       → сформировать представление для потребителя
```

### 4.5. Consumer Independence

Изменение формата Mindbox не должно требовать изменения канонической модели Product.

### 4.6. Explicit Delete

Исчезновение товара из очередной выгрузки не означает автоматическое удаление товара.

Удаление выполняется отдельным явным действием.

---

# 5. Архитектурная схема

Общий вид:

```text
                     SOURCES
                        │
          ┌─────────────┴─────────────┐
          │                           │
         1С                           Web
          │                           │
     Connector                    Connector
          │                           │
       Parser                      Parser
          │                           │
      Source DTO                 Source DTO
          │                           │
        Mapper                    Mapper
          │                           │
          └─────────────┬─────────────┘
                        ↓
                   RawProduct
                        ↓
                  TypeResolver
                        ↓
              Type-specific Normalizer
                        ↓
                NormalizedProduct
                        ↓
                 ProductAssembler
                        ↓
                Canonical Product
                        ↓
                    Validator
                        ↓
                   Repository
                        ↓
                   PostgreSQL
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
       REST API      Outbox        ExportService
                                      │
                           ┌──────────┼──────────┐
                           ↓          ↓          ↓
                         Mindbox     Web      Analytics
```

---

# 6. Источники данных

На первом этапе основными источниками являются:

* 1С;
* Web.

В дальнейшем источников может стать больше.

Каждый источник подключается через отдельный `Connector`.

Источники могут предоставлять данные в разных форматах:

* JSON;
* XML;
* CSV;
* API;
* файлы;
* другие форматы.

FeedService не должен предполагать, что все источники работают одинаково.

---

# 7. Потребители

Потребителями FeedService могут быть:

* Web;
* CRM / Mindbox;
* Analytics;
* мобильное приложение;
* внешние сервисы;
* другие внутренние системы.

Получение данных возможно через:

* REST API;
* события;
* готовые документы;
* периодические экспорты;
* другие механизмы, если они понадобятся.

Конкретный способ доставки определяется контрактом конкретного потребителя.

---

# 8. Слои приложения

Проект строится по принципу разделения ответственности.

```text
FeedService
│
├── Api
├── Application
├── Domain
└── Infrastructure
```

## Domain

Содержит:

* Product;
* FoodProduct;
* ToyProduct;
* Category;
* ProductType;
* ProductStatus;
* бизнес-правила;
* value objects.

Domain не знает:

* PostgreSQL;
* HTTP;
* 1С;
* Mindbox;
* JSON/XML;
* конкретные библиотеки.

---

## Application

Содержит:

* use cases;
* orchestration;
* interfaces;
* DTO;
* import pipeline;
* identity resolution;
* ProductAssembler;
* экспортные сценарии;
* application-level validation.

---

## Infrastructure

Содержит:

* connectors;
* parsers;
* source DTO;
* mappers;
* normalizers;
* repositories;
* EF Core;
* PostgreSQL;
* exporters;
* внешние интеграции;
* outbox publisher.

---

## API

Содержит:

* HTTP endpoints;
* request/response models;
* authentication/authorization;
* API-level validation.

---

# 9. Каноническая модель товара

Основная сущность:

```text
Product
```

Она содержит общие характеристики товара.

Специализированные типы наследуются от Product:

```text
Product
├── FoodProduct
├── ToyProduct
├── ClothingProduct
└── ...
```

Тип товара и категория являются **разными понятиями**.

Одна категория может содержать товары разных типов.

Например:

```text
Категория
└── Амуниция

    ├── ToyProduct
    ├── ClothingProduct
    └── ...
```

Поэтому нельзя определять `ProductType` только через `Category`.

---

# 10. Идентичность товара

Идентичность товара является фундаментальной частью модели.

Внутри FeedService используется собственный идентификатор:

```text
Product.Id
```

Это внутренний GUID канонического товара.

Внешние системы имеют собственные идентификаторы.

Поэтому внешний идентификатор хранится отдельно.

```text
Product
    │
    ├── Id = GUID
    │
    └── ProductExternalIdentifier
            ├── Source
            └── ExternalId
```

Например:

```text
Product
Id = 7f...a2

ExternalIdentifier
Source = OneC
ExternalId = 123456

ExternalIdentifier
Source = Web
ExternalId = 987654
```

Уникальность внешнего идентификатора:

```text
(Source, ExternalId)
```

### SKU

SKU является бизнес-атрибутом товара, но **не является универсальным идентификатором FeedService**.

SKU может использоваться как дополнительный признак сопоставления, но не должен быть единственным фундаментом identity resolution.

---

# 11. Identity Resolution

Identity Resolution отвечает на вопрос:

> Является ли полученный из источника товар уже существующим каноническим Product?

Базовый алгоритм:

```text
Source + ExternalId
        ↓
Поиск ProductExternalIdentifier
        ↓
 ┌──────┴──────┐
 Да            Нет
 ↓              ↓
Update         Create
```

На последующих этапах могут появиться дополнительные правила сопоставления:

* SKU;
* дополнительные бизнес-ключи;
* таблица ручных соответствий;
* другие признаки.

Эти правила инкапсулируются в отдельном компоненте:

```text
IProductIdentityResolver
```

Конкретные правила identity resolution могут расширяться без изменения модели Product.

---

# 12. Источник значения поля / Source of Truth

Один канонический товар может первоначально собираться из нескольких источников.

Например:

| Поле        | Источник |
| ----------- | -------- |
| SKU         | 1С       |
| Price       | 1С       |
| Category    | 1С       |
| Brand       | 1С       |
| Name        | 1С       |
| Description | Web      |
| PictureUrl  | Web      |
| Url         | Web      |

В дальнейшем часть данных может перейти в единый источник.

Архитектура должна поддерживать оба сценария:

```text
Несколько источников
        ↓
ProductAssembler
        ↓
Product
```

и в будущем:

```text
Один основной источник
        ↓
ProductAssembler
        ↓
Product
```

Правила выбора источника не должны быть размазаны по Normalizer-ам.

Их ответственность — нормализовать данные конкретного источника.

Ответственность за объединение принадлежит:

```text
ProductAssembler
```

---

# 13. Provenance — происхождение значения

FeedService должен предусматривать возможность определить:

> Откуда пришло конкретное значение товара?

Это особенно важно, пока разные поля приходят из разных систем.

Минимально должна сохраняться информация об источнике товара:

```text
Source
ExternalId
ReceivedAt
```

Для полей, где возможен конфликт источников, архитектура предусматривает возможность хранить provenance:

```text
Field
Source
ExternalId
UpdatedAt
```

Это позволит в будущем определить:

```text
Name → 1С
Description → Web
PictureUrl → Web
Price → 1С
```

На первом этапе механизм может быть реализован только для критичных или конфликтующих полей.

---

# 14. Product

Базовая модель:

```csharp
Product
{
    Guid Id;

    string SKU;
    string Name;
    string? Brand;

    Guid CategoryId;

    decimal? Price;
    decimal? OldPrice;

    string? PictureUrl;
    string? Url;
    string? Description;

    ProductStatus Status;

    DateTimeOffset CreatedAt;
    DateTimeOffset UpdatedAt;
}
```

`CreatedAt` и `UpdatedAt` хранятся в UTC.

---

# 15. FoodProduct

```text
FoodProduct : Product
```

Дополнительные поля:

```text
PetType
AgeGroup
Volume
Unit
Flavors
```

Например:

```text
PetType = Dog
AgeGroup = Adult
Volume = 15
Unit = Kg
Flavors = [...]
```

---

# 16. ToyProduct

```text
ToyProduct : Product
```

Дополнительные поля:

```text
PetType
Size
Color
Material
```

---

# 17. ProductType

Тип товара является отдельным атрибутом:

```text
Food
Toy
Clothing
Accessory
Medicine
...
```

Список типов расширяемый.

Тип товара определяется через `TypeResolver`.

Категория и ProductType не должны быть жёстко связаны.

---

# 18. Category

Категория является отдельной сущностью.

```text
Category
{
    Guid Id;
    string Name;
    Guid? ParentId;
}
```

Поддерживается иерархия:

```text
Товары
├── Собаки
│   ├── Корма
│   └── Амуниция
│
└── Кошки
    ├── Корма
    └── Игрушки
```

---

# 19. Категория источника и каноническая категория

Категория источника и каноническая категория **могут быть не равны**.

На первом этапе предполагается, что категория источника является корректной и может использоваться как основа для канонической категории.

Но архитектура должна позволять в будущем иметь:

```text
SourceCategory
        ↓
SourceCategoryMapping
        ↓
CanonicalCategory
```

Поэтому категория источника не должна напрямую считаться канонической категорией на уровне модели.

---

# 20. Brand

Brand хранится как атрибут товара.

На первом этапе отдельный полноценный справочник брендов не является обязательным.

При необходимости позднее Brand может быть вынесен в отдельную сущность без изменения общей архитектуры Product.

---

# 21. Хранение TPH

Для Product используется стратегия:

**TPH — Table Per Hierarchy.**

Все типы товаров хранятся в одной таблице `Products`.

Пример:

```text
Products
──────────────────────────────
Id
Discriminator

SKU
Name
Brand
CategoryId

Price
OldPrice

PictureUrl
Url
Description

PetType
AgeGroup
Volume
Unit

Size
Color
Material

Flavors

Status

CreatedAt
UpdatedAt
```

Специализированные поля для конкретного типа могут быть `NULL`.

Например:

```text
FoodProduct
PetType = Dog
Volume = 15
Unit = Kg

ToyProduct
PetType = Cat
Size = M
Color = Red
Material = Rubber
```

---

# 22. Индексы Product

Минимальный набор:

```text
PK(Id)

INDEX / UNIQUE
SKU

INDEX
CategoryId

INDEX
Discriminator

INDEX
Status

INDEX
CreatedAt

INDEX
UpdatedAt
```

Дополнительные индексы добавляются на основании реальных сценариев поиска и фильтрации.

---

# 23. ProductExternalIdentifier

Отдельная таблица:

```text
ProductExternalIdentifier
────────────────────────────
Id
ProductId
Source
ExternalId
CreatedAt
UpdatedAt
```

Ограничение:

```text
UNIQUE(Source, ExternalId)
```

Индекс:

```text
(Source, ExternalId)
```

Это позволяет одному Product иметь несколько внешних идентификаторов.

---

# 24. RawProduct

`RawProduct` — промежуточная модель, представляющая данные конкретного источника до нормализации.

Важно:

**RawProduct не является каноническим Product.**

Пример:

```text
RawProduct
{
    Source = OneC
    ExternalId = "123456"

    RawPayload =
    {
        "name": "Royal Canin Medium Adult 15 кг",
        "category": "Корм для собак",
        ...
    }
}
```

---

# 25. Хранение RawProduct

Исходный payload хранится в PostgreSQL в формате:

```text
JSONB
```

Предварительная структура:

```text
RawProductRecord
────────────────────────────
Id
Source
ExternalId

Payload
PayloadHash

ReceivedAt

Status
ProcessedAt

NormalizerVersion

ErrorCode
ErrorMessage
```

`PayloadHash` используется для определения повторно полученных данных и оптимизации обработки.

---

# 26. Жизненный цикл RawProduct

Основной поток:

```text
Received
   ↓
Processing
   ↓
Processed
```

При ошибке:

```text
Received
   ↓
Processing
   ↓
Failed / Quarantined
```

RawProduct сохраняется отдельно от канонического Product.

Это позволяет:

* посмотреть исходные данные;
* понять причину ошибки;
* найти новые правила нормализации;
* исправить данные в источнике;
* повторно обработать товар без повторной загрузки.

---

# 27. Ненормализованные товары / Quarantine

Товар, который невозможно корректно нормализовать или провалидировать, не должен попадать в каноническую модель.

Он переводится в:

```text
Quarantined
```

Причина сохраняется.

Например:

```text
TYPE_NOT_RESOLVED
INVALID_VOLUME
UNKNOWN_UNIT
REQUIRED_FIELD_MISSING
INVALID_CATEGORY
```

После изменения правил нормализации товар может быть обработан повторно.

Если причина находится в источнике, создаётся задача на исправление исходных данных.

---

# 28. Mapper

Mapper отвечает только за преобразование:

```text
Source DTO
    ↓
RawProduct
```

Он не должен:

* определять ProductType;
* парсить атрибуты;
* выбирать категорию;
* выполнять бизнес-логику.

Например:

```text
1C DTO
   ↓
OneCMapper
   ↓
RawProduct
```

AutoMapper может использоваться внутри Infrastructure, но внешний контракт должен оставаться собственным интерфейсом приложения.

---

# 29. Connector

Connector отвечает за получение данных из источника.

```text
IConnector
```

Например:

```text
OneCConnector
WebConnector
```

Connector отвечает за:

* подключение;
* авторизацию;
* получение данных;
* пагинацию;
* retry технических ошибок;
* передачу результата Parser-у.

Connector не должен выполнять нормализацию товара.

---

# 30. Pagination и batch processing

Большие источники не должны загружаться целиком в память.

Импорт выполняется страницами/батчами.

```text
1С
 ↓
Page 1 — 500 товаров
 ↓
Page 2 — 500 товаров
 ↓
Page 3 — 500 товаров
 ↓
...
```

Размер страницы конфигурируется.

Если источник поддерживает cursor-based pagination, предпочтительно использовать cursor.

Если нет — допустимы page/offset механизмы.

Внутри FeedService обработка также выполняется батчами.

Это позволяет:

* ограничить потребление памяти;
* контролировать транзакции;
* повторять отдельные партии;
* не останавливать весь импорт из-за одного товара.

---

# 31. Parser

Parser отвечает за преобразование технического формата:

```text
JSON/XML/CSV
      ↓
Source DTO
```

Parser не знает канонической модели Product.

Например:

```text
OneCJsonParser
WebJsonParser
CsvParser
XmlParser
```

---

# 32. TypeResolver

`TypeResolver` определяет:

```text
RawProduct
    ↓
ProductType
```

Контракт:

```csharp
IProductTypeResolver
{
    ProductType Resolve(RawProduct product);
}
```

Конкретные правила определения типа будут сформированы после анализа реального каталога.

Потенциальные признаки:

* категория;
* category code;
* характеристики;
* название;
* признаки источника;
* другие бизнес-признаки.

При этом категория не должна автоматически означать ProductType.

---

# 33. Normalizer

Normalizer отвечает за приведение данных к нормальной форме.

Например:

```text
"1,5 кг"
      ↓
Volume = 1.5
Unit = Kg
```

или:

```text
"для взрослых собак"
      ↓
PetType = Dog
AgeGroup = Adult
```

Архитектурно используются отдельные normalizer-ы:

```text
FoodNormalizer
ToyNormalizer
ClothingNormalizer
...
```

Общий интерфейс:

```csharp
IProductNormalizer
{
    ProductType SupportedType { get; }

    NormalizedProduct Normalize(RawProduct raw);
}
```

---

# 34. Почему Normalizer-ы разделены

Не следует создавать один огромный:

```text
UniversalProductNormalizer
```

с большим количеством:

```text
if ProductType == ...
```

Вместо этого:

```text
TypeResolver
      ↓
ProductType
      ↓
Normalizer Registry
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
Food Toy Clothing
```

Используемые паттерны:

* Strategy;
* Registry / Factory.

Это позволяет добавлять новые типы без переписывания существующей логики.

---

# 35. Нормализация и реальные правила

Правила нормализации не фиксируются исключительно на основании архитектуры.

Они формируются после анализа реальных данных.

Например:

```text
1,5 кг
1.5kg
1 500 г
1500г
15 кг
```

могут требовать разных правил парсинга.

Поэтому сначала анализируется реальный каталог, после чего создаётся:

```text
FoodNormalizer v1
ToyNormalizer v1
...
```

Версия нормализатора сохраняется в RawProduct.

---

# 36. NormalizedProduct

При наличии нескольких источников Normalizer не обязан сразу создавать финальный Product.

Он формирует:

```text
NormalizedProduct
```

Это нормализованный фрагмент данных конкретного источника.

Например:

```text
1С
 ↓
NormalizedProduct
{
    SKU
    Name
    Brand
    Price
    Category
}
```

Web:

```text
Web
 ↓
NormalizedProduct
{
    SKU
    Description
    PictureUrl
    Url
}
```

---

# 37. ProductAssembler

`ProductAssembler` объединяет нормализованные данные нескольких источников в один канонический Product.

```text
NormalizedProduct / 1С
          │
          ├──────┐
                 ↓
NormalizedProduct / Web
          │
          ↓
   ProductAssembler
          ↓
      Product
```

Assembler использует правила Source of Truth.

Например:

```text
Price       ← 1С
SKU         ← 1С
Category    ← 1С
Description ← Web
PictureUrl  ← Web
```

Это позволяет менять источники без изменения Normalizer-ов.

---

# 38. Validator

Validator проверяет уже собранный канонический Product.

Проверки:

* обязательные поля;
* корректность типов;
* допустимые значения;
* Category;
* ProductType;
* типоспецифичные ограничения;
* бизнес-ограничения;
* корректность связей.

Например:

```text
FoodProduct
Volume > 0
Unit задан
PetType задан
```

Если Product не проходит validation:

```text
Product не сохраняется
        ↓
RawProduct → Quarantine
```

---

# 39. Product lifecycle

Lifecycle товара и lifecycle обработки RawProduct разделены.

### ProductStatus

```text
Active
Inactive
Deleted
```

### ProcessingStatus

```text
Received
Processing
Processed
Failed
Quarantined
```

Например:

```text
Product = Active

Последний RawProduct:
ProcessingStatus = Failed
```

Это допустимое состояние.

Ошибка обновления товара не должна автоматически удалять уже существующий валидный Product.

---

# 40. Update vs Create

Алгоритм:

```text
RawProduct
     ↓
IdentityResolver
     ↓
Product найден?
   /        \
 Да         Нет
 ↓           ↓
Update      Create
```

При Update:

```text
Normalize
   ↓
Assemble
   ↓
Validate
   ↓
Compare
```

Если канонические данные не изменились:

```text
Skip
```

Если изменились:

```text
Update Product
       ↓
Outbox Event
```

---

# 41. Change Detection

FeedService должен определять, изменился ли товар.

Для этого могут использоваться:

* сравнение полей;
* `PayloadHash`;
* хэш канонической модели;
* комбинация подходов.

На уровне MVP достаточно сравнения значимых канонических полей.

Если товар не изменился, лишнее событие не создаётся.

---

# 42. Удаление товаров

Исчезновение товара из очередного импорта **не означает автоматическое удаление**.

Причины:

* источник может прислать неполную выгрузку;
* временная ошибка интеграции;
* товар мог отсутствовать только в конкретной выборке;
* нельзя удалять данные из-за технической ошибки источника.

Удаление выполняется отдельной операцией:

```text
DELETE /products/{id}
```

или отдельным application command.

Конкретная стратегия:

* soft delete;
* physical delete;

может быть определена отдельно.

Главный принцип:

> Import никогда не удаляет Product автоматически.

---

# 43. Idempotency

Повторная обработка одного и того же источника не должна создавать дубликаты.

Например:

```text
OneC + 123456
```

при повторной загрузке должен найти тот же:

```text
ProductExternalIdentifier
```

и обновить существующий Product.

Для этого используются:

* `(Source, ExternalId)`;
* PayloadHash;
* уникальные ограничения БД;
* idempotent application commands.

---

# 44. Import Job

Импорт является отдельным процессом.

Можно выделить:

```text
ImportJob
```

который содержит:

```text
Id
Source
StartedAt
FinishedAt
Status

Total
Processed
Created
Updated
Skipped
Failed
Quarantined
```

Это позволяет видеть результат загрузки:

```text
1000 received
950 processed
30 updated
15 created
5 quarantined
```

---

# 45. Частичный импорт

Ошибка одного товара не должна останавливать весь импорт.

Например:

```text
Batch 500 товаров

499 → успешно
1   → quarantine
```

Batch считается обработанным.

Техническая ошибка источника или БД может остановить batch и запустить retry.

---

# 46. Разделение ошибок

Ошибки разделяются по этапам.

### SourceAccessError

Ошибка доступа к источнику.

### ParseError

Невозможно разобрать формат.

### MappingError

Невозможно преобразовать DTO в RawProduct.

### TypeResolutionError

Не удалось определить тип.

### NormalizationError

Не удалось нормализовать данные.

### ValidationError

Product не прошёл бизнес-валидацию.

### PersistenceError

Ошибка сохранения.

### ExportError

Ошибка формирования документа/экспорта.

### DeliveryError

Ошибка доставки потребителю.

Также ошибки делятся на:

```text
Transient
Permanent
```

Transient ошибки могут повторяться автоматически.

Permanent ошибки переводят объект в Failed/Quarantined и требуют исправления данных или правил.

---

# 47. Повторная обработка

RawProduct хранится таким образом, чтобы его можно было обработать повторно.

Например:

```text
RawProduct
     ↓
Normalizer v1
     ↓
Failed
```

После изменения правила:

```text
Normalizer v2
     ↓
Reprocess
     ↓
Product
```

Для этого сохраняется:

```text
NormalizerVersion
```

Механизм массового reprocessing может быть реализован отдельной application-командой/job.

На первом этапе достаточно заложить такую возможность в архитектуру.

---

# 48. Repository

Repository отвечает за работу с канонической моделью.

```csharp
IProductRepository
```

Например:

```text
GetById()
GetByExternalIdentifier()
Create()
Update()
Delete()
```

Repository не должен содержать:

* правила нормализации;
* правила определения типа;
* правила identity resolution;
* логику формирования экспортов.

---

# 49. PostgreSQL

Предварительная СУБД:

**PostgreSQL**

Причины:

* структурированная модель;
* TPH;
* связи;
* ограничения;
* индексы;
* иерархия категорий;
* транзакции;
* JSONB для RawProduct;
* возможность использовать JSONB для отдельных гибких атрибутов.

Принцип:

> JSONB используется там, где данные действительно являются гибкими или исходными. Канонические ключевые поля Product остаются типизированными колонками.

---

# 50. Transactional Outbox

Для надёжной публикации изменений используется паттерн:

**Transactional Outbox.**

Изменение Product и создание OutboxMessage выполняются в одной транзакции.

```text
Transaction
│
├── UPDATE Product
│
└── INSERT OutboxMessage
        ↓
    COMMIT
        ↓
Outbox Publisher
        ↓
Consumer
```

Если приложение упало после изменения БД, но до публикации события, OutboxMessage останется и будет опубликован позже.

---

# 51. OutboxMessage

Пример:

```text
OutboxMessage
────────────────────────
Id
EventType

AggregateType
AggregateId

Payload

CreatedAt
PublishedAt

RetryCount
Status
```

Возможные события:

```text
ProductCreated
ProductUpdated
ProductDeleted
```

В будущем:

```text
ProductCategoryChanged
ProductPriceChanged
...
```

События должны содержать идентификатор канонического Product.

При необходимости в payload могут передаваться данные изменения.

---

# 52. Формирование документов и экспортов

FeedService отвечает не только за API, но и за формирование документов для потребителей.

Канонический Product:

```text
Product
   ↓
ExportService
   ↓
Consumer-specific document
```

Например:

```text
Product
   ↓
Mindbox Exporter
   ↓
XML / JSON

Product
   ↓
Analytics Exporter
   ↓
CSV

Product
   ↓
Web Exporter
   ↓
JSON
```

---

# 53. ExportService

Экспорт выделяется в отдельную ответственность.

```text
IExportService
```

Он отвечает за:

* выбор данных;
* формирование набора товаров;
* применение export profile;
* вызов нужного formatter;
* создание документа;
* сохранение результата/job;
* передачу потребителю.

---

# 54. Export Strategy

Для разных форматов используется Strategy.

```text
IExportFormatter
```

Например:

```text
MindboxXmlExporter
WebJsonExporter
AnalyticsCsvExporter
```

Каждый exporter знает только контракт своего потребителя.

Например:

```text
Canonical Product
       ↓
MindboxXmlExporter
       ↓
Mindbox XML
```

При этом канонический Product не знает о формате Mindbox.

---

# 55. Export Profile

Для сложных экспортов может использоваться профиль:

```text
ExportProfile
```

Он определяет:

* какие поля выгружать;
* какие типы товаров выгружать;
* формат;
* правила маппинга;
* дополнительные технические поля;
* ограничения выборки.

Например:

```text
MindboxProductFeed
    Product fields
    Category
    Brand
    Price
    Picture
    ExternalId
```

Это позволяет создавать разные представления одной канонической модели.

---

# 56. Экспорт не должен менять Product

Критически важно:

```text
Product
  ↓
Export
```

а не:

```text
Product
  ↓
изменили Product специально под Mindbox
```

Потребительская специфика должна находиться в Export layer.

---

# 57. Pagination в API

API FeedService должен поддерживать пагинацию для больших коллекций.

Например:

```text
GET /products?page=1&pageSize=100
```

или в будущем cursor-based:

```text
GET /products?cursor=...
```

Конкретный механизм может быть выбран после анализа потребителей.

---

# 58. Observability

FeedService должен иметь техническую наблюдаемость.

Минимально:

* структурированные логи;
* correlation/import ID;
* количество полученных товаров;
* количество созданных;
* количество обновлённых;
* количество пропущенных;
* количество ошибок;
* количество quarantine;
* время обработки;
* ошибки по этапам;
* retry count.

Пример:

```text
Import #12345

Received:    10000
Processed:    9975
Created:        300
Updated:       1200
Skipped:       8475
Quarantine:      25
Duration:     02:13
```

---

# 59. Audit

Для критичных операций желательно сохранять:

* кто/что запустил импорт;
* источник;
* время;
* результат;
* идентификатор job;
* изменения;
* операции удаления.

Это особенно важно для ручного удаления и повторной обработки.

---

# 60. Расширяемость

Добавление нового источника:

```text
NewSourceConnector
NewSourceParser
NewSourceMapper
```

не должно требовать изменения Domain.

Добавление нового типа:

```text
NewProductType
NewNormalizer
```

не должно ломать существующие Normalizer-ы.

Добавление нового потребителя:

```text
NewExporter
```

не должно менять Product.

Таким образом:

```text
Source changes
    ↓
Infrastructure

Product type changes
    ↓
Domain + Normalizer

Consumer changes
    ↓
Export layer
```

---

# 61. Используемые архитектурные паттерны

В проекте предполагается использование следующих паттернов.

### Adapter

Для Connector-ов и внешних систем.

```text
1С API
 ↓
OneCConnector
```

### Strategy

Для нормализаторов:

```text
FoodNormalizer
ToyNormalizer
...
```

и экспортёров:

```text
MindboxExporter
WebExporter
...
```

### Factory / Registry

Для выбора нужного Normalizer по ProductType.

### Repository

Для доступа к канонической модели.

### Transactional Outbox

Для надёжной публикации событий.

### Pipeline

Для последовательной обработки:

```text
Receive
→ Parse
→ Map
→ Resolve
→ Normalize
→ Assemble
→ Validate
→ Persist
→ Publish
```

---

# 62. Структура проекта

Предварительная структура:

```text
FeedService
│
├── FeedService.Api
│   ├── Controllers
│   ├── Requests
│   └── Responses
│
├── FeedService.Application
│   ├── Imports
│   ├── Products
│   ├── Identity
│   ├── Normalization
│   ├── Assembly
│   ├── Exports
│   ├── Interfaces
│   └── DTO
│
├── FeedService.Domain
│   ├── Products
│   ├── Categories
│   ├── ValueObjects
│   ├── Enums
│   └── Events
│
└── FeedService.Infrastructure
    ├── Persistence
    ├── Connectors
    ├── Parsers
    ├── Mappers
    ├── Normalizers
    ├── Exporters
    ├── Outbox
    └── ExternalServices
```

---

# 63. Основной сценарий импорта

Полный сценарий:

```text
1. Создаётся ImportJob
        ↓
2. Connector получает страницу источника
        ↓
3. Parser преобразует данные в Source DTO
        ↓
4. Mapper создаёт RawProduct
        ↓
5. RawProduct сохраняется
        ↓
6. TypeResolver определяет ProductType
        ↓
7. Выбирается Normalizer
        ↓
8. Normalizer создаёт NormalizedProduct
        ↓
9. IdentityResolver ищет существующий Product
        ↓
10. ProductAssembler собирает каноническую модель
        ↓
11. Validator проверяет Product
        ↓
12. Repository создаёт/обновляет Product
        ↓
13. ExternalIdentifier создаётся/обновляется
        ↓
14. OutboxMessage создаётся в той же транзакции
        ↓
15. Следующий товар
```

---

# 64. Сценарий ошибки

```text
Source
 ↓
RawProduct
 ↓
TypeResolver
 ↓
NormalizationError
 ↓
Quarantine
```

При этом остальные товары продолжают обрабатываться.

Если ошибка техническая:

```text
Connector
 ↓
Transient error
 ↓
Retry
```

Если retry исчерпан:

```text
ImportJob
 ↓
Failed
```

---

# 65. Сценарий обновления товара

```text
1С
 ↓
ExternalId = 123
 ↓
ProductExternalIdentifier
 ↓
Product #456
 ↓
Normalize
 ↓
Assemble
 ↓
Compare
 ↓
Changed
 ↓
UPDATE Product
 ↓
INSERT OutboxMessage
```

Если изменений нет:

```text
Skip
```

---

# 66. Сценарий нового товара

```text
1С
 ↓
ExternalId = 999
 ↓
ProductExternalIdentifier не найден
 ↓
IdentityResolver → Create
 ↓
Normalize
 ↓
Assemble
 ↓
Validate
 ↓
INSERT Product
 ↓
INSERT ExternalIdentifier
 ↓
INSERT OutboxMessage
```

---

# 67. Сценарий плохого товара

```text
1С
 ↓
RawProduct
 ↓
TypeResolver
 ↓
Не удалось определить тип
 ↓
Quarantine
```

Канонический Product не создаётся.

После появления нового правила:

```text
RawProduct
 ↓
TypeResolver v2
 ↓
FoodNormalizer
 ↓
Product
```

---

# 68. Сценарий нескольких источников

Например:

```text
1С
 ├── SKU
 ├── Name
 ├── Price
 └── Category

Web
 ├── Description
 ├── PictureUrl
 └── Url
```

Оба источника проходят собственный pipeline:

```text
1С → RawProduct → Normalize
Web → RawProduct → Normalize
```

После чего:

```text
NormalizedProduct
       ↓
ProductAssembler
       ↓
Canonical Product
```

---

# 69. Сценарий экспорта

```text
Product
   ↓
ExportService
   ↓
ExportProfile
   ↓
Exporter
   ↓
Document
   ↓
Consumer
```

Например:

```text
Product
 ↓
MindboxExportProfile
 ↓
MindboxXmlExporter
 ↓
products.xml
 ↓
Mindbox
```

---

# 70. Что считается готовым для MVP

MVP не обязан содержать все возможные типы и источники.

Минимальный вертикальный срез:

```text
1С
 ↓
Connector
 ↓
Parser
 ↓
Mapper
 ↓
RawProduct
 ↓
TypeResolver
 ↓
FoodNormalizer
 ↓
ProductAssembler
 ↓
Validator
 ↓
Product
 ↓
PostgreSQL
 ↓
Outbox
```

После этого можно добавлять:

* ToyNormalizer;
* Web source;
* несколько источников для одного товара;
* экспорты;
* дополнительные ProductType;
* reprocessing;
* дополнительные правила identity resolution.

---

# 71. Что необходимо определить до реализации конкретной бизнес-логики

Архитектурный каркас уже позволяет начинать разработку.

При этом следующие вещи должны быть определены на реальных данных:

### Identity

* какие идентификаторы реально приходят;
* какие товары совпадают между источниками;
* можно ли использовать SKU;
* какие дополнительные признаки нужны.

### TypeResolver

* какие категории существуют;
* какие признаки надёжны;
* как определяется ProductType.

### Normalizers

* реальные форматы названий;
* единицы измерения;
* значения атрибутов;
* варианты написания;
* исключения.

### Source of Truth

* какой источник является владельцем конкретного поля;
* как разрешаются конфликты.

Это **не блокирует архитектурный каркас**, но определяет конкретную реализацию ingestion.

---

# 72. Предварительные ADR

Перед активной реализацией фиксируются следующие архитектурные решения.

## ADR-001 — Product Identity

```text
Product.Id = внутренний GUID.

ExternalIdentifier =
(Source, ExternalId).

SKU не является универсальным identity key.
```

## ADR-002 — TPH

```text
Все ProductType хранятся в одной таблице Products.
```

## ADR-003 — RawProduct

```text
Исходные данные сохраняются в PostgreSQL JSONB.
RawProduct хранится отдельно от Product.
```

## ADR-004 — Source of Truth

```text
Источник определяется для каждого канонического поля.
Объединение выполняет ProductAssembler.
```

## ADR-005 — Product Lifecycle

```text
Product:
Active / Inactive / Deleted

Processing:
Received / Processing / Processed / Failed / Quarantined
```

## ADR-006 — Outbox

```text
Изменение Product и создание OutboxMessage
происходят в одной транзакции.
```

---

# 73. План первой реализации

Разработка начинается не со всех бизнес-правил сразу, а с вертикального среза.

### Этап 1. Каркас

Создать:

```text
Domain
Application
Infrastructure
API
```

и базовые интерфейсы:

```text
IProductRepository
IProductNormalizer
IProductTypeResolver
IProductMapper
IProductAssembler
IConnector
IParser
IExportFormatter
```

### Этап 2. Persistence

Создать:

```text
Products
Categories
ProductExternalIdentifiers
RawProducts
OutboxMessages
ImportJobs
```

и EF Core configurations.

### Этап 3. 1С

Реализовать:

```text
OneCConnector
OneCParser
OneCMapper
```

### Этап 4. Первый ProductType

Реализовать:

```text
TypeResolver
FoodNormalizer
```

на основании реальных данных.

### Этап 5. Import Pipeline

Реализовать полный путь:

```text
1С
→ RawProduct
→ Normalize
→ Assemble
→ Validate
→ Product
```

### Этап 6. Outbox

Добавить публикацию:

```text
ProductCreated
ProductUpdated
```

### Этап 7. Export

После появления канонического Product реализовать первый consumer-specific export.

---

# 74. Что сознательно оставлено расширяемым

На текущем этапе намеренно не фиксируются окончательно:

* полный список ProductType;
* полный набор атрибутов каждого типа;
* окончательные правила TypeResolver;
* окончательные regex/парсеры;
* полный механизм identity matching между источниками;
* окончательный механизм cursor/page pagination для каждого источника;
* TTL RawProduct;
* окончательная стратегия soft/hard delete;
* полный набор экспортов;
* конкретный транспорт событий;
* конкретный брокер сообщений.

Это не архитектурные пробелы, а параметры, которые должны определяться после анализа реальных данных и требований потребителей.

---

# 75. Итоговая концепция

FeedService является центральным слоем товарных данных:

```text
                  ┌──────────────┐
                  │      1С      │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │   Connector  │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │    Parser    │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │  RawProduct  │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ TypeResolver │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │  Normalizer  │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │   Assembler  │◄──── Web / другие источники
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │   Product    │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │  Validator   │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ PostgreSQL   │
                  └──────┬───────┘
                         │
              ┌──────────┴───────────┐
              │                      │
        ┌─────▼─────┐         ┌──────▼──────┐
        │   Outbox  │         │ ExportService│
        └─────┬─────┘         └──────┬──────┘
              │                      │
              ↓                 ┌────┼────┐
          Consumers             ↓    ↓    ↓
                              CRM  Web Analytics
```

Главный принцип архитектуры:

> **FeedService не просто передаёт товары между системами. Он формирует единое каноническое представление товара, сохраняя происхождение данных, возможность повторной обработки и независимость от конкретных потребителей.**

При этом архитектура не пытается заранее решить все бизнес-правила. Фундамент фиксируется сейчас, а конкретные правила нормализации, определения типа и сопоставления товаров формируются на основании реальных данных.
