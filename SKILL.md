---
name: adversarial-review
description: Adversarial AI code/plan review. Codex ревьюит, Claude правит, итеративный цикл до одобрения. Автодетект режима plan/code/code-vs-plan.
user_invocable: true
---

# Adversarial Code Review

Отправляет текущую работу на adversarial-ревью через внешнюю AI-модель (по умолчанию — OpenAI Codex). Автоматически определяет, что ревьюить: **план** или **код**. Claude правит по замечаниям ревьюера и переотправляет до одобрения. Максимум 5 раундов.

---

## Когда вызывать

- `/adversarial-review` — автодетект что ревьюить
- `/adversarial-review plan` — принудительно ревью плана
- `/adversarial-review code` — принудительно ревью кода
- `/adversarial-review <путь-к-файлу>` — ревью конкретного файла (аргумент содержит `/` или `.`)
- Переопределение reasoning: `/adversarial-review xhigh` или `/adversarial-review low` (одно из: `none`, `low`, `medium`, `high`, `xhigh`)
- Переопределение модели: `/adversarial-review model:gpt-5.3-codex` (аргумент с префиксом `model:`)

## Инструкции

> **Плейсхолдеры:** `${REVIEW_ID}`, `${CODEX_SESSION_ID}` и `${BASE_BRANCH}` в шагах ниже — это шаблонные плейсхолдеры, НЕ shell-переменные. Подставляй литеральные значения напрямую в каждый tool call.

### Шаг 1: Определить режим ревью

Определи, что ревьюить. Проверяй в порядке приоритета:

**1. Явный аргумент** (`plan`, `code`, путь к файлу) → использовать его.
   - Для `plan` → пропустить все git-проверки, перейти к шагу 2 (только REVIEW_ID).

**2. Claude Code Plan Mode** — если в контексте есть системное сообщение "Plan mode is active" → режим = `plan`, пропустить git. В Plan Mode код не редактируется, поэтому code/code-vs-plan невозможны.

**3. Автодетект** (без явного аргумента, вне Plan Mode):

1. Проверь наличие изменений кода (любой непустой — значит есть):
   - `git diff --name-only` — unstaged
   - `git diff --cached --name-only` — staged
   - `git diff --name-only ${BASE_BRANCH}...HEAD` — коммиты ветки
2. Проверь, есть ли план в текущем контексте разговора (из plan mode, задач или обсуждения).

| Изменения кода? | План в контексте? | Режим |
|----------------|-------------------|-------|
| Нет | Да | **plan** — ревью плана |
| Да | Да | **code-vs-plan** — ревью реализации против плана |
| Да | Нет | **code** — ревью изменений кода |
| Нет | Нет | Спросить пользователя, что ревьюить |

### Шаг 2: Сгенерировать Session ID и определить base branch

Сгенерируй уникальный `REVIEW_ID` самостоятельно, формат: `{unix_timestamp}-{случайное_4значное_число}`.
Пример: `1711872000-4821`. **НЕ используй bash** — подставляй значение напрямую в команды следующих шагов.

**Определение base branch (только для режимов `code` и `code-vs-plan`):**

Для режима `plan` — пропустить определение base branch, перейти к шагу 3.

Для остальных режимов определи base branch репозитория:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

Если команда вернула пустой результат (remote HEAD не настроен), используй fallback:

```bash
git rev-parse --verify main 2>/dev/null && echo main || echo master
```

Сохрани результат как `BASE_BRANCH` — используется в `git diff ${BASE_BRANCH}...HEAD` далее.

### Шаг 3: Подготовить материал для ревью

**Ревью плана:**

- Если план уже существует как файл (в `project/`, plan file от Plan Mode, memory или где-то в репо) — использовать путь напрямую. НЕ копировать. В Claude Code Plan Mode план всегда является файлом.
- Если план только в контексте разговора (вне Plan Mode) — записать через **Write tool** в `/tmp/claude-plan-${REVIEW_ID}.md`.
- **Обязательно вывести путь к файлу плана пользователю**, чтобы он мог открыть его в IDE:
  `📄 План для ревью: <путь-к-файлу>`

**Ревью кода:**

Собери список изменённых файлов:

1. `git diff --name-only` — unstaged changes
2. `git diff --cached --name-only` — staged changes

Объедини unstaged + staged (уникальные пути). Если оба пусты:

3. `git diff --name-only ${BASE_BRANCH}...HEAD` — коммиты ветки (fallback)

