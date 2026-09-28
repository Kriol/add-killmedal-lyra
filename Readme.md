# Lyra Starter Game — Custom Kill Streak HUD System

Модульное решение системы медалей и серий убийств (Kill Streak / Kill Medals) для **Lyra Starter Game** в Unreal Engine 5. 

Проект реализован с соблюдением архитектурных паттернов Lyra: динамическая регистрация UI через `UIExtensionSubsystem`, обработка событий через `GameplayMessageRouter` и оптимизированный шейдер на базе `Texture2DArray`.

---

## 📸 1. Демонстрация в игре

Система отображает текущую медаль серии и соответствующий текст поверх игрового интерфейса при совершении серии убийств.

![In-Game Kill Medal](Medal_InGame.png)

---

## 🎨 2. Граф материала (`M_KillMedal`)

Для оптимизации отрисовки используется единый мастер-материал на базе массивов текстур.

![Material Graph](M_KillMedal.png)

- **Массив текстур (`Texture2DArray`):** Все варианты медалей упакованы в один ресурс `StreakArray`.
- **Индексация слоев:** Параметр `KillStageIndex` зажимается через `Clamp` (0..4) и объединяется с UV-координатами (`TexCoord[0]`) через `Append` для выбора нужного кадра.
- **Коррекция альфа-канала:** Логика `Desaturation` $\rightarrow$ `Subtract` (`AlphaThreshold` = 0.15) $\rightarrow$ `Saturate` обеспечивает четкое маскирование прозрачности без артефактов полупрозрачных граней.

---

## ⚙️ 3. Логика компонента (`B_KillStreakComponent`)

Компонент отвечает за отслеживание убийств, фильтрацию суицидов, защиту от невалидных состояний и передачу данных в UI.

![Kill Streak Component Graph](B_KillStreakComponent.png)

- **Подписка на события:** Использование `ListenForGameplayMessages` на канале `Lyra.Elimination.Message`.
- **Защита от сработок во время смерти:**
  - При получении урона/смерти проверяется состояние `LyraHealthComponent -> Get Death State == Not Dead`.
  - Отладочный вызов `Debug_AddKill` полностью заблокирован, если игрок мертв или находится на экране ожидания респавна.
- **Регистрация UI:** Компонент регистрирует виджет в `UIExtensionSubsystem` по тегу `HUD.Slot.KillMedall` и вызывает делегат `On Kill Stage Changed` с задержкой автоматического авто-снятия (`Unregister`).

---

## 📐 4. Интеграция в HUD (`W_ShooterHUDLayout`)

Виджет не привязан намертво к HUD, а использует концепцию точки расширения (Extension Point).

![HUD Layout Extension Point](W_ShooterHUDLayout.png)

- В макет `W_ShooterHUDLayout` добавлен элемент `UIExtensionPoint_Kills`.
- Ему присвоен контекстный тег `HUD.Slot.KillMedall`, куда `B_KillStreakComponent` динамически проецирует виджет медали при активации серии.

---

## 🖥️ 5. Логика виджета (`W_KillMedal`)

Виджет принимает изменения серии и обновляет визуальное состояние.

### Подписка на события (Event Graph)
![Widget Event Graph](W_KillMeadlEventGraph.png)

- В `Event Construct` происходит получение `B_KillStreakComponent` от владельца и подписка на событие `On Kill Stage Changed`.
- В `Event Destruct` выполняется корректный сброс всех подписок (`Unbind all events`).

### Настройка и анимация (`SetupMedal`)
![Widget Setup Medal Function](W_KillMeadlSetupMedal_fun.png)

- **Dynamic Material Instance:** При изменении серии создается экземпляр `MI_UI_KillMedal`, в который передается параметр `KillStageIndex`.
- **Локализация текста:** Через узел `Select` индекс серии сопоставляется с текстовыми блоками:
  - `0` "KILL"
  - `1` "DOUBLE KILL"
  - `2` "TRIPLE KILL"
  - `3` "MULTI KILL"
  - `4` "MEGA KILL"
  
---