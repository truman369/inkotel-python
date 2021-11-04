<!-- TODO: разделить справку по cli и по функциям классов, разделить примеры -->
# Inkotel Scripts (Python)
> Скрипты для упрощения работы с коммутаторами в сети ИНКО

На данный момент реализовано:
- быстрое подключение к коммутаторам в интерактивном режиме
- отправка произвольных команд с получением результата их выполнения
- отправка команд из файла (поддерживаются шаблоны j2)
- локальная база данных коммутаторов с поиском по модели и расположению

## Установка

### Клонируем репозиторий

```shell
$ git clone https://git.truman.network/inkotel/scripts-python.git
$ cd scripts-python
```

### [ОПЦИОНАЛЬНО] создаем виртуальное окружение

```shell
$ python -m venv .venv
$ source .venv/bin/activate
```

### Устанавливаем зависимости

```shell
$ pip install -r requirements.txt
```

> Если при установке зависимости `easysnmp` возникает ошибка, возможно, в системе не установлена библиотека `net-snmp`. Подробнее в руководстве [easysnmp](https://easysnmp.readthedocs.io/en/latest/#installation).


> Если при запуске скриптов возникает ошибка привилегий, возможно, значение параметра ядра `net.ipv4.ping_group_range` равно `1 0`, нужно исправить его на `0 2147483647`. В разных дистрибутивах используются разные значения по умолчанию. Подробнее в документации [icmplib](https://github.com/ValentinBELYN/icmplib/blob/main/docs/6-use-icmplib-without-privileges.md).

## Настройка

Файлы конфигурации находятся в директории `config`. Используется формат `YAML`. С репозиторием поставляются файлы стандартных настроек `*.sample.yml`. Для начальной конфигурации можно использовать модуль `config.py`, при запуске он скопирует все стандартные настройки в соответствующие файлы.

### Логины и пароли

Файл `secrets.yml` содержит пароли для доступа к коммутаторам.

- `admin_profile` - профиль для узловых коммутаторов
- `user_profile` - профиль для коммутаторов уровня доступа

```yaml
---
admin_profile:
  login: admin
  password: adminpassword
user_profile:
  login: user
  password: userpassword
...
```

## Основные модули

### sw.py

> Используется для подключения к коммутаторам и работы с ними

Можно запускать как отдельный скрипт, так и импортировать классы и функции.

```shell
$ ./sw.py IP COMMAND
```

#### IP

ip-адрес коммутатора, поддерживается сокращенный формат `x.x`, в этом случае подразумевается `192.168.x.x`. Практически все консольные утилиты из данного репозитория поддерживают как полный, так и сокращенный формат записи ip.

#### COMMAND

##### show

Выводит краткую информацию о коммутаторе в виде строки формата:
```
model [short_ip] location
```
Расположение коммутатора будет выделено цветом:
- **зеленый** - обычные коммутаторы доступа
- **ярко-красный** - центральные свитчи
- **ярко-желтый** - узловые свитчи с маршрутизацией
- **ярко-зеленый** - узловые расширители портов 1G
- **ярко-синий** - узловые расширители портов 10G
- **бледно-синий** - узловые расширители портов 10G с урезанным CLI
- **ярко-голубой** - GPON на АТС
- **бледно-голубой** - GPON на Осипенко, CLI малофункционален
- **серый** - Huawey на АТС, доступ через telnet запрещен

Если указан флаг `--full`, второй строчкой выводится mac-адрес устройства.

##### connect

Подключается к коммутатору через `telnet` и передает управление пользователю. Стандартное приветствие коммутатора заменяется короткой строкой, генерируемой описанной выше командой `show`. Заголовок окна также меняется на `[short_ip] location`

##### send

Два режима работы:

- Отправляет одну или несколько команд на коммутатор, возвращает результат выполнения команд. Команды можно разделять либо символом ';', либо новой строкой. Пустые строки и отступы при этом игнорируются.

- При указании флага `--file` в качестве аргумента нужно передать имя файла шаблона из директории `templates`. После имени файла можно указать дополнительные аргументы в формате `key=value` для передачи внутрь шаблона.

#### Файлы шаблонов

Команда `send` поддерживает отправку списка команд из файла с возможностью использования шаблонов [jinja2](https://jinja.palletsprojects.com/). Внутри шаблонов можно использовать переменную `sw`, которая ссылается на экземпляр класса текущего коммутатора. Также внутрь шаблона передаются любые дополнительные именованные аргументы, если они указаны. Все файлы шаблонов хранятся в директории `templates`.

На данный момент протестированы на всех моделях коммутаторов следующие предустановленные шаблоны:
- `save.j2` - сохранение коммутаторов
- `port_state.j2` - изменение состояния порта. Параметры: `port` - номер порта. `state` - включить или выключить порт, поддерживаются значения `True`/`False`, либо `0`/`1`, если параметр не задан, то выполняется команда для просмотра состояния порта. `comment` - комментарий на порту, если параметр не задан, то старый комментарий не меняется, если явно задать пустую строку, `None`, `0` или `False`, то комментарий очищается.

### db.py

> Используется для работы с локальной базой данных коммутаторов

Можно запускать как отдельный скрипт, так и импортировать классы и функции. Данные хранятся в директории `data`. Используется база данных `SQLite`.

#### Генерация актуальной базы данных

```shell
$ ./db.py generate
```
В этом режиме из базы удаляются все старые записи, далее проверяются все возможные адреса коммутаторов и добавляются те, которые доступны на данный момент. Вся процедура на текущей сети ИНКО занимает около 2 минут из-за таймаутов ожидания недоступных коммутаторов. 

#### Вывод списка ip

```shell
$ ./db.py list
```
Выводит список всех ip адресов и общее количество записей в базе данных.

#### Работа с конкретным ip

```shell
$ ./db.py add IP
$ ./db.py get IP
$ ./db.py delete IP
```
Добавление коммутатора в базу, просмотр информации из базы, удаление коммутатора.

#### Поиск по базе

```shell
$ ./db.py search STRING
```

Выполняет поиск в названии модели или адресе установки коммутатора.


### wsgi.py

> api модуль для работы с локальной базой данных

- **'/'**, `GET` - получение списка всех ip адресов коммутаторов из базы в формате json

- **'/<ip address>'**, `GET`, `POST`, `DELETE` - просмотр, добавление, удаление коммутатора, никаких параметров передавать не нужно

## Примеры использования

### Быстрое подключение к коммутаторам

Добавляем простую функцию-алиас в `.bashrc` или `.zshrc`:

```sh
tt () {
    cur_dir=$(pwd)
    cd ~/scripts-python
    source .venv/bin/activate
    python sw.py $@ connect
    deactivate
    cd $cur_dir
}
```

Подключаемся к коммуторам командой:
```shell
$ tt 59.75
```

### Вывод всех коммутаторов qtech

```shell
$ ./db.py search qsw
```

### Отправка команд и вывод результата

```shell
$ ./sw.py 59.75 send "sh ports 1; conf ports 1 st d; sh ports 1; conf ports 1 st e"
$ ./sw.py 58.236 send "sh run int eth 1/2; conf t; int eth 1/2; no shut; end; sh run int eth 1/2; conf t; int eth 1/2; shut;"
$ ./sw.py 59.75 send "sh vlan
dquote> create vlan 666 tag 666; conf vlan 666 add tag 1
dquote> sh vlan"
```
### Работа в рамках одной telnet-сессии

```python
from sw import Switch

test_sw = Switch('192.168.59.75')

print(f'telnet содениение инициируется при вызове первой команды:\n',
      test_sw.send("sh sw"))
print('очищаем логи')
test_sw.send("cl log")
print('переходим в интерактивный режим, для выхода нажмите `Ctrl+]`')
test_sw.interact()
print('запускаем еще команды без вывода результатов')
test_sw.send(['create vlan 666 tag 666',
              'conf vlan 666 add tag 1'])
print(f'печатаем вывод sh vlan 666:\n', test_sw.send('sh vlan 666'))
print('проверяем логи, новых telnet соединений не должно быть')
print(test_sw.send("sh log"))
```

### Выполнение списка команд из файла

При импорте класса:
```python
result = test_sw.send(template='save.j2')
# пример с дополнительными аргументами:
result = test_sw.send(template='port_state.j2', port=1, state=False, comment='blocked port')
```
При запуске скрипта:
```shell
$ ./sw.py 59.75 send --file save.j2
$ ./sw.py 59.75 send --file port_state.j2 port=1 state=False comment='blocked port'
```
