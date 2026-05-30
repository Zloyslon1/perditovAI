# OpenClaw 5.5 — фикс багов после апдейта

> **Когда читать**: если ты обновил OpenClaw с 4.x на 5.5 и **агенты замолчали в Telegram-группах**, или валятся ошибки `Unknown model`, или сообщения «обрабатываются» но в чат ничего не приходит.

> **Версия документа**: 2026-05-07. Применимо к OpenClaw `2026.5.5+`. Время на фикс: 5 минут.

---

## 🔍 Симптомы — как понять что у тебя этот баг

Ты обновился на 5.5 и теперь:

1. **Боты в группах молчат на @-упоминания**, хотя в DM (личке) могут отвечать
2. В логе много `skipping group message: reason=no-mention` хотя пользователь явно тегает бота
3. ИЛИ боты **«печатают»** в Telegram (статус `… печатает`), но текст не приходит в чат
4. ИЛИ ловишь `lane task error: FailoverError: Unknown model: codex/gpt-5.X`
5. ИЛИ session уходит в `status=failed` сразу после твоего сообщения
6. Все 8 ботов `connected` в `openclaw channels status`, но реально не отвечают

Если хотя бы 2 из 6 совпадают — это твой случай.

---

## 🎯 Корневых причин **три**, накладываются друг на друга

### Причина 1 — `visibleReplies` сброшен на `message_tool`

В 4.27+ OpenClaw добавил режим **"private replies"** в группах: бот обрабатывает сообщение, но финальный текст НЕ публикует в чат, пока не вызовет специальный tool `message(action=send)`.

Доктор при апдейте сбросил `messages.groupChat.visibleReplies` на дефолт `"message_tool"` (=скрыто) — нужно вернуть на `"automatic"` (=сразу публиковать).

### Причина 2 — Doctor переименовал `openai-codex/*` → `openai/*`

В config-миграции 5.5 доктор автоматически зарелейбил модели:
- было: `openai-codex/gpt-5.4` (через **ChatGPT Pro OAuth**, бесплатно)
- стало: `openai/gpt-5.4` (через **OPENAI_API_KEY**, платно по токенам)

Если ты не правил это вручную — у тебя сейчас идут **платные вызовы** через OpenAI API key вместо бесплатной ChatGPT Pro квоты.

### Причина 3 — несуществующие модели `codex/gpt-5.4`

Если кто-то (или другая инструкция) посоветовал заменить `openai-codex/*` на новое имя `codex/*` — учти:

В провайдере `codex` в 5.5 **существуют только**:
- `codex/gpt-5.2`
- `codex/gpt-5.4-mini`
- `codex/gpt-5.5`

Модели **`codex/gpt-5.4` НЕТ**. Если она в твоём config — ловишь `Unknown model`.

**Самое надёжное** — оставить **старое имя `openai-codex/*`**, оно всё ещё работает в 5.5 со всем зоопарком (5.1, 5.2, 5.3-codex, 5.4, 5.4-mini, 5.4-pro, 5.5).

---

## ⚙️ Шаги фикса (5 минут)

### Шаг 0 — Backup (на всякий случай)

```bash
bash ~/.openclaw/backup.sh
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-pre-fix-$(date +%Y%m%d-%H%M%S)
echo "✓ backup готов"
```

### Шаг 1 — Проверить версию и текущий config

```bash
openclaw --version
openclaw config get agents.defaults.model
openclaw config get messages.groupChat
```

Что должно быть **до фикса** (то от чего лечим):
```
default model: { primary: "openai/gpt-5.4", fallbacks: ["openai/gpt-5.5"] }   # неправильно
messages.groupChat: { visibleReplies: "message_tool" }                        # неправильно
```

Если уже **правильно** (`openai-codex/*` и `automatic`) — этот фикс тебе не нужен, у тебя другая проблема. Переходи к разделу «Что ещё может быть» внизу.

### Шаг 2 — Вернуть `visibleReplies` на `automatic`

```bash
openclaw config set messages.groupChat.visibleReplies "automatic"
```

Проверка:
```bash
openclaw config get messages.groupChat
# должно показать:
# { visibleReplies: "automatic" }
```

### Шаг 3 — Откатить модели на `openai-codex/*`

Сделай через любой текстовый редактор `nano ~/.openclaw/openclaw.json` или скрипт:

```bash
python3 << 'EOF'
import json
F = '/Users/antonpolakov/.openclaw/openclaw.json'  # ← поменяй на свой путь если другое имя пользователя
with open(F) as f: cfg = json.load(f)

def replace(d, old, new):
    if isinstance(d, dict):
        for k, v in list(d.items()):
            if isinstance(v, str) and v.startswith(old + '/'):
                d[k] = v.replace(old + '/', new + '/', 1)
            elif isinstance(v, (dict, list)):
                replace(v, old, new)
    elif isinstance(d, list):
        for i, x in enumerate(d):
            if isinstance(x, str) and x.startswith(old + '/'):
                d[i] = x.replace(old + '/', new + '/', 1)
            elif isinstance(x, (dict, list)):
                replace(x, old, new)

# Переименовать openai/gpt-5.* → openai-codex/gpt-5.*
# (если у тебя кто-то ставил codex/* — тоже фиксим)
replace(cfg, 'openai', 'openai-codex')
replace(cfg, 'codex', 'openai-codex')

# Убедиться что модели объявлены в configured block
m = cfg.setdefault('agents', {}).setdefault('defaults', {}).setdefault('models', {})
for model_name in ('openai-codex/gpt-5.4', 'openai-codex/gpt-5.5'):
    if model_name not in m:
        m[model_name] = {}
        print(f'  + added: {model_name}')

with open(F, 'w') as f: json.dump(cfg, f, indent=2)
print('✓ Models renamed back to openai-codex/*')
EOF
```

> **ВНИМАНИЕ для скрипта**: в нём есть строка `replace(cfg, 'openai', 'openai-codex')`. Эта замена жадная: если у тебя есть конфигурации `openai/*` которые **специально** должны идти через API key (не OAuth), они тоже превратятся в `openai-codex/*`. Если такие есть — отредактируй config вручную, а не запускай скрипт.

### Шаг 4 — Проверить что модели существуют

```bash
openclaw models list --all | grep "^openai-codex/" | head -5
```

Должны увидеть `openai-codex/gpt-5.4` и `openai-codex/gpt-5.5` с пометкой `yes` (auth есть).

### Шаг 5 — Validate + restart

```bash
openclaw config validate
# должно: Config valid: ~/.openclaw/openclaw.json

openclaw gateway restart
# подождать пока probe ok:
sleep 10 && openclaw gateway status | grep Connectivity
```

### Шаг 6 — Проверить ответы в группах

Напиши любому из агентов с @-упоминанием в твоей рабочей группе. Он должен ответить.

Если **сессия уже была** в `status=failed` (валилась на `Unknown model`) — она не самовосстановится. Нужно сбросить ту конкретную сессию:

```bash
# Например для producer:topic:1337 в группе -1003788764013
SK="agent:producer:telegram:group:-1003788764013:topic:1337"
F=$(python3 -c "
import json
with open('/Users/antonpolakov/.openclaw/agents/producer/sessions/sessions.json') as fh: d = json.load(fh)
v = d.get('$SK')
print(v.get('sessionFile') if v and isinstance(v, dict) else '')
")
TS=$(date -u +%Y-%m-%dT%H-%M-%SZ)
[ -f "$F" ] && mv "$F" "${F}.reset.${TS}"
T="${F%.jsonl}.trajectory.jsonl"
[ -f "$T" ] && mv "$T" "${T}.reset.${TS}"

# Удалить запись из sessions.json
python3 -c "
import json
f = '/Users/antonpolakov/.openclaw/agents/producer/sessions/sessions.json'
with open(f) as fh: d = json.load(fh)
if '$SK' in d: del d['$SK']
with open(f, 'w') as fh: json.dump(d, fh, indent=2)
print('✓ session entry removed')
"

openclaw gateway restart
```

Замени `producer` на имя твоего залипшего агента, а `-1003788764013:topic:1337` на свой `chatId:topic`.

---

## ✅ Как понять что фикс сработал

После шагов 1-6 проверь:

```bash
# 1. Конфиг моделей правильный
openclaw config get agents.defaults.model
# должно быть: { primary: "openai-codex/gpt-5.4", fallbacks: ["openai-codex/gpt-5.5"] }

# 2. visibleReplies правильный
openclaw config get messages.groupChat
# должно быть: { visibleReplies: "automatic" }

# 3. Авторизация на codex есть
openclaw models status | grep -A1 "openai-codex"
# должно быть: openai-codex effective=profiles ... (oauth=2)

# 4. Все боты connected
openclaw channels status | grep Telegram | grep -v default
# все строки должны заканчиваться на "connected, ..., mode:polling"

# 5. После теста — ошибок нет
LOG=$(ls -t /tmp/openclaw/openclaw-*.log | head -1)
grep -c '"logLevelName":"ERROR"' "$LOG"
# минимально, идеально 0
```

