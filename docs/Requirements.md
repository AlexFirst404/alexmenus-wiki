# Requirements (условия)

Requirements-движок (`action/RequirementEvaluator.java`) — сердце AlexMenus. Условие — это YAML-карта с полем
`type` и параметрами. **Отсутствующее/пустое условие считается выполненным** (`true`). Неизвестный `type`
тоже даёт `true` (fail-open для незнакомых типов).

## Три места условий (scopes)

| Где | Ключ | Эффект |
|---|---|---|
| Предмет | `view-requirement` | Предмет **виден** только при выполнении условия. Иначе — пустая ячейка (клики не срабатывают). |
| Предмет | `click-requirement` | **Гейт клика**: не выполнено → `deny`, клик остановлен; выполнено → `success` + действия клика. |
| Меню (верхний уровень) | `open-requirement` | **Гейт открытия**: не выполнено → `deny`, меню не открывается. |

!!! info "`open-requirement` и `permission:` — разные гейты, и оба стоят на всех входах"
    `permission:` меню — статичное право без deny-действий; `open-requirement` — полноценное условие,
    которое умеет объяснить отказ. Порядок при входе в меню: сначала `permission:`, потом
    `open-requirement` (его `deny:` выполняется только если право уже прошло).

    С **1.10.0** оба гейта работают на **любом** пути открытия, включая действие [`open_menu`](Actions.md) и
    Java-API — раньше `permission:` держал только команду меню. Что могло сломаться — см.
    [Commands-and-Permissions](Commands-and-Permissions.md).

## Два формата записи

### Краткий (bare) — вся карта = условие

```yaml
view-requirement:
  type: permission
  permission: alexmenus.vip
```

### Полный (с обработчиками) — `require:` + `deny:`/`success:`

```yaml
click-requirement:
  require:                       # условие
    type: money
    amount: 100
  deny:                          # выполняется при провале (список действий)
    - type: message
      text: "<red>Недостаточно денег"
  success:                       # опц.: выполняется при успехе, ПЕРЕД действиями клика
    - type: sound
      sound: minecraft:entity.experience_orb.pickup
```

Замечания по форматам:

- **`view-requirement`** использует **только условие** (без `deny`/`success`). Но если написать его в форме
  с `require:`, обёртка корректно разворачивается — просто `deny`/`success` там не действуют.
- **`click-requirement`** и **`open-requirement`** понимают оба формата: краткий (вся карта — условие) и
  полный (`require:` — условие, соседние `deny:`/`success:` — списки действий).
- **Список вместо карты = неявный `all`** (каждый элемент должен пройти). Сделано так, чтобы ошибочно
  «списочное» условие **падало закрыто** (fail-closed), а не тихо исчезало.

```yaml
# список верхнего уровня = неявный all (нужно всё сразу)
open-requirement:
  - type: permission
    permission: alexmenus.beta
  - type: money
    amount: 500
```

## Типы условий

Любому условию можно добавить **`negate: true`** — инвертировать результат.

### `permission` (алиас `has_permission`)

Право у игрока.

```yaml
type: permission
permission: alexmenus.vip
```

Пустой `permission` → условие не проходит.

### `placeholder` (алиас `string`)

Разворачивает `placeholder` через PlaceholderAPI и сравнивает с `value` оператором `operator`.

```yaml
type: placeholder
placeholder: "%player_level%"
operator: ">="
value: "30"
```

**Операторы:**

| Оператор | Смысл |
|---|---|
| `==` (по умолчанию) | строго равно (без `operator` используется это) |
| `!=` | не равно |
| `equals_ignorecase` | равно без учёта регистра |
| `contains` | подстрока |
| `contains_ignorecase` | подстрока без учёта регистра |
| `regex` (алиас `matches`) | регулярка сопоставляется целиком (`matches()`); битая регулярка → `false` |
| `>` `<` `>=` `<=` | числовое сравнение (обе стороны парсятся в число; не-число → `false`) |

