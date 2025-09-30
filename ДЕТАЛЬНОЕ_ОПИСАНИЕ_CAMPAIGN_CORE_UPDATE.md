# Детальное описание процесса ведения рекламной кампании (Campaign Core Update)

## 1. Верхнеуровневое описание процесса

### 1.1 Назначение процесса

**Campaign Core Update** - это основной workflow для ведения и обновления рекламных кампаний в системе автоматизации рекламы. Процесс обеспечивает полный цикл обработки данных кампании от загрузки отчета до создания готовых объявлений с системой мониторинга и контроля выполнения.

### 1.2 Общая архитектура процесса

Процесс построен по принципу **оркестрации подпроцессов** с системой мониторинга и контроля выполнения:

```
Основной процесс (Campaign Core Update)
├── Инициализация и мониторинг
├── Подпроцесс 1: ADV_campaign_data_load (Загрузка данных)
├── Подпроцесс 2: ADV_campaign_analysis (Анализ кампании)
├── Подпроцесс 3: ADS headers and texts creation services for updates (Создание объявлений)
└── Завершение и отчетность
```

### 1.3 Используемые структуры данных

#### 1.3.1 Основные таблицы базы данных

**Таблица `business_processes`** - справочник процессов:
- `business_process_id` (bigint, PK) - уникальный идентификатор процесса
- `n8n_process_name` (text) - название процесса в n8n
- `root_process_id` (bigint) - ID корневого процесса (для подпроцессов)

**Таблица `business_processes_states`** - состояния выполнения процессов:
- `business_process_state_id` (bigint, PK) - уникальный идентификатор состояния
- `business_process_id` (bigint, FK) - ссылка на процесс
- `n8n_workflow_execution_id` (bigint) - ID выполнения workflow в n8n
- `n8n_workflow_root_execution_id` (bigint) - ID корневого выполнения
- `started` (timestamp) - время начала выполнения
- `finished` (timestamp) - время завершения
- `error_happened` (timestamp) - время возникновения ошибки
- `error_message` (text) - сообщение об ошибке
- `steps_total` (integer) - общее количество шагов
- `steps_passed` (integer) - количество выполненных шагов
- `tokens_used` (bigint) - количество использованных токенов ИИ
- `tokens_used_type` (text) - тип модели ИИ (gpt-5)

**Таблица `report_import`** - импорты отчетов:
- `import_id` (bigint, PK) - уникальный идентификатор импорта
- `start_date` (date) - начальная дата отчета
- `end_date` (date) - конечная дата отчета
- `import_date` (date) - дата импорта
- `company_id` (bigint) - ID компании
- `processed_date` (date) - дата обработки

**Таблица `workflow_entity`** - справочник workflows:
- `id` (bigint, PK) - уникальный идентификатор workflow
- `name` (text) - название workflow

#### 1.3.2 Временные таблицы для обработки данных

**Таблица `current_list`** - временные данные анализа кампании:
- `list_id` (bigint, PK) - уникальный идентификатор записи
- `key_word` (text) - ключевое слово из отчета
- `campaign_id` (bigint) - ID кампании в Яндекс.Директ
- `used_keyword` (text) - использованное ключевое слово
- `group_id` (bigint) - ID группы объявлений
- `condition_type` (text) - тип условия показа
- `category` (text) - категория объявления
- `ads_header` (text) - заголовок объявления
- `resolution` (text) - решение по фразе
- `count_views` (integer) - количество показов
- `count_clics` (integer) - количество кликов
- `count_convetions` (integer) - количество конверсий
- `ai_response_tokens` (integer) - токены ИИ для анализа
- `company_id` (bigint) - ID компании
- `report_id` (bigint) - ID отчета
- `service_id` (bigint) - ID услуги
- `is_adv_processed` (boolean) - флаг обработки объявления

**Таблица `current_minus_words`** - временные минус-слова:
- `minus_id` (bigint, PK) - уникальный идентификатор
- `minus_word` (text) - минус-слово
- `current_record_id` (bigint) - ссылка на запись из current_list
- `keep_minus` (boolean) - флаг сохранения минус-слова
- `minus_word_v2` (text) - обновленное минус-слово
- `keep_minus_v2` (boolean) - флаг сохранения обновленного минус-слова
- `comment` (text) - комментарий
- `company_id` (bigint) - ID компании

**Таблица `current_ads_list`** - временные объявления:
- Наследует все поля от `ads_list`
- `report_id` (bigint) - ID отчета

### 1.4 Ключевые особенности процесса

1. **Система мониторинга**: Полная интеграция с `business_processes_states` для отслеживания выполнения
2. **Пропуск выполненных процессов**: Механизм проверки уже выполненных подпроцессов
3. **Динамические вызовы**: Вызов под-workflows по ID из справочника
4. **Отслеживание прогресса**: Счетчик выполненных шагов
5. **Учет токенов ИИ**: Мониторинг использования токенов для каждого подпроцесса
6. **Обработка ошибок**: Логирование и отслеживание ошибок выполнения

## 2. Детальное описание каждого подпроцесса

### 2.1 Инициализация и настройка переменных

#### 2.1.1 Узел "When clicking 'Execute workflow'" (Manual Trigger)

**Тип**: `n8n-nodes-base.manualTrigger`
**Назначение**: Точка входа в workflow, запуск по требованию пользователя
**Параметры**: Нет дополнительных параметров
**Выходные данные**: Пустой JSON объект `{}`

#### 2.1.2 Узел "Set variables"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Установка начальных переменных для процесса
**Параметры**:
```javascript
{
  "yandex_report_dropbox_file_path": "https://www.dropbox.com/scl/fi/9kr6iebx4q6cstzp2xuwb/2025-09-04_2025-09-24_searchquery_kiberone-saratov.csv?rlkey=cew8cc5k2rfbq5wrco9inb4eu&dl=1",
  "threshhold": 0.79
}
```

