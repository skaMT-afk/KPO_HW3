# Антиплагиат — микросервисное веб-приложение

Домашняя работа №3 КПО  
синхронное межсервисное взаимодействие

## Архитектура

Система состоит из трёх сервисов:

1. **FileStorageService**  
   Принимает файлы, сохраняет их на диск, фиксирует факт сдачи в SQLite.  
   API:
   - `POST /api/files` — загрузка файла (multipart/form-data: file, studentId, studentName?, assignmentId).
   - `GET /api/files/{id}/meta` — метаданные файла.
   - `GET /api/files/{id}/content` — содержимое файла.

2. **FileAnalysisService**  
   Запрашивает файл у FileStorageService, вычисляет хэш текста и определяет плагиат.  
   Плагиат = существует **более ранняя** сдача другим студентом с тем же хэшем по тому же заданию.  
   API:
   - `POST /api/reports/analyze` — анализ одной сдачи.
   - `GET /api/reports/by-assignment/{assignmentId}` — отчёты по заданию.

   Для визуализации формирует URL облака слов через QuickChart Word Cloud API:  
   `https://quickchart.io/wordcloud?text=...&width=600&height=400&maxNumWords=50`.  

3. **ApiGateway**  
   Центральная точка входа для клиентов.  
   API:
   - `POST /api/works/submit`  
     1. Отправляет файл в FileStorageService.  
     2. После успешного сохранения вызывает FileAnalysisService.  
     3. Возвращает:
        ```json
        {
          "submission": { ... },
          "report": { ... } | null,
          "analysisStatus": "Completed | AnalysisServiceUnavailable | ErrorFromAnalysisService:XXX"
        }
        ```
   - `GET /api/works/{assignmentId}/reports` — проксирует запрос в FileAnalysisService.

## Обработка ошибок

- Если **FileStorageService** недоступен — ApiGateway возвращает `503 Service Unavailable`.
- Если **FileAnalysisService** недоступен:
  - сдача сохраняется;
  - в ответе `report = null`, `analysisStatus = "AnalysisServiceUnavailable"`.

Таким образом, падение одного микросервиса не ломает систему целиком.  

## Запуск

```bash
docker compose up --build
```

После запуска:

- Swagger ApiGateway: `http://localhost:5000/swagger`
- Через него можно:
  - сдать работу (`POST /api/works/submit`);
  - посмотреть отчёты (`GET /api/works/{assignmentId}/reports`).
