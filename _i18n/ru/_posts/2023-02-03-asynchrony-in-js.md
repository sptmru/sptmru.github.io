---
layout: post
title:  "Асинхронность в JavaScript: практика и теория"
date:   2023-02-03 15:31:43 +0400
description: Статья об асинхронности в JS, V8, event loop и таком прочем
tags: [ 'js' ]
categories: [ 'javascript' ]
languages: [ 'ru' ]
disqus_comments: true
related_posts: true


---

Достаточно важная как для браузерного JS, так и для Node.js (на котором мы остановимся подробнее) тема — асинхронность.
Все мы слышали и знаем, что JavaScript — язык асинхронный, и все мы знаем, что он еще и однопоточный.

Если чуть задуматься об этом, становится непонятно: а как такое в принципе возможно? Если у нас есть только один поток, то что произойдет, если мы выполним в нем блокирующую операцию — например, обратимся к файловой системе или базе данных? Разве может успешно существовать язык программирования, с программами на котором может одновременно работать только один процесс?

Конечно, в JS это решено, и решено довольно интересно. И, как и все интересные штуки, эта штука достаточно сложна. Поэтому эта статья — статья-перевертыш: мы начнем с того, как на практике работать с асинхронными задачами, а уже потом разберем, что там под капотом происходит.

## Практика

### Callbacks

Итак, для начала вернемся во времена до выхода ES6, существенно прокачавшего JS. Тогда, в эти прекрасные дни, асинхронные штуки мы делали примерно так:

```js
function blockingFunc (callback) {
  // do blocking things
  callback();
}

function logDone () {
  console.log('Done');
}

blockingFunc(logDone);
```

Что здесь происходит? У нас есть некая функция `blockingFunc()`, которая делает нечто, блокирующее наш поток, а потом выполняет некую функцию `callback()`, которую она принимает аргументом (да, в JavaScript мы можем передать в функцию аргументом другую функцию. Это называется «функции первого класса». На самом деле, мы можем даже передать в функцию целый класс, и получится у нас класс первого класса. Но это уже другая тема).

В чем проблема такого подхода к написанию кода? А в том, что называли «callback hell». Мы выполняем функцию, в которую передаем функцию-коллбек, в которую передавали свою функцию-коллбек…и очень скоро это превращалось во что-то не очень читаемое. Смотрите:

```js
getBeef('John', function(rawBeef) {
  sliceBeef(rawBeef, function(slicedBeef) {
    cookBeef(slicedBeef, function(cookedBeef) {
      serveBeef(cookedBeef, function(servedBeef) {
        console.log(servedBeef)
      })
    })
  })
})
```

Видите проблему такого подхода? Мне кажется даже, что здесь я запутался в коллбеках :) а читать такое попросту мучительно.

### Promises

К счастью, вместе с ES6 к нам пришли промисы (Promise) и async/await. Что это такое и как это использовать, рассмотрим ниже.