**Детальное описание полей**:
- `yandex_report_dropbox_file_path` (string) - URL файла отчета кампании в Dropbox
  - Содержит CSV файл с данными о показах, кликах, конверсиях
  - Формат: `https://www.dropbox.com/scl/fi/[file_id]/[filename].csv?rlkey=[key]&dl=1`
  - Параметр `dl=1` обеспечивает прямую загрузку файла
- `threshhold` (number) - порог схожести для анализа релевантности (0.79)
  - Используется для определения релевантности ключевых слов
  - Значение 0.79 означает 79% схожести для принятия решения

#### 2.1.3 Узел "Mark_workwlow_started"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Отметка начала выполнения процесса в системе мониторинга
**SQL-запрос**:
```sql
INSERT INTO business_processes_states (business_process_id, n8n_workflow_execution_id, started)
SELECT MAX(business_process_id), $2, NOW()
FROM public.business_processes
WHERE n8n_process_name = $1;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{$workflow.name}}` - название текущего workflow ("campaign core update")
- `$2` = `{{$execution.id}}` - уникальный ID выполнения workflow в n8n
- `MAX(business_process_id)` - получает максимальный ID процесса с указанным именем
- `NOW()` - текущая дата и время начала выполнения
- Создает новую запись в `business_processes_states` для отслеживания выполнения

#### 2.1.4 Узел "Init total counter"

**Тип**: `n8n-nodes-base.code`
**Назначение**: Инициализация глобального счетчика шагов
**JavaScript код**:
```javascript
const workflowData = $getWorkflowStaticData('global');
workflowData.steps_passed = 0;
return items;
```

**Детальное описание кода**:
- `$getWorkflowStaticData('global')` - получает глобальные данные workflow
- `workflowData.steps_passed = 0` - инициализирует счетчик выполненных шагов
- `return items` - передает входящие данные дальше по цепочке

#### 2.1.5 Узел "Mark_workflow_total_steps(manually)"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Установка общего количества шагов процесса
**SQL-запрос**:
```sql
UPDATE business_processes_states 
SET steps_total = $3 
WHERE business_process_id = (
    SELECT MAX(business_process_id) 
    FROM business_processes 
    WHERE n8n_process_name = $1
) 
AND n8n_workflow_execution_id = $2;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{$workflow.name}}` - название workflow
- `$2` = `{{$execution.id}}` - ID выполнения
- `$3` = `3` - общее количество шагов (задается вручную)
- Обновляет запись в `business_processes_states` с общим количеством шагов

### 2.2 Подпроцесс 1: ADV_campaign_data_load (Загрузка данных кампании)

#### 2.2.1 Узел "ADV_campaign_data_load"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Установка названия подпроцесса для загрузки данных
**Параметры**:
```javascript
{
  "workflow_name": "ADV_campaign_data_load"
}
```

#### 2.2.2 Узел "Execute a SQL query" (Проверка выполнения подпроцесса)

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Проверка, был ли уже выполнен подпроцесс загрузки данных
**SQL-запрос**:
```sql
WITH cutoff AS (
  SELECT MAX(st2.started) AS last_finished_started
  FROM public.business_processes_states st2
  JOIN public.business_processes bp2
    ON bp2.business_process_id = st2.business_process_id
  WHERE bp2.n8n_process_name = $1
    AND st2.finished IS NOT NULL
),
main AS (
  SELECT st.*
  FROM public.business_processes_states st
  JOIN public.business_processes bp
    ON bp.business_process_id = st.business_process_id
  CROSS JOIN cutoff c
  WHERE bp.n8n_process_name = $1
    AND st.finished IS NULL
    AND (c.last_finished_started IS NULL OR st.started > c.last_finished_started)
)
SELECT
  s.business_process_state_id,
  s.business_process_id,
  s.started,
  s.finished,
  s.error_message,
  s.n8n_workflow_execution_id,
  s.n8n_workflow_root_execution_id
FROM main m
JOIN public.business_processes_states s
  ON s.n8n_workflow_root_execution_id = m.n8n_workflow_execution_id
JOIN public.business_processes p
  ON p.business_process_id = s.business_process_id
WHERE p.n8n_process_name = $2
ORDER BY s.started desc;
```

**Детальное описание SQL-запроса**:

**CTE `cutoff`**:
- Находит максимальное время начала последнего успешно завершенного выполнения основного процесса
- `st2.finished IS NOT NULL` - только завершенные процессы
- Определяет "точку отсечения" для проверки новых выполнений

**CTE `main`**:
- Находит незавершенные выполнения основного процесса после последнего успешного завершения
- `st.finished IS NULL` - только незавершенные процессы
- `st.started > c.last_finished_started` - только процессы, начатые после последнего успешного завершения

**Основной SELECT**:
- Ищет подпроцессы, связанные с незавершенными выполнениями основного процесса
- `s.n8n_workflow_root_execution_id = m.n8n_workflow_execution_id` - связь по корневому execution_id
- Возвращает информацию о состоянии подпроцесса

#### 2.2.3 Узел "If SUB process already executed"

**Тип**: `n8n-nodes-base.if`
**Назначение**: Проверка, был ли подпроцесс уже выполнен
**Условие**:
```javascript
$json.finished IS NOT NULL
```

**Детальное описание условия**:
- Проверяет поле `finished` в результате SQL-запроса
- Если `finished` не пустое - подпроцесс уже выполнен, переходим к следующему шагу
- Если `finished` пустое - подпроцесс не выполнен, необходимо его запустить

#### 2.2.4 Узел "get company_id from lust succeed report"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Получение ID компании из последнего успешного отчета
**SQL-запрос**:
```sql
SELECT * FROM report_import 
WHERE import_id = (
    SELECT MAX(import_id) 
    FROM report_import
);
```

