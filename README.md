#自制一个mvvm框架
Vue.js 可以说是MVVM 架构的最佳实践，专注于 MVVM 中的 ViewModel，不仅做到了数据双向绑定，而且也是一款相对比较轻量级的JS 库，API 简洁，很容易上手。
Vue.js 是采用 Object.defineProperty 的 getter 和 setter，并结合观察者模式来实现数据绑定的。当把一个普通 Javascript 对象传给 Vue 实例来作为它的 data 选项时，Vue 将遍历它的属性，用 Object.defineProperty 将它们转为 getter/setter。用户看不到 getter/setter，但是在内部它们让 Vue 追踪依赖，在属性被访问和修改时通知变化。
注：观察者模式又叫做发布订阅模式，它定义了一种一对多的关系，让多个观察者对象同时监听某一个主题对象，这个主题对象的状态发生改变时就会通知所有观察着对象。
假设有如下代码，data 里的name会和视图中的{{name}}一一映射，修改 data 里的值，会直接引起视图中对应数据的变化。

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>自制一个mvvm框架</title>
</head>
<body>

<div id="app" >
    <h1>{{name}}是我的名字，我今年 {{age}}岁了</h1>
    <h2>输入名字：</h2>
    <input v-model="name"  type="text">
    <button v-on:click=sayhi>点我哦</button>
</div>
 function mvvm(){
        //todo...
    }

 let vm = new mvvm({
        el: '#app',
        data: {
            name: 'caixiaoting',
            age: 24
        },
        methods: {
            sayhi() {
                alert(`hi ${this.name}`)
            }
        }
    })
```

如何实现上述 mvvm 呢？

我们可以先参考以下vue的mvvm实现方法

![mei](https://www.tinnypea.com/2018/09/implement-a-simple-mvvm-framework-based-on-the-understanding-of-vue/mvvm.png)
```
observer：数据监听，由observer对数据模型使用Object.defineProperty对数据的get、set方法进行劫持，设置数据时通知订阅者(watcher)更新，获取数据时绑定订阅者，
compile：模板/指令编译器，将html文件中的指令(v-model,@clickj,{{}})等特定规则的字符串解析成mvvm中的数据，并且将其添加为一个观察者，同时执行具体的dom更新、事件绑定等
watcher：订阅者，模板/数据观察者，一旦观察到了数据的变动，则进行数据更新
dep：消息订阅器,连接compile与watcher，在数据劫持中绑定watcher
```

其实就是一方面先进行模版解析（complie）一方面进行数据劫持（observer）再由mvvm进行整合

那么一个简单的mvvm框架就可以写出来了，如下：
```
  class mvvm {
        constructor(opts) {
            this.init(opts)//一上来先把可用的东西都挂载在实例上
            observe(this.$data)//数据劫持
            new Compile(this)//对数据和元素进行编译
        }
```
我们将以上分为三步处理，先看第一步，将可用的东西都挂载在实例上，如数据，方法
```
init(opts){
          //获取当前的可用的东西
            this.$el = document.querySelector(opts.el)
            this.$data = opts.data || {}
            this.$methods = opts.methods || {}

            //把$data 中的数据直接代理到当前 vm 对象
            for(let key in this.$data) {
                Object.defineProperty(this, key, {
                    enumerable: true,
                    configurable: true,
                    get: ()=> {  //这里用了箭头函数，所有里面的 this 就指代外面的 this 也就是 vm
                        return this.$data[key]
                    },
                    set: newVal=> {
                        this.$data[key] = newVal
                    }
                })
            }

            //让 this.$methods 里面的函数中的 this，都指向当前的 this，也就是 vm
            for(let key in this.$methods) {
                this.$methods[key] = this.$methods[key].bind(this)
            }
        }

    }
