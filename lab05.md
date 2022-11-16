# Lab05. Подготовка матрицы users x items по логам из Kafka для прогнозирования пола и возраста

## I. Задача с высоты птичьего полета

В этой задаче входными данными являются выходные данные Lab4a – датасеты в папках `/user/name.surname/visits/view/*`, `/user/name.surname/visits/buy/*`. 

Для данного тренировочного датасета приготовьте матрицу типа 'users x items', в которой строки – юзеры (uid), а колонки – товары, которые просмотрел данный юзер или купил. Для каждого просмотренного товара должна быть заведена колонка "view_item_id", а для каждого купленного товара – колонка "buy_item_id". Для каждого юзера в каждой ячейке должно быть указано сколько раз он или она посетили или купили данный товар (0 если ни разу). Названия полей item_id должны быть нормализованы – приведены к нижнему регистру, пробелы и тире заменены на подчерк. (т.е. например `view_computers_2" и так далее)

При повторном запуске с новыми входными данными, ваша программа должна уметь добавлять строки с новыми пользователями к существующей матрице. 

Но этим дело не ограничится. Приложения для Лабы 4 и Лабы 5 должны будут работать совместно, как на конвеере.

Сначала вы обработаете уже имеющийся датасет по пути `/user/name.surname/visits/`, а затем в Kafka придут новые данные, которые сначала обрабатываются приложением из Лабы 4. Затем эти новые данные обработываются приложением из Лабы 5 и записываются по новому пути.

Функция добавления новых столбцов (товаров) проверяться не будет. Функция обновления существующих данных (старый покупатель посмотрел или купил новые товары на другой день) тоже не будет проверяться. Но вы можете сами заняться реализацией этих функций в свое удовольствие.

![Alt text](images/img5.png?raw=true "Архитектура")


## II. Оформление работы

В вашем репо в подпапке `lab05/users_items` положите `sbt-project` под названием `users_items` с главным классом `users_items` в файле `users_items.scala`.

Проект должен компилироваться и запускаться следующим образом:

```
cd lab05/users_items
sbt package
spark-submit --class users_items --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.5 .target/scala-2.11/users_items_2.11-1.0.jar 
```

## III. Программа

`users_items.scala` должна принимать следующие опции через объект `spark.conf`:

* `spark.users_items.update`: 0 или 1 режим работы, сделать новую матрицу `users_items` или добавить строки к существующей. По умолчанию - 1, добавить строки в матрицу.
* `spark.users_items.output_dir`: абсолютный или относительный путь к выходным данным.
* `spark.users_items.input_dir`: абсолютный или относительный путь к входным данным. 

В выходной директории вы должны записать матрицу `users-items` в формате parquet по пути `YYYYMMDD`, где дата соответствует последнему дню в данных. То есть если у вас в папке `spark.users_items.input_dir` самые последние данные датированы `20200429`, то ваша матрица `users-items` должна располагаться по пути `20200429` в директории `spark.users_items.output_dir`. Используйте временную зону UTC.

С опцией `update=1` и при наличии данных в `output_dir`, ваша программа считывает последнюю версию матрицы `users-items` в `output_dir`, добавляет данные из `input_dir` и записывает новую матрицу по новому пути в соответствии с датой последних данных из `input_dir`. См. пример ниже.

## IV. Проверка

1. Запустите приложение `users_items.scala` из Лабы 5 со следующими агрументами (имеющийся датасет `visits`): 
* `spark.users_items.input_dir=/user/name.surname/visits`
* `spark.users_items.output_dir=/user/name.surname/users-items`
* `spark.users_items.update=0`

2. Когда оно отработает, запустите чекер

3. Чекер скопирует вашу матрицу `users-items` локально в директорию `$checker_dir/users-items`, проверит наличие директории `$checker_dir/users-items/20200429` и выборочно проверит одного пользователя (`user1`)

4. Далее чекер запишет новую порцию данных (`visits2`) в ваш топик кафка `name_surname`. Придут сообщения, датированные новым днем.

5. Затем чекер самостоятельно соберет и запустит приложение `filter` из Лабы 4а со следующими параметрами:

* `spark.filter.topic_name=name_surname`
* `spark.filter.offset=earliest`
* `spark.filter.output_dir_prefix=file:///$checker_dir/visits2`

:warning: Обратите внимание, что в топике не должно быть данных перед запуском! То есть следуют переждать период примерно 20 минут перед повторным запуском. Иначе результат может получиться неверным.

5. Когда приложение `filter` отработает, чекер будет ожидать появление новой директории `$checker_dir/visits2` локально. Далее чекер запустит приложение `users_items` из Лабы 5 со следующими параметрами:

* `spark.users_items.input_dir=file:///$checker_dir/visits2`
* `spark.users_items.output_dir=file:///$checker_dir/users-items`
* `spark.users_items.update=1`

Оно должно считать новые данные из `visits2` и старую матрицу из пути `$checker_dir/users-items/20200429` (где дата соответствует последнему дню в visits) и записать новую матрицу со старыми+новыми данными по пути `$checker_dir/users-items/20200430` (где дата должна соответствовать последнему дню новых данных из visits2).

6. Чекер выборочно проверит данные другого пользователя (`user2`) в новой матрице.


### Поля чекера

* `00_git_correct` = True/False – репозиторий прошел проверку
* `01_info_git_errors` = "" – ошибки проверки репозитория
* `02_users_items1_correct` = True/False – данные пользователя из набора `visits` в матрице `users_items` прошли проверку
* `03_info_users_items1_data` = {} – данные проверки пользователя из набора `visits` в матрице `users_items`
* `04_info_users_items1_errors` = "" – ошибки проверки пользователя из набора `visits` в матрице `users_items`
* `05_info_kafka_errors` = "" – удалось ли подключиться к брокеру и др.
* `06_info_number_sent` = "" – количество посланных записей
* `07_info_number_recieved` = "" – количество полученных записей
* `08_total_number_correct` = True/False – совпадает ли общее количество принятых записей с количеством отосланных
* `09_info_checksums` = {} – "чексуммы" данных для просмотров и покупок из набора `visits2`
* `10_info_subtotals` = {} – количество записей в данных для просмотров и покупок из набора `visits2`
* `11_checksums_correct` = True/False – данные для просмотров правильные.
* `12_subtotals_correct` = True/False – данные для покупок правильные.
* `13_users_items2_correct` = True/False – данные проверки пользователя из нового набора `visits2` в обновленной матрице `users_items` прошли проверку
* `14_info_users_items2_data` = {} – данные проверки пользователя из нового набора `visits2` в обновленной матрице `users_items`
* `15_info_users_items2_error` = "" – ошибки проверки пользователя из нового набора `visits2` в обновленной матрице `users_items`
* `16_info_your_data` = {} – другие данные
* `17_info_errors` = "" – другие ошибки, если есть
* `18_lab_result` = True/False – общий результат

Логи чекера доступны по пути: `/tmp/logs/sb1laba05/$username/lab.log`