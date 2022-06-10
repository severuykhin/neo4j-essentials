### Конфигурация для работы с графовой СУБД Neo4j

1. docker-compose file
2. Процедура Apoc - расширение функционала neo4j
- https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases/tag/4.2.0.2
- Скачиваем apoc-4.2.0.2-all.jar (актуально для версии Neo4j 4.2.*)
- Кладем в директорию plugins

Для взаимодействия используется язык - Cypher. Создан специально для взаимодействия с графами


### Создание ноды
Для создания ноды есть ключевое слово CREATE

```
CREATE (u:User:Human {id: 1, name: "Igor"})
# Создали ноду с лейблами User и Human и данными
# u - Алиас запроса - он не сохраняется в базу
# :User и :Human - лейблы ноды, для группировок нод в бд
# {id: 1, name: "Igor"} - данные ноды в формате json

CREATE (u:User {id: 2, name: "Chucky"})
# Создали ноду с лейблом User
```

### Поисковые запросы
Для поиска и получения нод используется клюевое слово MATCH
В нашем случае для получения ноды мы можем выполнить разные запросы

```
# Вернуть все ноды с лейблом User
MATCH (u:User) return u
# Вернутся Igor и Chucky
```

```
# Вернуть все ноды с лейблом Human
MATCH (h:Human) return h
# Вернутся только Igor
```

```
# Вернуть все ноды с лейблом User И Human
MATCH (n:User:Human) return n
# Вернутся только Igor
```

Уточняющие условия c использование WHERE clause
```
# Получить пользователя у которого ID равен 1
MATCH (u:User) WHERE u.id = 1 RETURN u

# Оговорка. У нод есть свой внутренний идентификатор - поле <id>. Искать по нему таким образом не получится.
# Для подобного сценария есть свой специфический запрос
MATCH (u:User) WHERE ID(u) = 0 RETURN u
# Вернется User Igor - так как у этой ноды <id> = 0
```  

Создание ноды сразу с отношениями
```
MATCH (u:User) WHERE ID(u) = 0
CREATE (s:Something {name: "Something1", id: apoc.create.uuid()})
CREATE (u)-[r:OWN]->(s)
RETURN s
# Для пользователя с ID = 0 мы создали некую сущность и связали их отношением под лейблом OWN ((u)-[r:OWN]->(s))
# Стрелкой помечены направления отношения
```

Для отношений можно задать свойства
```
MATCH (u:User) WHERE ID(u) = 0
CREATE (s:Something {name: "Something1", id: apoc.create.uuid()})
CREATE (u)-[r:OWN {created_at: date()}]->(s)
RETURN s
```

Получение конкрентной ноды со всеми связями
```
MATCH (a:User {name: 'Igor'})-[r]-(b) RETURN r, a, b
# Но такой запрос вернет только связи первого уровня
```


### Попробуем натянуть логику коммитов и реализовать сценарии, которые понадобтся нам в работе
Создадим первый коммит
```
# Каждый коммит указыавет на некое состояние в момент создания - tree_id
# Так же коммиту нужен автор
CREATE (c:Commit {
	author_id: "222",
	tree_id: "123-123-123", 
    id: apoc.create.uuid()
})
RETURN c;
```

Создадим следующий коммит, который будет указывать на первый коммит в качестве родительского
```
MATCH (c:Commit {id: "22abfab5-15a2-40d3-aa96-665c4bc9ffb9"})
CREATE (c2:Commit {
	id: apoc.create.uuid(),
    author_id: "222",
    tree_id: "123-123-124"
})
CREATE (c2)-[r:CHILD {created_at: date()}]->(c)
# Ищем некий предыдущий коммит
# Создаем новый
# Тут же указываем связь с лейблом CHILD от с2 до с - c2 child of c
```

Получим коммит с2 со связью
```
MATCH (c:Commit {id: "30f5c4db-a292-4933-b401-c2e26ac4d9e5"})-[r:CHILD]-(b) RETURN c, r, b
```

Добавим еще пару коммитов
```
MATCH (c1:Commit {id: "30f5c4db-a292-4933-b401-c2e26ac4d9e5"})
CREATE (c2:Commit {
	id: apoc.create.uuid(),
    author_id: "333",
    tree_id: "123-123-125"
})
CREATE (c2)-[r:CHILD {created_at: date()}]->(c1)
RETURN c2, r

MATCH (c1:Commit {id: "3d270386-dd98-4c2d-976d-5349c1d3ce89"})
CREATE (c2:Commit {
	id: apoc.create.uuid(),
    author_id: "333",
    tree_id: "123-123-125"
})
CREATE (c2)-[r:CHILD {created_at: date()}]->(c1)
RETURN c2, r
```

Получим всю ветку  
```
# Получить все ноды с лейблом Commit и проследовать по всем связям
MATCH path = (c:Commit {id: "76055848-e35a-478f-87f1-98427e50348d"})-[*]->(parent_c) return path
# Или получить все ноды с лейблом Commit и проследовать связям с конкретным лейблом
MATCH path = (c:Commit {id: "76055848-e35a-478f-87f1-98427e50348d"})-[:CHILD*]->(parent_c) return path
```


Добавим ветвление
Для этого нам понадобится создать такую ноду в графе, 
которая будет ссылаться на какую то другую ноду, на которую уже ссылается какая другая нода
```
MATCH (c1:Commit {id: "30f5c4db-a292-4933-b401-c2e26ac4d9e5"})
CREATE (c2:Commit {
	id: apoc.create.uuid(),
    author_id: "444",
    tree_id: "123-123-127"
})
CREATE (c2)-[r:CHILD {created_at: date()}]->(c1)
RETURN c2, r
```


Посмотрим вообще всю ветку коммитов   
```
MATCH path = (c:Commit)-[:CHILD*]->(parent_c) return path

# Результат немного не в том формате, который нас интересует
```

Поэтому скорвертируем ответ в древовидный json
```
MATCH path = (c:Commit)-[:CHILD*]->(parent_c)
WITH collect(path) as ps
CALL apoc.convert.toTree(ps) yield value
RETURN value

```


Попробуем найти LCA для двух коммитов
```
MATCH (c1) -[*0..]-> (lca) <-[*0..]- (c2) 
WHERE c1.id = "3200f261-68eb-4617-9085-b3b441947a8a" and c2.id = "76055848-e35a-478f-87f1-98427e50348d" 
RETURN lca

# Вернет самый ближайший общий коммит для с1 и с2 ("30f5c4db-a292-4933-b401-c2e26ac4d9e5") 
```

Сэмулируем работу мержа
Создадим коммит, который будет ссылаться на c1 и c2
```
MATCH (c1:Commit {id: "3200f261-68eb-4617-9085-b3b441947a8a"})
WITH c1
MATCH (c2:Commit {id: "76055848-e35a-478f-87f1-98427e50348d"})
CREATE (c3:Commit {
	id: apoc.create.uuid(),
    author_id: "444",
    tree_id: "123-123-128"
})
CREATE (c3)-[r1:CHILD {created_at: date()}]->(c1)
CREATE (c3)-[r2:CHILD {created_at: date()}]->(c2)
RETURN c3, r1, r2
```



Посмотреть как СУБД обрабатывает запрос
```
# Добавить ключевое слово EXPLAIN или PROFILE
PROFILE MATCH (n) RETURN n LIMIT 25
# или
EXPLAIN MATCH (n) RETURN n LIMIT 25
```


