---
title: JavaScript 原型和原型链(继承)
index_img: /img/js/JavaScript.jpg
tags: JavaScript
date: 2019-08-07 20:20:21
---

原型和原型链是让 JavaScript 在众多语言中稍显独特的一个方面，了解原型和原型链是前端的必备技能了。但是说到原型和原型链，那么距离继承就不是很远了，这篇文章是对原型和原型链的一个总结，中间也会掺杂继承相关的内容。
<!-- more -->

emmmm，那么先从类说起吧~

### 类与原型

#### 类

了解过后端语言(类似: Java、C++)的小伙伴都应该知道，它们的继承都是基于类的继承。

> 类是面向对象（Object Oriented）语言实现信息封装的基础，称为类类型。每个类包含数据说明和一组操作数据或传递消息的函数。类的实例称为对象。是描述了一种代码的组织结构形式，一种在软件中对真实世界中问题领域的建模方法。

写法是这样的:

```java
public class Parent {
    private String name;  
    private int id;

    public Parent(String myName, int myId) {
        name = myName;
        id = myId;
    }
    /* something */
}

public class Child extends Parent {
    public Penguin(String myName, int myId) {
        super(myName, myId);
    }
    /* something */
}
```

在 ES6 之后，```class```的语法在 JavaScript 中也得到了支持，只是这个```class```和面向对象语言中的类是不同的。虽说写法差不多，但是最终转化为 ES5 代码之后仍然是通过原型和原型链实现的。

#### 原型

![原型和原型链](/img/js/原型和原型链.png)

上面的图就简单的描述了一个原型和原型链，有 **3** 个概念:

- 构造函数: JS中所有函数都可以作为构造函数，前提是被 new 操作符操作
- 实例: 由构造函数 new 出来的结果
- 原型对象: 构造函数有一个 prototype 属性，这个属性会指向**原型**对象

它们之间的关系在图上也有说明:

- 通过构造函数 ```new``` 出来的是一个实例
- 构造函数有一个 ```prototype``` 属性指向**原型**对象
- 原型对象又有 ```constructor``` 属性指回了构造函数
- 实例对象和原型对象是通过 ```__proto__``` 连接起来的

那么**原型链**就来了———从一个对象中去找一个属性，如果在当前对象中没有找到，那么会通过当前对象的```__proto__```属性一直往上找原型对象上是否有这个属性，直到找到 Object 对象还没有找到该属性，才证明这个属性是不存在，否则只要找到了，那么这个属性就是存在的，从这里可以看出 JS 对象和原型对象的关系就像一条链条一样，这个称之为原型链

### 继承

说完原型链肯定紧跟着就来了继承。JavaScript 的继承就是依靠原型和原型链来维持实现的。

先来几个一般的继承实现:

#### 构造函数继承

```js
function Parent(){
    this.name = 'parent1';
}

Parent.prototype.say = function() { /* something */ }

function Child() {
    Parent.call(this); // apply 也可以 更改this指向
    this.type = 'child1';
}
```

这种方式的缺点是 子类是没有继承父类的 prototype 上的属性和方法。

#### 原型链继承

```js
function Parent(){
    this.name = 'parent2';
}
Parent.prototype.say = function() { /* something */ }
function Child() {
    this.type = 'child2';
}
Child.prototype = new Parent(); // 把父类的实例赋值给子类的prototype，即子类的实例的 __proto__
```

这种方式的缺点是 一旦更改了根据子类创建的某一个实例的某一个继承自原型对象上的属性，那么所有的实例的这个属性都会改变，因为他们公用了一个原型对象。

#### 组合继承

```js
function Parent(){
    this.name = 'parent3';
}

function Child() {
    Parent.call(this); // apply 也可以
    this.type = 'child3';
}
Child.prototype = new Parent();
```

这种方式解决了之前原型链继承和构造函数继承的不足。
但是这种方式执行了**两次**父类的初始化，且父类和子类的 prototype 是同一个。重写了 Child 的原型对象，因此给子类添加原型方法必须在替换原型之后。

#### 寄生组合式继承

```js
function Parent(){
    this.name = 'parent3';
}

function Child() {
    Parent.call(this); // apply 也可以
    this.type = 'child3';
}
// 解决父类和子类prototype是同一个
// const obj = {};
// obj.__proto__ = Parent.prototype;
// Child.prototype = obj;
Child.prototype = Object.create(Parent.prototype);

//  Object.setPrototypeOf(Child.prototype, Parent.prototype);
// 解决子类的 constructor 指向父类
Child.prototype.constructor = Child;

// 此处的 setPrototypeOf 可以获取到父级的 静态方法 到子级，babel转译的时候会看到这样一个优化
Object.setPrototypeOf(Child, Parent);

Child.prototype.fn = function() {};
Child.prototype.attr = '属性值';
```

这是目前在 ES5 时代最好的办法了，但是只能在更改 Child 的 prototype 之后 给子类型原型添加属性和方法。

#### ES6 class继承

ES6 的 class 关键字是真的方便了JS的继承方式。但是这仅仅是一个语法糖，内部的实现仍然是```寄生组合式继承```

```js
// es6 代码
class Parent {
}

class Child extends Parent {
    constructor() {
        super()
    }
}

// 经过babel(v7.5.0)转换成浏览器可解析代码
function _inherits(subClass, superClass) {
  /* something... */
  // 给子级的prototype赋值
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    // 增强对象，弥补因重写原型而失去的默认的 constructor 
    constructor: { value: subClass, writable: true, configurable: true }
  });
  // 指定子级的原型对象为父级，串起一个链
  if (superClass) _setPrototypeOf(subClass, superClass);
}

function _setPrototypeOf(o, p) {
  _setPrototypeOf =
    Object.setPrototypeOf ||
    function _setPrototypeOf(o, p) {
      o.__proto__ = p;
      return o;
    };
  return _setPrototypeOf(o, p);
}
```

原型和原型链是 JavaScript 实现继承的基础，明白其中的关系就能更好的编写出复用性代码~~

最后贴出一张大佬[王福朋](https://www.cnblogs.com/wangfupeng1988/p/3979533.html)整理的一张图，其中描述的是原型到原型链的整体流向。

![原型与原型链](/img/js/原型链.png)
