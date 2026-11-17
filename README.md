# Отчетик :)

Все в папке `app`. Запуск:

```sh
cd app
cp .env.example .env
docker compose up -d --build --wait
```

`test.sh`вызывает `/`, `/increment` и `/readyz`

## 4 балла

```sh
docker build -t hw3-app .
docker run -d --name hw3-app -p 127.0.0.1:8080:8000 -e GREETING="куку" hw3-app
./test.sh
# {"greeting": "куку", "count": 0, "hostname": "e106afd9d43f", "storage": "sqlite"}
# {"greeting": "куку", "count": 1, "hostname": "e106afd9d43f", "storage": "sqlite"}
# {"status": "ready", "storage": "sqlite"}
docker logs hw3-app
# 2026-11-17 20:20:55,839 listening port=8000 storage=sqlite
# 2026-11-17 20:20:57,812 http 172.17.0.1 "GET / HTTP/1.1" 200 -
# 2026-11-17 20:20:57,821 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
# 2026-11-17 20:20:57,828 http 172.17.0.1 "GET /readyz HTTP/1.1" 200 -
```

 тут я просто в -e передаю env гритинг и смотрю что все работает.

## 6 баллов

Тут в compose был `STORAGE: sqlite` и volume `counter-data:/data`. Порт и гритинг берутся из `.env`

```sh
docker compose up -d --build --wait
./test.sh
# {"greeting": "Hello RAINL", "count": 6, "hostname": "a38b920f83cd", "storage": "sqlite"}
# {"greeting": "Hello RAINL", "count": 7, "hostname": "a38b920f83cd", "storage": "sqlite"}
# {"status": "ready", "storage": "sqlite"}
docker compose down
docker compose up -d --wait
./test.sh
# {"greeting": "Hello RAINL", "count": 7, "hostname": "401ef590ff7a", "storage": "sqlite"}
# {"greeting": "Hello RAINL", "count": 8, "hostname": "401ef590ff7a", "storage": "sqlite"}
# {"status": "ready", "storage": "sqlite"}
```

Контейнер новый, а счетчик остался старым

## 8 баллов

Добавил redis с volume на `/data` и `--appendonly yes`. У app `STORAGE=redis` и `REDIS_URL=redis://redis:6379/0`,  открыт только порт нашего приложения. Healthcheck у обоих, app стартует ток когда redis прошел healthy штуку

```sh
curl -X POST 127.0.0.1:8080/increment
# {"greeting": "Hello RAINL", "count": 11, "hostname": "1bac01f7bb44", "storage": "redis"}
curl -X POST 127.0.0.1:8080/increment
# {"greeting": "Hello RAINL", "count": 12, "hostname": "1bac01f7bb44", "storage": "redis"}
docker compose down
docker compose up -d --wait
curl 127.0.0.1:8080/
# {"greeting": "Hello RAINL", "count": 12, "hostname": "201e047eead1", "storage": "redis"}
```

После down/up все еще 12. Дальше стопаю redis:

```sh
docker compose stop redis
curl 127.0.0.1:8080/healthz
# {"status": "alive"}
curl 127.0.0.1:8080/readyz
# {"error": "storage_unavailable"}
curl -X POST 127.0.0.1:8080/increment
# {"error": "storage_unavailable"}
```

поднимаю обратно

```sh
docker compose start redis
curl 127.0.0.1:8080/
# {"greeting": "Hello RAINL", "count": 12, "hostname": "201e047eead1", "storage": "redis"}
curl -X POST 127.0.0.1:8080/increment
# {"greeting": "Hello RAINL", "count": 13, "hostname": "201e047eead1", "storage": "redis"}
```

Hostname тот же, app сам переподключился. то что был conn error видно, потом снова стало хорошо.

```sh
docker compose logs app | grep -v 127.0.0.1
# app-1  | 2026-11-17 20:22:04,984 listening port=8000 storage=redis
# app-1  | 2026-11-17 20:22:10,367 http 172.20.0.1 "GET / HTTP/1.1" 200 -
# app-1  | 2026-11-17 20:22:10,620 http 172.20.0.1 "GET /healthz HTTP/1.1" 200 -
# app-1  | 2026-11-17 20:22:18,257 storage_unavailable: ConnectionError
# app-1  | 2026-11-17 20:22:18,257 http 172.20.0.1 "GET /readyz HTTP/1.1" 503 -
# app-1  | 2026-11-17 20:22:25,757 storage_unavailable: ConnectionError
# app-1  | 2026-11-17 20:22:25,757 http 172.20.0.1 "POST /increment HTTP/1.1" 503 -
# app-1  | 2026-11-17 20:22:32,002 http 172.20.0.1 "GET / HTTP/1.1" 200 -
# app-1  | 2026-11-17 20:22:32,009 http 172.20.0.1 "POST /increment HTTP/1.1" 200 -
```
