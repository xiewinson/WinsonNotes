# JavaScript 的 This

1. __全局范围的 this__
当 `this` 在任何函数之外调用时，即全局环境中被调用，在浏览器中默认指向 `Window` 对象
```JavaScript
console.log(this)
// 输出 Window {postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, frames: Window, …}
```

2. __this 在对象的构造函数中__

`this` 指向由这个构造函数创造的实例
```JavaScript
function Person(name){
	this.name = name;
}
var p = new Person('p')
console.log(p.name) // 输出 p 
```

3. __this 在对象的方法内__

与对象所关联的函数一般称之为方法，`this` 指向的是对象本身
```JavaScript
class Student {
	say(){
		console.log(this);
	}
}
var s = new Student()
s.say() // 输出对象本身
```

4. __this 在简单函数中__

浏览器中，简单函数的 `this` 总是指向 `Window`，就算在对象方法中调用简单函数。
```JavaScript
function logThis(){
	console.log(this.pName)
}

var p = {
	pName: 'p',
	say(){
		console.log(this.pName);
		logThis();
	}
}
p.say() // 第一行输出 p，第二行输出 undefined
```

5. __this 在箭头函数中__

箭头函数是本身没有执行的上下文，没有本身的 `this`， 是找离它最近的一个执行环境作为执行上下文，即它由外围最近一层非箭头函数决定。
```JavaScript
var q = {
	say(){
		setTimeout(function(){
			console.log(this)
			this.log()
		}
		,1000)
	},
	log(){
		console.log('qqqq');
	}
}
q.say() 
// 首先会输出这个 this 是 Window
// 然后会抛出异常：Uncaught TypeError: this.log is not a function

var q = {
	say(){
		setTimeout(()=>{
			console.log(this)
			this.log()
		}
		,1000)
	},
	log(){
		console.log('qqqq');
	}
}
q.say()
// 正确，先输出 this 是当前对象，然后输出 'qqqq'

// 我们再试试不加 this，直接调用这个 log 方法
var q = {
	say(){
		setTimeout(()=>{
			console.log(this)
			log()
		}
		,1000)
	},
	log(){
		console.log('qqqq');
	}
}
q.say() // 抛出异常，Uncaught ReferenceError: log is not defined at setTimeout 

var q = {
	x:'哈哈哈哈',
	say: () => console.log(this.x),
}

q.say() // 输出 undefined，因为此处箭头函数最近的作用域是 `Window` 函数
```
6. __使用 call/apply/bind/ 改变 this 指向__

这种情况下 `this` 指向的对象就是绑定的对象
```JavaScript
var p = {
	x: 10
}

function foo(){
	console.log(this.x);
}

foo() // 输出 undefined

foo.call(p) // 输出 10

foo.apply(p) // 输出 10

foo.bind(p)() // 输出 10
```