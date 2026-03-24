# Техническое задание на Frontend-разработку (React/MobX/AntD)
## ModernGlass ShapeViewer - Миграция с Delphi на React

---

## 5.2 Специальные задачи

### 5.2.1 Загрузка контролек при входе в заказ

**Файлы Delphi:**
- `OutputUnits\CheckListUnit.pas` (429 строк) — `TCheckListFrame`, `CheckGrid`, `ProjectTextRichEdit`
- `MainUnit.pas` — `LoadCheckList()`, `CheckListErrorMode()`
- `DataModules\u_DrawingDM.pas` — `GetCalcItemList()`
- `DataModules\u_TransactionDM.pas` — `SaveCheckList()`

**Таблицы БД:**
- `mg_order.check_list` — список сообщений контролек
- `mg_order.check_list_tr` — переводы текстов контролек
- `mg_order.msg_row` — справочник сообщений
- `mg_order.ITEM_HASH` — хэши для проверки изменений

**Описание задачи:**
При загрузке заказа отображается список контролек (сообщений об ошибках и предупреждениях) для каждой позиции.

**Требования к UI:**
- Фрейм с 2 панелями: левая (CheckGrid с сообщениями), правая (ProjectTextRichEdit с текстом проекта)
- Разделение сообщений на 2 секции: "Ошибки" (msg_type=1) и "Предупреждения" (msg_type=2)
- Кнопка "Замена стекол" отображается только если есть "Повтор стекла"

**Функциональные требования:**
- Загрузка контролек: `GET /api/orders/{orderNo}/items/{itemNo}/checklist`
- Загрузка текста проекта: `GET /api/orders/{orderNo}/project-text`
- Проверка на критические ошибки (msg_type = 1)
- Проверка на "Повтор стекла": `function CheckRepeatGlassMSGExists(chklist: TCheckList): boolean`

**Состав работ:**
1. API сервис для загрузки контролек
2. MobX store: загрузка данных, вычисление ошибок/предупреждений
3. UI компонент: фрейм с 2 колонками (список + текст проекта)
4. Интеграция с главной формой (автооткрытие при выборе позиции)

**Оценка: 10 часов**

---

### 5.2.2 Замена стекол на МГ

**Файлы Delphi:**
- `OutputUnits\CheckListUnit.pas` — `ChangeGlassButtonClick()`, `TChangeGlass`, `GetNewCode()`
- `InputUnits\FormChangeGlass\GlasInputUnit.pas` — `TGlasInputForm`
- `InputUnits\FormChangeGlass\LayerInputUnit.pas` — `TLayerInputForm`
- `InputUnits\u_ChangeGlassCodesFrame.pas` — `TChangeGlassCodesFrame`

**Таблицы БД:**
- `mg_order.auf_pos` — позиции заказа
- `mg_order.auf_pos_comp` — компоненты позиций (стекла)
- `mg_glass.glass` — справочник стекол
- `mg_glass.mg_replacement` — таблица замен на МГ

**Описание задачи:**
При обнаружении повторяющихся стекол в заказе система предлагает замену на МГ-аналоги.

**Требования к UI:**
- Модальное окно со списком уникальных стекол
- Select для выбора замены из МГ-списка
- Превью изменений (какие позиции изменятся)

**Функциональные требования:**
- Загрузка позиций: `GET /api/orders/{orderNo}/items`
- Загрузка МГ-замен: `GET /api/glass/{glCode}/mg-replacements`
- Логика замены: `function GetNewCode(gl_code: integer; mg_glass: boolean): integer`
- Применение замен: `POST /api/orders/{orderNo}/glass-changes`

**Состав работ:**
1. API сервис для загрузки МГ-замен
2. MobX store: загрузка данных, формирование превьера
3. UI компонент: модальное окно с 2 колонками
4. Валидация перед сохранением

**Оценка: 14 часов**

---

### 5.2.3 Постобработка при входе/получении заказа

