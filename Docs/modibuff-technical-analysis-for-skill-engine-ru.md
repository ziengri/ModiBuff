# Технический Анализ ModiBuff Для Проектирования Движка Навыков

Этот документ фиксирует, как реально устроен `ModiBuff` на уровне кода, какие у него основные компоненты, как они взаимодействуют во время выполнения, и какие архитектурные идеи особенно полезны для переноса в новый `skill engine`.

Источник анализа:

1. `README.md`
2. `ModifierExamples.md`
3. `Docs/modibuff-recipe-components-ru.md`
4. `Docs/skill-system-architecture-ru.md`
5. `Docs/skill-engine-events-skills-units-ru.md`
6. runtime-код `ModiBuff`, `ModiBuff.Units`, `ModiBuff.Extensions.*`
7. граф проекта через `MCP codebase-memory`

## 1. Что такое ModiBuff по факту

`ModiBuff` это не просто система баффов/дебаффов в узком смысле.

На практике это:

1. декларативный DSL для описания игровых модификаторов;
2. компилятор `recipe -> generator -> runtime instance`;
3. высокопроизводительный pooled runtime без GC в горячем цикле;
4. система жизненного цикла эффектов: `Init`, `Interval`, `Duration`, `Stack`, callback-триггеры;
5. механизм подписки модификаторов на события юнитов;
6. каркас для кастов, аур, стаков, состояний, dispel, cost/cooldown/chance/checks;
7. частично сериализуемая authoring- и runtime-модель.

Ключевая идея библиотеки:

`authoring описывает поведение декларативно -> runtime компилирует это в компактные компоненты -> юниты исполняют модификаторы как набор примитивов`

Это уже очень близко к архитектуре движка навыков, где текстовое описание или JSON потом превращаются в executable primitive graph.

## 2. Слои проекта

По структуре репозитория проект делится на несколько слоев.

### 2.1 `ModiBuff`

Это ядро.

Содержит:

1. `ModifierRecipe` и `ModifierRecipes`;
2. генерацию модификаторов;
3. runtime-классы `Modifier`, `ModifierController`, `ModifierPool`;
4. time/stack/target components;
5. effect interfaces и infra вокруг callback/register/remove/revert;
6. базовые unit interfaces `IUnit`, `IModifierOwner`, `IModifierApplierOwner`.

Это engine-agnostic слой.

### 2.2 `ModiBuff.Units`

Это opinionated gameplay implementation поверх ядра.

Содержит:

1. готовый `Unit`;
2. встроенные stats и combat methods;
3. status effect controllers;
4. конкретные эффекты: `DamageEffect`, `HealEffect`, `AddDamageEffect`, `StatusEffectEffect` и т.д.;
5. recipe extensions: `ApplyCondition`, `ApplyCooldown`, `EffectChance`, `CallbackUnit(...)` и т.д.;
6. event/callback runtime для боевых действий.

Если `ModiBuff` это core runtime primitives, то `ModiBuff.Units` это reference combat model.

### 2.3 `ModiBuff.Extensions.Serialization.Json`

Тонкий слой JSON-сериализации save/load через `System.Text.Json`.

### 2.5 `ModiBuff.Examples`

Примеры использования и минимальный игровой bootstrap.

### 2.6 `ModiBuff.Tests`

Большой тестовый набор. По названиям тестов видно, какие блоки считаются ключевыми:

1. duration;
2. cooldown;
3. cost;
4. chance;
5. callbacks;
6. aura;
7. stack;
8. modifierless effects;
9. serialization data;
10. centralized custom logic.

Для переноса в новый движок это хороший список обязательных capability areas.

## 3. Главная архитектурная ось

Весь проект держится на одной важной цепочке:

`ModifierRecipe -> ModifierRecipeData -> ModifierGenerator -> pooled Modifier -> ModifierController on Unit`

Это главный pipeline системы.

### 3.1 Authoring слой

Автор описывает поведение через fluent builder:

```csharp
Add("DoT")
    .Interval(1)
    .Effect(new DamageEffect(2), EffectOn.Interval)
    .Remove(5)
    .Refresh();
```

### 3.2 Compilation слой

Рецепт не исполняется напрямую.

Он сначала:

1. валидируется;
2. конвертируется в `ModifierRecipeData`;
3. превращается в `ModifierGenerator`;
4. generator знает, как быстро создавать runtime instances.

### 3.3 Runtime слой

Во время игры `ModifierController` не интерпретирует fluent DSL повторно.

Он:

1. берет `id` модификатора;
2. достает заготовленный generator через pool;
3. арендует `Modifier` из `ModifierPool`;
4. настраивает target/source;
5. запускает `Init`, `Stack`, `Refresh` и дальнейший `Update`.

