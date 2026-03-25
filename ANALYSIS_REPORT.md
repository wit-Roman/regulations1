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
1. **Создание вкладки:**
   ```pascal
   aSheet := TNxTabSheet.Create(PageControl1);
   aSheet.Caption := 'Чек лист';
   ```

2. **Создание фрейма:**
   ```pascal
   CheckListFrame := TCheckListFrame.Create(
     PageControl1.Pages[PageControl1.PageCount-1], 
     chklist, 
     Projtext
   );
   ```

3. **Инициализация UI:**
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

5. **Финализация:**
   - Коммит транзакции: `DBCommit`
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
1. API сервис для загрузки контролек и текста проекта
2. API сервис для загрузки МГ-замен стекол
3. API сервис для применения замен стекол
4. MobX store: загрузка данных, вычисление ошибок/предупреждений, управление заменами
5. UI компонент: фрейм с 2 панелями (список сообщений + текст проекта)
6. UI компонент: модальное окно замены стекол с превью изменений
7. Интеграция с главной формой (автооткрытие при загрузке заказа)
