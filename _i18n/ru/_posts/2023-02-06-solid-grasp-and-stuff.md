---
layout: post
title:  "SOLID, GRASP и другие принципы разработки"
date:   2023-02-07 08:30:43 +0400
description: Статья о шаблонах и принципах проектирования — больше об ООП, чуть меньше о JS.
tags: [ 'theory', 'js' ]
categories: [ 'theory', 'javascript' ]
languages: [ 'ru' ]

---

Собственно, да, об этом и статья. Конечно, знание этих принципов делает нас, как разработчиков, лучше (а собеседования — проще). Но стоит помнить: мы в Node.js не всегда пишем в чистом ООП-стиле, и эти принципы не всегда имеют изначально заложенный в них смысл в Node.js-разработке.

Но давайте сначала разберемся, о чем речь, а потом будем решать, полезно нам оно или нет.

## SOLID

Когда-то, в начале 2000-x, небезызвестный Роберт Мартин назвал пять основных принципов объектно-ориентированного программирования, а чуть менее известный Майкл Фэзерс составил из них акроним. Давайте разберем акроним обратно и подробнее остановимся на каждой букве.

### S: Single Responsibility Protocol (принцип единственной ответственности)

Принцип звучит примерно так: для каждого класса должно быть определено единственное назначение. Все ресурсы, необходимые для его осуществления, должны быть инкапсулированы в этот класс и подчинены только этой задаче.
Есть еще одна формулировка: "Класс должен иметь одну и только одну причину для изменения".

Если представить себе крайнюю степень нарушения этого принципа, то мы получим антипаттерн "God Object", "божественный объект", то есть класс, который делает сразу все, содержит в себе сразу все и невероятно сложен для изменения, поскольку одно изменение в одной его части может повлиять на другие части класса и на другие классы, его использующие — соотвественно, на все приложение. Конечно, так делать не нужно, в том числе в JS.

Есть одна проблема: если не рассматривать TypeScript, то Node.js не очень-то умеет в инкапсуляцию, приватные свойства и методы просто не существуют в JS и становятся предметом договоренностей (вроде "а давайте именовать приватные свойства и методы, начиная с подчеркивания"). Это не то чтобы проблема, спасибо линтерам, но я счел нужным об этом упомянуть. В остальном — да, давайте писать наши классы (или объекты, если мы не используем ООП) так, чтобы они имели только одно назначение и только одну причину для изменений.

### O: Open-Closed Principle (принцип открытости-закрытости)

Формулируется этот принцип так: "Программные сущности (классы, модули, объекты, функции и так далее) должны быть открыты для расширения, но закрыты для изменения". Собственно, в классическом ООП это значит примерно то, что мы должны предпочитать создание дочерних сущностей для расширения функциональности объекта вместо того, чтобы изменять родительский объект для достижения этой цели.

И да, этот принцип (с поправкой на использование абстрактных интерфейсов и наследования от них вместо наследования от родительского класса) вполне реализуем и имеет смысл в TypeScript. Но даже там гораздо логичнее отказаться от наследования, которое работает достаточно странно и опасно в JS — вместо этого можно использовать композицию, используя принцип "composition over inheritance". Это еще и гораздо более гибко. Смотрите:

```js
// наследование

class Person {
  eat() {
    console.log('I am eating');
  }
  breathe() {
    console.log('I am breathing');
  }
  swim() {
    console.log('I am swimming');
  }
}

class Magician extends Person {
  trick() {
    console.log('I am doing a trick');
  }
}

const liv = new Magician();
const harry = new Magician();

//Liv can:
liv.eat();
liv.breathe();
liv.swim();
liv.trick();
//I am eating
//I am breathing
//I am swimming
//I am doing a trick
//Harry can:
harry.eat();
harry.breathe();
harry.swim();
harry.trick();
//I am eating
//I am breathing
//I am swimming
//I am doing a trick

```

В чем проблема кода выше? Мы не можем сделать так, чтобы Magician не наследовал все методы родительского класса Person. Мы можем сократить количество методов в Person, насоздавать кучу классов под разные нужды...а можем сделать вот так:

```js
// композиция

const eat = function () {
    return {
        eat: () => { console.log('I am eating'); }
    }
}
const breathe = function () {
    return {
        breathe: () => { console.log('I am breathing'); }
    }
}
const swim = function () {
    return {
        swim: () => { console.log('I am swimming'); }
    }
}
const trick = function () {
    return {
        trick: () => { console.log('I am doing a trick'); }
    }
}

const nonEatingMagician = () => {
  return Object.assign(
     {},
     breathe(),
     trick(),
   );
}

const nonSwimmingPerson = () => {
  return Object.assign(
    {},
    eat(),
    breathe(),
  )
}

const nonSwimmingMagician = () => {
  return Object.assign(
    {},
    eat(),
    breathe(),
    trick(),
  )
}
```

Видите, насколько это гибко? И разве же это нарушает суть принципа открытости / закрытости? В общем, что я хочу сказать: предпочитайте композицию наследованию, и вы автоматически будете соблюдать принцип открытости / закрытости.

### L: Liskov Substitution Principle (Принцип подстановки Барбары Лисков)

Барбара Лисков, американская ученая, в далеком 1987 году сформулировала этот принцип подстановки так:
"Пусть q(x) является свойством, верным относительно объектов x некоторого типа T. Тогда q(y) также должно быть верным для объектов y типа S, где S — подтип типа T". И спасибо дядюшке Бобу за то, что он переформулировал это гораздо понятнее:

"Объекты в программе должны быть заменяемыми на экземпляры их подтипов без изменения правильности выполнения программы". Так понятнее, да? Вот только что с этим делать в чистом JS? Как мы можем гарантировать, что объект — подтип другого объекта? Я не вижу применения этому принципу в JS. Зато в TypeScript — без проблем:

```ts
function getArea(shapes: Shape[]) {
    return shapes.reduce(
        (previous, current) => previous + current.area(),
        0
    );
}
```

Функция getArea без проблем будет работать с любым классом, который реализует интерфейс `Shape`.

### I: Interface Segregation Principle (принцип разделения интерфейса)

Тут все просто: чем больше интерфейсов, тем лучше. Вот только есть проблема: в JS нет интерфейсов. С трудом можно натянуть этот принцип на прототипное наследование: чем больше прототипов, тем лучше. Только вот мы уже уговорились забить на наследование в пользу композиции. Поэтому я продемонстрирую этот принцип на TS:

Представим, что у нас есть две доменных сущности: `Rectangle` и `Circle`, реализующих интерфейс `Shape`.
Интерфейс `Shape` требует от наследников реализации метода area(), считающего площадь и явно принадлежащего к бизнес-логике.

```ts
interface Shape {
    area(): number;
}

class Rectangle implements Shape {

    public width: number;
    public height: number;

    public area() {
        return this.width * this.height;
    }
}

class Circle implements Shape {

    public radius: number;

    public area() {
        return this.radius * this.radius * Math.PI;
    }
}
```

Предположим, что нам потребовался метод для сериализации этих сущностей, что относится скорее к архитектуре, чем к бизнес-логике: таким образом, мы не можем просто добавить метод `serialize()` в интерфейс `Shape`, ведь это нарушит принцип единственной ответственности: интерфейс не может отвечать и за бизнес-логику, и за архитектуру. Что мы можем сделать? Добавить больше интерфейсов!

```ts
interface Shape {
    area(): number;
}

interface RectangleInterface {
    width: number;
    height: number;
}

interface CircleInterface {
    radius: number;
}

interface Serializable {
    serialize(): string;
}

class Rectangle implements RectangleInterface, Shape {

    public width: number;
    public height: number;

    public area() {
        return this.width * this.height;
    }
}

class Circle implements CircleInterface, Shape {

    public radius: number;

    public area() {
        return this.radius * this.radius * Math.PI;
    }
}

```

Видите, что произошло? Большее количество интерфейсов позволило нам разделить бизнес-логику и архитектуру. В TS это возможно. В JS — нет.

### D: Dependency Inversion Principle (принцип инверсии зависимости)

Сформулировать этот принцип можно примерно так: "Классы должны зависеть от абстракций, а не от конкретных деталей". Опять! Абстракции в JS? Не слышал о таком. А в TS — без проблем, знай подсовывай интерфейсы классам в качестве зависимостей.

### SOLID: итоги