```
第二步就是 数据劫持
```
//先写好数据监听器
function observe(data) {
        if(!data || typeof data !== 'object') return
        for(var key in data) {
            let val = data[key]
            let subject = new Subject()//new一个订阅者即上图的dep消息订阅器
            Object.defineProperty(data, key, {
                enumerable: true,//可以遍历
                configurable: true,//可以删除
                get: function() {
                    if(currentObserver){
                        currentObserver.subscribeTo(subject)
                    }
                    return val
                },
                set: function(newVal) {
                    val = newVal
                    console.log('start notify...')
                    subject.notify()
                }
            })
            if(typeof val === 'object'){
                observe(val)
            }
        }
    }

    let id = 0
    let currentObserver = null

    class Subject {
        constructor() {
            this.id = id++
            this.observers = []
        }
        addObserver(observer) {
            this.observers.push(observer)
        }
        removeObserver(observer) {
            var index = this.observers.indexOf(observer)
            if(index > -1){
                this.observers.splice(index, 1)
            }
        }
        notify() {
            this.observers.forEach(observer=> {
                observer.update()
            })
        }
    }

    class Observer{
        constructor(vm, key, cb) {
            this.subjects = {}
            this.vm = vm
            this.key = key
            this.cb = cb
            this.value = this.getValue()
        }
        update(){
            let oldVal = this.value
            let value = this.getValue()
            if(value !== oldVal) {
                this.value = value
                this.cb.bind(this.vm)(value, oldVal)
            }
        }
        subscribeTo(subject) {
            if(!this.subjects[subject.id]){
                console.log('subscribeTo.. ', subject)
                subject.addObserver(this)
                this.subjects[subject.id] = subject
            }
        }
        getValue(){
            currentObserver = this
            let value = this.vm[this.key]   //等同于 this.vm.$data[this.key]
            currentObserver = null
            return value
        }
    }
```
第三步：对元素和文本编译
```
    class Compile {
        constructor(vm){
            this.vm = vm
            this.node = vm.$el
            this.compile()
        }
        compile(){
            this.traverse(this.node)
        }
        traverse(node){//判断是不是节点元素
            if(node.nodeType === 1){//1是元素
                this.compileNode(node)   //解析节点上的v-bind 属性
                node.childNodes.forEach(childNode=>{
                    this.traverse(childNode)
                })
            }else if(node.nodeType === 3){ //处理文本
                this.compileText(node)
            }
        }
        compileText(node){
            let reg = /{{(.+?)}}/g
            let match
            console.log(node)
            while(match = reg.exec(node.nodeValue)){
                let raw = match[0]
                let key = match[1].trim()
                node.nodeValue = node.nodeValue.replace(raw, this.vm[key])
                new Observer(this.vm, key, function(val, oldVal){
                    node.nodeValue = node.nodeValue.replace(oldVal, val)
                })
            }
        }

        //处理指令
        compileNode(node){
            let attrs = [...node.attributes] //类数组对象转换成数组，也可用其他方法
            attrs.forEach(attr=>{
                //attr 是个对象，attr.name 是属性的名字如 v-model， attr.value 是对应的值，如 name
                if(this.isModelDirective(attr.name)){
                    this.bindModel(node, attr)
                }else if(this.isEventDirective(attr.name)){
                    this.bindEventHander(node, attr)
                }
            })
        }
        bindModel(node, attr){
            let key = attr.value       //attr.value === 'name'
            node.value = this.vm[key]
            new Observer(this.vm, key, function(newVal){
                node.value = newVal
            })
            node.oninput = (e)=>{
                this.vm[key] = e.target.value  //因为是箭头函数，所以这里的 this 是 compile 对象
            }
        }
        bindEventHander(node, attr){       //attr.name === 'v-on:click', attr.value === 'sayHi'
            let eventType = attr.name.substr(5)       // click
            let methodName = attr.value
            node.addEventListener(eventType, this.vm.$methods[methodName])
        }

        //判断属性名是否是指令
        isModelDirective(attrName){
            return attrName === 'v-model'
        }

        isEventDirective(attrName){
            return attrName.indexOf('v-on') === 0
        }

    }

```

做完上述之后，我们可以看下效果:
https://hello-tsai.github.io/myBookmarks/mvvm.html
源码：
https://github.com/hello-tsai/myBookmarks/blob/master/mvvm.html
