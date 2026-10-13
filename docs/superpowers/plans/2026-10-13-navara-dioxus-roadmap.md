# Переход gev на Navara + Dioxus: дорожная карта

> Этот документ — спецификация верхнего уровня. Каждая фаза получает свой
> пошаговый план в `docs/superpowers/plans/` перед стартом. Фаза 0 уже
> расписана: [2026-10-13-phase0-navara-dioxus-spike.md](2026-10-13-phase0-navara-dioxus-spike.md).

## Цель

Заменить CesiumJS на [Navara](https://github.com/maplibre/navara) как движок
глобуса, а DOM-интерфейс на чистом JS — на [Dioxus](https://dioxuslabs.com)
(веб-рендерер, Rust → wasm), и постепенно перенести портируемую логику gev
в Rust, который собирается в тот же wasm-модуль, что и интерфейс.

## Целевая архитектура

```
index.html
 ├─ #gev-ui        Dioxus (crates/gev_ui → wasm): панели, HUD, поиск, настройки, i18n
 ├─ #globe         Navara ThreeView (свой wasm + Three.js): глобус, слои, эффекты
 └─ world overlay  JS, src/overlays/: подписи поверх глобуса, обновляются каждый кадр

JS-рантайм gev (как сейчас): createApplication, слои, голос, инструменты, сервер
   ▲  src/bridge/uiBridge.js: темы-снимки (plain data) + действия (схемы как у голоса)
   ▼
Rust (тот же wasm, что UI): crates/gev_core — логика, перенесённая из src/sources, src/data, src/tools
```

Принципы:

1. **Одна большая замена за раз.** Движок (фазы 1–3) и интерфейс (фаза 4)
   идут разными дорожками и сливаются в main независимо. Каждый шаг можно
   откатить отдельно.
2. **Сначала шов, потом замена.** Перед тем как писать код на Navara, слой
   переводится на движково-нейтральный интерфейс поверх Cesium без изменения
   поведения. Замена движка — это вторая реализация того же интерфейса.
3. **Флаг, а не ветка.** `?engine=cesium|navara` и настройка в локальном
   хранилище. Cesium остаётся рабочим до фазы 6.
4. **Тесты gev — это спецификация.** 530 файлов `*.test.mjs` (≈180 тыс. строк)
   переносятся вместе с кодом. Логика, переписанная на Rust, проверяется
   эталонными фикстурами, которые выдаёт текущая JS-реализация.
5. **Vite остаётся хостом.** Dev-сервер, middleware провайдеров из `server/`,
   npm-зависимости Navara. Rust собирается `cargo` + `wasm-bindgen` в
   `generated/` и импортируется из JS. `dx serve` не используется, `asset!()`
   из Dioxus не используется: CSS остаётся в пайплайне Vite.

## Исходные данные (замер 2026-10-13)

| Что | Сколько |
|---|---|
| Продакшен-JS в `src/` | ≈215 тыс. строк (`src/` целиком ≈395 тыс., из них ≈180 тыс. тесты) |
| Файлы, импортирующие `cesium` (без тестов) | ≈250, больше всего в `src/data`, `src/layers/*`, `src/ui` |
| `src/layers` | ≈113 тыс. строк, 25+ семейств слоёв |
| `src/ui` | ≈37 тыс. строк, чистый DOM (`createElement` ×250) |
| Наложение на 3D-тайлы (`ClassificationType.CESIUM_3D_TILE`/`BOTH`) | ≈20 файлов: traffic, submarineCables, installations, cyclones, directions, streetLevel, annotations, data |
| Navara | `@navaramap/three` 0.2.1 в npm, бета, MIT/Apache-2.0, Three.js + Rust/wasm-ядро на `bevy_ecs` 0.19 |
| Тулчейн на машине | cargo 1.99, dx 0.7.9, wasm-bindgen 0.2.127, node 26 |

Уже существующие швы, на которые опирается план:

- `createApplication` с четырьмя конструкторами (`docs/APPLICATION.md`) —
  `createScene` единственное место, где создаётся viewer.
- Портируемые графы без Cesium/DOM (`docs/CODE-BOUNDARIES.md`) и проверка
  `npm run check:boundaries` — по ним видно, что можно переносить в Rust первым.
- `src/app/stateChannel.js` публикует неизменяемые снимки plain data —
  готовый формат для моста в Rust.
- 31 схема действий в `src/voice/actionSchemas.js` — общий словарь действий
  для голоса и интерфейса.
- Подписи поверх глобуса уже вынесены в world-overlay host (`src/overlays/`),
  Cesium-подписи запрещены правилами репозитория.

## Фазы

Размеры условные: S — дни, M — 1–2 недели, L — несколько недель, XL — больше
месяца на одного инженера.

### Фаза 0 — проверка (M) · ворота GO/NO-GO

Отдельная страница `spike.html`, которая не трогает приложение. Измеряем
Navara против Cesium на одних и тех же сценах и проверяем связку
Dioxus ↔ JS. Подробный план — отдельный документ.

Ворота (все пункты обязательны для GO):

| # | Критерий | Порог |
|---|---|---|
| G1 | Производительность: 10 000 синтетических самолётов, сцены `global`, `city`, `orbit` | p95 времени кадра Navara ≤ 1,1 × Cesium на каждой сцене |
| G2 | Наложение линии на Google 3D-тайлы | Есть встроенный способ или прототип, который рисует линию по крышам и земле без «висения» и провалов |
| G3 | Доступ к Google | Работает маршрут с ключом; для маршрута с токенами (`Authorization: Bearer` + продление) есть рабочий путь или понятный патч |
| G4 | Точность | Нет видимого дрожания объектов на дистанции 500 м |
| G5 | Мост Dioxus ↔ JS | Снимок доходит из JS в Rust, действие доходит из Rust в JS; gzip-размер `gev_ui.wasm` ≤ 1 МБ |
| G6 | Старт | Время до первых тайлов Navara ≤ 1,2 × Cesium |

NO-GO по G1/G2 останавливает переезд движка; фаза 4 (Dioxus) при этом
остаётся возможной поверх Cesium.

### Фаза 1 — швы на Cesium, без изменения поведения (L)

1a. **Геометрия без Cesium.** Новый модуль `src/geo/`: эллипсоид WGS84,
`cartesian`, `cartographic`, ENU-фреймы, геодезические расстояния, матрицы.
Логика слоёв (`policy.js`, `motion.js`, `queries.js`, `src/data/*`) переходит
на него с `Cesium.Cartesian3`/`Cartographic`/`Ellipsoid`/`Math`/`Matrix4`.
Каждая функция `src/geo/` покрывается тестом, где эталон — результат Cesium
на тех же входах. Тот же набор эталонов сохраняется в `fixtures/geo/*.json`
для будущего крейта `gev_geo`.

1b. **Интерфейс движка `src/globe/engine.js`.** Контракт (JSDoc-типы):

- камера: `getCamera`, `setCamera`, `flyTo`, `lookAt`, `follow`, события движения;
- выбор: `pickObject(screenXY)`, `pickPosition(screenXY)`;
- рельеф: `sampleHeight`, `sampleHeightMostDetailed`;
- проекция: `worldToScreen`, `screenToWorld` (для world-overlay host);
- источники: Google 3D-тайлы, Ion, рельеф, растровые подложки, атрибуции;
- кадр: `onPreRender`, `onPostRender`, `requestRender`;
- эффекты: регистрация пост-эффектов визуальных пресетов.

Реализация `src/globe/cesium/` переносит туда `src/app/viewer.js`,
`src/app/scene.js` (часть про viewer), `src/maps/google3d.js`,
`src/renderGovernor.js`, `src/creditKeyboard.js`.

1c. **Рендереры слоёв.** Каждое семейство слоёв получает интерфейс рендерера
(например, `createFlightsRenderer(engine)`), а весь код Cesium семейства
уезжает в `src/layers/<family>/render/cesium.js`. Контроллеры, политики и
данные говорят только с интерфейсом. Порядок: flights → military → vessels →
satellites → остальное.

1d. **Своё время.** `src/time/clock.js` вместо `Cesium.Clock`/`JulianDate`.

1e. **Правило в `scripts/check-import-directions.mjs`:** `cesium` можно
импортировать только из `src/globe/cesium/**` и `src/layers/*/render/cesium*.js`.

Выход: правило из 1e проходит; `npm test`, `check:boundaries`, `build`, все
`qa:*` зелёные; скриншоты QA совпадают с main.

### Фаза 2 — движок Navara за флагом (L)

2a. `src/globe/navara/` — вторая реализация `engine.js`: `ThreeView`,
`DefaultPlugin`, Google-тайлы (ключ и токены), Ion, рельеф quantized-mesh,
растровые подложки для `MapStackController`, `AttributionPlugin` в контейнер
атрибуций gev, камера, выбор, рельеф, проекция.

2b. Набор примитивов `src/globe/navara/primitives/`:

- `movers` — `InstancedMesh` + `PickableInstancedMeshWrapper`, позиции
  относительно опорной точки (RTE) для точности;
- `billboards` — инстансированные квадраты с атласом иконок;
- `polylines`, `polygons` — `vector`-слой для статичного GeoJSON,
  свои меши для часто меняющегося;
- `models` — `InstancedGltfModelMeshDesc`;
- `drape` — способ из G2 (встроенный или свой).

2c. Визуальные пресеты: `src/ui/visualEffects.js`, `src/styles/snow.js`,
`src/styles/anime.js`, `src/scopeMask.js`, `src/cyberSonarGpu.js`,
`src/cockpitCloudEffects.js`, `src/bloom.js` — шейдеры с `czm_*`
переписываются в эффекты `postprocessing` и регистрируются как `EffectDesc`.

2d. World-overlay host переходит на `engine.worldToScreen` и работает на
обоих движках.

Выход: `?engine=navara` загружает приложение: карта, камера, поиск, HUD,
share-ссылки, маршруты работают; слои пока отключены.

### Фаза 3 — перенос слоёв на Navara (XL)

Каждый слой получает `render/navara.js` с тем же интерфейсом, что
`render/cesium.js`. Для каждого слоя — чек-лист паритета (вид, клик, фокус,
трекинг, голосовые запросы, share-ссылка, скриншоты `qa:*` с `?engine=navara`).

| Волна | Слои | Главная сложность |
|---|---|---|
| W1 движущиеся | flights, military, localAdsb, vessels, satellites (+orbit, celestialRing), launches | 10k+ объектов, интерполяция, трекинг камерой |
| W2 точки | earthquakes, firms, cctv, alpr, bikeshare, radio, installations, recentImagery, transit | Иконки, кластеры, карточки |
| W3 наложенные | traffic, submarineCables, perimeters, cyclones, directions, annotations, adminBoundaries | Наложение на 3D-тайлы (G2) |
| W4 особые | weather, wind (GRIB, частицы), streetLevel (Mapillary + камера Three), cockpit, director/scenes, cyberSonar | Свои шейдеры и камера |

Выход: все слои на Navara; `engine=navara` по умолчанию; Cesium — запасной
вариант на один релиз.

### Фаза 4 — интерфейс на Dioxus (XL, параллельно с фазами 2–3)

Можно начинать сразу после фазы 0: интерфейс говорит с мостом, а не с
движком.

4a. **Инфраструктура.** Cargo workspace в корне (`Cargo.toml`, `crates/`),
`scripts/build-ui-wasm.mjs` (cargo → wasm-bindgen → wasm-opt → `generated/`),
`npm run dev` параллельно запускает пересборку wasm, CI: `cargo fmt --check`,
`cargo clippy -D warnings`, `cargo test`, сборка wasm, `setup-doctor`
проверяет тулчейн. Pinokio получает собранный wasm, Rust нужен только
разработчикам.

4b. **Мост.** Темы — существующие каналы состояния (`stateChannel`
навигации, поиска, режиссёра, снимки оболочки, каталог слоёв и
`feedState`). Действия — схемы `src/voice/actionSchemas.js` плюс действия
только для интерфейса. Rust-типы действий генерируются из JSON Schema
(крейт `typify`). Rust-типы снимков пишутся вручную; контрактный тест: JS
выгружает фикстуры снимков, `cargo test` их десериализует.

4c. **i18n.** Rust читает те же `src/i18n/locales/*.json` (`include_str!`),
с тем же поиском ключей и правилами множественного числа. Проверка
непереведённых строк расширяется на `.rs` (`t!("…")`).

4d. **Перенос панелей** от слабо связанных к сильно связанным: Provider
Settings (`keySetup`) → атрибуции → список слоёв и `mapStackChips` → поиск
(`locationSearch`) → показания HUD → правая панель (`rightPanelRail`) →
режиссёр и сцены → кабина (cockpit) → street level и радио. На каждую панель:
компонент Dioxus монтируется в существующий слот, старый DOM-модуль
удаляется в том же PR, `*Markup.test.mjs` переписываются в Rust-тесты через
`dioxus-ssr`.

4e. **Доступность и клавиатура** — паритет с `hudA11y`,
`keyboardFocusStyles`, `panelEscape`, `contextTabKeyboard`.

Выход: DOM-модули `src/ui/*` удалены, `applicationShell` создаёт корень
Dioxus.

### Фаза 5 — логика в Rust (XL, постепенно, после 4b)

Порядок задаётся портируемыми графами из `CODE-BOUNDARIES.md`:

1. `src/sources/*`, нормализация записей flights/military/vessels, ingestion;
2. `feedState`, lifecycle;
3. `src/search` (геокодинг, адаптеры);
4. `src/tools/queries`;
5. исполнение голосовых действий и сессия (WebRTC остаётся браузерным через
   `web-sys`);
6. режиссёр, data packs, share-ссылки;
7. спутники: `sgp4` вместо `satellite.js`, сверка эталонами.

Метод на каждый модуль: JS выдаёт эталонные фикстуры → Rust-порт их
воспроизводит → вызовы переключаются на wasm → JS-модуль удаляется.

Остаются на JS осознанно: адаптер Navara и рендереры слоёв (Three.js),
world-overlay host, Mapillary, `hls.js`, `server/` на Node (перевод на
`axum` — отдельное решение).

### Фаза 6 — удаление Cesium и уборка (M)

Удалить `cesium`, `vite-plugin-cesium`, Cesium-настройки `build/vite.js`,
`src/globe/cesium/`, `render/cesium.js`; обновить `package.json#exports`,
`scripts/package-boundaries.json`, `docs/APPLICATION.md`,
`docs/CODE-BOUNDARIES.md`, `docs/PERFORMANCE.md`, `THIRD_PARTY_NOTICES.md`,
Pinokio-лаунчер.

## Зависимости между фазами

```
Ф0 ──GO──► Ф1 ──► Ф2 ──► Ф3 ──► Ф6
  └──────► Ф4 (4a, 4b) ──► Ф4 (4c–4e)
                 └──────► Ф5
```

После фазы 1 движок и интерфейс идут параллельно разными людьми или агентами.

## Риски

| Риск | Вероятность | Что делаем |
|---|---|---|
| Navara в бете, ломающие изменения в минорных версиях | Высокая | Точная версия в `package.json`, обновление отдельным PR с прогоном `qa:*`; при необходимости форк |
| Нет наложения на 3D-тайлы | Средняя | Проверка в Ф0 (G2); свой drape-проход по буферу глубины; предложить в апстрим |
| `source.update()` пересоздаёт источник целиком | Подтверждено | Всё живое — через свои меш-дескрипторы (2b), а не через `vector`-слой |
| Глобальный масштаб: Navara в основном показывают на масштабе города | Средняя | Сцены `global` и `orbit` в Ф0 |
| Токены Google с продлением | Средняя | G3; патч в `event/index.ts` апстриму, если надо |
| Dioxus web без `dx serve`: нет горячей перезагрузки RSX | Подтверждено | `cargo watch` + перезагрузка страницы; пересмотреть, если станет больно |
| Две системы сборки, два языка | Подтверждено | Один вход `npm run dev`/`npm run build`; doctor проверяет тулчейн |
| Размер загрузки: wasm Navara + wasm UI + Three.js | Средняя | Бюджет в `docs/PERFORMANCE.md`, замер в Ф0 (G5, G6) |
| Регрессии видимого поведения при переносе слоёв | Высокая | Чек-лист паритета на каждый слой, `qa:*` на обоих движках |

## Что нужно решить до старта фазы 1

1. Итог фазы 0: GO/NO-GO по G1–G6 (`docs/migration/phase0-report.md`).
2. Кто держит дорожку движка, а кто дорожку интерфейса.
3. Идём ли мы в апстрим Navara с drape и токенами или держим форк.
