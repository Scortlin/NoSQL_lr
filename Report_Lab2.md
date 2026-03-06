# Отчет по Практической работе №2. Изучение и применение различных типов NoSQL баз данных на бизнес-кейсах (Polyglot Persistence)

**Студент:** [Быков Владимир Валерьевич]

**Группа:**[БД-251м, Магистратура "Бизнес-информатика"]

**Вариант:** 7

---

## 1. Введение. Описание бизнес-кейса и архитектуры

### 1.1 Бизнес-кейс
Данная работа посвящена разработке концепта аналитической платформы для стримингового сервиса, специализирующегося на контенте 2010-х годов. Цель — проанализировать тренды качества кино за это десятилетие. Использование одной реляционной СУБД для всех задач неэффективно, поэтому применяется подход **Polyglot Persistence**:
*   **MongoDB (Document Store).** Хранение каталога контента и профилей пользователей. Гибкая JSON-схема позволяет легко добавлять новые атрибуты (новые форматы видео, типы подписок).
*   **Cassandra (Wide-Column).** Логирование просмотров (Clickstream). Генерируется огромный непрерывный поток событий (паузы, перемотки). Архитектура Cassandra обеспечивает высочайшую скорость записи (High Write Throughput).
*   **GraphDB (Graph/RDF).** Построение рекомендательной системы. Графовая БД эффективно обрабатывает сложные многоуровневые связи (Пользователь -> лайкнул -> Фильм -> имеет жанр -> Схожий фильм).


## 2. Развертывание инфраструктуры

Запуск сервисов выполняется на виртуальной машине через терминал:

```bash
# 1. Запуск MongoDB
cd ~/Downloads/dba/nonrel/mongo 
sudo docker compose down && sudo docker compose up -d

# 2. Запуск Cassandra
cd ~/Downloads/dba/nonrel/cassandra
sudo docker compose down && sudo docker compose up -d

# 3. Запуск GraphDB
cd ~/Downloads/dba/nonrel/graphdb
sudo docker compose down && sudo docker compose up -d
```

---

**Результат запуска контейнеров**
![Скриншот запуска контейнеров](screenshots/photo1.png)
![Скриншот активных контейнеров](screenshots/photo2.png)
![Скриншот GraphDB](screenshots/photo3.png)
## 3. Генерация и проверка баз данных (Python + Faker)

Для выполнения аналитических задач необходимо наполнить СУБД тестовыми данными. В скрипт ниже **встроена проверка успешности записи данных и пояснение архитектурных особенностей**.

В **JupyterLab** создаем ноутбук и выполняем следующий код:

```python
!pip install pymongo cassandra-driver Faker matplotlib pandas SPARQLWrapper

from pymongo import MongoClient, ASCENDING
from cassandra.cluster import Cluster
from faker import Faker
from SPARQLWrapper import SPARQLWrapper, JSON
import random
import matplotlib.pyplot as plt
import pandas as pd

fake = Faker()

# Подключение к MongoDB
mongo_client = MongoClient("mongodb://root:abc123!@localhost:27017/")
mongo_db = mongo_client["cinema_db"]
users_col = mongo_db["users"]
movies_col = mongo_db["movies"]

users_col.drop()
movies_col.drop()

# Генерация пользователей
for i in range(100):
    user = {
        "user_id": i,
        "name": fake.name(),
        "email": fake.unique.email(),
        "subscription": random.choice(["basic", "premium", "ultra"])
    }
    users_col.insert_one(user)

# Создание индекса по email
users_col.create_index([("email", ASCENDING)], unique=True)
print("Индекс по email создан")

# Генерация фильмов
genres = ["Drama", "Comedy", "Action", "Sci-Fi", "Thriller"]
directors = ["Christopher Nolan", "Denis Villeneuve", "Greta Gerwig", "Bong Joon-ho", "Leonardo DiCaprio", "Amy Adams"]
actors = ["Leonardo DiCaprio", "Amy Adams", "Ryan Gosling", "Emma Stone", "Greta Gerwig", "Bong Joon-ho"]

for i in range(1, 51):
    movie = {
        "movie_id": i,
        "title": fake.catch_phrase().replace('"', ''),
        "release_year": random.randint(2010, 2019),
        "genre": random.choice(genres),
        "director": random.choice(directors),
        "actors": random.sample(actors, random.randint(2, 3)),
        "rating": round(random.uniform(5.0, 9.0), 1)
    }
    movies_col.insert_one(movie)

mongo_count = movies_col.count_documents({})
sample_movie = movies_col.find_one()
print("--- 1. Проверка MongoDB (Document Store) ---")
print(f"[ДАННЫЕ]: Всего документов: {mongo_count}")
print(f"[ДАННЫЕ]: Пример документа: {sample_movie}")

# Подключение к Cassandra
cass_cluster = Cluster(['127.0.0.1'], port=29042, protocol_version=4)
cass_session = cass_cluster.connect()

cass_session.execute("""
    CREATE KEYSPACE IF NOT EXISTS viewing_history
    WITH replication = {'class':'SimpleStrategy', 'replication_factor':1}
""")
cass_session.set_keyspace('viewing_history')

cass_session.execute("DROP TABLE IF EXISTS user_views")
cass_session.execute("""
    CREATE TABLE user_views (
        user_id int,
        movie_id int,
        view_timestamp timestamp,
        watch_duration_min int,
        PRIMARY KEY (user_id, view_timestamp, movie_id)
    ) WITH CLUSTERING ORDER BY (view_timestamp DESC)
""")

for _ in range(500):
    cass_session.execute(
        "INSERT INTO user_views (user_id, movie_id, view_timestamp, watch_duration_min) VALUES (%s, %s, %s, %s)",
        (random.randint(0, 99), random.randint(1, 50), fake.date_time_between(start_date='-1y'), random.randint(5, 240))
    )

print("Cassandra: Загружено 500 логов просмотров")
```

**Действие в GraphDB:**
1. Открыть `http://localhost:17200`.
2. Создать репозиторий `movies_repo`.
3. Перейти в *Import -> RDF*, загрузить `movies_graph.ttl` и импортировать.

---

## 4. Выполнение Варианта 7

### 4.1. Задание 1. MongoDB. Использовать индексацию. Создать индекс по полю email в users и объяснить explain().

#### 4.1.1. Создание индекса

Как было показано в шаге 3.1, создан уникальный индекс по полю email в коллекции users.
```python
users_col.create_index([("email", ASCENDING)], unique=True)
```
Этот индекс гарантирует уникальность email-адресов на уровне базы данных, предотвращая создание дублирующихся аккаунтов.


#### 4.1.2. Анализ запроса с помощью explain()

```python
# Выполнение explain() для поиска по email
sample_email = users_col.find_one()["email"]
explain_result = users_col.find({"email": sample_email}).explain()

print("\n=== РЕЗУЛЬТАТ explain() ===")
print(f"Просканировано документов: {explain_result['executionStats']['totalDocsExamined']}")
print(f"Время выполнения: {explain_result['executionStats']['executionTimeMillis']} мс")
```



### 4.2. Задание 2. (GraphDB / SPARQL)

#### 4.2.1. Запрос 1: Фильмы 2010-х годов с рейтингом > 7.0

```python
# SPARQL запрос 1: Фильмы 2010-х с рейтингом > 7.0
sparql = SPARQLWrapper("http://localhost:17200/repositories/movies_repo")

query1 = """
PREFIX ex: <http://example.org/cinema#>

SELECT ?title ?year ?rating
WHERE {
    ?movie a ex:Movie ;
           ex:title ?title ;
           ex:release_year ?year ;
           ex:rating ?rating .
    FILTER (?year >= 2010 && ?year <= 2019 && ?rating > 7.0)
}
ORDER BY DESC(?rating)
"""

sparql.setQuery(query1)
sparql.setReturnFormat(JSON)

print("\n=== ЗАПРОС 1: Фильмы 2010-х с рейтингом > 7.0 ===")
results = sparql.query().convert()

data_list = []
for result in results["results"]["bindings"]:
    data_list.append({
        "Title": result["title"]["value"],
        "Year": int(result["year"]["value"]),
        "Rating": float(result["rating"]["value"])
    })

df1 = pd.DataFrame(data_list)
print(df1.to_string(index=False))
```
![Запрос 1](screenshots/photo4.png)

Запрос успешно отфильтровал фильмы по десятилетию и рейтингу, отсортировав их от лучших к худшим.


