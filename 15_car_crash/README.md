## Предсказание риска водителя оказаться виновником ДТП
### Постановка задачи
На основании датасета о ДТП в отдельном регионе нужно разработать модель, которая будет предсказывать риск водителя оказаться виновником ДТП. Заказчик исследования - каршеринговая компания, модель планируется использовать в бизнесе.

### Исходные данные
Исходные данные представляют собой SQL базу данных из трех таблиц. Данные содержат ошибки, пропуски, однако датасет большой, в главной таблице 1_400_000 записей, соответствующих различным ДТП. Поля для связи таблиц есть, данные можно успешно объединить. В целом датасет пригоден для дальнейшей работы. 

### Статистический анализ факторов ДТП
Был проведен статистический анализ факторов ДТП. Была проверена зависимость количества ДТП от времени года (явная - отсутствует), выявлены наиболее типичные обстоятельства самых серьезных ДТП. Кроме того, были поставлены дополнительные задачи для дальнейшего исследования.

Также была выявлена неполнота исходных данных: данные за 2012 год условно полные только за 5 месяцев, начиная с июня данные неполные или отсутствуют.

### Формирование датасета для обучения моделей
На основании исходной базы данных был сформирован датасет для обучения моделей. Для этого были оставлены:
* все данные начиная с 2012 года (как более свежие)
* все происшествия, в которых стороной ДТП является водитель машины
* все происшествия с значимыми последствиями (кроме типа 'scratch')

### Предобработка данных
В ходе предобработки данных были удалены лишние колонки `case_id`, `id`, `party_number` (служебные колонки), `county_city_location`, `county_location` (поскольку данные из одного региона, а задача не выявить зависимость риска от локации внутри региона, а экстраполировать данные региона на любой другой регион), `direction` (направление движения предположительно имеет значение только внутри заданного региона). 

Колонки `collision_date` и `collision_time` были удалены. Данные из колонки `collision_time` были перенесены в колонку `time` (категориальные значения: утро, день, ночь, вечер) на этапе выгрузки данных. 

В датасете были для удобства переименованы некоторые колонки, изменен тип данных для некоторых колонок, заполнены пропуски, устранены выбросы (в колонке `vehicle_age`). В результате в категориальных колонках осталось 95 уникальных значений.

В готовом к обучению моделей датасете 195_898 строк и 23 колонки.

### Исследование корреляций в исходном датасете
Корреляции целевого признака с обучающими в исходном датасете слабые. Наиболее сильные две обратные корреляции (рассчитаны по Спирмену); виновность водителя коррелирует с:
* `party_count` - количеством сторон-участников ДТП (К = -0.28)
* `insurance_premium` - суммой страховки (К = -0.13)

### Обучение моделей
На датасете обучено три вида моделей: линейная регрессия, CatBoost и LightGBM. Гиперпараметры для CatBoost и LightGBM подобраны на RandomizedSearchCV, для линейной регрессии - на GridSearchCV. 

В качестве ключевой метрики выбрана F1 score - среднее гармоническое полноты и точности.

### Выбор лучшей модели
Лучшая из получившихся моделей - catboost_model_4, обученная с настройками, подобранными на RandomizedSearchCV. Это разработка компании Яндекс.

Метрики модели на валидационных данных:

* F1 : 0.7431756663761068
* AUC_ROC : 0.7105323707217235
* Best threshold: 0.4

При запуске модели на тестовых данных метрика получилась примерно такая же:

* F1 : 0.7388942595405791
* AUC_ROC : 0.7071206946239839
* Best threshold: 0.4

Другие модели на тесте дают похожий результат.

### Анализ основных факторов на основании анализа весов модели

Выделение моделью важнейших факторов представляет особый интерес при условии слабой корреляции целевого признака с обучающими, которую мы выявили на этапе оценки датасета.

Поскольку модель CatBoost обучалась на датасете, кодированном методом OHE, то в качестве важных она выделила отдельные варианты перечисленных ниже факторов.

Модель определила как наиболее важные следующие факторы: `primary_collision_factor`, `party_count`, `pcf_violation_category`, `party_sobriety`. Важными также показались `motor_vehicle_incolved_with` и `insurance_premium`. Все это факторы, описывающие поведение водителя, а также взаимодействие водителя с другими участниками ДТП.

Неважными для модели оказались конструктивные особенности автомобиля и его состояние, погода, освещенность и состояние дороги, а также тип местности, где произошло ДТП.

#### Сопоставление с результатами модели LGB
Модель LGB определила как наиболее важные следующие факторы: 'insurance_premium', 'pcf_violation_category', 'distance' (расстояние от главное дороги). Менее важными показались факторы: 'party_count', 'motor_vehicle_involved_with', 'driver_condition'.

Неважными для модели оказались конструктивные особенности автомобиля и его состояние, погода, освещенность и состояние дороги, время суток, время года, а также тип местности, где произошло ДТП.

### Предложения по дальнейшей доработке модели
Имеет смысл обучить модель на всем датасете, не только на данных за 2012 год, и посмотреть результат.

Обучить модель, предсказывающую риск того, что водитель станет виновником ДТП, на этих данных можно, но стоит дообучить модель, чтобы повысить полноту предсказаний.