**Детальное описание SQL-запроса**:
- `MAX(import_id)` - получает максимальный ID импорта (последний импорт)
- Возвращает все поля из `report_import` для последнего импорта
- Содержит `company_id`, `start_date`, `end_date`, `import_date`, `processed_date`

#### 2.2.5 Узел "company_record_id"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Передача данных о компании в подпроцесс
**Параметры**:
```javascript
{
  "company_id": "={{ $('get company_id from lust succeed report').item.json.company_id }}",
  "import_id": "={{ $('get company_id from lust succeed report').item.json.import_id }}"
}
```

**Детальное описание полей**:
- `company_id` - ID компании из последнего отчета
- `import_id` - ID импорта отчета для связи данных

#### 2.2.6 Узел "get workflow id by name"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Получение ID workflow по названию из справочника n8n
**SQL-запрос**:
```sql
SELECT id
FROM workflow_entity
WHERE name = $1
ORDER BY id DESC
LIMIT 1;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{ $('ADV_campaign_data_load').item.json.workflow_name }}` - название workflow
- `ORDER BY id DESC` - сортировка по убыванию ID (последний созданный)
- `LIMIT 1` - возвращает только один результат
- Возвращает `id` workflow для вызова

#### 2.2.7 Узел "set input data for SUB"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Подготовка входных данных для подпроцесса
**Параметры**:
```javascript
{
  "root_execution_id": "={{ $execution.id }}",
  "yandex_report_dropbox_file_path": "={{ $('Set variables').item.json.yandex_report_dropbox_file_path }}",
  "threshhold": "={{ $('Set variables').item.json.threshhold }}"
}
```

**Детальное описание полей**:
- `root_execution_id` - ID корневого выполнения для связи подпроцесса с основным
- `yandex_report_dropbox_file_path` - URL файла отчета из начальных переменных
- `threshhold` - порог схожести из начальных переменных

#### 2.2.8 Узел "Call SUB"

**Тип**: `n8n-nodes-base.executeWorkflow`
**Назначение**: Вызов подпроцесса загрузки данных
**Параметры**:
- `workflowId`: `={{$json.id}}` - ID workflow из справочника
- `workflowInputs`: Передача подготовленных данных

**Детальное описание**:
- Динамически вызывает workflow по ID
- Передает все подготовленные входные данные
- Ожидает завершения выполнения подпроцесса

#### 2.2.9 Узел "Get current step saved"

**Тип**: `n8n-nodes-base.code`
**Назначение**: Обновление счетчика выполненных шагов
**JavaScript код**:
```javascript
const workflowData = $getWorkflowStaticData('global');
workflowData.steps_passed = workflowData.steps_passed + 1;
for (const item of $input.all()) {
  item.json.steps_passed = workflowData.steps_passed;
}
return $input.all();
```

**Детальное описание кода**:
- `workflowData.steps_passed + 1` - инкремент счетчика шагов
- `for (const item of $input.all())` - обновление всех входящих элементов
- `item.json.steps_passed` - добавление поля с текущим шагом в каждый элемент

#### 2.2.10 Узел "Mark_workflow_passed_steps_and_company_id"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Сохранение прогресса выполнения в БД
**SQL-запрос**:
```sql
UPDATE business_processes_states 
SET steps_passed = $3 
WHERE business_process_id = (
    SELECT MAX(business_process_id) 
    FROM business_processes 
    WHERE n8n_process_name = $1
) 
AND n8n_workflow_execution_id = $2;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{$workflow.name}}` - название основного workflow
- `$2` = `{{$execution.id}}` - ID выполнения
- `$3` = `{{$json.steps_passed}}` - количество выполненных шагов
- Обновляет запись в `business_processes_states` с текущим прогрессом

#### 2.2.11 Узел "update workflow tokens usage"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Обновление статистики использования токенов ИИ
**SQL-запрос**:
```sql
UPDATE business_processes_states 
SET tokens_used_type = $3, 
    tokens_used = tokens_used + (
        SELECT bps.tokens_used 
        FROM business_processes bp1,
             business_processes_states bps,
             business_processes_states bps2 
        WHERE bp1.n8n_process_name = $2 
          AND bp1.business_process_id = bps.business_process_id 
          AND bps.n8n_workflow_root_execution_id = bps2.n8n_workflow_execution_id 
          AND bps.finished IS NOT NULL 
          AND bps2.n8n_workflow_execution_id = $1 
        ORDER BY bps2.business_process_state_id DESC 
        LIMIT 1
    ) 
WHERE n8n_workflow_execution_id = $1;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{$execution.id}}` - ID выполнения основного workflow
- `$2` = `{{$('ADV_campaign_data_load').item.json.workflow_name}}` - название подпроцесса
- `$3` = `'gpt-5'` - тип модели ИИ
- Подзапрос находит количество токенов, использованных в подпроцессе
- Суммирует токены с уже накопленными в основном процессе

### 2.3 Подпроцесс 2: ADV_campaign_analysis (Анализ кампании)

#### 2.3.1 Узел "ADV_campaign_analysis"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Установка названия подпроцесса анализа
**Параметры**:
```javascript
{
  "workflow_name": "ADV_campaign_analysis"
}
```

#### 2.3.2 Узел "Execute a SQL query1" (Проверка выполнения анализа)

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Проверка выполнения подпроцесса анализа
**SQL-запрос**: Аналогичен запросу для загрузки данных (см. 2.2.2)

#### 2.3.3 Узел "If SUB process already executed1"

**Тип**: `n8n-nodes-base.if`
**Назначение**: Проверка выполнения подпроцесса анализа
**Условие**: `$json.finished IS NOT NULL`

#### 2.3.4 Узел "get workflow id by name1"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Получение ID workflow анализа
**SQL-запрос**: Аналогичен запросу для загрузки данных (см. 2.2.6)

