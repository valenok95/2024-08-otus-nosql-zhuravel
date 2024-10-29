Необходимо:

построить шардированный кластер из 3 кластерных нод( по 3 инстанса с репликацией) и с кластером конфига(3 инстанса);
добавить балансировку, нагрузить данными, выбрать хороший ключ шардирования, посмотреть как данные перебалансируются между шардами;
поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло.
настроить аутентификацию и многоролевой доступ;

запускаем из текущей директории docker-compose up содержащий 10 инстансов (3 конфига, по 3 инстанса для обоих шардов и монгос)

//Сконфигуриуем реплика сет для конфиг сервера.
mongosh --port 40001

rs.initiate({
...  "_id": "config-replica-set",
...  members : [
...  {"_id": 0, "host": "mongo-configsvr-1:40001"},
...  {"_id": 1, "host": "mongo-configsvr-2:40002"},
...  {"_id": 2, "host": "mongo-configsvr-3:40003" }
...  ]
... })

//Проверяем что реплики созданы и выбран PRIMARY
rs.status()

exit

//Сконфигуриуем реплика сет для первого шарда.
mongosh --port 40011

rs.initiate({
...  "_id": "shard-replica-set-1",
...  members : [
...  {"_id": 0, "host": "mongo-shard-1-rs-1:40011"},
...  {"_id": 1, "host": "mongo-shard-1-rs-2:40012"},
...  {"_id": 2, "host": "mongo-shard-1-rs-3:40013" }
...  ]
... })

//Проверяем что реплики созданы и выбран PRIMARY
rs.status()

exit

//Сконфигуриуем реплика сет для второго шарда.

mongosh --port 40021

rs.initiate({
...  "_id": "shard-replica-set-2",
...  members : [
...  {"_id": 0, "host": "mongo-shard-2-rs-1:40021"},
...  {"_id": 1, "host": "mongo-shard-2-rs-2:40022"},
...  {"_id": 2, "host": "mongo-shard-2-rs-3:40023" }
...  ]
... })

//Проверяем что реплики созданы и выбран PRIMARY
rs.status()

exit

//Заходим на монгос и добавляем шарды.

mongosh --port 40100

sh.addShard("shard-replica-set-1/mongo-shard-1-rs-1:40011,mongo-shard-1-rs-2:40012,mongo-shard-1-rs-3:40013")
sh.addShard("shard-replica-set-2/mongo-shard-2-rs-1:40021,mongo-shard-2-rs-2:40022,mongo-shard-2-rs-3:40023")

sh.status() // смотрим, что шарды добавлены.

use bank  //переключаемся в базу bank
// наполняем данными коллекцию клиентов банка
 for (var i=0; i<2000000; i++) { db.clients.insert({name: "user-"+i, amount: Math.random()*100, guid:crypto.randomUUID()}) } 
// создаем индекс по полю guid (хорошая селективность, уникальность, равномерное распределение)
db.clients.createIndex({guid:1})

// шардируем коллекцию по созданному индексу guid
use admin

db.runCommand({shardCollection: "bank.clients", key: {guid: 1}})

// Уменьшаем размер чанков для распределения.
use config

db.settings.updateOne(
 { _id: "chunksize" },
 { $set: { _id: "chunksize", value: 1 } },
 { upsert: true }
)
// посмотрим распределение по шардам
use bank 
db.clients.getShardDistribution()
// наблюдаем копирование данных на шард shard-replica-set-1 , скопирована половина (миллион) записей. Однако на исходном шарде по прежнему 2 миллиона. (count больше истины на миллион)
Shard shard-replica-set-1 at shard-replica-set-1/mongo-shard-1-rs-1:40011,mongo-shard-1-rs-2:40012,mongo-shard-1-rs-3:40013
{
  data: '101.39MiB',
  docs: 989598,
  chunks: 101,
  'estimated data per chunk': '1MiB',
  'estimated docs per chunk': 9798
}
---
Shard shard-replica-set-2 at shard-replica-set-2/mongo-shard-2-rs-1:40021,mongo-shard-2-rs-2:40022,mongo-shard-2-rs-3:40023
{
  data: '204.92MiB',
  docs: 2000000,
  chunks: 1,
  'estimated data per chunk': '204.92MiB',
  'estimated docs per chunk': 2000000
}
---
Totals
{
  data: '306.31MiB',
  docs: 2989598,
  chunks: 102,
  'Shard shard-replica-set-1': [
    '33.1 % data',
    '33.1 % docs in cluster',
    '107B avg obj size on shard'
  ],
  'Shard shard-replica-set-2': [
    '66.89 % data',
    '66.89 % docs in cluster',
    '107B avg obj size on shard'
  ]
}

exit

// Уроним инстансы primary для config-server и обоих шардов:
rs.status() // находим primary для каждого реплика сета  stateStr: 'PRIMARY'
'mongo-configsvr-3:40003' // primary конфигов
'mongo-shard-1-rs-3:40013' // primary первого шарда
'mongo-shard-2-rs-1:40021' // primary второго шарда

docker stop mongo-configsvr-3
docker stop mongo-shard-1-rs-3
docker stop mongo-shard-2-rs-1
// погасили, смотрим статус реплика сетов, лидеры переизбраны.
rs.status()
 // видим в массиве members индикатор здоровья отключенной реплики:
 name: 'mongo-shard-2-rs-1:40021'
      health: 0
exit
// проверим статус шардирования, обнаруживаем что данные распределились равномерно 50 на 50 между шардами. и сумма на двух шардах ровно 2 миллиона = количество записей в коллекции. БАЛАНСИРОВКА ДОСТИГНУТА!
Shard shard-replica-set-1 at shard-replica-set-1/mongo-shard-1-rs-1:40011,mongo-shard-1-rs-2:40012,mongo-shard-1-rs-3:40013
{
  data: '101.39MiB',
  docs: 989598,
  chunks: 1,
  'estimated data per chunk': '101.39MiB',
  'estimated docs per chunk': 989598
}
---
Shard shard-replica-set-2 at shard-replica-set-2/mongo-shard-2-rs-1:40021,mongo-shard-2-rs-2:40022,mongo-shard-2-rs-3:40023
{
  data: '103.52MiB',
  docs: 1010402,
  chunks: 1,
  'estimated data per chunk': '103.52MiB',
  'estimated docs per chunk': 1010402
}
---
Totals
{
  data: '204.92MiB',
  docs: 2000000,
  chunks: 2,
  'Shard shard-replica-set-1': [
    '49.47 % data',
    '49.47 % docs in cluster',
    '107B avg obj size on shard'
  ],
  'Shard shard-replica-set-2': [
    '50.52 % data',
    '50.52 % docs in cluster',
    '107B avg obj size on shard'
  ]
}
// при старте остановленных контейнеров - primary переизбран НЕ будет, они станут secondary.
// в случае отказа двух secondary работа сета продолжается, в случае отказа primary и secondary оставшаяся реплика не может стать лидером, так как недостаточно голосов. и replica // set становится недоступным. (не получается запросить данные с шарда) 
Encountered non-retryable error during query :: caused by :: Could not find host matching read preference { mode: "primary" } for set shard-replica-set-1
// Переходим к настройке ролевой модели
