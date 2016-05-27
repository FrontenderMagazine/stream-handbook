<article class="markdown-body entry-content">
## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][1]

This document covers the basics of how to write [node.js][2] programs with 
[streams][3].  
You also could read a **[chinese edition][4]**

![cc-by-3.0][5]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][6]

You can install this handbook with npm. Just do:

    npm install -g stream-handbook
    

Now you will have a `stream-handbook` command that will open this readme file
in your`$PAGER`. Otherwise, you may continue reading this document as you are
presently doing.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][7]

    "We should have some ways of connecting programs like garden hose--screw in
    another segment when it becomes necessary to massage data in
    another way. This is the way of IO also."
    

[Doug McIlroy. October 11, 1964][8]

![doug mcilroy][9]

Streams come to us from the [earliest days of unix][10] and have proven
themselves over the decades as a dependable way to compose large systems out of 
small components that[do one thing well][11]. In unix, streams are implemented
by the shell with`|` pipes. In node, the built-in [stream module][3] is used by
the core libraries and can also be used by user-space modules. Similar to unix, 
the node stream module's primary composition operator is called`.pipe()` and
you get a backpressure mechanism for free to throttle writes for slow consumers.

Streams can help to [separate your concerns][12] because they restrict the
implementation surface area into a consistent interface that can be[reused][13]
[use libraries][14] that operate abstractly on streams to institute higher-
level flow control.

Streams are an important component of [small-program design][15] and 
[unix philosophy][11] but there are many other important abstractions worth
considering. Just remember that[technical debt][16] is the enemy and to seek
the best abstractions for the problem at hand.

![brian kernighan][17]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][18]

I/O in node is asynchronous, so interacting with the disk and network involves
passing callbacks to functions. You might be tempted to write code that serves 
up a file from disk like this:

    var http = require('http');
    var fs = require('fs');
    
    var server = http.createServer(function (req, res) {
        fs.readFile(__dirname + '/data.txt', function (err, data) {
            res.end(data);
        });
    });
    server.listen(8000);

This code works but it's bulky and buffers up the entire `data.txt` file into
memory for every request before writing the result back to clients. If
`data.txt` is very large, your program could start eating a lot of memory as it
serves lots of users concurrently, particularly for users on slow connections.

The user experience is poor too because users will need to wait for the whole
file to be buffered into memory on your server before they can start receiving 
any contents.

Luckily both of the `(req, res)` arguments are streams, which means we can
write this in a much better way using`fs.createReadStream()` instead of 
`fs.readFile()`:

    var http = require('http');
    var fs = require('fs');
    
    var server = http.createServer(function (req, res) {
        var stream = fs.createReadStream(__dirname + '/data.txt');
        stream.pipe(res);
    });
    server.listen(8000);

Here `.pipe()` takes care of listening for `'data'` and `'end'` events from the
`fs.createReadStream()`. This code is not only cleaner, but now the `data.txt`
file will be written to clients one chunk at a time immediately as they are 
received from the disk.

Using `.pipe()` has other benefits too, like handling backpressure
automatically so that node won't buffer chunks into memory needlessly when the 
remote client is on a really slow or high-latency connection.

Want compression? There are streaming modules for that too!

    var http = require('http');
    var fs = require('fs');
    var oppressor = require('oppressor');
    
    var server = http.createServer(function (req, res) {
        var stream = fs.createReadStream(__dirname + '/data.txt');
        stream.pipe(oppressor(req)).pipe(res);
    });
    server.listen(8000);

Now our file is compressed for browsers that support gzip or deflate! We can
just let[oppressor][19] handle all that content-encoding stuff.

Once you learn the stream api, you can just snap together these streaming
modules like lego bricks or garden hoses instead of having to remember how to 
push data through wonky non-streaming custom APIs.

Streams make programming in node simple, elegant, and composable.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][20]

There are 5 kinds of streams: readable, writable, transform, duplex, and "
classic
".

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][21]

All the different types of streams use `.pipe()` to pair inputs with outputs.

`.pipe()` is just a function that takes a readable source stream `src` and
hooks the output to a destination writable stream`dst`:

    src.pipe(dst)
    

`.pipe(dst)` returns `dst` so that you can chain together multiple `.pipe()`
calls together:

    a.pipe(b).pipe(c).pipe(d)

which is the same as:

    a.pipe(b);
    b.pipe(c);
    c.pipe(d);

This is very much like what you might do on the command-line to pipe programs
together:

    a | b | c | d
    

except in node instead of the shell!

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][22]

Readable streams produce data that can be fed into a writable, transform, or
duplex stream by calling`.pipe()`:

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][23]

Let's make a readable stream!

    var Readable = require('stream').Readable;
    
    var rs = new Readable;
    rs.push('beep ');
    rs.push('boop\n');
    rs.push(null);
    
    rs.pipe(process.stdout);

    $ node read0.js
    beep boop
    

`rs.push(null)` tells the consumer that `rs` is done outputting data.

Note here that we pushed content to the readable stream `rs` before piping to
`process.stdout`, but the complete message was still written.

This is because when you `.push()` to a readable stream, the chunks you push
are buffered until a consumer is ready to read them.

However, it would be even better in many circumstances if we could avoid
buffering data altogether and only generate the data when the consumer asks for 
it.

We can push chunks on-demand by defining a `._read` function:

    var Readable = require('stream').Readable;
    var rs = Readable();
    
    var c = 97;
    rs._read = function () {
        rs.push(String.fromCharCode(c++));
        if (c > 'z'.charCodeAt()) rs.push(null);
    };
    
    rs.pipe(process.stdout);

    $ node read1.js
    abcdefghijklmnopqrstuvwxyz
    

Here we push the letters `'a'` through `'z'`, inclusive, but only when the
consumer is ready to read them.

The `_read` function will also get a provisional `size` parameter as its first
argument that specifies how many bytes the consumer wants to read, but your 
readable stream can ignore the`size` if it wants.

Note that you can also use `util.inherits()` to subclass a Readable stream, but
that approach doesn't lend itself very well to comprehensible examples.

To show that our `_read` function is only being called when the consumer
requests, we can modify our readable stream code slightly to add a delay:

    var Readable = require('stream').Readable;
    var rs = Readable();
    
    var c = 97 - 1;
    
    rs._read = function () {
        if (c >= 'z'.charCodeAt()) return rs.push(null);
    
        setTimeout(function () {
            rs.push(String.fromCharCode(++c));
        }, 100);
    };
    
    rs.pipe(process.stdout);
    
    process.on('exit', function () {
        console.error('\n_read() called ' + (c - 97) + ' times');
    });
    process.stdout.on('error', process.exit);

Running this program we can see that `_read()` is only called 5 times when we
only request 5 bytes of output:

    $ node read2.js | head -c5
    abcde
    _read() called 5 times
    

The setTimeout delay is necessary because the operating system requires some
time to send us the relevant signals to close the pipe.

The `process.stdout.on('error', fn)` handler is also necessary because the
operating system will send a SIGPIPE to our process when`head` is no longer
interested in our program's output, which gets emitted as an EPIPE error on
`process.stdout`.

These extra complications are necessary when interfacing with the external
operating system pipes but are automatic when we interface directly with node 
streams the whole time.

If you want to create a readable stream that pushes arbitrary values instead of
just strings and buffers, make sure to create your readable stream with
`Readable({ objectMode: true })`.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][24]

Most of the time it's much easier to just pipe a readable stream into another
kind of stream or a stream created with a module like[through][25] or 
[concat-stream][26], but occasionally it might be useful to consume a readable
stream directly.

    process.stdin.on('readable', function () {
        var buf = process.stdin.read();
        console.dir(buf);
    });

    $ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js 
    <Buffer 61 62 63 0a>
    <Buffer 64 65 66 0a>
    <Buffer 67 68 69 0a>
    null
    

When data is available, the `'readable'` event fires and you can call `.read()`

When the stream is finished, `.read()` returns `null` because there are no more
bytes to fetch.

You can also tell `.read(n)` to return `n` bytes of data. Reading a number of
bytes is merely advisory and does not work for object streams, but all of the 
core streams support it.

Here's an example of using `.read(n)` to buffer stdin into 3-byte chunks:

    process.stdin.on('readable', function () {
        var buf = process.stdin.read(3);
        console.dir(buf);
    });

Running this example gives us incomplete data!

    $ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js 
    <Buffer 61 62 63>
    <Buffer 0a 64 65>
    <Buffer 66 0a 67>
    

This is because there is extra data left in internal buffers and we need to
give node a "kick" to tell it that we are interested in more data past the 3 
bytes that we've already read. A simple`.read(0)` will do this:

    process.stdin.on('readable', function () {
        var buf = process.stdin.read(3);
        console.dir(buf);
        process.stdin.read();
    });

Now our code works as expected in 3-byte chunks!

    $ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js 
    <Buffer 61 62 63>
    <Buffer 0a 64 65>
    <Buffer 66 0a 67>
    <Buffer 68 69 0a>

You can also use `.unshift()` to put back data so that the same read logic will
fire when`.read()` gives you more data than you wanted.

Using `.unshift()` prevents us from making unnecessary buffer copies. Here we
can build a readable parser to split on newlines:

    var offset = ;
    
    process.stdin.on('readable', function () {
        var buf = process.stdin.read();
        if (!buf) return;
        for (; offset < buf.length; offset++) {
            if (buf[offset] === 0x0a) {
                console.dir(buf.slice(, offset).toString());
                buf = buf.slice(offset + 1);
                offset = ;
                process.stdin.unshift(buf);
                return;
            }
        }
        process.stdin.unshift(buf);
    });

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
    

However, there are modules on npm such as [split][27] that you should use
instead of rolling your own line-parsing logic.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][28]

A writable stream is a stream you can `.pipe()` to but not from:

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][29]

Just define a `._write(chunk, enc, next)` function and then you can pipe a
readable stream in:

    var Writable = require('stream').Writable;
    var ws = Writable();
    ws._write = function (chunk, enc, next) {
        console.dir(chunk);
        next();
    };
    
    process.stdin.pipe(ws);

    $ (echo beep; sleep 1; echo boop) | node write0.js 
    <Buffer 62 65 65 70 0a>
    <Buffer 62 6f 6f 70 0a>
    

The first argument, `chunk` is the data that is written by the producer.

The second argument `enc` is a string with the string encoding, but only when
`opts.decodeString` is `false` and you've been written a string.

The third argument, `next(err)` is the callback that tells the consumer that
they can write more data. You can optionally pass an error object`err`, which
emits an`'error'` event on the stream instance.

If the readable stream you're piping from writes strings, they will be
converted into`Buffer`s unless you create your writable stream with 
`Writable({ decodeStrings: false })`.

If the readable stream you're piping from writes objects, create your writable
stream with`Writable({ objectMode: true })`.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][30]

To write to a writable stream, just call `.write(data)` with the `data` you
want to write!

    process.stdout.write('beep boop\n');

To tell the destination writable stream that you're done writing, just call 
`.end()`. You can also give `.end(data)` some `data` to write before ending:

    var fs = require('fs');
    var ws = fs.createWriteStream('message.txt');
    
    ws.write('beep ');
    
    setTimeout(function () {
        ws.end('boop\n');
    }, 1000);

    $ node writing1.js 
    $ cat message.txt
    beep boop
    

If you care about high water marks and buffering, `.write()` returns false when
there is more data than the`opts.highWaterMark` option passed to `Writable()`
in the incoming buffer.

If you want to wait for the buffer to empty again, listen for a `'drain'` event
.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][31]

Transform streams are a certain type of duplex stream (both readable and
writable). The distinction is that in Transform streams, the output is in some 
way calculated from the input.

You might also hear transform streams referred to as "through streams".

Through streams are simple readable/writable filters that transform input and
produce output.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][32]

Duplex streams are readable/writable and both ends of the stream engage in a
two-way interaction, sending back and forth messages like a telephone. An rpc 
exchange is a good example of a duplex stream. Any time you see something like:

you're probably dealing with a duplex stream.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][33]

Classic streams are the old interface that first appeared in node 0.4. You will
probably encounter this style of stream for a long time so it's good to know how
they work.

Whenever a stream has a `"data"` listener registered, it switches into 
`"classic"` mode and behaves according to the old API.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][34]

Classic readable streams are just event emitters that emit `"data"` events when
they have data for their consumers and emit`"end"` events when they are done
producing data for their consumers.

`.pipe()` checks whether a classic stream is readable by checking the
truthiness of`stream.readable`.

Here is a super simple readable stream that prints `A` through `J`, inclusive
:

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

    $ node classic0.js
    ABCDEFGHIJ
    

To read from a classic readable stream, you register `"data"` and `"end"`
listeners. Here's an example reading from`process.stdin` using the old readable
stream style:

    process.stdin.on('data', function (buf) {
        console.log(buf);
    });
    process.stdin.on('end', function () {
        console.log('__END__');
    });

    $ (echo beep; sleep 1; echo boop) | node classic1.js 
    <Buffer 62 65 65 70 0a>
    <Buffer 62 6f 6f 70 0a>
    __END__
    

Note that whenever you register a `"data"` listener, you put the stream into
compatability mode so you lose the benefits of the new streams2 api.

You should pretty much never register `"data"` and `"end"` handlers yourself
anymore. If you need to interact with legacy streams, use libraries that you can
`.pipe()` to instead where possible.

For example, you can use [through][25] to avoid setting up explicit `"data"`
and`"end"` listeners:

    var through = require('through');
    process.stdin.pipe(through(write, end));
    
    function write (buf) {
        console.log(buf);
    }
    function end () {
        console.log('__END__');
    }

    $ (echo beep; sleep 1; echo boop) | node through.js 
    <Buffer 62 65 65 70 0a>
    <Buffer 62 6f 6f 70 0a>
    __END__
    

or use [concat-stream][26] to buffer up an entire stream's contents:

    var concat = require('concat-stream');
    process.stdin.pipe(concat(function (body) {
        console.log(JSON.parse(body));
    }));

    $ echo '{"beep":"boop"}' | node concat.js 
    { beep: 'boop' }
    

Classic readable streams have `.pause()` and `.resume()` logic for
provisionally pausing a stream, but this was merely advisory. If you are going 
to use`.pause()` and `.resume()` with classic readable streams, you should use
[through][25] to handle buffering instead of writing that yourself.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][35]

Classic writable streams are very simple. Just define `.write(buf)`, 
`.end(buf)` and `.destroy()`.

`.end(buf)` may or may not get a `buf`, but node people will expect 
`stream.end(buf)` to mean `stream.write(buf); stream.end()` and you shouldn't
violate their expectations.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][36]

*   [core stream documentation][37]
*   You can use the [readable-stream][38] module to make your streams2 code
    compliant with node 0.8 and below. Just
   `require('readable-stream')` instead of `require('stream')` after you 
    `npm install readable-stream`.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][39]

These streams are built into node itself.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][40]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][41]
[process.stdin][42] 

This readable stream contains the standard system input stream for your program
.

It is paused by default but the first time you refer to it `.resume()` will be
called implicitly on the[next tick][43].

If process.stdin is a tty (check with [`tty.isatty()`][44]) then input events
will be line-buffered. You can turn off line-buffering by calling
`process.stdin.setRawMode(true)` BUT the default handlers for key combinations
such as`^C` and `^D` will be removed.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][45]
[process.stdout][46] 

This writable stream contains the standard system output stream for your
program.

`write` to it if you want to send data to stdout

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][47]
[process.stderr][48] 

This writable stream contains the standard system error stream for your program
.

`write` to it if you want to send data to stderr

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][49]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][50]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][51]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][52]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][53]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][54]
[net.connect()][55] 

This function returns a [duplex stream] that connects over tcp to a remote host
.

You can start writing to the stream right away and the writes will be buffered
until the`'connect'` event fires.

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][56]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][57]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][58]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][59]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][60]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][61]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][62]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][63]

### 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][64]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][65]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][66]
[through][67] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][68]
[from][69] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][70]
[pause-stream][71] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][72]
[concat-stream][73] 

concat-stream will buffer up stream contents into a single buffer. `concat(cb)`
`cb(body)` with the buffered `body` when the stream has finished.

For example, in this program, the concat callback fires with the body string 
`"beep boop"` once `cs.end()` is called. The program takes the body and upper-
cases it, printing`BEEP BOOP.`

    var concat = require('concat-stream');
    
    var cs = concat(function (body) {
        console.log(body.toUpperCase());
    });
    cs.write('beep ');
    cs.write('boop.');
    cs.end();

    $ node concat.js
    BEEP BOOP.
    

Here's an example usage of concat-stream that will parse incoming url-encoded
form data and reply with a stringified JSON version of the form parameters:

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

    $ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
    {"beep":"boop","dinosaur":"trex"}
    

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][74]
[duplex][75] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][76]
[duplexer][77] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][78]
[emit-stream][79] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][80]
[invert-stream][81] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][82]
[map-stream][83] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][84]
[remote-events][85] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][86]
[buffer-stream][87] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][88]
[event-stream][89] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][90]
[auth-stream][91] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][92]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][93]
[mux-demux][94] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][95]
[stream-router][96] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][97]
[multi-channel-mdm][98] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][99]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][100]
[crdt][101] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][102]
[delta-stream][103] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][104]
[scuttlebutt][105] 

[scuttlebutt][105] can be used for peer-to-peer state synchronization with a
mesh topology where nodes might only be connected through intermediaries and 
there is no node with an authoritative version of all the data.

The kind of distributed peer-to-peer network that [scuttlebutt][105] provides
is especially useful when nodes on different sides of network barriers need to 
share and update the same state. An example of this kind of network might be 
browser clients that send messages through an http server to each other and 
backend processes that the browsers can't directly connect to. Another use-case 
might be systems that span internal networks since IPv4 addresses are scarce.

[scuttlebutt][105] uses a [gossip protocol][106] to pass messages between
connected nodes so that state across all the nodes will
[eventually converge][107] on the same value everywhere.

Using the `scuttlebutt/model` interface, we can create some nodes and pipe them
to each other to create whichever sort of network we want:

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

The network we've created is an undirected graph that looks like:

    a <-> b <-> c
          ^
          |
          v
          d <-> e
    

Note that nodes `a` and `e` aren't directly connected, but when we run this
script:

    $ node model.js
    x => 555 from 1347857300518
    

the value that node `a` set finds its way to node `e` by way of nodes `b` and
`d`. Here all the nodes are in the same process but because [scuttlebutt][105]
uses a simple streaming interface, the nodes can be placed on any process or 
server and connected with any streaming transport that can handle string data.

Next we can make a more realistic example that connects over the network and
increments a counter variable.

Here's the server which will set the initial `count` value to 0 and `count ++`
every 320 milliseconds, printing all updates to count:

    var Model = require('scuttlebutt/model');
    var net = require('net');
    
    var m = new Model;
    m.set('count', '');
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

Now we can make a client that connects to this server, updates the count on an
interval, and prints all the updates it receives:

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

The client is slightly trickier since it should wait until it has an update
from somebody else to start updating the counter itself or else its counter 
would be zeroed.

Once we get the server and some clients running we should see a sequence like
this:

    count = 183
    count = 184
    count = 185
    count = 186
    count = 187
    count = 188
    count = 189
    

Occasionally on some of the nodes we might see a sequence with repeated values
like:

    count = 147
    count = 148
    count = 149
    count = 149
    count = 150
    count = 151
    