#### 2.3.5 Узел "set input data for SUB1"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Подготовка данных для анализа
**Параметры**:
```javascript
{
  "root_execution_id": "={{ $execution.id }}",
  "company_id": "={{ $('company_record_id').item.json.company_id }}",
  "threshhold": "={{ $('Set variables').item.json.threshhold }}",
  "import_id": "={{ $('company_record_id').item.json.import_id }}"
}
```

**Детальное описание полей**:
- `root_execution_id` - ID корневого выполнения
- `company_id` - ID компании из предыдущего шага
- `threshhold` - порог схожести для анализа
- `import_id` - ID импорта для связи данных

#### 2.3.6 Узел "Call SUB1"

**Тип**: `n8n-nodes-base.executeWorkflow`
**Назначение**: Вызов подпроцесса анализа кампании
**Параметры**: Аналогичны вызову загрузки данных

#### 2.3.7 Узел "Get current step saved1"

**Тип**: `n8n-nodes-base.code`
**Назначение**: Обновление счетчика шагов после анализа
**JavaScript код**: Аналогичен коду для загрузки данных

#### 2.3.8 Узел "Mark_workflow_passed_steps1"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Сохранение прогресса после анализа
**SQL-запрос**: Аналогичен запросу для загрузки данных

#### 2.3.9 Узел "update workflow tokens usage1"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Обновление статистики токенов после анализа
**SQL-запрос**: Аналогичен запросу для загрузки данных

### 2.4 Подпроцесс 3: ADS headers and texts creation services for updates (Создание объявлений)

#### 2.4.1 Узел "ADS headers and texts creation services for updates"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Установка названия подпроцесса создания объявлений
**Параметры**:
```javascript
{
  "workflow_name": "ADS headers and texts creation services for updates"
}
```

#### 2.4.2 Узел "Execute a SQL query2" (Проверка выполнения создания объявлений)

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Проверка выполнения подпроцесса создания объявлений
**SQL-запрос**: Аналогичен запросу для загрузки данных

#### 2.4.3 Узел "If SUB process already executed2"

**Тип**: `n8n-nodes-base.if`
**Назначение**: Проверка выполнения подпроцесса создания объявлений
**Условие**: `$json.finished IS NOT NULL`

#### 2.4.4 Узел "get workflow id by name2"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Получение ID workflow создания объявлений
**SQL-запрос**: Аналогичен запросу для загрузки данных

#### 2.4.5 Узел "set input data for SUB2"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Подготовка данных для создания объявлений
**Параметры**:
```javascript
{
  "root_execution_id": "={{ $execution.id }}",
  "company_id": "={{ $('company_record_id').item.json.company_id }}",
  "threshhold": "={{ $('Set variables').item.json.threshhold }}",
  "import_id": "={{ $('company_record_id').item.json.import_id }}"
}
```

#### 2.4.6 Узел "Call SUB2"

**Тип**: `n8n-nodes-base.executeWorkflow`
**Назначение**: Вызов подпроцесса создания объявлений
**Параметры**: Аналогичны предыдущим вызовам

#### 2.4.7 Узел "Get current step saved2"

**Тип**: `n8n-nodes-base.code`
**Назначение**: Обновление счетчика шагов после создания объявлений
**JavaScript код**: Аналогичен предыдущим обновлениям

#### 2.4.8 Узел "Mark_workflow_passed_steps"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Сохранение прогресса после создания объявлений
**SQL-запрос**: Аналогичен предыдущим запросам

#### 2.4.9 Узел "update workflow tokens usage2"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Обновление статистики токенов после создания объявлений
**SQL-запрос**: Аналогичен предыдущим запросам

### 2.5 Подпроцесс 4: ADS headers and texts creation services for updates (Повторное создание объявлений)

#### 2.5.1 Узел "ADS headers and texts creation services for updates1"

**Тип**: `n8n-nodes-base.set`
**Назначение**: Установка названия для повторного создания объявлений
**Параметры**:
```javascript
{
  "workflow_name": "ADS headers and texts creation services for updates"
}
```

**Примечание**: Это дублирование подпроцесса создания объявлений для обработки дополнительных данных или повторной генерации.

#### 2.5.2 Остальные узлы подпроцесса 4

Все остальные узлы (Execute a SQL query3, If SUB process already executed3, get workflow id by name3, set input data for SUB3, Call SUB3, Get current step saved3, Mark_workflow_passed_steps2, update workflow tokens usage3) работают аналогично подпроцессу 3, но с суффиксом "3" в названиях.

### 2.6 Завершение процесса

#### 2.6.1 Узел "Mark_workflow_completed"

**Тип**: `n8n-nodes-base.postgres`
**Назначение**: Отметка завершения основного процесса
**SQL-запрос**:
```sql
UPDATE business_processes_states 
SET finished = NOW() 
WHERE business_process_id = (
    SELECT MAX(business_process_id) 
    FROM business_processes 
    WHERE n8n_process_name = $1
) 
AND n8n_workflow_execution_id = $2;
```

**Детальное описание SQL-запроса**:
- `$1` = `{{$workflow.name}}` - название основного workflow
- `$2` = `{{$execution.id}}` - ID выполнения
- `NOW()` - текущая дата и время завершения
- Обновляет запись в `business_processes_states` с временем завершения

## 3. Схема потока выполнения

