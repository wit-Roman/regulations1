# Техническое задание на Frontend-разработку (React/MobX/AntD)
## ModernGlass ShapeViewer - Миграция с Delphi на React

---

## 5.2 Специальные задачи

### 5.2.1 Загрузка контролек при входе в заказ

#### Цель
Обеспечить отображение списка контрольных сообщений (ошибок, предупреждений, информационных сообщений) при загрузке заказа, а также предоставить возможность замены повторяющихся стекол на МГ-аналоги.

#### Расположение
- **Модуль:** `OutputUnits\CheckListUnit.pas` (429 строк)
- **Основной класс:** `TCheckListFrame` (наследник `TFrame`)
- **Главная форма:** `MainUnit.pas` — `TMainForm`
- **DataModule:** `DataModules\u_DrawingDM.pas` — `TDrawingDM`
- **Транзакции:** `DataModules\u_TransactionDM.pas` — `TTransactionDM`

**Таблицы БД:**
- `mg_order.check_list` — список сообщений контролек
- `mg_order.check_list_tr` — переводы текстов контролек
- `mg_order.msg_row` — справочник сообщений
- `mg_order.ITEM_HASH` — хэши для проверки изменений
- `liorder.auf_pos` — позиции заказа (стекла)
- `liorder.auf_komp` — компоненты позиций
- `mg_order.mg_glass` — таблица замен МГ-стекол
- `liorder.master_glass` — справочник стекол
- `liorder.auf_text` — тексты позиций

#### Предусловия
1. Заказ загружен через `LoadButtonClick()` в главной форме
2. Номер заказа введен в `OrderEdit` и прошел валидацию
3. Заказ не занят другим пользователем (проверка через `CheckOrderIsBusy()`)
4. Успешно выполнены:
   - `LoadOrder()` — загрузка данных заказа
   - `SetDeaultOrderParam()` — сохранение параметров по умолчанию
   - `LoadCheckList()` — загрузка списка контролек и текста проекта
5. Получен список контролек через `DrawingDM.GetCheckList(Order_no)`
6. Получен текст проекта через `DrawingDM.GetProjText(ProjNr)` и `DrawingDM.GetCustomerReqText(CustNo)`

#### Роли
| Роль | Описание |
|------|----------|
| **Пользователь** | Оператор, загружающий заказ для просмотра/редактирования |
| **TMainForm** | Главная форма, управляющая загрузкой заказа и отображением контролек |
| **TCheckListFrame** | Фрейм отображения списка контролек и текста проекта |
| **TDrawingDM** | DataModule для получения данных из БД (контрольки, текст проекта, стекла) |
| **TTransactionDM** | DataModule для сохранения изменений (замена стекол) |

#### Логика работы

##### Этап 1: Загрузка данных (MainUnit.LoadCheckList)
1. **Инициализация загрузки:**
   - Сохранение предыдущего списка: `lastchklist := chklist`
   - Вызов `DrawingDM.GetCheckList(Order_no)` для получения новых контролек

2. **Проверка ZEI-файлов для 99 фигур:**
   - Чтение параметра: `DrawingDM.getShpwParamValue(17)`
   - Если параметр = 1, вызывается `Check99ShapeZeiFilesExists(EText)`
   - При отсутствии файлов добавляется ошибка: `msg_type = 0`, текст: "На диске S отсутствуют файлы, необходимые для Prod"

3. **Загрузка текста проекта:**
   - Вызов `DrawingDM.GetProjText(ProjNr)` — основной текст проекта
   - Вызов `DrawingDM.GetCustomerReqText(CustNo)` — требования заказчика
   - Объединение текстов: если оба не пустые, разделяются строкой `"-----"`

4. **Проверка результата:**
   - Возврат `true`, если обе операции успешны
   - При ошибке — добавление сообщения в `EText`

##### Этап 2: Отображение контролек (MainUnit.UploadCheckList)
**Инициализация UI:**
   - Вызов `CheckListFrame.InitializeCheckListFrame`
   - Активация вкладки: `PageControl1.ActivePageIndex := 0`
   - Блокировка меню: `SetMenuControlsStatus(false)`, `SetbuttonStatus(false)`

##### Этап 3: Инициализация UI (CheckListUnit.InitializeCheckListFrame)
1. **Заполнение сетки сообщений:**
   - Для каждого сообщения из `fCheckList`:
     - Добавление строки в `CheckGrid`
     - Установка текста: `CheckGrid.CellByName['Message','Last'].AsString := fCheckList[i].msg_text`
     - Стилизация по типу сообщения:
       | msg_type | Цвет текста | Шрифт | Описание |
       |----------|-------------|-------|----------|
       | 0 | `clRed` | `fsBold` | Ошибка |
       | 1 | `clGreen` | `fsBold` | Предупреждение |
       | 2 | `clBlack` | обычный | Информация |
       | 3 | `clGreen` | `fsBold` | Уведомление |