Это важное отличие от многих data-driven систем: декларативность есть на authoring-слое, но runtime после компиляции работает как набор простых компонентов и массивов.

## 4. Authoring модель: `ModifierRecipes` и `ModifierRecipe`

### 4.1 `ModifierRecipes` как registry и build-container

`ModifierRecipes` это центральный контейнер рецептов.

Его обязанности:

1. хранить recipes и manual generators;
2. выдавать уникальные `id` через `ModifierIdManager`;
3. уметь предварительно `Register(...)` имена;
4. финализировать setup через `CreateGenerators()`;
5. хранить метаданные модификаторов: `ModifierInfo`, `TagType`, `AuraId`, custom data.

Важно: именно `CreateGenerators()` завершает authoring phase. До него recipes это только описание.

Это уже очень похоже на стадию `compile/build asset database` в полноценном движке навыков.

### 4.2 `ModifierRecipe` как high-level DSL

`ModifierRecipe` это основной fluent builder.

Он хранит практически весь authoring graph будущего модификатора:

1. metadata;
2. список `EffectWrapper`;
3. stack policy;
4. time policy;
5. apply/effect checks;
6. callback registrations;
7. remove/dispel behavior;
8. save instructions.

По сути `ModifierRecipe` это in-memory AST/IR для модификатора.

Это важное наблюдение для нового движка:

не нужно воспринимать `ModifierRecipe` просто как builder API;
на самом деле это промежуточное структурированное представление навыка.

### 4.3 Что реально умеет recipe DSL

DSL покрывает несколько ортогональных осей поведения.

#### A. Metadata и identity

1. `name`, `displayName`, `description`;
2. `id`;
3. `Tag(...)`, `SetTag(...)`, `RemoveTag(...)`;
4. `Data<T>(...)`;
5. `Aura(...)`;
6. `InstanceStackable()`.

#### B. Time / lifetime

1. `Interval(float)`;
2. `Duration(float)`;
3. `Remove(float)`;
4. `Refresh()` / `Refresh(RefreshType)`.

#### C. Stack policy

1. `Stack(WhenStackEffect, ...)`;
2. single stack timer;
3. independent stack timers;
4. max stacks;
5. every X stacks.

#### D. Apply gating

1. `ApplyCheck(...)`;
2. `ApplyCondition(...)`;
3. `ApplyCooldown(...)`;
4. `ApplyChargesCooldown(...)`;
5. `ApplyChance(...)`;
6. `ApplyCost(...)`.

#### E. Effect gating

1. `EffectCheck(...)`;
2. `EffectCondition(...)`;
3. `EffectCooldown(...)`;
4. `EffectChance(...)`;
5. `EffectCost(...)`.

#### F. Effect binding

1. `Effect(IEffect, EffectOn)`;
2. `ModifierAction(...)`;
3. `Dispel(...)`;
4. callbacks.

#### G. Event integration

1. `CallbackUnit(...)`;
2. `Callback(...)`;
3. `CallbackEffect(...)`;
4. `CallbackEffectUnits(...)`.

Именно эта ортогональность делает `ModiBuff` сильной базой для skill engine.

Навык здесь не один монолитный тип, а композиция нескольких независимых behavioral axes.

## 5. `EffectOn` как главный внутренний glue-layer

Один из самых важных элементов проекта это `EffectOn`.

Он связывает together:

1. authoring DSL;
2. runtime arrays эффектов;
3. callback lanes;
4. remove/action/stack semantics.

Основные значения:

1. `Init`
2. `Interval`
3. `Duration`
4. `Stack`
5. `CallbackUnit`
6. `CallbackEffect`
7. `CallbackEffectUnits`
8. дополнительные lanes `2/3/4`

Что это значит архитектурно:

`ModiBuff` не хранит skill как дерево узлов в runtime.
Он нормализует описание до набора trigger channels, у каждого из которых есть массив эффектов.

То есть внутренне структура ближе к:

```text
modifier instance = {
  init effects[]
  interval effects[]
  duration effects[]
  stack effects[]
  callback lane 1 effects[]
  callback lane 2 effects[]
  ...
}
```

Для нового движка это очень сильная идея, если нужна высокая производительность.

## 6. Компиляция рецепта: как DSL становится runtime

### 6.1 `ModifierRecipe.CreateModifierGenerator()`

Это центральная compile step.

На этой стадии recipe:

1. валидируется;
2. финализирует remove wrapper;
3. автоматически выставляет внутренние `TagType` флаги;
4. выводит `DispelType` на основе используемых trigger-ов;
5. собирает `ModifierRecipeData`.

Важно, что часть тегов вычисляется автоматически, а не задается автором напрямую:

1. `IsInit`
2. `IsStack`
3. `IsRefresh`
4. `IsInstanceStackable`
5. `IsAura`

Это хороший паттерн для нового движка: системные runtime-флаги должны выводиться из декларации, а не заполняться вручную пользователем или LLM.

### 6.2 `ModifierRecipeData` как compiled descriptor

`ModifierRecipeData` это уже не fluent API, а компактная структурированная форма для generator-а.

Он содержит:

1. `Id`, `Name`;
2. массивы и списки wrappers;
3. apply/effect checks;
4. `IsAura`, `Tag`;
5. `Interval`, `Duration`, refresh flags;
6. stack config.

Для skill engine это прямой аналог `CompiledSkillDefinition`.

### 6.3 `ModifierGenerator`

`ModifierGenerator` это фабрика runtime instances.

Он:

1. раскладывает checks по типам интерфейсов;
2. создает `ModifierEffectsCreator`;
3. знает, сколько time components нужно;
4. умеет создавать `Modifier` и `ModifierCheck`.

Ключевая идея: тяжелая подготовка выносится в compile/setup phase, а runtime creation остается максимально дешевым.

## 7. Runtime объект: `Modifier`

`Modifier` это runtime instance конкретного активного эффекта/скилла на цели.

Он содержит:

1. identity: `Id`, `GenId`, `Name`;
2. `InitComponent`;
3. массив `ITimeComponent[]`;
4. `StackComponent`;
5. `ModifierCheck` для effect-level gating;
6. `ITargetComponent`;
7. `ISetDataEffect[]`;
8. `EffectStateInfo` для UI/debug;
9. `EffectSaveState` для сохранения mutable effect state.

Важно понимать смысл `GenId`:

1. `Id` это тип модификатора;
2. `GenId` это уникальный id runtime-инстанса данного типа.

Это особенно важно для `instance stackable` модификаторов.

### 7.1 Runtime методы `Modifier`

Главные операции:

1. `UpdateSingleTargetSource(...)` / `UpdateTargets(...)`
2. `Init()`
3. `InitLoad()`
4. `Update(delta)`
5. `Refresh()`
6. `Stack()`
7. `ResetStacks()`
8. `SetData(...)`
9. `SaveState()` / `LoadState(...)`
10. `ResetState()`

Это уже полноценный lifecycle state machine runtime object-а.

## 8. Component-based runtime внутри `Modifier`

Хотя внешне система выглядит как fluent recipe API, внутри runtime она реально component-based.

### 8.1 `InitComponent`

Отвечает за init-phase.

Особенности:

1. отделяет `IRegisterEffect` от обычных эффектов;
2. сначала регистрирует callbacks/subscriptions;
3. потом прогоняет `ModifierCheck`, если он есть;
4. потом исполняет обычные init effects;
5. отдельно имеет `InitLoad()` для сценария загрузки состояния, где нужно зарегистрировать callbacks, но не переигрывать геймплейный эффект.

Это очень важная идея для нового движка:

`OnApply` и `OnLoadRestore` это не одно и то же.

### 8.2 `IntervalComponent`

Отвечает за периодические тики.

Особенности:

1. хранит timer;
2. поддерживает refresh;
3. может получать custom interval через `ModifierData`;
4. использует `ModifierCheck.CheckUse(...)` перед тиком;
5. умеет работать и с single-target, и с multi-target.

### 8.3 `DurationComponent`

Отвечает за delayed one-shot trigger по истечении времени.

Чаще всего он используется для remove, но формально это общий delayed trigger.

### 8.4 `StackComponent`

Это один из самых сильных блоков всей системы.

Он хранит:

1. текущее число stack-ов;
2. max stacks;
3. `WhenStackEffect` policy;
4. single stack timer;
5. independent per-stack timers;
6. stack revert effects;
7. stack mutable state reset logic.

Ключевые свойства:

1. stack в `ModiBuff` это не отдельный тип эффекта, а orthogonal runtime behavior;
2. stack effect может менять state, триггерить payload или оба варианта сразу;
3. истечение stack timer может откатывать накопленный effect state через `IStackRevertEffect`.

Это практически готовая модель для нового skill engine.

### 8.5 `ITargetComponent`

Есть две реализации:

1. `SingleTargetComponent`
2. `MultiTargetComponent`

Переключение зависит от того, aura это или обычный modifier.

Это простой, но важный слой абстракции: effect payload не обязан знать, single target или multi target у него снаружи, runtime сам передаст корректный target set.

## 9. Как создаются и раскладываются эффекты

### 9.1 `EffectWrapper`

`EffectWrapper` решает вопрос shared definition vs per-instance mutable state.

Его логика:

1. если effect stateless, используется один shared instance;
2. если effect mutable и умеет `IShallowClone`, wrapper создаст clone на runtime instance;
3. после сборки runtime `Reset()` сбрасывает clone.

Это критически важно.

Благодаря этому authoring object может быть один, а runtime mutable state остается локальным для инстанса модификатора.

Для нового движка это нужно сохранять почти без изменений.

### 9.2 `ModifierEffectsCreator`

Это один из самых важных внутренних классов.

Он делает сразу несколько вещей:

1. группирует effects по `EffectOn`;
2. собирает arrays для `Init`, `Interval`, `Duration`, `Stack`, callback lanes;
3. выделяет `IRevertEffect[]`;
4. выделяет `ISetDataEffect[]`;
5. выделяет savable effects и effects with state info;
6. прокидывает callback effect arrays в special register effects;
7. связывает `RemoveEffect` с revertible effects;
8. связывает dispel register effect с remove effect;
9. сбрасывает временные clones wrappers после сборки.

По сути это low-level linker/modifier runtime assembler.

Это очень полезный паттерн для skill engine:

нужен отдельный слой, который из декларации собирает уже сшитые runtime structures.

## 10. Effect model

### 10.1 Базовый контракт

Минимальный интерфейс крайне простой:

```csharp
public interface IEffect
{
    void Effect(IUnit target, IUnit source);
}
```

Это сильная сторона `ModiBuff`: runtime payload минимален.

Но вокруг него есть набор специальных capability-интерфейсов.

### 10.2 Capability-based effect design

Один effect может дополнительно реализовывать:

1. `IStackEffect`
2. `IStackRevertEffect`
3. `IRevertEffect`
4. `ISetDataEffect`
5. `IMutableStateEffect`
6. `IStateReset`
7. `ICallbackEffect`
8. `IMetaEffectOwner`
9. `IPostEffectOwner`
10. `IEffectStateInfo`
11. `ISavableEffect`
12. `ISaveableRecipeEffect`

Архитектурно это означает, что effect в `ModiBuff` это не просто payload function.

Это extension point, который может участвовать:

1. в stacking;
2. в revert/remove;
3. в serialization;
4. в UI/debug introspection;
5. в callback integration;
6. в meta/post processing chain.

### 10.3 Пример: `DamageEffect`

`DamageEffect` хорошо показывает философию системы.

Он одновременно:

1. наносит урон;
2. умеет накапливать дополнительное значение через stack state;
3. умеет callback-trigger behavior;
4. поддерживает meta effects;
5. поддерживает post effects;
6. поддерживает conditions;
7. умеет сохранять runtime state;
8. умеет сохранять recipe state.

Это очень выразительно, но также показывает и один риск системы:

часть concrete effect-ов берет на себя слишком много разных ролей.

Для нового skill engine лучше сохранить composability, но сильнее разделить:

1. payload;
2. modifiers/meta;
3. persistence;
4. conditions.

## 11. Checks как отдельный слой

`ModiBuff` очень правильно разделяет:

1. `ApplyCheck` и
2. `EffectCheck`.

Это одна из лучших архитектурных идей проекта.

### 11.1 `ApplyCheck`

Проверяет, можно ли вообще наложить modifier.

Примеры:

1. хватает ли маны;
2. прошел ли шанс;
3. нет ли кулдауна;
4. удовлетворяет ли цель условиям.

### 11.2 `EffectCheck`

Проверяет, можно ли выполнить конкретный effect trigger уже внутри существующего modifier instance.

Примеры:

1. можно ли сработать этому тику;
2. не в кулдауне ли effect trigger;
3. выполняются ли условия на текущем состоянии юнита.

### 11.3 Почему это важно

Это дает корректную семантику для skill engine.

Иначе часто смешиваются два разных вопроса:

1. можно ли применить instance;
2. можно ли исполнить действие instance.

Для `LLM -> DTO -> runtime` это distinction обязательно нужно сохранить.

## 12. `ModifierController`: runtime container на юните

`ModifierController` это основной менеджер активных modifier instances на одном `IUnit`.

Его обязанности:

1. хранить активные modifiers;
2. обновлять их каждый frame/tick;
3. добавлять новые modifiers;
4. обрабатывать refresh/stack semantics при повторном наложении;
5. удалять modifiers;
6. поддерживать dispel;
7. отдавать references для UI/debug/save.

### 12.1 Поведение при повторном применении

Если modifier не `InstanceStackable`, то повторное `Add(id, target, source)` не всегда создает новый instance.

Вместо этого controller:

1. находит существующий modifier;
2. обновляет source;
3. при необходимости накладывает новые data;
4. при `IsInit` снова вызывает `Init()`;
5. при `IsRefresh` делает `Refresh()`;
6. при `IsStack` делает `Stack()`;
7. триггерит `UseScheduledCheck()`.