```
1. Инициализация
   ├── Set variables (установка переменных)
   ├── Mark_workwlow_started (отметка начала)
   ├── Init total counter (инициализация счетчика)
   └── Mark_workflow_total_steps (установка общего количества шагов)

2. Подпроцесс 1: Загрузка данных
   ├── ADV_campaign_data_load (установка названия)
   ├── Execute a SQL query (проверка выполнения)
   ├── If SUB process already executed (условная проверка)
   │   ├── get company_id from lust succeed report (получение ID компании)
   │   ├── company_record_id (передача данных)
   │   ├── get workflow id by name (получение ID workflow)
   │   ├── set input data for SUB (подготовка данных)
   │   └── Call SUB (вызов подпроцесса)
   ├── Get current step saved (обновление счетчика)
   ├── Mark_workflow_passed_steps_and_company_id (сохранение прогресса)
   └── update workflow tokens usage (обновление токенов)

3. Подпроцесс 2: Анализ кампании
   ├── ADV_campaign_analysis (установка названия)
   ├── Execute a SQL query1 (проверка выполнения)
   ├── If SUB process already executed1 (условная проверка)
   │   ├── get workflow id by name1 (получение ID workflow)
   │   ├── set input data for SUB1 (подготовка данных)
   │   └── Call SUB1 (вызов подпроцесса)
   ├── Get current step saved1 (обновление счетчика)
   ├── Mark_workflow_passed_steps1 (сохранение прогресса)
   └── update workflow tokens usage1 (обновление токенов)

4. Подпроцесс 3: Создание объявлений
   ├── ADS headers and texts creation services for updates (установка названия)
   ├── Execute a SQL query2 (проверка выполнения)
   ├── If SUB process already executed2 (условная проверка)
   │   ├── get workflow id by name2 (получение ID workflow)
   │   ├── set input data for SUB2 (подготовка данных)
   │   └── Call SUB2 (вызов подпроцесса)
   ├── Get current step saved2 (обновление счетчика)
   ├── Mark_workflow_passed_steps (сохранение прогресса)
   └── update workflow tokens usage2 (обновление токенов)

5. Подпроцесс 4: Повторное создание объявлений
   ├── ADS headers and texts creation services for updates1 (установка названия)
   ├── Execute a SQL query3 (проверка выполнения)
   ├── If SUB process already executed3 (условная проверка)
   │   ├── get workflow id by name3 (получение ID workflow)
   │   ├── set input data for SUB3 (подготовка данных)
   │   └── Call SUB3 (вызов подпроцесса)
   ├── Get current step saved3 (обновление счетчика)
   ├── Mark_workflow_passed_steps2 (сохранение прогресса)
   └── update workflow tokens usage3 (обновление токенов)

6. Завершение
   └── Mark_workflow_completed (отметка завершения)
```

## 4. Ключевые особенности реализации

### 4.1 Система мониторинга

- **Полная интеграция** с `business_processes_states` для отслеживания выполнения
- **Отслеживание прогресса** через счетчик `steps_passed`
- **Логирование ошибок** в поле `error_message`
- **Учет токенов ИИ** для каждого подпроцесса

### 4.2 Пропуск выполненных процессов

- **Сложная SQL-логика** для определения уже выполненных подпроцессов
- **CTE (Common Table Expressions)** для эффективной обработки данных
- **Условные переходы** на основе результатов проверки

### 4.3 Динамические вызовы

- **Справочник workflows** в таблице `workflow_entity`
- **Получение ID** по названию workflow
- **Передача параметров** в подпроцессы

### 4.4 Обработка данных

- **Временные таблицы** для промежуточных результатов
- **Связывание данных** через `company_id` и `import_id`
- **Передача параметров** между подпроцессами

## 5. Детальное описание подпроцессов

### 5.1 Подпроцесс ADV_campaign_data_load (Загрузка данных кампании)

#### 5.1.1 Общее описание подпроцесса

**Назначение**: Загрузка и обработка данных кампании из CSV файла отчета Яндекс.Директ, размещенного в Dropbox.

**Входные параметры**:
- `yandex_report_dropbox_file_path` (string) - URL файла отчета в Dropbox
- `threshhold` (number) - порог схожести для анализа (0.79)
- `root_execution_id` (number) - ID корневого выполнения для связи

**Выходные данные**:
- Записи в таблице `current_list` с данными кампании
- Запись в таблице `report_import` с информацией об импорте

#### 5.1.2 Детальное описание узлов

**Узел "Download File"**
- **Тип**: `n8n-nodes-base.httpRequest`
- **Назначение**: Загрузка CSV файла отчета из Dropbox
- **Параметры**:
  - `url`: `={{ $json.yandex_report_dropbox_file_path }}` - URL файла из входных данных
  - `responseFormat`: `file` - загрузка как файл
- **Детальное описание**: Выполняет HTTP GET запрос к Dropbox для получения CSV файла с данными отчета кампании. Файл содержит информацию о показах, кликах, конверсиях по каждому поисковому запросу.

**Узел "Extract from File"**
- **Тип**: `n8n-nodes-base.extractFromFile`
- **Назначение**: Извлечение данных из CSV файла
- **Параметры**:
  - `delimiter`: `;` - разделитель полей в CSV (точка с запятой)
- **Детальное описание**: Парсит CSV файл с разделителем ";" и преобразует каждую строку в JSON объект. Стандартные поля отчета Яндекс.Директ:
  - `Поисковый запрос` - ключевое слово, по которому показалось объявление
  - `№ Кампании` - ID кампании в Яндекс.Директ
  - `Условие показа` - использованное ключевое слово в объявлении
  - `№ Группы` - ID группы объявлений
  - `Тип соответствия` - точное/фразовое/широкое соответствие
  - `Категория таргетинга` - категория объявления
  - `Заголовок` - заголовок показанного объявления
  - `Показы` - количество показов
  - `Клики` - количество кликов
  - `Конверсии` - количество конверсий

**Узел "Loop Over Items"**
- **Тип**: `n8n-nodes-base.splitInBatches`
- **Назначение**: Обработка данных по батчам для эффективности
- **Параметры**: Стандартные настройки батчинга
- **Детальное описание**: Разделяет массив записей на части для обработки. Это позволяет обрабатывать большие отчеты без превышения лимитов памяти и времени выполнения.