**Файлы Delphi:**
- `DataModules\u_DrawingDM.pas` — `CalcAndSaveDataForOrder()`, `TCalcOrderThread`, `TSaveCalcOrderThread`
- `ProgressUnit.pas` — `TProgressForm`
- `MainUnit.pas` — `RecalcOrderClick()`

**Таблицы БД:**
- `mg_order.calc_dimensions` — расчетные размеры
- `mg_order.calc_edges` — значения по сторонам
- `mg_order.calc_params` — параметры расчета
- `mg_order.ITEM_HASH` — хэши для проверки изменений

**Описание задачи:**
После загрузки заказа выполняется расчет параметров (геометрия, ступени, снятие покрытия, логотипы).

**Требования к UI:**
- Модальное окно с прогресс-баром (0-100%)
- Отображение текущего шага расчета
- Лог выполнения (список сообщений)
- Кнопка "Отменить"

**Функциональные требования:**
- Запуск расчета: `POST /api/orders/{orderNo}/calculate`
- WebSocket для прогресса: `/api/jobs/{jobId}/ws`
- Отмена расчета: `POST /api/jobs/{jobId}/cancel`

**Состав работ:**
1. API сервис для запуска расчета
2. WebSocket сервис для получения прогресса
3. MobX store: управление статусом расчета, прогресс, лог
4. UI компонент: модальное окно с прогрессом и логом

**Оценка: 12 часов**

---

## 5.3 Особые задачи

### 5.3.1 Job расчёта параметров заказа + рассылка

**Файлы Delphi:**
- `MainUnit.pas` — `AutoCalcTimerTimer`, `RecalcDataForOrder`
- `DataModules\u_DrawingDM.pas` — `StartAutoCalc`, `CalcAndSaveDataForOrder()`
- `DataModules\u_DrawingDM.pas` — `TCalcOrderThread`, `TSaveCalcOrderThread`

**Таблицы БД:**
- `mg_order.calc_dimensions` — расчетные размеры
- `mg_order.calc_edges` — значения по сторонам
- `mg_order.calc_params` — параметры расчета
- `mg_order.ITEM_HASH` — хэши для проверки изменений

**Описание задачи:**
Автоматический расчет параметров заказа по таймеру с рассылкой уведомлений.

**Требования к UI:**
- Панель задач расчета (список job + прогресс)
- Индикация статуса (pending/running/complete/failed)
- Лог выполнения
- Кнопка "Отменить"

**Функциональные требования:**
- Запуск job: `POST /api/jobs/calculate`
- Polling статуса: `GET /api/jobs/{jobId}/status`
- WebSocket: `/api/jobs/{jobId}/ws`
- Рассылка уведомлений (WebSocket)

**Состав работ:**
1. API сервис для запуска расчета
2. Job queue (Bull/Redis) на backend
3. WebSocket сервис для уведомлений
4. UI компонент: панель задач расчета

**Оценка: 20 часов**

---

### 5.3.2 Выполнение кеширования и хеширования

**Файлы Delphi:**
- `DataModules\u_DrawingDM.pas` — `GetCalcItemList()`
- `DataModules\u_TransactionDM.pas` — `SaveHash()`

**Таблицы БД:**
- `mg_order.ITEM_HASH` — хэши для проверки изменений

**Описание задачи:**
Кэширование результатов расчета геометрии для ускорения повторных обращений.

**Требования к UI:**
- Панель статистики кэша (hits/misses/size)
- Кнопка "Очистить кэш"
- Инвалидация по заказу

**Функциональные требования:**
- Проверка хэша: `GET /api/cache/check/{orderNo}/{itemNo}`
- Инвалидация: `POST /api/cache/invalidate/{orderNo}`
- Статистика: `GET /api/cache/stats`

**Состав работ:**
1. API сервис для управления кэшем
2. Redis кэширование на backend
3. UI компонент: панель статистики

**Оценка: 12 часов**

---

### 5.3.3 Реализация расчёта набора геометрий для различных этапов

**Файлы Delphi:**
- `OutputUnits\CanvasFrameUnit.pas` (50,878 строк) — `TCanvasFrame`, `TCanvasDrawMode`
- `drawing_types.pas` — `TCanvasDrawMode` (All, Decoating_all, Paint_Edges_all, TermLogo, Profiles, CNC_Input, SpecShapeInput...)