These values are due to [scuttlebutt's][105] history-based conflict resolution
algorithm which is hard at work ensuring that the state of the system across all
nodes is eventually consistent.

Note that the server in this example is just another node with the same
privledges as the clients connected to it. The terms "client" and "server" here 
don't affect how the state synchronization proceeds, just who initiates the 
connection. Protocols with this property are often called symmetric protocols. 
See[dnode][108] for another example of a symmetric protocol.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][109]
[append-only][110] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][111]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][112]
[request][113] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][114]
[oppressor][19] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][115]
[response-stream][116] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][117]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][118]
[reconnect][119] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][120]
[kv][121] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][122]
[discovery-network][123] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][124]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][125]
[tar][126] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][127]
[trumpet][128] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][129]
[JSONStream][130] 

Use this module to parse and stringify json data from streams.

If you need to pass a large json collection through a slow connection or you
have a json object that will populate slowly this module will let you parse data
incrementally as it arrives.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][131]
[json-scrape][132] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][133]
[stream-serializer][134] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][135]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][136]
[shoe][137] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][138]
[domnode][139] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][140]
[sorta][141] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][142]
[graph-stream][143] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][144]
[arrow-keys][145] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][146]
[attribute][147] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][148]
[data-bind][149] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][150]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][151]
[hyperstream][152] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][153]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][154]
[baudio][155] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][156]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][157]
[dnode][108] 

[dnode][108] lets you call remote functions through any kind of stream.

Here's a basic dnode server:

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

then you can hack up a simple client that calls the server's `.transform()`
function:

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

Fire up the server, then when you run the client you should see:

    $ node client.js
    beep => BOOP
    

The client sent `'beep'` to the server's `transform()` function and the server
called the client's callback with the result, neat!

The streaming interface that dnode provides here is a duplex stream since both
the client and server are piped to each other
(`c.pipe(d).pipe(c)`) with requests and responses coming from both sides.

The craziness of dnode begins when you start to pass function arguments to
stubbed callbacks. Here's an updated version of the previous server with a multi
-stage callback passing dance:

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

Here's the updated client:

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

After we spin up the server, when we run the client now we get:

    $ node client.js
    beep:10 => BOOOOOOOOOOP
    

It just works!™

The basic idea is that you just put functions in objects and you call them from
the other side of a stream and the functions will be stubbed out on the other 
end to do a round-trip back to the side that had the original function in the 
first place. The best thing is that when you pass functions to a stubbed 
function as arguments, those functions get stubbed out on the*other* side!

This approach of stubbing function arguments recursively shall henceforth be
known as the "turtles all the way down" gambit. The return values of any of your
functions will be ignored and only enumerable properties on objects will be sent,
json-style.

It's turtles all the way down!

![turtles all the way][158]

Since dnode works in node or on the browser over any stream it's easy to call
functions defined anywhere and especially useful when paired up with
[mux-demux][94] to multiplex an rpc stream for control alongside some bulk data
streams.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][159]
[rpc-stream][160] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][161]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][162]
[tap][163] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][164]
[stream-spec][165] 

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][166]

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][167]

The [append-only][110] module can give us a convenient append-only array on top
of[scuttlebutt][105] which makes it really easy to write an eventually-
consistent, distributed chat that can replicate with other nodes and survive 
network partitions.

TODO: the rest

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][168]

We can build a socket.io-style event emitter api over streams using some of the
libraries mentioned earlier in this document.

First we can use [shoe][169] to create a new websocket handler server-side and
[emit-stream][79] to turn an event emitter into a stream that emits objects.
The object stream can then be fed into[JSONStream][130] to serialize the
objects and from there the serialized stream can be piped into the remote 
browser.

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

Inside the shoe callback we can emit events to the `ev` function. Here we'll
just emit different kinds of events on intervals:

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

Finally the shoe instance just needs to be bound to an http server:

    var http = require('http');
    var server = http.createServer(require('ecstatic')(__dirname));
    server.listen(8080);
    
    sock.install(server, '/sock');

Meanwhile on the browser side of things just parse the json shoe stream and
pass the resulting object stream to`eventStream()`. `eventStream()` just
returns an event emitter that emits the server-side events:

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

Use [browserify][170] to build this browser source code so that you can 
`require()` all these nifty modules browser-side:

    $ browserify main.js -o bundle.js
    

Then drop a `<script src="/bundle.js"></script>` into some html and
open it up in a browser to see server-side events streamed through to the 
browser side of things.

With this streaming approach you can rely more on tiny reusable components that
only need to know how to talk streams. Instead of routing messages through a 
global event system socket.io-style, you can focus more on breaking up your 
application into tinier units of functionality that can do exactly one thing 
well.

For instance you can trivially swap out JSONStream in this example for 
[stream-serializer][134] to get a different take on serialization with a
different set of tradeoffs. You could bolt layers over top of shoe to handle
[reconnections][119] or heartbeats using simple streaming interfaces. You could
even add a stream into the chain to use namespaced events with
[eventemitter2][171] instead of the EventEmitter in core.

If you want some different streams that act in different ways it would likewise
be pretty simple to run the shoe stream in this example through mux-demux to 
create separate channels for each different kind of stream that you need.

As the requirements of your system evolve over time, you can swap out each of
these streaming pieces as necessary without as many of the all-or-nothing risks 
that more opinionated framework approaches necessarily entail.

## 
[<svg class="octicon octicon-link" height="16" width="16"><path></path></svg>][172]

We can use some streaming modules to reuse the same html rendering logic for
the client and the server! This approach is indexable, SEO-friendly, and gives 
us realtime updates.

Our renderer takes lines of json as input and returns html strings as its
output. Text, the universal interface!

render.js:

    var through = require('through');
    var hyperglue = require('hyperglue');
    var fs = require('fs');
    var html = fs.readFileSync(__dirname + '/static/row.html', 'utf8');
    
    module.exports = function () {
        return through(function (line) {
            try { var row = JSON.parse(line) }
            catch (err) { return this.emit('error', err) }
    
            this.queue(hyperglue(html, {
                '.who': row.who,
                '.message': row.message
            }).outerHTML);
        });
    };

We can use [brfs][173] to inline the `fs.readFileSync()` call for browser code
and[hyperglue][174] to update html based on css selectors. You don't need to
use hyperglue necessarily here; anything that can return a string with html in 
it will work.

The `row.html` used is just a really simple stub thing:

row.html:

    <div class="row">
      <div class="who"></div>
      <div class="message"></div>
    </div>

The server will just use [slice-file][175] to keep everything simple. 
[slice-file][175] is little more than a glorified `tail/tail -f` api but the
interfaces map well to databases with regular results plus a changes feed like 
couchdb.

server.js:

    var http = require('http');
    var fs = require('fs');
    var hyperstream = require('hyperstream');
    var ecstatic = require('ecstatic')(__dirname + '/static');
    
    var sliceFile = require('slice-file');
    var sf = sliceFile(__dirname + '/data.txt');
    
    var render = require('./render');
    
    var server = http.createServer(function (req, res) {
        if (req.url === '/') {
            var hs = hyperstream({
                '#rows': sf.slice(-5).pipe(render())
            });
            hs.pipe(res);
            fs.createReadStream(__dirname + '/static/index.html').pipe(hs);
        }
        else ecstatic(req, res)
    });
    server.listen(8000);
    
    var shoe = require('shoe');
    var sock = shoe(function (stream) {
        sf.follow(-1,).pipe(stream);
    });
    sock.install(server, '/sock');

The first part of the server handles the `/` route and streams the last 5 lines
from`data.txt` into the `#rows` div.

The second part of the server handles realtime updates to `#rows` using 
[shoe][169], a simple streaming websocket polyfill.

Next we can write some simple browser code to populate the realtime updates
from[shoe][169] into the `#rows` div:

    var through = require('through');
    var render = require('./render');
    
    var shoe = require('shoe');
    var stream = shoe('/sock');
    
    var rows = document.querySelector('#rows');
    stream.pipe(render()).pipe(through(function (html) {
        rows.innerHTML += html;
    }));

Just compile with [browserify][176] and [brfs][173]:

    $ browserify -t brfs browser.js > static/bundle.js
    

And that's it! Now we can populate `data.txt` with some silly data:

    $ echo '{"who":"substack","message":"beep boop."}' >> data.txt
    $ echo '{"who":"zoltar","message":"COWER PUNY HUMANS"}' >> data.txt
    

then spin up the server:

    $ node server.js
    

then navigate to `localhost:8000` where we will see our content. If we add some
more content:

    $ echo '{"who":"substack","message":"oh hello."}' >> data.txt
    $ echo '{"who":"zoltar","message":"HEAR ME!"}' >> data.txt
    

then the page updates automatically with the realtime updates, hooray!