Branch берётся ТОЛЬКО когда нет локальных изменений — иначе контекст раздувается.
Для branch в промпте указывай команду `git diff ${BASE_BRANCH}...HEAD` (полный diff).

Ревьюер имеет доступ к репо и сам прочитает полный diff и файлы.
В промпт (шаг 4) передай список файлов и какие git diff команды запускать.

**Много файлов (> 50):** если объединённый список превышает 50 путей,
передай в промпт только git-команды без списка файлов — ревьюер разберётся сам.

Если все источники пусты — нет изменений для ревью, сообщи пользователю.

**Ревью кода против плана:** подготовить путь к плану И собрать список изменённых файлов (как выше).

### Шаг 4: Сформировать промпт и запустить первый раунд

Сформируй промпт в зависимости от режима. Все промпты используют adversarial stance.

**Промпт для ревью плана:**

```
<role>
You are a senior adversarial reviewer of implementation plans.
Your job is to break confidence in the plan, not to validate it.
</role>

<operating_stance>
Default to skepticism. Assume the plan has gaps until the evidence says otherwise.
Do not give credit for good intent or likely follow-up work.
If something only works on the happy path, treat that as a real weakness.
</operating_stance>

<task>
Review the implementation plan in <plan-path>.
</task>

<attack_surface>
Check each area. Skip if not applicable:
- Feasibility — will this approach actually work given the codebase and constraints?
- Missing steps — what is forgotten or assumed but not stated?
- Risk areas — what could go wrong during implementation? Data loss? Downtime?
- Sequencing — are steps in the right order? Are there hidden dependencies?
- Alternatives — is there a simpler or more robust approach?
- Rollback — can this be safely reverted if it fails halfway?
- Security — auth, data exposure, injection, unsafe operations
</attack_surface>

<finding_bar>
Each finding MUST answer:
1. What can go wrong? (concrete scenario, not hypothetical)
2. Why is this plan vulnerable? (cite specific section)
3. Impact — what breaks and how badly?
4. Recommendation — specific change to the plan
</finding_bar>

<scope_exclusions>
DO NOT comment on: formatting, wording style, speculative issues without concrete trigger scenario.
</scope_exclusions>

<calibration>
Prefer one strong finding over several weak ones.
If the plan is solid, say so clearly — false positives erode trust.
</calibration>

<output_format>
## Summary
One paragraph: what this plan does and your overall assessment.

## Findings
For each finding:
### [severity: critical|high|medium] Finding title
- **Section:** which part of the plan
- **What can go wrong:** ...
- **Why vulnerable:** ...
- **Impact:** ...
- **Recommendation:** ...

If no findings: "No actionable findings."

## Verdict
Rules: approve if no findings or all low severity; revise if any high/critical.
Choose exactly one. The LAST line of your response must be one of:
VERDICT: APPROVED
VERDICT: REVISE
</output_format>
```

**Промпт для ревью кода (<= 50 файлов):**

```
<role>
You are a senior adversarial code reviewer.
Your job is to break confidence in the change, not to validate it.
</role>

<operating_stance>
Default to skepticism. Assume the change can fail in subtle, high-cost,
or user-visible ways until the evidence says otherwise.
Do not give credit for good intent, partial fixes, or likely follow-up work.
If something only works on the happy path, treat that as a real weakness.
</operating_stance>

<task>
Review the code changes in this repo. Changed files:

<список файлов из --name-only>

Changes include: <unstaged changes / staged changes / unstaged + staged changes / branch changes vs ${BASE_BRANCH}>.
Run <git diff commands> to see the full diffs.
</task>

<attack_surface>
Check each area. Skip if not applicable to this change:
- Auth & permissions: bypasses, privilege escalation, missing checks
- Data integrity: loss, corruption, partial writes, constraint violations
- Race conditions: TOCTOU, concurrent access, deadlocks
- Rollback safety: can this change be safely reverted?
- Schema drift: migrations, backward compatibility, data format changes
- Error handling: swallowed errors, missing retries, cascading failures
- Observability: will operators know when this breaks?
</attack_surface>

<finding_bar>
Each finding MUST answer:
1. What can go wrong? (concrete scenario, not hypothetical)
2. Why is this code vulnerable? (cite specific file and lines)
3. Impact — what breaks and how badly? (data loss > downtime > degraded UX)
4. Recommendation — specific fix with code reference
</finding_bar>

<scope_exclusions>
DO NOT comment on: code style, formatting, naming conventions,
speculative issues without concrete trigger scenario,
"nice to have" improvements unrelated to correctness or safety.
</scope_exclusions>

<calibration>
Prefer one strong finding over several weak ones.
Severity: critical (data loss/security) > high (bug in prod) > medium (edge case).
If the change is solid, say so clearly — false positives erode trust.
</calibration>

<output_format>
## Summary
One paragraph: what this change does and your overall assessment.

## Findings
For each finding:
### [severity: critical|high|medium] Finding title
- **File:** path/to/file.ext lines N-M
- **What can go wrong:** ...
- **Why vulnerable:** ...
- **Impact:** ...
- **Recommendation:** ...

If no findings: "No actionable findings."

## Verdict
Rules: approve if no findings or all low severity; revise if any high/critical.
Choose exactly one. The LAST line of your response must be one of:
VERDICT: APPROVED
VERDICT: REVISE
</output_format>
```