**Таблицы БД:**
- `mg_order.shape_lines` — линии
- `mg_order.shape_arcs` — дуги
- `mg_order.shape_ellipses` — эллипсы

**Описание задачи:**
Расчет геометрии для различных режимов отрисовки (13 режимов).

**Требования к UI:**
- Переключение режимов отрисовки
- Кэширование результатов расчета

**Функциональные требования:**
- API: `GET /api/geometry?mode={mode}&orderNo={orderNo}&itemNo={itemNo}`
- 13 режимов: All, Decoating_all, Paint_Edges_all, Paint_all, TermLogo, Profiles, CNC_Input, SpecShapeInput...

**Состав работ:**
1. API сервис для расчета геометрии
2. 13 endpoint для каждого режима
3. Konva.js отрисовка

**Оценка: 40 часов**

---

### 5.3.4 Поддержка процесса моллирования

**Файлы Delphi:**
- `OutputUnits\CanvasFrameUnit.pas` — `FillCurvedData`, `ChangeGeometryParamForCurvedGlass`
- `DataModules\u_DrawingDM.pas` — `getHeatTime()`

**Таблицы БД:**
- `mg_proc_mode.HEAT_TIME_PARAMS` — параметры нагрева
- `mg_order.curved_glass` — гнутые стекла

**Описание задачи:**
Расчет параметров моллирования (время нагрева, геометрия гнутого стекла).

**Требования к UI:**
- Панель с параметрами моллирования
- Input: ширина, длина
- Output: время нагрева

**Функциональные требования:**
- Расчет времени: `GET /api/molliering/heat-time?width={w}&length={l}`
- Расчет геометрии: `GET /api/molliering/curved-geometry/{orderNo}/{itemNo}`

**Состав работ:**
1. API сервис для расчета моллирования
2. UI компонент: панель параметров
3. Konva.js: отрисовка гнутой геометрии

**Оценка: 20 часов**

---

### 5.3.5 Триггер на изменение заказа в order

**Файлы Delphi:**
- `TOC\Threads\ThreadUpdate.pas`
- `TOC\Threads\ThreadUpdateDM.pas`

**Таблицы БД:**
- `mg_order.auf_kopf` — заголовки заказов
- `mg_order.auf_pos` — позиции заказов

**Описание задачи:**
Отслеживание изменений в заказе и автоматическое обновление UI.

**Требования к UI:**
- Notification при изменении заказа
- Автообновление данных

**Функциональные требования:**
- WebSocket: `/api/orders/{orderNo}/changes`
- События: order.changed, item.added, item.removed

**Состав работ:**
1. WebSocket сервис для уведомлений
2. MobX store: обработка событий
3. UI notification

**Оценка: 8 часов**

---

### 5.3.6 Событие: создание прогона в prod

**Файлы Delphi:**
- `PrdSequence.pas` — `ProdQueue` (TOraQuery)
- `TOC\Frames\masterProdQueueFram.pas`
- `TOC\Frames\BatchSeqFram.pas`

**Таблицы БД:**
- `mg_prod.prod_queue` — производственная очередь
- `mg_prod.batch_seq` — прогоны

**Описание задачи:**
Создание прогона и добавление в производственную очередь.

**Требования к UI:**
- Modal создания прогона
- Select: номер прогона, заказы, группа оборудования

**Функциональные требования:**
- Создание: `POST /api/batches/create`
- Добавление в очередь: `POST /api/prod-queue/add`

**Состав работ:**
1. API сервис для создания прогона
2. UI компонент: modal создания
3. Интеграция с производственной очередью

**Оценка: 12 часов**

---

## 5.4 Главная форма. Производственный чертёж

**Файлы Delphi:**
- `MainUnit.pas` (2,183 строки) — `TMainForm`, `DataGrid`, `LeftPanel`, `RightPanel`
- `MainUnit.dfm` (5,517 строк)
- `OutputUnits\CanvasFrameUnit.pas` (50,878 строк) — `TCanvasFrame`

