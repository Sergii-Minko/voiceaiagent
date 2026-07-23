# Запуск проекту в PowerShell (Windows)

Всі команди виконуються з кореня проекту:
```
C:\LocalVoiceAiOriginal\VoiceAgentAi\voiceaiagent>
```

---

## 1. Запуск контейнера (готовий образ, без перезбірки)

```powershell
docker compose -f docker-compose.yml -f docker-compose.gpu.yml -f docker-compose.local-models.yml up -d
```

`-d` — фоновий режим. Якщо хочеш бачити логи в реальному часі — прибери `-d`.

## 2. Перевірка статусу всіх сервісів

```powershell
docker logs voiceaiagent-app-1 --tail 30
```

Коли всі моделі завантажаться — побачиш рядок:
```
local-voice-ai is ready
```

## 3. Перевірка health-ендпоінтів

```powershell
docker compose exec app curl -s http://127.0.0.1:8880/health
docker compose exec app curl -s http://127.0.0.1:8000/health
docker compose exec app curl -s http://127.0.0.1:11434/v1/models
```

## 4. Відкрити фронтенд

Відкрий в браузері: http://localhost:8080

## 5. Зупинити контейнер

```powershell
docker compose -f docker-compose.yml -f docker-compose.gpu.yml -f docker-compose.local-models.yml down
```

## 6. Перезібрати образ (якщо вносили зміни в код)

```powershell
docker compose -f docker-compose.yml -f docker-compose.gpu.yml -f docker-compose.local-models.yml up --build -d
```

---

## Структура моделей на диску

Всі моделі завантажуються з локальної директорії:
```
C:\LocalVoiceAiOriginal\VoiceAgentAi\AgentModels\
```

Якщо шлях інший — задай змінну перед запуском:
```powershell
$env:LOCAL_MODELS_DIR = "C:\твій\шлях\до\моделей"
```

## Вимоги

- Docker Desktop з включеною підтримкою NVIDIA Container Toolkit
- GPU: NVIDIA RTX 5090 (32 GB VRAM) або інша CUDA-сумісна