**Промпт для ревью кода (> 50 файлов):**

Тот же промпт, что выше, но секция `<task>` без списка файлов:
```
<task>
Review the code changes in this repo.
Changes include: <unstaged changes / staged changes / ...>.
Run <git diff commands> to see changed files and full diffs.
</task>
```

**Промпт для ревью кода против плана:**

Тот же промпт для ревью кода, но секция `<task>` дополняется:
```
<task>
Review the code changes in this repo against the implementation plan in <plan-path>.
Changed files:

<список файлов или пусто если > 50>

Changes include: <тип>.
Run <git diff commands> to see the full diffs.
</task>
```

И в `<attack_surface>` добавляются пункты:
```
- Completeness: does the implementation cover all plan steps?
- Deviations: where does the code differ from the plan? Are deviations justified?
- Missing: what from the plan is not yet implemented?
```

**Запуск Codex — шаблон команды:**

Флаги:
- `-m gpt-5.4` — модель (переопределяется аргументом `model:...`)
- `-c model_reasoning_effort=high` — глубина рассуждения (переопределяется аргументом `xhigh`, `low` и т.д.)
- `-s read-only` — ревьюер только читает, не пишет
- `-o /tmp/codex-review-${REVIEW_ID}.md` — файл для записи ответа

```bash
timeout 600 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -s read-only \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  "ПРОМПТ"
```

**Важно:**
- Всегда оборачивай `codex exec` в `timeout 600` (10 минут). Если Codex зависнет — команда завершится с кодом 124.
- Используй параметр `timeout: 620000` в Bash tool для запаса.
- Команда **синхронная**: когда она вернулась, файл `-o` уже готов. **НЕ** используй poll-loop (`while/sleep`).
- Если exit code = 124 (таймаут) — сообщи пользователю и предложи повторить.

**После запуска:** найди в выводе строку `session id: <uuid>` и сохрани значение как `CODEX_SESSION_ID` — оно нужно для `resume` в последующих раундах.

**Примечания:**
- Модель по умолчанию: `gpt-5.4` с `model_reasoning_effort=high`. Пользователь может переопределить через аргументы.
- Всегда `-s read-only` — ревьюер не должен писать файлы.
- `-o` для захвата вывода в файл. **НЕ** запускай в background — команда сама вернёт управление.

### Шаг 5: Прочитать ревью и проверить вердикт

1. Прочитать `/tmp/codex-review-${REVIEW_ID}.md`
2. Показать пользователю **дословно** (verbatim) — не перефразировать findings ревьюера:

```
## Adversarial Review — Раунд N (режим: <plan|code|code-vs-plan>, модель: gpt-5.4)

[Отзыв ревьюера — дословно]
```

3. Проверить вердикт:
   - **VERDICT: APPROVED** → перейти к Шагу 8 (Готово)
   - **VERDICT: REVISE** → перейти к Шагу 6 (Правки)
   - Нет явного вердикта → считать parse failure, запустить resume/fallback с просьбой дать чёткий вердикт
   - Достигнут максимум (5 раундов) → перейти к Шагу 8 с пометкой

### Шаг 6: Внести правки

По замечаниям ревьюера:

**Для ревью плана:** исправить план — адресовать каждое замечание. Обновить файл плана (или temp-файл). Показать пользователю:

```
### Правки (Раунд N)
- [Что изменено и почему, один пункт на замечание]
```

**Для ревью кода:** исправить код напрямую — редактировать файлы, запустить тесты если применимо. Показать пользователю:

```
### Исправления (Раунд N)
- [Что исправлено и почему, один пункт на замечание]
```

