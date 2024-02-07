---
layout: post
title:  "Реактивное программирование: теория и практика"
date:   2023-03-12 01:20:43 +0400
description: Статья о реактивном программировании с примерами на Node.js Streams.
tags: [ 'js', 'mq', 'theory' ]
categories: [ 'javascript', 'mq', 'theory' ]
languages: [ 'ru' ]
disqus_comments: true
related_posts: true

---

## Введение

Вообще, реактивное программирование — не самая сложная штука.
Поэтому я постарался сконцентрироваться на реальном его применении в Node.js, и поэтому здесь будет больше кода, чем текста :)

### Что такое реактивное программирование?

Если по-простому, то реактивное программирование — это парадигма программирования, выстроенная вокруг событий и реагирования на них.
Было бы гораздо проще показать примеры реактивного программирования с использованием [EventEmitter](https://nodejs.dev/en/learn/the-nodejs-event-emitter/) (о нем у меня тоже будет статья, не сомневайтесь), но мне было интереснее разобраться с такой штукой, как [Node.js Streams](https://nodejs.org/api/stream.html#stream).
К тому же, получившийся пример вполне себе применим на практике.

Вообще, Streams обычно используются для чтения и записи буферизированных данных (бинарных файлов и всего такого).
Но, подумал я, что, если с помощью него работать с API? :)

В примере мы читаем данные из настоящего API, получая данные порционно (имитируя пагинацию), и сохраняем их в массив.
Понятное дело, что никто не мешает нам их обрабатывать и сохранять в базу, да и вообще делать с ними что угодно.

Кому не терпится посмотреть на код, то — [вот](https://github.com/sptmru/reactive-programming-on-streams). Остальных приглашаю разобраться вместе со мной :)

### API

Для того, чтобы продемонстрировать чтение из API стримами, я набросал простенькое API на Fastify. Вот оно:

```ts
import Fastify, { FastifyInstance, RouteShorthandOptions, RouteHandlerMethod } from 'fastify'


const srv: FastifyInstance = Fastify({})

interface Objects {
  [key: string]: number,
}

interface APIResponse {
  objects: Objects,
  start?: number,
  end?: number,
}

interface RequestQuery {
  offset?: number,
  limit?: number,
}

const opts: RouteShorthandOptions = {
  schema: {
    querystring: {
      type: 'object',
      properties: {
          offset: { type: 'number'},
          limit: {type: 'number'}
      },
    }
  },
}

const handler: RouteHandlerMethod = async (req, _res) => {
  const response: APIResponse = {
      objects: {}
  };

  const query = req.query as RequestQuery;

  const offset = Number(query?.offset) || 0;
  const limit  = Number(query?.limit) || 50;

  const finalNum = 10000;

  const end = (Number(offset) + Number(limit)) < finalNum
    ? (Number(offset) + Number(limit))
    : finalNum;

    for (let i = offset; i <= end; i++) {
      response.objects[i] = i;
    }

  response.start = offset;
  response.end = end;

  return response;
};

srv.get('/', opts, handler);

const start =  async () => {
  try {
    await srv.listen({ port: 3000 });
  } catch (err) {
    srv.log.error(err);
    process.exit(1);
  }
}

start();
```

Как мы видим, оно включает в себя всего один эндпойнт, который может отдать объекты вида `[key: string]: number` и поддерживает пагинацию (то есть, мы можем указать, с какого и по какое число нам сгенерировать объекты).
Есть и лимит: числа больше `10000` он не отдаст (сделано это для того, чтобы чтение из API всегда было конечно).
Все, больше ничего интересного: все самое интересное — в клиенте.

### Клиент

Клиент основан на Node.js-стримах и содержит два класса, наследующихся от [Readable](https://nodejs.org/api/stream.html#class-streamreadable) и [Writable](https://nodejs.org/api/stream.html#class-streamwritable) соответственно.
Тот, что `Readable`, порционно читает данные из API, соблюдая пагинацию — а тот, что `Writable`, эти данные так же порционнно собирает в массив.

Это немного напоминает очереди сообщений, о которых тоже будет статья.
Давайте смотреть код:

```ts
import { Writable, Readable } from 'node:stream';

const dbArr: object[] = [];

interface Objects {
    [key: string]: number;
}

interface APIResponse {
    start: number,
    end: number,
    objects: Objects,
}

class ApiReadable extends Readable {

    private uri: string;
    private offset: number;
    private limit: number

    constructor() {
        super({ objectMode: true });
        this.uri = 'http://localhost:3000/';
        this.offset = 0;
        this.limit = 100;
    }

    async getAPI(): Promise<APIResponse|null> {
        const response = await fetch(`${this.uri}?limit=${this.limit}&offset=${this.offset}`);
        const result = await response.json();

        // кончились данные в API? Вернем null;
        if (result?.end && (result.end - result.start) < this.limit ) {
            return null;
        }
        // вернем ответ API в виде JSON
        return result;
    }

    // переопределим встроенный в Readable метод _read
    override async _read() {
        // заберем нужное количество данных из API
        const result = await this.getAPI();

        // если данных нет, отдадим null во Writable — это сгенерит событие end
        if (!result) {
            this.push(null);
            return;
        } else {
            // а если данные есть, пихнем их во Writable
            this.push(result.objects);
        }

        // сместим пагинацию вперед
        this.offset = result.end > this.offset ? result.end + 1 : result.end;
    }
}

class ObjWritable extends Writable {

    private dbArr: object[];

    // конструктор принимает архив, в который будем пихать данные, полученные от Readable
    constructor(dbArr: object[]) {
        super({ highWaterMark: 5, objectMode: true });
        this.dbArr = dbArr;
    }

    writeToObj(chunk: object) {
        this.dbArr.push(chunk);
    }

    // переопределим метод Writable.prototype._write - теперь он пишет в наш массив полученные данные и исполняет свой дефолтный коллбек
    override _write(chunk: object, _encoding: string, callback: Function) {
        this.writeToObj(chunk);
        callback();
    }

    // то же самое произойдет и с дефолтным методом _writev, который пишет несколько кусков (chunks) данных за раз
    override _writev(chunks: object[], callback: Function) {
        this.dbArr.push(...chunks);
        callback();
    }

}

// создадим экземпляры наших классов
const objWritable = new ObjWritable(dbArr);
const apiReadable = new ApiReadable();

apiReadable.pipe(objWritable); // и самое интересное: подпишем Writable на Readable
apiReadable.on('end', () => console.log(dbArr));
```

С комментами понятнее? Да, там внутри EventEmitter :) но как красиво! Забираем данные из API порционнно, порционно их куда угодно пишем.
Обратите внимание на параметр `highWaterMark` у `Writable` — это количество одновременно хранимых данных. Так можно и rate limiting безо всяких брокеров сообщений соорудить :)

В общем, мне кажется, это отличный и небанальный пример реактивного программирования на Node.js.

## Выводы

Вообще, JavaScript и на фронте, и на бэке прекрасно подходит для реализации event-driven архитектуры.
И реактивное программирование может быть не какой-то далекой от реальной разработки теорией, а вполне себе применимой штукой.

Stay tuned! Хоть блог и для новичков по большей части, но иногда я буду писать и что-то повеселее :)
