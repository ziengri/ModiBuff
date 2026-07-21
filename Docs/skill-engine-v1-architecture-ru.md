# Архитектура Нового Skill Engine V1

Этот документ собирает в одну проектную архитектуру:

1. идеи из `Docs/skill-system-architecture-ru.md`;
2. выводы из `Docs/skill-engine-events-skills-units-ru.md`;
3. разбор `Docs/modibuff-recipe-components-ru.md`;
4. технический анализ `Docs/modibuff-technical-analysis-for-skill-engine-ru.md`;
5. текущие проектные мысли про `IUnit`, `SkillController`, `EffectController`, `Actions` и `Effects`.

Цель документа:

1. выбрать базовую архитектуру нового движка;
2. понять, что берем из `ModiBuff`;
3. понять, что меняем принципиально;
4. оставить после обсуждения основу, от которой можно уже идти в код.

## 1. Короткий итог

Рекомендуемая архитектура такая:

1. `Unit` остается основным источником боевых событий;
2. core движка зависит от интерфейсов, а не от конкретного игрового движка;
3. публичная authoring-модель строится как `Skill -> Entries -> Actions / AppliedEffects`;
4. `Action` и `Effect` разделяются;
5. runtime внутри компилируется в более плоскую и быструю модель, близкую к лучшим идеям `ModiBuff`;
6. persistent logic живет в `EffectInstance`, instant logic исполняется сразу через `ActionExecutor`;
7. `SkillController` и `EffectController` должны быть разными подсистемами;
8. строгая часть системы должна жить в compile/validation/runtime слоях, а не в свободном authoring-слое.

В одной фразе:

`Снаружи skill описывается как композиция trigger/condition/action/effect, внутри компилируется в строгие runtime primitives с unit-local events и stateful effect instances.`

## 2. Главные принципы

Для проекта AI Skill Crafting System рекомендую зафиксировать такие принципы.

### 2.1 Свобода снаружи, строгость внутри

Игрок может описывать навык свободно.

Но runtime не должен исполнять свободный текст, динамические делегаты или размытые структуры.

Значит нужен пайплайн:

`fantasy text -> normalized skill dto -> validated compiled skill -> runtime execution`

### 2.2 Данные отделены от runtime-логики

Навык должен иметь:

1. authoring-представление;
2. compiled-представление;
3. runtime-instance.

Нельзя смешивать их в один объект.

### 2.3 Core engine-agnostic

Ядро не должно зависеть от Godot, Unity или конкретной игры.

Интеграция с движком идет через:

1. интерфейсы юнитов;
2. интерфейсы мира;
3. adapter-слой;
4. визуальные/сетевые мосты.

### 2.4 События рождаются в gameplay methods

Не нужен глобальный event bus как основа боевой логики.

Истинные события должны возникать внутри реальных операций сущности:

1. `Attack(...)`
2. `TakeDamage(...)`
3. `Heal(...)`
4. `Cast(...)`
5. `AddStatus(...)`
6. `RemoveStatus(...)`

### 2.5 Runtime должен быть композиционным

Большинство навыков должны собираться из небольших блоков:

1. triggers;
2. conditions;
3. target resolution;
4. actions;
5. effects;
6. lifetime;
7. stacking.

## 3. Что берем из ModiBuff, а что меняем

## 3.1 Что берем почти без изменений

Из `ModiBuff` стоит перенести:

1. разделение authoring и runtime;
2. compile phase между ними;
3. `ApplyCheck` vs `EffectCheck`;
4. lifecycle triggers: `OnApply`, `OnTick`, `OnExpire`, `OnStack`, `OnRemove`;
5. callback registration как часть жизненного цикла instance-а;
6. pooled runtime instances;
7. mutable state внутри runtime instance;
8. отдельный `InitLoad`-аналог для восстановления из save/load;
9. ортогональную модель `duration + interval + stack`, а не enum с взаимоисключающими типами.

## 3.2 Что меняем принципиально

От `ModiBuff` лучше отойти в следующих местах:

1. публичный contract не должен быть завязан на heavy callback API;
2. публичная модель не должна течь внутренними runtime tags;
3. skill authoring должен быть DTO-first, а не builder-first;
4. `Skill`, `Effect`, `Action` надо развести понятнее;
5. top-level сущность должна называться `Skill`, а не `Modifier`.

## 3.3 Главный архитектурный компромисс

Не нужно выбирать строго один из двух вариантов:

1. или полностью `ModiBuff`-стиль;
2. или полностью отдельная иерархия `Skill -> Effect -> Action` без его идей.

Лучшее решение гибридное:

1. на уровне authoring и DTO делаем явное разделение `Skill / Effect / Action`;
2. на уровне compiled runtime flatten-им это в компактные trigger lanes и stateful instances, как в `ModiBuff`.

Это дает и понятность, и производительность.

## 4. Слои нового движка

Рекомендую сразу мыслить движок как 6 слоев.

### 4.1 Authoring Layer

Тут живет:

1. JSON/DTO контракт;
2. LLM output normalization;
3. редакторские схемы;
4. skill templates.

Этот слой удобный, выразительный, но не runtime-critical.

### 4.2 Validation Layer

Тут живет:

1. проверка схемы;
2. проверка поддерживаемых trigger/action/effect типов;
3. балансные ограничения;
4. whitelist допустимых комбинаций;
5. семантические проверки.

Это слой, который превращает "почти правильно" в строго валидный контракт.

### 4.3 Compilation Layer

Тут живет:

1. компиляция DTO в `CompiledSkillDefinition`;
2. нормализация conditions;
3. раскладка actions/effects по internal triggers;
4. подготовка таблиц и фабрик runtime instances;
5. предварительное связывание registries.

### 4.4 Runtime Core Layer

Тут живет:

1. `SkillEngine`;
2. `SkillController`;
3. `EffectController`;
4. `EffectInstance`;
5. `ActionExecutor`;
6. `ConditionEvaluator`;
7. pools;
8. update loop.

### 4.5 Integration Layer

Тут живет адаптация под конкретную игру или движок:

1. `UnityUnitAdapter`;
2. `GodotUnitAdapter`;
3. `TestUnit`;
4. world query adapters;
5. save/load bridges.

### 4.6 Presentation Layer (Пока не рассматриваем, не в приоритете)

Тут живет визуал и внешние реакции:

1. VFX;
2. SFX;
3. анимации;
4. HUD;
5. combat log;
6. replay/debug events.

Этот слой не должен напрямую менять боевой runtime.

## 5. Рекомендуемая доменная модель

Ниже модель, которую я рекомендую сделать базовой.

### 5.1 `SkillDefinition`

Это top-level описание навыка.

Содержит:

1. `SkillId`;
2. metadata;
3. cast policy;
4. target policy;
5. skill-level apply conditions;
6. список `SkillEntryDefinition`.

Навык не обязан быть только активным кастом.

`SkillDefinition` может описывать:

1. активную способность;
2. пассивку;
3. реакцию на событие;
4. granted skill;
5. aura-emitter;
6. статусный источник.

### 5.2 `SkillEntryDefinition`

Это минимальная логическая ветка навыка.

Один skill состоит из нескольких entries.

Каждый entry содержит:

1. trigger;
2. entry conditions;
3. target resolver;
4. execution policy;
5. либо instant actions;
6. либо `ApplyEffectAction`;
7. либо их комбинацию.

Именно `SkillEntryDefinition` должен быть базовой единицей компиляции, а не весь skill как один огромный monolith.

### 5.3 `EffectDefinition`

Это описание persistent stateful эффекта.

Он нужен, когда поведение живет дольше одного момента.

Содержит:

1. `EffectId`;
2. lifetime policy;
3. interval policy;
4. stack policy;
5. internal triggers;
6. actions на этих триггерах;
7. remove/revert policy;
8. effect-local conditions.

Примерно так:

1. `OnApply -> AddArmor(5)`
2. `OnTick -> DealDamage(2)`
3. `OnExpire -> BurstDamage(10)`
4. `OnRemove -> RevertArmor(5)`

### 5.4 `ActionDefinition`

Это самая маленькая gameplay-операция.

Action сам по себе мгновенный.

Он не живет 5 секунд и не хранит стаки.

Это делает либо `SkillEntry`, либо `EffectInstance`, который его запускает.

Примеры базовых actions:

1. `DealDamageAction`
2. `HealAction`
3. `ApplyStatusAction`
4. `ApplyEffectAction`
5. `RemoveEffectAction`
6. `ModifyResourceAction`
7. `DispelAction`
8. `GrantSkillAction`
9. `SpawnEntityAction`
10. `TriggerSkillAction`

## 6. Почему `Action` и `Effect` надо разделить

Это главное отличие, которое я рекомендую относительно исходной `ModiBuff` модели.

### 6.1 Что такое `Action`

`Action` отвечает на вопрос:

`что сейчас сделать?`

Например:

1. нанести урон;
2. вылечить;
3. применить статус;
4. выдать другой эффект.

### 6.2 Что такое `Effect`

`Effect` отвечает на вопрос:

`какое состояние или поведение должно продолжать жить после применения?`

Например:

1. горение на 5 секунд;
2. бафф брони со стаками;
3. аура вокруг носителя;
4. шипы, реагирующие на входящий удар.

### 6.3 Почему это полезно

Такое разделение сильно лучше для:

1. `LLM -> DTO`;
2. объяснимости навыка;
3. редакторского authoring;
4. балансных правил;
5. повторного использования примитивов.

### 6.4 Но runtime не должен становиться древом из ста объектов

На runtime нельзя буквально хранить сложное дерево объектов на каждый эффект.

Поэтому:

1. authoring разделяем;
2. compiled representation уплощаем;
3. runtime исполняет компактные таблицы, массивы и instances.

Это и есть правильный компромисс между твоим вариантом и `ModiBuff`-подходом.

## 7. Рекомендуемая модель `IUnit`

Твоя мысль про `IUnit` и наследуемые интерфейсы правильная по направлению, но ее нужно сделать аккуратно.

## 7.1 Что не стоит делать

Не стоит делать один гигантский `IUnit`, который требует от любой сущности вообще все:

1. mana;
2. attack;
3. heal;
4. statuses;
5. summon;
6. inventory;
7. cast;
8. threat;
9. locomotion.

Также не стоит дробить модель на слишком много микроскопических интерфейсов, которые будут постоянно кастоваться на runtime.

## 7.2 Что стоит делать

Лучше взять модель:

1. маленький базовый `IUnit`;
2. набор capability interfaces по стабильным доменам;
3. typed events на самом `IUnit`;
4. skill runtime работает через capability resolution.

### 7.3 Базовый `IUnit`

Рекомендуемый минимум:

```csharp
public interface IUnit
{
    UnitId Id { get; }
    UnitTeam Team { get; }
    bool IsAlive { get; }
    IUnitEvents Events { get; }
    ISkillController Skills { get; }
    IEffectController Effects { get; }
}
```

Важно:

`IUnit` это не место для всех боевых операций мира.

Это минимальная сущность-носитель skill runtime.

### 7.4 Capability interfaces

Рекомендую такие группы.

#### Health / damage

```csharp
public interface IDamageable
{
    DamageResult TakeDamage(in DamageRequest request);
}

public interface IHealable
{
    HealResult Heal(in HealRequest request);
}

public interface IHealthOwner
{
    float CurrentHealth { get; }
    float MaxHealth { get; }
}
```

#### Resources

```csharp
public interface IManaOwner
{
    float CurrentMana { get; }
    float MaxMana { get; }
    bool TrySpendMana(float value);
}
```

#### Stats

```csharp
public interface IStatOwner
{
    float GetStat(StatId statId);
    void AddStatModifier(in StatModifier modifier);
    void RemoveStatModifier(in StatModifier modifier);
}
```

#### Statuses

```csharp
public interface IStatusHost
{
    bool AddStatus(in StatusApplication application);
    bool RemoveStatus(StatusId statusId);
    bool HasStatus(StatusId statusId);
}
```

#### Combat actions

```csharp
public interface IAttackUnit
{
    AttackResult Attack(IUnit target, in AttackRequest request);
}

public interface ICastUnit
{
    CastResult Cast(SkillId skillId, in CastRequest request);
}
```

## 7.5 Практическая рекомендация по интерфейсам

Рекомендация такая:

1. capability interfaces оставить;
2. но группировать их по доменам, а не по каждой микровозможности;
3. `IAttacker` и `IDamageable` нормальны;
4. но избегать десятков слишком узких интерфейсов, если они не дают реальной пользы;
5. лучше 8-12 устойчивых capability interfaces, чем 40 мелких.

## 8. Event-модель

Тут рекомендация однозначная.

### 8.1 События должны жить в `Unit`

Основа:

`Unit methods -> typed events -> skill/effect subscriptions`

А не:

`global event bus -> кто угодно слушает что угодно`

### 8.2 Минимальный event catalog v1

Для первой версии хватит такого набора:

1. `BeforeAttack`
2. `OnAttack`
3. `WhenAttacked`
4. `AfterAttacked`
5. `BeforeCast`
6. `OnCast`
7. `HealthChanged`
8. `ManaChanged`
9. `StatusAdded`
10. `StatusRemoved`
11. `EffectApplied`
12. `EffectRemoved`
13. `UnitDied`

