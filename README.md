# Avalora Lua API — документация

Система пользовательских HUD-скриптов на Lua. Пишете обычный `.lua` файл — он рисуется поверх игрового интерфейса. Никакой перекомпиляции клиента не нужно: сохранили файл — правки подхватились.

Движок — **LuaJ 3.0.1** (Lua 5.2), уже вшит в jar клиента.

---

## Содержание

- [Игровой API — Camera, Movement, Hotbar, World](#игровой-api--camera-movement-hotbar-world)

1. [Быстрый старт](#быстрый-старт)
2. [Где лежат скрипты](#где-лежат-скрипты)
3. [Категория Lua в ClickGui](#категория-lua-в-clickgui)
4. [Система координат](#система-координат)
5. [Точки входа](#точки-входа)
6. [API](#api)
   - [Rect — фигуры](#rect--фигуры)
   - [Text — текст](#text--текст)
   - [Color — цвета](#color--цвета)
   - [Image — картинки, головы, предметы](#image--картинки-головы-предметы)
   - [Client — клиент и экран](#client--клиент-и-экран)
   - [Player — игрок](#player--игрок)
   - [Anim — анимации](#anim--анимации)
   - [Scissor — обрезка](#scissor--обрезка)
   - [Key — клавиши](#key--клавиши)
   - [Rotation — ротации KillAura](#rotation--ротации-killaura)
7. [Глобальные константы](#глобальные-константы)
8. [Шрифты](#шрифты)
9. [Easing (кривые анимации)](#easing-кривые-анимации)
10. [Песочница и ограничения](#песочница-и-ограничения)
11. [Обработка ошибок](#обработка-ошибок)
12. [Полный пример](#полный-пример)
13. [Частые вопросы](#частые-вопросы)

---

## Быстрый старт

Создайте файл `avalora/scripts/hello.lua`:

```lua
function on_render()
    Rect.draw(100, 100, 160, 40, 8, Color.rgba(10, 10, 14, 180))
    Text.draw("Привет, Avalora", 114, 114, 12, Color.rgb(255, 255, 255), "semibold")
end
```

Откройте ClickGui → категория **Lua** → включите тумблер `Отрисовка HUD` и тумблер на карточке `hello.lua`. Панель появится на экране.

---

## Где лежат скрипты

| Путь | Что это |
|---|---|
| `avalora/scripts/*.lua` | сами скрипты (папка рядом с `avalora/configs`) |
| `avalora/scripts/state.json` | какие скрипты включены (пишется автоматически) |

Папка создаётся при первом запуске. Вместе с ней выкладываются три примера:

- `watermark.lua` — ватермарка с fps/ping/часами;
- `info_panel.lua` — панель с головой игрока, HP, координатами, TPS;
- `keystrokes.lua` — индикатор нажатых клавиш WASD.

Примеры выкладываются **только если файла с таким именем ещё нет** — ваши правки не перезатрутся. Хотите вернуть оригинал — удалите файл и нажмите `Обновить`.

Кодировка файлов — **UTF-8**. Русский текст внутри строк работает.

---

## Категория Lua в ClickGui

В сайдбаре появилась категория **Lua**. Она открывает не обычную сетку модулей, а страницу управления скриптами.

**Верхняя панель:**

| Кнопка | Действие |
|---|---|
| `Обновить` | пересканировать папку и перекомпилировать всё |
| `Папка` | открыть `avalora/scripts` в проводнике |

Рядом с заголовком — счётчик `N файлов • M активно`.

**Строка пилюль:**

| Пилюля | Что делает |
|---|---|
| `Отрисовка HUD` | главный выключатель — включает/выключает модуль `Lua HUD`. Если он выключен, ни один скрипт не рисуется |
| `Авто-перезагрузка` | раз в секунду проверять время изменения файлов и перекомпилировать изменённые |
| `Рисовать в меню` | рисовать скрипты, когда открыт ClickGui или другой экран. По умолчанию выключено (чат — исключение, в нём рисует всегда) |

**Карточка скрипта:**

- имя файла и статус: `работает` / `выключен` / `ошибка компиляции`;
- тумблер справа — включить/выключить (сразу пишется в `state.json`);
- `Перезагрузить` — перекомпилировать только этот файл;
- `Открыть` — открыть `.lua` в системном редакторе.

Сломанный скрипт подсвечивается красным, а текст ошибки печатается прямо на карточке.

Поиск в шапке ClickGui фильтрует список по имени файла.

---

## Система координат

Главная фишка: **координаты не зависят от монитора и от GUI-scale**.

Холст всегда имеет **высоту 1080 виртуальных единиц**. Масштаб считается как `высота_экрана / 1080` и применяется одинаково к X и к Y — поэтому круги остаются кругами, а текст не растягивается.

Следствия:

- `y = 100` — это всегда одна и та же доля высоты экрана. На 1080p, на 1440p, на 4K, при любом GUI-scale — визуально одно и то же место.
- `Client.height()` **всегда возвращает 1080**.
- `Client.width()` **зависит от соотношения сторон**: 1920 на 16:9, ~2560 на 21:9, 1440 на 4:3.

Поэтому:

```lua
-- ПРАВИЛЬНО: привязка к правому краю
local x = Client.width() - PANEL_WIDTH - 24

-- НЕПРАВИЛЬНО: на ультравайде панель окажется в середине
local x = 1920 - PANEL_WIDTH - 24
```

`Client.scale()` отдаёт сам коэффициент — обычно он не нужен, но может пригодиться для отладки.

---

## Точки входа

Объявляете глобальные функции — клиент их вызывает. Нужна хотя бы одна рабочая точка входа из `on_tick`, `on_render`, `on_rotation` или `on_gui_render`.

```lua
function on_load()    end  -- один раз при компиляции/включении скрипта
function on_render()  end  -- каждый кадр; только тут можно рисовать
function on_tick()    end  -- 20 раз в секунду, игровой тик; рисовать нельзя
function on_unload()  end  -- при выключении или перед перезагрузкой
```

Для игрового скрипта достаточно одной функции `on_tick()`.

Храните состояние между вызовами `on_tick()`. Длительные действия разбивайте на шаги, чтобы не задерживать игровой поток.

Глобальные переменные скрипта живут между кадрами и **не видны другим скриптам** — у каждого файла своя песочница.

---

## API

Соглашения:
- цвет — целое число ARGB, получается через таблицу `Color`;
- `radius` — радиус закругления углов;
- параметры в квадратных скобках необязательны;
- шрифт по умолчанию — `"medium"`, размер текста по умолчанию — `8`.

### Игровой API — Camera, Movement, Hotbar, World

Имена чувствительны к регистру. Вызывайте функции через точку. Координаты мировые, в блоках. Векторы возвращаются как Lua-таблицы `{x, y, z}`. Методы, меняющие игру, возвращают `false` вне собственного `on_tick()` или пока `Movement.ready()` возвращает `false`.

**Camera**

- `Camera.rotation() → yaw, pitch | nil` — углы взгляда игрока в градусах.
- `Camera.eye() → vec | nil` — точка глаз игрока, начало луча взаимодействия.
- `Camera.position() → vec | nil` — последняя позиция камеры рендера, включая вид от третьего лица.
- `Camera.setRotation(yaw, pitch) → bool` — поворачивает взгляд игрока. Положительный pitch смотрит вниз, предел ±90°.
- `Camera.lookAt(x, y, z) → bool` — поворачивает взгляд к точке. Для плавного поворота меняйте углы небольшими шагами из `on_tick()`.

**Movement**

- `Movement.ready() → bool` — есть живой локальный игрок в мире, экран закрыт, игра без паузы, игрок может действовать.
- `Movement.position() → vec | nil` — позиция ног.
- `Movement.onGround() → bool` — игрок стоит на поверхности.
- `Movement.walk(forward, strafe[, jump[, sneak[, sprint]]]) → bool` — запрашивает ввод на один игровой тик. Оси принимают только −1, 0 или 1. Положительные значения идут вперёд и влево. Флаги по умолчанию `false`. Удержание требует повторного вызова каждый тик. Ручной ввод имеет приоритет. Между скриптами приоритет у первого запроса за тик.
- `Movement.stop()` — отзывает ещё не применённый ввод этого скрипта. При выключении или ошибке ввод сбрасывается автоматически.

**Hotbar**

- `Hotbar.selected() → slot | nil` — выбранный слот от 1 до 9.
- `Hotbar.get(slot) → {id, count, block} | nil` — снимок стека или `nil` для пустого слота. `block` означает, что предмет относится к `BlockItem`.
- `Hotbar.select(slot) → bool` — выбирает слот основной руки.

**World**

- `World.dimension() → string | nil` — ID измерения, например `minecraft:overworld`.
- `World.block(x, y, z) → {id, air, replaceable, solid} | nil` — снимок блока. Координаты округляются вниз. Вне высоты строительства и в незагруженных чанках возвращает `nil`.
- `World.raycast([x, y, z]) → {x, y, z, blockX, blockY, blockZ, side} | nil` — первое пересечение луча из глаз с контуром блока. Без аргументов луч идёт по взгляду. С координатами луч заканчивается в указанной точке, которая должна быть в пределах досягаемости. `side` принимает `up`, `down`, `north`, `south`, `east`, `west`.
- `World.useBlock([x, y, z]) → bool` — обычное взаимодействие основной рукой с блоком, найденным тем же лучом. Для установки выберите слот с блоком и цельтесь в нужную грань опоры. На весь Lua API доступна одна попытка взаимодействия за тик. `true` означает, что клиент принял действие. Проверьте изменение через `World.block` после вызова, поскольку сервер может его отклонить.

**TShipBuilder.** Готовый пример лежит в `assets/avalora/scripts/TShipBuilder.lua` внутри JAR. Клиент извлекает его в `avalora/scripts` при инициализации. Включите скрипт в категории Lua и закройте меню. На ровной площадке он ищет место для вертикальной перевёрнутой T из пяти блоков, наводится на опору, при необходимости прыгает и между постройками делает короткие случайные проходы. После исчерпания блоков останавливается. Для нового запуска выключите и включите скрипт. В творческом режиме расход зависит от правил Minecraft, поэтому запасы могут не убывать.

**Шаблон запроса для ИИ**

> Напиши один файл .lua для Avalora по приложенному LUA_API.md. Используй только перечисленные в нём методы. Игровые действия выполняй в глобальном on_tick() после проверки Movement.ready(). Храни состояние между тиками. Для установки блока выбери слот 1–9, проверь опору и грань через World.raycast, после World.useBlock проверь блок на следующих тиках. Ограничь ожидание и число попыток. Верни готовый Lua-скрипт для avalora/scripts.

### Rect — фигуры

```lua
Rect.draw(x, y, w, h, radius, color)
Rect.draw(x, y, w, h, radius, color1, color2)                 -- вертикальный градиент
Rect.draw(x, y, w, h, radius, c1, c2, c3, c4)                 -- градиент по углам
Rect.gradient(x, y, w, h, radius, c1, c2, c3, c4)             -- явный градиент по углам
Rect.outline(x, y, w, h, radius, thickness, color)            -- рамка
Rect.blur(x, y, w, h, radius [, intensity=12 [, alpha=1]])    -- размытие фона без тонировки
Rect.glass(x, y, w, h, radius, color [, intensity=12])        -- размытие + тонировка (матовое стекло)
Rect.glow(x, y, w, h, radius [, glow=8 [, color]])            -- внешний ореол
Rect.shadow(x, y, w, h, radius [, glow=10 [, color]])         -- то же, но тёмный по умолчанию
Rect.line(x1, y1, x2, y2 [, thickness=1 [, color]])           -- линия
Rect.dashed(x, y, w, h, radius, thickness, color, dash, gap, phase)  -- пунктирная рамка
Rect.hovered(x, y, w, h) -> boolean                           -- курсор внутри области
```

`Rect.hovered` возвращает `true` только когда открыт какой-нибудь экран (иначе курсор захвачен игрой). Полезно вместе с `Рисовать в меню`.

Анимированный пунктир: увеличивайте `phase` со временем, например `Client.time() / 40 % 1000`.

### Text — текст

```lua
Text.draw(text, x, y, size, color [, font])          -- x = левый край
Text.center(text, centerX, y, size, color [, font])  -- centerX = центр строки
Text.right(text, rightX, y, size, color [, font])    -- rightX = правый край
Text.width(text, size [, font]) -> number            -- ширина строки
Text.height(size) -> number                          -- высота строки (≈ size)
Text.fade(text, x, y, size, color, maxWidth [, font])-- обрезка с растворением по краю
```

`Text.width` — основной инструмент вёрстки: считаете ширину заранее и подгоняете под неё фон.

```lua
local label = "Avalora"
local w = Text.width(label, 14, "bold")
Rect.draw(20, 20, w + 28, 34, 8, Color.rgba(10, 10, 14, 180))
Text.draw(label, 34, 30, 14, Color.rgb(255, 255, 255), "bold")
```

### Color — цвета

Создание:

```lua
Color.rgb(r, g, b)            -- 0..255, альфа = 255
Color.rgba(r, g, b, a)        -- альфа 0..255
Color.hex("#RRGGBB")          -- также "#AARRGGBB", "RRGGBB", "0xRRGGBB"
```

Модификация:

```lua
Color.alpha(color, a)         -- задать альфу 0..255
Color.opacity(color, k)       -- задать альфу долей 0..1
Color.lerp(from, to, t)       -- смешать, t в 0..1
Color.mix(c1, c2, t)          -- то же, другая формула
Color.darker(color, amount)   -- затемнить, amount 0..1
Color.lighter(color, amount)  -- осветлить, amount 0..1
```

Динамика:

```lua
Color.gradient(speed, index, c1, c2, ...)  -- «бегущий» градиент клиента; index — номер элемента
Color.rainbow([speed=4000 [, offset=0]])   -- радуга по времени, speed в мс на круг
Color.pulse(c1, c2 [, speed=2000])         -- плавное «дыхание» между двумя цветами
```

Цвета активной темы клиента (меняются вместе с темой в ClickGui):

```lua
Color.client()      -- основной акцент
Color.theme()       -- цвет темы
Color.theme2()      -- второй цвет темы
Color.main()        -- основной цвет интерфейса
Color.text()        -- цвет текста
Color.background()  -- цвет фона окна
Color.icons()       -- цвет иконок
```

Разбор:

```lua
Color.red(color)        -- 0..255
Color.green(color)
Color.blue(color)
Color.get_alpha(color)
```

### Image — картинки, головы, предметы

```lua
Image.draw(path, x, y, w, h [, radius=0 [, tint]])
Image.head(x, y, size [, radius=0 [, tint]])                 -- голова локального игрока
Image.head_of(nickname, x, y, size [, radius=0 [, tint]])     -- голова игрока рядом
Image.item(slot, x, y, size [, alpha=255])                    -- предмет со счётчиком и прочностью
Image.item_sprite(slot, x, y, size [, alpha=255])             -- только иконка
```

`path` — идентификатор ресурса, например `"minecraft:textures/gui/title/mojangstudios.png"` или ваш `"avalora:textures/логотип.png"`.

`slot` для `Image.item` / `Image.item_sprite` / `Player.item`:

| Значение | Слот |
|---|---|
| `"hand"` | основная рука (по умолчанию) |
| `"offhand"` | вторая рука |
| `"helmet"` / `"head"` | шлем |
| `"chestplate"` / `"chest"` | нагрудник |
| `"leggings"` / `"legs"` | штаны |
| `"boots"` / `"feet"` | ботинки |
| число `0..40` | слот инвентаря напрямую |

`head_of` ищет игрока по нику среди загруженных сущностей — если его нет в радиусе прогрузки, ничего не нарисуется.

### Client — клиент и экран

```lua
Client.fps()        -- кадров в секунду
Client.tps()        -- TPS сервера (мгновенный)
Client.avg_tps()    -- средний TPS
Client.ping()       -- пинг локального игрока, мс

Client.width()      -- ширина холста в виртуальных единицах (зависит от соотношения сторон)
Client.height()     -- высота холста, всегда 1080
Client.scale()      -- коэффициент виртуальные → экранные единицы

Client.time()       -- System.currentTimeMillis(), для собственных анимаций
Client.clock()      -- локальное время строкой "HH:mm:ss"

Client.username()   -- ник из сессии
Client.server()     -- адрес сервера или "singleplayer"
Client.dimension()  -- "overworld" / "the_nether" / "the_end" / имя измерения

Client.in_game()    -- игрок существует
Client.chat_open()  -- открыт чат
```

### Player — игрок

```lua
Player.name()
Player.health([useServerHealth=true])  -- HP с поправкой на серверное значение
Player.max_health()
Player.absorption()                    -- золотые сердца
Player.armor()                         -- броня, 0..20
Player.hunger()                        -- сытость, 0..20
Player.air()                           -- воздух под водой

Player.x()   Player.y()   Player.z()
Player.yaw() Player.pitch()
Player.bps()                           -- скорость в блоках в секунду

Player.sneaking()   Player.sprinting()
Player.on_ground()  Player.in_water()
```

`Player.item([slot])` — таблица про предмет:

```lua
local it = Player.item("hand")
-- it.name        строка, название предмета
-- it.count       количество
-- it.damage      текущий износ
-- it.max_damage  максимальная прочность (0 если предмет не ломается)
-- it.durability  остаток прочности долей 0..1 (1 если предмет не ломается)
```

`Player.effects()` — массив активных эффектов:

```lua
for i, e in ipairs(Player.effects()) do
    -- e.name       название эффекта (переведённое)
    -- e.amplifier  уровень, начиная с 1
    -- e.duration   осталось тиков
    -- e.seconds    осталось секунд
end
```

### Anim — анимации

Анимации хранятся по строковому ключу внутри скрипта и живут между кадрами.

```lua
Anim.run(key, target [, duration=300 [, easing="cubic_out"]]) -> number
Anim.value(key) -> number     -- текущее значение без пересчёта
Anim.reset(key [, value=0])   -- мгновенно сбросить
Anim.lerp(from, to, t)        -- простая линейная интерполяция чисел
```

`Anim.run` нужно вызывать **каждый кадр** — она и считает шаг, и возвращает текущее значение.

```lua
function on_render()
    local visible = Player.health() < 10
    local a = Anim.run("alert", visible and 1 or 0, 200, "cubic_out")
    if a > 0.01 then
        Rect.draw(100, 100, 200 * a, 30, 6, Color.opacity(Color.rgb(235, 70, 70), a))
    end
end
```

### Scissor — обрезка

Ограничивает область рисования. Обязательно закрывать `pop()`.

```lua
Scissor.push(x, y, w, h)
-- всё, что рисуется здесь, обрезается по прямоугольнику
Scissor.pop()
```

### Key — клавиши

Движение читается через игровые биндинги, поэтому переназначенные клавиши учитываются.

```lua
Key.forward()  Key.back()   Key.left()   Key.right()
Key.jump()     Key.sneak()  Key.sprint()
Key.attack()   Key.use()

Key.pressed(code) -> boolean   -- произвольная клавиша по GLFW-коду
Key.code("f") -> number        -- GLFW-код одного символа: A-Z, 0-9; иначе -1
Key.mouse(button) -> boolean   -- 0 ЛКМ, 1 ПКМ, 2 СКМ
```

```lua
if Key.pressed(Key.code("g")) then
    Text.draw("G зажата", 100, 100, 12, Color.rgb(255, 255, 255))
end
```

### Rotation — ротации KillAura

С помощью Lua вы можете писать свои алгоритмы ротаций, которые **автоматически появляются** в выпадающем списке выбора ротаций в модуле **KillAura** (настройка `Ротация`).

#### Как объявить ротацию:

**Способ 1 (рекомендуемый):** просто объявите глобальную функцию `on_rotation` в файле скрипта (например, `lol.avalora.lua`):

```lua
function on_rotation(cur_yaw, cur_pitch, target_yaw, target_pitch, target)
    -- cur_yaw, cur_pitch       - текущие углы игрока
    -- target_yaw, target_pitch - идеальные углы наведения в хитбокс цели
    -- target                   - таблица с данными цели
    
    local dy = Rotation.delta(cur_yaw, target_yaw)
    local dp = Rotation.delta(cur_pitch, target_pitch)
    
    local new_yaw = cur_yaw + dy * 0.4
    local new_pitch = Rotation.clamp_pitch(cur_pitch + dp * 0.4)
    
    -- Применяем чувствительность мыши (GCD) для обхода античитов:
    new_yaw, new_pitch = Rotation.gcd(new_yaw, new_pitch)
    
    return new_yaw, new_pitch
end

-- Опционально: хук при ударе
function on_attack(target)
    -- вызывается при совершении атаки по цели
end
```
Имя ротации совпадает с именем файла скрипта (например, `lol.avalora.lua` -> ротация `lol.avalora`).

**Способ 2:** явная регистрация через `Rotation.register`:

```lua
Rotation.register("my_rotation", function(cur_yaw, cur_pitch, target_yaw, target_pitch, target)
    return new_yaw, new_pitch
end)
```

#### Методы `Rotation`:

```lua
Rotation.delta(from, to) -> number            -- кратчайшая разница углов (-180..180)
Rotation.wrap(angle) -> number                -- приведение угла к [-180, 180]
Rotation.clamp_pitch(pitch) -> number         -- ограничение pitch к [-90, 90]
Rotation.step(cur, target, max_step) -> number-- шаг к target с ограничением скорости
Rotation.lerp(cur, target, factor) -> number  -- интерполяция углов с учётом переворота
Rotation.gcd(yaw, pitch) -> yaw, pitch        -- применение Minecraft mouse sensitivity GCD
Rotation.angle_to(x, y, z) -> yaw, pitch      -- расчёт углов взгляда от глаз игрока в точку (x,y,z)
Rotation.predict(target, ticks) -> x, y, z    -- расчёт упреждения по скорости цели
Rotation.set_return_profile(yawSpeed, pitchSpeed, timeout) -- отводка камеры при потере цели
Rotation.state                                -- свободная таблица для хранения состояния между тиками
```

#### Поля таблицы `target`:

- `target.name` — ник или имя сущности;
- `target.health`, `target.max_health`, `target.absorption` — здоровье цели;
- `target.x`, `target.y`, `target.z`, `target.eye_y` — координаты цели;
- `target.yaw`, `target.pitch` — углы взгляда цели;
- `target.distance` — дистанция от глаз локального игрока до цели;
- `target.motion_x`, `target.motion_y`, `target.motion_z` — вектор скорости;
- `target.on_ground`, `target.is_gliding`, `target.is_sprinting`, `target.is_sneaking` — флаги состояния;
- `target.hurt_time` — тики после получения урона (для таймингов ударов);
- `target.bounding_box` — таблица габаритов: `min_x`, `min_y`, `min_z`, `max_x`, `max_y`, `max_z`, `width`, `height`.

---

## Глобальные константы

| Константа | Значение |
|---|---|
| `SCRIPT_NAME` | имя файла без `.lua` |
| `DESIGN_WIDTH` | `1920` — справочная ширина эталонного монитора |
| `DESIGN_HEIGHT` | `1080` — высота холста |

Для вёрстки используйте `Client.width()` / `Client.height()`, а не `DESIGN_*`.

---

## Шрифты

Последний аргумент в функциях `Text` — имя шрифта строкой. Регистр не важен. Неизвестное имя молча заменяется на `medium`.

**Текстовые:**
`regular`, `medium`, `semibold`, `bold`, `biko`, `rubik`, `divine`, `moonward`, `suisse`, `gui`, `hud`, `multiplayer`, `other`

**Иконочные:**
`icons`, `icons2`, `icons_nurik`, `icons_minced`, `hud_icons`, `avalora`, `lupa`

Иконочные шрифты рисуют глиф вместо буквы: `Text.draw("A", x, y, 12, color, "icons")`. Какая буква какой значок — смотрите перебором.

---

## Easing (кривые анимации)

Четвёртый аргумент `Anim.run`. Регистр не важен, по умолчанию `cubic_out`.

```
linear        smooth
quad_in       quad_out       quad_in_out
cubic_in      cubic_out      cubic_in_out
quartic_out   quintic_out
sine_in       sine_out       sine_in_out
expo_in       expo_out       expo_in_out
circ_out
back_in       back_out       back_in_out
elastic_out   bounce_out
```

Практика: `cubic_out` / `quintic_out` — для появления элементов, `expo_out` — для резкого влёта, `back_out` — с легким «перелётом», `elastic_out` / `bounce_out` — для акцентных эффектов.

---

## Песочница и ограничения

Каждый скрипт работает в своей изолированной среде. Доступны стандартные библиотеки `string`, `math`, `table`, часть `os`.

**Удалено полностью:**

`io`, `require`, `dofile`, `loadfile`, `load`, `loadstring`, `luajava`, `package`, `debug`

**Удалено из `os`:**

`os.execute`, `os.exit`, `os.remove`, `os.rename`, `os.tmpname`, `os.getenv`, `os.setlocale`

**Осталось из `os`:** `os.time`, `os.date`, `os.clock`, `os.difftime`.

Причина простая: скрипты передают друг другу, и один файл не должен уметь читать/писать чужие файлы, запускать процессы или дотягиваться до Java-классов через `luajava`.

Другие ограничения:

- рисовать можно **только** из `on_render` — вне виртуального холста вызовы `Rect`/`Text` работать не будут;
- скрипты не видят глобальные переменные друг друга;
- прямого доступа к сети, файлам и отправке пакетов нет.

---

## Обработка ошибок

**Ошибка при компиляции** (синтаксис, ошибка в теле файла или в `on_load`): скрипт не запускается, на карточке появляется красный текст ошибки, в чат приходит одно сообщение `Lua [имя]: текст`.

**Ошибка в рантайме** (внутри `on_render` или `on_tick`): скрипт **автоматически выключается**, чтобы не уронить весь HUD. Ошибка пишется на карточку и в чат, состояние сохраняется в `state.json`.

Одинаковый текст ошибки не спамится в чат повторно.

Починили файл → `Перезагрузить` на карточке (или просто сохраните файл, если включена `Авто-перезагрузка`) → включите тумблер обратно.

---

## Полный пример

Панель здоровья и брони с анимацией, привязкой к правому краю и корректной работой на любом мониторе.

```lua
-- health.lua
local WIDTH   = 190
local HEIGHT  = 58
local MARGIN  = 24
local PADDING = 12

local function bar(x, y, w, h, fraction, color)
    Rect.draw(x, y, w, h, h / 2, Color.rgba(255, 255, 255, 22))
    if fraction > 0.001 then
        Rect.draw(x, y, w * fraction, h, h / 2, color)
    end
end

function on_load()
    -- Разовая инициализация: тут можно посчитать константы
end

function on_render()
    if not Client.in_game() then return end

    -- Привязка к правому нижнему углу через API, а не через 1920/1080
    local x = Client.width() - WIDTH - MARGIN
    local y = Client.height() - HEIGHT - MARGIN

    -- Плавное появление панели
    local show = Anim.run("show", 1, 320, "cubic_out")
    y = y + (1 - show) * 20

    -- Фон: матовое стекло + тонкая рамка
    Rect.glass(x, y, WIDTH, HEIGHT, 9, Color.opacity(Color.rgba(10, 10, 14, 165), show), 16)
    Rect.outline(x, y, WIDTH, HEIGHT, 9, 1, Color.rgba(255, 255, 255, 16 * show))

    -- Голова и ник
    Image.head(x + PADDING, y + PADDING, 22, 4)
    Text.draw(Player.name(), x + PADDING + 30, y + PADDING + 2, 11,
        Color.opacity(Color.rgb(255, 255, 255), show), "semibold")

    -- Здоровье
    local hp, maxHp = Player.health(), Player.max_health()
    if maxHp <= 0 then maxHp = 20 end
    local frac = math.min(1, math.max(0, hp / maxHp))

    local hpColor = Color.lerp(Color.rgb(235, 70, 70), Color.rgb(90, 220, 120), frac)
    local filled  = Anim.run("hp", frac, 240, "quintic_out")

    local barX = x + PADDING + 30
    local barW = WIDTH - PADDING * 2 - 30

    bar(barX, y + PADDING + 18, barW, 4, filled, hpColor)

    Text.right(math.floor(hp + 0.5) .. " / " .. math.floor(maxHp),
        x + WIDTH - PADDING, y + PADDING + 2, 10, hpColor, "medium")

    -- Броня и сытость мелкой строкой
    local info = "броня " .. Player.armor() .. "   сытость " .. Player.hunger()
        .. "   " .. string.format("%.1f bps", Player.bps())
    Text.draw(info, x + PADDING, y + HEIGHT - 18, 8,
        Color.rgba(150, 150, 160, 255 * show), "regular")

    -- Предупреждение при низком HP: пульсирующая рамка
    local danger = Anim.run("danger", frac < 0.35 and 1 or 0, 260, "cubic_out")
    if danger > 0.01 then
        local pulse = Color.pulse(Color.rgba(235, 70, 70, 40), Color.rgba(235, 70, 70, 170), 900)
        Rect.outline(x, y, WIDTH, HEIGHT, 9, 1.5, Color.opacity(pulse, danger))
    end
end

function on_unload()
    -- Очистка, если что-то накапливали
end
```

---

## Частые вопросы

**Скрипт не рисуется.**
Проверьте по порядку: пилюля `Отрисовка HUD` включена → тумблер на карточке включён → статус карточки `работает` → на карточке нет красного текста → координаты внутри `0..Client.width()` / `0..1080`.

**Ничего не видно, когда открыт ClickGui.**
Так задумано. Включите пилюлю `Рисовать в меню`.

**Панель уезжает на другом мониторе.**
Вы захардкодили `1920`. Используйте `Client.width()`.

**Правки в файле не подхватываются.**
Включите `Авто-перезагрузка` или нажмите `Перезагрузить` на карточке. Авто-перезагрузка проверяет файлы раз в секунду и только у включённых скриптов.

**Русский текст — крякозябры.**
Сохраните файл в UTF-8 без BOM.

**Скрипт сам выключился.**
Значит в `on_render` или `on_tick` вылетела ошибка — её текст на карточке и в чате. Клиент выключает такой скрипт, чтобы не потерять весь HUD.

**Как сделать элемент по центру экрана.**
```lua
local x = Client.width() / 2
Text.center("по центру", x, Client.height() / 2, 14, Color.rgb(255, 255, 255))
```

**Как поделиться скриптом.**
Просто отправьте `.lua` файл. Получатель кладёт его в `avalora/scripts` и нажимает `Обновить`.
