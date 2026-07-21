# Схема Event + Skills + Units Для Нового Движка

Короткая практическая рекомендация для нового skill engine, если цель:

1. сохранить выразительность `ModiBuff`;
2. сделать удобный authoring-слой;
3. подготовить пайплайн `LLM -> DTO/json -> runtime`.

## 1. Главная идея

Хорошая базовая модель для боевого runtime:

`Unit methods -> typed events -> modifier/skill subscriptions -> effects`

А не:

`global bus -> все слушают всё`

Для core combat лучше, чтобы событие рождалось там, где реально произошла игровая операция.

## 2. Что должно быть источником событий

Источником событий должен быть `Unit`.

Именно в методах `Unit` должна жить каноническая игровая логика:

```csharp
Attack(target)
TakeDamage(value, source)
Heal(value, source)
Cast(skillId, target)
AddStatus(status)
RemoveStatus(status)
```

И именно внутри них должны подниматься typed-события.

Это хорошо совпадает с текущей моделью `ModiBuff.Units.Unit`, где callback-ы живут на юните, а modifier лишь регистрирует подписки.

## 3. Минимальный набор событий

Для первой версии движка не нужен большой event catalog.

Достаточно 8-12 базовых событий:

1. `BeforeAttack`
2. `OnAttack`
3. `WhenAttacked`
4. `AfterAttacked`
5. `HealthChanged`
6. `OnHeal`
7. `WhenHealed`
8. `OnCast`
9. `StatusAdded`
10. `StatusRemoved`
11. `ModifierApplied`
12. `ModifierRemoved`

Этого уже хватает для контр-ударов, аур, дотов, хот-эффектов, on-hit, on-cast, berserk-порогов и большинства ARPG/MOBA-like механик.

## 4. Какие триггеры делать у скиллов

Удобно разделить триггеры на 3 категории.

### 4.1 `UnitTrigger`

Триггеры от событий юнита.

Примеры:

1. `WhenAttacked`
2. `OnAttack`
3. `OnCast`
4. `WhenHealed`

Это прямой аналог `CallbackUnit`/`CallbackType`-подхода в `ModiBuff`.

### 4.2 `StatTrigger`

Триггеры от изменения численных данных.

Примеры:

1. `HealthChanged`
2. `ManaChanged`
3. `DamageChanged`

Нужны для логики вида:

1. "если здоровье упало ниже 30%";
2. "если урон был увеличен";
3. "если мана достигла 0".

### 4.3 `ModifierInternalTrigger`

Внутренние события самого modifier/skill instance.

Примеры:

1. `OnApply`
2. `OnExpire`
3. `OnTick`
4. `OnStack`
5. `OnMaxStack`

Это нужно для duration, interval, stacks, aura life cycle.

## 5. Что такое skill runtime instance

Полезно разделить authoring skill и runtime skill instance.

### Authoring

Описывает, что должно происходить:

1. какие триггеры есть;
2. какие условия есть;
3. какие действия запускаются.

### Runtime instance

Хранит состояние:

1. `source`;
2. `target` или target set;
3. таймеры;
4. стаки;
5. зарегистрированные callback-и;
6. mutable state для отдельных эффектов.

Это очень близко к тому, как `Modifier` в `ModiBuff` держит time components, stack component, target component и register/revert behavior.

## 6. Предлагаемая минимальная runtime-модель

### `Unit`

Хранит:

1. stats;
2. active modifiers/skills;
3. typed callback registries;
4. status controllers;
5. legal action checks.

### `ModifierInstance`

Хранит:

1. `modifierId`;
2. `sourceUnit`;
3. `targetUnit` или `targetGroup`;
4. duration/interval state;
5. stacks;
6. ссылки на подписки;
7. effect state.

### `Effect`

Минимально:

```csharp
interface IEffect
{
    void Apply(EffectContext context);
}
```

Если нужен remove/revert:

```csharp
interface IRevertEffect
{
    void Revert(EffectContext context);
}
```

### `EffectContext`

Минимальный контекст:

```csharp
source
target
ownerModifier
value
tags
skillId/modifierId
```

Не стоит делать один бесформенный `Dictionary<string, object>` как главный runtime-контекст.

## 7. Что компилировать из DTO

Для LLM/JSON-слоя лучше генерировать не код и не прямые delegate-ы, а декларативный контракт.

Пример:

```json
{
  "id": "thorns_basic",
  "triggers": [
    {
      "type": "unit",
      "when": "WhenAttacked",
      "conditions": ["source_in_range"],
      "actions": [
        { "type": "deal_damage", "value": 10, "target": "source" }
      ]
    }
  ]
}
```

Потом компиляция выглядит так:

1. `trigger.type + when` -> callback registration на `Unit`
2. `conditions` -> checks
3. `actions` -> effect instances
4. `duration/interval/stacks` -> time/stack components

Это хорошо маппится обратно в `ModiBuff` recipe/runtime.

## 8. Что лучше не делать

Если цель именно боевая система и authoring pipeline, лучше избегать:

1. string-based event names как основной runtime API;
2. глобального event bus для core combat;
3. прямого смешивания DTO и runtime instance;
4. "универсального события" с кучей optional полей для всех случаев;
5. вызова game effects напрямую из UI или authoring слоя.

Глобальный bus можно оставить как вторичную систему для:

1. debug;
2. analytics;
3. replay;
4. UI notifications.

Но не как основу боевой логики.

## 9. Практический вывод

Если делать свой движок рядом с идеями `ModiBuff`, то хороший минимальный путь такой:

1. `Unit` содержит typed event registries;
2. `Skill/ModifierInstance` при apply регистрирует callback-и на `Unit`;
3. при remove отписывает их;
4. события рождаются только внутри реальных методов `Unit`;
5. DTO описывает trigger + conditions + actions;
6. compiler переводит DTO в runtime primitives.

Это даст:

1. понятный combat flow;
2. удобную отладку;
3. хорошую совместимость с `ModiBuff` идеями;
4. нормальную базу для `LLM -> JSON -> runtime`.

## 10. Если брать за основу именно `ModiBuff`

То для нового authoring engine особенно полезно сохранить следующие идеи:

1. разделение `ApplyCheck` и `EffectCheck`;
2. отдельные lifecycle triggers: `Init`, `Interval`, `Duration`, `Stack`;
3. callback registration как часть apply/remove modifier-а;
4. revert/unregister на remove;
5. хранение mutable state на runtime instance, а не в authoring описании.

Это одна из самых сильных сторон текущего `ModiBuff` дизайна, и её стоит переносить, а не изобретать заново.

## 11. Как думать про instant, periodic, duration, stack

Самая частая ошибка при проектировании skill engine: делать один enum вида:

1. `Instant`
2. `Periodic`
3. `Duration`
4. `Stack`

Проблема в том, что это не взаимоисключающие типы.

На практике это разные оси поведения.

### 11.1 Правильная модель: ортогональные свойства

Эффект лучше описывать не одним `kind`, а несколькими независимыми блоками:

1. `Trigger`
2. `Execution`
3. `Lifetime`
4. `Stacking`
5. `Payload`

Примерно так:

```text
effect = when it starts
       + how often it runs
       + how long it lives
       + how stacks modify it
       + what it actually does
```

### 11.2 Что означает каждая ось

#### `Trigger`

Когда эффект запускается впервые:

1. `OnApply`
2. `OnHit`
3. `OnCast`
4. `WhenAttacked`

#### `Execution`

Как он исполняется:

1. `Once`
2. `Interval(1s)`
3. `OnExpire`
4. `OnStackChanged`

#### `Lifetime`

Сколько живет инстанс эффекта:

1. `Instant`
2. `Duration(5s)`
3. `Permanent`

#### `Stacking`

Как работают повторные наложения:

1. `NoStack`
2. `AddStack`
3. `RefreshDuration`
4. `IndependentStacks`
5. `MaxStacks(n)`

#### `Payload`

Что эффект реально делает:

1. `DealDamage`
2. `ModifyDefense`
3. `ApplyStatus`
4. `Heal`

## 12. Почему stack не должен быть отдельным типом эффекта

`Stack` это не отдельный effect kind.

Это модификатор поведения уже существующего эффекта или modifier instance.

То есть:

1. `periodic damage over time` может быть stackable;
2. `temporary defense buff` может быть stackable;
3. `instant burst` тоже может накапливать charge/state, если так задумано.

Поэтому хороший runtime думает так:

1. есть instance;
2. у instance есть lifetime;
3. у instance есть execution policy;
4. у instance есть stack policy;
5. payload использует текущее число stack-ов при расчете.

Это очень близко к модели `ModiBuff`, где `Interval`, `Duration` и `Stack` существуют совместно, а не вместо друг друга.

