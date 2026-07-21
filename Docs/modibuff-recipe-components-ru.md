# Компоненты Recipe В ModiBuff

Краткая карта текущих компонентов recipe-системы `ModiBuff`.

Основано на:

1. `ModiBuff/Core/Modifier/Creation/Recipe/ModifierRecipe.cs`
2. `ModiBuff.Units/Recipe/ModifierRecipeExtensions.cs`
3. `README.md`
4. `ModifierExamples.md`
5. тестах `ModiBuff.Tests`

Цель этого файла: сначала зафиксировать, из каких частей реально состоит recipe-модель `ModiBuff`, а потом уже по одной решать, как переносить их в новый authoring-слой.

## 1. Точка входа

### `ModifierRecipes`

Контейнер, в котором объявляются рецепты и потом создаются генераторы.

Основные методы:

```csharp
ModifierRecipe Add(string name, string displayName = "", string description = "")
void Add(string name, string displayName, string description, in ModifierGeneratorFunc createFunc,
    TagType tag = TagType.Default, int? auraId = null, object customModifierData = null)
void Register(params string[] names)
void CreateGenerators()
IModifierGenerator GetGenerator(string name)
```

Смысл:

1. `Add(...)` создает fluent recipe
2. `Add(... createFunc ...)` добавляет manual generator
3. `Register(...)` резервирует имена и `id`, важно для `ApplierEffect` и порядка ссылок между modifier-ами
4. `CreateGenerators()` финализирует setup

---

## 2. Главный Builder: `ModifierRecipe`

`ModifierRecipe` это текущий high-level DSL/builder для modifier-а.

Конструктор:

```csharp
ModifierRecipe(int id, string name, string displayName, string description,
    ModifierIdManager idManager, EffectTypeIdManager effectTypeIdManager)
```

На практике обычно создается только через `ModifierRecipes.Add(...)`.

---

## 3. Базовые Metadata И Misc Блоки

### Инстансы и аура

```csharp
ModifierRecipe InstanceStackable()
ModifierRecipe Aura(int id = 0)
```

Смысл:

1. `InstanceStackable()` разрешает несколько отдельных инстансов одного modifier-а на одной цели
2. `Aura(id)` переключает modifier в multi-target/aura режим

### Теги

```csharp
ModifierRecipe Tag(int tag)
ModifierRecipe Tag(TagType tag)
ModifierRecipe SetTag(TagType tag)
ModifierRecipe RemoveTag(TagType tag)
```

Отдельные extension helpers:

```csharp
ModifierRecipe LegalTarget(LegalTarget target)
ModifierRecipe CustomStack(CustomStackEffectOn customStackEffectOn)
```

Смысл:

1. теги управляют внутренним поведением и фильтрацией
2. `CustomStack(...)` это специальный helper вокруг `Tag(CustomStack)` + `ModifierAction(Stack, ...)`

### Custom data

```csharp
ModifierRecipe Data<T>(T data) where T : notnull
```

Позволяет прикрепить arbitrary metadata к modifier-у.

---

## 4. Lifetime И Time Components

### Интервал и длительность

```csharp
ModifierRecipe Interval(float interval)
ModifierRecipe Duration(float duration)
ModifierRecipe Remove(float duration)
ModifierRecipe RemoveApplier(float duration, ApplierType applierType, bool hasApplyChecks)
ModifierRecipe Remove(RemoveEffectOn removeEffectOn)
ModifierRecipe Refresh()
ModifierRecipe Refresh(RefreshType type)
```

Связанные enum:

```csharp
enum RefreshType
{
    Duration,
    Interval,
}

[Flags]
enum RemoveEffectOn
{
    None,
    Stack,
    CallbackUnit,
    CallbackEffect,
    CallbackUnit2,
    CallbackUnit3,
    CallbackUnit4,
    CallbackEffect2,
    CallbackEffect3,
    CallbackEffect4,
}
```

Смысл:

1. `Interval` задает периодические тики
2. `Duration` задает delayed trigger
3. `Remove(float)` чаще всего используется как remove-after-X-seconds
4. `Refresh()` сбрасывает таймеры при повторном наложении
5. `Remove(RemoveEffectOn...)` связывает удаление со stack/callback trigger-ом

---

## 5. Stack Components

### Основной метод

```csharp
ModifierRecipe Stack(WhenStackEffect whenStackEffect, int? maxStacks = null,
    int? everyXStacks = null, float? singleStackTime = null, float? independentStackTime = null)
```