**Узел "Insert into current_list"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Сохранение данных кампании во временную таблицу
- **Операция**: `insert` в таблицу `current_list`
- **Маппинг полей**:
  ```javascript
  {
    "key_word": "={{ $json['Поисковый запрос'] }}",
    "campaign_id": "={{ $json['№ Кампании'] }}",
    "used_keyword": "={{ $json['Условие показа'] }}",
    "group_id": "={{ $json['№ Группы'] }}",
    "condition_type": "={{ $json['Тип соответствия'] }}",
    "category": "={{ $json['Категория таргетинга'] }}",
    "ads_header": "={{ $json['Заголовок'] }}",
    "count_views": "={{ $json['Показы'] }}",
    "count_clics": "={{ $json['Клики'] }}",
    "count_convetions": "={{ $json['Конверсии'] }}",
    "company_id": "={{ $json.company_id }}",
    "report_id": "={{ $('report import record').item.json.import_id }}",
    "service_id": 0
  }
  ```
- **Детальное описание полей**:
  - `key_word` - поисковый запрос из отчета
  - `campaign_id` - ID кампании для связи с Яндекс.Директ
  - `used_keyword` - ключевое слово, которое сработало в объявлении
  - `group_id` - ID группы объявлений
  - `condition_type` - тип соответствия (точное/фразовое/широкое)
  - `category` - категория таргетинга объявления
  - `ads_header` - заголовок показанного объявления
  - `count_views` - статистика показов
  - `count_clics` - статистика кликов
  - `count_convetions` - статистика конверсий
  - `company_id` - ID компании из входных данных
  - `report_id` - ID отчета для связи данных
  - `service_id` - ID услуги (устанавливается в 0, заполняется позже)

**Узел "report import record"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Создание записи об импорте отчета
- **SQL-запрос**:
  ```sql
  INSERT INTO report_import (start_date, end_date, import_date, company_id, processed_date)
  VALUES ($1, $2, NOW(), $3, NULL)
  RETURNING import_id;
  ```
- **Параметры**:
  - `start_date` - начальная дата отчета (извлекается из названия файла)
  - `end_date` - конечная дата отчета (извлекается из названия файла)
  - `company_id` - ID компании из входных данных
- **Детальное описание**: Создает запись в таблице `report_import` для отслеживания импортированных отчетов. Возвращает `import_id` для связи с данными в `current_list`.

### 5.2 Подпроцесс ADV_campaign_analysis (Анализ кампании)

#### 5.2.1 Общее описание подпроцесса

**Назначение**: Анализ данных кампании с использованием ИИ для определения релевантности ключевых слов, поиска минус-слов и упоминаний конкурентов.

**Входные параметры**:
- `company_id` (number) - ID компании для анализа
- `threshhold` (number) - порог схожести для анализа
- `import_id` (number) - ID импорта отчета
- `root_execution_id` (number) - ID корневого выполнения

**Выходные данные**:
- Обновленные записи в `current_list` с результатами анализа
- Записи в `current_minus_words` с найденными минус-словами
- Обновленная статистика в `key_words`

#### 5.2.2 Детальное описание узлов

**Узел "Loop Over Items"**
- **Тип**: `n8n-nodes-base.splitInBatches`
- **Назначение**: Обработка записей кампании по батчам
- **Детальное описание**: Разделяет записи из `current_list` на части для анализа. Каждая запись содержит данные о показе объявления по конкретному поисковому запросу.

**Узел "Check exact match"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Проверка точного соответствия поискового запроса с ключевыми словами компании
- **JavaScript код**:
  ```javascript
  // Получаем поисковый запрос и условие показа
  const searchQuery = $json.key_word;
  const usedKeyword = $json.used_keyword;
  
  // Проверяем точное соответствие
  const isExactMatch = searchQuery === usedKeyword;
  
  return [{
    json: {
      ...$json,
      is_exact_match: isExactMatch,
      needs_ai_analysis: !isExactMatch
    }
  }];
  ```
- **Детальное описание логики**:
  - Сравнивает `key_word` (поисковый запрос) с `used_keyword` (условие показа)
  - Если совпадают - помечает как точное соответствие
  - Если не совпадают - требует анализа ИИ для определения релевантности

**Узел "Get company keywords"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Получение ключевых слов компании для сравнения
- **SQL-запрос**:
  ```sql
  SELECT key_word, service_id, service_name
  FROM key_words kw
  JOIN company_services cs ON kw.service_id = cs.service_id
  WHERE kw.company_id = $1
    AND kw.is_relevant = true
  ORDER BY kw.service_id;
  ```
- **Параметры**: `$1` = `{{ $json.company_id }}`
- **Детальное описание**: Получает все релевантные ключевые слова компании с привязкой к услугам для анализа соответствия.

**Узел "OpenAI Analysis"**
- **Тип**: `n8n-nodes-base.openAi`
- **Назначение**: Анализ релевантности с использованием ИИ
- **Параметры**:
  - `model`: `gpt-4o` - модель для анализа
  - `messages`: Структурированный промпт для анализа
- **Промпт для анализа**:
  ```javascript
  {
    "role": "user",
    "content": `Проанализируй релевантность поискового запроса "${searchQuery}" для компании "${companyName}" с услугами: ${servicesList}.
    
    Критерии анализа:
    1. Соответствие тематике услуг компании
    2. Коммерческая направленность запроса
    3. Релевантность для целевой аудитории
    
    Верни JSON:
    {
      "is_relevant": boolean,
      "reasoning": "обоснование решения",
      "minus_words": ["список минус-слов если не релевантен"],
      "competitor_mentions": ["упоминания конкурентов если есть"]
    }`
  }
  ```
- **Детальное описание**: ИИ анализирует каждый поисковый запрос на предмет релевантности для компании. Определяет:
  - Релевантность запроса для услуг компании
  - Минус-слова для исключения нерелевантных показов
  - Упоминания конкурентов в запросах