Без PlaceholderAPI плейсхолдер не разворачивается (строка остаётся как есть), поэтому такое условие обычно
не проходит — см. [Placeholders-and-Integrations](Placeholders-and-Integrations.md).

### `money` (алиас `has_money`) — нужен Vault

Хватает ли у игрока денег (`amount`).

```yaml
type: money
amount: 100
```

Без Vault всегда `false` (краша нет). Альтернатива без прямой Vault-проверки — `placeholder` по
`%vault_eco_balance%`.

### `has_item`

Есть ли у игрока `amount` подходящих предметов. Поле `amount` — сколько нужно (по умолчанию `1`), все
остальные поля — **фильтр предмета**: тот же словарь, что у действия `take_item` и у `input.accept`
(см. [Input-Slots](Input-Slots.md)).

```yaml
type: has_item
material: DIAMOND
amount: 3          # по умолчанию 1; минимум 1
```

| Поле фильтра | Что проверяет |
|---|---|
| `material` | Материал: строка или список материалов. |
| `name-equals` | Точное имя (регистр важен). |
| `name-contains` | Подстрока имени (регистр не важен). |
| `lore-contains` | Строка или список подстрок лора. |
| `cmd` | custom-model-data. |
| `tag` | PDC-ключ `namespace:key` — проверяется наличие. |
| `min-amount` | Минимальный размер стека. |
| `enchants` | Минимальные уровни зачарований. |

```yaml
# «есть именной кузнечный слиток», обычным не подойдёт
type: has_item
material: GOLD_INGOT
name-contains: "Кузнечный"
amount: 1
```

!!! warning "Что изменилось в 1.7.0"
    - `has_item` смотрит **только 36 основных слотов** (хотбар + основной инвентарь) — **броня и офф-хенд
      больше не считаются**. Гейт вида «надет `DIAMOND_HELMET`» через `has_item` **перестанет
      срабатывать**: проверяй экипировку плейсхолдером. Причина — списание (`take_item`) никогда не
      трогало броню и офф-хенд, а гейт их видел: плату можно было держать в левой руке, гейт пропускал,
      списание промахивалось и награда выдавалась даром.
    - Сравнение идёт **полным фильтром**, а не только по материалу — как и у `take_item`.
    - Требование **без единого поля фильтра** считается **невыполненным** (пустой фильтр совпал бы с любым
      предметом). В лог пишется предупреждение.

Считает суммарное количество по всем подходящим стакам. Неизвестный материал пропускается с предупреждением
в лог.

### `exp` (алиас `has_exp`)

Хватает ли опыта.

```yaml
type: exp
amount: 30
level: true        # true — сравнивать по УРОВНЯМ; false/нет — по очкам опыта
```

`level: true` → сравнивает `player.getLevel()`. `level: false`/нет → сравнивает истинный суммарный опыт
(`calculateTotalExperiencePoints()` — надёжнее ванильного счётчика).

### `chance` — случайное условие

Проходит с вероятностью `percent` процентов. Дробные значения работают.

```yaml
type: chance
percent: 12.5      # выполняется в 12.5% случаев
```

Работает и внутри композитов — например «право **и** повезло»:

```yaml
click-requirement:
  require:
    type: all
    of:
      - type: permission
        permission: alexmenus.lucky
      - type: chance
        percent: 25
  deny:
    - type: message
      text: "&7Не в этот раз."
```

Отсутствующий/нечисловой `percent` — **fail-open** (условие считается выполненным) плюс предупреждение в
лог: опечатка не должна намертво запирать игроков в меню.

> Для «одна ветка из нескольких» лучше подходит действие [`outcome`](Actions.md): оно выбирает ровно один
> исход по весам, а не бросает независимые монетки.

### `slot_empty` / `slot_filled` / `slot_matches` — содержимое input-слота