2. **Отображение текста проекта:**
   - Загрузка в `ProjectTextRichEdit.Text := fProjtext`
   - Расчет высоты панели: `PTextHeight = Lines.Count * 16 + HeaderHeight`
   - Ограничение высоты: `228 <= PTextHeight <= 340`

3. **Управление видимостью панелей:**
   - Если `chklist` пустой → `FChecklistPanel.Visible = false`
   - Если `Projtext` пустой → `FProjectPanel.Visible = false`
   - Если оба пустые → скрыть обе панели и сплиттер

4. **Кнопка "Замена стекол":**
   - Вызов `CheckRepeatGlassMSGExists(chklist)`
   - Проверка наличия сообщения с `msg_id = 521` ("Повтор стекла")
   - Если найдено → `ChangeGlassPanel.Visible = true`

##### Этап 4: Замена стекол (CheckListUnit.ChangeGlassButtonClick)
1. **Загрузка данных о стеклах:**
   - SQL-запрос к `liorder.auf_pos` и `liorder.auf_komp`
   - Получение всех стекол заказа с разбивкой по:
     - Позициям (`auf_pos`)
     - Панелям (`pane` 1-5)
     - Компонентам (`komp_lage_nr`)
   - Определение базового стекла: `base_glas` (через `mg_order.mg_glass.base_glass_id`)

2. **Формирование списка замен:**
   - Для каждого уникального `base_glas`:
     - Загрузка МГ-аналогов: `mg_order.mg_glass` где `base_glass_id = base_glas`
     - Сортировка по `glass_seq`
     - Создание записи `TChangeGlass`:
       ```pascal
       TChangeGlass = record
         Gl_code: integer;           // Код стекла
         Gl_NexNum: integer;         // Счетчик замен
         base_set: boolean;          // Флаг установки базового
         MG_GL_List: array of integer; // Список МГ-аналогов
       end;
       ```

3. **Распределение замен:**
   - Для каждого компонента:
     - Вызов `GetNewCode(base_glas, mg_glass)`
     - Если `mg_glass = false` и `base_set = false` → возврат оригинального кода
     - Иначе → возврат следующего МГ-аналога из списка
     - Инкремент счетчика: `Gl_NexNum := Gl_NexNum + 1`

4. **Сохранение изменений:**
   - Для каждого компонента с новым стеклом:
     - **Обновление позиции** (`komp_lage_nr = 0`):
       ```sql
       UPDATE liorder.auf_pos 
       SET glas{pane_no} = :glass_new
       WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo
       ```
     - **Обновление компонента** (`komp_lage_nr > 0`):
       ```sql
       UPDATE liorder.auf_komp 
       SET komp_art_num = :glass_new,
           comp_desc = (SELECT glass_desc FROM master_glass WHERE glass_id = :glass_new)
       WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo 
         AND glas_nr = :pane-1 AND komp_lage_nr = :comp-1
       ```
     - **Обновление текста позиции:**
       ```sql
       UPDATE liorder.auf_text 
       SET zl_str = (SELECT glass_desc FROM master_glass WHERE glass_id = :glass_new)
       WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo 
         AND zl_mod = 1 AND view_id = :pane-1
       ```

5. **По сабмиту:**
   - Запись: `DBCommit`
   - Перепроверка заказа: `MainForm.ReCheckOrder(true)`
   - Закрытие фрейма при отсутствии ошибок

**Функциональные требования:**
- Загрузка контролек: `GET /api/orders/{orderNo}/checklist`
- Загрузка текста проекта: `GET /api/orders/{orderNo}/project-text`
- Загрузка МГ-замен: `GET /api/glass/{glCode}/mg-replacements`
- Применение замен: `POST /api/orders/{orderNo}/glass-changes`
- Проверка на критические ошибки (msg_type = 0)
- Проверка на "Повтор стекла": `msg_id = 521`

**Состав работ:**
1. API репозиторий для загрузки контролек и текста проекта
2. API репозиторий для загрузки МГ-замен стекол
3. API репозиторий для применения замен стекол
4. MobX store: загрузка данных, вычисление ошибок/предупреждений, управление заменами
5. UI компонент: фрейм с 2 панелями (список сообщений + текст проекта)
6. UI компонент: модальное окно замены стекол с превью изменений
7. Интеграция с главной формой (автооткрытие при загрузке заказа)

