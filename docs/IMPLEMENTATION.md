# Реализация edu-voice-labs

Подробное описание того, **как устроена** система: данные, UI, авто-демо, голос, генерация.

Связанные файлы:

- Продуктовый обзор: [../README.md](../README.md)
- JSON Schema: [../lab-kit/schema/presentation.schema.json](../lab-kit/schema/presentation.schema.json)
- Skill генерации: [../.grok/skills/edu-voice-lab/SKILL.md](../.grok/skills/edu-voice-lab/SKILL.md)
- Пример manifest: [../lab-kit/examples/plavanie-voice.manifest.json](../lab-kit/examples/plavanie-voice.manifest.json)

---

## 1. Цели реализации

| Цель | Как закрыто |
|------|-------------|
| Ученик «видит закон» | Canvas + live readouts, не статичная картинка |
| Не нужен учитель у доски 24/7 | Авто-демо ведёт опыт по шагам |
| Можно потрогать | Галочка → manual |
| Voice AI не импровизирует | Жёсткий сценарий: beats + fullNarration |
| Деплой без бэкенда | Static HTML → GitHub Pages |

---

## 2. Архитектура

```text
┌────────────────────────────────────────────────────────────┐
│  labs/<id>.html  (runtime)                                 │
│  ┌──────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │ Slide    │  │ Simulator   │  │ Voice UI              │  │
│  │ navigator│  │ state+physics│  │ panel / 🎤 / slide  │  │
│  └────┬─────┘  └──────┬──────┘  └───────────┬───────────┘  │
│       │               │                     │              │
│       └─────────── window.__LAB ────────────┘              │
│                    manifest in JS                          │
└────────────────────────────────────────────────────────────┘
         ▲                           │
         │ generate                  │ export
         │                           ▼
┌────────────────┐         ┌─────────────────────────────┐
│ Grok skill     │         │ .narration.txt              │
│ edu-voice-lab  │────────►│ .manifest.json              │
│ + lab-kit      │         │ voice-context.html (сводка) │
└────────────────┘         └─────────────────────────────┘
```

Нет сервера, нет БД. Manifest дублируется:

1. В **отдельном JSON-файле** (для внешних систем / git).
2. **Внутри HTML** как JS-объект `manifest` (чтобы страница работала offline одним файлом).

---

## 3. Контракт данных (manifest)

Корень schema: `lab-kit/schema/presentation.schema.json`.

### 3.1. Блоки

| Блок | Обязателен | Содержание |
|------|------------|------------|
| `meta` | да | id, title, subject, grade, lang, durationSec |
| `learning` | да | goal, objectives, misconceptions, formulas (latex + **plain**), takeaways |
| `ui` | да | theme, layout (`scroll-lesson` \| `slides`), sections |
| `simulator` | да | engine, controls, readouts, defaults, verdictRules |
| `autoDemo` | да | enabledByDefault, loop, steps[] |
| `voiceScript` | да | persona, style, beats[], fullNarration |
| `interaction` | да | autoDefault, manualUnlock, questions[] |
| `export` | нет | пути html/manifest/narration |

### 3.2. autoDemo.steps[]

```json
{
  "id": "double-U",
  "durationMs": 5200,
  "state": { "U": 12, "R": 6, "closed": true },
  "highlight": ["U", "I"],
  "sayId": "b-double-u",
  "waitForVoice": true
}
```

- `state` — целевые значения controls (tween или snap).
- `sayId` — id реплики в `voiceScript.beats`.
- `highlight` — подсветка ctrl/gauge по `data-control`.

### 3.3. voiceScript.beats[]

```json
{
  "id": "b-q1",
  "phase": "question",
  "say": "Если U удвоим, ток станет больше? Во сколько раз?",
  "onScreen": "подсветка U",
  "autoStepId": "double-U",
  "expectStudent": "think"
}
```

**Правила `say`:**

- разговорный текст, без markdown и LaTeX-символов;
- числа удобны для TTS («двенадцать вольт»);
- формулы словами («и равно у на эр»).

`fullNarration` = склейка всех `say` (или отредактированный continuous script).

### 3.4. Engines (каталог)

| engine | Примеры тем | Controls (типично) |
|--------|-------------|--------------------|
| `sliders-instruments` | Ом, давление | range + toggle |
| `forces-balance` | рычаг | F, d с двух сторон |
| `compare-three-outcomes` | плавание, трение | material + rho |
| `assembly` / hybrid | схемы | mode chips + lamp toggles |
| `graph-area` | интеграл / путь | n, a, b, profile |
| `custom` | прочее | engineHint обязателен |

Физика **учебная**: упрощения допустимы, если написаны в theory (напр. \(g=10\), \(R_{\text{лампы}}=\text{const}\)).

---

## 4. Runtime HTML (slide-deck)

Референс: `labs/zakon-oma.html` (и три соседние лабы).

### 4.1. Слои UI

1. **`.deck`** — колонка: slides + nav.  
2. **`.slides` > `.slide`** — полноэкранные кадры; один `.active`.  
3. **Nav** — prev/next, dots, counter.  
4. **Lab slide** — toolbar (auto), canvas, controls, gauges, status, voice-panel.  
5. **🎤 FAB + modal** — полный скрипт для voice.  
6. **Последний slide** — fullNarration + список beats.

### 4.2. Навигация по слайдам

| Input | Действие |
|-------|----------|
| Кнопки | prev / next |
| Точки | jump |
| `←` `→` `PageUp/Down` `Space` | листание |
| Swipe | touch |
| `#sN` | deep link |

`go(n)` → `onSlideEnter(n)`:

- показывает beat по `data-voice`;
- если lab-slide → `labActive=true`, старт auto;
- иначе → pause tween.

### 4.3. Состояние симулятора

Типичный `state`:

```js
{
  auto: true,
  // domain fields: U, R, h, rho, F1, ...
  labActive: false,
  stepIndex: 0,
  tween: null | { from, to, t0, dur },
  holdTimer: null,
  beatListeners: []
}
```

**Цикл кадра:**

1. Если `auto && labActive && tween` — lerp controls (smoothstep).  
2. По `u>=1` — snap, `advanceAuto()` / loop.  
3. `updateReadouts()` + `draw(canvas)`.

**setAuto(on):**

- `on` → lock controls, `startStep(0)`;
- `off` → unlock, stop tween, clear highlights.

### 4.4. window.__LAB

Минимальный публичный API:

| Метод | Описание |
|-------|----------|
| `manifest` | полный объект |
| `getState()` | domain + auto + slide |
| `setAuto(bool)` / `isAuto()` | режим |
| `getVoiceBeats()` | beats[] |
| `getNarrationText()` | fullNarration |
| `getQuestions()` | questions[] |
| `onBeat(cb)` | подписка; return unsubscribe |
| `goToStep(i)` | jump auto step (если auto) |
| `goToSlide?.(n)` | slide-deck |
| `exportForTTS()` | пакет для voice host |

Опционально: `window.__qa` в старых 3D-лабах (тестовые хуки).

---

## 5. Voice surface (где виден текст)

| Место | Что показывает |
|-------|----------------|
| `labs/voice-context.html` | все темы: full + beats + JSON + copy |
| `labs/index.html` | баннер + таблица ссылок на txt/json |
| Кнопка **🎤 Скрипт** | modal: full + beats + copy |
| Последний слайд | full + ol beats |
| Панель «Сейчас говорит учитель» | текущий `say` |
| `*.narration.txt` | raw file |
| `*.manifest.json` | machine-readable |

Это сделано специально: voice-модель **не должна** вытаскивать текст из canvas.

---

## 6. Пайплайн генерации (агент)

Skill: `.grok/skills/edu-voice-lab/SKILL.md`.

```text
1. Input: тема, класс, (~duration)
2. Выбрать engine + misconception-hook
3. Написать толстый manifest (quality budget)
4. Собрать HTML (slide shell + canvas sim + auto + voice UI)
5. narration.txt = fullNarration
6. Self-check checklist
7. Сообщить пути + как отдать voice
```

### Self-check (must)

- [ ] Авто двигает параметры на lab-slide  
- [ ] Manual после uncheck  
- [ ] Re-check auto снова работает  
- [ ] Каждый step.sayId существует в beats  
- [ ] ≥2 questions связаны с beats  
- [ ] Формула/модель ок на 2–3 крайних точках  
- [ ] Нет placeholder/lorem  
- [ ] `exportForTTS()` не пустой  
- [ ] Текст голоса виден (🎤 / voice-context / narration.txt)

### Визуальный стиль

- Фон `#f6f5fb`, акцент `#7359ff`
- Cards, gauges, status strip
- Контент слайдов **по центру**
- Lab slide шире (`max-width ~920px`)

---

## 7. Реализованные модели (кратко)

### Закон Ома (`zakon-oma`)

- \(I = U/R\) при `closed`, иначе \(0\)
- \(P = U\cdot I\), яркость лампы от \(P\)
- Steps: base → double U → double R → open key → bright

### Давление (`davlenie-zhidkosti`)

- \(p = \rho \cdot g \cdot h\), \(g=10\)
- Verdict по \(p\) (слабо / ощутимо / сильно)
- Steps: 1 м → 10 м → Мёртвое море → 0.5 м

### Рычаг (`rychag`)

- \(M_1=F_1 d_1\), \(M_2=F_2 d_2\)
- Угол доски от \(\Delta M\)
- Steps: balance → heavy right → far left → kid vs adult

### Последовательно / параллельно (`posled-parallel`)

- Series: \(R=R_1+R_2\); обрыв любой → \(I=0\)
- Parallel: \(1/R=\sum 1/R_i\); ветка off не гасит другую
- \(R_{\text{lamp}}=6\,\Omega\) (учебное)

---

## 8. Деплой

- Static hosting: **GitHub Pages**, branch `main`, path `/`.
- Root `index.html` → redirect на `labs/index.html`.
- После push: https://`<user>`.github.io/edu-voice-labs/

```bash
git add labs/ lab-kit/ docs/ README.md index.html
git commit -m "Describe change"
git push origin main
```

Убедиться, что push идёт с аккаунта, у которого write access (в проекте: `dari0606`).

---

## 9. Ограничения и долг

| Ограничение | Комментарий |
|-------------|-------------|
| Нет единого npm-билда | Каждая лаба — single HTML (плюс/минус) |
| Three.js бандл | Только в старых 3D-файлах; slide-лабы без него |
| Синхрон voice↔UI | Host должен сам слушать `onBeat` / `autoStepId` |
| a11y | Canvas слабо доступен screen reader; текст дублируется в UI |
| Нет STT проверки ответов | questions + acceptKeywords — задел |
| Дубль manifest | JSON-файл и inline JS могут разъехаться — обновлять оба |

### Возможные следующие шаги

1. Общий `lab-runtime.js` (nav + auto + voice modal) вместо копипаста.  
2. CI: smoke-тест `__LAB.exportForTTS()` в headless browser.  
3. Voice host reference (Web Speech API или внешний TTS).  
4. Расширить catalog engines (трение, Ньютон, работа тока).

---

## 10. Глоссарий

| Термин | Значение |
|--------|----------|
| **Lab** | Одна тема = HTML + manifest + narration |
| **Beat** | Одна реплика голоса + фаза |
| **Auto-demo** | Проигрывание `steps[]` с tween state |
| **Manual** | Ученик владеет controls |
| **Engine** | Тип симулятора из каталога |
| **fullNarration** | Continuous text для простого TTS |
| **exportForTTS** | JSON-пакет для voice host |

---

*Документ отражает состояние репозитория на момент написания. При смене API `__LAB` или schema — обновлять этот файл.*
