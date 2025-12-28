## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Шаги настройки
1. Инициализация реплика-сета конфигурационного сервера
Инициализирует конфигурационный сервер для хранения метаданных кластера.

bash
docker compose exec -T configsvr mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "configsvr:27017" }
  ]
})
EOF
2. Инициализация реплика-сета первого шарда
Создает первый шард для хранения данных.

bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1ReplSet",
  members: [
    { _id: 0, host: "shard1:27018" }
  ]
})
EOF
3. Инициализация реплика-сета второго шарда
Создает второй шард для распределения данных.

bash
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2ReplSet",
  members: [
    { _id: 0, host: "shard2:27019" }
  ]
})
EOF
4. Подключение шардов к маршрутизатору
Добавляет шарды в кластер через mongos.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1ReplSet/shard1:27018")
sh.addShard("shard2ReplSet/shard2:27019")
EOF
5. Включение шардирования для базы данных
Активирует шардирование для указанной базы данных.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
sh.enableSharding("somedb")
EOF
6. Создание коллекции и настройка шардирования
Создает коллекцию и настраивает хэшированное шардирование по полю _id.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
sh.shardCollection("somedb.helloDoc", { "_id": "hashed" })
EOF
7. Проверка статуса шардирования
Отображает текущее состояние кластера.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
sh.status()
EOF
8. Добавление тестовых данных
Вставляет 1000 тестовых документов для проверки распределения.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
for (let i = 0; i < 1000; i++) {
  db.helloDoc.insertOne({
    "name": "Document " + i,
    "value": i,
    "timestamp": new Date()
  })
}
EOF
9. Проверка распределения данных
Показывает как данные распределены между шардами.

bash
docker compose exec -T mongos mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.getShardDistribution()
EOF