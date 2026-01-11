1. Запуск контейнеров
bash
docker compose up -d --build
2. Инициализация MongoDB кластера
2.1 Конфигурационный сервер
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
2.2 Шард 1
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
2.3 Шард 2
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
2.4 Добавление шардов в маршрутизатор
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1ReplSet/shard1a:27018,shard1b:27018,shard1c:27018")
sh.addShard("shard2ReplSet/shard2a:27019,shard2b:27019,shard2c:27019")
EOF
3. Создание базы и настройка шардирования
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "_id": "hashed" })
EOF
4. Добавление тестовых данных
bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
for (let i = 0; i < 1500; i++) {
  db.helloDoc.insertOne({
    "name": "User_" + (i % 100),
    "age": Math.floor(Math.random() * 50) + 18,
    "timestamp": new Date(),
    "data": "Test data " + i
  })
}
print("Inserted 1500 documents")
EOF
5. Проверка работы
5.1 Основной статус
bash
curl http://localhost:8080/  # Должно быть "cache_enabled": true
5.2 Тест кеширования
bash
# Первый запрос (медленный, >1 сек)
time curl -s "http://localhost:8080/helloDoc/users" | head -5

# Второй запрос (быстрый, <100мс)в 
time curl -s "http://localhost:8080/helloDoc/users" | head -5
5.3 Проверка данных
bash
curl "http://localhost:8080/helloDoc/count"  # Должно быть 1500 документов