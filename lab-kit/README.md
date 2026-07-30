# lab-kit

Набор для **генерации** учебных HTML-лаб под голосовую ИИ.

Полная документация продукта: [../README.md](../README.md)  
Архитектура и runtime: [../docs/IMPLEMENTATION.md](../docs/IMPLEMENTATION.md)

---

## Состав

| Путь | Роль |
|------|------|
| `schema/presentation.schema.json` | JSON-контракт (meta, learning, sim, autoDemo, voice, questions) |
| `templates/lab-shell.html` | Ранний каркас auto/manual + `window.__LAB` (scroll-lesson) |
| `examples/plavanie-voice.manifest.json` | Пример толстого manifest |
| `../.grok/skills/edu-voice-lab/SKILL.md` | Skill агента `/edu-voice-lab` |

Готовые slide-deck лабы лежат в **`../labs/`** (референс реализации — `zakon-oma.html`).

---

## Генерация

В Grok:

```text
/edu-voice-lab
тема: <тема>, <класс> класс, ~3 минуты, слайды + авто-демо
```

Выход:

```text
labs/<id>.html
labs/<id>.manifest.json
labs/<id>.narration.txt
```

Non-negotiables:

1. Авто-демо **вкл. по умолчанию**
2. Manual после снятия галочки
3. Voice script: beats + fullNarration (видимый в UI / voice-context)
4. Научная модель ок для класса

---

## Engine catalog (шпаргалка)

| engine | Темы |
|--------|------|
| `sliders-instruments` | Ом, давление, мощность |
| `forces-balance` | рычаг, момент |
| `compare-three-outcomes` | плавание, трение |
| `assembly` / hybrid | схемы цепей |
| `graph-area` | интеграл, путь |
| `custom` | с `engineHint` |

---

## Voice

- Plain: `labs/<id>.narration.txt`
- Full UI: `labs/voice-context.html`
- Runtime: `window.__LAB.exportForTTS()`

Host: читать только `beats[].say` / `fullNarration`, на `question` — пауза.