Это очень важная runtime semantics.

Для нового движка это можно описывать как `reapply policy`.

### 12.2 Instance stackable режим

Если есть `TagType.IsInstanceStackable`, controller не объединяет instance-ы, а создает отдельные runtime объекты.

Это полезно для:

1. нескольких DoT от разных источников;
2. уникально откатываемых баффов;
3. independent timers per application.

### 12.3 Remove и deferred cleanup

Удаление не всегда делается мгновенно в том же месте, где сработал `RemoveEffect`.

Есть паттерн `PrepareRemove(...) -> actual Remove(...) later`.

Это защищает систему от опасного изменения коллекции активных modifiers прямо во время итерации/update/callback chain.

Для skill engine это важная часть execution safety.

## 13. `ModifierPool`: performance core

В README это заявлено как одна из главных особенностей, и код это подтверждает.

`ModifierPool`:

1. держит отдельный пул на каждый modifier id;
2. отдельно пулит `ModifierCheck` для apply-check appliers;
3. заранее аллоцирует instances через generators;
4. при `Rent` просто выдает готовый object;
5. при `Return` вызывает `ResetState()` и кладет instance обратно.

### Почему это важно

Это не только оптимизация.

Это меняет требования к архитектуре:

1. runtime instance должен уметь полностью reset-иться;
2. mutable state должен жить внутри controllable components/effects;
3. registration/revert должны быть симметричны;
4. переходы жизненного цикла должны быть детерминированы.

Если новый движок тоже планирует high-frequency combat и большое число активных skill instances, этот подход стоит повторить.

## 14. `ModifierControllerPool`

В `ModiBuff.Units.Unit` пулится не только `Modifier`, но и сами контроллеры:

1. `ModifierController`
2. `ModifierApplierController`

Это означает, что система ориентирована не просто на много эффектов, а на массовое создание/сброс юнитов тоже.

Для skill engine в изоляции это не обязательно, но для wave-based / roguelike / ARPG / auto-battler сценариев это полезно.

## 15. Applier модель

Кроме обычных modifier instances есть отдельный слой `ModifierApplierController`.

Это механизм для логики вида:

1. модификатор добавляет владельцу возможность кастовать другой модификатор;
2. атака владельца автоматически навешивает другой модификатор;
3. есть checked и non-checked applier-ы.

С точки зрения skill engine это уже почти `granted ability` / `passive trigger grant` system.

Полезная идея:

эффект может не только менять статы или наносить урон, но и выдавать владельцу новые executable actions.

## 16. Callback/event модель

Это одна из самых важных частей проекта для навыков.

### 16.1 Где живут события

В `ModiBuff.Units` события рождаются в `Unit`.

Примеры методов:

1. `PreAttack`;
2. `Attack`;
3. `TakeDamage`;
4. `Heal`;
5. `TryCast`;
6. `AddDamage`;
7. `Dispel`;
8. `StrongDispel`.

То есть источник событий здесь не глобальный bus, а канонические методы gameplay-сущности.

Это очень правильное решение и его обязательно стоит переносить.

### 16.2 Какие события есть

Есть два основных семейства.

#### `CallbackUnitType`

Это готовые unit event lanes, куда регистрируются массивы `IEffect`.

Примеры:

1. `WhenAttacked`
2. `AfterAttacked`
3. `WhenKilled`
4. `WhenHealed`
5. `BeforeAttack`
6. `OnAttack`
7. `OnCast`
8. `OnKill`
9. `OnHeal`
10. `StrongDispel`
11. `StrongHit`

#### `CallbackType`

Это более общий delegate-based callback слой.

Примеры:

1. `Dispel`
2. `StrongDispel`
3. `PoisonDamage`
4. `CurrentHealthChanged`
5. `DamageChanged`
6. `StrongHit`
7. `StatusEffectAdded`
8. `StatusEffectRemoved`
9. `OnCast`
10. `Update`

### 16.3 Как modifier подписывается на события

Через специальные register effects:

1. `CallbackUnitRegisterEffect<T>`
2. `CallbackRegisterEffect<T>`
3. `CallbackEffectRegisterEffect<T>`
4. `CallbackEffectRegisterEffectUnits<T>`

Они вешаются на `EffectOn.Init`, а затем `InitComponent` сначала регистрирует их, а не исполняет как обычный payload.

При remove они отписываются через `IRevertEffect`.

Это очень сильный паттерн:

`подписка на события это тоже эффект жизненного цикла modifier-а`

Для нового движка навыков это одна из ключевых идей, которую не надо терять.

### 16.4 Ограничение текущего подхода

Слой callback-ов в `ModiBuff` очень мощный, но API местами перегружен:

1. много разных callback registration variants;
2. несколько lane-ов `CallbackUnit2/3/4`, `CallbackEffect2/3/4`;
3. часть contract-ов сложна для чистого JSON/DTO authoring;
4. custom delegate signatures ухудшают переносимость.

Для нового skill engine лучше:

1. оставить typed events;
2. ограничить публичный authoring contract меньшим числом trigger categories;
3. custom callback signatures увести в advanced/internal layer.

## 17. `Unit` как reference combat runtime

`ModiBuff.Units.Unit` показывает, как библиотека предполагает интеграцию в игру.

Этот класс одновременно:

1. хранит stats;
2. реализует боевые методы;
3. содержит `ModifierController` и `ModifierApplierController`;
4. держит status effect controllers;
5. хранит event registries;
6. управляет legal actions;
7. содержит aura targets и level data.

Это не minimal entity abstraction, а довольно жирная reference implementation.

### 17.1 Сильная сторона `Unit`

Главная сильная сторона не в количестве интерфейсов, а в том, что события рождаются внутри real gameplay methods.

Например:

1. `Attack()` сначала проверяет legal action;
2. затем применяет attack modifiers;
3. затем триггерит `OnAttack`;
4. затем наносит урон цели;
5. затем при необходимости триггерит `OnKill`.

Такой flow хорошо поддается reasoning и отладке.

### 17.2 Event recursion guard

В `UnitCallbacks.cs` есть счетчики recursion/event depth и `MaxEventCount`.

Это защита от бесконечных цепочек, когда callback вызывает действие, которое вызывает callback снова.

Для skill engine это крайне важно. Любая expressive event-driven система рано или поздно сталкивается с:

1. re-entrant execution;
2. looped triggers;
3. `on-hit -> apply -> callback -> cast -> on-hit` цепочками.

Такую защиту или более явную execution budget policy нужно иметь с самого начала.

## 18. Status effects и legal actions

В `ModiBuff.Units` есть отдельные контроллеры статус-эффектов и legal action checks.

На практике это дает два полезных шаблона:

1. combat verbs (`Act`, `Cast`, `Attack`) проверяются централизованно;
2. status changes могут сами становиться источниками событий.

Это полезно для skill engine, потому что статус-эффекты перестают быть просто числами или флагами и становятся частью event model.

## 19. Meta effects и post effects

`ModiBuff.Units` поддерживает два слоя поверх основного payload-а.

### Meta effects

Меняют значение до применения эффекта.

Примеры:

1. additive modifiers;
2. multiplier based on distance;
3. value based on stat difference;
4. modifier id choice based on unit type.

### Post effects

Реагируют на результат после применения эффекта.

Примеры:

1. lifesteal;
2. add damage on kill;
3. heal from poison stacks.

Для нового движка это очень полезная идея, потому что позволяет не раздувать число concrete payload actions.

В терминах skill engine это можно переименовать в:

1. `pre-processors`;
2. `post-processors`.

## 20. Data injection в runtime

`Modifier.SetData(IList<IData>)` позволяет runtime-параметризовать instance после создания.

Это используется для:

1. изменения interval/duration;
2. стартовых stack-ов;
3. передачи effect-specific data;
4. таргетированного обновления нужного эффекта по типу и номеру.

Это сильный паттерн для skill engine:

authoring definition остается общей,
а runtime instance может получать контекстно-зависимые параметры от каста, предмета, таланта, proc-а или LLM-компилятора.

## 21. Tag model

`TagType` в `ModiBuff` выполняет две роли сразу:

1. gameplay tagging;
2. internal runtime flags.

Внутренние флаги:

1. `IsInit`;
2. `IsRefresh`;
3. `IsStack`;
4. `IsInstanceStackable`;
5. `CustomStack`;
6. `ZeroDefaultStacks`;
7. `IsAura`;
8. `CustomRefresh`.

Это удобно, но для нового движка лучше развести:

1. user/gameplay tags;
2. derived runtime behavior flags.

Иначе authoring contract начинает протекать внутренними деталями исполнения.

## 22. Targeting model

`Targeting` в `ModiBuff` это небольшой, но важный слой.

Он позволяет effect-у быть переиспользуемым для схем:

1. `target <- source`
2. `source <- target`
3. `target <- target`
4. `source <- source`

Это хороший пример того, как expressive behavior можно получить без explosion числа effect-классов.

Для нового движка это стоит сохранить, возможно в более декларативной форме:

1. `target = target|source`
2. `source = target|source`

## 23. Auras

Ауры в `ModiBuff` не являются отдельной глобальной подсистемой.

Архитектурно это modifier с `TagType.IsAura`, который:

1. использует `MultiTargetComponent`;
2. получает target set через `IAuraOwner.GetAuraTargets(auraId)`;
3. может иметь собственный interval и aura-effect recipes.

Это очень хорошее решение.

Аура здесь это не отдельный monster-type system, а частный случай общей modifier модели с multi-target execution.

Для skill engine это сильный ориентир.

## 24. Modifierless effects

`ModifierLessEffects` это отдельный слой для мгновенных эффектов без полного lifecycle modifier instance.

Ограничения там осознанные:

1. нельзя использовать revertible effects;
2. нельзя использовать mutable state effects.

Это уже зачаток разделения между:

1. `instant actions`;
2. `stateful persistent instances`.

Для нового движка это почти напрямую маппится на разделение:

1. `cast actions`;
2. `buff/debuff/status/runtime modifiers`.

## 25. Сериализация

Сериализация в `ModiBuff` устроена серьезнее, чем просто `save active modifiers`.

### 25.1 Что сериализуется

1. id maps (`ModifierIdManager`, `EffectIdManager`);
2. unit state;
3. `ModifierController` state;
4. `ModifierApplierController` state;
5. status effect controllers;
6. timers;
7. stacks;
8. mutable effect state.

### 25.2 Почему нужен id remap

При загрузке `ModifierIdManager.LoadState(...)` и `EffectIdManager.LoadState(...)` строят old-to-new id mapping по именам.

Это важная идея:

save file не обязан полагаться на жестко стабильные numeric ids между сессиями.

Для authoring pipeline это очень полезно, особенно если рецепты могут пересобираться.

### 25.3 Отдельный `InitLoad()`

При `ModifierController.LoadState(...)` после восстановления runtime instance вызывается `InitLoad()`, а не обычный `Init()`.

То есть система разделяет:

1. восстановление подписок и служебного состояния;
2. повторное проигрывание gameplay effect-а.

Это критично для корректного save/load skill runtime.

## 26. Godot resource authoring

`ModiBuff.Extensions.Godot.ModifierRecipesGodot` показывает, как recipe DSL можно обернуть в data authoring слой.

Схема там такая:

1. загрузить `.tres` resources;
2. провалидировать их;
3. конвертировать resource fields в вызовы `ModifierRecipe` fluent API;
4. присвоить/сохранить ids;
5. затем использовать обычный `CreateGenerators()` runtime pipeline.

Это очень важное доказательство архитектурной правильности `ModiBuff`:

его runtime уже умеет жить отдельно от конкретного authoring front-end.

То есть новый `LLM -> JSON -> compiler` слой может работать по той же схеме.

## 27. Как это маппится на будущий skill engine

Если смотреть на `ModiBuff` как на базу для нового движка навыков, то наиболее ценны следующие идеи.

### 27.1 Что точно стоит перенести

1. разделение `authoring definition` и `runtime instance`;
2. compile phase между ними;
3. `ApplyCheck` vs `EffectCheck`;
4. lifecycle triggers: `Init`, `Interval`, `Duration`, `Stack`, unit callbacks;
5. callback registration как часть modifier lifecycle;
6. mutable effect state внутри runtime instance;
7. pooled runtime;
8. `InitLoad()` как отдельную стадию восстановления;
9. typed events из методов `Unit`, а не глобальный bus;
10. orthogonal model для time/stack/trigger/payload.

### 27.2 Что лучше переработать

1. уменьшить сложность callback API;
2. убрать протекание внутренних runtime flags в public authoring модель;
3. сильнее развести payload, conditions, meta, persistence roles;
4. сделать DTO-first contract вместо delegate-heavy public API;
5. вынести `Unit` в reference package, а core оставить еще более минимальным;
6. сделать формальную intermediate representation для skill compiler-а.

### 27.3 Что в `ModiBuff` уже является прототипом skill engine

По сути уже сейчас `ModiBuff` умеет:

1. описывать passive и active behavior;
2. навешивать runtime instances на entities;
3. обрабатывать periodic, duration, stack, callback logic;
4. собирать сложные composite effects;
5. восстанавливать runtime state после save/load.

Не хватает в основном:

1. удобного DTO/JSON authoring contract;
2. formal compiler из текстового/JSON skill description;
3. более строгой публичной модели trigger/action/condition;
4. более явной skill-level abstraction поверх `modifier`.

## 28. Основные архитектурные сильные стороны

`ModiBuff` особенно силен в следующем:

1. очень хороший runtime lifecycle design;
2. грамотное разделение apply vs execution gating;
3. высокая composability через capability-интерфейсы;
4. продуманная pooling-модель;
5. event subscriptions как часть жизненного цикла;
6. поддержка save/load mutable runtime state;
7. ауры и стаки реализованы как частные случаи общей модели, а не отдельные системы.

