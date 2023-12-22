---
layout: post
title:  "Профилирование Node.js-приложений"
date:   2023-03-09 18:40:43 +0400
description: Статья о профилировании и оптимизации Node.js-приложений.
tags: [ 'js' ]
categories: [ 'javascript' ]
disqus_comments: true
related_posts: true

---

## С чего начнем?

Иногда нам сложно понять, какого черта приложение работает медленно, хотя не должно. 
А иногда оно вроде бы и работает ОК, но нужно, чтобы оно работало быстрее. 
Можно, конечно, долго и вдумчиво читать код, рандомно что-то менять, замерять скорость и надеяться только на себя.

К счастью, в Node.js встроен помощник на такие случаи — профайлер. Точнее, он встроен в V8 (что это такое, можно почитать [здесь](https://sptm.dev/2023/asynchrony-in-js/) или даже [здесь](https://v8.dev)).
Профайлер V8 — это такая штука, которая делит стек выполняющейся программы (о стеке можно почитать [в моей статье об управлении памятью в JS](https://sptm.dev/2023/memory-management-in-js/)) на фрагменты через равные промежутки времени.
А представляет он нам результаты этих фрагментов с учетом оптимизаций (к примеру, JIT-компиляции) в виде ряда тиков. Примерно вот так:

```csv
code-creation,LazyCompile,0,0x2d5000a337a0,396,"bp native array.js:1153:16",0x289f644df68,~
code-creation,LazyCompile,0,0x2d5000a33940,716,"hasOwnProperty native v8natives.js:198:30",0x289f64438d0,~
code-creation,LazyCompile,0,0x2d5000a33c20,284,"ToName native runtime.js:549:16",0x289f643bb28,~
code-creation,Stub,2,0x2d5000a33d40,182,"DoubleToIStub"
code-creation,Stub,2,0x2d5000a33e00,507,"NumberToStringStub"
```

Не очень понятно, правда? В прошлом пришлось бы отдельно собрать V8, чтобы иметь возможность анализировать эти тики. 
Но, к счастью, в 2016 году у Node.js появились опции `--prof` и `--prof-process`, которые позволяют делать это сильно проще.

Давайте представим, что пишем сверх-хайлоад-микросервис по созданию и аутентификации пользователей на Express, а потом отпрофилируем его, чтобы понять, как его можно ускорить.

```ts
import express from 'express';
import crypto from 'crypto';


const app = express();
const users = [];


app.get('/newUser', (req, res) => {
    let username = req.query.username || '';
    const password = req.query.password || '';

    username = username.replace(/[!@#$%^&*]/g, '');

    if (!username || !password || users[username]) {
        return res.sendStatus(400);
    }

    const salt = crypto.randomBytes(128).toString('base64');
    const hash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

    users[username] = { salt, hash };

    res.sendStatus(200);
});


app.get('/auth', (req, res) => {
    let username = req.query.username || '';
    const password = req.query.password || '';

    username = username.replace(/[!@#$%^&*]/g, '');

    if (!username || !password || !users[username]) {
        return res.sendStatus(400);
    }

    const { salt, hash } = users[username];
    const encryptHash = crypto.pbkdf2Sync(password, salt, 10000, 512, 'sha512');

    if (crypto.timingSafeEqual(hash, encryptHash)) {
        res.sendStatus(200);
    } else {
        res.sendStatus(401);
    }
});



app.listen(3000, () => {
    console.log('Example app listening on port 3000')
});
```

## Профилирование

Вот мы написали приложение, выкатили, а пользователи жалуются, что все медленно. Мы запустим приложение с опцией `--prof` и сымитируем нагрузку на приложение с помощью [ApacheBench](https://httpd.apache.org/docs/2.4/programs/ab.html):

```bash
node --prof express-app.js # запускаем приложение с опцией --prof

# создаем пользователя
curl -X GET "http://localhost:3000/newUser?username=user&password=password" 

# имитируем множество авторизаций
ab -k -c 20 -n 250 "http://localhost:3000/auth?username=user&password=password"
```

```
# получаем такой результат:

Concurrency Level:      20
Time taken for tests:   6.620 seconds
Complete requests:      250
Failed requests:        0
Keep-Alive requests:    250
Total transferred:      57250 bytes
HTML transferred:       500 bytes
Requests per second:    37.77 [#/sec] (mean)
Time per request:       529.579 [ms] (mean)
Time per request:       26.479 [ms] (mean, across all concurrent requests)
Transfer rate:          8.45 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   2.0      0       9
Processing:    29  506 129.5    502    1038
Waiting:       29  506 129.6    502    1038
Total:         29  507 128.8    502    1038

Percentage of the requests served within a certain time (ms)
  50%    502
  66%    503
  75%    507
  80%    529
  90%    535
  95%    551
  98%   1001
  99%   1005
 100%   1038 (longest request)
```

Получаем, что наше приложение может обработать только 37.77 запросов в секунду, и запрос в среднем занимает около полусекунды, что не очень-то для хайлоад-приложения.
Поскольку мы запустили наше приложение с опцией `--prof`, в каталоге с приложением появится файл вида `isolate-0x<много цифр>-v8.log`.
Разберемся в этом файле с помощью тикового процессора, встроенного в Node.js (флаг `--prof-process`):

```bash
node --prof-process isolate-0xnnnnnnnnnnnn-v8.log > processed.txt
```

Откроем файл `processed.txt`, увидим кучу секций, разбитых по языкам. Глянем в Summary:

```
[Summary]:
   ticks  total  nonlib   name
     11    0.2%    0.2%  JavaScript
   5454   99.1%   99.7%  C++
     18    0.3%    0.3%  GC
     36    0.7%          Shared libraries
      4    0.1%          Unaccounted
```

Целых 99 с чем-то процентов заняли задачи, выполняемые C++. Выглядит не очень и само по себе, но посмотрим в раздел C++, чтобы понять, какие функции заняли больше процессорного времени:

```
[C++]:
   ticks  total  nonlib   name
   5357   97.3%   98.0%  T node::contextify::ContextifyScript::~ContextifyScript()
     42    0.8%    0.8%  T node::builtins::BuiltinLoader::CompileFunction(v8::FunctionCallbackInfo<v8::Value> const&)
     20    0.4%    0.4%  T node::contextify::ContextifyContext::CompileFunction(v8::FunctionCallbackInfo<v8::Value> const&)
     16    0.3%    0.3%  T _semaphore_destroy
      9    0.2%    0.2%  T _posix_spawnattr_setmacpolicyinfo_np
      3    0.1%    0.1%  t std::__1::basic_ostream<char, std::__1::char_traits<char> >& std::__1::__put_character_sequence<char, std::__1::char_traits<char> >(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, char const*, unsigned long)
      2    0.0%    0.0%  t std::__1::ostreambuf_iterator<char, std::__1::char_traits<char> > std::__1::__pad_and_output<char, std::__1::char_traits<char> >(std::__1::ostreambuf_iterator<char, std::__1::char_traits<char> >, char const*, char const*, char const*, std::__1::ios_base&, char)
      2    0.0%    0.0%  T _mach_get_times
      1    0.0%    0.0%  T _mig_dealloc_reply_port
      1    0.0%    0.0%  T __simple_putline
      1    0.0%    0.0%  T ___psynch_cvbroad
```

Все еще не очень понятно :( на самом деле, иногда все становится понятно и из этой секции — но не в этот раз.
Чтобы лучшве понять, что это за `node::contextify::ContextifyScript::~ContextifyScript()`, отбирающий 98% процессорного времени, обратимся к секции `Bottom up (heavy) profile`, которая предоставляет информацию о том,где чаще всего вызывается каждая функция. Там мы видим это:

```
[Bottom up (heavy) profile]:
  Note: percentage shows a share of a particular caller in the total
  amount of its parent calls.
  Callers occupying less than 1.0% are not shown.

   ticks parent  name
   5357   97.3%  T node::contextify::ContextifyScript::~ContextifyScript()
   5036   94.0%    Function: ^pbkdf2Sync node:internal/crypto/pbkdf2:68:20
   4993   99.1%      Function: ^<anonymous> file:///Users/sptm/Documents/Development/test/fastify/express-app.js:28:18
   4993  100.0%        Function: ^handle /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/layer.js:86:49
   4888   97.9%          Function: ^next /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:116:16
   4888  100.0%            Function: ^dispatch /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:98:45
    105    2.1%          LazyCompile: ~next /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:116:16
    105  100.0%            Function: ^dispatch /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:98:45
    194    3.6%    LazyCompile: ~pbkdf2Sync node:internal/crypto/pbkdf2:68:20
    173   89.2%      LazyCompile: ~<anonymous> file:///Users/sptm/Documents/Development/test/fastify/express-app.js:28:18
    107   61.8%        Function: ^handle /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/layer.js:86:49
    107  100.0%          LazyCompile: ~next /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:116:16
    107  100.0%            LazyCompile: ~dispatch /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:98:45
     66   38.2%        LazyCompile: ~handle /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/layer.js:86:49
     66  100.0%          LazyCompile: ~next /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:116:16
     66  100.0%            LazyCompile: ~dispatch /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:98:45
     21   10.8%      LazyCompile: ~<anonymous> file:///Users/sptm/Documents/Development/test/fastify/express-app.js:9:21
     21  100.0%        LazyCompile: ~handle /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/layer.js:86:49
     21  100.0%          LazyCompile: ~next /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:116:16
     21  100.0%            LazyCompile: ~dispatch /Users/sptm/Documents/Development/test/fastify/node_modules/express/lib/router/route.js:98:45
```

## Оптимизация

А вот отсюда очевидно, что практически все процессорное время отнимает функция `pbkdf2Sync()`, которой мы генерируем хэш на основе пароля.
К счастью, мы понимаем, что эта функция работает синхронно и блокирует event loop (если не понимаем, о чем речь, читаем вот эту [статью об асинхронности](https://sptm.dev/2023/asynchrony-in-js/)).
Попробуем использовать асинхронную версию функции `pbkdf2Sync()` (`pbkdf2()`):

```ts
app.get('/auth', (req, res) => {
  let username = req.query.username || '';
  const password = req.query.password || '';

  username = username.replace(/[!@#$%^&*]/g, '');

  if (!username || !password || users[username]) {
    return res.sendStatus(400);
  }

  crypto.pbkdf2(password, users[username].salt, 10000, 512, 'sha512', (err, hash) => {
    if (users[username].hash.toString() === hash.toString()) {
      res.sendStatus(200);
    } else {
      res.sendStatus(401);
    }
  });
});
```

И снова сымитируем нагрузку. Получим такие результаты:

```
Concurrency Level:      20
Time taken for tests:   0.073 seconds
Complete requests:      250
Failed requests:        0
Non-2xx responses:      250
Keep-Alive requests:    250
Total transferred:      62000 bytes
HTML transferred:       2750 bytes
Requests per second:    3444.38 [#/sec] (mean)
Time per request:       5.807 [ms] (mean)
Time per request:       0.290 [ms] (mean, across all concurrent requests)
Transfer rate:          834.19 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   3.2      0      14
Processing:     1    4   2.7      3      18
Waiting:        1    3   2.6      3      16
Total:          1    4   5.4      3      24

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      3
  75%      3
  80%      4
  90%      7
  95%     22
  98%     23
  99%     23
 100%     24 (longest request)
```

Сильно лучше, правда? От 38 запросов в секунду мы выросли практически на два порядка, до 3444 запросов. 
А один запрос теперь занимает около 5 миллисекунд вместо 529.

## Итоги

Надеюсь, на этом полностью синтетическом и выдуманном примере мы научились профилировать наши приложения.
Ну, и поняли, насколько вредны синхронные блокирующие операции :)
