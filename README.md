# edu-voice-labs

Интерактивные **HTML-презентации** по физике для школы: слайды, симулятор с **авто-демо**, сценарий для **голосовой ИИ**.

**Live (GitHub Pages):** https://dari0606.github.io/edu-voice-labs/  
**Репозиторий:** https://github.com/dari0606/edu-voice-labs

---

## Что это

Не PowerPoint и не просто «красивый лендинг». Это:

1. **Слайды** — листание (кнопки, ← →, пробел, свайп).
2. **Симулятор** — canvas + ползунки; физика/формулы считаются в браузере.
3. **Авто-демо** — по умолчанию **включено**: параметры сами меняются по сценарию.
4. **Ручной режим** — снял галочку «Авто-демо» → ученик крутит сам.
5. **Голосовой контекст** — готовый текст (`fullNarration` + `beats`), который берёт voice-модель.

Целевая аудитория: 7–8 класс (и расширяемо).

---

## Быстрый старт

### Открыть локально

```bash
# из корня репо
open index.html
# или
open labs/index.html
```

Либо любой файл из `labs/*.html` двойным кликом / Live Server.

### Открыть в сети

| Страница | URL |
|----------|-----|
| Хаб | https://dari0606.github.io/edu-voice-labs/labs/ |
| **Текст для голосовой ИИ** | https://dari0606.github.io/edu-voice-labs/labs/voice-context.html |
| Закон Ома | …/labs/zakon-oma.html |
| Давление в жидкости | …/labs/davlenie-zhidkosti.html |
| Рычаг | …/labs/rychag.html |
| Последовательно / параллельно | …/labs/posled-parallel.html |

---

## Каталог готовых лаб

| id | Тема | Класс | Модель |
|----|------|-------|--------|
| `zakon-oma` | Закон Ома | 8 | \(I = U/R\), ключ, лампа |
| `davlenie-zhidkosti` | Давление в жидкости | 7 | \(p = \rho g h\), \(g=10\) |
| `rychag` | Рычаг | 7 | \(M=F\cdot d\), равновесие |
| `posled-parallel` | Последовательно / параллельно | 8 | \(R_1+R_2\), \(1/R=\ldots\) |
| `plavanie-tel` | Плавание (ранняя версия) | 7 | плотности, не slide-deck |

Ранние прототипы в корне: `30-plavanie-tel-*.html`, `elektricheskaya-cep-3d.html`, `integral-3d.html` (отдельные full-page 3D/2D лабы, не slide-формат).

---

## Структура репозитория

```text
.
├── index.html                 # редирект → labs/
├── README.md                  # этот файл
├── docs/
│   └── IMPLEMENTATION.md      # подробная реализация
├── lab-kit/                   # «как генерировать»
│   ├── schema/presentation.schema.json
│   ├── templates/lab-shell.html
│   ├── examples/plavanie-voice.manifest.json
│   └── README.md
├── labs/                      # готовые продукты
│   ├── index.html             # хаб
│   ├── voice-context.html     # все скрипты для voice
│   ├── <id>.html              # слайды + симулятор
│   ├── <id>.manifest.json     # полный контракт
│   └── <id>.narration.txt     # plain TTS
└── .grok/skills/edu-voice-lab/
    └── SKILL.md               # skill для Grok: /edu-voice-lab
```

---

## Флоу генерации новой темы

```text
Тема + класс
    ↓
Grok + skill edu-voice-lab
    ↓
manifest (педагогика + sim + autoDemo + voiceScript)
    ↓
labs/<id>.html  +  .manifest.json  +  .narration.txt
    ↓
Ученик: слайды / авто-демо
Voice:  narration / beats / exportForTTS()
    ↓
(опционально) git push → GitHub Pages
```

### Команда в Grok

```text
/edu-voice-lab
тема: <название>, <класс> класс, ~3 минуты, слайды + авто-демо
```

Подробности skill: `.grok/skills/edu-voice-lab/SKILL.md`  
Схема JSON: `lab-kit/schema/presentation.schema.json`

### Обязательные артефакты на тему

| Файл | Назначение |
|------|------------|
| `labs/<id>.html` | UI: слайды, sim, auto/manual, 🎤 Скрипт |
| `labs/<id>.manifest.json` | Контракт: learning, controls, auto steps, beats, questions |
| `labs/<id>.narration.txt` | Весь текст озвучки одной строкой/абзацем |

---

## Авто-демо и ручной режим

| | Авто (default) | Manual |
|--|----------------|--------|
| Галочка `#autoDemo` | ✅ checked | ☐ unchecked |
| Ползунки | locked, tween по `autoDemo.steps` | живые |
| Текст учителя | `beats[].say` по `sayId` | можно крутить и читать status |
| Когда | слайд симулятора | тот же слайд |

Связь: `autoDemo.steps[].sayId` → `voiceScript.beats[].id`.

---

## Контракт для голосовой ИИ

### Где взять текст

1. **UI:** https://…/labs/voice-context.html  
2. **На лабе:** кнопка **🎤 Скрипт**, последний слайд «Текст для озвучки»  
3. **Файлы:** `*.narration.txt`, `*.manifest.json`  
4. **JS в браузере:**

```js
window.__LAB.exportForTTS()
// → { meta, persona, style, beats, fullNarration, questions, autoDemo }

window.__LAB.getNarrationText()
window.__LAB.getVoiceBeats()
window.__LAB.onBeat(cb)      // подписка на смену реплики
window.__LAB.setAuto(true|false)
window.__LAB.goToSlide?.(n)  // если slide-deck
```

### Рекомендуемый host-flow

```text
load beats / fullNarration
for each beat in order:
  if beat has autoStepId → sync UI step (optional)
  speak(beat.say)   // only plain text, no markdown
  if phase == question → pause ~2–2.5 s
outro already says: «сними Авто-демо и попробуй сам»
```

Фазы beats: `hook → setup/demo → question → answer → formula → transfer → summary`.

---

## Типичная структура слайдов (10)

1. Титул  
2. Загадка / misconception  
3. **Симулятор** (авто-демо)  
4. Вопрос 1  
5. Формула / закон  
6. Вопрос 2  
7. Типичные ошибки  
8. В жизнь  
9. Итог  
10. **Текст для голосовой ИИ** (full script)

Навигация: «Назад / Далее», точки, клавиши, свайп, hash `#s3`.

---

## Технологии

- **Single-file HTML** (CSS + JS внутри), без сборки для лаб.
- **Canvas 2D** для схем (по умолчанию; Three.js только в старых 3D-прототипах).
- **Без бэкенда** — всё локально в браузере.
- **GitHub Pages** — ветка `main`, корень `/`.

---

## Деплой

```bash
git add -A
git commit -m "Add lab: <id>"
git push origin main
# Pages: Settings → Pages → branch main, / (root)
# URL: https://<user>.github.io/edu-voice-labs/
```

Аккаунт деплоя (текущий): `dari0606/edu-voice-labs`.

---

## Документация дальше

- [docs/IMPLEMENTATION.md](docs/IMPLEMENTATION.md) — архитектура, API, schema, чеклист качества  
- [lab-kit/README.md](lab-kit/README.md) — kit для генерации  
- [.grok/skills/edu-voice-lab/SKILL.md](.grok/skills/edu-voice-lab/SKILL.md) — инструкции агенту  

---

## Лицензия / использование

Учебный прототип. Контент и симуляции — упрощённые модели школьного уровня (это указано в UI, где важно).
