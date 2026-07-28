# lab-kit — генератор учебных презентаций под голосовую ИИ

## Зачем

Нужны не «слайды», а:

1. **Интерактив** (ползунки, режимы, приборы)
2. **Авто-демо по умолчанию** + галочка «выключить и крутить сам»
3. **Толстое описание** для голосовой ИИ: что говорить, когда спрашивать, что на экране

## Структура

```
lab-kit/
  schema/presentation.schema.json   # контракт JSON
  examples/plavanie-voice.manifest.json
  templates/lab-shell.html          # каркас UI + auto/manual + window.__LAB
labs/                               # готовые лабы (после генерации)
.grok/skills/edu-voice-lab/         # skill: /edu-voice-lab
```

## Как пользоваться

В Grok:

```
/edu-voice-lab
тема: давление в жидкостях, 7 класс, ~3 минуты озвучки
```

или просто:

```
сделай интерактивную презентацию: закон Ома, 8 класс, с авто-демо и сценарием для голоса
```

На выходе:

- `labs/<id>.html`
- `labs/<id>.manifest.json`
- `labs/<id>.narration.txt`

## Галочка авто-демо

| Состояние | Поведение |
|-----------|-----------|
| ✅ включена (default) | UI сам гоняет `autoDemo.steps`, ползунки locked |
| ☐ снята | tween stop, controls живые, ученик крутит |

## Что видит голосовая ИИ

```js
// в браузере
window.__LAB.exportForTTS()
// → { meta, persona, style, beats, fullNarration, questions, autoDemo }

window.__LAB.getNarrationText()  // plain string
window.__LAB.onBeat(cb)         // синхрон с UI
window.__LAB.setAuto(true|false)
```

Схема полей: `schema/presentation.schema.json`.

## Качество

Skill специально требует **толстый** manifest: misconceptions, вопросы, произносимые формулы, связка step↔beat. Не скелет «lorem + один range».