**Узел "Process AI Response"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Обработка ответа ИИ и обновление данных
- **JavaScript код**:
  ```javascript
  const aiResponse = $json.choices[0].message.content;
  const analysis = JSON.parse(aiResponse);
  
  return [{
    json: {
      ...$json,
      resolution: analysis.is_relevant ? "relevant" : "not_relevant",
      ai_reasoning: analysis.reasoning,
      minus_words: analysis.minus_words || [],
      competitor_mentions: analysis.competitor_mentions || [],
      ai_response_tokens: $json.usage.total_tokens
    }
  }];
  ```
- **Детальное описание**: Парсит JSON ответ от ИИ и обновляет запись с результатами анализа:
  - `resolution` - решение о релевантности
  - `ai_reasoning` - обоснование решения ИИ
  - `minus_words` - список минус-слов
  - `competitor_mentions` - упоминания конкурентов
  - `ai_response_tokens` - количество использованных токенов

**Узел "Update current_list"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Обновление записи с результатами анализа
- **SQL-запрос**:
  ```sql
  UPDATE current_list 
  SET resolution = $2,
      ai_response_tokens = $3,
      is_adv_processed = true
  WHERE list_id = $1;
  ```
- **Параметры**:
  - `$1` = `{{ $json.list_id }}` - ID записи
  - `$2` = `{{ $json.resolution }}` - результат анализа
  - `$3` = `{{ $json.ai_response_tokens }}` - токены ИИ

**Узел "Insert minus words"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Сохранение найденных минус-слов
- **SQL-запрос**:
  ```sql
  INSERT INTO current_minus_words (minus_word, current_record_id, company_id, comment)
  VALUES ($1, $2, $3, $4);
  ```
- **Параметры**:
  - `$1` = `{{ $json.minus_word }}` - минус-слово
  - `$2` = `{{ $json.current_record_id }}` - ID записи из current_list
  - `$3` = `{{ $json.company_id }}` - ID компании
  - `$4` = `{{ $json.comment }}` - комментарий к минус-слову

**Узел "Update key_words statistics"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Обновление статистики ключевых слов
- **SQL-запрос**:
  ```sql
  UPDATE key_words 
  SET wordstat_count = wordstat_count + $2,
      ai_response_tokens = ai_response_tokens + $3,
      minus_words_processed = true
  WHERE key_word = $1 AND company_id = $4;
  ```
- **Детальное описание**: Обновляет статистику ключевых слов с учетом новых данных из отчета:
  - Увеличивает счетчик показов
  - Добавляет использованные токены ИИ
  - Отмечает обработку минус-слов

### 5.3 Подпроцесс ADS headers and texts creation services for updates (Создание объявлений)

#### 5.3.1 Общее описание подпроцесса

**Назначение**: Создание рекламных объявлений (заголовков и текстов) на основе проанализированных данных кампании с использованием ИИ.

**Входные параметры**:
- `company_id` (number) - ID компании
- `threshhold` (number) - порог схожести
- `import_id` (number) - ID импорта отчета
- `root_execution_id` (number) - ID корневого выполнения

**Выходные данные**:
- Записи в таблице `ads_list` с созданными объявлениями
- Обновленные записи в `current_list` с привязкой к объявлениям

#### 5.3.2 Детальное описание узлов

**Узел "Loop Over Items"**
- **Тип**: `n8n-nodes-base.splitInBatches`
- **Назначение**: Обработка записей для создания объявлений по батчам
- **Параметры**:
  - `reset`: `={{ $json.is_first }}` - сброс при первом запуске
- **Детальное описание**: Обрабатывает записи из `current_list` с релевантными ключевыми словами для создания объявлений.

**Узел "Get a file" (GitHub)**
- **Тип**: `n8n-nodes-base.github`
- **Назначение**: Получение промпта для создания объявлений из GitHub
- **Параметры**:
  - `resource`: `file` - получение файла
  - `owner`: `Romanychlogin` - владелец репозитория
  - `repository`: `n8n_prompts` - репозиторий с промптами
  - `filePath`: `ads_creation_header.txt` - путь к файлу промпта
- **Детальное описание**: Загружает промпт для создания объявлений из GitHub репозитория. Промпт содержит инструкции для ИИ по созданию эффективных рекламных объявлений.

**Узел "Extract from File1"**
- **Тип**: `n8n-nodes-base.extractFromFile`
- **Назначение**: Извлечение текста промпта из файла
- **Параметры**:
  - `operation`: `text` - извлечение как текст
- **Детальное описание**: Извлекает содержимое файла промпта для использования в запросах к ИИ.

**Узел "Code1"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Сохранение промпта в глобальные данные workflow
- **JavaScript код**:
  ```javascript
  const workflowData = $getWorkflowStaticData('global');
  workflowData.my_prompt = items[0].json;
  return items;
  ```
- **Детальное описание**: Сохраняет промпт в глобальные данные workflow для использования во всех итерациях создания объявлений.

**Узел "Get company services"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Получение услуг компании для контекста создания объявлений
- **SQL-запрос**:
  ```sql
  SELECT service_name, service_description, service_url
  FROM company_services
  WHERE company_id = $1
  ORDER BY service_id;
  ```
- **Параметры**: `$1` = `{{ $json.company_id }}`
- **Детальное описание**: Получает список услуг компании для включения в контекст создания объявлений.

