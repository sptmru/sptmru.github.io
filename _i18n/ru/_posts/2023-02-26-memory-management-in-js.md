---
layout: post
title:  "Управление памятью в JS"
date:   2023-02-26 20:10:43 +0400
description: Статья о cборке мусора в JavaScript — то, что часто остается незаметным
tags: [ 'js' ]
categories: [ 'javascript' ]
languages: [ 'ru' ]
disqus_comments: true
related_posts: true

---

## Введение

JavaScript (а точнее, V8) управляет памятью незаметно для нас, и писать код можно, вообще об этом не заморачиваясь.
Но чем серьезнее проект, тем больше приходится беспокоиться об этом, не допускать утечек памяти и в принципе понимать, как оно работает под капотом.
Об этом я сегодня и постараюсь рассказать. Статья, наверное, получится не совсем для новичков — но и им это может быть интересно.

## Garbage Collector (сборщик мусора)

Вот мы пишем код, создаем функции, объекты, генераторы, [Proxy и Reflect](https://sptm.dev/2023/proxy-and-reflect/), вот это все.
Очевидно, оно занимает оперативную память (если чуточку подробнее, то примитивы хранятся в стеке, а все остальное — в куче, но сейчас это не так важно для нас).
Соответственно, существует некий жизненный цикл памяти. Давайте о нем чуть подробнее.

### Выделение памяти

Каждый раз, когда мы что-то в нашем коде создаем (переменную, функцию, вот это все), JavaScript незаметно для нас выделяет под это кусок памяти.
Собственно, пока это все — дальше мы разберем это подробнее в главе о стеке и куче.

### Использование памяти

Этот процесс намного ближе к нам: создавая что-то, мы в память это пишем, используя — читаем. Да, вот так все просто.

### Освобождение памяти

Это за нас тоже делает JS (V8), и делает он это незаметно для нас. Приятно писать на языках высокого уровня, правда?
Дальше разберем, как и когда он это делает.

## Стек и куча

Выделение памяти — понятие очень абстрактное. А где именно хранятся примитивы, функции, объекты? Есть две таких структуры данных: стек и куча.
Давайте о них подробнее.

### Стек (статическое выделение памяти)

Стек — это, во-первых, такая структура данных. Список элементов, которые организованы и обрабатываются по принципу [LIFO](https://ru.wikipedia.org/wiki/LIFO) — last in, first out (последним пришел, первым ушел).
По этому поводу можно глянуть [мою статью об асинхронности в JS](https://sptm.dev/2023/asynchrony-in-js/), но там не совсем об этом.

А во-вторых, это место, где JS хранит примитивные значения (`string`, `number`, `boolean`, `null`, `undefined`) и ссылки на объекты (на все, что не примитивные значения).
Размер таких данных не изменится, поэтому движку удобно — он выделяет фиксированный объем памяти для каждого значения.
Процесс выделения памяти прямо перед выполнением называется статическим. Кстати, на размер примитивных значений существует нефиксированный лимит данных.
Но то такое, не так важно, как лимит памяти для кучи.

### Куча (динамическое выделение памяти)

Куча, если сильно углубиться, это такое [дерево](https://ru.wikipedia.org/wiki/Дерево_(структура_данных)), но нам это не так важно.
А важно то, что все, что не примитивы, JS хранит в куче. Если для примитивов заранее известен размер, то объекты, функции и все такое фиксированного размера не имеют.
Поэтому JS выделяет память по мере необходимости (динамически), что может порождать утечки памяти (об этом позже).

Давайте набросаем небольшой пример — без кода скучновато:

```ts
const obj = {
    val1: '1',
    val2: '2',
};

// под этот объект JS выделит память в куче — черт его знает, сколько свойств мы добавим / удалим / поменяем позже
// кстати, свойства этого объекта — вполне себе примитивы

const arr1 = [ 'string1', 'string2' ];

// массив — это объект, и под него память тоже выделится в куче

const test1 = 'test1';
const test2 = 2;

// а вот это — примитивы. Они хранятся в стеке. Кстати, интересный факт: примитивы неизменяемы, пофиг, объявлены они как var, let или const
// вместо изменения примитивов JS создает новые.
```

### Ссылки (не те, что в [HTTP](https://sptm.dev/2023/http-in-details/))

Все переменные (пофиг, на что они указывают / что хранят) хранятся в стеке. Примитивы там хранятся в прямом смысле.
С остальным чуть сложнее: в стеке хранятся ссылки на них в куче. В куче порядка нет, поэтому JS и хранит ссылки на них в стеке.

## Концепт достижимости

Достижимость — сверхважное понятие для того, чтобы понять, как работает сборка мусора. Но ничего шибко сложного в этом нет.
Давайте просто: достижимые значения — это те, что находятся в стеке (или ссылка на них находится в стеке).
Примеры корневых достижимых значений: выполняемая в текущий момент функция, ее переменные и параметры, другие функции во вложенной цепочке вызовов (и их параметры и переменные), глобальные значения.
А любое другое значение будет достижимым, если они доступны из корневых достижимых значений.

Давайте по примерам:

```ts
// в obj1 находится ссылка на объект в куче, а свойство prop1, как примитив, хранится в стеке. Оба значения достижимы,
let obj1 = {
    prop1: '1',
}
// обнуляем obj1, и ссылка теряется. Объект obj1 и его свойства становятся недостижимыми
obj1 = null;
```
```ts
// а если так?
let obj1 = {
    prop1: '1',
}
const obj2 = obj1;
obj1 = null;
// хоть мы и обнулили ссылку в obj1, ссылка на этот объекта осталась в obj2, поэтому он все еще достижим
```
#### Взаимосвязанные объекты

Чуть сложнее:
```ts
function combine(val1, val2) {
    val1.prop1 = val2;
    val2.prop2 = val1;

    return {
        prop1: val1,
        prop2: val2,
    }
}

let combinedValues = combine({
    property1: "1"
}, {
    property2: "2"
});
```

Что мы тут сделали? Функция `combine()` пересекает объекты, давая им ссылки друг на друга, возвращая объект со ссылками на два предыдущих.
Сейчас все объекты достижимы.
А если так?

```ts
delete combinedValues.prop1;
delete combinedValues.prop2.val2;
```

И все, у `prop2` входящих ссылок больше нет. Но если не удалять одну из ссылок, все объекты останутся достижимыми.
А что еще прикольного можно сделать?

```ts
combinedValues = null; // обнуляем combinedValues, он недостижим, и вместе с ним недостижимыми становятся его свойства
```

## Сборка мусора

Вообще, сборка мусора — процесс простой: когда объект/переменная недостижима, он очищает занимаемую ими память.
Но проблема есть: однозначно решить, нужна ли выделенная память прямо в момент перехода ее в состояние недостижимости, не выйдет.

Нужны какие-то алгоритмы, и они есть: они не полностью точны, но близки к тому. Два самых популярных таких алгоритма — это подсчет ссылок и алгоритм пометок (mark and sweep).

### Алгоритм подсчета ссылок

Алгоритм супер-простой: он уничтожает объекты, на которые ссылок больше нет. Но есть проблемка: циклические ссылки он обрабатывать не умеет. Смотрите:

```ts
let obj1 = {
  prop1: '1',
};
let obj2 = {
  prop2: '2',
}

obj1.obj2 = obj2;
obj2.obj2 = obj1;
obj1 = null;
obj1 = null;
```

Вроде как мы обнулили `obj1` и `obj2`, в чем проблема признать их недостижимыми и очистить занимаемую ими память?
Доступа к ним уже нет. Да вот только алгоритм, подсчитывающий ссылки, видит циклические ссылки объектов друг на друга и не считает их недостижимыми.

### Mark and Sweep (алгоритм пометок)

Проблему циклических ссылок решает алгоритм пометок. Как он работает? Тоже достаточно просто.
Он проверяет, можно ли получить доступ к обьекту через корневой объект (в Node.js это `global`, в браузере — `window`).
Он помечает недоступные объекты (mark) как недостижимые, а после выметает (sweep) их из памяти.

#### Проблемы алгоритмов очистки памяти

Есть беда: алгоритмы не знают, в какой момент память станет ненужной.
И поэтому приложения на JS могут использовать больше памяти, чем им на самом деле нужно.
Ко всему прочему, в однопоточном JS сборщик мусора не может работать постоянно, иначе наш код работать не будет вообще.
Сборщик мусора запускается с некоторой периодичностью, но этой периодичностью управлять мы не можем.

Для этого и существуют языки с ручным управлением памяти типа С или Rust.
В Rust этот вопрос решен особенно интересно, почитайте [документацию](https://doc.rust-lang.org/book/), если интересно — полезно для эрудиции.

#### Оптимизации алгоритмов очистки памяти

Движки JS стараются оптимизировать очистку памяти как можно эффективнее. Вот некоторые из интересных оптимизаций.

##### Сборка по поколениям (Generational Collection)

Объекты делятся на два набора: старые и новые. Обычно многие объекты живут недолго: создаются, делают свою работу и умирают.
Соответственно, имеет смысл новые объекты проверять чаще. Те, что не проверяются долго, считаются старыми и проверяются реже.

##### Инкрементальная сборка (Incremental collection)

Если объектов много, и сборщик мусора будет их все обходить, это будет долго. Поэтому множество всех объектов делится на части.
Тогда одна большая сборка мусора превращается в несколько небольших, и это меньше блокирует поток (а он у нас один, читаем [это](https://sptm.dev/2023/asynchrony-in-js/)).

##### Сборка в свободное время (Idle-time collection)

А тут все просто: сборщик мусора старается работать тогда, когда процессор наименее загружен.

## Утечки памяти

Давайте коротко разберем то, что делать не нужно, чтобы уменьшить вероятность утечек памяти.

### Глобальные переменные

Не используйте глобальные переменные! Не используйте `var` вместо `let` и `const`!
Это присоединит переменную к глобальному объекту (`window` или `global`) и помешает алгоритму mark-and-sweep очищать занимаемую имм память, и ваше приложение потечет.

### Забытые таймеры

Забытые таймеры не очистятся никогда. Смотрите:

```ts
const object = {};
const interval = setInterval(function() {
  doSomething(object);
}, 2000);
```

Пока `interval` не будет очищен, каждые две секунды будет выполняться `doSomething(object)`.
Не забывайте о `clearInterval`, очищайте интервалы!

## Итоги

Итак, что нужно помнить:

- мы никак не можем повлиять на сборку мусора, она выполняется автоматически
- достижимые объекты занимают драгоценную память
- как мы видели в примерах выше, если на объект есть ссылка, он недостижим — смотрите примеры с взаимоссылающимися объектами

А если хочется мясца про V8 и кишки, гляньте [офигенную статью](https://jayconrod.com/posts/55/a-tour-of-v8-garbage-collection) о деталях работы сборки мусора в V8.

Понимать, как работает garbage collection, важно: если нам требуется низкоуровневая оптимизация, поэтому статейку прочитать полезно,
Я не перевожу ее здесь, чтобы не усложнять статью.
Всем спасибо за внимание, stay tuned!