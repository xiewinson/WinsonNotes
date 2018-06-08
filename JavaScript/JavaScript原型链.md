# JavaScript 原型链

* 首先明确，`Function` 也是一个 `Object`，每个 `Object` 都有 `__proto__` 属性，但其实 `__proto__` 不是一个标准属性在有些浏览器上并没有，而只有 `Function` 有 `prototype` 属性
* `__proto__` 到底指向谁，跟如何创造一个对象有关系：

    *  通过字面量的形式，`__proto__` 指向的是 `Object.prototype`
    ```JavaScript
    var student = {
        name: 'stu'
    }
    student.__proto__ === Object.prototype // 输出 true
    ```
    * 通过声明构造函数，然后 `new` 关键字创造出来的对象， `__proto__` 指向的是构造函数的 `prototype` 对象。每个函数都有一个 `prototype` 属性，而在其中有一个名为 `constructor` 的对象指向了函数本身。最后，由构造函数 `new` 出来的对象有一个 `constructor` 属性，这个 `constructor` 实际上指向的就是构造函数的 `prototype` 中的 `constructor`。回头看第一种字面量创建的方式，和使用 `Object` 作为构造方法创建对象的效果是是一致的，相当于创建了一个继承自 `Object` 的对象
    ```JavaScript
    function A(name){
        this.name = name
    }
    var a = new A('a')
    a.__proto__ === A.prototype // 输出 true
    a.constructor === A.prototype.constructor // 输出 true
    ```
    * 使用 `Object.create(proto, [propertiesObject])` 方法创建对象时，`__proto__` 指向的就是传入的原型对象
    ```JavaScript
    var parent = {
        name: "p"
    }
    var child = Object.create(parent)
    child.__proto__ === parent // 输出 true
    ```
* 所以原型链就是子类存在一个隐藏属性 `__proto__` 指向父类的 `proptype`，父类的 `__proto__` 又指向更高层级父类的 `proptype`，形成了一根链条，在寻找一个实例的方法/属性时，在实例上没找到就沿着链条向上层去寻找
* `__proto__` 只是在一些浏览器上能访问到的，它是一个隐藏属性，开发中不要去使用
* 最后，还是推荐使用 `ES6` 中的新特性去做面向对象相关的操作