**Узел "Set API JSON"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Подготовка JSON для запроса к OpenAI API
- **JavaScript код**:
  ```javascript
  return [{
    json: {
      "model": "gpt-5",
      "input": [
        {
          "role": "developer",
          "content": [
            {
              "type": "input_text",
              "text": "ты специалист по контекстной рекламе. Твоя задача точно выполнять инструкции"
            }
          ]
        },
        {
          "role": "user",
          "content": [
            {
              "type": "input_text",
              "text": $json.my_prompt
            }
          ]
        }
      ],
      "text": {
        "format": {
          "type": "json_schema",
          "name": "header",
          "strict": true,
          "schema": {
            "type": "object",
            "properties": {
              "header": {
                "type": "string",
                "description": "A string with no more than 56 characters.",
                "minLength": 0,
                "maxLength": 56
              }
            },
            "required": ["header"],
            "additionalProperties": false
          }
        },
        "verbosity": "low"
      },
      "reasoning": {
        "effort": "minimal",
        "summary": null
      },
      "tools": [],
      "store": false,
      "include": [
        "reasoning.encrypted_content",
        "web_search_call.action.sources"
      ]
    }
  }];
  ```
- **Детальное описание**: Формирует структурированный запрос к OpenAI API с:
  - Моделью `gpt-5` для создания объявлений
  - Ролью разработчика с инструкциями
  - Пользовательским промптом из GitHub
  - JSON Schema для валидации ответа
  - Ограничением длины заголовка до 56 символов

**Узел "OpenAI"**
- **Тип**: `n8n-nodes-base.openAi`
- **Назначение**: Создание заголовка объявления с использованием ИИ
- **Параметры**: Использует JSON из предыдущего узла
- **Детальное описание**: Отправляет запрос к OpenAI API для создания заголовка объявления на основе:
  - Ключевого слова из отчета
  - Услуг компании
  - Промпта из GitHub
  - Ограничений Яндекс.Директ

**Узел "Process header response"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Обработка ответа ИИ и извлечение заголовка
- **JavaScript код**:
  ```javascript
  const response = $json.choices[0].message.content;
  const headerData = JSON.parse(response);
  
  return [{
    json: {
      ...$json,
      ad_header: headerData.header,
      ai_tokens_used: $json.usage.total_tokens
    }
  }];
  ```
- **Детальное описание**: Парсит JSON ответ от ИИ и извлекает созданный заголовок объявления.

**Узел "Create ad text"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Создание текста объявления
- **JavaScript код**: Аналогичен созданию заголовка, но с промптом для текста
- **Детальное описание**: Создает текст объявления с учетом:
  - Созданного заголовка
  - Ключевого слова
  - Услуг компании
  - Ограничений длины текста

**Узел "Generate URL"**
- **Тип**: `n8n-nodes-base.code`
- **Назначение**: Генерация URL с UTM-метками
- **JavaScript код**:
  ```javascript
  const baseUrl = $json.service_url || $json.company_url;
  const utmParams = `?utm_source=yandex&utm_medium=cpc&utm_campaign=${$json.campaign_id}&utm_term=${encodeURIComponent($json.key_word)}`;
  
  return [{
    json: {
      ...$json,
      ad_url: baseUrl + utmParams
    }
  }];
  ```
- **Детальное описание**: Создает URL с UTM-метками для отслеживания:
  - `utm_source=yandex` - источник трафика
  - `utm_medium=cpc` - тип рекламы
  - `utm_campaign` - ID кампании
  - `utm_term` - ключевое слово

**Узел "Insert into ads_list"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Сохранение созданного объявления в базу данных
- **SQL-запрос**:
  ```sql
  INSERT INTO ads_list (group_name, key_word, ad_header, ad_text, group_id, ad_url, company_id, service_id, report_id)
  VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
  RETURNING ad_id;
  ```
- **Параметры**:
  - `group_name` - название группы объявлений
  - `key_word` - ключевое слово
  - `ad_header` - созданный заголовок
  - `ad_text` - созданный текст
  - `group_id` - ID группы из отчета
  - `ad_url` - URL с UTM-метками
  - `company_id` - ID компании
  - `service_id` - ID услуги
  - `report_id` - ID отчета

**Узел "Update current_list"**
- **Тип**: `n8n-nodes-base.postgres`
- **Назначение**: Связывание записи отчета с созданным объявлением
- **SQL-запрос**:
  ```sql
  UPDATE current_list 
  SET service_id = $2, is_adv_processed = true
  WHERE list_id = $1;
  ```
- **Детальное описание**: Обновляет запись в `current_list` с привязкой к созданному объявлению и услуге.

## 6. Заключение

Процесс **Campaign Core Update** представляет собой сложную систему оркестрации подпроцессов с полным мониторингом и контролем выполнения. Он обеспечивает:

1. **Автоматическую загрузку** данных кампании из Dropbox
2. **Анализ релевантности** ключевых слов с использованием ИИ
3. **Создание объявлений** на основе проанализированных данных
4. **Мониторинг выполнения** каждого этапа процесса
5. **Учет ресурсов** (токены ИИ) для каждого подпроцесса
6. **Пропуск уже выполненных** подпроцессов для оптимизации

### 6.1 Ключевые особенности подпроцессов

**ADV_campaign_data_load**:
- Загрузка CSV файлов с разделителем ";"
- Маппинг полей отчета на структуру БД
- Создание записей импорта для отслеживания

**ADV_campaign_analysis**:
- Проверка точного соответствия ключевых слов
- ИИ-анализ релевантности несоответствующих запросов
- Автоматическое определение минус-слов
- Поиск упоминаний конкурентов

**ADS headers and texts creation services for updates**:
- Получение промптов из GitHub репозитория
- Создание заголовков и текстов с помощью gpt-5
- Генерация URL с UTM-метками
- Связывание объявлений с услугами компании

### 6.2 Техническая архитектура

Система построена с учетом:
- **Надежности** - полное логирование и мониторинг
- **Масштабируемости** - батчевая обработка больших объемов
- **Эффективности** - пропуск уже выполненных процессов
- **Отслеживания** - детальная статистика по токенам ИИ
- **Интеграции** - связь с внешними сервисами (Dropbox, GitHub, OpenAI)

Система обеспечивает полный цикл обработки рекламных кампаний от загрузки данных до создания готовых объявлений с возможностью мониторинга и контроля на каждом этапе.