Если в Telegram пишешь агенту с @-упоминанием — получаешь ответ за 10-60 секунд (зависит от сложности вопроса и модели).

---

## 🔙 Откат если что-то пошло сильно не так

```bash
# Восстановить config из автоматического backup
cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json
openclaw gateway restart

# Если совсем сломалось — откатить версию OpenClaw
npm install -g openclaw@2026.4.26
# и восстановить config из снэпшота `~/.openclaw/backups/<timestamp>/`
```

---

## 🤔 Что ещё может быть, если фикс не помог

### A. `mentionPatterns` пустые

После 4.29 обработка @-упоминаний стала строже. Если у бота нет встроенной детекции через @username (например, ты пишешь по русскому имени «куратор» вместо `@true_curator_ai_bot`) — нужны явные паттерны.

Проверь:
```bash
openclaw config get agents.list | python3 -c "
import json, sys
agents = json.load(sys.stdin)
for a in agents:
    gc = a.get('groupChat')
    print(f'  {a.get(\"id\")}: mentionPatterns = {(gc or {}).get(\"mentionPatterns\")}')
"
```

Если у всех `None` — добавь паттерны на каждого агента в `agents.list[].groupChat.mentionPatterns` ИЛИ глобально через `messages.groupChat.mentionPatterns`. Пример паттернов для куратора:

```json
"mentionPatterns": [
  "@?true_curator_ai_bot",
  "(?<!\\w)Куратор(?!\\w)",
  "(?<!\\w)куратор(?!\\w)"
]
```

### B. ChatGPT Pro OAuth токен протух

```bash
openclaw models status | grep -A2 "openai-codex"
# Если видно "expired" — нужно перелогиниться:
openclaw login --provider openai-codex
# или (в новых версиях):
openclaw auth add openai-codex --type oauth
```

После — `openclaw gateway restart`.

### C. Агент пишет «exec размышления» в чат

В 4.27+ Hermes-style режим может leak'ать tool-calls в чат если у группы нет hygiene-systemPrompt. Лечится добавлением правил в `channels.telegram.accounts.<agent>.groups.*.systemPrompt`:

```
Не показывай и не пересказывай служебные tool calls/tool results, exec, команды оболочки, OCR-скрипты, логи инструментов и внутреннюю диагностику; пользователю отдавай только итоговый ответ и безопасные команды, которые он сам должен выполнить.
```

### D. Сетевые ошибки `EHOSTUNREACH` / `UND_ERR_*`

Это **не баг OpenClaw**, это твой Wi-Fi/провайдер/VPN моргает. Проверь:

```bash
curl -sS -o /dev/null -w "telegram: %{http_code} (%{time_total}s)\n" --max-time 5 https://api.telegram.org/
```

Если время ответа > 2 сек или http_code не 302/200 — лечи на стороне сети (роутер, VPN, кабель).

---

## 📞 Если ничего не помогло

1. Соберите дамп для куратора:
   ```bash
   openclaw status --all > ~/Downloads/openclaw-dump-$(date +%Y%m%d-%H%M%S).txt
   openclaw doctor >> ~/Downloads/openclaw-dump-$(date +%Y%m%d-%H%M%S).txt
   tail -200 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log >> ~/Downloads/openclaw-dump-$(date +%Y%m%d-%H%M%S).txt
   ```
2. Скиньте файл куратору с описанием:
   - какая версия OpenClaw (`openclaw --version`)
   - симптомы (молчит, ошибка, стак)
   - в какой группе и кому писали
   - время последней попытки

---

## 📚 Полная история этого бага

Этот баг возникает после апдейта **OpenClaw 4.x → 5.5** из-за того что doctor выполняет автомиграцию config с двумя проблемами одновременно:

1. **Сбрасывает `visibleReplies` на default `message_tool`** — поведение из 4.27, которое в групповых чатах превращается в режим «бот думает молча»
2. **Зачем-то релейбит provider name `openai-codex` → `openai`** — это разные провайдеры (OAuth vs API key) с разными биллингами

Кроме того в 5.5 появился новый провайдер `codex` с ограниченным набором моделей (нет gpt-5.4) — это сбивает с толку тех кто вручную пытается «обновить» имена.

**Самое стабильное решение**: оставить `openai-codex/*` (старое имя) и `visibleReplies: automatic`. Эта связка работала с 4.x и продолжает работать в 5.5+ без изменений.

---

*Дата создания: 2026-05-07. Если столкнёшься с другими симптомами после апдейтов — напиши куратору, добавим в эту инструкцию.*
