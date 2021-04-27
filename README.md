# dataBaseLesson

## Что такое Sql-инъекция?

**SQL инъекция** — это один из самых доступных способов взлома сайта.
Суть таких инъекций – внедрение в данные (передаваемые через GET, POST запросы или значения Cookie) произвольного SQL кода. Если сайт уязвим и выполняет такие инъекции, то по сути есть возможность творить с БД (чаще всего это MySQL) что угодно.


## Как вычислить уязвимость, позволяющую внедрять SQL инъекции?

Довольно легко. Например, есть тестовый сайт test.ru. На сайте выводится список новостей, с возможностью детального просомтра. Адрес страницы с детальным описанием новости выглядит так: *test.ru/* *?detail=1*. Т.е через GET запрос переменная detail передаёт значение 1 (которое является идентификатором записи в табице новостей).

Изменяем GET запрос на *?detail=1'* или *?detail=1"*. Далее пробуем передавать эти запросы серверу, т.е заходим на *test.ru/?* *detail=1'* или на *test.ru/?detail=1"*.

Если при заходе на данные страницы появляется ошибка, значит сайт уязвим на SQL инъекции.

![Ошибка](https://habrastorage.org/getpro/habr/post_images/98c/b32/191/98cb321919ea63fc4524e5e885896a83.jpg)

**Возможные SQL инъекции (SQL внедрения)**
1) Наиболее простые — сворачивание условия WHERE к истиностному результату при любых значениях параметров.
2) Присоединение к запросу результатов другого запроса. Делается это через оператор UNION.
3) Закомментирование части запроса.

## Практика. Варианты взлома сайта с уязвимостью на SQL внедрения

Итак, у нас есть уже упоминавшийся сайт **test.ru**. В базе хранится 4 новости, 3 из которых выводятся. Разрешение на публикацию новости зависит от парметра public (если параметр содержит значение 1, то новость публикуется).

![Список новостей, разрешённых к публикации](https://habrastorage.org/getpro/habr/post_images/d26/ea2/0b7/d26ea20b7eaa192eac01ffc180c62e66.jpg)

При обращении к странице ****test.ru/?detail=4**, которая должна выводить четвёртую новость появляется ошибка – новость не найдена.
В нашем случае новость существует, но она запрещена к публикации.

![](https://habrastorage.org/getpro/habr/post_images/968/a44/6b7/968a446b76347ad0edab65196f37e8e7.jpg)

Но так как мы уже проверяли сайт на уязвимость и он выдавал ошибку БД, то пробуем перебирать возможные варианты запросов.

**Тестирую следующие варианты:**

    test.ru/?detail=4+OR+1
    test.ru/?detail=4+--
    test.ru/?detail=4+UNION+SELECT+*+FROM+news+WHERE+id=4


В итоге удача улыбнулась и два запроса (первый и третий) вернули нам детальное описание четвёртой новости

![](https://habrastorage.org/getpro/habr/post_images/32e/a51/6c8/32ea516c8d15554506911bd78ec30ba3.jpg)

## Разбор примера изнутри

За получение детального описания новости отвечает блок кода:

    $detail_id=$_GET['detail'];
    $zapros="SELECT * FROM `$table_news` WHERE `public`='1' AND `id`=$detail_id ORDER BY `position` DESC";

Мало того, что $detail_id получает значение без какой либо обработки, так ещё и конструкция `id`=$detail_id написана криво, лучше придерживаться *`id`='$detail_id'* (т.е сравниваемое значение писать в прямых апострофах).

Глядя на запрос, получаемый при обращении к странице через *test.ru/?detail=4+OR+1*

    SELECT * FROM `news` WHERE `public`='1' AND `id`=4 OR 1 ORDER BY `position` DESC

становится не совсем ясно, почему отобразилась 4-ая новость. Дело в том, что запрос вернул все записи из таблицы новостей, отсортированные в порядке убывания сверху. И таким образом наша 4-ая новость оказалась самой первой, она же и вывелась как детальная. Т.е просто совпадение.

Разбираем запрос, сформированный при обращении через *test.ru/?detail=4+UNION+SELECT+*+FROM+news+WHERE+id=4*.

Тут название таблицы с новостями (в нашем случае это news) бралось логическим перебором.
Итак, выполнился запрос *SELECT * FROM `news` WHERE `public`='1' AND `id`=4 UNION SELECT * FROM news WHERE id=4 ORDER BY `position` DESC*. К нулевому результату первой части запроса (до UNION) присоединился результат второй части (после UNION), вернувшей детальное описание 4-ой новости.

## Защита от Sql-инъекции

### Приведение к целочисленному типу

В SQL-запросы часто подставляются целочисленные значения, полученные от пользователя. В примерах выше использовался идентификатор города, полученный из параметров запроса. Этот идентификатор можно принудительно привести к числу. Так мы исключим появление в нём опасных выражений. Если хакер передаст в этом параметре вместо числа SQL код, то результатом приведения будет ноль, и логика всего SQL-запроса не изменится.

PHP умеет присваивать переменной новый тип. Этот код принудительно назначит переменной целочисленный тип:

    $city_id = $_GET['city_id'];
    settype($city_id, 'integer');

### Экранирование 

Экранирование добавляет в строке перед кавычками (и другими спецсимволами) знак обратного слэша \.
Такая обработка лишает кавычки их статуса — они больше не определяют конец значения и не могут повлиять на логику SQL-выражения.

За экранирование значений отвечает функция *mysqli_real_escape_string()*.
Этот код обработает значение из параметра, сделав его безопасным для использования в запросе:

    $city_name = mysqli_real_escape_string($link, $_GET['search']);
    $sql = "SELECT * FROM cities WHERE name LIKE('%$city_name%')";

### Подготовленные выражения

Так как данные не отделены от **SQL-кода**, они могут влиять на логику всего выражения. К счастью, **MySQL** предлагает способ передачи данных отдельно от кода. Такой способ называется подготовленными запросами.

Выполнение подготовленных запросов состоит из двух этапов: вначале формируется шаблон запроса — обычное SQL-выражение, но без действительных значений, а затем, отдельно, в **MySQL** передаются значения для этого шаблона.
Первый этап называется подготовкой, а второй — выражением. Подготовленный запрос можно выполнять несколько раз, передавая туда разные значения.

**Этап подготовки**
На этапе подготовки формируется SQL-запрос, где на месте значений будут находиться знаки вопроса — плейсхолдеры. Эти плейсхолдеры в дальнейшем будут заменены на реальные значения. Шаблон запроса отправляется на сервер **MySQL** для анализа и синтаксической проверки.
Пример:

    $sql = "SELECT * FROM cities WHERE name = ?";
    $stmt = mysqli_prepare($link, $sql);

Этот код сформирует подготовленное выражение для выполнения вашего запроса.

За подготовкой идёт выполнение. Во время запуска запроса PHP привязывает к плейсхолдерам реальные значения и посылает их на сервер. За передачу значений в подготовленный запрос отвечает функция *mysqli_stmt_bind_param()*. Она принимает тип и сами переменные:

    mysqli_stmt_bind_param($stmt, 's', $_GET['search']);

После выполнения запроса получить его результат в формате mysqli_result можно функцией mysqli_stmt_get_result():

    $res = mysqli_stmt_get_result($stmt);

    // чтение данных
    while ($row = mysqli_fetch_assoc($res)) {
        // ассоциативный массив с очередной записью из результата
        var_dump($row);
    }

Значения привязанных к запросу переменных сервер экранирует автоматически. Привязанные переменные отправляются на сервер отдельно от запроса и не могут влиять на него. Сервер использует эти значения непосредственно в момент выполнения, уже после того, как был обработан шаблон выражения. Привязанные параметры не нуждаются в экранировании, так как они никогда не подставляются непосредственно в строку запроса.