Связанный enum:

```csharp
enum WhenStackEffect
{
    Always,
    OnMaxStacks,
    EveryXStacks,
}
```

Смысл параметров:

1. `whenStackEffect` когда срабатывает stack effect
2. `maxStacks` верхний лимит
3. `everyXStacks` шаг для `EveryXStacks`
4. `singleStackTime` один общий таймер на все стаки
5. `independentStackTime` отдельный таймер на каждый stack

Это один из самых сильных и уже хорошо работающих блоков `ModiBuff`.

---

## 6. Apply Checks И Effect Checks

### Базовые низкоуровневые методы

```csharp
ModifierRecipe ApplyCheck(Func<IUnit, bool> check)
ModifierRecipe ApplyCheck(ICheck check)

ModifierRecipe EffectCheck(Func<IUnit, bool> check)
ModifierRecipe EffectCheck(ICheck check)
```

### Удобные extension methods из `ModifierRecipeExtensions`

```csharp
ModifierRecipe ApplyCondition(ConditionType conditionType)
ModifierRecipe ApplyCondition(StatType statType, float statValue,
    ComparisonType comparisonType = ComparisonType.GreaterOrEqual)
ModifierRecipe ApplyCondition(LegalAction legalAction)
ModifierRecipe ApplyCondition(StatusEffectType statusEffectType)
ModifierRecipe ApplyCondition(string modifierName)

ModifierRecipe ApplyCooldown(float cooldown)
ModifierRecipe ApplyChargesCooldown(float cooldown, int charges)
ModifierRecipe ApplyChance(float chance)
ModifierRecipe ApplyCost(CostType costType, float cost)
ModifierRecipe ApplyCostPercent(CostType costType, float costPercent)

ModifierRecipe EffectCondition(ConditionType conditionType)
ModifierRecipe EffectCondition(StatType statType, float statValue,
    ComparisonType comparisonType = ComparisonType.GreaterOrEqual)
ModifierRecipe EffectCondition(LegalAction legalAction)
ModifierRecipe EffectCondition(StatusEffectType statusEffectType)
ModifierRecipe EffectCondition(string modifierName)

ModifierRecipe EffectCooldown(float cooldown)
ModifierRecipe EffectChance(float chance)
ModifierRecipe EffectCost(CostType costType, float cost)
```

Смысл:

1. `Apply*` проверяют, можно ли вообще наложить modifier
2. `Effect*` проверяют, можно ли выполнить effect trigger внутри уже существующего modifier-а

Это важное разделение, его стоит сохранить и в новом authoring-слое.

---

## 7. Effect Bindings

### Основной метод

```csharp
ModifierRecipe Effect(IEffect effect, EffectOn effectOn)
```

Главный trigger mask:

```csharp
[Flags]
enum EffectOn
{
    None = 0,
    Init = 1,
    Interval = 2,
    Duration = 4,
    Stack = 8,
    CallbackUnit = 16,
    CallbackEffect = 32,
    CallbackEffectUnits = 64,
    CallbackUnit2 = 128,
    CallbackUnit3 = 256,
    CallbackUnit4 = 512,
    CallbackEffect2 = 1024,
    CallbackEffect3 = 2048,
    CallbackEffect4 = 4096,
    CallbackEffectUnits2 = 8192,
    CallbackEffectUnits3 = 16384,
    CallbackEffectUnits4 = 32768,
}
```

Смысл:

1. effect можно повесить на `Init`, `Interval`, `Duration`, `Stack`
2. можно привязать к callback lane
3. один effect может иметь несколько `EffectOn` флагов

Именно `EffectOn` сейчас является главным внутренним glue-layer recipe-системы.

---

## 8. Modifier Actions

### Метод

```csharp
ModifierRecipe ModifierAction(ModifierAction modifierAction, EffectOn effectOn)
```

Enum:

```csharp
[Flags]
enum ModifierAction
{
    Refresh = 1,
    ResetStacks = 2,
    Stack = 4,
}
```

Смысл:

1. `Refresh` вручную обновляет interval/duration таймеры
2. `ResetStacks` сбрасывает stacks
3. `Stack` вручную триггерит stack action

Это уже отдельный слой поверх обычных `Effect`-ов.

---

## 9. Callback И Event Components

### `CallbackUnit`

```csharp
ModifierRecipe CallbackUnit<TCallbackUnit>(TCallbackUnit callbackType)
```

