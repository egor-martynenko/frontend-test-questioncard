# Часть 1. Архитектура QuestionCard

## Структура компонентов

QuestionCard (container)
├── QuestionStem
├── AnswerOptions
│ └── AnswerOption
├── ActionBar
│ └── CheckAnswerButton
└── Explanation (conditional)



### Ответственность компонентов
- **QuestionCard (container)**  
  Управляет состояниями вопроса и бизнес-логикой.  
  Обрабатывает edge cases: быстрые клики пользователя, гонки API, смену вопроса.
- **QuestionStem**  
  Рендерит контент вопроса (TipTap JSON + KaTeX).  
  Полностью презентационный компонент, без состояния.
- **AnswerOptions / AnswerOption**  
  Отображает варианты ответов.  
  Отвечает за визуальные состояния: `selected`, `disabled`, `correct`, `wrong`.
- **ActionBar / CheckAnswerButton**  
  Кнопка «Проверить».  
  Дизейблится в состояниях `checking` и `checked`.
- **Explanation**  
  Отображается условно, только после проверки ответа.  
  Не влияет на логику выбора.
---

## Хранение состояния

### URL (router) 
/quiz?questionId=2
- `questionId` логично хранить в URL, так как это **состояние маршрута**
- URL является источником правды для текущего вопроса
- Обеспечивает deep link, back/forward navigation и восстановление состояния при перезагрузке
### Локальное состояние `isChecked`
(local state)
Состояния взаимодействия пользователя остаются **локальными**, 
так как они относятся только к текущему вопросу и не являются навигационными.
---

## Смена questionId
При смене `questionId` в URL сбрасывается всё локальное состояние `QuestionCard`:
- выбранный ответ
- результат проверки
- loading / error состояния
- визуальные эффекты (подсветки, дизейблы)
Также отменяются или игнорируются async-запросы предыдущего вопроса, чтобы избежать race conditions.  
URL остаётся единственным источником правды для активного вопроса.
---

## Быстрые клики пользователя
При быстрых кликах:
- UI сразу обновляется и остаётся отзывчивым
- интеракции блокируются в состояниях `checking` и `checked`
- все async-запросы привязываются к текущему `questionId`
- устаревшие ответы API игнорируются
Это предотвращает двойные сабмиты и race conditions, сохраняя устойчивость интерфейса.

# Часть 2. Псевдокод логики
STATE:
  questionId
  selectedAnswer = null
  status = 'idle'        // idle | checking | checked
  checkResult = null     // correct | wrong | null

--------------------------------
ON_ANSWER_SELECT(answerId):
  if (status != 'idle') return
    selectedAnswer = answerId

--------------------------------
CAN_CHECK_ANSWER():
  return selectedAnswer != null
    AND status == 'idle'

--------------------------------
ON_CHECK_ANSWER():
    if (!CAN_CHECK_ANSWER()) return

  status = 'checking'
  disable answer options
  disable check button

  response = API.checkAnswer(questionId, selectedAnswer)

  if (response.questionId != questionId) return   // игнорировать устаревшие данные

  checkResult = response.result
  status = 'checked'

--------------------------------
SHOULD_SHOW_EXPLANATION():
  return status == 'checked'

--------------------------------
IS_CHECK_BUTTON_DISABLED():
  return selectedAnswer == null
  OR status != 'idle'

--------------------------------
IS_ANSWER_DISABLED():
  return status != 'idle'

--------------------------------
ON_QUESTION_CHANGE(newQuestionId):
  questionId = newQuestionId

  selectedAnswer = null
  checkResult = null
  status = 'idle'

# Часть 3. Edge cases и UX

1. Explanation отсутствует
- explanation не рендерится
- показывается нейтральный текст-заглушка «Объяснение недоступно для этого вопроса»
- layout не «прыгает», отступы сохраняются

2. В stem только формулы
- формулы рендерятся через KaTeX
- центрирование по ширине контейнера
- увеличенный line-height / scale для читаемости
- отсутствие лишних текстовых отступов

3. В stem очень длинный текст
- текст скроллится вместе со страницей
- нет фиксированной высоты у QuestionStem
- кнопка «Проверить» всегда доступна (не перекрывается контентом)
- переносы строк и word-break настроены

4. KaTeX упал с ошибкой
- падение KaTeX не ломает QuestionCard
- вместо формулы отображается fallback: «Не удалось отобразить формулу»
- остальной текст вопроса остаётся видимым
- ошибка логируется (Sentry / console)

5. Пользователь пытается изменить ответ после check
- варианты ответов заблокированы
- выбранный ответ подсвечен
- показывается результат (correct / wrong)
- изменение ответа невозможно без смены вопроса

6. Пользователь в demo-режиме 
* Explanation
- explanation скрыт или заблюрен
- вместо контента — overlay
* Текст объяснения причины
- «Объяснение доступно только в полной версии»
- «Перейдите на платный план, чтобы видеть разборы решений»
* CTA
- заметная кнопка: «Открыть полный доступ»
- клик ведёт на оплату / апгрейд
- UI остаётся интерактивным, без блокировки всего экрана