**Оценка: 10 часов**

---

### 5.2.2 Замена стекол на МГ

#### Цель
Автоматизировать замену повторяющихся стекол в заказе на МГ-аналоги (многослойные стекла) для оптимизации производства и снижения количества уникальных стекол в заказе.

#### Расположение
- **Модуль замены:** `OutputUnits\CheckListUnit.pas` — `TCheckListFrame.ChangeGlassButtonClick()`
- **Модуль ввода стекла:** `InputUnits\FormChangeGlass\GlasInputUnit.pas` — `TGlasInputForm`
- **Модуль ввода слоя:** `InputUnits\FormChangeGlass\LayerInputUnit.pas` — `TLayerInputForm`
- **Фрейм кодов замены:** `InputUnits\u_ChangeGlassCodesFrame.pas` — `TChangeGlassCodesFrame`
- **Главная форма:** `MainUnit.pas` — `TMainForm`

**Таблицы БД:**
- `liorder.auf_pos` — позиции заказа (основные стекла)
- `liorder.auf_komp` — компоненты позиций (дополнительные стекла)
- `mg_order.mg_glass` — таблица замен МГ-стекол
- `liorder.master_glass` — справочник стекол
- `liorder.KOMP_AUFBAU` — структура компонентов стекла

#### Предусловия
1. Заказ загружен и отображается в главной форме
2. В чек-листе присутствует сообщение с `msg_id = 521` ("Повтор стекла")
3. Кнопка "Замена стекол" (`ChangeGlassButton`) видима и активна
4. Пользователь инициировал замену нажатием кнопки

#### Роли
| Роль | Описание |
|------|----------|
| **Пользователь** | Оператор, выполняющий замену стекол |
| **TCheckListFrame** | Фрейм контролек, инициирующий замену |
| **TGlasInputForm** | Модальное окно выбора стекла для замены |
| **TLayerInputForm** | Модальное окно выбора слоя для замены |
| **TChangeGlassCodesFrame** | Фрейм отображения кодов замены |
| **TTransactionDM** | DataModule для сохранения изменений в БД |
| **TDrawingDM** | DataModule для загрузки данных о стеклах и МГ-заменах |

#### Логика работы

##### Этап 1: Инициация замены (ChangeGlassButtonClick)
1. **Подготовка структур данных:**
   - Создание списка ошибок: `ErrText := TStringList.Create`
   - Инициализация списка позиций: `OrderItems: TOrderItems`
   - Инициализация списка стекол: `GlassList: TChangeGlassList`

2. **Загрузка данных о стеклах заказа:**
   - SQL-запрос к `liorder.auf_pos` и `liorder.auf_komp`:
     ```sql
     SELECT auf_pos, pane, coalesce(komp_lage_nr, 0) komp_lage_nr,
            coalesce(comp_product, 0) comp_product, Glas,
            coalesce((SELECT mgl.base_glass_id FROM mg_order.mg_glass mgl 
                      WHERE mgl.glass_id = Glas), 0) base_glas
     FROM (
       SELECT t.auf_pos, t.pane, ak.komp_lage_nr+1 komp_lage_nr,
              mg_order.get_comp_product(t.auf_nr, t.auf_pos, t.pane, 
                                        ak.komp_lage_nr+1, 0) comp_product,
              CASE WHEN komp_art_num IS NULL THEN t.glas ELSE komp_art_num END Glas
       FROM (
         SELECT auf_nr, auf_pos, variante, pane, glas FROM (
           SELECT auf_nr, auf_pos, variante, pane,
                  CASE WHEN pane=1 THEN glas1 WHEN pane=2 THEN glas2
                       WHEN pane=3 THEN glas3 WHEN pane=4 THEN glas4
                       WHEN pane=5 THEN glas5 ELSE 0 END glas
           FROM (
             SELECT ap.auf_nr, ap.auf_pos, variante,
                    glas1, glas2, glas3, ap.ap_glass4 glas4, 
                    ap.ap_glass5 glas5, pane
             FROM generate_series(1,5) Pane, liorder.AUF_POS ap
             WHERE ap.auf_nr = :Order_no AND ap.variante = 0
           ) t
         ) t WHERE glas <> 0
       ) t LEFT JOIN liorder.AUF_KOMP ak 
         ON t.auf_nr = ak.auf_nr AND t.auf_pos = ak.auf_pos 
         AND t.pane-1 = ak.glas_nr AND ak.komp_art_typ = 0
         AND t.variante = ak.variante
     ) t ORDER BY auf_pos, pane, comp_product
     ```
   - Параметр: `Order_no` — номер заказа