Основной enum для `ModiBuff.Units`:

```csharp
enum CallbackUnitType
{
    WhenAttacked,
    AfterAttacked,
    WhenKilled,
    WhenHealed,
    BeforeAttack,
    OnAttack,
    OnCast,
    OnKill,
    OnHeal,
    StrongDispel,
    StrongHit,
}
```

Это built-in unit events.

### `Callback`

```csharp
ModifierRecipe Callback<TCallback>(TCallback callbackType, UnitCallback callback)
ModifierRecipe Callback<TCallback>(params Callback<TCallback>[] callbacks)
ModifierRecipe Callback<TCallback>(TCallback callbackType, Delegate callback)
ModifierRecipe Callback<TCallback, TStateData>(TCallback callbackType,
    Func<CallbackStateContext<TStateData>> @event)
```

Это custom callback registration.

### `CallbackEffect`

```csharp
ModifierRecipe CallbackEffect<TCallbackEffect>(TCallbackEffect callbackType,
    Func<IEffect, Delegate> @event)

ModifierRecipe CallbackEffect<TCallbackEffect, TStateData>(TCallbackEffect callbackType,
    Func<IEffect, CallbackStateContext<TStateData>> @event)
```

### `CallbackEffectUnits`

```csharp
ModifierRecipe CallbackEffectUnits<TCallbackEffect>(TCallbackEffect callbackType,
    Func<IEffect, Func<IUnit, IUnit, Delegate>> @event)
```

Основной enum для callback effect событий:

```csharp
enum CallbackType
{
    Dispel,
    StrongDispel,
    PoisonDamage,
    CurrentHealthChanged,
    DamageChanged,
    StrongHit,
    StatusEffectAdded,
    StatusEffectRemoved,
    OnCast,
    Update,
}
```

Это самый сложный и самый проблемный слой текущего API.

Для будущего authoring v1 логично сразу считать его advanced-зоной и не тащить в основной contract все custom варианты.

---

## 10. Dispel И Remove Helpers

### Dispel

```csharp
ModifierRecipe Dispel(DispelType dispelType = DispelType.Basic)
```

Смысл:

1. регистрирует modifier как dispellable
2. добавляет внутренний `DispelRegisterEffect`
3. связывается с lifecycle modifier-а

### Remove effect

`Remove(...)` в recipe это не просто флаг, а отдельный внутренний `RemoveEffect` wrapper.

Это важно, потому что удаление в `ModiBuff` уже является отдельной runtime-концепцией.

---

## 11. Основные Effect-Классы

Ниже не все эффекты в проекте, а те, что чаще всего встречаются в `README`, `ModifierExamples` и тестах.

### `DamageEffect`

```csharp
DamageEffect(float damage, bool valueIsRevertible = false,
    StackEffectType stackEffect = StackEffectType.Effect, float? stackValue = null,
    Targeting targeting = Targeting.TargetSource)

DamageEffect SetMetaEffects(params IMetaEffect<float, float>[] metaEffects)
DamageEffect SetPostEffects(params IPostEffect<float>[] postEffects)
```

### `HealEffect`

```csharp
HealEffect(float heal, HealEffect.EffectState effectState = HealEffect.EffectState.None,
    StackEffectType stack = StackEffectType.Effect, float? stackValue = null,
    Targeting targeting = Targeting.TargetSource)

HealEffect SetMetaEffects(params IMetaEffect<float, float>[] metaEffects)
HealEffect SetPostEffects(params IPostEffect<float>[] postEffects)
```

### `AddDamageEffect`

```csharp
AddDamageEffect(float damage, EffectState effectState = EffectState.None,
    StackEffectType stackEffect = StackEffectType.Effect, float? stackValue = null,
    Targeting targeting = Targeting.TargetSource)
```

### `StatusEffectEffect`

```csharp
StatusEffectEffect(StatusEffectType statusEffectType, float duration, bool revertible = false,
    StackEffectType stackEffect = StackEffectType.Effect, float? stackValue = null)
```

### `ApplierEffect`

```csharp
ApplierEffect(string modifierName, ApplierType? applierType = null,
    bool hasApplyChecks = false, Targeting targeting = Targeting.TargetSource)

ApplierEffect SetMetaEffects(params IMetaEffect<int, int>[] metaEffects)
```

Это главный built-in способ строить multi-modifier skills.

### Другие часто используемые

