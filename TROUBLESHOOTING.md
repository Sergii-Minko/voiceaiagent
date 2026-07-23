# Аналіз помилок та висновки — Local Voice Agent

Файл створено за підсумками налагодження проекту **Local Voice Agent** в Docker з локальними моделями на Windows 11 + NVIDIA RTX 5090.

---

## 1. Docker — монтування томів (bind mount)

### Помилка
```
파일이 존재하지 않습니다 / file not found
```
або контейнер запускається, але всередині `ls /models/llm` нічого не показує.

### Причина
Docker Desktop на Windows використовує **WSL 2** для Linux-контейнерів. Шляхи формату `C:\Users\...` мають бути записані у форматі, зрозумілому WSL:
- `C:\Users\...` → `/c/Users/...` (тільки в WSL)
- Docker Compose на Windows автоматично конвертує Windows-шляхи в Linux, але **тільки якщо шлях починається з букви диска** (`C:\...`), а не змінної середовища.

### Рішення
- У `docker-compose.yml` використовувати **змінні середовища** для шляхів: `"${LOCAL_MODELS_DIR:-C:/LocalVoiceAiOriginal/...}/subdir"`
- Шлях з `C:` (з прямим слешем `/`) — Docker сам конвертує
- У змінних середовища PowerShell задавати **без лапок**:
  ```powershell
  $env:LOCAL_MODELS_DIR = "C:\LocalVoiceAiOriginal\VoiceAgentAi\AgentModels"
  ```