3. **Формирование массива позиций:**
   - Группировка по `auf_pos` (позиция заказа)
   - Для каждой позиции:
     - Сохранение `pane_no` (панель 1-5)
     - Сохранение `comp_no` (номер компонента, 0 для основного стекла)
     - Сохранение `Glas` (код стекла)
     - Вычисление `base_glas` (базовое стекло через `mg_glass.base_glass_id`)
     - Флаг `mg_glass = true`, если стекло уже является МГ

##### Этап 2: Формирование списка замен
1. **Уникализация базовых стекол:**
   - Для каждой позиции и компонента:
     - Проверка уникальности: `checkGlassExists(base_glas)`
     - Если стекло новое → добавление в `GlassList`

2. **Загрузка МГ-аналогов:**
   - Для каждого уникального `base_glas`:
     ```sql
     SELECT mgl.glass_id
     FROM mg_order.mg_glass mgl
     WHERE mgl.base_glass_id = :glass
     ORDER BY mgl.glass_seq
     ```
   - Заполнение массива `MG_GL_List`

3. **Структура записи замены:**
   ```pascal
   TChangeGlass = record
     Gl_code: integer;           // Код базового стекла
     Gl_NexNum: integer;         // Счетчик замен (начиная с 1)
     base_set: boolean;          // Флаг: установлено ли базовое стекло
     MG_GL_List: array of integer; // Массив кодов МГ-аналогов
   end;
   ```

##### Этап 3: Распределение замен
1. **Алгоритм GetNewCode:**
   - Вход: `gl_code` (базовое стекло), `mg_glass` (флаг МГ)
   - Если `mg_glass = false` и `base_set = false`:
     - Возврат оригинального кода: `result := gl_code`
     - Установка флага: `base_set := true`
   - Иначе:
     - Проверка доступности аналога: `Gl_NexNum-1 <= High(MG_GL_List)`
     - Возврат кода из списка: `result := MG_GL_List[Gl_NexNum-1]`
     - Инкремент счетчика: `Gl_NexNum := Gl_NexNum + 1`

2. **Применение замен к компонентам:**
   - Для каждой позиции и компонента:
     - Вызов: `glass_new := GetNewCode(base_glas, mg_glass)`
     - Если `glass_new > 0` → сохранение в `glass_new`
     - Иначе → добавление ошибки в `ErrText`

3. **Проверка ошибок:**
   - Если `ErrText.Text <> ''`:
     - Вывод сообщения: `MessageDlg('Ошибка! Замена не возможна.' + ErrText.Text)`
     - Прерывание операции

##### Этап 4: Сохранение изменений
1. **Обновление основных стекол (komp_lage_nr = 0):**
   - Для панелей 1-3:
     ```sql
     UPDATE liorder.auf_pos 
     SET glas{pane_no} = :glass_new
     WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo AND variante = 0
     ```
   - Для панелей 4-5:
     ```sql
     UPDATE liorder.auf_pos 
     SET ap_glass{pane_no} = :glass_new
     WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo AND variante = 0
     ```

2. **Обновление текста позиции:**
   ```sql
   UPDATE liorder.auf_text 
   SET zl_str = (SELECT mg.glass_desc FROM liorder.master_glass mg 
                 WHERE mg.glass_id = :glass_new)
   WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo 
     AND variante = 0 AND zl_mod = 1 AND view_id = :pane-1
   ```

3. **Обновление компонентов (komp_lage_nr > 0):**
   ```sql
   UPDATE liorder.auf_komp 
   SET komp_art_num = :glass_new,
       comp_desc = (SELECT mg.glass_desc FROM liorder.master_glass mg 
                    WHERE mg.glass_id = :glass_new)
   WHERE auf_nr = :OrderNo AND auf_pos = :ItemNo AND variante = 0
     AND komp_art_typ = 0 AND glas_nr = :pane-1 
     AND komp_lage_nr = :comp-1 AND komp_art_num = :glass_old
   ```

4. **Транзакция:**
   - Начало: автоматически при создании `TTransactionDM`
   - Коммит: `DBCommit` при успехе
   - Откат: `DBRollback` при ошибке

5. **Финализация:**
   - Перепроверка заказа: `MainForm.ReCheckOrder(true)`
   - Закрытие фрейма контролек
   - Вывод сообщения об успехе

**Требования к UI:**
- Модальное окно со списком уникальных стекол для замены
- Select для выбора замены из МГ-списка
- Превью изменений (какие позиции будут изменены)
- Индикация текущей замены (какое стекло заменяется на какое)

