## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```
2. Инициализация реплика-сета конфигурационного сервера (3 ноды)
bash
docker compose exec -T configsvr1 mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "configsvr1:27017" },
    { _id: 1, host: "configsvr2:27017" },
    { _id: 2, host: "configsvr3:27017" }
  ]
})
EOF
3. Инициализация реплика-сета первого шарда (3 ноды)
bash
docker compose exec -T shard1a mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1ReplSet",
  members: [
    { _id: 0, host: "shard1a:27018", priority: 2 },
    { _id: 1, host: "shard1b:27018", priority: 1 },
    { _id: 2, host: "shard1c:27018", priority: 1 }
  ]
})
EOF
4. Инициализация реплика-сета второго шарда (3 ноды)
bash
docker compose exec -T shard2a mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2ReplSet",
  members: [
    { _id: 0, host: "shard2a:27019", priority: 2 },
    { _id: 1, host: "shard2b:27019", priority: 1 },
    { _id: 2, host: "shard2c:27019", priority: 1 }
  ]
})
EOF
5. Подключение шардов к маршрутизатору
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1ReplSet/shard1a:27018,shard1b:27018,shard1c:27018")
sh.addShard("shard2ReplSet/shard2a:27019,shard2b:27019,shard2c:27019")
EOF
6. Включение шардирования для базы данных
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
sh.enableSharding("somedb")
EOF
7. Создание коллекции и настройка шардирования по ключу
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
sh.shardCollection("somedb.helloDoc", { "_id": "hashed" })
EOF
8. Добавление тестовых данных (более 1000 документов)
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
for (let i = 0; i < 1500; i++) {
  db.helloDoc.insertOne({
    "name": "Document " + i,
    "value": i,
    "timestamp": new Date(),
    "data": "Some test data for document number " + i
  })
}
print("Inserted 1500 documents")
EOF
9. Проверка статуса репликации
bash
# Проверка статуса реплика-сета shard1
docker compose exec -T shard1a mongosh --port 27018 --quiet <<EOF
rs.status()
EOF

# Проверка статуса реплика-сета shard2
docker compose exec -T shard2a mongosh --port 27019 --quiet <<EOF
rs.status()
EOF
10. Проверка распределения данных по шардам
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.getShardDistribution()
EOF
11. Проверка количества документов
bash
# Общее количество документов через mongos
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

# Количество документов в каждом шарде
docker compose exec -T shard1a mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2a mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF