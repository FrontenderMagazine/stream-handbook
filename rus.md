# Работа с потоками в node.js

Статья является вольным переводом [stream-handbook](https://github.com/substack/stream-handbook) и охватывает основы создания [node.js](http://nodejs.org/) приложений с использованием [потоков](http://nodejs.org/docs/latest/api/stream.html).

# Предисловие

```
"Нам нужен способ взаимодействия между программами, наподобие того как садовый шланг можно подключать к разным сегментам и изменять направление воды. То же самое можно сделать с вводом-выводом данных"
```

[Дуглас Макилрой. 11 октября 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

Потоки пришли к нам из [первых дней эпохи Unix](http://www.youtube.com/watch?v=tc4ROCJYbm0) и зарекомендовали себя в течении многих десятилетий как надежный способ создания сложных систем из маленьких компонентов, которые [делают что-то одно, но делают это хорошо](https://ru.wikipedia.org/wiki/Философия_UNIX). В Unix потоки реализуются в оболочке с помощью знака `|` (pipe). В node встроенный [модуль потоков](http://nodejs.org/docs/latest/api/stream.html) используется в базовых библиотеках, кроме этого его можно подключать в свой код. Подобно Unix, в node основной метод модуля потоков называется `.pipe()`. Он позволяет соединять потоки с разной скоростью передачи данных таким образом что данные не будут потеряны.

Потоки помогают [разделять ответственность](https://ru.wikipedia.org/wiki/Разделение_ответственности), поскольку позволяют вынести все взаимодействие в отдельный интерфейс, который может быть [использован повторно] (http://www. faqs.org/docs/artu/ch01s06.html#id2877537). Вы сможете подключить вывод одного потока на ввод другого, и [использовать библиотеки](http://npmjs.org) которые будут работать с подобными интерфейсами на более высоком уровне.

Потоки - важный элемент микроархитектурного дизайна и философии UNIX, но кроме этого есть еще достаточное количество важных абстракций для рассмотрения. Всегда помните своего врага ([технический долг] (http://c2.com/cgi/wiki?TechnicalDebt)) и ищите наиболее подходящие для решения задач абстракции.

![brian kernighan](http://substack.net/images/kernighan.png)

***

# Почему вы должны использовать потоки

Ввод-вывод в node асинхронен, поэтому взаимодействие с диском и сетью происходит через передачу калбеков в функции (прим. переводчика: а также через обещания, генераторы, и т.п.). Следующий код отдает файл браузеру:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        res.end(data);
    });
});
server.listen(8000);
```

Этот код работает, но он буферизирует весь `data.txt` в память при каждом запросе. Если `data.txt` достаточно большой, ваша программа начнет потреблять слишком много оперативной памяти, особенно при большом количестве подключений юзеров с медленными каналами связи.

Пользователи также будут недовольны, потому что им придется ждать пока весь файл не будет считан в память на сервере перед отправкой.

К счастью, оба аргумента `(req, res)` являются потоками, а это значит что мы можем переписать код с использованием `fs.createReadStream()` вместо `fs.readFile()`:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(res);
});
server.listen(8000);
```

Теперь `.pipe()` самостоятельно слушает события `'data'` и`'end'` от потока созданного через `fs.createReadStream()`. Этот код не только чище, но теперь и `data.txt` доставляется клиенту сразу, по частям по мере чтения его с диска.

Использование `.pipe()` имеет ряд других преимуществ, например автоматическая обработка скорости ввода-вывода - node не будет буферизировать лишние части файла в память пока предыдущие части не отправлены клиенту с медленным соединением.

Вам хочется сжатия? Их есть у меня!

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

Теперь наш файл cжимается для браузеров, которые поддерживают gzip или deflate! Мы просто отдаем модулю [opressor](https://github.com/substack/oppressor) всю логику обработки content-encoding и забываем про нее.

После того как вы ознакомитесь с API потоков, вы сможете писать потоковые модули и соединять их как кусочки лего, вместо того чтобы изобретать свои велосипеды и пытаться запомнить все способы взаимодействия между непотоковыми модулями.

Потоки делают программирование в node простым, элегантным и компонуемым.

# Основы

Существует 5 видов потоков: на чтение (readable), на запись (writeable), трансформирующие (transform), дуплексные (duplex) и классические (classic).

## pipe()

Все типы потоков могут использовать`.pipe()` для соединения входов с выходами.

`.pipe()` всего лишь функция, которая берет поток на чтение `src` и соединяет его вывод с вводом потока на запись `dst`:

```
src.pipe(dst)
```

`.pipe(dst)` возвращает `dst`, так что вы можете связывать сразу несколько потоков:

``` js
a.pipe(b).pipe(c).pipe(d)
```
или то же самое:

``` js
a.pipe(b);
b.pipe(c);
c.pipe(d);
```

Аналогично в Unix вы можете связать утилиты вместе:

```
a | b | c | d
```

## Потоки на чтение (readable)

Потоки на чтение производят данные которые с помощью `.pipe()` могут быть переданы в поток на запись, трансформирующий или дуплексный поток:

``` js
readableStream.pipe(dst)
```

### Создание потока на чтение

Давайте создадим считываемый поток!

``` js
var Readable = require('stream').Readable;

var rs = new Readable;
rs.push('beep ');
rs.push('boop\n');
rs.push(null);

rs.pipe(process.stdout);
```

```
$ node read0.js
beep boop
```

`rs.push(null)` сообщает потребителю, что `rs` закончил вывод данных.

Заметьте, мы отправили содержимое в поток на чтение `rs` ДО привязывания его к `process.stdout`, но сообщение все равно появилось в консоли.

Когда вы посылаете с помощью `.push()` данные в поток на чтение, они буферизируются до тех пор пока потребитель не будет готов их прочитать.

Тем не менее, в большинстве случаев будет лучше если мы не будем их буферизировать совсем, вместо этого будем генерировать их только когда данные запрашиваются потребителем.

Мы можем посылать данные кусками, определив функцию `._read`:

``` js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97;
rs._read = function () {
    rs.push(String.fromCharCode(c++));
    if (c > 'z'.charCodeAt(0)) rs.push(null);
};

rs.pipe(process.stdout);
```

```
$ node read1.js
abcdefghijklmnopqrstuvwxyz
```

Теперь мы помещаем буквы  от `'a'` до `'z'` включительно, но только тогда когда потребитель будет готов их прочитать.

Метод `_read` также получает в первом аргументе параметр `size`, который указывает сколько байт потребитель хочет прочитать - он необязательный, так что ваша реализация потока может его игнорировать.

Обратите внимание, вы также можете использовать `util.inherits()` для наследования от базового потока, но такой подход может быть непонятен тому кто будет читать ваш код.

Чтобы продемонстрировать, что наш метод `_read` вызовется только когда потребитель запросит данные, добавим задержку в наш поток:

```js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97 - 1;

rs._read = function () {
    if (c >= 'z'.charCodeAt(0)) return rs.push(null);

    setTimeout(function () {
        rs.push(String.fromCharCode(++c));
    }, 100);
};

rs.pipe(process.stdout);

process.on('exit', function () {
    console.error('\n_read() called ' + (c - 97) + ' times');
});
process.stdout.on('error', process.exit);
```

Запустив программу, мы увидим, что если мы запросим 5 байт - `_read ()` вызовется 5 раз:

```
$ node read2.js | head -c5
abcde
_read() called 5 times
```

Задержка через setTimeout необходима, так как операционной системе требуется определенное время чтобы послать сигнал о закрытии конвейера.

Обработчик `process.stdout.on('error', fn)` также необходим, поскольку операционная система пошлет SIGPIPE нашему процессу, когда утилите `head` больше не будет нужен результат нашей программы (в этом случае будет вызвано событие EPIPE в потоке `process.stdout`).

Эти усложнения необходимы при взаимодействии с конвейером в операционной системе, но в случае реализации потоков чисто в коде они будут обработаны автоматически.

Если вы хотите создать читаемый поток, который выдает произвольные форматы данных вместо строк и буферов, убедитесь что вы его инициализировали с соответствующей опцией: `Readable ({ objectMode: true })`.

### Использование потока на чтение

В большинстве случаев мы будем подключать такой поток к другому потоку, созданному нами или модулями наподобие [through](https://npmjs.org/package/through),  [concat-stream](https://npmjs.org/package/concat-stream). Но иногда может потребоваться использовать его напрямую.

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    console.dir(buf);
});
```

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js
<Buffer 61 62 63 0a>
<Buffer 64 65 66 0a>
<Buffer 67 68 69 0a>
null
```

Когда данные становятся доступными, возникает событие `'readable'`, и вы можете вызвать `.read()` чтобы получить следующую порцию данных из буффера.

Когда поток завершится, `.read()` вернет `null`, потому что не останется доступных для чтения байтов.

Вы можете запросить определенное количество байтов: `.read(n)`. Указание необходимого размера носит рекомендательный характер, и не сработает на потоков возвращающих объекты. Однако, все базовые потоки поддерживают эту опцию.

Пример чтения в буффер порциями по 3 байта:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf.toString());
});
```

Но, при запуске этого примера мы получим не все данные:

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js
'abc'
'\nde'
'f\ng'
```
Это произошло потому что последняя порция данных осталвсь во внутреннем буфере, и нам надо "подопнуть" их. Сделаем мы это сообщив с помощью `.read(0)` что нам надо больше данных чем только что полученые 3 байта:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf.toString());
    process.stdin.read(0);
});
```

Теперь наш код работает как и ожидалось.

``` js
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js
'abc'
'\nde'
'f\ng'
'hi\n'
```

В случае если вы получили больше данных чем вам требуется - можно использовать `.unshift()` чтобы вернуть их назад.

Использование `.unshift()` помогает нам предотвратить получение ненужных частей. К примеру, создадим парсер который разделяет абзац на строки с делителем - переносом строки:

``` js
var offset = 0;

process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    if (!buf) return;
    for (; offset < buf.length; offset++) {
        if (buf[offset] === 0x0a) {
            console.dir(buf.slice(0, offset).toString());
            buf = buf.slice(offset + 1);
            offset = 0;
            process.stdin.unshift(buf);
            return;
        }
    }
    process.stdin.unshift(buf);
});
```

```
$ tail -n +50000 /usr/share/dict/american-english | head -n10 | node lines.js
'hearties'
'heartiest'
'heartily'
'heartiness'
'heartiness\'s'
'heartland'
'heartland\'s'
'heartlands'
'heartless'
'heartlessly'
```

Код выше приведен только для примера, если вам действительно нужно будет разбить строку - лучше будет воспользователься специализированым модулем [split](https://npmjs.org/package/split) и не изобретать велосипед.

## Потоки на запись

В поток на запись можно послать данные используя `.pipe()`, но прочитать их уже не получится:

``` js
src.pipe(writableStream)
```

### Создание потока на запись

Просто определяем методо `._write(chunk, enc, next)`, и теперь в наш поток можно передавать данные:

``` js
var Writable = require('stream').Writable;
var ws = Writable();
ws._write = function (chunk, enc, next) {
    console.dir(chunk);
    next();
};

process.stdin.pipe(ws);
```

```
$ (echo beep; sleep 1; echo boop) | node write0.js
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
```

Первый аргумент, `chunk`, это данные которые посылает отправитель.

Второй аргумент, `enc`, это строка с названием кодировки. Она используется только в случае когда опция `opts.decodeString` установлена в `false`, и вы отправляете строку.

Третий аргумент, `next(err)`, является калбеком, сообщающим отправителю что можно послать еще данные. Если вы вызовите его с параметром `err`, в потоке будет создано событие `'error'`.

В случае если поток из которого вы читаете передает строки, они будут преобразовываться в `Buffer`. Чтобы отключить преобразование - создайте поток на запись с соответствующим параметром: `Writable({ decodeStrings: false })`.

Если поток на чтение передает объекты - явно укажите это в параметрах: `Writable({ objectMode: true })`.

### Отправка данных в поток на запись

Чтобы передать данные в поток на запись - вызовите `.write(data)`, где `data` это наобр данных которые вы хотите записать.

``` js
process.stdout.write('beep boop\n');
```

Если вы хотите сообщить что вы закончили запись - вызовите `.end()` (или `.end(data)` чтобы отправить еще немного данных перед завершением):

``` js
var fs = require('fs');
var ws = fs.createWriteStream('message.txt');

ws.write('beep ');

setTimeout(function () {
    ws.end('boop\n');
}, 1000);
```

```
$ node writing1.js
$ cat message.txt
beep boop
```

Не беспокойтесь о синхронизации данных и буферизации, `.write()` вернет `false` если в буфере скопилось данных больше чем указывалось в параметре `opts.highWaterMark` при создании потока. В этом случае следует подождать события `'drain'`, которое сигнализирует о том что данные можно снова писать.

## Трансформирующие потоки (transform)

Трансформирующие потоки это частный случай дуплексных потоков (в обоих случаех они могут использоваться как для записи, так и чтения). Разница в том, что в случае трансформации отдаваемые данные так или иначе зависят от того что подается на вход.

Возможно, вы также встречали второе название таких потоков - "сквозные" ("through streams"). В любом случае, это просто фильтры которые преобразовывают входящие данные и отдают их.

## Дуплексные потоки (duplex)

Дуплексные потоки наследуют методы как от потоков на чтение, так и от потоков на запись. Это позволяет им действовать в обоих направлениях - читать данные, и записывать их в обе стороны. В качестве аналогии можно привести телефон. Если вам требуется сделать что-нибудь типа такого:

``` js
a.pipe(b).pipe(a)
```

значит вам нужен дуплексный поток.

## Классические потоки

В первых версиях node существовал "классический" интерфейс потоков. Его потом переработали, и в текущем виде практически нигде не используют. Однако, возможно вы встретитесь с ним при поддержке старых версий программ, поэтому мы разберем как он работает.

Когда поток регистририрует функцию-слушателя события `"data"`, он переключается в `"classic"` режим, и начинает вести себя в соответствии со старым API.

### Классические потоки на чтение

Думайте о классических потоках на чтения как об обычных эмиттерах событий, которые создают событие `"data"` каждый раз когда появляются данные для получателей, и событие `"end"` когда данные заканчиваются.

Метод `.pipe()` узнает является ли "классический" поток потоком на чтение, проверяя правдивость свойства `stream.readable`.

К примеру, вот простейший поток на чтение который печатает буквы от `A` до `J`:

``` js
var Stream = require('stream');
var stream = new Stream;
stream.readable = true;

var c = 64;
var iv = setInterval(function () {
    if (++c >= 75) {
        clearInterval(iv);
        stream.emit('end');
    }
    else stream.emit('data', String.fromCharCode(c));
}, 100);

stream.pipe(process.stdout);
```

```
$ node classic0.js
ABCDEFGHIJ
```

Для чтения из классическового потока, надо зарегистрировать функции-слушатели на события `"data"` и `"end"`. Пример чтения данных из `process.stdin`  используя "классический" стиль:

``` js
process.stdin.on('data', function (buf) {
    console.log(buf);
});
process.stdin.on('end', function () {
    console.log('__END__');
});
```

```
$ (echo beep; sleep 1; echo boop) | node classic1.js
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

Запомните, что всегда когда вы регистрируете слушателя на событие `"data"` - в целях совместимости он превращается "классический", и вы теряете все преимущества нового API потоков. Поэтому, старайтесь никогда не регистрировать обработчики на `"data"` и `"end"` самостоятельно. Для взаимодействия со старыми версиями потоков, вместо этого, там где возможно используйте специализированные библиотеки которые возьмут все вопросы совместимости на себя.

К примеру, чтобы избежать установки слушателей `"data"` и `"end"` подойдет модуль [through](https://npmjs.org/package/through):

``` js
var through = require('through');
process.stdin.pipe(through(write, end));

function write (buf) {
    console.log(buf);
}
function end () {
    console.log('__END__');
}
```

```
$ (echo beep; sleep 1; echo boop) | node through.js
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

а для буферизации всего содержимого потока сойдет [concat-stream](https://npmjs.org/package/concat-stream):

``` js
var concat = require('concat-stream');
process.stdin.pipe(concat(function (body) {
    console.log(JSON.parse(body));
}));
```

```
$ echo '{"beep":"boop"}' | node concat.js
{ beep: 'boop' }
```

У классических потоков на чтение для остановки и продолжения есть методы `.pause()` и `.resume()`, но их использования следует избегать. Если вам необходим этот фуникционал - рекомендуется не создавать логику самостоятельно, а использовать модуль [through](https://npmjs.org/package/through).

### Классические потоки на запись

Классические потоки на запись очень просты. Просто опишите методы `.write(buf)`, `.end(buf)`
и `.destroy()`.

В основном разработчики под node ожидают что метод  `stream.end(buf)` будет эмулировать поведение `stream.write(buf); stream.end()`, и нам не следует их разочаровывать.

## Узнать больше

Вы прочитали про базовые понятия касающиеся потоков, если вы хотите узнать больше - обратитесь к актуальной [документация по потокам](http://nodejs.org/docs/latest/api/stream.html#stream_stream). В случае если вам понадобится сделать API2 потоков совместимым с "классическим" API - используйте модуль [readable-stream](https://npmjs.org/package/readable-stream). Просто подключите его в свой проект: `require('readable-stream')` вместо `require('stream')`.

***

# Встроеные потоки

Эти потоки поставляются с node и могут быть использованы без дополнительных библиотек.

## process

### [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin)

Поток на чтение содержит стандартный системный поток ввода для вашей программы.

По умолчанию он находится в режиме паузы, но после первого вызова `.resume()` он начнет исполняться в
[следующем системном тике](http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback).

Если process.stdin указывает на терминал (проверяется вызовом
[`tty.isatty()`](http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd)), тогда входящие данные будут буферизироваться построчно. Вы можете выключить построчную буферизацию вызвав `process.stdin.setRawMode(true)`. Однако, имейте ввиду что в этом случае обработчики системных нажатий (таких как`^C` и `^D`) будут удалены.

### [process.stdout](http://nodejs.org/api/process.html#process_process_stdout)

Поток на запись, содержащий стандартный системный вывод для вашей программы. Посылайте туда данные, если вам нужно передать их в stdout.

### [process.stderr](http://nodejs.org/api/process.html#process_process_stderr)

Поток на запись, содержащий стандартный системный вывод ошибок для вашей программы. Посылайте туда данные, если вам нужно передать их в stderr.

## [child_process.spawn()](https://nodejs.org/api/child_process.html)

Данная функция запускает процесс, и возвращает объект содержащий stderr/stdin/stdout потоки данного процесса.

### [fs.createReadStream()](https://nodejs.org/api/fs.html#fs_fs_createreadstream_path_options)

Поток на чтение, содержащий указанный файл. Используйте, если вам надо прочесть большой файл без больших затрат ресурсов.

### [fs.createWriteStream()](https://nodejs.org/api/fs.html#fs_fs_createwritestream_path_options)

Поток на запись, позволяющий сохранить переданные данные в файл.

## net

### [net.connect()](http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener)

Данная функция вернет дуплексный поток, который позволяет подключиться к удаленному хосту по протоколу tcp.

Все данные которые вы будете в него записывать будут буферизироваться до тех пор, пока не возникнет событие `'connect'`.

### [net.createServer()](https://nodejs.org/api/net.html#net_net_createserver_options_connectionlistener)

Создает сервер для обработки входящих соединений. Параметром передается калбек, который вызывается при создании соединения, и содержит поток на запись.

``` js
const net = require('net');
const server = net.createServer((c) => {
  // 'connection' listener
  console.log('client connected');
  c.on('end', () => {
    console.log('client disconnected');
  });
  c.write('hello\r\n');
  c.pipe(c);
});
server.on('error', (err) => {
  throw err;
});
server.listen(8124, () => {
  console.log('server bound');
});
```

### [http.request()](https://nodejs.org/api/http.html#http_http_request_options_callback)

Создает поток на чтение, позволяющий сделать запрос к веб-серверу и вернуть результат.

### [http.createServer()](https://nodejs.org/api/http.html#http_http_createserver_requestlistener)

Создает сервер для обработки входящих веб-запросов. Параметром передается калбек, который вызывается при создании соединения, и содержит поток на запись.

### [zlib.createGzip()](https://nodejs.org/api/zlib.html#zlib_zlib_creategzip_options)

Трансформирующий поток, который отдает на выходе запакованый gzip.

### [zlib.createGunzip()](https://nodejs.org/api/zlib.html#zlib_zlib_creategunzip_options)

Трансформирующий поток, распаковывает gzip-поток.

### [zlib.createDeflate()](https://nodejs.org/api/zlib.html#zlib_zlib_createdeflate_options)

### [zlib.createInflate()](https://nodejs.org/api/zlib.html#zlib_zlib_createinflate_options)

***

# Потоки для управления потоками

## [through](https://github.com/dominictarr/through)

Простой способ создания дуплексного потока или конвертации "классического" в современный.

## [from](https://github.com/dominictarr/from)

Аналог through, только для создания потока для чтения.

## [pause-stream](https://github.com/dominictarr/pause-stream)

Позволяет буферизировать поток и получать результат буфера в произвольный момент.

```js
var ps = require('pause-stream')();

badlyBehavedStream.pipe(ps.pause())

aLittleLater(function (err, data) {
  ps.pipe(createAnotherStream(data))
  ps.resume()
})
```

## [concat-stream](https://github.com/maxogden/node-concat-stream)

Буферизирует поток в один общий буфер. `concat(cb)` принимает параметром только один аргумент - функцию `cb(body)`, которая вернет `body` когда поток завершится.

К примеру, в этой программа модуль concat-stream вернет строку `"beep boop"` только после того как вызовется `cs.end()`. Результат работы программы - перевод строки в верхний регистр.

``` js
var concat = require('concat-stream');

var cs = concat(function (body) {
    console.log(body.toUpperCase());
});
cs.write('beep ');
cs.write('boop.');
cs.end();
```

```
$ node concat.js
BEEP BOOP.
```

Следующий пример обработает строку с параметрами, и вернет их уже в JSON:

``` js
var http = require('http');
var qs = require('querystring');
var concat = require('concat-stream');

var server = http.createServer(function (req, res) {
    req.pipe(concat(function (body) {
        var params = qs.parse(body.toString());
        res.end(JSON.stringify(params) + '\n');
    }));
});
server.listen(5005);
```

```
$ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
{"beep":"boop","dinosaur":"trex"}
```

## [duplex](https://github.com/dominictarr/duplex) [duplexer](https://github.com/Raynos/duplexer)

Создание дуплексного потока.

## [emit-stream](https://github.com/substack/emit-stream)

Конвертирует события (event-emitter) в поток, и обратно.

В данном примере будет создан сервер который автоматически будет отправлять все события клиенту:

```js
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');
var net = require('net');

var server = (function () {
    var ev = createEmitter();

    return net.createServer(function (stream) {
        emitStream(ev)
            .pipe(JSONStream.stringify())
            .pipe(stream)
        ;
    });
})();
server.listen(5555);

var EventEmitter = require('events').EventEmitter;

function createEmitter () {
    var ev = new EventEmitter;
    setInterval(function () {
        ev.emit('ping', Date.now());
    }, 2000);

    var x = 0;
    setInterval(function () {
        ev.emit('x', x ++);
    }, 500);

    return ev;
}
```

Клиент, со своей стороны, может автоматически конвертировать приходящие данные обратно в события:

```js
var emitStream = require('emit-stream');
var net = require('net');

var stream = net.connect(5555)
    .pipe(JSONStream.parse([true]))
;
var ev = emitStream(stream);

ev.on('ping', function (t) {
    console.log('# ping: ' + t);
});

ev.on('x', function (x) {
    console.log('x = ' + x);
});
```

## [invert-stream](https://github.com/dominictarr/invert-stream)

Создает из двух потоков один, "соединяя" вход первого потока с выходом второго и наоборот.

Данная программа создаст из stdin и stdout дуплексный поток:

```js
var spawn = require('child_process').spawn
  var invert = require('invert-stream')

  var ch = spawn(cmd, args)
  var inverted = invert()

  ch.stdout.pipe(inverted.other).pipe(ch.sdin)

  //now, we have just ONE stream: inverted

  //write to che ch's stdin
  inverted.write(data)

  //read from ch's stdout
  inverted.on('data', console.log)
```

## [map-stream](https://github.com/dominictarr/map-stream)

Создает трансформирующий поток для заданой асинхронной функции.

```js
var map = require('map-stream')

map(function (data, callback) {
  //transform data
  // ...
  callback(null, data)
})
```

## [remote-events](https://github.com/dominictarr/remote-events)

Позволяет объединять несколько эмиттеров событий в единый поток.

## [buffer-stream](https://github.com/Raynos/buffer-stream)

Дуплексный поток, буферизирующий запись в него.

## [event-stream](https://github.com/dominictarr/event-stream)

Управление асинхронным кодом с использованием потоков. _(примечание переводчика: в случае если вы хотите использовать мощь потоков для синхронизации своего кода - обратите внимание на активно развивающуюся библиотеку [highland](https://github.com/caolan/highland) от автора async.js)_

## [auth-stream](https://github.com/Raynos/auth-stream)

Добавление слоя авторизации для доступа к потокам.

***

# Мультифункциональные потоки

## [mux-demux](https://github.com/dominictarr/mux-demux)

Создание мультифункциональных потоков на основе любых текстовых. В данном примере используется поток, работающий с датами:

```js
var MuxDemux = require('mux-demux')
var net = require('net')

net.createServer(function (con) {
  con.pipe(MuxDemux(function (stream) {
    stream.on('data', console.log.bind(console))
  })).pipe(con)
}).listen(8642, function () {
  var con = net.connect(8642), mx
  con.pipe(mx = MuxDemux()).pipe(con)

  var ds = mx.createWriteStream('times')

  setInterval(function () {
    ds.write(new Date().toString())
  }, 1e3)
})
```

## [stream-router](https://github.com/Raynos/stream-router)

Роутер для потоков, созданых с помощью `mux-demux`.

## [multi-channel-mdm](https://github.com/Raynos/multi-channel-mdm)

Создание постоянных потоков (каналов) из потоков `mux-demux`.

***

# Потоки с сохранением состояний

Данная коллекция потоков предполагает, что операции над данными всегда возвращают один и тот же результат вне зависимости от порядка этих операций.

## [crdt](https://github.com/dominictarr/crdt)

## [delta-stream](https://github.com/Raynos/delta-stream)

## [scuttlebutt](https://github.com/dominictarr/scuttlebutt)

К примеру, `scuttlebutt` может быть использован для синхронизации состояния между узлами mesh-сети, где узлы непосредственно не связаны между собой и нет единого мастера. Пример: распределенный торрент-клиент.

Под капотом `scuttlebutt` для обмена сообщений используется широко известный протокол [gossip](https://en.wikipedia.org/wiki/Gossip_protocol), который гарантирует что все узлы будут возвращать [последнее актуальное значение](https://ru.wikipedia.org/wiki/Консистентность_в_конечном_счёте).

Используя интерфейс `scuttlebutt/model`, мы можем создавать клиентов и связывать их между собой:

``` js
var Model = require('scuttlebutt/model');
var am = new Model;
var as = am.createStream();

var bm = new Model;
var bs = bm.createStream();

var cm = new Model;
var cs = cm.createStream();

var dm = new Model;
var ds = dm.createStream();

var em = new Model;
var es = em.createStream();

as.pipe(bs).pipe(as);
bs.pipe(cs).pipe(bs);
bs.pipe(ds).pipe(bs);
ds.pipe(es).pipe(ds);

em.on('update', function (key, value, source) {
    console.log(key + ' => ' + value + ' from ' + source);
});

am.set('x', 555);
```

Мы создали сеть в форме ненаправленного графа, которая выглядит так:

```
a <-> b <-> c
      ^
      |
      v
      d <-> e
```

Узлы `a` и `e` напрямую не соединены, но если мы выполним команду:

```
$ node model.js
x => 555 from 1347857300518
```

то увидим что узел  `a` будет доступен узлу `e` через узлы `b`и `d`. Учитывая то, что `scuttlebutt` использует простой потоковый интерфейс, и все узлы гарантированно получат данные - мы можем соединить любой процесс, сервер или транспорт которые поддерживают обработку строк.

Давайте создадим более реалистичный пример. В нем мы будем соединяться через сеть, и увеличивать счетчик каждые 320 милисекунд на всех узлах:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
m.set('count', '0');
m.on('update', function (key, value) {
    console.log(key + ' = ' + m.get('count'));
});

var server = net.createServer(function (stream) {
    stream.pipe(m.createStream()).pipe(stream);
});
server.listen(8888);

setInterval(function () {
    m.set('count', Number(m.get('count')) + 1);
}, 320);
```

Теперь созданим клиента, который подключается к серверу, получает обновления и выводит их на экран:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
var s = m.createStream();

s.pipe(net.connect(8888, 'localhost')).pipe(s);

m.on('update', function cb (key) {
    // wait until we've gotten at least one count value from the network
    if (key !== 'count') return;
    m.removeListener('update', cb);

    setInterval(function () {
        m.set('count', Number(m.get('count')) + 1);
    }, 100);
});

m.on('update', function (key, value) {
    console.log(key + ' = ' + value);
});
```

Клиент получился чуть-чуть сложнее, так как ему приходится ждать обновления от остальных участников прежде чем убедиться что он может увеличить счетчик.

После того как мы запустим сервер и несколько клиентов - мы увидим изменения счетчика наподобии такого:

```
count = 183
count = 184
count = 185
count = 186
count = 187
count = 188
count = 189
```

Время от времени на некоторых узлах мы будем замечать что значения повторяются:

```
count = 147
count = 148
count = 149
count = 149
count = 150
count = 151
```

Это происходит потому, что мы не предоставили достаточно данных алгоритму для разрешения временных конфликтов, и ему сложнее поддерживать синхронизацию всех узлов. Вся эта магия выходит за пределы данной статьи, поэтому рекомендуем самостоятельно изучить `scuttlebutt`.

Заметьте, в вышеприведенных примерах сервер это всего лишь еще один узел с теми же привилегиями что и остальные клиенты. Понятия "клиент" и "сервер" не затрагивают способы синхронизации данных, в данном сервер это "тот кто первым создал соединение". Подобные протоколы называют "симметричными", еще один пример подобного протокола можно посмотреть в реализации модуля [dnode](https://github.com/substack/dnode).

***

# http-потоки

Модули из данной категории предоставляют интерфейс работы с http response/request потоками.

## [request](https://github.com/mikeal/request)

## [oppressor](https://github.com/substack/oppressor)

## [response-stream](https://github.com/substack/response-stream)

***

# Потоки ввода-вывода

## [reconnect-core](https://github.com/juliangruber/reconnect-core)

Базовый настраиваемый интерфейс для переподключения потоков при возникновении проблем в сети.

## [kv](https://github.com/dominictarr/kv)

Абстрактный поток, предоставляющий враппер для доступа к различным key-value хранилищам.

***

# Потоки-анализаторы

## [trumpet](https://github.com/substack/node-trumpet)

Трансформация html-текста с использованием css-селекторов.

## [JSONStream](https://github.com/dominictarr/JSONStream)

Стриминг `JSON.parse` и `JSON.stringify`. Примеры использования - обработка большого объема JSON-данных при недостаточном количестве оперативной памяти, обработка json "на лету" при получении его через медленные каналы, и т.п.

***

# Потоки для браузера

## [shoe](https://github.com/substack/shoe)

Стриминг вебсокет событий.

## [graph-stream](https://github.com/substack/graph-stream)

Отрисовывает график в браузее по мере прихода новых данных.

***

# rpc streams

## [dnode](https://github.com/substack/dnode)

Данный модуль дает вам возможность вызывать удаленные функции (RPC) через любой поток.

Для примера, создадим простой сервер `dnode`:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

потом напишем клиента, который вызывает метод сервера `.transform()`:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

После запуска, клиент выведет следующий текст:

```
$ node client.js
beep => BOOP
```

Клиент послал `'beep'` на сервер, запросив выполнение метода `.transform()`, сервер вернул результат.

Интерфейс, который предоставляет `dnode`, является дуплексным потоком. Таким образом, так как и клиент и сервер подключены друг к другу (`c.pipe(d).pipe(c)`), запросы можно выполнять в обе стороны.

`dnode` раскрывает себя во всей красе когда вы начинаете передавать аргументы к предоставленым методам. Посмотрим на обновленную версию предыдущего сервера:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(function (n, fn) {
                var oo = Array(n+1).join('o');
                fn(s.replace(/[aeiou]{2,}/, oo).toUpperCase());
            });
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

Вот обновленный клиент:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (cb) {
        cb(10, function (s) {
            console.log('beep:10 => ' + s);
            d.end();
        });
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

После запуска клиента, мы увидим:

```
$ node client.js
beep:10 => BOOOOOOOOOOP
```

Сервер увидел аргумент, и выполнил функцию с ним!

Основная идея такая: вы просто ложите функцию в объект, и на другой стороне земного шара вызываете идентичную функцию с нужными вам аргументами. Вместо того чтобы выполниться локально, данные передаются на сервер и функция возвращает результат удаленного выполнения. Это просто работает.

`dnode` работает через потоки как в node, так и в браузере. Удобно комбинировать потоки через [mux-demux](https://github.com/dominictarr/mux-demux) для создания мультиплексного потока, работающего в обе стороны.


***

# Тестовые потоки

## [tap](https://github.com/isaacs/node-tap)

Фреймворк для тестирования node.js на основе потоков.

## [stream-spec](https://github.com/dominictarr/stream-spec)

Способ описания спецификации потоков, для автоматизации их тестирования.

***

# Примеры мощных комбинаций

## Распределенный устойчивый чат

Модуль [append-only](http://github.com/Raynos/append-only) предоставляет нам иммутабельный массив данных, который вместе с [scuttlebutt](https://github.com/dominictarr/scuttlebutt) позволяет создать событийно-последовательный распределенный чат, который сможет взаимодействовать с остальными узлами и пережить разбиение сети.

## Свой socket.io с блекджеком

Мы можем создать собственное API для генерации событий через websocket с использованием потоков.

Сперва, используем [shoe](http://github.com/substack/shoe) для создания серверного обработчика вебсокетов, и [emit-stream](https://github.com/substack/emit-stream) чтобы превратить эмиттер событий в поток который генерирует объекты.
Далее, поток с объектами мы подключаем к [JSONStream](https://github.com/dominictarr/JSONStream) чтобы сеарелизовать их для передачи через сеть.

``` js
var EventEmitter = require('events').EventEmitter;
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var sock = shoe(function (stream) {
    var ev = new EventEmitter;
    emitStream(ev)
        .pipe(JSONStream.stringify())
        .pipe(stream)
    ;
    ...
});
```

Теперь мы можем прозрачно генерировать события используя метод эмиттера `ev`. К примеру, несколько событий через разные промежутки времени:

``` js
var intervals = [];

intervals.push(setInterval(function () {
    ev.emit('upper', 'abc');
}, 500));

intervals.push(setInterval(function () {
    ev.emit('lower', 'def');
}, 300));

stream.on('end', function () {
    intervals.forEach(clearInterval);
});
```

Наконец, экземпляр `shoe` привяжем к http-серверу:

``` js
var http = require('http');
var server = http.createServer(require('ecstatic')(__dirname));
server.listen(8080);

sock.install(server, '/sock');
```

Между тем, на стороне браузера поток от `shoe` содержащий json обрабатывается и получившиеся объекты передаются в `eventStream()`. Таким образом, `eventStream()` возвращает эмиттер который генерирует переданные сервером события:

``` js
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var parser = JSONStream.parse([true]);
var stream = parser.pipe(shoe('/sock')).pipe(parser);
var ev = emitStream(stream);

ev.on('lower', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toLowerCase();
    document.body.appendChild(div);
});

ev.on('upper', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toUpperCase();
    document.body.appendChild(div);
});
```

Используем [browserify](https://github.com/substack/node-browserify) для генерации кода в браузере, чтобы мы могли делать `require()` прямо в файле:

```
$ browserify main.js -o bundle.js
```

Подключаем `<script src="/bundle.js"></script>` в html-страницу и открываем ее в браузере, наслаждаемся серверными событиями которые отображаются в браузере.

Начав использовать потокозависимый подход к разработке программ, вы заметите что стали больше полагаться на маленькие переиспользуемые компоненты которым не нужно ничего кроме общего интерфейса потоков. Вместо маршрутизации сообщений через глобальную систему событий и настройки обработчиков, вы сфокусируетесь на разбитии приложения на мелкие компоненты, хорошо выполняющими какую-то одну задачу.

В примере выше вы можете легко заменить `JSONStream` на [stream-serializer](https://github.com/dominictarr/stream-serializer) чтобы получить немного другой вид сериализации. Вы можете добавить дополнительный слой чтобы обрабатывать потери связи с помощью [reconnections](https://github.com/dominictarr/reconnect). Если вы захотите использовать события с областью видимости - вы вставите дополнительный поток с поддержкой [eventemitter2](https://npmjs.org/package/eventemitter2). В случае если вам потребуется изменить поведение некоторых частей потока вы сможете пропустить его через `mux-demux` и разделить на отдельные каналы каждый со своей логикой.

С течением времени, при изменении требований к приложению, вы легко сможете заменять устаревшие компоненты новыми, с гораздо меньшим риском получить в результате неработающую систему.