Читают ячейку, куда игрок кладёт свой предмет (`input:`, см. [Input-Slots](Input-Slots.md)). Поле `slot` —
номер input-слота **открытого сейчас** меню.

```yaml
# в слоте что-то лежит
type: slot_filled
slot: 13
```

```yaml
# в слоте лежит именно алмазный меч с остротой 4+
type: slot_matches
slot: 13
material: DIAMOND_SWORD
enchants:
  sharpness: 4
```

| Тип | Истинно, когда |
|---|---|
| `slot_empty` | В слоте ничего нет. |
| `slot_filled` | В слоте что-то лежит. |
| `slot_matches` | В слоте лежит предмет, проходящий фильтр. |

`slot_matches` принимает **те же поля фильтра**, что `input.accept`: `material` (строка/список),
`name-equals`, `name-contains`, `lore-contains`, `cmd`, `tag`, `min-amount`, `enchants`.

Несуществующий (или не-input) слот ведёт себя как **пустой**: `slot_empty` проходит,
`slot_filled`/`slot_matches` — нет. Такой случай один раз пишется в лог с указанием меню и номера слота —
если гейт «всегда открыт», ищи это предупреждение.

### `all` / `any` — композиты

`all` — все вложенные условия из `of:` должны пройти. `any` — хотя бы одно (пустой `of:` у `any` → `true`).

```yaml
type: any
of:
  - type: permission
    permission: alexmenus.vip
  - type: money
    amount: 1000
```

### `not` — отрицание

`of:` — одно условие или список. Для **списка** `not` истинно, когда **хотя бы одно** вложенное условие
проваливается (отрицание неявного `all`).

```yaml
type: not
of:
  type: permission
  permission: alexmenus.banned
```

### `negate: true` — инверсия на любом условии

```yaml
type: has_item
material: DIAMOND
amount: 64
negate: true       # истинно, когда алмазов НЕ 64+
```

`negate` и `not` можно комбинировать с композитами для сложной логики.

---

## Рецепты

### VIP-предмет (виден только VIP)

```yaml
items:
  '13':
    material: NETHER_STAR
    name: "<aqua>VIP-бонус"
    view-requirement:
      type: permission
      permission: alexmenus.vip
    clicks:
      any:
        - type: message
          text: "<aqua>Спасибо за VIP!"
```

### Покупка за деньги (Vault)

```yaml
items:
  '11':
    material: GOLDEN_APPLE
    name: "<gold>Золотое яблоко — 100$"
    click-requirement:
      require:
        type: money
        amount: 100
      deny:
        - type: message
          text: "<red>Недостаточно денег (нужно 100$)."
    clicks:
      any:
        - type: take_money
          amount: 100
        - type: give_item
          material: GOLDEN_APPLE
          amount: 1
        - type: message
          text: "<green>Куплено!"
```

> С 1.7.0 порядок в этом списке важен: если `take_money` не смог списать (не хватило денег, нет Vault), то
> **ни `give_item`, ни «Куплено!» не выполнятся**. Ставь списание первым — тогда гейт и оплата не смогут
> разойтись. См. [Actions](Actions.md) → «Оплата обрывает цепочку».

### Меню, доступное только с 30 уровня

```yaml
type: chest
title: "<gold>Элитный зал"
rows: 3
open-requirement:
  require:
    type: exp
    amount: 30
    level: true
  deny:
    - type: message
      text: "<red>Нужен 30-й уровень."
items:
  '13': { material: DIAMOND, name: "<aqua>Награда" }
```

### Комбинированное условие (право ИЛИ богатство)

```yaml
open-requirement:
  require:
    type: any
    of:
      - type: permission
        permission: alexmenus.elite
      - type: placeholder
        placeholder: "%vault_eco_balance%"
        operator: ">="
        value: "100000"
  deny:
    - type: message
      text: "<red>Нужен ранг Elite или 100k на счету."
```

Действия в `deny`/`success` — любые из [Actions](Actions.md).