**Таблицы БД:**
- `mg_order.auf_kopf` — заголовки заказов
- `mg_order.auf_pos` — позиции заказов
- `mg_order.shpw_shape` — фигуры
- `mg_prod.batch_seq` — прогоны

**Описание задачи:**
Основная форма приложения для загрузки и отображения заказов/прогонов.

**Требования к UI:**
- LeftPanel (245px): `OrderEdit`, `LoadButton`, `DataGrid`
- RightPanel: `PageControl1` с вкладками
- Кнопки: `RotateButton`, `MirrorButton`, `KompasSaveButton`, `PdfSaveButton`

**Функциональные требования:**
- Загрузка по заказу: `GET /api/orders/{orderNo}/items`
- Поворот: `POST /api/orders/{orderNo}/items/{itemNo}/rotate`
- Зеркало: `POST /api/orders/{orderNo}/items/{itemNo}/mirror`
- Экспорт DXF: `POST /api/orders/{orderNo}/items/{itemNo}/export/dxf`

**Состав работ:**
1. API сервис для загрузки данных
2. MobX store: orderItems, selectedItems, loading
3. UI компонент: Layout с Sider + Content
4. Вкладки с чертежами (Konva.js)

**Оценка: 34 часа**

---

## 5.5 Заказ

### 5.5.1 Общее требование для всех инструментов

**Файлы Delphi:**
- `InputUnits\TechTextInputUnit.pas` (837 строк)
- `DataModules\BaseDB\u_BaseDrawingDM.pas`

**Таблицы БД:**
- `mg_order.techtext`
- `mg_order.tt_assign`

**Описание задачи:**
Все модальные формы заказа должны иметь единый стиль и поведение.

**Требования:**
- Единый базовый компонент (HOC)
- Кнопки OK/Cancel
- Валидация данных

**Состав работ:**
1. HOC для модальных окон
2. Базовый интерфейс IOrderModalProps
3. Валидация форм

**Оценка: 4 часа**

---

### 5.5.2 Дополнительные условия по заказу (F4)

**Файлы Delphi:**
- `InputUnits\ConditionMarkUnit.pas` — `TConditionMarkFrame`

**Таблицы БД:**
- `mg_order.techtext` — технические тексты
- `mg_order.tt_assign` — назначения текстов
- `mg_prod.mach_area` — области оборудования

**Описание задачи:**
Форма для ввода дополнительных условий по заказу.

**Требования к UI:**
- 4 TextArea поля (ОКП, Доп. текст, Производство, Доп. производство)
- Select "Область оборудования"

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/tech-text`
- Сохранение: `PUT /api/orders/{orderNo}/tech-text`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. UI компонент: фрейм с Form + TextArea

**Оценка: 10 часов**

---

### 5.5.3 Особые указания (F5)

**Файлы Delphi:**
- `InputUnits\TechTextInputUnit.pas` — `TTechTextInputFrame`, `OKSRichEdit`, `ProductionRichEdit`

**Таблицы БД:**
- `mg_order.techtext` — технические тексты

**Описание задачи:**
Форма для ввода особых указаний по заказу.

**Требования к UI:**
- 2 TextArea поля (Основной текст, Доп. указания)

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/special-instructions`
- Сохранение: `PUT /api/orders/{orderNo}/special-instructions`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. UI компонент: фрейм с TextArea

**Оценка: 8 часов**

---

### 5.5.4 Параметры 99 формы

**Файлы Delphi:**
- `InputUnits\SpecShapeInputUnit.pas` (1,033 строки) — `TSpecShapeInputFrame`, `DrawPanel`, `DataGrid`, `EINSTGrid`

**Таблицы БД:**
- `mg_order.shape_lines` — линии 99 фигуры
- `mg_order.shape_arcs` — дуги 99 фигуры
- `mg_order.shape_ellipses` — эллипсы 99 фигуры

**Описание задачи:**
Форма для ввода параметров 99 фигуры.