## 29. Основные архитектурные ограничения

Есть и ограничения, важные для проектирования следующей версии.

### 29.1 API complexity

Часть expressive power достигнута за счет перегруженности API.

Особенно это касается:

1. callback system;
2. effect lane variants;
3. некоторых advanced effect contracts.

### 29.2 Modifier-centric terminology

Система концептуально уже переросла термин только `modifier`.

Она обслуживает:

1. skills;
2. casts;
3. passive triggers;
4. aura zones;
5. status lifecycles;
6. instant actions.

Для нового движка стоит перейти к более общей терминологии:

1. `SkillDefinition`;
2. `SkillInstance`;
3. `EffectEntry`;
4. `Trigger`;
5. `Action`;
6. `Condition`.

### 29.3 DTO-unfriendly public surface

Многие вещи сейчас удобно писать в C#, но не так удобно транслировать из LLM/JSON напрямую.

Значит новый слой должен быть не поверх уже готовых delegate-API, а над более строгой декларативной схемой.

## 30. Практический вывод для проекта AI Skill Crafting

Если цель проекта:

1. принимать фантазию игрока текстом;
2. переводить ее в структурированный skill contract;
3. исполнять это в объяснимом и балансируемом runtime;

то `ModiBuff` дает очень сильную основу именно на runtime-уровне.

Лучше всего воспринимать его так:

1. не как готовый финальный authoring engine для LLM;
2. а как качественное ядро runtime-примитивов и lifecycle-модели.

Хорошая стратегия для нового движка на его основе:

1. сохранить core execution model;
2. поверх него ввести свой `Skill DTO / JSON IR`;
3. сделать compiler `DTO -> compiled skill graph -> runtime primitives`;
4. ограничить публичный authoring contract более простыми типизированными trigger/action/condition блоками;
5. advanced/custom callback behavior оставить как internal escape hatch.

## 31. Короткая итоговая формула

Если свести архитектуру `ModiBuff` к одной формуле, то она такая:

`ModiBuff = compiled modifier runtime with pooled instances, typed unit-trigger integration, orthogonal time/stack/effect model, and reversible lifecycle-driven behavior`

Для нового движка навыков это особенно ценно потому, что:

1. свобода описания может жить во внешнем AI/DTO-слое;
2. а строгость и исполнимость уже поддерживаются очень хорошей внутренней моделью runtime.

## 32. Конкретные файлы, на которые стоит опираться дальше

Если потом использовать этот анализ как базу для проектирования нового движка, в первую очередь нужно смотреть в эти файлы:

1. `ModiBuff/Core/Modifier/Creation/Recipe/ModifierRecipe.cs`
2. `ModiBuff/Core/Modifier/Creation/Recipe/ModifierRecipes.cs`
3. `ModiBuff/Core/Modifier/Creation/Recipe/ModifierRecipeData.cs`
4. `ModiBuff/Core/Modifier/Creation/Generation/ModifierGenerator.cs`
5. `ModiBuff/Core/Modifier/Creation/Recipe/ModifierEffectsCreator.cs`
6. `ModiBuff/Core/Modifier/Modifier.cs`
7. `ModiBuff/Core/Modifier/ModifierController.cs`
8. `ModiBuff/Core/Pool/ModifierPool.cs`
9. `ModiBuff/Core/Modifier/Components/Main/InitComponent.cs`
10. `ModiBuff/Core/Modifier/Components/Main/IntervalComponent.cs`
11. `ModiBuff/Core/Modifier/Components/Main/DurationComponent.cs`
12. `ModiBuff/Core/Modifier/Components/Main/StackComponent.cs`
13. `ModiBuff/Core/Modifier/Components/Effect/CallbackUnitRegisterEffect.cs`
14. `ModiBuff/Core/Modifier/Components/Effect/RemoveEffect.cs`
15. `ModiBuff.Units/Unit/Unit.cs`
16. `ModiBuff.Units/Unit/UnitCallbacks.cs`
17. `ModiBuff.Units/Recipe/ModifierRecipeExtensions.cs`
18. `ModiBuff.Units/Effects/DamageEffect.cs`
19. `ModiBuff.Extensions.Godot/ModifierRecipesGodot.cs`
20. `ModiBuff.Extensions.Serialization.Json/SaveController.cs`

## 33. Что можно сделать следующим шагом

На основе этого документа логичный следующий шаг один из трех:

1. выделить минимальный `Skill DTO contract v1` на базе идей `ModiBuff`;
2. составить `mapping table` между фэнтези-описанием навыка и runtime-примитивами `ModiBuff`;
3. спроектировать новый `authoring/compiler layer`, который будет компилировать `LLM output -> skill IR -> runtime`.