### Висновок
На Windows у Docker Compose використовуй **`C:/шлях/до/папки`** (з косою рискою `/`), а не `C:\шлях\до\папки` (з оберненою рискою `\`).

---

## 2. ARG у Dockerfile не доступний після FROM

### Помилка
```
# ERROR: DOWNLOAD_AGENT_FILES not available in build stage
```
або `RUN if [ "${DOWNLOAD_AGENT_FILES}" != "skip" ]; then ... fi` завжди виконується, навіть якщо передано `--build-arg DOWNLOAD_AGENT_FILES=skip`.

### Причина
Docker ARG, оголошені **до першого `FROM`**, не доступні в наступних стадіях. Вони існують тільки в контексті "global scope", який зникає після першого `FROM`.

### Рішення
Після кожного `FROM` потрібно повторно оголосити ARG:
```dockerfile
ARG LLAMA_IMAGE=...
FROM ${LLAMA_IMAGE} AS llama-bin

FROM ${PYTHON_BASE} AS runtime
ARG DOWNLOAD_AGENT_FILES   # <-- повторне оголошення
```

### Висновок
Після кожного `FROM` в Dockerfile — повторюй `ARG <ім'я>`, навіть якщо він уже був оголошений перед першим `FROM`.

---

## 3. Kokoro TTS — KPipeline не приймає `model_path`

### Помилка
```
TypeError: KPipeline.__init__() got an unexpected keyword argument 'model_path'
```

### Причина
У версії `kokoro` 0.9.4 `KPipeline.__init__()` має сигнатуру:
```python
KPipeline(lang_code, repo_id=None, model=KModel|bool=True, trf=False, ...)
```
Параметр `model_path` **не існує**. Шлях до .pth файлу потрібно передавати через **клас `KModel`**, а потім готовий об'єкт моделі — в `KPipeline`.

### Рішення
```python
from kokoro.model import KModel
model = KModel(model="/models/kokoro/kokoro-v1_0.pth")
_pipeline = KPipeline(lang_code="a", model=model)
```

### Висновок
Завжди перевіряй актуальну сигнатуру `__init__()` через:
```python
import kokoro
help(kokoro.KPipeline.__init__)
```
Бібліотеки можуть змінювати API між версіями. Не довіряй GitHub README — дивись код або help().

---

## 4. NeMo ASRModel.from_pretrained() не приймає локальний шлях

### Помилка
```
HFValidationError: Repo id must be in the form 'repo_name' or 'namespace/repo_name':
  '/models/nemotron/nemotron-speech-streaming-en-0.6b.nemo'
```

### Причина
`ASRModel.from_pretrained()` викликає Hugging Face Hub `validate_repo_id()`, який перевіряє, що ім'я моделі не містить слешів (крім одного `/` між namespace та repo_name). Абсолютні шляхи на зразок `/models/...nemo` **завжди відхиляються**.

### Рішення
Замість `from_pretrained()` використовувати `ASRModel.restore_from()`:
```python
if os.path.isfile(local_path):
    asr_model = ASRModel.restore_from(restore_path=local_path)
else:
    asr_model = ASRModel.from_pretrained(model_id)
```

### Висновок
NeMo має два методи завантаження:
- **`from_pretrained()`** — тільки для Hugging Face repo_id (або директорії з config.json).
- **`restore_from()`** — для локальних `.nemo` файлів, повних шляхів, архівів.

Перевіряй документацію конкретної версії NeMo (2.7.3 у нашому випадку).

---

## 5. UV pip install з CUDA — довге збирання

### Помилка
```
~17 хвилин на збирання Docker образу
```

### Причина
`uv pip install ".[ml,whisper]"` з `TORCH_INDEX_URL=https://download.pytorch.org/whl/cu124` завантажує:
- torch (502 MiB)
- nvidia-cublas (403 MiB)
- nvidia-cudnn (349 MiB)
- nvidia-nccl, nvidia-curand, тощо.

### Рішення
- Використовувати `--mount=type=cache,target=/root/.cache/uv` для кешування між збірками
- Для CPU-образу використовувати `TORCH_INDEX_URL=https://download.pytorch.org/whl/cpu` — набагато легше

### Висновок
CUDA-залежності torch — ~1.5 GB. Перша збірка завжди довга. Кешуй `uv cache` через Docker BuildKit mount.

---

## 6. Docker Compose — змінні середовища з лапками

### Помилка
```
/bin/sh: 1: Syntax error: "(" unexpected
```
або змінна передається з символами лапок усередині контейнера.

### Причина
У `docker-compose.yml` значення змінних пишуться **без лапок**:
```yaml
environment:
  LLAMA_OFFLINE: "1"       # правильно — YAML string
  LLAMA_MODEL_PATH: /models/llm/model.gguf  # правильно
```
Але в командному рядку PowerShell лапки важливі:
```powershell
$env:VAR = "значення"   # потрібні лапки
```

### Висновок
У YAML файлах значення без лапок — це рядки. У PowerShell — навпаки, лапки обов'язкові.

---

## 7. Docker Compose overlay — файли не об'єднуються магічно

### Помилка
Після додавання нового сервісу в `docker-compose.local-models.yml` він не з'являється.

### Причина
Overlay (`-f файл1 -f файл2`) працює тільки для **однакових ключів**. Якщо в `local-models.yml` описано той самий `services.app`, він доповнює/перезаписує поля. Але новий сервіс має бути в тому ж файлі, що основний, або додаватись як окремий `docker-compose` файл з новим `services:` блоком.

### Рішення
```yaml
# docker-compose.local-models.yml
services:
  app:                    # <-- той самий сервіс, що в docker-compose.yml
    volumes: [...]        # додається
    environment: [...]    # додається/перезаписує
```

### Висновок
Docker Compose merge працює по ключах. Новий том або змінна просто додаються до існуючого сервісу.

---

## 8. GPU — перевірка доступу

### Помилка
```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]
```

### Причина
Не встановлений **NVIDIA Container Toolkit**.

### Рішення
```powershell
# Перевірка
docker run --rm --gpus all ubuntu nvidia-smi

# Встановлення (якщо не працює):
# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
```

### Висновок
Перед запуском GPU-контейнерів завжди перевіряй `docker run --gpus all ubuntu nvidia-smi`.

---

## Загальні висновки для наступних проектів

1. **Docker на Windows** — шляхи у форматі `C:/папка/файл` (коса риска), а не `C:\папка\файл`
2. **Після кожного FROM** — повторюй ARG, якщо він потрібен у стадії
3. **Не довіряй README на GitHub** — перевіряй реальну сигнатуру через `help()` або код
4. **NeMo** має два методи: `from_pretrained()` (тільки для HF) і `restore_from()` (для локальних файлів)
5. **Kokoro** завантажує модель через `KModel(model=.pth)`, а не через `model_path` в `KPipeline`
6. **Перша збірка Docker з CUDA** завжди довга — кешуй через BuildKit
7. **Docker Compose overlay** — це просто merge словників, не чарівне об'єднання
8. **GPU в Docker** потребує NVIDIA Container Toolkit окремо
9. **Всі помилки були типовими** — несумісність версій бібліотек, неправильне API, особливості Docker на Windows. Жодна не була унікальною.

---

## Перелік виправлених файлів

| Файл | Що змінено |
|---|---|
| [local_voice_ai/services/kokoro/server.py](local_voice_ai/services/kokoro/server.py) | Замінено `KPipeline(lang_code, model_path=...)` на `KModel(model=...)` + `KPipeline(lang_code, model=model)` |
| [local_voice_ai/services/nemotron/server.py](local_voice_ai/services/nemotron/server.py) | Замінено `ASRModel.from_pretrained(local_path)` на `ASRModel.restore_from(restore_path=local_path)` |
| [Dockerfile](Dockerfile) | Додано повторне `ARG DOWNLOAD_AGENT_FILES` після `FROM ... AS runtime` |
| [docker-compose.local-models.yml](docker-compose.local-models.yml) | Створено — overlay з bind-mounts та змінними середовища для локальних моделей |
