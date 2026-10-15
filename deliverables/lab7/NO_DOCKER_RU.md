# Лабораторная №7 без локального Docker и WSL

Этот путь использует **MongoDB Community Server + MongoDB Compass на Windows** и **удалённую Cassandra/CQL-базу**. Архив с 768 записями остаётся тем же. Docker Desktop и Ubuntu на компьютере для этого пути не нужны.

## 1. MongoDB на Windows

1. Скачай MongoDB Community Server для Windows x64 с официальной страницы: https://www.mongodb.com/try/download/community . Выбери поддерживаемый стабильный выпуск и MSI-установщик.
2. Запусти MSI, выбери Complete и установку MongoDB как службы Windows (Install MongoD as a Service). Запомни имя службы; стандартное — MongoDB.
3. Установи Compass из установщика или отдельно: https://www.mongodb.com/try/download/compass .
4. Открой Compass, создай подключение по адресу `mongodb://localhost:27017` и подключись.
5. Создай базу `hospital_lab7` с коллекцией `patients` через Create database.
6. Открой коллекцию → Add Data → Import JSON or CSV file. Выбери `data/patients.json` из распакованного архива, формат JSON, выполни импорт.
7. Проверь число документов: **768**. Импортировать один и тот же набор повторно поверх коллекции не нужно.

Пункты интерфейса могут немного отличаться между версиями Compass. Команды ниже выполняются во встроенном MongoDB Shell/mongosh, открываемом из Compass. Если в установленной версии такого пункта нет, установи mongosh с https://www.mongodb.com/try/download/shell и подключись командой `mongosh "mongodb://localhost:27017"`.

```javascript
use hospital_lab7
db.version()
db.patients.createIndex({id: 1}, {unique: true})
db.patients.countDocuments({})
db.patients.find({}, {_id: 0}).sort({id: 1}).forEach(p => printjson(p))
db.patients.findOne({id: 3}, {_id: 0})
db.patients.findOne({id: 2}, {_id: 0})
db.patients.updateOne({id: 2}, {$set: {room: 215}})
db.patients.findOne({id: 2}, {_id: 0})
db.patients.deleteOne({id: 768})
db.patients.findOne({id: 768})
db.patients.countDocuments({})
db.patients.countDocuments({age: {$gt: 40}})
db.patients.find({age: {$gt: 40}}, {_id: 0}).sort({id: 1}).limit(5)
db.patients.find({}, {_id: 0}).sort({age: -1, id: 1}).limit(5)
db.patients.find({age: {$gt: 40}}, {_id: 0}).sort({age: -1, id: 1}).limit(5)
```

Выполняй команды по одной и делай снимки результатов. Ожидается: 768 до удаления; палата id=2 меняется 102 → 215; удаляется одна запись id=768; затем 767 документов; возраст >40 — 194 записи. Первые возраста при сортировке: 81, 72, 70, 69, 69.

В Documents можно отдельно показать Filter `{"age":{"$gt":40}}` и Sort `{"age":-1,"id":1}`.

## 2. Cassandra через облачный CQL-сервис

Apache Cassandra 4.1 из исходной работы ориентирована на Linux; для обычной нативной установки на Windows нет официального пути, аналогичного MSI MongoDB. Поэтому здесь используется удалённая база с CQL, например соответствующий тип базы в DataStax Astra DB: https://astra.datastax.com/ .

Сначала проверь, доступна ли в аккаунте **база с CQL** и CQL-консоль/подключение. Один только векторный Data API этого задания не заменяет. Доступность типов баз, тарифы и названия разделов зависят от текущих условий провайдера. Не создавай платный ресурс только ради следования инструкции. Если CQL-варианта нет, нужен другой доступный сервер Cassandra/CQL; набор данных и команды сохраняются.

1. Войди или зарегистрируйся в сервисе самостоятельно. Пароли и токены в чат присылать не нужно.
2. При создании CQL-базы задай keyspace `hospital_lab7` через интерфейс сервиса, если это доступно.
3. Дождись состояния готовности базы.
4. Открой CQL-консоль или рекомендованный провайдером CQL-клиент.
5. Выполни:

```sql
USE hospital_lab7;
CREATE TABLE IF NOT EXISTS patients (
    id int PRIMARY KEY,
    name text,
    age int,
    diagnosis text,
    room int
);
```

Облачный сервис может запрещать CREATE KEYSPACE из CQL и управлять репликацией самостоятельно. Тогда keyspace создаётся в панели, а учебная команда с SimpleStrategy/RF=1 не применяется.

## 3. Загрузить 768 записей через CQL-консоль

Для первой загрузки используй пустую учебную таблицу. Для полного повторения разрешена очистка **только своей учебной таблицы**:

```sql
TRUNCATE hospital_lab7.patients;
```

В папке [cql_browser](cql_browser) подготовлены восемь файлов:

- `01_insert_001_100.cql` — первые 100 записей;
- `02_insert_101_200.cql`;
- `03_insert_201_300.cql`;
- `04_insert_301_400.cql`;
- `05_insert_401_500.cql`;
- `06_insert_501_600.cql`;
- `07_insert_601_700.cql`;
- `08_insert_701_768.cql` — последние 68 записей.

Открой файл на GitHub через Raw или в Блокноте и выполни его содержимое в CQL-консоли. Затем переходи к следующему. Это отдельные INSERT-команды, не большой BATCH.

Возможность вставить сразу много команд зависит от конкретной веб-консоли. Если она принимает только один оператор, используй поддерживаемый провайдером CQL-клиент для исполнения файла; не склеивай все записи в один BATCH. Команда SOURCE из старой Docker-инструкции читает файл с машины клиента и сама по себе не загружает локальный файл в удалённую браузерную консоль.

После каждой части можно проверять:

```sql
SELECT COUNT(*) FROM hospital_lab7.patients;
```

Ожидаемые значения: 100, 200, 300, 400, 500, 600, 700, **768**. При повторном INSERT того же ID Cassandra выполняет upsert, поэтому счётчик не удваивается.

## 4. Выполнить задания Cassandra

После импорта всех частей:

```sql
USE hospital_lab7;
SELECT * FROM patients;
SELECT * FROM patients WHERE id = 3;
SELECT id, room FROM patients WHERE id = 2;
UPDATE patients SET room = 215 WHERE id = 2;
SELECT id, room FROM patients WHERE id = 2;
SELECT * FROM patients WHERE id = 768;
DELETE FROM patients WHERE id = 768;
SELECT * FROM patients WHERE id = 768;
SELECT COUNT(*) FROM patients;
SELECT * FROM patients WHERE id = 1;
SELECT id, name, age, diagnosis, room FROM patients WHERE id IN (1, 2, 3);
```

Должно остаться **767** строк. Для id=2 room=215. Для id=768 — пустой результат. Запросы по ID возвращают соответственно одну и три строки.

Если консоль показывает SELECT постранично, просмотри следующие страницы; для полного экспорта используй поддерживаемый клиентом экспорт. Команды PAGING OFF и SOURCE относятся к cqlsh и могут отсутствовать в другом клиенте.

## 5. Что показать преподавателю

1. Титульный лист: Табидулла Абылайхан, АЖ45, №7.
2. Compass: подключение, hospital_lab7, patients, исходные 768 записей.
3. MongoDB: поиск ID, изменение room, удаление, итог 767, возраст >40 — 194, сортировка.
4. CQL-консоль: keyspace, таблица, импорт, поиск, изменение, удаление, итог 767, запросы = и IN.
5. Данные и исходное описание из архива. Возраст и диабетическая метка реальные; имя-код и палата учебные.
6. Сравнение двух моделей и вывод.

Сделай свои скриншоты через Win+Shift+S. Исходный отчёт документирует уже проверенный запуск MongoDB 7.0.16 и Cassandra 4.1.7 в контейнерах. Для нового варианта измени описание среды, фактическую версию MongoDB, сведения об облачной CQL-базе, правила создания keyspace и иллюстрации. Не оставляй утверждение об одноузловом RF=1 для облака, где репликацией управляет провайдер.

Новый облачный запуск пока не проверен: регистрация, доступность CQL-базы, импорт и результаты должны быть подтверждены в твоём аккаунте. Исходные 13 проверок в архиве относятся к прежнему выполнению и автоматически на новую среду не распространяются.