**Функциональные требования:**
- Загрузка позиций: `GET /api/orders/{orderNo}/items`
- Загрузка МГ-замен: `GET /api/glass/{glCode}/mg-replacements`
- Логика замены: алгоритм `GetNewCode(gl_code, mg_glass)`
- Применение замен: `POST /api/orders/{orderNo}/glass-changes`
- Валидация перед сохранением (проверка доступности аналогов)

**Состав работ:**
1. API сервис для загрузки данных о стеклах заказа
2. API сервис для загрузки МГ-замен
3. API сервис для применения замен стекол
4. MobX store: управление списком замен, вычисление новых кодов
5. UI компонент: модальное окно с 2 колонками (оригиналы/замены)
6. UI компонент: превью изменений по позициям
7. Валидация перед сохранением

**Оценка: 16 часов**

---

### 5.2.3 Постобработка при входе/получении заказа

#### Цель
Выполнить автоматический расчет параметров заказа после загрузки (геометрия, ступени, снятие покрытия, логотипы, опорные стороны) и сохранить результаты расчета в базу данных для последующего отображения чертежей.

#### Расположение
- **Модуль расчета:** `DataModules\u_DrawingDM.pas` — `TDrawingDM.CalcAndSaveDataForOrder()`
- **Поток расчета:** `DataModules\u_DrawingDM.pas` — `TCalcOrderThread`
- **Поток сохранения:** `DataModules\u_DrawingDM.pas` — `TSaveCalcOrderThread`
- **Прогресс-бар:** `ProgressUnit.pas` — `TProgressForm`
- **Главная форма:** `MainUnit.pas` — `TMainForm.RecalcDataForOrder()`

**Таблицы БД:**
- `mg_order.calc_dimensions` — расчетные размеры
- `mg_order.calc_edges` — значения по сторонам
- `mg_order.calc_params` — параметры расчета
- `mg_order.ITEM_HASH` — хэши для проверки изменений
- `mg_order.sup_side` — опорные стороны
- `mg_order.decoating` — снятие покрытия
- `mg_order.term_logo` — логотипы закалки
- `mg_order.lam_logo` — логотипы ламината

#### Предусловия
1. Заказ загружен через `LoadButtonClick()`
2. Номер заказа введен в `OrderEdit` и прошел валидацию
3. Данные заказа сохранены через `SetDeaultOrderParam()`
4. Чек-лист загружен и проверен (нет критических ошибок)
5. Пользователь инициировал расчет (автоматически или через `RecalcOrderClick()`)

#### Роли
| Роль | Описание |
|------|----------|
| **Пользователь** | Оператор, инициирующий расчет заказа |
| **TMainForm** | Главная форма, управляющая процессом расчета |
| **TDrawingDM** | DataModule для получения данных и управления расчетом |
| **TCalcOrderThread** | Поток выполнения расчета параметров |
| **TSaveCalcOrderThread** | Поток сохранения результатов расчета |
| **TProgressForm** | Модальное окно с прогресс-баром и логом |

#### Логика работы

##### Этап 1: Инициация расчета (MainUnit.RecalcDataForOrder)
1. **Подготовка к расчету:**
   - Проверка загруженного заказа: `CheckOrderLoaded`
   - Блокировка интерфейса: `Screen.Cursor := crSQLWait`
   - Подключение к БД: `DrawingDM.DBConnect`
   - Установка флага: `DrawingDM.CalcOrderData := true`
   - Открытие query: `DrawingDM.OpenCommonDataQuery(OrderNo, 0)`

2. **Запуск расчета:**
   ```pascal
   LogText := '';
   DrawingDM.CalcAndSaveDataForOrder(OrderNo, RecalcAll, LogText);
   ```
   - Параметр `RecalcAll`:
     - `true` — полный пересчет всех позиций
     - `false` — пересчет только измененных позиций

3. **Обработка результатов:**
   - Если `LogText <> ''` → вывод сообщений об ошибках: `MessageCalcErrModal(LogText)`
   - Сброс флага: `DrawingDM.CalcOrderData := false`
   - Закрытие query: `DrawingDM.CloseCommonDataQuery`
   - Отключение от БД: `DrawingDM.DBDisconnect`

##### Этап 2: Подготовка данных (DrawingDM.CalcAndSaveDataForOrder)
1. **Загрузка списка для расчета:**
   ```pascal
   CalcItemList := GetCalcItemList(Order_no, RecalcAll);
   ```
   - Возвращает массив `TCalcItemList` — позиции для расчета
   - При `RecalcAll = false` — только позиции с измененными хэшами

