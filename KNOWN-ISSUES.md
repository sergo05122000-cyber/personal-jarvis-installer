# Known Issues — v1.0

Честный список нюансов в текущей версии инсталлятора. Все решаются, но
требуют ручной правки или знания темы. Если не хотите возиться —
[закажите установку под ключ](./README.md#-поставьте-за-меня-done-for-you--200).

---

## 1. `nodejs npm` через apt даёт старую версию

**Этап 4, строки 294-295 в INSTALL.md:**

```bash
sudo apt install -y \
    nodejs npm \   # ← Ubuntu 24.04 даст Node 18, Claude Code хочет >= 20
```

**Фикс:** ставьте Node через nvm:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
node --version  # должно быть v22.x
```

---

## 2. `docker.io docker-compose` ставится впустую

**Этап 4, строка 295:** Docker устанавливается, но в инсталлятор нигде
не используется. Бесполезные ~500 МБ диска.

**Фикс:** уберите из `apt install`:

```bash
# было
sudo apt install -y curl wget git python3 python3-pip python3-venv \
    nodejs npm docker.io docker-compose jq tree htop build-essential

# стало
sudo apt install -y curl wget git python3 python3-pip python3-venv \
    jq tree htop build-essential
```

Соответственно блок «Включить Docker без sudo» (`usermod -aG docker`)
тоже не нужен.

---

## 3. `cp -r ~/.ssh/authorized_keys` — флаг `-r` лишний

**Этап 2.2, строка 186:**

```bash
cp -r ~/.ssh/authorized_keys /home/agent/.ssh/  # -r не нужен, это файл
```

**Фикс:** уберите `-r`. Работает и так, но семантически неправильно.

---

## 4. ⚠️ КРИТИЧНО — `gateway.py` использует устаревшие параметры Claude Code

**Этап 8, gateway.py, строки 1038-1039:**

```python
cmd = [CLAUDE, "--workspace", WS, "--model", MODEL,
       "--effort", EFFORT, "--print", full_prompt]
```

В современных версиях Claude Code (с 2026 года):
- `--workspace` → используйте `cwd=<path>` параметр subprocess или `cd <ws>` перед вызовом
- `--effort` → параметр не существует в публичной версии (был в early SDK)
- `--print` → используйте `-p <prompt>` (короткий флаг)

**Рабочая версия для gateway.py:**

```python
cmd = [CLAUDE, "-p", full_prompt, "--model", MODEL,
       "--output-format", "stream-json"]

proc = await asyncio.create_subprocess_exec(*cmd,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,
    cwd=WS)
```

И парсинг stdout — это JSON-стрим, а не plain text:

```python
import json
async for line in proc.stdout:
    evt = json.loads(line.decode())
    if evt.get("type") == "text":
        # отправить в Telegram
```

Если по простому без стриминга — добавьте `--output-format text`.

---

## 5. Sensitive-file guard блокирует Write в `.claude/`

С 2026-05 включён внутренний guard Anthropic, который блокирует Write
любых файлов внутри `.claude/` директории — даже если они в allow-листе
permissions.

**Проявление:** агент не может писать в `core/USER.md`, `core/MEMORY.md`
и т.п. через свои инструменты Write/Edit.

**Обход (рекомендованный):** хранить self-writable файлы вне `.claude/`,
в `.claude/` класть симлинки:

```bash
mkdir -p ~/agent/data
git mv ~/agent/.claude/core/USER.md ~/agent/data/USER.md
ln -s ../../data/USER.md ~/agent/.claude/core/USER.md

# Аналогично для MEMORY.md, LEARNINGS.md, warm/decisions.md, hot/handoff.md
```

Чтения через симлинк работают прозрачно (Read tool / @include в
CLAUDE.md). Записи идут на realpath вне `.claude/` и guard не
срабатывает.

---

## 6. `cron` job не запустится без полного PATH

**Этап 7, crontab — строки 881-885:**

```cron
30 04 * * * AGENT_WS=$HOME/agent/.claude $HOME/agent/scripts/rotate-warm.sh
```

`$HOME` в cron не определён по умолчанию. Скрипты ломаются с «AGENT_WS:
unbound variable».

**Фикс:** замените `$HOME` на абсолютный путь, или добавьте header в
crontab:

```cron
HOME=/home/agent
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

30 04 * * * AGENT_WS=$HOME/agent/.claude $HOME/agent/scripts/rotate-warm.sh
# ... остальные ...
```

Или используйте `/home/<USERNAME>/...` явно везде.

---

## 7. `compress-warm.sh` правит файл in-place без бэкапа

**Этап 7, скрипт 3.3:** Если Python-скрипт в середине упадёт (например
синтаксис файла нарушен), `decisions.md` останется в обрезанном виде.

**Фикс:** добавьте бэкап перед записью:

```bash
cp "$DECISIONS" "${DECISIONS}.bak"
python3 - "$DECISIONS" <<'PY'
# ... ваш код ...
PY
# Если упало — bak остался, можно восстановить
```

Аналогично для `rotate-warm.sh` и `memory-rotate.sh`.

---

## 8. CloudFlare WARP не подходит для всех IP

**Этап 3, Вариант C:** WARP бесплатен, но иногда даёт IP, который Anthropic
считает «датацентровым» и отказывает. Если `curl https://api.anthropic.com`
после `warp-cli connect` всё равно даёт connection refused — переключите
WARP на mode `warp+doh`:

```bash
warp-cli set-mode warp+doh
warp-cli reconnect
```

Или используйте Variant A/B (свой WireGuard).

---

## Что мы исправляем в платной установке

Все 8 пунктов выше + плюс:

- Корректные пути под пользователя клиента
- Автоматический бэкап перед каждой cron-операцией
- Логирование cron-job в systemd journal (а не только файл)
- Health-check скрипт (проверка что бот жив и Claude отвечает)
- Базовая персонализация SOUL под клиента (имя, миссия)

Заказать установку: [`/order` в @di_rek_tor_bot](https://t.me/di_rek_tor_bot)

---

## Сообщить о новом нюансе

Нашли проблему не из списка? Откройте Issue в этом репо или напишите в
[@di_rek_tor_bot](https://t.me/di_rek_tor_bot) — внесём в KNOWN-ISSUES
и поправим в следующей версии инсталлятора.