### 8.3 Внутренние события effect instance

Кроме unit events будут еще internal events самого effect instance:

1. `OnApply`
2. `OnRefresh`
3. `OnTick`
4. `OnStackChanged`
5. `OnMaxStack`
6. `OnExpire`
7. `OnRemove`
8. `OnLoadRestore`

### 8.4 Нужен ли global bus

Только как вторичный слой для:

1. UI;
2. analytics;
3. replay;
4. debug tooling.

Но не как основа combat logic.

## 9. Модель skill runtime

Нужно четко разделить несколько вещей.

### 9.1 `SkillDefinition`

Что навык должен делать.

### 9.2 `SkillRuntimeState`

Состояние навыка у конкретного владельца:

1. cooldown;
2. charges;
3. granted flags;
4. unlock state;
5. passive subscriptions.

### 9.3 `EffectInstance`

Активный persistent instance эффекта на цели.

Содержит:

1. `effectId`;
2. `sourceUnit`;
3. `ownerUnit`;
4. `target` или target set;
5. timers;
6. stacks;
7. runtime data;
8. subscription handles;
9. state for savable actions.

Это прямой наследник лучших идей `Modifier` из `ModiBuff`, но уже в более ясной доменной терминологии.

## 10. Controllers и сервисы

Тут очень важно не слепить один перегруженный controller.

## 10.1 `SkillEngine`

Глобальный runtime facade.

Отвечает за:

1. registries;
2. compiled definitions;
3. pools;
4. shared executors/evaluators;
5. обновление мира или группы юнитов.

## 10.2 `SkillController`

Живет на юните.

Отвечает за:

1. список доступных навыков;
2. cooldowns;
3. charges;
4. cast attempts;
5. skill-level checks;
6. passive skill subscriptions.

`SkillController` не должен обновлять тики всех эффектов.

## 10.3 `EffectController`

Живет на юните.

Отвечает за:

1. активные `EffectInstance`;
2. их update;
3. apply/reapply/remove;
4. refresh/stack policy;
5. dispel;
6. save/load state.

То есть именно он является наследником `ModifierController`.

## 10.4 `ActionExecutor`

Отдельный сервис, который умеет исполнять базовые actions.

Он:

1. получает `ActionDefinition`;
2. строит execution context;
3. проверяет capability interfaces;
4. вызывает нужные методы на `IUnit` или `IWorldContext`.

## 10.5 `ConditionEvaluator`

Отдельный сервис для:

1. skill conditions;
2. effect conditions;
3. event conditions;
4. value comparisons;
5. target filtering.

## 10.6 `TargetResolver`

Отдельный сервис для:

1. single target;
2. self;
3. allies;
4. enemies;
5. area;
6. cone;
7. line;
8. summon/world object targets.

Это важно не прятать внутрь каждого skill класса.

## 11. `IWorldContext` обязателен

Чтобы движок был переносимым, нужен интерфейс мира.

Примерно такой:

```csharp
public interface IWorldContext
{
    float Time { get; }
    IReadOnlyList<IUnit> QueryUnits(in TargetQuery query);
    void Spawn(in SpawnRequest request);
    void Despawn(EntityId entityId);
}
```

Без этого skill engine быстро упрется в невозможность делать:

1. AoE;
2. summon;
3. projectiles;
4. world-targeted actions;
5. aura queries.

## 12. Trigger / Condition / Action / Effect модель

Для authoring v1 рекомендую именно такую схему.

### 12.1 Trigger

Когда entry срабатывает.

Типы:

1. `CastTrigger`
2. `UnitEventTrigger`
3. `PassiveStartTrigger`
4. `ManualTrigger`
5. `EffectInternalTrigger`

### 12.2 Condition

Можно ли исполнить trigger или action.

Разделение:

1. `SkillApplyCondition`
2. `EntryCondition`
3. `EffectExecutionCondition`
4. `ActionCondition`

Это концептуальное продолжение `ApplyCheck` vs `EffectCheck`.

### 12.3 Action

Что сделать сразу.

### 12.4 Effect

Какое состояние создать, чтобы оно жило дальше.

## 13. Orthogonal model вместо enum `effectType`

Очень важно не скатиться в упрощение вида:

1. `Instant`
2. `Periodic`
3. `Duration`
4. `Stackable`

Это не типы, а разные оси поведения.

Правильная модель:

