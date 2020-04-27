# Тествое задание

## TODO

Написать key-value базу данных
соответствующую следущим условиям:

* Принемает входящие по RabbitMQ из очереди `config.INCOME_QUEUE`
* Отправляет ответ в очередь `config.OUTCOME_QUEUE`
* Формат сообщений разработать и задокументировать самостоятельно
* Поддерживать операции чтения, вставки, удаления
* Обеспечивает персистентность данных через механизиы snapshot + log
* Запускаться через docker
* Должен написан docker-compose файл, обеспечивающий персестентность базы

## Решение

### База данных

Ключ и значение храняться в файле с данными базы.
Индекс БД содержит записи содержащие размер ключа и значения, а также их положение в файле данных базы.
Так же индекс содержит диапазоны удаленных записей.

В случае добавления пары ключ-значение, если запись существовала она удаляется.
Производится поиск свободного места в диапазоне удалленных записей и выдается наименьшая свободная позиция или,
если позция не найдета, пара дописывается в конец файла.

В случае запроса на удаление, реального удаления из файла с данными не производится,
а позиция и размер записи лишь добавляется в диапазо удаленных в индексе.  

Так же ведеться журнал транзакций изменения индекса БД. Если индекс не был обновлен,
например в случае падения приложения, транзакции будут применены к индексу при инициализации.
Каждые N транзакций файл индекса перезаписывается, а файл транзакций очищается.

Snapshot реализован следующим образом. При создании снапшота сохраняется полная копия индекса, файл транзакций и
файл разницы данных. При перезаписи данных в базе, оргиниальные данные сохраняются и транзакия
сохраняются в файлах снапшота. В индексе базы записывается ссылка на текущий снапшот.

Лог операций с базой ведеться в проки, реализующим интерфейс хранилища (БД), и записывается в файлы
с разными уровнями логгирования.

Реализация базы данных отделена от клиента, что бы при необходимости ее можно было заменить.

### Очереди сообщений

Очередь сообщений, также как и база, предполагает замену реализации и разделена на клиент и реализацию.
Ради этого такж е реализована передача id сообщения в теле самого сообщения, а не через свойства сообщения RabbitMQ. 
Формат сообщений подразумевает версионность, на случай будущих зменений.

Формат сообщений на запрос к базе:

1 байт   | 1 байт            | 1 байт        | 2 байта            | 2 байта      | N байт     | N байт     | N байт
-------- | ----------------- | ------------- | ------------------ | ------------ | ---------- |----------- | -----
Пропущен | Версия протокола  | Метод запроса | Размер id запроса  | Размер ключа | id запроса | Тело ключа | Тело значения

Тело значения опускаеться если не укзан метод set.

Формат сообщений ответа от базы:

1 байт   | 1 байт            | 1 байт | 1 байт        | 2 байта            | N байт     | N байт      
-------- | ----------------- | ------ | ------------- | ------------------ | ---------- |------------ 
Пропущен | Версия протокола  | Статус | Метод запроса | Размер id запроса  | id запроса | Тело ответа

Тело ответа опускаеться если не укзан метод get.

Первый байт зарезервирован для непредвиденных случаев (флаги запроса, увелиечение версии больше чем 255 и т.д.).
Id запроса может принемать любое значение ограниченое размером 2^16 байт,
хотя в следующей версии протокола его можно сделать определенным, например twitter snowflake.

### Docker-compose файл

Для обеспечения персистентности все данные БД и логи сохраняются в именованый вольюм.

### Демо

Так же написан небольшой gateway для демонстрации на 80-м порту. 