2. **Создание прогресс-бара:**
   ```pascal
   ProgressForm := TProgressForm.Create(nil);
   ProgressForm.Caption := 'Расчет данных..';
   ProgressForm.ProgressBar1.Max := length(CalcItemList) + 3;
   ProgressForm.ProgressBar1.Position := 0;
   ```
   - Если позиций > 7 → показ окна: `ProgressForm.Show`
   - Если `RecalcAll = false` → режим `fsStayOnTop`

3. **Инициализация лога:**
   ```pascal
   setlength(CalcOrderLog, 0);
   ```

##### Этап 3: Расчет параметров (TCalcOrderThread.Execute)
1. **Создание потока:**
   ```pascal
   CalcOrderThread := TCalcOrderThread.Create(
     @CalcItemList,      // Список позиций
     @CalcOrderLog,      // Лог ошибок
     ProgressForm.ProgressBar1,  // Прогресс-бар
     DrawingSettings     // Настройки отображения
   );
   ```

2. **Запуск потока:**
   ```pascal
   CalcOrderThread.Resume;
   CalcOrderThread.Waitfor;
   ```

3. **Методы расчета (TCalcOrderThread):**
   - `CalculateSupSideData(i)` — расчет опорных сторон
   - `CalculateTurnSideData(i)` — расчет поворота сторон
   - `CalculateRightAngle(i)` — проверка прямых углов
   - `CalculateLineLength(i)` — расчет длин линий
   - `CalculateItemStepData(i)` — расчет ступеней
   - `CalculateDecoatData(i)` — расчет снятия покрытия
   - `SetLogoMirrorTextForTermProc(i)` — текст зеркала логотипа
   - `SetMirrorForGlassByTermLogo(i)` — зеркало стекла по логотипу

4. **Обновление прогресс-бара:**
   ```pascal
   procedure TCalcOrderThread.ProgressBarNext;
   begin
     fProgressBar.Position := fProgressBar.Position + 1;
   end;
   ```

5. **Регистрация ошибок:**
   ```pascal
   procedure SetItemError(i: integer; LineText: string);
   begin
     // Добавление записи в CalcOrderLog
     // Line_items: массив [order_no, item_no, pane_no, comp_no]
     // Line_text: текст ошибки
   end;
   ```

##### Этап 4: Сохранение результатов (TSaveCalcOrderThread.Execute)
1. **Смена статуса:**
   ```pascal
   ProgressForm.Caption := 'Сохранение..';
   ProgressForm.Repaint;
   ```

2. **Создание потока сохранения:**
   ```pascal
   SaveCalcOrderThread := TSaveCalcOrderThread.Create(
     @CalcItemList,    // Список позиций
     Order_no,         // Номер заказа
     RecalcAll,        // Флаг полного пересчета
     ProgressForm.ProgressBar1
   );
   ```

3. **Запуск и ожидание:**
   ```pascal
   SaveCalcOrderThread.Resume;
   SaveCalcOrderThread.Waitfor;
   ```

4. **Проверка результата:**
   ```pascal
   if not SaveCalcOrderThread.DataSaved then
     raise Exception.Create('Ошибка сохранения данных');
   ```

5. **Метод сохранения (TSaveCalcOrderThread):**
   - Для каждой позиции из `CalcItemList`:
     - Сохранение в `mg_order.calc_dimensions`
     - Сохранение в `mg_order.calc_edges`
     - Сохранение в `mg_order.calc_params`
     - Обновление хэша в `mg_order.ITEM_HASH`
   - Транзакционное выполнение через `TTransactionDM`

##### Этап 5: Формирование отчета
1. **Сбор лога:**
   ```pascal
   for i := low(CalcOrderLog) to high(CalcOrderLog) do
   begin
     if LogText = '' then
       LogText := '№: ' + DrawingDm.FromArrToStr(CalcOrderLog[i].Line_items) + 
                  '. ' + CalcOrderLog[i].Line_text
     else
       LogText := LogText + #13#10 + '№: ' + 
                  DrawingDm.FromArrToStr(CalcOrderLog[i].Line_items) + 
                  '. ' + CalcOrderLog[i].Line_text;
   end;
   ```

2. **Формат записи:**
   - `№: [order_no, item_no, pane_no, comp_no]. Текст ошибки`

3. **Обновление прогресс-бара:**
   ```pascal
   ProgressForm.Repaint;
   ```

4. **Очистка:**
   ```pascal
   FreeAndNil(ProgressForm);
   ```

##### Этап 6: Закрытие прогресс-бара (TProgressForm.FormCloseQuery)
```pascal
procedure TProgressForm.FormCloseQuery(Sender: TObject; var CanClose: Boolean);
begin
  CanClose := (ProgressBar1.Position = 0);
end;
```
- Запрет закрытия во время выполнения расчета