**Требования к UI:**
- `DrawPanel`: Canvas с геометрией
- `RPanel`: 4 ComboBox для выбора сторон
- `DataGrid`: параметры (SwPane, S_X, S_Y)
- `EINSTGrid`: настройки Einst

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/items/{itemNo}/shape99`
- Сохранение: `PUT /api/orders/{orderNo}/items/{itemNo}/shape99`
- Валидация: `POST /api/orders/{orderNo}/items/{itemNo}/shape99/validate`

**Состав работ:**
1. API сервис для геометрии
2. UI компонент: фрейм с Canvas + панель параметров
3. Konva.js: отрисовка линий, дуг, эллипсов

**Оценка: 20 часов**

---

### 5.5.5 Опорная сторона

**Файлы Delphi:**
- `InputUnits\SupportSideInputUnit.pas` — `TSupportSideFrame`
- `InputUnits\ChangeSupSideUnit.pas` — `TChangeSupSideForm`

**Таблицы БД:**
- `mg_order.sup_side` — опорные стороны

**Описание задачи:**
Форма для выбора опорной стороны и отступов.

**Требования к UI:**
- Canvas с контуром + маркер стороны
- Select "Опорная сторона" (1-4)
- InputNumber: отступы

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/sup-side`
- Сохранение: `PUT /api/orders/{orderNo}/sup-side`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. Konva.js: контур + маркер стороны

**Оценка: 8 часов**

---

### 5.5.6 Клапан давления

**Файлы Delphi:**
- `InputUnits\KlapanInputUnit.pas` — `TKlapanInputFrame`
- `InputUnits\ChangeKlapanPositionUnit.pas` — `TChangeKlapanPositionForm`

**Таблицы БД:**
- `mg_order.pressure_valve` — клапаны давления

**Описание задачи:**
Форма для указания параметров клапана давления.

**Требования к UI:**
- Switch "Наличие клапана"
- Select "Сторона" (1-4)
- InputNumber "Расстояние от угла"

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/pressure-valve`
- Сохранение: `PUT /api/orders/{orderNo}/pressure-valve`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. UI компонент: фрейм с Form

**Оценка: 4 часа**

---

### 5.5.7 Краевое окрашивание (F6)

**Файлы Delphi:**
- `InputUnits\EdgePaintingInputUnit.pas` — `TEdgePaintingInputFrame`
- `InputUnits\EdgePaintingMassInputUnit.pas` — `TEdgePaintingMassInputForm`
- `OutputUnits\EdgePaintingUnit.pas` (206 строк) — `TEdgePaintingFrame`

**Таблицы БД:**
- `mg_order.edge_painting` — краевое окрашивание
- `mg_order.edge_painting_values` — значения по сторонам

**Описание задачи:**
Форма для указания параметров краевого окрашивания.

**Требования к UI:**
- Canvas с подсветкой сторон
- Radio "Сторона" (1 или 2)
- InputNumber для каждой стороны (1-8)

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/edge-painting`
- Сохранение: `PUT /api/orders/{orderNo}/edge-painting`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. Konva.js: подсветка сторон

**Оценка: 12 часов**

---

### 5.5.8 Снятие покрытия (F7)

**Файлы Delphi:**
- `InputUnits\DecoatingInputUnit.pas` — `TDecoatingInputFrame`
- `InputUnits\SetDecoatingValueUnit.pas` — `TSetDecoatingValueForm`
- `OutputUnits\DecoatingUnit.pas` (191 строка) — `TDecoatingFrame`

**Таблицы БД:**
- `mg_order.decoating` — снятие покрытия
- `mg_order.decoating_values` — значения снятия

**Описание задачи:**
Форма для указания параметров снятия покрытия.