1. `Trigger`
2. `Execution`
3. `Lifetime`
4. `Stacking`
5. `Payload`

То есть один и тот же эффект может одновременно иметь:

1. `Duration(5s)`
2. `Interval(1s)`
3. `AddStack`
4. `OnApplyAction`
5. `OnTickAction`

Это один из самых ценных выводов из `ModiBuff`.

## 14. Рекомендуемая authoring-модель skill

Для DTO/JSON хороший минимальный каркас такой:

```json
{
  "id": "fire_strike",
  "cast": {
    "targeting": "enemy_unit",
    "cost": [{ "resource": "mana", "value": 15 }],
    "cooldownSeconds": 6
  },
  "entries": [
    {
      "trigger": { "type": "on_cast_success" },
      "actions": [
        { "type": "deal_damage", "value": 25, "target": "selected_target" },
        { "type": "apply_effect", "effectId": "burning_basic", "target": "selected_target" }
      ]
    }
  ],
  "effects": [
    {
      "id": "burning_basic",
      "lifetime": { "type": "duration", "seconds": 5 },
      "execution": { "type": "interval", "seconds": 1 },
      "stacking": { "mode": "add", "refreshDuration": true, "maxStacks": 5 },
      "onTick": [
        { "type": "deal_damage", "valuePerStack": 3, "target": "owner" }
      ]
    }
  ]
}
```

Это уже:

1. понятно LLM;
2. понятно редактору;
3. валидируемо;
4. компилируемо в runtime.

## 15. Как это должно работать на runtime

Ниже рекомендуемый поток исполнения.

### 15.1 Cast flow

1. `SkillController.TryCast(skillId, castRequest)`
2. `SkillApplyCondition` проверяются
3. стоимость и cooldown обрабатываются
4. `TargetResolver` находит target set
5. активируются подходящие `SkillEntry`
6. instant actions исполняются через `ActionExecutor`
7. `ApplyEffectAction` создает или переиспользует `EffectInstance`

### 15.2 Effect apply flow

1. `EffectController.Apply(effectId, target, source, data)`
2. определяется reapply policy
3. если instance уже есть, применяются `refresh/stack/reuse rules`
4. если instance нет, он арендуется из pool
5. регистрируются unit-event subscriptions
6. выполняются `OnApply` actions
7. instance начинает жить в update loop

### 15.3 Update flow

1. `EffectController.Update(delta)`
2. каждый `EffectInstance` обновляет timers
3. по `Interval` срабатывают `OnTick` actions
4. по `Duration` срабатывают `OnExpire` actions
5. remove делается deferred-safe образом

## 16. Reapply policy

Это надо сделать формальной частью модели, а не прятать в код.

Для persistent effects рекомендую поддержать:

1. `IgnoreIfExists`
2. `RefreshDuration`
3. `AddStack`
4. `AddStackAndRefresh`
5. `CreateIndependentInstance`
6. `Replace`

Это прямой логический наследник того, как `ModiBuff` ведет себя при повторном `Add(...)`.

## 17. Stacking model

Для v1 лучше брать простой, но сильный вариант.

### 17.1 Что нужно поддержать сразу

1. `maxStacks`
2. `refreshDurationOnReapply`
3. `single shared stack count`
4. `OnStackChanged`
5. `OnMaxStack`

### 17.2 Что можно отложить

1. независимые таймеры на каждый stack;
2. разные sub-instance модели на каждый stack;
3. очень сложные stack revert chains.

### 17.3 Почему так

Потому что модель:

1. `один instance`;
2. `у него есть currentStacks`;
3. actions читают `currentStacks`;

закрывает очень много механик и проще в authoring.

Это же было рекомендовано и в предыдущих docs.

## 18. Строгость системы: где она должна жить

Ты правильно пишешь, что система должна быть одновременно гибкой и строгой.

Правильный ответ:

строгость не в том, чтобы сделать 100 жестких классов вручную.

Строгость должна жить здесь:

1. в schema DTO;
2. в validator rules;
3. в compiler checks;
4. в action/effect registries;
5. в allowed combinations;
6. в runtime invariants.

Тогда снаружи система гибкая, а внутри контролируемая.

## 19. Save/Load модель

Если движок планируется как реальный gameplay runtime, save/load надо заложить сразу.

Нужно сохранять:

1. skill runtime states;
2. effect instances;
3. timers;
4. stacks;
5. mutable action/effect state;
6. granted skills;
7. id remap information.

Также нужен отдельный runtime stage:

1. `OnApply`
2. `OnLoadRestore`

Это обязательно сохранить из идей `ModiBuff`.

## 20. Визуал и презентация

Логика навыка не должна напрямую дергать:

1. VFX prefab;
2. animator;
3. sound emitter;
4. camera shake.

Вместо этого runtime должен публиковать presentation-friendly events или commands:

1. `DamageAppliedPresentationEvent`
2. `SkillCastPresentationEvent`
3. `EffectAppliedPresentationEvent`
4. `ProjectileSpawnRequested`

Это позволит одинаково использовать движок в:

1. Unity;
2. Godot;
3. headless tests;
4. серверной симуляции.

## 21. Рекомендуемая структура модулей

Рекомендую уже на уровне проекта ориентироваться на такую структуру:

```text
SkillEngine.Core/
  Abstractions/
  Runtime/
  Runtime/Controllers/
  Runtime/Instances/
  Runtime/Actions/
  Runtime/Conditions/
  Runtime/Targeting/
  Runtime/Pooling/

SkillEngine.Authoring/
  Dto/
  Validation/
  Compilation/
  Registries/

SkillEngine.Integration.Unity/
SkillEngine.Integration.Godot/
SkillEngine.TestHost/
```

## 22. Практическое решение по твоему спору

Ты обозначил два пути:

1. `эффект` как отдельная stateful сущность, а действия как базовые primitives;
2. или подход ближе к `ModiBuff`.

Моя рекомендация:

выбрать путь `1`, но runtime сделать по лучшим принципам `2`.

То есть:

1. public authoring: `Skill -> Entry -> Action / ApplyEffect`
2. persistent behavior: `EffectDefinition -> EffectInstance`
3. compiled runtime: плоские lanes и optimized components

Это лучший баланс для этого проекта.

## 23. Что я бы выбрал как baseline v1

Если зафиксировать архитектуру максимально практично, то я бы выбрал именно это.

### 23.1 Core entities

1. `IUnit`
2. `IWorldContext`
3. `SkillDefinition`
4. `SkillEntryDefinition`
5. `EffectDefinition`
6. `ActionDefinition`
7. `EffectInstance`

### 23.2 Runtime services

1. `SkillEngine`
2. `SkillController`
3. `EffectController`
4. `ActionExecutor`
5. `ConditionEvaluator`
6. `TargetResolver`

### 23.3 Must-have behavior

1. unit-local typed events;
2. duration;
3. interval;
4. stacks;
5. reapply policy;
6. save/load hooks;
7. pooling;
8. apply/effect condition split.

## 24. Открытые решения для обсуждения

Перед реализацией я бы отдельно обсудил только эти вопросы.

### 24.1 Насколько широким делать `IUnit`

Рекомендация: минимальный `IUnit` + capability interfaces.

### 24.2 Делать ли stat system частью движка

Варианты:

1. встроить минимальную stat abstraction в core;
2. оставить stat manipulation через adapter interfaces.

Для переносимости я бы выбрал второй путь с минимальным `IStatOwner`.

### 24.3 Делать ли projectiles частью skill engine

Варианты:

1. projectile как отдельный world object с собственным lifecycle;
2. projectile как чисто визуальный carrier для delayed action.

Поддержать стоит оба, но в core engine не зашивать жестко только один.

### 24.4 Нужен ли effect-local mutable custom state в v1

Ответ: да.

Без этого быстро станет трудно делать сложные реактивные механики.

## 25. Финальная рекомендация

Если принимать архитектурное решение уже сейчас, то я рекомендую зафиксировать следующее:

1. делаем `engine-agnostic core`;
2. `Unit` является источником typed events;
3. делаем `IUnit` минимальным и расширяем capability interfaces;
4. разделяем `Skill`, `Effect`, `Action` на уровне authoring и DTO;
5. `SkillController` и `EffectController` держим отдельно;
6. `EffectInstance` становится главным runtime state object;
7. compile/runtime слой строим по сильным идеям `ModiBuff`;
8. public authoring API делаем проще и строже, чем у `ModiBuff`.

## 26. Что логично делать следующим документом

После этого документа логичный следующий шаг один из двух:

1. зафиксировать `Skill DTO v1` с полями и enum-ами;
2. зафиксировать `Core interfaces v1` (`IUnit`, `IWorldContext`, `ISkillController`, `IEffectController`, events, requests, results).

Именно эти два документа уже превратят архитектуру в конкретную спецификацию на реализацию.
