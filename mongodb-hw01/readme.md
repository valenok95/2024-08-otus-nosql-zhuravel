Необходимо:
установить MongoDB одним из способов: ВМ, докер;
заполнить данными;
написать несколько запросов на выборку и обновление данных


В рамках домашнего задания проделана сделующая работа:
1.создан файл docker-compose.yml (см в текущей папке)
2. Запущена MongoDB .
3. Для удобства используется клиент Mongo Compass со строкой подключения mongodb://admin:my_password@localhost:27017/?directConnection=true
4. Используется утилита mongosh. use test - для подключения к тестовой базе.
5. db.createCollection("cats") - создали коллекцию для хранения информации по кошкам.
6. наполняем данными  db.collection.insertMany([
    { name: "Bob", age: 5, weight: 4
    },
    { name: "Timoshka", age: 7, weight: 4
    },
    { name: "Murka", age: 8, weight: 7
    }
])
7. Обновили кота db.collection.updateOne({name:"Bob"},{$set:{"weight":5}})
8. Выборка по фильтру db.cats.find({weight:{ $gt:5}})
9. создали индекс по полю age db.cats.createIndex({age:1})
10. Команда проверки на попадание в индекс при выборке db.cats.find({age:7}).explain() - попадание в индекс подтверждено:  
winningPlan: {
      stage: 'FETCH',
      inputStage: {
        stage: 'IXSCAN',
        keyPattern: {
          age: 1
        },
        indexName: 'age_1' ....
11. убираем индекс db.cats.dropIndex("age_1")
12. исследуем запрос выборки теперь db.cats.find({age:7}).explain() - Теперь запрос не использует индексы:
winningPlan: {
      stage: 'COLLSCAN',
      filter: {
        age: {
          '$eq': 7