**Требования к UI:**
- Canvas с контуром со смещением (пунктир)
- Input "Величина снятия"
- Input "Смещение шлифовки"

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/decoating`
- Сохранение: `PUT /api/orders/{orderNo}/decoating`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. Konva.js: контур со смещением

**Оценка: 10 часов**

---

### 5.9 Логотип (F8)

**Файлы Delphi:**
- `InputUnits\TermLogoInputUnit.pas` — `TTermLogoInputFrame`
- `InputUnits\TermLogoPosInputUnit.pas` — `TTermLogoPosInputFrame`
- `OutputUnits\TermLogoFrameUnit.pas` (265 строк) — `TTermLogoFrame`
- `OutputUnits\TermLogoMonUnit.pas` — `TTermLogoMonFrame`

**Таблицы БД:**
- `mg_order.term_logo` — логотипы закалки
- `mg_order.lam_logo` — логотипы ламината
- `mg_logo.types` — типы логотипов
- `mg_logo.templates` — шаблоны логотипов

**Описание задачи:**
Форма для выбора и размещения логотипа.

**Требования к UI:**
- Canvas с логотипом
- Select "Тип логотипа"
- Radio "Цвет" (Черный/Белый/Матовый)
- Radio "Положение"
- InputNumber: X, Y, Угол
- Switch "Зеркально"

**Функциональные требования:**
- Загрузка типов: `GET /api/orders/{orderNo}/logo-types`
- Загрузка данных: `GET /api/orders/{orderNo}/logo`
- Сохранение: `PUT /api/orders/{orderNo}/logo`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. Konva.js: отрисовка логотипа

**Оценка: 28 часов**

---

### 5.5.10 Профили (F9)

**Файлы Delphi:**
- `InputUnits\ProfilesInputFormUnit.pas` — `TProfilesInputForm`
- `InputUnits\PartProfilesInputFormUnit.pas` — `TPartProfilesInputForm`
- `OutputUnits\ProfilesFullScreenUnit.pas` — `TProfilesFullScreenForm`

**Таблицы БД:**
- `mg_order.ig_profiles` — профили стеклопакетов
- `mg_order.ig_profile_parts` — части профилей
- `mg_directory.profiles` — справочник профилей

**Описание задачи:**
Форма для отображения профилей изделия.

**Требования к UI:**
- Canvas с линиями профилей
- Список профилей (выбор кликом)
- Параметры: длина, ширина, высота, кол-во

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/profiles`
- Сохранение: `PUT /api/orders/{orderNo}/profiles`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. Konva.js: отрисовка профилей

**Оценка: 32 часа**

---

### 5.5.11 Ввод CNC чертежей

**Файлы Delphi:**
- `InputUnits\CNCDrawingInputUnit.pas` — `TCNCDrawingInputForm`

**Таблицы БД:**
- `mg_order.cnc_drawings` — CNC чертежи
- `mg_order.cnc_blocks` — блоки CNC
- `mg_order.cnc_params` — параметры CNC

**Описание задачи:**
Форма для загрузки CNC чертежей (DXF).