We're now using exactly the same rendering logic on both the client and the
server to serve up SEO-friendly, indexable realtime content. Hooray!</article
>

 [1]: https://github.com/substack/stream-handbook#stream-handbook
 [2]: http://nodejs.org/
 [3]: http://nodejs.org/docs/latest/api/stream.html
 [4]: https://github.com/jabez128/stream-handbook
 [5]: img/80x15.png
 [6]: https://github.com/substack/stream-handbook#node-packaged-manuscript
 [7]: https://github.com/substack/stream-handbook#introduction
 [8]: http://cm.bell-labs.com/who/dmr/mdmpipe.html
 [9]: img/mcilroy.png
 [10]: http://www.youtube.com/watch?v=tc4ROCJYbm0
 [11]: http://www.faqs.org/docs/artu/ch01s06.html
 [12]: http://www.c2.com/cgi/wiki?SeparationOfConcerns
 [13]: http://www.faqs.org/docs/artu/ch01s06.html#id2877537
 [14]: http://npmjs.org
 [15]: https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/
 [16]: http://c2.com/cgi/wiki?TechnicalDebt
 [17]: img/kernighan.png
 [18]: https://github.com/substack/stream-handbook#why-you-should-use-streams
 [19]: https://github.com/substack/oppressor
 [20]: https://github.com/substack/stream-handbook#basics
 [21]: https://github.com/substack/stream-handbook#pipe
 [22]: https://github.com/substack/stream-handbook#readable-streams
 [23]: https://github.com/substack/stream-handbook#creating-a-readable-stream
 [24]: https://github.com/substack/stream-handbook#consuming-a-readable-stream
 [25]: https://npmjs.org/package/through
 [26]: https://npmjs.org/package/concat-stream
 [27]: https://npmjs.org/package/split
 [28]: https://github.com/substack/stream-handbook#writable-streams
 [29]: https://github.com/substack/stream-handbook#creating-a-writable-stream
 [30]: https://github.com/substack/stream-handbook#writing-to-a-writable-stream
 [31]: https://github.com/substack/stream-handbook#transform
 [32]: https://github.com/substack/stream-handbook#duplex
 [33]: https://github.com/substack/stream-handbook#classic-streams
 [34]: https://github.com/substack/stream-handbook#classic-readable-streams
 [35]: https://github.com/substack/stream-handbook#classic-writable-streams
 [36]: https://github.com/substack/stream-handbook#read-more
 [37]: http://nodejs.org/docs/latest/api/stream.html#stream_stream
 [38]: https://npmjs.org/package/readable-stream
 [39]: https://github.com/substack/stream-handbook#built-in-streams
 [40]: https://github.com/substack/stream-handbook#process
 [41]: https://github.com/substack/stream-handbook#processstdin
 [42]: http://nodejs.org/docs/latest/api/process.html#process_process_stdin

 [43]: http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback
 [44]: http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd
 [45]: https://github.com/substack/stream-handbook#processstdout
 [46]: http://nodejs.org/api/process.html#process_process_stdout
 [47]: https://github.com/substack/stream-handbook#processstderr
 [48]: http://nodejs.org/api/process.html#process_process_stderr
 [49]: https://github.com/substack/stream-handbook#child_processspawn
 [50]: https://github.com/substack/stream-handbook#fs
 [51]: https://github.com/substack/stream-handbook#fscreatereadstream
 [52]: https://github.com/substack/stream-handbook#fscreatewritestream
 [53]: https://github.com/substack/stream-handbook#net
 [54]: https://github.com/substack/stream-handbook#netconnect

 [55]: http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener
 [56]: https://github.com/substack/stream-handbook#netcreateserver
 [57]: https://github.com/substack/stream-handbook#http
 [58]: https://github.com/substack/stream-handbook#httprequest
 [59]: https://github.com/substack/stream-handbook#httpcreateserver
 [60]: https://github.com/substack/stream-handbook#zlib
 [61]: https://github.com/substack/stream-handbook#zlibcreategzip
 [62]: https://github.com/substack/stream-handbook#zlibcreategunzip
 [63]: https://github.com/substack/stream-handbook#zlibcreatedeflate
 [64]: https://github.com/substack/stream-handbook#zlibcreateinflate
 [65]: https://github.com/substack/stream-handbook#control-streams
 [66]: https://github.com/substack/stream-handbook#through
 [67]: https://github.com/dominictarr/through
 [68]: https://github.com/substack/stream-handbook#from
 [69]: https://github.com/dominictarr/from
 [70]: https://github.com/substack/stream-handbook#pause-stream
 [71]: https://github.com/dominictarr/pause-stream
 [72]: https://github.com/substack/stream-handbook#concat-stream
 [73]: https://github.com/maxogden/node-concat-stream
 [74]: https://github.com/substack/stream-handbook#duplex-1
 [75]: https://github.com/dominictarr/duplex
 [76]: https://github.com/substack/stream-handbook#duplexer
 [77]: https://github.com/Raynos/duplexer
 [78]: https://github.com/substack/stream-handbook#emit-stream
 [79]: https://github.com/substack/emit-stream
 [80]: https://github.com/substack/stream-handbook#invert-stream
 [81]: https://github.com/dominictarr/invert-stream
 [82]: https://github.com/substack/stream-handbook#map-stream
 [83]: https://github.com/dominictarr/map-stream
 [84]: https://github.com/substack/stream-handbook#remote-events
 [85]: https://github.com/dominictarr/remote-events
 [86]: https://github.com/substack/stream-handbook#buffer-stream
 [87]: https://github.com/Raynos/buffer-stream
 [88]: https://github.com/substack/stream-handbook#event-stream
 [89]: https://github.com/dominictarr/event-stream
 [90]: https://github.com/substack/stream-handbook#auth-stream
 [91]: https://github.com/Raynos/auth-stream
 [92]: https://github.com/substack/stream-handbook#meta-streams
 [93]: https://github.com/substack/stream-handbook#mux-demux
 [94]: https://github.com/dominictarr/mux-demux
 [95]: https://github.com/substack/stream-handbook#stream-router
 [96]: https://github.com/Raynos/stream-router
 [97]: https://github.com/substack/stream-handbook#multi-channel-mdm
 [98]: https://github.com/Raynos/multi-channel-mdm
 [99]: https://github.com/substack/stream-handbook#state-streams
 [100]: https://github.com/substack/stream-handbook#crdt
 [101]: https://github.com/dominictarr/crdt
 [102]: https://github.com/substack/stream-handbook#delta-stream
 [103]: https://github.com/Raynos/delta-stream
 [104]: https://github.com/substack/stream-handbook#scuttlebutt
 [105]: https://github.com/dominictarr/scuttlebutt
 [106]: https://en.wikipedia.org/wiki/Gossip_protocol
 [107]: https://en.wikipedia.org/wiki/Eventual_consistency
 [108]: https://github.com/substack/dnode
 [109]: https://github.com/substack/stream-handbook#append-only
 [110]: http://github.com/Raynos/append-only
 [111]: https://github.com/substack/stream-handbook#http-streams
 [112]: https://github.com/substack/stream-handbook#request
 [113]: https://github.com/mikeal/request
 [114]: https://github.com/substack/stream-handbook#oppressor
 [115]: https://github.com/substack/stream-handbook#response-stream
 [116]: https://github.com/substack/response-stream
 [117]: https://github.com/substack/stream-handbook#io-streams
 [118]: https://github.com/substack/stream-handbook#reconnect
 [119]: https://github.com/dominictarr/reconnect
 [120]: https://github.com/substack/stream-handbook#kv
 [121]: https://github.com/dominictarr/kv
 [122]: https://github.com/substack/stream-handbook#discovery-network
 [123]: https://github.com/Raynos/discovery-network
 [124]: https://github.com/substack/stream-handbook#parser-streams
 [125]: https://github.com/substack/stream-handbook#tar
 [126]: https://github.com/creationix/node-tar
 [127]: https://github.com/substack/stream-handbook#trumpet
 [128]: https://github.com/substack/node-trumpet
 [129]: https://github.com/substack/stream-handbook#jsonstream
 [130]: https://github.com/dominictarr/JSONStream
 [131]: https://github.com/substack/stream-handbook#json-scrape
 [132]: https://github.com/substack/json-scrape
 [133]: https://github.com/substack/stream-handbook#stream-serializer
 [134]: https://github.com/dominictarr/stream-serializer
 [135]: https://github.com/substack/stream-handbook#browser-streams
 [136]: https://github.com/substack/stream-handbook#shoe
 [137]: https://github.com/substack/shoe
 [138]: https://github.com/substack/stream-handbook#domnode
 [139]: https://github.com/maxogden/domnode
 [140]: https://github.com/substack/stream-handbook#sorta
 [141]: https://github.com/substack/sorta
 [142]: https://github.com/substack/stream-handbook#graph-stream
 [143]: https://github.com/substack/graph-stream
 [144]: https://github.com/substack/stream-handbook#arrow-keys
 [145]: https://github.com/Raynos/arrow-keys
 [146]: https://github.com/substack/stream-handbook#attribute
 [147]: https://github.com/Raynos/attribute
 [148]: https://github.com/substack/stream-handbook#data-bind
 [149]: https://github.com/travis4all/data-bind
 [150]: https://github.com/substack/stream-handbook#html-streams
 [151]: https://github.com/substack/stream-handbook#hyperstream
 [152]: https://github.com/substack/hyperstream
 [153]: https://github.com/substack/stream-handbook#audio-streams
 [154]: https://github.com/substack/stream-handbook#baudio
 [155]: https://github.com/substack/baudio
 [156]: https://github.com/substack/stream-handbook#rpc-streams
 [157]: https://github.com/substack/stream-handbook#dnode
 [158]: img/all_the_way_down.png
 [159]: https://github.com/substack/stream-handbook#rpc-stream
 [160]: https://github.com/dominictarr/rpc-stream
 [161]: https://github.com/substack/stream-handbook#test-streams
 [162]: https://github.com/substack/stream-handbook#tap
 [163]: https://github.com/isaacs/node-tap
 [164]: https://github.com/substack/stream-handbook#stream-spec
 [165]: https://github.com/dominictarr/stream-spec
 [166]: https://github.com/substack/stream-handbook#power-combos

 [167]: https://github.com/substack/stream-handbook#distributed-partition-tolerant-chat
 [168]: https://github.com/substack/stream-handbook#roll-your-own-socketio
 [169]: http://github.com/substack/shoe
 [170]: https://github.com/substack/node-browserify
 [171]: https://npmjs.org/package/eventemitter2

 [172]: https://github.com/substack/stream-handbook#html-streams-for-the-browser-and-the-server
 [173]: http://github.com/substack/brfs
 [174]: https://github.com/substack/hyperglue
 [175]: https://github.com/substack/slice-file
 [176]: http://browserify.org