## 13. Практическое правило проектирования

Хорошее правило:

`Скилл` состоит не из одного эффекта, а из набора отдельных effect entries.

То есть один skill может содержать:

1. instant hit;
2. отдельный DoT modifier;
3. отдельный buff modifier.

Не надо пытаться втиснуть все в один "супер-эффект".

## 14. Как раскладывать примерный скилл

Пример:

"Нанести 5 урона, наложить стак эффекта который наносит 2 урона в секунду, и стак эффекта который увеличивает защиту на 1%"

Правильное разложение:

### 14.1 Entry 1: мгновенный hit

1. trigger: `OnCastHit` или `OnApply`
2. execution: `Once`
3. lifetime: `Instant`
4. payload: `DealDamage(5)`

### 14.2 Entry 2: stackable DoT modifier

1. trigger: `OnCastHit`
2. execution: `Interval(1s)`
3. lifetime: `Duration(Xs)`
4. stacking: `AddStack + RefreshDuration + MaxStacks?`
5. payload: `DealDamage(2 * currentStacks)` или `DealDamage(2)` per stack model

### 14.3 Entry 3: stackable defense buff modifier

1. trigger: `OnCastHit` или `OnApply`
2. execution: `ApplyOnce`, `RevertOnRemove`
3. lifetime: `Duration(Xs)` или `PermanentUntilRemoved`
4. stacking: `AddStack + RefreshDuration + MaxStacks?`
5. payload: `AddDefensePercent(1% * currentStacks)`

И это должны быть 3 отдельных runtime-узла, а не один effect object.

## 15. Два варианта интерпретации stackable periodic эффекта

Для stacked DoT есть две нормальные модели. Их важно выбрать явно.

### Модель A: один modifier instance, stacks усиливают tick

Пример:

1. 1 stack -> 2 damage/sec
2. 2 stacks -> 4 damage/sec
3. 3 stacks -> 6 damage/sec

Это проще в runtime и обычно лучше для authoring.

### Модель B: каждый stack это отдельный sub-instance

Пример:

1. первый stack живет свои 5 секунд;
2. второй stack живет свои 5 секунд;
3. каждый stack тикает отдельно.

Это сложнее, но полезно для poison/bleed-like механик.

Если нет сильной причины, лучше сначала брать модель A.

## 16. Что я бы рекомендовал для первой версии

Для первой версии движка сделать только такие правила:

1. один `ModifierInstance` может одновременно иметь `duration`, `interval` и `stacks`;
2. periodic payload читает текущее число stack-ов;
3. stat buff payload тоже читает текущее число stack-ов;
4. повторное наложение по умолчанию делает `+1 stack` и `refresh duration`;
5. instant effects не живут как instance, а исполняются сразу.

Это уже покрывает очень много реальных скиллов и хорошо маппится обратно в `ModiBuff`.

## 17. DTO-модель, которая не ломается от сочетаний

Полезная форма описания:

```json
{
  "entries": [
    {
      "trigger": "on_cast_hit",
      "execution": { "type": "once" },
      "payload": { "type": "damage", "value": 5 }
    },
    {
      "trigger": "on_cast_hit",
      "execution": { "type": "interval", "seconds": 1 },
      "lifetime": { "type": "duration", "seconds": 5 },
      "stacking": { "mode": "add", "refresh": true, "maxStacks": 10 },
      "payload": { "type": "damage", "valuePerStack": 2 }
    },
    {
      "trigger": "on_cast_hit",
      "execution": { "type": "apply_once_revert_on_remove" },
      "lifetime": { "type": "duration", "seconds": 5 },
      "stacking": { "mode": "add", "refresh": true, "maxStacks": 10 },
      "payload": { "type": "defense_percent", "valuePerStack": 1 }
    }
  ]
}
```

Такой контракт намного устойчивее, чем один enum `effectType`.

## 18. Как это уже проявляется в `ModiBuff`

В `ModiBuff.Tests/CentralizedCustomLogicTests.cs` есть хороший пример poison-подхода:

1. `.Stack(WhenStackEffect.Always)`
2. `.Effect(new PoisonDamageEffect(), EffectOn.Interval | EffectOn.Stack)`
3. `.Interval(1)`
4. `.Remove(5).Refresh()`

То есть stack, interval и duration работают одновременно на одном modifier-е.

Это хороший сигнал, что и в новом движке их тоже надо проектировать как совместимые свойства, а не как разные категории эффектов.