```csharp
PoisonDamageEffect(...)
DispelStatusEffectEffect(StatusEffectType statusEffectType)
AttackActionEffect(Targeting targeting = Targeting.TargetSource)
HealActionEffect()
CastActionEffect(string modifierName)
```

---

## 12. Meta Effects

`MetaEffect` меняет значение основного effect-а до применения.

Часто встречающиеся классы:

```csharp
StatPercentMetaEffect(StatType statType, Targeting targeting = Targeting.TargetSource)
ReverseValueMetaEffect()
MultiplyValueMetaEffect(float multiplier)
AddValueMetaEffect(float value)
AddValueBasedOnStatDiffMetaEffect(StatType statType, Targeting targeting = Targeting.TargetSource)
LegalActionMetaEffect(float multiplier, LegalAction legalAction, bool inverted,
    Targeting targeting = Targeting.TargetSource)
DynamicEffectBasedOnManaSpentMetaEffect((float spentMana, float multiplier)[] values)
ModifierIdBasedOnUnitTypeMetaEffect((UnitType unitType, string modifierName)[] values)
```

Смысл:

1. value scaling
2. value inversion
3. conditional multipliers
4. выбор modifier-а для `ApplierEffect`

---

## 13. Post Effects

`PostEffect` выполняется после основного effect-а и обычно использует его результат.

Часто встречающиеся классы:

```csharp
LifeStealPostEffect(float lifeStealPercent, Targeting targeting = Targeting.TargetSource)
DamagePostEffect(Targeting targeting = Targeting.TargetSource)
AddDamageOnValuePostEffect(Targeting targeting)
AddDamageOnKillPostEffect(float damage, Targeting targeting = Targeting.TargetSource)
HealFromPoisonStacksPostEffect(...)
```

Часто используются через:

```csharp
effect.SetPostEffects(...)
```

---

## 14. Effect-Local Conditions

Кроме `ApplyCondition` и `EffectCondition`, в `ModiBuff.Units` есть еще conditions на уровне конкретного effect/meta/post.

База:

```csharp
abstract record Condition(Targeting Targeting = Targeting.TargetSource)
```

Частые реализации:

```csharp
AndCondition(params ICondition[] conditions)
OrCondition(params ICondition[] conditions)
ValueFull(StatTypeCondition statType, bool invert = false)
ValueLow(StatTypeCondition statType, bool invert = false)
ValueComparison(...)
ValueComparisonPercent(...)
StatusEffectCond(StatusEffectType statusEffectType, ...)
DebuffEffectCond(...)
LevelCond(...)
```

Это отдельный слой условий, который сейчас живет внутри effect ecosystem, а не на уровне `ModifierRecipe`.

---

## 15. Вспомогательные Enum И Параметры

### `Targeting`

```csharp
enum Targeting
{
    TargetSource,
    SourceTarget,
    TargetTarget,
    SourceSource,
}
```

Определяет, кто становится финальной целью и источником effect-а.

### `EffectState`

```csharp
[Flags]
enum EffectState
{
    None,
    ValueIsRevertible = 1,
    IsRevertible = 2,
    IsTogglable = 4,
}
```

Используется в эффектах вроде `AddDamageEffect`.

Важно: у `HealEffect` свой отдельный nested enum `HealEffect.EffectState`.

### `StackEffectType`

Используется внутри stack-aware effect-ов вроде `DamageEffect`, `AddDamageEffect`, `StatusEffectEffect`.

Нужен для комбинаций вроде:

1. просто триггерить effect на stack
2. добавлять значение на stack
3. скейлить значение от количества stack-ов

---

## 16. Что Уже Видно Как Отдельные Компоненты Для Будущего Разбора

Если разбивать recipe-систему на будущие части rewrite, то сейчас видны такие отдельные блоки:

1. `ModifierRecipes` как registry/setup слой
2. `ModifierRecipe` как fluent builder
3. identity и metadata
4. lifetime
5. stacking
6. apply checks
7. effect checks
8. `EffectOn` binding model
9. обычные effects
10. `ModifierAction`
11. `ApplierEffect`
12. callback unit events
13. callback effect events
14. custom callbacks и stateful callbacks
15. dispel/remove model
16. tags и legal target
17. custom data
18. meta effects
19. post effects
20. effect-local conditions

Это уже хороший список для следующего шага: разбирать, какие из этих блоков сохраняем почти как есть, а какие нужно перепроектировать в новом authoring-слое.