Promise — это под капотом те же коллбеки, но использовать и читать их гораздо приятнее.  Давайте перепишем код выше с использованием промисов. Представим, что функция `getBeef()` выглядит примерно так: (небольшое отступление: в ней я использовал еще одну фичу ES6 — стрелочные функции, о них можно прочитать у меня [здесь](https://github.com/sptmlearningjs/execution-context-in-js):

```js
function getBeef(name) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (beefExists) {
        const beef = true;
        resolve(beef);
      } else {
        reject('No beef available')
      }
    }, 300)
  });
}
```

Что происходит здесь? Функция `getBeef(name)` принимает имя человека, для которого мы в итоге будем готовить мясо и возвращает объект Promise.

Promise, как написано в документации, это такой прокси-объект для значения, которое не обязательно известно, когда промис создается. В нашем случае мы не знаем, есть ли у нас мясо, когда создаем промис, и для проверки нам нужно немного времени. Мы не хотим ждать, блокируя основной поток, поэтому проверяем это асинхронно.

Promise находится в одном из трех состояний:

- pending: изначальное состояние созданного промиса
- fulfilled: означает, что операция выполнена успешно
- rejected: означает, что операция провалена

Мы считаем промис выполненным (settled), если он либо выполнен успешно (fulfilled), либо провален (rejected), но не находится в ожидании (pending). Часто говорят, что промис зарезолвился (resolved), когда он выполнен.

Когда мы создаем промис, мы устанавливаем значения, которые он вернет в случае, когда он выполнен успешно (с помощью функции `resolve()`) и когда он провален (с помощью `reject()`). В нашем случае, если промис выполнен успешно (условная переменная `beefExists === true`), мы вернем `beef` (то самое мясо), а если промис провален, мы вернем сообщение, что мяса, к сожалению, нет.

У Promise есть три метода: `then()`,  `catch()`, `finally()`.
`then()` принимает два аргумента: первый — это коллбек, выполняющийся, когда промис успешно зарезолвился, второй — коллбек, который выполнится, если промис провалится.
Часто второй аргумент опускают, ведь есть `catch()`, который принимает один аргумент — коллбек, который выполнится, если промис провалится. `finally()` тоже принимает один аргумент, который выполнится, когда промис зарезолвится любым образом: провалится ли — или же успешно выполнится.

Все эти методы возвращают новый Promise, что очень удобно, ведь их можно связывать в цепочки (chaining). Теперь давайте перепишем весь наш код на коллбеках с использованием промисов:

```js
getBeef('John')
  .then(sliceBeef)
  .then(cookBeef)
  .then(serveBeef)
  .then((servedBeef) => servedBeef ? console.log('Beef is served!'): console.log('Beef is not served'))
  .catch((err) => console.log(err));

```

Гораздо более читабельно, правда?

Есть еще два интересных метода, связанных с промисами: `Promise.all()` и `Promise.allSettled()`. Оба позволяют комбинировать промисы, получая из них новый, но `Promise.all()` будет считаться fulfilled только тогда, когда все переданные в него промисы будут fulfilled. Для `Promise.allSettled()` же, чтобы считаться fulfilled, достаточно, чтобы все промисы, в него переданные, не были pending (то есть все промисы должны как-то отрезолвиться, fulfilled или rejected — без разницы). Давайте посмотрим на примере:

```js
const promise1 = Promise.resolve(true); // fulfilled promise
const promise2 = Promise.resolve(false); // also fulfilled promise
const promise3 = Promise.reject(true); // rejected promise
const promise4 = new Promise((resolve, reject) => {}); // pending promise

Promise.all([promise1, promise2])
  .then(() => console.log('fulfilled!'))
  .catch(() => new Error('rejected!')); // fulfilled! (as all promises are fulfilled)

Promise.all([promise1, promise2, promise3])
  .then(() => console.log('fulfilled!'))
  .catch(() => console.log('rejected!')); // rejected! (as promise3 is rejected)

Promise.all([promise1, promise2, promise4])
  .then(() => console.log('fulfilled!'))
  .catch(() => console.log('rejected!'));
  // nothing, Promise.all is still pending (for Promise.all to be resolved, all promises passed into it should be resolved)

Promise.allSettled([promise1, promise2, promise3])
  .then(() => console.log('fulfilled!'))
  .catch(() => new Error('rejected!')); // fulfilled! (as all promises are resolved)

Promise.allSettled([promise1, promise2, promise4])
  .then(() => console.log('fulfilled!'))
  .catch(() => console.log('rejected!'));
  // nothing, Promise.allSettled is still pending (for Promise.allSettled to be resolved, all promises passed into it should be resolved)
```

Есть еще один прикольный метод, связанный с промисами — `Promise.race()`. Он похож на предыдущие два, разница лишь в том, что резолвится `Promise.race()` тогда, когда резолвится хотя бы один из переданных в него промисов. Глянем на примере:

```js
const promise1 = new Promise((resolve, reject) => {
  setTimeout(resolve, 500, 'one');
});

const promise2 = new Promise((resolve, reject) => {
  setTimeout(resolve, 100, 'two');
});

Promise.race([promise1, promise2]).then((value) => {
  console.log(value);
  // Both resolve, but promise2 is faster
});
// Expected output: "two"
```

### Async / Await

Итак, с промисами разобрались. Учитывая следующую тему, которую мы в этой статье разберем, стоит сказать: промисы — это не какая-то около-deprecated-штука, это вполне себе актуальная и часто используемая функциональность JS, и она никуда не пропадет. Почему я это говорю?

А потому что в JS можно писать асинхронный  код, который буквально выглядит как синхронный: для этого в ES6 вместе с промисами подъехал синтаксический сахар над ними — `async` и `await`. Давайте воспользуемся предыдущим опытом и перепишем код, что мы писали выше, с использованием `async/await`:

```js
async () => {
  try {
    const beef = await getBeef('John');
    const slicedBeef = await sliceBeef(beef);
    const cookedBeef = await cookBeef(slicedBeef);
    const servedBeef = await serveBeef(cookedBeef);

    servedBeef ? console.log('Beef is served!') : console.log('Beef is not served!');
  } catch (err) {
    console.log(err);
  }
}
```

Стало еще более читабельно, правда? Но здесь нет ничего нового — это всего лишь синтаксический сахар (хотя, конечно, очень приятный) над промисами (которые, в свою очередь, просто синтаксический сахар над коллбеками. Но какая разница? Удобнее стало? Стало. Вот и отлично.)

## Теория

Что ж, с практикой мы вроде как разобрались. Пора закопаться поглубже и разобраться, как оно все работает. А работает оно довольно интересно.

Давайте для начала разберемся, из чего состоит Node.js.

### Компоненты Node.js

#### V8

Cобственно, движок, который компилирует и выполняет JavaScript-код, занимается выделением памяти под объекты и включает в себя сборщик мусора, который освобождает память, занимаемую объектом, когда ссылок на объект уже нет и он уже не нужен.

Раньше V8 занимался JIT-компиляцией всегда, теперь же (с версии 7.4) у него появилась опция `—jitless`, в этом режиме он работает исключительно как интерпретатор. Это полезно, когда мы не можем аллоцировать память в рантайме (к примеру, в iOS, некоторых смарт-TV и прочем таком). Нам, как бэкенд-разработчикам, знать такое не обязательно, но разве же знания бывают лишними?

Итак, V8. Компилирует и выполняет JS-код, работает со стеком, управляет выделением памяти и сборкой мусора, а также
обеспечивает нас всеми типами данных, операторами, объектами и функциями.

#### libuv

К сожалению, V8 не может работать с операционными системами напрямую — это было бы сложно поддерживать (к примеру, у *nix-систем есть проблемы с поддержкой non-blocking I/O-операций, и работа с ними напрямую была бы как минимум крайне нетривиальной задачей). Поэтому приходится использовать библиотеку-посредник между V8 и ОС — собственно, libuv и используется. Удобно, что есть один мультиплатформенный инструмент, реализующий асинхронный ввод-вывод — без такого инструмента существование Node.js было бы попросту невозможно.

#### C++ - аддоны

Да, для Node.js написано множество аддонов на C++. С помощью них, например, реализованы воркеры, и мы можем в дополнение к основному потоку выполнения разворачивать дополнительные дочерние треды.

#### Node.js - биндинги

Для того, чтобы связываться и работать с какими-то внешними штуками, Node.js нужны биндинги — по сути, это библиотеки, которые позволяют нам использовать библиотеки, написанные на других языках, вместе с Node.js. Это нужно, к примеру, для разного рода I/O: соединение с базами данных, файловыми системами и прочими такими штуками.

### Event Loop

И вот теперь, наконец, мы подошли к главному, к тому, ради чего эта статья и затевалась — к возможности ответить на вопрос, как же все-таки Node.js выполняет неблокирующие  (асинхронные операции ввода-вывода, несмотря на то, что он однопоточный — оказывается, он с помощью libuv отдает множество работы операционной системе.

Учитывая то, что большинство современных ядер ОС многопоточны, они могут выполнять множество операций одновременно в фоне. Когда какая-то из этих операций завершается, ядро операционной системы дает об этом знать Node.js, чтобы соответствующий коллбек мог быть добавлен в poll-очередь (мы объясним, что это, чуть позже), чтобы со временем быть выполненным.

Итак, когда Node.js запускается, он запускает event loop и исполняет входящий скрипт, который может выполнять асинхронные запросы к API, планировать таймеры или вызывать `process.nextTick()`, после чего начинает исполнять цикл событий (собственно, event loop).

Давайте для начала вкратце определимся, как примерно работает event loop, что такое микротаски и макротаски, а затем разберем фазы event loop.

#### Микротаски и макротаски

Помимо обычных синхронных задач, в JS существуют так называемые микротаски и макротаски.

Микротаски имеют приоритет над макротасками, и, пока выполняются задачи из очереди микротасок, до макротасок дело не дойдет. Микротаски — это промисы, то, что запланировано с помощью `process.nextTick()` и `queueMicrotack()`, в браузере — еще и результат работы `MutationObserver` (вкратце — такая штука, генерящая уведомления об изменении определенных DOM-элементов).
Макротаски — это то, что выполняется только тогда, когда очередь микротасок пуста. Это то, что мы планируем с помощью `setTimeout()`, `setImmediate()`, `setInterval()`, в браузере — еще и рендеринг. С учетом того, что микротаски блокируют выполнение макротасок, напрашивается очевидный вывод — не стоит забивать очередь микротасок, иначе мы заблокируем все макротаски.

Если вкратце, event loop работает так: выполняет микротаски, пока они есть, ставя себя на паузу, затем продолжается, исполняет макротаску и возвращается к микротаскам.

#### Фазы Event Loop

Для начала — коротко перечислим все эти фазы и объясним, что в них происходит, после чего перейдем к деталям.

- timers -  в этой фазе исполняются коллбеки, запланированные с помощью `setTimeout()` и `setInterval()`
- pending callbacks - выполняются коллбеки ввода/вывода, отложенные до следующей итерации цикла событий
- idle, prepare - используются исключительно для внутренних нужд
- poll - получаются новые события ввода/вывода, выполняются связанные с вводом/выводом коллбеки (почти все, за исключением тех, которые запланированы таймерами и `setImmediate()`). Поток будет блокироваться здесь, когда это уместно. Тут мы как раз работаем с очередью микротасок.
- check - `setImmediate()` - коллбеки вызываются здесь
- close callbacks - закрываются некоторые коллбеки типа socket.on(‘close’, …)

Между каждым запуском цикла событий, Node.js проверяет, не ожидает ли он каких-то асинхронных операций ввода-вывода, таймеров и останавливается без ошибок, если ничего не ожидается.

Давайте разбирать эти фазы чуть подробнее.

#### timers

Таймер указывает некий порог времени, после которого коллбек может быть выполнен, это не точное время, через которое коллбек точно выполнится. Коллбеки таймеров будут запущены так рано, как могут, после того, как указанное в таймере время пройдет — однако, планирование ОС или выполнение других коллбеков может отложить их выполнение.

На самом деле, фаза poll контролирует, когда таймеры выполняются. К примеру, скажем, вы запланировали таймаут так, чтобы он выполнился после задержки в 100 мс, а потом ваш скрипт начинает асинхронно читать файл, что занимает 95 мс.
Когда event loop входит в фазу poll, его очередь пуста (`fs.readFile()`еще не выполнена), так что он подождет то количество миллисекунд, которого хватит для выполнения ближайшего по времени таймера. Пока event loop ждет 95 миллисекунд, `fs.readFile()` заканчивает читать файл, и его коллбек, которому требуется 10 мс для выполнения добавляется в очередь poll и выполняется. Когда он заканчивает выполнение, в очереди больше нет коллбеков, поэтому event loop увидит, что порог ближайшего таймера достигнут, и вернется к фазе таймеров, чтобы выполнить коллбек таймера. Таким образом, в этом примере общая задержка между планированием таймера и выполнением его коллбека составит 105 мс.

Чтобы перебор микротасок не приводил к тому, чтобы в event loop ничего не поступало, библиотека `libuv` тоже имеет зависимое от ОС ограничение максимального количества ивентов, прежде чем она прекратит запрашивать новые события.

#### pending callbacks

На этом этапе выполняются коллбеки для разного рода системных операций. К примеру, если TCP-сокет получает ошибку `ECONNREFUSED` (в соединении отказано), некоторые *nix-системы захотят подождать перед тем, как сообщить об ошибке. Это и уйдет в очередь, чтобы выполниться в фазе pending callbacks.

#### poll

Фаза poll имеет два основных назначения:

- вычислить, сколько I/O будет блокировать поток
- обработать события в poll queue

Когда event loop переходит в фазу poll и нет ни одного запланированного таймера, случится что-то из этого:

- если очередь микротасок не пуста, event loop пройдется по очереди микротасок, выполняя их синхронно до тех пор, пока либо очередь не опустеет, либо не будет достигнут тот самый лимит на максимальное количество ивентов в libuv
- если же очередь микротасок пуста, произойдет еще одно из двух событий:
  - если что-то было запланировано с помощью `setImmediate()`, event loop закончит poll-фазу и выполнит то, что было запланировано
  - если ничего не было запланировано с помощью `setImmediate()`,  event loop подождет, пока коллбеки добавятся в очередь, и немедленно их выполнит

Как только очередь микротасок опустеет, event loop проверит наличие таймеров, пороговые значения которых был достигнуты. Если один или несколько таймеров доступны, event loop вернется к фазе timers и выполнит коллбеки этих таймеров.

#### check

Фаза check позволяет исполнять коллбеки сразу же после того, как фаза poll завершена. Если фаза poll простаивает, и что-то запланировано с помощью `setImmediate()`, event loop может продолжиться до фазы check вместо того, чтобы ждать.

На самом деле, `setImmediate()` — это такой специальный таймер, который запускается в отдельной фазе event loop. Он использует API `libuv`, чтобы планировать коллбеки к выполнению после завершения poll-фазы.

Как правило, по мере выполнения event loop в итоге переходит к фазе poll, где он будет ожидать входящего соединения, запроса или чего-то вроде того. Впрочем, если коллбек был запланирован с помощью `setImmediate()`, а фаза poll бездействует, она завершится и перейдет к фазе check вместо того≤ чтобы ждать poll-событий.

#### close callbacks

Если сокет или дескриптор внезапно закроются, ивент 'close' будет сгенерирован на этом этапе. Иначе он будет отправлен через `process.nextTick()`.

#### setImmediate() vs setTimeout()

`setImmediate()` и `setTimeout()` очень похожи, но ведут себя немного по-разному в зависимости от того, когда они вызваны.

- `setImmediate()` выполнит то, что запланировано, как только текущая poll-фаза будет завершена
- `setTimeout` выполнит запланированное после того, как минимальная указанная задержка в миллисекундах истечет

Порядок, в котором выполняются таймеры, зависит от контекста, в котором они вызываются. Если оба вызываются из основного модуля, то время будет зависеть от производительности процесса.

Например, если мы запустим вот этот скрипт, который не находится в I/O-цикле (то есть, в основном модуле)
, порядок, в котором они выполнятся, неопределен, поскольку связан с производительностью процесса:

```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```bash
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

Однако, если мы эти вызовы закинем в цикл I/O, коллбек `setImmediate()` всегда выполнится первым:

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```bash
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

Главное преимущество `setImmediate()` — это то, что оно всегда будет выполнено перед всеми таймерами внутри I/O-цикла, вне зависимости от того, как много таймеров существует.

#### process.nextTick() vs setImmmediate()

У нас есть две штуки, похожие друг на друга, но их названия могут нас путать.

`process.nextTick()` срабатывает сразу же на тоей же фазе event loop.
`setImmediate()` срабатывает на следующей итерации / "тике" event loop. Выглядит так, что имена функций нужно обменять местами.

`process.nextTick()` срабатывает быстрее, чем `setImmediate()`, но это легаси, и это вряд ли изменится — если это поменять, мы сломаем огромное количество пакетов в `npm`. Каждый день в `npm` добавляются новые модули, каждый день добавляется больше потенциальных поломок. Имена могут сбивать с толку, но они не изменятся.

Рекомендуется во всех случаях использовать `setImmediate()` и забыть о `process.nextTick()`. Совсем.

#### Но в каких случаях использовать process.nextTick()?

Есть две причины:

1. Разрешить пользователям обрабатывать ошибки, очищать все ненужные ресурсы или, возможно, повторить запрос снова перед тем, как event loop продолжится.
2. Иногда нам нужно разрешить выполнение коллбека после раскручивания стека вызовов, но до продолжения event loop.

Вот пример:

```js
const server = net.createServer();
server.on('connection', (conn) => {});

server.listen(8080);
server.on('listening', () => {});
```

Предположим, что `listen` запускается в начале event loop, но коллбек `listening` помещен в `setImmediate()`.Если имя хоста не передано, привязка к порту произойдет немедленно. Чтобы event loop продолжился, он лджлжен добраться до фазы poll — то есть существует ненулевая вероятность, что мы получим соединение и запустим событие `connection` до события `listening`.

Вот другой пример — расширение `EventEmitter` и отправка события из конструктора:

```js
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {
  constructor() {
    super();
    this.emit('event');
  }
}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

Нельзя сгенерировать событие из конструктора, поскольку скрипт не будет обработан до момента, когда пользователь назначает коллбек этому ивенту. Таким образом, внутри самого конструктора мы можем использовать `process.nextTick()`, чтобы установить коллбек для отправки `event` после того, как конструктор завершен — то есть все произойдет так, как мы и ожидаем.

## Выводы

Асинхронность — важнейшая тема в JavaScript. Понимая все тонкости того, как все это работает изнутри, можно писать гораздо более предсказуемый и производительный код. И да, есть соблазн пропустить к чертям теорию, попытавшись в нее не вникать (она ведь так сложна :) ) — но я бы крайне рекомендовал все же вникнуть поглубже, это даст вам возможность писать код намного более осознанно. И да, о теории — напоследок я порекомендую [вот это](https://www.youtube.com/watch?v=8aGhZQkoFbQ) выступление Филиппа Робертса с JSConf о event loop — в дополнение к этой статье это добавит вам понимания того, как все-таки однопоточный JavaScript работает асинхронно и очень быстро. Enjoy!