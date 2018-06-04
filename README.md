# wheel-mvvm
# 一个简单的mvvm框架

### Object.defineProperty()
```
在介绍框架之前先说一下Object.defineProperty()的用法
Object.defineProperty(obj, prop, descriptor)
obj: 目标对象
prop: 要定义或者修改的属性名称
descriptor: 将被定义或修改的属性描述符
看个用法：
var obj = {}
Object.defineProperty(obj, 'intro', {
    enumerable: true //设置该属性可被遍历到
    configurable: true ////设置该属性可被修改、删除
    value : 'hello world'
})
其中enumerable的值如果是false就无法被遍历到，configurable的值如果是false就无法被删除、修改
value：数据描述符
还有一个数据描述符是writable默认是fasle无法通过赋值运算符修改和configurable的区别是前者设置属性能修改，后者设置属性能删除
var obj = {}
var age
Object.defineProperty(obj, 'age', {
    get: function(){
        console.log('get age...')
        return age
    },
    set: function(val){
        console.log('set age...')
        age = val
    }
})
get和set叫存取描述符
get: 给属性提供一个getter的方法，如果没有getter则为undefined。该方法返回值被当作属性的值，默认为undefined
set: 给属性提供一个setter的方法，如果没有setter则为undefined。该方法接受唯一参数值，并将该参数的新值赋给该属性。默认是undefined
数据描述符和存取描述符是无法同时存在的
```

### 数据劫持
```
var data = {
  name: 'dade'
}

observe(data)

function observe(data){
 if(!data || typeof data === 'object') return 
 
  for(let key in data){
    let value = data[key]
    Object.defineProperty(data, key, {
    enumerable: true,
    configuration: true,
    get: function(){
        console.log(`get ${value}`)
        return value
    },
    set: function(val){
        console.log(`set ${value}`)
        value = val
    }
  })
  if(typeof value === 'object') {
    observe(value)
  }
 }
}
首先要把data交给observe函数处理，函数会遍历data的属性对每个属性设置enumerable、configuration、get、set，
属性的值通过get返回那么在return之前可以对值进行拦截处理，
通过set设置也可在赋值之前对值进行处理
```
### MVVM框架
其中用到了观察者模式，每一个属性就是一个主题，当主题修改时会去通知观察者
M: model也就是数据层，将数据提供给视图层
```
model:
let vm = new mvvm({
	el: '#app',
	data: {
		name: 'jirengu',
		age: 18
	},
	methods: {
		sayHi() {
			alert(`hi ${this.name}`)
		}
	}
})
```
V: view意味着视图，将数据展现给用户
```
view:
<div id="app" >
  <input v-model="name" v-on:click="sayHi" type="text">
  <h1>{{name}} 's age is {{age}}</h1>
</div>
```
VM: viewmodel是将分离的数据层和视图层连接起来的中间层。当我们修改model的时候会同步到view中，
同样修改view那么model也会被修改，vm就是两者之间的桥梁
```
class mvvm { //建立一个mvvm对象
	constructor(opts) {
		this.init(opts)
		observe(this.data)
		new Compile(this) //解析对象中的view和data
	}
	init(opts) {
		this.el = document.querySelector(opts.el)
		this.data = opts.data || {}
		this.methods = opts.methods || {}

		for (let key in this.data) { //进行数据劫持
			Object.defineProperty(this, key, {
				enumerable: true,
				configurable: true,
				get: () => {
					return this.data[key] //等同于 this.vm.$data[this.key]
				},
				set: newVal => {
					this.data[key] = newVal
				}
			})
		}	
		for(let key in this.methods) {
			this.methods[key] = this.methods[key].bind(this)
		}
	}
}
```
解析dom对象
```
class Compile {
	constructor(vm) {
		this.vm = vm
		this.node = vm.el
		this.compile()
	}
	compile() {
		this.traverse(this.node)
	}
	traverse(node) {
		if(node.nodeType === 1) { //nodeType为1表示一个元素节点，有孩子
			this.compileNode(node)
			node.childNodes.forEach(child => {
				this.traverse(child)
			})
		}else if(node.nodeType === 3) { //表示文本
			this.compileText(node)
		}
	}
	compileText(node) { //通过正则去匹配{{name}}
		let reg = /{{(.+?)}}/g
		let match
		while(match = reg.exec(node.nodeValue)) {
			let raw = match[0]
			let key = match[1].trim()
			node.nodeValue = node.nodeValue.replace(raw, this.vm[key])
			new Observer(this.vm, key, function(newVal, oldVal) { //建立观察者，一旦key的value发生变化立即将旧值替换成新值
				node.nodeValue = node.nodeValue.replace(oldVal, newVal)
			})
		}
	}
	compileNode(node) {  //绑定 v-model和事件
		let attrs = [...node.attributes]
		attrs.forEach(attr => {
			if(this.isModelDirective(attr.name)) {
				this.bindModel(node, attr)
			}else if(this.isEventDirective(attr.name)) {
				this.bindEventHandler(node, attr)
			}
		})
	}
	bindModel(node, attr){ //绑定v-model中的值
		let key = attr.value
		node.value = this.vm[key]
		new Observer(this.vm, key, function(newVal) {
			node.value = newVal
		})
		node.oninput = (e) => {
			this.vm[key] = e.target.value
		}
	}
	bindEventHandler(node, attr) {
		let event = attr.name.substr(5)
		let methodName = attr.value
		node.addEventListener(event, this.vm.methods[methodName])
	}
	isModelDirective(attrName) {
		return attrName === 'v-model'
	}
	isEventDirective(attrName) {
		return attrName.indexOf(attrName) === 0
	}
}
```
解析Model
```
let currentObserver = null

function observe(data) {
	if(!data || typeof data !== 'object') return
	for(let key in data) {
		let val = data[key]
		let subject = new Subject() //为每一个属性建立主题
		Object.defineProperty(data, key, {
			enumerable: true,
			configurable: true,
			get: function() {
				if(currentObserver) {
					console.log('notify')
					currentObserver.subscribeTo(subject) //订阅主题
				}
				return val
			},
			set: function(newVal) {
				val = newVal
				subject.notify() //当主题发生变化时通知观察者
			}
		})
		if (typeof val === 'object') {
			observe(val)
		}
	}
}
```

观察者
```
class Observer {
	constructor(vm, key, cb) {
		this.subjects = {}
		this.vm = vm
		this.key = key
		this.cb = cb
		this.value = this.getValue()
	}
	update() {
		let oldVal = this.value
		let value = this.getValue()
		if(value !== oldVal){
			this.value = value
			this.cb.bind(this.vm)(value, oldVal)
		}
	}
	subscribeTo(subject) {
		if(!this.subjects[subject.id]) {
			subject.addObserver(this)
			this.subjects[subject.id] = subject
		}
	}
	getValue() {
		currentObserver = this
		let value = this.vm[this.key]
		currentObserver = null
		return value
	}
}
```
发布者
```
let id = 0

class Subject {
	constructor() {
		this.id = id++
		this.observers = []
	}
	addObserver(observer) {
		this.observers.push(observer)
	}
	removeObserver(observer) {
		 let index = this.observers.indexOf(observer)
		 if(index > -1) {
		 	this.observers.splice(index, 1)
		 }
	}
	notify() {
		this.observers.forEach(observer => {
			observer.update()
		})
	}
}
```