**Требования к UI:**
- Модальное окно с прогресс-баром (0-100%)
- Отображение текущего шага расчета
- Лог выполнения (список сообщений об ошибках)
- Кнопка "Отменить" (при возможности прерывания)
- Автоматическое закрытие после завершения

**Функциональные требования:**
- Запуск расчета: `POST /api/orders/{orderNo}/calculate`
- Получение статуса: `GET /api/jobs/{jobId}/status` (polling)
- Отмена расчета: `POST /api/jobs/{jobId}/cancel`
- Получение лога: `GET /api/jobs/{jobId}/log`

**Состав работ:**
1. API сервис для запуска расчета заказа
2. API сервис для получения статуса расчета (polling)
3. API сервис для отмены расчета
4. API сервис для получения лога расчета
5. MobX store: управление статусом расчета, прогресс, лог ошибок
6. UI компонент: модальное окно с прогресс-баром
7. UI компонент: список сообщений лога
8. Интеграция с главной формой (автозапуск при загрузке заказа)

**Оценка: 12 часов**
---

## 5.5 Заказ

### 5.5.1 Общее требование для всех инструментов

**Автор:** Курочкин Л.М.

#### Общее описание
Обязательное действие при закрытии любой формы пункта меню «Заказ» — вызов метода выполнения контролек и отображение результатов.

#### Цель
Обеспечить автоматическую проверку и отображение контрольных сообщений (контролек) после закрытия любой формы редактирования параметров заказа для своевременного информирования пользователя об ошибках, предупреждениях и изменениях в заказе.

#### Расположение
- **Основной модуль:** `MainUnit.pas` — `TMainForm.ReCheckOrder()`
- **Модуль контролек:** `OutputUnits\CheckListUnit.pas` — `TCheckListFrame`
- **DataModule:** `DataModules\u_DrawingDM.pas` — `TDrawingDM.GetCheckList()`
- **Пункты главного меню:** `Заказ-*` (Дополнительные условия, Особые указания, Краевое окрашивание, Снятие покрытия, Логотип, Профили, Опорная сторона, Клапан давления, Параметры 99 формы, CNC чертежи, Рассчитать заказ)

**Таблицы БД:**
- `mg_order.check_list` — список сообщений контролек
- `mg_order.check_list_tr` — переводы текстов контролек
- `mg_order.msg_row` — справочник сообщений
- `mg_order.ITEM_HASH` — хэши для проверки изменений
- `liorder.auf_pos` — позиции заказа (для проверки изменений)
- `liorder.auf_komp` — компоненты позиций

#### Предусловия
1. Была открыта одна из форм пункта меню «Заказ-*»:
   - Дополнительные условия (F4)
   - Особые указания (F5)
   - Краевое окрашивание (F6)
   - Снятие покрытия (F7)
   - Логотип (F8)
   - Профили (F9)
   - Опорная сторона
   - Клапан давления
   - Параметры 99 формы
   - CNC чертежи
   - Рассчитать заказ

2. Пользователь закрыл форму (сохранением или отменой изменений)
3. Заказ загружен и отображается в главной форме
4. Номер заказа введен в `OrderEdit` и прошел валидацию

#### Роли
| Роль | Права и действия |
|------|------------------|
| **Оператор** | Закрытие форм ввода параметров, просмотр контролек, реакция на предупреждения |

*Примечание: Детализация ролей будет уточнена позже. В текущей версии системы разграничение прав по ролям (Технолог, Мастер смены) не реализовано — все пользователи имеют равный доступ к функциям.*

#### Логика работы

##### Этап 1: Закрытие формы ввода параметров
1. **Инициация закрытия:**
   - Пользователь закрывает форму (нажатием OK/Cancel или закрытием окна)
   - Форма отправляет событие закрытия в главную форму

2. **Вызов метода проверки:**
   ```pascal
   procedure TMainForm.ReCheckOrder(ShowAll: boolean): boolean;
   ```
   - Параметр `ShowAll`:
     - `true` — отображать все контрольки независимо от изменений
     - `false` — отображать только при наличии изменений

##### Этап 2: Проверка необходимости отображения контролек
1. **Блокировка обновления интерфейса:**
   ```pascal
   LockWindowUpdate(Handle);
   ```
   - Предотвращение мерцания экрана во время проверки

2. **Подключение к БД:**
   ```pascal
   DrawingDM.DBConnect;
   ```

3. **Загрузка нового списка контролек:**
   ```pascal
   if not LoadCheckList(OrderEdit.AsInteger, Projtext, EText) then
   begin
     DrawingDM.DBDisconnect;
     exit;
   end;
   ```
   - Вызов `DrawingDM.GetCheckList(Order_no)` для получения актуального списка
   - Сравнение с предыдущим списком (`lastchklist`)

