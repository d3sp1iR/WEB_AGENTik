# Cardio Web Agent

Автоматический агент для загрузки медицинских файлов (ЭКГ, PDF, CSV) на сервер и выполнения удалённых команд.

## Возможности

- C++ агент — отслеживает папку и отправляет новые файлы на сервер
- Node.js агент — получает и выполняет задания с сервера (команды, чтение файлов, изменение конфига)
- Автоматическая регистрация на сервере
- Bearer-токен авторизация
- Поддержка multipart/form-data загрузки файлов

## Архитектура

Оба агента работают независимо и могут использоваться по отдельности. C++ агент сканирует локальную папку и загружает новые файлы на сервер. Node.js агент опрашивает сервер на предмет заданий (FILE, TASK, CONF, TIMEOUT) и отправляет результаты выполнения.

## API для сервера

### C++ агент

POST /api/v1/agents/register — регистрация
Тело запроса:
```json
{
  "uid": "nurse-pc-001",
  "platform": "linux",
  "hostname": "nurse-workstation",
  "version": "1.0.0"
}
```
Ответ сервера:
```json
{
  "session_token": "ваш-токен"
}
```
POST /api/v1/agents/:uid/upload — загрузка файла
multipart/form-data, поля: file (файл), patient_id (строка, обычно "0")

### Node.js агент

POST /wa_task/ — получить задание
Тело запроса:
```json
{
  "UID": "sviat-008",
  "descr": "web-agentik1",
  "access_code": "57756a-4310-f6f0-26fe-17fe4ea9"
}
```
Ответ (есть задание):
```json
{
  "status": "RUN",
  "task_code": "TASK",
  "session_id": "12345",
  "options": "ls -la"
}
```
Ответ (нет заданий):
```json
{
  "status": "IDLE"
}
```

POST /wa_result/ — отправить результат
multipart/form-data, поля:
- result_code: число (0 = успех)
- result: JSON строка вида {"UID":"...","access_code":"...","message":"...","files":0,"session_id":"..."}
- file: файл (опционально, если files=1)

## Типы заданий (Node.js агент)

FILE — прочитать файл, options = путь к файлу
TASK — выполнить shell-команду, options = команда
CONF — изменить параметр конфигурации, options = key=value
TIMEOUT — изменить интервал опроса (сек), options = число

## Настройка

C++ агент — config.json:
```json
{
  "server_url": "http://localhost:3000",
  "agent_uid": "nurse-pc-001",
  "platform": "linux",
  "hostname": "nurse-workstation",
  "version": "1.0.0",
  "watch_dir": "/home/sviat/cardio_data",
  "poll_interval": 10,
  "log_file": "agent.log"
}
```

Node.js агент — конфиг в коде:
```javascript
const CONFIG = {
  UID: 'sviat-008',
  descr: 'web-agentik1',
  access_code: '57756a-4310-f6f0-26fe-17fe4ea9',
  server: 'https://xdev.arkcom.ru:9999/app/webagent1/api',
  interval: 10000
};
```

## Сборка и запуск

C++ агент (зависимости: libcurl, nlohmann/json, C++17):
mkdir build && cd build
cmake ..
make
./cardio_agent ./config.json

Node.js агент:
npm install axios dotenv form-data
node agent.js

## Поддерживаемые типы файлов (C++ агент)

.ecg, .csv, .pdf, .txt, .xml

## Логирование

C++ агент пишет в stdout и agent.log в формате [2025-03-14 12:00:01] [INFO] сообщение
Node.js агент пишет только в stdout

## Важные замечания

- Node.js агент отключает проверку SSL (NODE_TLS_REJECT_UNAUTHORIZED = '0'). Для production используйте валидные сертификаты.
- C++ агент запоминает уже отправленные файлы и не отправляет их повторно.
- Node.js агент имеет таймаут выполнения команды 30 секунд.
- При отсутствии отслеживаемой папки агент создаёт её автоматически.