SOLID очень завязан на ООП, которого в полноценном виде нет в JS. Мы можем (и должны) сверяться с SOLID при написании ООП-кода, но, к примеру, в функциональном стиле программирования (который вполне себе мы можем использовать в JS) нет места большинству принципов SOLID (да всем, кроме первого, наверное). Другое дело — TypeScript, который позволяет и поощряет программирование в ООП-стиле — и для него все эти принципы вполне актуальны.

Рассмотрим набор шаблонов, который ощутимо меньше цепляется за ООП — GRASP.

## GRASP

General responsibility assignment software patterns. Так этот GRASP расшифровывается. Общие шаблоны распределения ответственностей. Заметьте, и в SOLID многое было посвящено распределению ответственностей — заставляет задуматься, насколько это важно в разработке ПО. Ну да ладно.

GRASP — это девять шаблонов распределения ответственностей. Давайте разберем их.

1. Information Expert (Информационный эксперт).

    Информационный эксперт — это штука, связанная с S в SOLID. Звучит описание шаблона примерно так: "Ответственность должна быть назначена тому, кто владеет максимумом информации для исполнения — информационному эксперту". То есть, класс (или объект), владеющий максимумом информации о некой доменной области, должен отвечать и за исполнение задач этой доменной области. По сути, тот же принцип единственной ответственности.

2. Creator (Создатель).

    Если вы слышали о паттерне проектирования "фабрика"/"абстрактная фабрика", то это оно. Точнее, не совсем оно. Этот шаблон отвечает на вопрос "кто (какая фабрика) должен создавать объекты некоторого типа А?". И отвечает он так: назначить объекту B создавать объекты А, если:

      - содержит или агрегирует объекты A;
      - записывает объекты A;
      - активно использует объекты A;
      - обладает данными для создания объектов A;

    Ну, то есть, ответственность за создание объектов A должна быть назначена тому, кто владеет максимумом информации для исполнения. Выходит, что по отношению к объектам A объект B — информационный эксперт.

3. Controller (Контроллер).

    Та самая буква C в паттерне MVC. Штука, которая отвечает за прием запросов от пользователя и делегирование их исполнения соответствующим информационным экспертам.

4. Low Coupling (Низкое связывание).

    Этот шаблон немного связан с буквой I в SOLID (да и с буквой S тоже). Речь о том, что объекты (классы) должны быть слабо связаны и как можно более независимы друг от друга в смысле внесения изменений (в идеале, во вполне достижимом идеале, полностью независимы).

5. High Cohesion (Высокая связность).

    Тут речь о том, что обязанности данного элемента тесно связаны и сфокусированы. Разбиение программ на классы и подсистемы является примером деятельности, которая увеличивает связность системы.

6. Polymorphism (Полиморфизм)

    Да, тот самый, из ООП. Реализуется в том числе паттернами "Адаптер" и "Стратегия", и в целом говорит нам о том, что разные объекты могут наследоваться от одних и тех же интерфейсов, реализуя их методы по-разному. Помним о буковке I в SOLID.

7. Pure Fabrication (Чистая выдумка).

    О чем здесь речь? Да практически о том, о чем и в шаблоне Creator. Стоит задача: создавать объекты A, не используя средства класса A. Решение: создать Creator (создать создателя, лол) для класса A.

8. Redirection (Перенаправление).

    Хороший пример перенаправления — шаблон "Контроллер", который становится посредником между моделью и представлением в паттерне MVC, таким образом реализуя шаблон Low Coupling, уменьшая связывание между классами.

9. Устойчивость к изменениям (Protected Variations).

    И тут все тоже довольно просто: надо объектам взаимодействовать? Пусть взаимодействуют через специально для этого созданный интерфейс. Пример шаблона Redirection, тоже уменьшает связность.

### GRASP. Выводы

Оно вроде все тоже про ООП, да? Но понятия связности и ответственности есть в любой системе, и мы должны учитывать эти шаблоны при разработке в том числе на JS.

## DRY

Тут все просто: Don't repeat yourself. Не надо копипастить код, это антипаттерн. Выносите несколько раз используемый код в методы. Только если это не нарушает SOLID и GRASP.

## Выводы

ООП-принципы в большинстве своем применимы к JS и полностью применимы к TS. И учить их надо не только для прохождения собесов :)