4. **Сравнение списков контролек:**
   ```pascal
   CheckListChanged := CompareCheckLists(lastchklist, chklist);
   ```
   - Проверка наличия новых сообщений
   - Проверка удаления сообщений
   - Проверка изменения типа сообщения

##### Этап 3: Принятие решения об отображении
1. **Проверка условий отображения:**
   ```pascal
   if (length(chklist) > 0) and 
      ((CheckListChanged) or (CheckListErrorMode) or (ShowAll)) then
     UploadCheckList(Projtext)
   ```

2. **Условия отображения контролек:**
   | Условие | Описание | Результат |
   |---------|----------|-----------|
   | **Красные контрольки** | Присутствуют сообщения с `msg_type = 0` (ошибки) | Отображать |
   | **Изменён список** | Список контролек изменился после закрытия формы | Отображать |
   | **Параметр ShowAll** | Передан флаг «отображать всегда все» | Отображать |
   | **Нет контролек и нет HP-стекол** | Список пуст и нет стекол типа HP | Не отображать |

3. **Проверка на HP-стекла:**
   ```pascal
   function HasHPGlass(OrderNo: integer): boolean;
   ```
   - Проверка наличия стекол с специальным типом (HP — Heat Protected / Закаленные)
   - Если HP-стекла есть → требуется отображение контролек даже при пустом списке

##### Этап 4: Отображение контролек
1. **Вызов метода отображения:**
   ```pascal
   procedure UploadCheckList(Projtext: string);
   ```

2. **Создание вкладки с чек-листом:**
   ```pascal
   aSheet := TNxTabSheet.Create(PageControl1);
   aSheet.PageControl := PageControl1;
   aSheet.Caption := 'Чек лист';
   
   CheckListFrame := TCheckListFrame.Create(
     PageControl1.Pages[PageControl1.PageCount-1],
     chklist,
     Projtext
   );
   ```

3. **Инициализация UI:**
   ```pascal
   CheckListFrame.InitializeCheckListFrame;
   PageControl1.ActivePageIndex := 0;
   SetMenuControlsStatus(false);
   SetbuttonStatus(false);
   ```

4. **Активация вкладки:**
   - Переключение на вкладку «Чек лист»
   - Блокировка кнопок главного меню
   - Блокировка элементов управления

##### Этап 5: Альтернативный сценарий (нет контролек)
1. **Проверка необходимости показа других вкладок:**
   ```pascal
   if not DecoatingFrameForSiliconSPWasShown then
     CheckLoadDecoatingFrameForSiliconSP;
   ```
   - Проверка необходимости автоматической загрузки вкладки «Снятие покрытия для силиконовых СП»

2. **Активация элементов управления:**
   ```pascal
   SetMenuControlsStatus(true);
   SetbuttonStatus(true);
   ```

##### Этап 6: Завершение проверки
1. **Разблокировка интерфейса:**
   ```pascal
   LockWindowUpdate(0);
   ```

2. **Отключение от БД:**
   ```pascal
   DrawingDM.DBDisconnect;
   ```

3. **Возврат результата:**
   ```pascal
   result := true;  // Успешно
   ```
   или
   ```pascal
   result := false;  // Ошибка или нет необходимости отображения
   ```

##### Этап 7: Обработка исключений
```pascal
except
  result := false;
  SetbuttonStatus(false);
  SetMenuControlsStatus(false);
  MessageErr('Ошибка получения данных!');
end;
```

#### Замечания
*Будут уточнены позже*

**Требования к UI:**
- Автоматическое отображение вкладки «Чек лист» при наличии изменений
- Блокировка элементов управления при отображении контролек
- Индикация критических ошибок (красный цвет)
- Поддержка параметра «отображать всегда все»

**Функциональные требования:**
- Загрузка контролек: `GET /api/orders/{orderNo}/checklist`
- Сравнение списков: клиентское или серверное
- Проверка HP-стекол: `GET /api/orders/{orderNo}/items/glass-types`
- Отображение контролек: автоматическое переключение на вкладку

**Состав работ:**
1. API сервис для загрузки актуального списка контролек
2. API сервис для проверки типов стекол в заказе
3. MobX store: управление состоянием контролек, флаг изменений
4. HOC (Higher-Order Component) для обработки закрытия форм ввода параметров
5. Метод проверки необходимости отображения контролек
6. Интеграция со всеми формами пункта меню «Заказ»
7. Логирование вызовов метода проверки

**Оценка: 8 часов**

---
