# Bestsellers Report

## Содержание
* [Как работает Bestsellers Report](#Как-работает-bestsellers-report)
  * [Сбор данных для отчета](#Сбор-данных-для-отчета)
  * [Выгрузка отчета](#Выгрузка-отчета)
* [Возможное решение проблемы](#Возможное-решение-проблемы)


## Как работает Bestsellers Report
Этот отчет выводит самые продаваемые товары, сгруппированные по определенному периоду (день, месяц, год) за заданный диапазон дат.

### Сбор данных для отчета

В `Mage_Sales_Model_Observer` по крону переодически вызывается метод 
`aggregateSalesReportBestsellersData($schedule)`.

В этом методе создается объект класса `Mage_Sales_Model_Resource_Report_Bestsellers` и вызывается метод `aggregate($date)`.

Для отчета строятся таблицы `sales_bestsellers_aggregated_daily`, `sales_bestsellers_aggregated_monthly` и `sales_bestsellers_aggregated_yearly`.

Структура таблиц схожа:

`id`, `period`, `store_id`, `product_id`, `product_type_id`, `product_name`, `product_price`, `qty_ordered`, `rating_pos`

В этом методе `aggregate` собираются данные таблицы `sales_bestsellers_aggregated_daily`.

Собираются все товары из не отмененных заказов, группируются по `period`, `store_id` и `product_id`.

За счет группировки считается общее количество заказанных единиц товара (`qty_ordered`) за определенный период для конкретного стора.

Вот как это выглядит (троеточием я сократил незначительные для понимания общей картины части):
```sql
INSERT INTO `sales_bestsellers_aggregated_daily` 
            (
                `period`, 
                `store_id`, 
                `product_id`, 
                `product_type_id`, 
                `product_name`, 
                `product_price`, 
                `qty_ordered` 
            ) 
SELECT     STRAIGHT_JOIN Date(Date_add(`source_table`.`created_at`, INTERVAL -28800 second)) AS `period`,
           `source_table`.`store_id`, 
           `order_item`.`product_id`, 
           `product`.`type_id`                                                        AS `product_type_id`,
           Min(Ifnull(product_name.value, product_default_name.value))                AS `product_name`,
           Min(Ifnull((Ifnull(product_price.value, product_default_price.value)), 0)) AS `product_price`,
           Sum(order_item.qty_ordered)                                                AS `qty_ordered`
FROM       `sales_flat_order` AS `source_table`
INNER JOIN `sales_flat_order_item` AS `order_item` ...
INNER JOIN `catalog_product_entity` AS `product` ...
LEFT JOIN  `catalog_product_entity_varchar` AS `product_name` ...          -- получаем данные атрибута name,  где store_id равно store_id заказа
LEFT JOIN  `catalog_product_entity_varchar` AS `product_default_name` ...  -- получаем данные атрибута name,  где store_id равен 0
LEFT JOIN  `catalog_product_entity_decimal` AS `product_price` ...         -- получаем данные атрибута price, где store_id равно store_id заказа
LEFT JOIN  `catalog_product_entity_decimal` AS `product_default_price` ... -- получаем данные атрибута price, где store_id равен 0
WHERE      (source_table.state != 'canceled')                              -- данные собираем из всех заказов, кроме отмененных
GROUP BY   Date(Date_add(`source_table`.`created_at`, INTERVAL -28800 second)), 
           `source_table`.`store_id`,
           `order_item`.`product_id` 
HAVING (1=0) on duplicate KEY UPDATE ...                                   -- обновляются все поля, если есть дубли
```

Далее метод `aggregate` вызывает метод `_updateRatingPos` три раза для каждого вида агрегации.
```php
$this->_updateRatingPos(self::AGGREGATION_DAILY);   // self::AGGREGATION_DAILY   = 'daily'
$this->_updateRatingPos(self::AGGREGATION_MONTHLY); // self::AGGREGATION_MONTHLY = 'monthly'
$this->_updateRatingPos(self::AGGREGATION_YEARLY);  // self::AGGREGATION_YEARLY  = 'yearly'
```

Метод `_updateRatingPos` передает аргументы в метод `getBestsellersReportUpdateRatingPos` объекта класса `Mage_Sales_Model_Resource_Helper_Mysql4`.

```php
protected function _updateRatingPos($aggregation)
{
    $aggregationTable   = $this->getTable('sales/bestsellers_aggregated_' . $aggregation);

    $aggregationAliases = array(
        'daily'   => self::AGGREGATION_DAILY,
        'monthly' => self::AGGREGATION_MONTHLY,
        'yearly'  => self::AGGREGATION_YEARLY
    );
    Mage::getResourceHelper('sales')
        ->getBestsellersReportUpdateRatingPos($aggregation, $aggregationAliases,
            $this->getMainTable(), $aggregationTable);

    return $this;
}
```

А метод `getBestsellersReportUpdateRatingPos` в свою очередь передает их в метод `updateReportRatingPos` объекта класса `Mage_Reports_Model_Resource_Helper_Mysql4`.

```php
public function getBestsellersReportUpdateRatingPos($aggregation, $aggregationAliases,
    $mainTable, $aggregationTable
) {
    /** @var $reportsResourceHelper Mage_Reports_Model_Resource_Helper_Interface */
    $reportsResourceHelper = Mage::getResourceHelper('reports');

    if ($aggregation == $aggregationAliases['monthly']) {
        $reportsResourceHelper
            ->updateReportRatingPos('month', 'qty_ordered', $mainTable, $aggregationTable);
    } elseif ($aggregation == $aggregationAliases['yearly']) {
        $reportsResourceHelper
            ->updateReportRatingPos('year', 'qty_ordered', $mainTable, $aggregationTable);
    } else {
        $reportsResourceHelper
            ->updateReportRatingPos('day', 'qty_ordered', $mainTable, $aggregationTable);
    }

    return $this;
}
```
Тут ничего существенного не происходило, просто собрались данные для метода `updateReportRatingPos`, хоть и криво.

```php
public function updateReportRatingPos($type, $column, $mainTable, $aggregationTable)
```

Поминим, что вызова было три, соответственно `updateReportRatingPos` у `Mage_Reports_Model_Resource_Helper_Mysql4` вызовется три раза для каждого вида агрегации.

То есть, в итоге, аргументы придут такие:
```php
updateReportRatingPos('day',   'qty_ordered', 'sales_bestsellers_aggregated_daily', 'sales_bestsellers_aggregated_daily')
updateReportRatingPos('month', 'qty_ordered', 'sales_bestsellers_aggregated_daily', 'sales_bestsellers_aggregated_monthly')
updateReportRatingPos('year',  'qty_ordered', 'sales_bestsellers_aggregated_daily', 'sales_bestsellers_aggregated_yearly')
```
По сути, из таблицы `sales_bestsellers_aggregated_daily` неким образом собираются `sales_bestsellers_aggregated_monthly` и `sales_bestsellers_aggregated_yearly`.

В этом методе по каждому стору считается свой рейтинг товаров. Каждому товару присваивается свое место в рейтинге (`rating_pos`).

Наболее продаваемый товар (с наибольшим `qty_ordered`) получает `rating_pos = 1` для определенного стора (`store_id`), и так по убыванию. 

Вот как это выглядит.

Запрос для `sales_bestsellers_aggregated_daily`
```sql
INSERT INTO `sales_bestsellers_aggregated_daily` 
            ( 
                `period`, 
                `store_id`, 
                `product_id`, 
                `product_name`, 
                `product_price`, 
                `id`, 
                `product_type_id`, 
                `qty_ordered`, 
                `rating_pos` 
            ) 
SELECT `t`.`period`, 
       `t`.`store_id`, 
       `t`.`product_id`, 
       `t`.`product_name`, 
       `t`.`product_price`, 
       `t`.`id`, 
       `t`.`product_type_id`, 
       `t`.`qty_ordered`, 
       `t`.`rating_pos` 
FROM   ( 
          SELECT `t`.`period`, 
                 `t`.`store_id`, 
                 `t`.`product_id`, 
                 `t`.`product_name`, 
                 `t`.`product_price`, 
                 `t`.`id`, 
                 `t`.`product_type_id`, 
                 `t`.`total_qty` AS `qty_ordered`, 
                 (@pos := IF(t.`store_id` <> @prevstoreid 
          OR     t.period <> @prevperiod, 1, @pos+1)) AS `rating_pos`, 
                 (@prevstoreid := t.`store_id`)       AS `prevstoreid`, 
                 (@prevperiod := t.period)            AS `prevperiod` 
          FROM   ( 
                          SELECT   `t`.`period`, 
                                   `t`.`store_id`, 
                                   `t`.`product_id`, 
                                   `t`.`product_name`, 
                                   `t`.`product_price`, 
                                   `t`.`id`, 
                                   `t`.`product_type_id`, 
                                   Sum(t.qty_ordered)                   AS `total_qty` 
                          FROM     `sales_bestsellers_aggregated_daily` AS `t` 
                          GROUP BY `t`.`store_id`, 
                                   `t`.`period`, 
                                   `t`.`product_id` 
                          ORDER BY `t`.`store_id` ASC, 
                                   `t`.`period` ASC, 
                                   IF(t.product_type_id IN ('grouped', 
                                                            'configurable', 
                                                            'bundle'), 1, 0), 
                                   `total_qty` DESC) AS `t`) AS `t` 
on duplicate KEY UPDATE ...
```
Запрос для `sales_bestsellers_aggregated_monthly`
```sql
INSERT INTO `sales_bestsellers_aggregated_monthly` 
            ( 
                `period`, 
                `store_id`, 
                `product_id`, 
                `product_name`, 
                `product_price`, 
                `product_type_id`, 
                `qty_ordered`, 
                `rating_pos` 
            ) 
SELECT Date_format(t.period, '%Y-%m-01') AS `period`, 
       `t`.`store_id`, 
       `t`.`product_id`, 
       `t`.`product_name`, 
       `t`.`product_price`, 
       `t`.`product_type_id`, 
       `t`.`qty_ordered`, 
       `t`.`rating_pos` 
FROM   ( 
          SELECT `t`.`period`, 
                 `t`.`store_id`, 
                 `t`.`product_id`, 
                 `t`.`product_name`, 
                 `t`.`product_price`, 
                 `t`.`product_type_id`, 
                 `t`.`total_qty` AS `qty_ordered`, 
                 (@pos := IF(t.`store_id` <> @prevstoreid 
          OR     Date_format(t.period, '%Y-%m-01') <> @prevperiod, 1, @pos+1)) AS `rating_pos`, 
                 (@prevstoreid := t.`store_id`)                                AS `prevstoreid`,
                 (@prevperiod := Date_format(t.period, '%Y-%m-01'))            AS `prevperiod` 
          FROM   ( 
                          SELECT   `t`.`period`, 
                                   `t`.`store_id`, 
                                   `t`.`product_id`, 
                                   `t`.`product_name`, 
                                   `t`.`product_price`, 
                                   `t`.`product_type_id`, 
                                   Sum(t.qty_ordered)                   AS `total_qty` 
                          FROM     `sales_bestsellers_aggregated_daily` AS `t` 
                          GROUP BY `t`.`store_id`, 
                                   Date_format(t.period, '%Y-%m-01'), 
                                   `t`.`product_id` 
                          ORDER BY `t`.`store_id` ASC, 
                                   Date_format(t.period, '%Y-%m-01'), 
                                   IF(t.product_type_id IN ('grouped', 
                                                            'configurable', 
                                                            'bundle'), 1, 0), 
                                   `total_qty` DESC) AS `t`) AS `t` 
on duplicate KEY UPDATE ...
```
Запрос для `sales_bestsellers_aggregated_yearly`
```sql
INSERT INTO `sales_bestsellers_aggregated_yearly` 
            ( 
                `period`, 
                `store_id`, 
                `product_id`, 
                `product_name`, 
                `product_price`, 
                `product_type_id`, 
                `qty_ordered`, 
                `rating_pos` 
            ) 
SELECT Date_format(t.period, '%Y-01-01') AS `period`, 
       `t`.`store_id`, 
       `t`.`product_id`, 
       `t`.`product_name`, 
       `t`.`product_price`, 
       `t`.`product_type_id`, 
       `t`.`qty_ordered`, 
       `t`.`rating_pos` 
FROM   ( 
          SELECT `t`.`period`, 
                 `t`.`store_id`, 
                 `t`.`product_id`, 
                 `t`.`product_name`, 
                 `t`.`product_price`, 
                 `t`.`product_type_id`, 
                 `t`.`total_qty` AS `qty_ordered`, 
                 (@pos := IF(t.`store_id` <> @prevstoreid 
          OR     Date_format(t.period, '%Y-01-01') <> @prevperiod, 1, @pos+1)) AS `rating_pos`, 
                 (@prevstoreid := t.`store_id`)                                AS `prevstoreid`,
                 (@prevperiod := Date_format(t.period, '%Y-01-01'))            AS `prevperiod` 
          FROM   ( 
                          SELECT   `t`.`period`, 
                                   `t`.`store_id`, 
                                   `t`.`product_id`, 
                                   `t`.`product_name`, 
                                   `t`.`product_price`, 
                                   `t`.`product_type_id`, 
                                   Sum(t.qty_ordered)                   AS `total_qty` 
                          FROM     `sales_bestsellers_aggregated_daily` AS `t` 
                          GROUP BY `t`.`store_id`, 
                                   Date_format(t.period, '%Y-01-01'), 
                                   `t`.`product_id` 
                          ORDER BY `t`.`store_id` ASC, 
                                   Date_format(t.period, '%Y-01-01'), 
                                   IF(t.product_type_id IN ('grouped', 
                                                            'configurable', 
                                                            'bundle'), 1, 0), 
                                   `total_qty` DESC) AS `t`) AS `t` 
on duplicate KEY UPDATE ...
```

Очень грубый пример того, что могло собраться:

| Период       | Товар  | store_id | qty_ordered | rating_pos |
| ------------ | :----: | :------: | :---------: | :--------: |
| Январь 2017  | Чашка  | 1        | 10          | 1          |
| Январь 2017  | Кружка | 1        | 7           | 2          |
| Январь 2017  | Лопата | 1        | 3           | 3          |
| Январь 2017  | Кружка | 2        | 17          | 1          |
| Январь 2017  | Чашка  | 2        | 9           | 2          |
| Январь 2017  | Лопата | 2        | 5           | 3          |
| Январь 2017  | Грабли | 2        | 1           | 4          |
| Февраль 2017 | Грабли | 1        | 3           | 1          |
| Февраль 2017 | Лопата | 1        | 1           | 2          |
| Февраль 2017 | Кружка | 2        | 15          | 2          |
| Февраль 2017 | Лопата | 2        | 2           | 2          |


Только хранится все это не в таком упорядоченном виде.

[Вернуться к содержанию...](#Содержание)

### Выгрузка отчета

Посмотрим теперь, как выгружается отчет, для этого смотрим его коллекцию - `Mage_Sales_Model_Resource_Report_Bestsellers_Collection`.

Также в админке откроем отчет и выберем период `месяц`, даты `01.01.2016` - `30.11.2017`, показывать отчет `по всем сторам`.

Выведем запрос в `_afterLoad`.
```sql
 SELECT Date_format(period, '%Y-%m') AS `period`,
       Sum(qty_ordered)             AS `qty_ordered`,
       `sales_bestsellers_aggregated_monthly`.`product_id`,
       Max(product_name)            AS `product_name`,
       Max(product_price)           AS `product_price`,
       `sales_bestsellers_aggregated_monthly`.`product_type_id`
FROM   `sales_bestsellers_aggregated_monthly`
WHERE  ( rating_pos <= 5 )
       AND ( store_id IN( 10, 11, 5, 12,
                          9, 8, 7, 6 ) )
       AND ( period >= '2016-01-01' )
       AND ( period <= '2017-11-30' )
       AND ( product_type_id NOT IN ( 'grouped', 'configurable', 'bundle' ) )
GROUP  BY `period`,
          `product_id`
ORDER  BY `period` ASC,
          `qty_ordered` DESC
```
Что тут происходит:
1. Собираются данные из таблицы `sales_bestsellers_aggregated_monthly`
2. Группируются по периоду и id товара
3. Сортируются сначала по периоду по возрастанию
4. Сортируются по количеству заказаных товаров по убыванию
5. Суммируется количество заказанных единиц товара
6. Фильтруется по указанному периоду дат
7. Фильтруется по указанным сторам
8. Отсекаются товары с типами `grouped`, `configurable`, `bundle` (потому что они составные)
9. Отсекаются все товары с rating_pos меньше 5

Получается такое, к примеру:
(`store_id` добавлен специально, чтобы можно было кое-что понять, хотя в запросе он не селектится)

| Период       | Товар  | store_id | qty_ordered | rating_pos |
| ------------ | :----: | :------: | :---------: | :--------: |
| Январь 2017  | Кружка | 2        | 17          | 1          |
| Январь 2017  | Чашка  | 1        | 10          | 1          |
| Январь 2017  | Чашка  | 2        | 9           | 2          |
| Январь 2017  | Кружка | 1        | 7           | 2          |
| Январь 2017  | Лопата | 2        | 5           | 3          |
| Январь 2017  | Лопата | 2        | 3           | 3          |
| Январь 2017  | Грабли | 2        | 1           | 4          |
| Февраль 2017 | Кружка | 1        | 15          | 2          |
| Февраль 2017 | Грабли | 1        | 3           | 1          |
| Февраль 2017 | Лопата | 2        | 2           | 2          |
| Февраль 2017 | Лопата | 1        | 1           | 2          |

Важный момент. Там стоит фильтр `rating_pos <= 5`. Причем эта `5` захардкожена в коллекции.

Создается ощущение, что разработчики мадженты хотели выбрать максимум 5 товаров для каждой группы, ограничив `rating_pos`.

Почему тогда в отчете иногда видно большее количество?

Вспомним тот факт, что для каждого стора у нас свой ретинг, поэтому когда мы выбрали несколько сторов (в нашем случае вообще все), у нас в одной группе могут встречаться товары с одинаковым `rating_pos`.

Именно поэтому мы видим больше товаров за период, нежели 5. Посмотрите таблицу выше, чтобы это понять.

Тем не менее, это не единственный косяк.

Тогда получается, что если делать выгрузку только для одного стора, все будет работать правильно? Нет. 

Потому что даже в одном сторе почему-то встречаются товары с одинаковой позицией в рейтинге.

Кроме того, есть еще одна проблема.

Если выбрать немного другой диапазон дат, запрос примет совершенно другой вид.

Какой диапазон? Не границы периода (в нашем случае - месяца), то есть не 1 и 30 числа.

Например. Выберем период `месяц`, даты `02.01.2015` - `29.11.2017`, показывать отчет `по всем сторам`.
```sql
(SELECT Date_format(period, '%Y-%m') AS `period`,
        Sum(qty_ordered)             AS `qty_ordered`,
        `sales_bestsellers_aggregated_monthly`.`product_id`,
        Max(product_name)            AS `product_name`,
        Max(product_price)           AS `product_price`,
        `sales_bestsellers_aggregated_monthly`.`product_type_id`
 FROM   `sales_bestsellers_aggregated_monthly`
 WHERE  ( rating_pos <= 5 )
        AND ( store_id IN( 10, 11, 5, 12,
                           9, 8, 7, 6 ) )
        AND ( period >= '2015-02-01' )
        AND ( period <= '2017-10-31' )
        AND ( product_type_id NOT IN ( 'grouped', 'configurable', 'bundle' ) )
 GROUP  BY `period`,
           `product_id`)
UNION ALL
(SELECT Date_format(period, '%Y-%m') AS `period`,
        Sum(qty_ordered)             AS `qty_ordered`,
        `sales_bestsellers_aggregated_daily`.`product_id`,
        Max(product_name)            AS `product_name`,
        Max(product_price)           AS `product_price`,
        `sales_bestsellers_aggregated_daily`.`product_type_id`
 FROM   `sales_bestsellers_aggregated_daily`
 WHERE  ( period >= '2015-01-02' )
        AND ( period <= '2015-01-31' )
        AND ( product_type_id NOT IN ( 'grouped', 'configurable', 'bundle' ) )
        AND ( store_id IN( 10, 11, 5, 12,
                           9, 8, 7, 6 ) )
 GROUP  BY `product_id`
 ORDER  BY `qty_ordered` DESC
 LIMIT  5)
UNION ALL
(SELECT Date_format(period, '%Y-%m') AS `period`,
        Sum(qty_ordered)             AS `qty_ordered`,
        `sales_bestsellers_aggregated_daily`.`product_id`,
        Max(product_name)            AS `product_name`,
        Max(product_price)           AS `product_price`,
        `sales_bestsellers_aggregated_daily`.`product_type_id`
 FROM   `sales_bestsellers_aggregated_daily`
 WHERE  ( period >= '2017-11-01' )
        AND ( period <= '2017-11-29' )
        AND ( product_type_id NOT IN ( 'grouped', 'configurable', 'bundle' ) )
        AND ( store_id IN( 10, 11, 5, 12,
                           9, 8, 7, 6 ) )
 GROUP  BY `product_id`
 ORDER  BY `qty_ordered` DESC
 LIMIT  5)
ORDER  BY `period` ASC,
          `qty_ordered` DESC  
```
Тут происходит примерно то же самое, что и выше. Но выглядит этот так, как будто пытались сделать лимит по каждой группе периода (каждому месяцу), построив дополнительные запросы.

Самое забавное, что лимит ставится только на обе границы выбранного диапазона (`2015-01` и `2017-11`), а весь остальной промежуток от `2015-2` до `2017-10` идет вообще без лимита и одним запросом.

Представим, что эта задумка удалась (хотя это не так). Тогда для каждого месяца строится свой запрос и все объединяется через `UNION_ALL`. 

Тогда что будет, если я хочу отчет за 10 лет - насколько страшный запрос я увижу?

`UNION_ALL`, который объединяет `12 * 10 = 120` запросов?

В общем, сделано все это как-то очень криво.

[Вернуться к содержанию...](#Содержание)

### Возможное решение проблемы

Cгруппировав и отсортировав по периоду товары, отсортировав их по количеству заказанных по убыванию, нужно брать только верхние `N` строк в каждой группе. При таком ограничении лишних строк не будет.

`N` - максимальное количество товаров в группе, которые выводим (задается прямо в фильтре грида, а дефолтное значение - в конфиге). Таким образом еще уберется хардкод пятерки.

Хочу заметить, что `rating_pos` особо и не нужен при выгрузке данных для этого отчета. Так как есть сумма заказанных единиц товара и сортировка по убыванию. Да и сам отчет о самых продаваемых товарах, а не о позиции в рейтинге.

Как-то так.

| Период       | Товар  | store_id | qty_ordered | rating_pos | group_row_num  |
| ------------ | :----: | :------: | :---------: | :--------: | :------------: |
| Январь 2017  | Кружка | 2        | 17          | 1          | 1              |
| Январь 2017  | Чашка  | 1        | 10          | 1          | 2              |
| Январь 2017  | Чашка  | 2        | 9           | 2          | 3              |
| Январь 2017  | Кружка | 1        | 7           | 2          | 4              |
| Январь 2017  | Лопата | 2        | 5           | 3          | 5              |
| Январь 2017  | Лопата | 2        | 3           | 3          | 6              |
| Январь 2017  | Грабли | 2        | 1           | 4          | 7              |
| Февраль 2017 | Кружка | 1        | 15          | 2          | 1              |
| Февраль 2017 | Грабли | 1        | 3           | 1          | 2              |
| Февраль 2017 | Лопата | 2        | 2           | 2          | 3              |
| Февраль 2017 | Лопата | 1        | 1           | 2          | 4              |

Затем фильтруем `where group_row_num <= 5`. Теперь количество строк в группе строго ограничивается.

Вот так:
```sql
SET @previousPeriod = '0000-00-00';
SELECT `e2`.*
   FROM   (SELECT `e`.*,
                  ( @periodrownumber := IF(period <> @previousperiod, 1,
                                        @periodrownumber + 1) )
                                                AS `period_row_number`,
                  ( @previousperiod := period ) AS `previous_period`
           FROM   (SELECT Date_format(period, '%Y-%m') AS `period`,
                          Sum(qty_ordered)             AS `qty_ordered`,
                          `sales_bestsellers_aggregated_monthly`.`product_id`,
                          Max(product_name)            AS `product_name`,
                          Max(product_price)           AS `product_price`,
                          `sales_bestsellers_aggregated_monthly`.`product_type_id`
                   FROM   `sales_bestsellers_aggregated_monthly`
                   WHERE  ( rating_pos <= '15' )
                          AND ( store_id IN( 10, 11, 5, 12,
                                             9, 8, 7, 6 ) )
                          AND ( period >= '2015-01-02' )
                          AND ( period <= '2017-11-29' )
                          AND ( product_type_id NOT IN ( 'grouped', 'configurable',
                                                         'bundle' ) )
                   GROUP  BY `period`,
                             `product_id`,
                             `product_type_id`
                   ORDER  BY `period` ASC,
                             `qty_ordered` DESC) AS `e`) AS `e2`
   WHERE  ( e2.period_row_number <= '15' )
```
Возможно, можно было сделать запрос более компактным, но это самое компактное, что у меня нормально заработало, больше ужать не удалось.

То есть, подобным образом стоит переписать запрос в `Mage_Sales_Model_Resource_Report_Bestsellers_Collection`, который выгружает данные для отчета.

В итоге, всегда будет только один запрос для указанного диапазона со строгим ограничем.

Теперь, если считать, что маджента правильно собирает данные для отчета в эти три таблицы, все будет работать нормально.

Что значит "если считать, что маджента правильно собирает данные"?

Цитата выше:
> Потому что даже в одном сторе почему-то встречаются товары с одинаковой позицией в рейтинге.

Может, это был индивидуальный косяк.

Тем не менее, опять же , это не особо важно.

Потому что запросы, которые строят таблицы, вроде бы вполне логичные. Я имею в виду, что подсчет общей суммы заказанных единиц товара вроде считается нормально. 

Да и потому что не стоит вообще смотреть на `rating_pos`:
> Хочу заметить, что `rating_pos` особо и не нужен при выгрузке данных для этого отчета. Так как есть сумма заказанных единиц товара и сортировка по убыванию. Да и сам отчет о самых продаваемых товарах, а не о позиции в рейтинге.

[Вернуться к содержанию...](#Содержание)
