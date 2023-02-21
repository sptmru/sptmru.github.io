---
layout: post
title:  "Inversion of Control и Dependency Injection — что и зачем. Максимально коротко."
date:   2023-02-20 16:31:43 +0400
desciption: Статья об IoC и DI — зачем, что это и как, все с примерами и предельно понятно.
tags: theory
categories: theory

---

## Введение

Вообще, я уже писал о принципах проектирования (SOLID, GRASP, вот эти вот штуки) [здесь](https://sptm.dev/2023/solid-grasp-and-stuff/). Но принципы, которые я опишу здесь, вполне заслуживают отдельной статьи.

Dependency Injection и Inversion of Control — штуки связанные, но это не одно и то же. Можно считать, что Dependency Injection — это реализация Inversion of Control. Поэтому с IoC и начнем.

## Inversion of Control

Мы уже знакомы с такой штукой в ООП, как связывание (coupling). И даже знаем, что, согласно GRASP (да и SOLID тоже, смотрим буквы S и I), связывание в приложении должно быть низким. Это значит, что классы (объекты) должны быть как можно более независимы друг от друга. Об этом говорит и Single Responsible Principle, и Open-Closed Principle, и принцип подстановки Лисков — да, в общем-то, весь SOLID об этом разными словами рассказывает.

Так вот, Inversion of Control помогает этого достичь. Вообще, IoC — это такая больше теоретическая идея, суть которой в том, что класс свои зависимости не подтягивает сам, а мы ему их подсовываем, таким образом увеличивая гибкость этого самого класса, позволяя ему работать с разными типами зависимостей.

То есть, мы можем сделать так:

```ts
class Car {
  constructor() {
    this.engine = new Engine;
    this.transmission = new Transmission;
    this.chassis = new Chassis;
  }
}
```

В чем проблема такого подхода? Класс Car тесно связан с конкретными классами `Engine`, `Transmission` и `Chassis`. Если мы захотим использовать другие классы в качестве зависимостей, нам придется лепить другой класс:

```ts
class ElectroCar {
  constructor() {
    this.engine = new ElectroEngine();
    this.transmission = new Transmission();
    this.chassis = new Chassis();
  }
}
```

Идея не очень, правда? И связывание ощутимо увеличивается. Тут мы плавно переходим к паттерну Dependency Injection.

## Dependency Injection

А что, если не класс будет подтягивать зависимости изнутри, а мы будем подсовывать ему нужные нам классы? Получится интереснее:

```ts
class Engine{...};
class ElectroEngine {...};

class Transmission {...};
class ElectroTransmission {...};

class Chassis {...};
class ElectroChassis {...}

class Car {
  constructor(engine, transmission, chassis) {
    this.engine = engine;
    this.transmission = transmission;
    this.chassis = chassis;
  }

  const car = new Car(new Engine(), new Transmission(), new Chassis());
  const electroCar = new Car(new ElectroEngine(), new ElectroTransmission(), new ElectroChassis());
}
```

Гораздо удобнее, правда? Связность уменьшилась: теперь `Car` не зависит напрямую от каких-то классов — мы сами решаем, что в него прокинуть. Это и есть Inversion of Control, реализованная с помощью Dependency Injection.

## Дополнение

Часто фреймворки берут на себя эти удобства, и это становится еще красивее. К фреймворкам мы привязываться здесь не будем — эта статья объясняет принцип, не касаясь реализации.

## Выводы

А вывод здесь один — используйте, пожалуйста, Dependency of Injection, не надо увеличивать связность. Еще хочется сказать, что зависимости можно мокать, таким образом сильно облегчая юнит-тестирование наших классов. Но о тестировании — обязательно, но чуть позже :)