#### 4.2.2. Запрос 2: Фильмы с одинаковыми актерами и режиссерами

```python
# SPARQL запрос 2: Фильмы с одинаковыми актерами и режиссерами
query2 = """
PREFIX ex: <http://example.org/cinema#>

SELECT DISTINCT ?title ?directorName
WHERE {
    ?movie a ex:Movie ;
           ex:title ?title ;
           ex:hasDirector ?director .
    ?director ex:name ?directorName .
    ?actor a ex:Actor ;
           ex:name ?directorName .
    ?movie ex:hasActor ?actor .
}
"""

sparql.setQuery(query2)
sparql.setReturnFormat(JSON)

print("\n=== ЗАПРОС 2: Фильмы с режиссером в роли актера ===")
results = sparql.query().convert()

data_list = []
for result in results["results"]["bindings"]:
    data_list.append({
        "Title": result["title"]["value"],
        "Director/Actor": result["directorName"]["value"]
    })

df2 = pd.DataFrame(data_list)
if len(df2) > 0:
    print(df2.to_string(index=False))
else:
    print("Нет фильмов, где режиссер также является актером")
```
Этот запрос ищет фильмы, в которых режиссер также указан в списке актеров.

![Запрос 2](screenshots/photo5.png)


### 4.3. Задание 3. Бизнес-аналитика (Тренд качества кино (по рейтингу) в 2010-х годах.)

**Бизнес-вопрос** Как изменялся средний рейтинг фильмов на протяжении 2010-х годов? Был ли какой-то тренд (рост, падение, стагнация)?

#### 4.3.1. Извлечение и агрегация данных

```python
# Агрегация среднего рейтинга по годам из MongoDB
pipeline = [
    {"$match": {"release_year": {"$gte": 2010, "$lte": 2019}}},
    {"$group": {
        "_id": "$release_year",
        "avg_rating": {"$avg": "$rating"},
        "count": {"$sum": 1}
    }},
    {"$sort": {"_id": 1}}
]

trend_data = list(movies_col.aggregate(pipeline))
df = pd.DataFrame(trend_data)
df.rename(columns={"_id": "year"}, inplace=True)

print("\n=== СРЕДНИЙ РЕЙТИНГ ПО ГОДАМ ===")
print(df.to_string(index=False))
```

**Результат агрегации**

![Скриншот результата агрегации](screenshots/photo6.png)

#### 4.3.2. Визуализация и анализ тренда

```python
# Построение графика
plt.figure(figsize=(10, 5))
plt.plot(df["year"], df["avg_rating"], marker='o', linestyle='-', linewidth=2, color='blue')
plt.axhline(y=df["avg_rating"].mean(), color='red', linestyle='--',
            label=f'Среднее за десятилетие: {df["avg_rating"].mean():.2f}')

plt.title("Тренд качества кино в 2010-х годах", fontsize=14)
plt.xlabel("Год", fontsize=12)
plt.ylabel("Средний рейтинг", fontsize=12)
plt.grid(True, linestyle='--', alpha=0.7)
plt.legend()
plt.xticks(df["year"])
plt.tight_layout()
plt.show()
```
**Результат визуализации**
![Скриншот результата визуализации](screenshots/photo7.png)

Общий тренд: Наблюдается положительная динамика среднего рейтинга фильмов на протяжении 2010-х годов. 
Несмотря на колебания, линия тренда показывает, что фильмы к концу десятилетия оценивались выше, чем в начале.
С 2014 года средний рейтинг фильмов был выше среднего рейтинга десятилетия (кроме 2016 года).


## 5. Итоговые выводы по Polyglot Persistence

В ходе работы через код Python и запросы доказана необходимость нескольких СУБД:
1. **MongoDB** обеспечила хранение каталога (гибкость к изменениям). Бессхемная структура позволила легко создать коллекции users и movies.
2. **Cassandra** доказала способность записывать телеметрию (логи просмотров) с минимальной задержкой.
3. **GraphDB** обеспечила извлечение инсайтов.  SPARQL-запросы позволили интуитивно находить сложные, многоуровневые связи (фильмы одного режиссера, режиссеры-актеры), которые в реляционной базе потребовали бы множества сложных JOIN-операций