**Пропустить** правку, если она противоречит явным требованиям пользователя — отметить это для пользователя.

### Шаг 7: Переотправить в Codex (Раунды 2-5)

**Resume — основной путь.** Экономит токены и сохраняет контекст сессии. Свежий `codex exec` без resume — **аварийный fallback**, расходует значительно больше токенов. Использовать только при ошибке resume.

1. Запусти resume с подавлением stderr (`2>/dev/null`):

```bash
timeout 600 codex exec resume ${CODEX_SESSION_ID} \
  "I've revised based on your feedback.

Here's what I changed:
[Список правок]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE" 2>/dev/null
```

Используй `timeout: 620000` в параметрах Bash tool.

**Почему `2>/dev/null`:** `codex exec` по дизайну разделяет потоки — progress/metadata → stderr, финальный ответ модели → stdout. Подавление stderr даёт чистый вывод без CLI-шума. Результат Bash tool = только ревью.

2. Проверь результат по exit code:
   - **exit 0** — успех. stdout содержит чистое ревью. Показать пользователю напрямую (Write в файл и Read **не нужны**). Проверить VERDICT: последняя непустая строка stdout = `VERDICT: APPROVED` или `VERDICT: REVISE`. Если вердикт отсутствует → вывод мог быть обрезан, перейти к Fallback. Далее применить обработку вердикта из Шага 5 (APPROVED → Шаг 8, REVISE → Шаг 6).
   - **exit 124** — таймаут. Сообщи пользователю: "Ревьюер не ответил за 10 минут" и предложи повторить.
   - **другой exit code** — сообщи пользователю: "Resume не удался (exit code N)". Перейди к Fallback. Диагностика без stderr недоступна — не пытайся парсить stdout как ошибку.

**Fallback** — если `resume` не сработал (сессия истекла, session ID не захвачен, ошибка):

1. Собрать список изменённых файлов (аналогично Шагу 3).
2. Запустить свежий `codex exec -o` с описанием предыдущих раундов в промпте.

Вернуться к **Шагу 5**.

### Шаг 8: Итоговый результат

**Одобрено:**
```
## Adversarial Review — Итог (режим: <режим>, модель: gpt-5.4)

**Статус:** ✅ Одобрено после N раунд(ов)

[Итоговый отзыв]

---
**Проверено и одобрено ревьюером. Ожидает вашего решения.**
```

**Достигнут максимум раундов:**
```
## Adversarial Review — Итог (режим: <режим>, модель: gpt-5.4)

**Статус:** ⚠️ Достигнут максимум (5 раундов) — не полностью одобрено

**Оставшиеся замечания:**
[Нерешённые вопросы]

---
**У ревьюера остались замечания. Просмотрите их и решите, как действовать дальше.**
```

### Шаг 9: Очистка

**В Claude Code Plan Mode:** пропустить любой cleanup (включая deferred). rm вызовет permission prompt. Файлы подчистятся при следующем вызове вне Plan Mode.

**Вне Plan Mode:**

```bash
rm -f /tmp/claude-plan-${REVIEW_ID}.md /tmp/codex-review-${REVIEW_ID}.md
```

Если пользователь отклонил rm — продолжить без ошибки.

НЕ удалять файлы планов, которые существовали до ревью (только temp-файлы, созданные этим скиллом). Старые temp-файлы от предыдущих сессий безвредны в /tmp и очистятся ОС при перезагрузке.

## Правила

- Claude **активно правит** по замечаниям ревьюера — это НЕ просто передача сообщений
- Findings ревьюера показываются **дословно** (verbatim) — не перефразировать, не сокращать
- Автодетект режима ревью по контексту; аргументы пользователя имеют приоритет
- При явном аргументе `plan` или в Claude Code Plan Mode: пропускать git-проверки и определение base branch
- Resume — основной путь для повторных раундов. Свежий exec — аварийный fallback (дорогой по токенам)
- Cleanup — best-effort: в Plan Mode пропускать, при отказе продолжать без ошибки
- Предпочитать существующие файлы, не создавать лишние копии
- Всегда read-only sandbox — ревьюер никогда не пишет файлы
- Максимум 5 раундов для защиты от бесконечных циклов
- Показывать пользователю отзывы и правки каждого раунда
- Если Codex CLI не установлен или упал — сообщить пользователю: `npm install -g @openai/codex`
- Если правка противоречит явным требованиям пользователя — пропустить и объяснить почему
