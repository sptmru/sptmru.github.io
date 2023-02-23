---
layout: post
title:  "Proxy и Reflect — зачем и когда?"
date:   2023-02-21 11:10:43 +0400
description: Статья о встроенных в JS монадических штуках, о которых мало кто знает — Proxy и Reflect
tags: js
categories: js
disqus_comments: true
related_posts: true

---

## Введение

Вообще, `Proxy` и `Reflect` используются довольно редко. Но мы тут учим JS довольно глубоко, правда? Давайте разбираться.

### Proxy

`Proxy` — это, как может быть понятно из названия, такая штука, которая оборачивается вокруг другого объекта и перехватывает (и может модифицировать) любые действия с ним. Вообще, `Proxy` — тоже объект, но несколько уникальный — собственных свойств у него нет.

Давайте посмотрим на простой пример:

```ts
const obj = { name: "Name Surname", age: 34 };

const proxy = new Proxy(obj, {
  get(target, prop) { // перехватываем чтение свойства
    if (prop in target) { // если свойство есть
      return target[prop]; // возвращаем его
    } else {
      // иначе возвращаем непереведённую фразу
      return 'no such property';
    }
  }
}
);

// обратимся напрямую к объекту
console.log(obj.age); // 34
console.log(obj.profession) // undefined

// а теперь обратимся к тому же объекту через Proxy

// такое свойство у объекта obj есть, оно и вернется (34)
console.log(proxy.age); // такое свойство у объекта obj есть, оно и вернется (34)

// а такого свойства у объекта obj нет, но мы получим не undefined, а строку 'no such property'
console.log(proxy.profession);
```

Не то чтобы применимо, но позволяет устанавливать дефолтные значения для несуществующих объектов.
Давайте перехватим добавление значения — к примеру, создадим массив, в котором могут быть только строки:

```ts
let strings = [];
strings = new Proxy(strings, { 
  set(target, prop, val) { 
      if (typeof val == 'string') {
        target[prop] = val;
        return true;
      } else {
        return false;
      }
    }
});


strings.push(1); // true;
console.log(strings); [ 'string' ] // строка добавилась в массив

strings.push(1); // TypeError
strings.push({}) // TypeError

console.log(strings); [ 'string' ] // массив не изменился, правда, код не дойдет сюда и упадет с TypeError
```

Приятная функциональность? Да, но падения кода с TypeError — такое себе. Как такое исключить, мы рассмотрим в главе о `Reflect`.

Давайте разберем еще один пример, связанный с перечислениями:

```ts
let product = {
  productName: 'RedBull',
  price: '30', // cents,
  _quantityInStock: 150,
};

product = new Proxy(product, {
  ownKeys(target) {
    return Object.keys(target).filter(key => !key.startsWith('_'));
  }
});

/// мы не увидим здесь _quantityInStock, потому что исключили ключи, начинающиеся с _
for(let key in product) {
  console.log(key);
}

// то же самое для этих методов:
console.log(Object.keys(product)); // productName, price
console.log(Object.values(product)); // 'RedBull', 30
```

Как мы видим, применений у `Proxy` достаточно. А о нем даже большая часть опытных разработчиков не знает (как и о `Reflect`, который мы в следующей главе разберем).

### Reflect

`Reflect` — это тоже встроенный объект, который упрощает работу с `Proxy`. Основное отличие: с `Reflect` мы можем обращаться к внутренним методам типа `[[Get]]` и `[[Set]]`. Что это такое? Для большинства действий с объектами в спецификации JavaScript есть так называемый «внутренний метод», который на самом низком уровне описывает, как его выполнять. Например, `[[Get]]` – внутренний метод для чтения свойства, `[[Set]]` – для записи свойства, и так далее. Эти методы используются только в спецификации, мы не можем обратиться напрямую к ним по имени.

А вот Объект Reflect делает это возможным. Его методы – минимальные обёртки вокруг внутренних методов.

Простой пример (хоть и не очень полезный):

```ts
let obj = {};

Reflect.set(obj, 'testProp', 1);

alert(obj.testProp); // 1
```

Видите? Мы используем `[[Set]]` напрямую. Для чего это может быть нужно? Ну, к примеру, давайте перепишем тот пример, который в случае с `Proxy` падает с `TypeError`:

```ts
const strings = {};

const stringsProxy = new Proxy(strings, {
  set(target, prop, val, receiver) {
    try {
      if (typeof val == 'string') {
        return Reflect.set(target, prop, val, receiver);
      }
    } catch (err) {
      console.log('поймали ошибочку', err);
    }
    
  }
});

stringsProxy.val1 = 'string'; // true
console.log(strings); // { val1: 'string'} — строковое значение добавилось в объеет

// теперь мы не падаем с TypeError, просто возвращаем false и ничего в массив не добавляем
stringsProxy.val2 = 1; // false
stringsProxy.val3 = {}; // false

console.log(strings); //{ val1: 'string'} — объект не изменился, потому что мы пытались добавить нестроковые значения
```

## Вывод

Да, `Proxy` и `Reflect` непопулярны. Но они могут и промисы заменить, и генераторы, и вообще всякое могут. Кстати, `Reflect` и `Proxy` — вполне себе монадические структуры. О монадах — [здесь](https://sptm.dev/2023/monads-in.js/). Stay tuned!

