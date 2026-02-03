# QuestionCard — Frontend Test

## Часть 1. Архитектура компонента (FSD)

### Структура слоёв

```
src/
├── shared/
│   └── ui/
│       ├── TipTapRenderer/
│       ├── KaTeXRenderer/
│       ├── Button/
│       └── ErrorBoundary/
├── entities/
│   └── question/
│       ├── model/types.ts
│       └── ui/QuestionStem/
├── features/
│   └── answer-question/
│       ├── model/useAnswerQuestion.ts
│       └── ui/AnswerOptions/
└── widgets/
    └── question-card/
        └── ui/QuestionCard/
```

### Дерево компонентов

```
QuestionCard (widgets)
├── QuestionStem (entities/question)
│   └── TipTapRenderer (shared)
│       └── KaTeXRenderer (shared)
├── AnswerOptions (features/answer-question)
├── ActionBar
│   └── CheckButton
└── Explanation (conditional)
    └── TipTapRenderer (shared)
```

### Ответы на вопросы

**Где хранится selectedAnswer?**
Локальный стейт в `useAnswerQuestion` хуке (features слой). Не глобальный — нет смысла шарить между компонентами.

**Где хранится isChecked?**
Там же, в `useAnswerQuestion`. Связан с selectedAnswer — одна фича, один хук.

**Что сбрасывается при смене questionId?**
Всё локальное состояние: `selectedAnswer`, `isChecked`, `isSubmitting`. Реализуется через `useEffect` с зависимостью от `questionId` или через `key={questionId}` на компоненте.

**Что будет при быстрых кликах?**
Без защиты — race condition, множественные запросы, некорректный UI. Решение: `isSubmitting` флаг + disabled состояние кнопки во время запроса.

---

## Часть 2. Псевдокод логики

```
State:
  selectedAnswer: string | null = null
  isChecked: boolean = false
  isSubmitting: boolean = false

Derived:
  canCheck = selectedAnswer !== null && !isChecked && !isSubmitting
  canChangeAnswer = !isChecked
  showExplanation = isChecked && explanation !== null

onSelectAnswer(answerId):
  if (isChecked) return
  selectedAnswer = answerId

onCheckAnswer():
  if (!canCheck) return
  
  isSubmitting = true
  
  try:
    await api.submitAnswer(questionId, selectedAnswer)
    isChecked = true
  catch:
    showError("Не удалось проверить ответ")
  finally:
    isSubmitting = false

onQuestionChange(newQuestionId):
  selectedAnswer = null
  isChecked = false
  isSubmitting = false

UI Rules:
  CheckButton.disabled = !canCheck
  CheckButton.loading = isSubmitting
  AnswerOption.disabled = !canChangeAnswer
  Explanation.visible = showExplanation
```

---

## Часть 3. Edge Cases и UX

| Кейс | UI решение |
|------|------------|
| **Explanation отсутствует** | После check не показываем блок explanation вообще. Не пустой блок, а отсутствие элемента. |
| **В stem только формулы** | Нормальный рендер через KaTeXRenderer. Min-height на контейнер, чтобы не схлопывался. |
| **Очень длинный текст в stem** | Max-height + scroll внутри блока. Или collapsible с "Показать полностью". |
| **KaTeX упал с ошибкой** | ErrorBoundary ловит, показывает исходный LaTeX код как fallback: `$formula$`. Не ломает весь компонент. |
| **Смена ответа после check** | Заблокировано. `canChangeAnswer = !isChecked`. Визуально: options в disabled состоянии. |
| **Demo режим** | См. ниже |

### Demo режим — детально

```
if (isDemo):
  Explanation:
    - blur filter на контент
    - overlay с текстом: "Объяснения доступны в полной версии"
    - CTA кнопка: "Получить доступ" → переход на /pricing
  
  После N вопросов:
    - modal с ограничением
    - счётчик оставшихся вопросов в header
```

UI текст для demo:
- "Разблокируйте объяснения для глубокого понимания материала"
- Кнопка: "Попробовать бесплатно" или "Оформить подписку"