**Требования к UI:**
- Canvas с CNC геометрией
- Кнопка "Загрузить DXF"
- Список блоков

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/cnc`
- Загрузка DXF: `POST /api/cnc/upload`

**Состав работ:**
1. API сервис для загрузки/сохранения
2. DXF парсер (backend)
3. Konva.js: отрисовка CNC геометрии

**Оценка: 20 часов**

---

### 5.5.12 Открыть заказ 1С

**Файлы Delphi:**
- `DataModules\BaseDB\u_BaseDrawingDM.pas` — `OpenOrderIn_1CArhive()`
- `MainUnit.pas` — `OpenOrderIn1CArhiveClick()`

**Таблицы БД:**
- `mg_order.auf_kopf` — заголовки заказов

**Описание задачи:**
Форма для открытия заказа в архиве 1С.

**Требования к UI:**
- Модальное окно подтверждения
- Индикация статуса

**Функциональные требования:**
- Открытие: `POST /api/1c/archive/open`

**Состав работ:**
1. API сервис для открытия заказа в 1С
2. UI компонент: подтверждение

**Оценка: 6 часов**

---

### 5.5.13 Отчет. Список шпросов

**Файлы Delphi:**
- `MainUnit.pas` — `GetSprListClick()`
- `OutputUnits\CanvasFrameUnit.pas` — `GetSprInfo`

**Таблицы БД:**
- `mg_order.sprosse` — шпросы
- `mg_order.sprosse_parts` — части шпросов
- `mg_directory.sprosse` — справочник шпросов

**Описание задачи:**
Форма для отображения списка шпросов.

**Требования к UI:**
- Canvas с линиями шпроссов
- Таблица: №, Описание, Ширина, Кол-во
- Кнопка "Экспорт"

**Функциональные требования:**
- Загрузка: `GET /api/orders/{orderNo}/sprosse`
- Экспорт: `GET /api/orders/{orderNo}/sprosse/export`

**Состав работ:**
1. API сервис для загрузки/экспорта
2. Konva.js: отрисовка шпроссов

**Оценка: 10 часов**

---

### 5.5.14 Рассчитать заказ

**Файлы Delphi:**
- `MainUnit.pas` — `RecalcDataForOrder`
- `DataModules\u_DrawingDM.pas` — `CalcAndSaveDataForOrder()`, `TCalcOrderThread`

**Таблицы БД:**
- `mg_order.calc_dimensions` — расчетные размеры
- `mg_order.calc_edges` — значения по сторонам
- `mg_order.calc_params` — параметры расчета
- `mg_order.ITEM_HASH` — хэши

**Описание задачи:**
Форма для запуска расчета параметров заказа.

**Требования к UI:**
- Прогресс-бар (0-100%)
- Текущий шаг расчета
- Лог выполнения
- Кнопка "Отменить"

**Функциональные требования:**
- Запуск: `POST /api/orders/{orderNo}/calculate`
- WebSocket: `/api/jobs/{jobId}/ws`

**Состав работ:**
1. API сервис для запуска расчета
2. WebSocket сервис для прогресса
3. UI компонент: модальное окно с прогрессом

**Оценка: 16 часов**

---

## Сводная таблица оценки Frontend

| Раздел | Пункт | Задача | Часы |
|--------|-------|--------|------|
| **5.2** | | **Специальные задачи** | |
| | 5.2.1 | Загрузка контролек | 10 |
| | 5.2.2 | Замена стекол на МГ | 14 |
| | 5.2.3 | Постобработка заказа | 12 |
| **5.3** | | **Особые задачи** | |
| | 5.3.1 | Job расчета + рассылка | 20 |
| | 5.3.2 | Кеширование и хеширование | 12 |
| | 5.3.3 | Расчет геометрии (этапы) | 40 |
| | 5.3.4 | Процесс моллирования | 20 |
| | 5.3.5 | Триггер изменения заказа | 8 |
| | 5.3.6 | Создание прогона в prod | 12 |
| **5.4** | | Главная форма | 34 |
| **5.5** | | **Заказ (14 пунктов)** | |
| | 5.5.1 | Общее требование | 4 |
| | 5.5.2 | Доп. условия (F4) | 10 |
| | 5.5.3 | Особые указания (F5) | 8 |
| | 5.5.4 | Параметры 99 формы | 20 |
| | 5.5.5 | Опорная сторона | 8 |
| | 5.5.6 | Клапан давления | 4 |
| | 5.5.7 | Краевое окрашивание (F6) | 12 |
| | 5.5.8 | Снятие покрытия (F7) | 10 |
| | 5.9 | Логотип (F8) | 28 |
| | 5.5.10 | Профили (F9) | 32 |
| | 5.5.11 | Ввод CNC чертежей | 20 |
| | 5.5.12 | Открыть заказ 1С | 6 |
| | 5.5.13 | Список шпросов | 10 |
| | 5.5.14 | Рассчитать заказ | 16 |
| **ИТОГО** | | | **366** |

---

## Итоговые метрики Frontend

| Параметр | Значение |
|----------|----------|
| **Команда** | 2 Frontend-разработчика |
| **Общая оценка** | **366 часов** |
| **Срок** | **9 недель (~2.5 месяца)** |
| **Стек** | React 18 + TypeScript + MobX + Ant Design 5 + Konva.js |

---

*Техническое задание для ShapeViewer Frontend-разработки*
*Дата: Март 2026*
