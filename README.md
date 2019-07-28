# demoMVVM
MVVM 框架的全称是 Model-View-ViewModel,它是 MVC（Model-View-Controller）的变种。在 MVC 框架中，负责数据源的模型（Model）可以直接和视图进行交互，而根据软件工程的模块解耦的原则之一，需要将数据和视图分离，开发者只需关心数据，而将视图 DOM 封装，当数据变化时，可以及时地将表现层面对应的数据也同步更新，如下图：
![Vue 的 MVVM 基本框架](https://upload-images.jianshu.io/upload_images/16798642-8cb0c0330d1bac96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
要实现上述框架，首先将数据要封装起来。当 Model 中的数据更改的时候，需要将该数据改变反应出来，也就是——数据劫持。

### 第一步——实现数据劫持
实现数据劫持，我们需要用到一个 Object 对象的核心方法：`Object.defineProperty` ，在 MDN 的定义中，它是这样的：
>Object.defineProperty 会在对象上定义一个新属性，或者修改一个对象的已有属性，并将这个对象返回。

它可以实现在一个对象（也就是数据 Model）上定义一个新属性，或修改已有属性（数据修改），并且将对象返回。这就实现了我们基本的对数据改变后返回通知的需求（通过相应方法）。
```
Object.defineProperty(obj, prop, descriptor)
```

三个参数中，分别是：需要修改的对象，需要修改对象中的某个属性，对该属性的描述。因此在实际应用中，如果需要对某个对象的全部属性进行劫持，则需要用类似`for-in`循环、`Object.keys`等枚举的方法。
```
// exp1
var dataObj = {
  name: 'fejv',
  age: 22,
  skill:['javascript', 'html', 'CSS', 'ES6']
}

// 劫持函数
function observe(data) {
  if(!data|| typeof data !== 'object') return ;
  for(let key in data) {
    let val = data[key];
    Object.defineProperty(data, key,{
      enumerable: true,
      configurable: true,
      get:function() {      
        console.log('get fun '+val);
        return val;
      },
      set:function(newVal) {
        console.log('new value: '+newVal);
        val = newVal;
      }
    });
    if(typeof val === 'object') {
      observe(val);
    }
  }
}

observe(dataObj);
/* 
// 绑定测试
>dataObj.name = 'jab';
>"new value: jab"
>"jab"
>dataObj.name
>"jab"
*/
```
上面代码中，设置了一个简单的数据模型 `dataObj`，它有3个属性，每一个属性的变动都需要被观察（劫持）。因此我们在观察函数 `observe` 中使用了 `for-in` 循环，将所有在`dataObj`中的属性都进行了观察。在修改了 `dataObj.name`后，调用了`set`函数，将对象`dataObj`的值修改为了新的值，实现了对象中属性值的绑定。
[实现一个数据劫持](http://js.jirengu.com/yihoyolona/1/edit)

### 第二步——实现发布订阅模式
在实现了对数据源（Model）的数据劫持后，我们需要能够将变化通知到视图（View），因此运用到了 javaScript 设计模式中的“发布——订阅模式”。发布的角色——被订阅的频道，就是数据源（Model）中的数据，它将数据的变化发布出去；而订阅者的角色就是（View），它订阅数据源的变化，并且根据变化的数据改变自己的视图。
首先，我们要实现一个被订阅者，也就是一个频道。它需要具有发布消息；可以增加/删除订阅者到列表，并且在发布更新的时候需要将更新的内容发布出去。
```
// exp2
class Subject {
  constructor(){
    this.observers = [];  // 订阅者列表
  }
  addObserver(observer) {
    this.observers.push(observer);  // 增加订阅者
  }
  removeObserver(observer) {
    var index = this.observers.indexOf(observer);
    if(index > -1) {
      this.observers.splice(index, 1);  // 删除订阅者
    }
  }
  notify(msg){
    this.observers.forEach( observer => {
      observer.update(msg);  // 发布更新
    });
  }
}
```
该频道由一个基本的订阅者数组组成，具有增加/删除订阅者的功能，并且在发布新消息之后用 `notify(msg)` 函数发布到全部的订阅者（观察者）`observer`中。我们还需要一个观察者的原型：
```
// exp3
class Observer {
  constructor(name) {
    this.name = name;
  }
  update(msg) {
    console.log(this.name+' update: '+ msg);
  }
  subscribe(sub) {
   sub.addObserver(this);
  }
}
```
在上面的观察者（订阅者）中，使用了 ES6 的写法，观察者主要的更新函数和订阅函数写了出来，当使用`Observer`构造函数生成一个新的 `Observer` 对象之后，执行该对象中的订阅函数 `subscribe` 才会将其增加到生成的频道中，当该频道更新，会将更新发布到曾订阅过他的函数中。
```
// exp4
>var subA =  new Subject;
>var fejv = new Observer('fejv');
>fejv.subscribe(subA);
>subA.notify('subA initial version');
>"fejv update: subA initial version"
>subA.notify("version 2");
>"fejv update: version 2"
```
观察者模式或者发布订阅模式的两种写法
[原型写法](http://js.jirengu.com/calozupixa/1/edit)
[ES6写法](http://js.jirengu.com/suvaroheza/1/edit?html,js,console,output)

### 第三步——实现数据的单向绑定
实现数据的观察者模式（发布——订阅模式）之后，我们需要结合数据劫持和发布订阅模式，将数据劫持中，劫持的数据变化发布到所有的对应的订阅者，在 MVVM 中就是将变化的数据劫持反应到 View 的网页模板中。

借鉴 Vue 的模板语法：
```
<div id="app" >
  <h1>{{name}} 's age is {{age}}</h1>
</div>
```
我们需要监控 `name` 和 `age` 作为变量的数据源（Model）中的变化，并且在即使反应到视图 View 上，因此我们要解析 html 模板，取得变量，将每个变量观察起来，当它产生变化的时候，将原本数据源的数据一并修改并且反应到视图中。

首先需要一个入口文件
```
// exp5
class MVVM {
  constructor(opts) {
    this.data = opts.data;
    this.node = document.querySelector(opts.node);
    this.observers = [];
    observe(this.data);
    this.compile(this.node);
  }
  
  compile(node) {
    console.log('compile fun in class MVVM');
    if(node.nodeType === 1) {  
        // 节点仍是 DOM 结构，继续解析子节点
      node.childNodes.forEach(childNode => {
        this.compile(childNode);
      });
    }else if(node.nodeType === 3) {
      this.renderText(node); // 已解析到文字
    }
  }
  
  // 匹配函数，匹配模板语言中的 name 和 age 变量
  renderText(node) {
    console.log('render fun in class MVVM');
    let reg = /{{(.+?)}}/g;
    let match;
    while(match = reg.exec(node.nodeValue)) {
      let sample = match[0];
      let key = match[1].trim();
      // console.log(sample,key);
      node.nodeValue = node.nodeValue.replace(sample, this.data[key]);
      new Observer(this, key, function(newVal, oldVal) {
        node.nodeValue = node.nodeValue.replace(oldVal, newVal);
      });
    }
  }
}
/*
let demoMVVM = new MVVM({
  node: '#app',
  data:{
    name: 'fejv',
    age: 23
  }
});
*/
```
该入口函数中，先将数据源观察起来（数据劫持），以便在数据有更改的时候即使通知到各数据视图，然后解析 HTML 中的节点，将解析后的节点换成相应的值，也就是`demo.data.name/age`中的值。
 
这是初始化的时候，将数据中的值换到 HTML 的 DOM 模板上，但是当数据源的值改变时我们，需要及时地将更改的值换到 HTML 页面中，就是：
```
// exp6
new Observer(this, key, function(newVal, oldVal) {
  node.nodeValue = node.nodeValue.replace(oldVal, newVal);
});
```
这需要配合在 `Observer class ` 的 `updata` 函数中：
```
// exp7
class Observer {
  // ....
  update() {
    var oldVal = this.val;
    var newVal = this.getVal();
    if(oldVal !== newVal) {
      this.val = newVal;
      this.callback.bind(this.vm)(oldVal, newVal)
    }
  }
}
```
在新的 `update` 函数中，将上一次的值设置为旧值，最新的值需要调用 `getVal` 函数获取，然后将新旧值一并传入回调函数中，由回调函数执行将新知更换，`getVal` 函数就成了获取新值的关键。
```
// exp8
class Observer {
  //....
  getVal() {
    console.log('getVal fun in class Observer');
    currentObs = this;
    let val = this.vm.data[this.key];  
    // 在获取vm.data 的值的时候，会调用observe函数中的 get 函数
    currentObs = null;
    return val;
  }
}
```
由于数值的更改是在 sujects 对所有 observers 发出的，因此需要在调用 `Observer` 中的 `get` 函数时，将该观察者（`observer`）添加到`sujects`的列表中。但在`observer`函数中无法访问`Observer` 对象，因此上面代码中，将当前的`Observer`赋值给一个全局的`currentObs`，并在调用`observe` 函数中的`get`函数时，将这个全局的，也就是当前的`Observe`添加到`subject`频道中，当下次有值更新的时候，才能`notify`到相关的`Observer`。
```
// exp9
function observe(data) {
  //...
  Object.defineProperty(data, key, {
    //...
    get:function() {
      if(currentObs) {
        console.log('get fun in observe fun,current observer is not null');
        currentObs.subscribeTo(subj);
      }
      return val;
    }
  });
}
```
配合相关的上述函数，加以修改，就实现了简单的单向绑定。
[单向绑定的ES6写法](http://js.jirengu.com/qikurebiti/1/edit?html,js)

### 第四步——双向绑定的实现
双向绑定，就是在第三步单向绑定的基础上，数据流从 Model => ViewModel => View，增加到Model <=> ViewModel <=> View，也就是视图中的可以改变数据，改变可以反馈到数据源中，再从数据源反馈到表现的视图中。
![双向绑定示意图](https://upload-images.jianshu.io/upload_images/16798642-33b19a9b4b1a3fbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

跟单向绑定的区别就是在于：
1. 需要监控视图上（View）输入的值，作为数据源（Model）更改的来源；
2. 实现初步的视图上的对事件进行绑定，例如 Vue 中的`v-on: click`等语法。

因此需要在模板语言上有一个输入框：
```
// exp10
  <div id="app">
    <input v-model="name" v-on:click="hello">
    <input v-model="age" >
    <h3>{{name}}'s age is: {{age}}</h3>
  </div>
```
上述代码中参考了 Vue 框架的 HTML 语言模板，用两个输入框作为`name`和`age`的数值的双向绑定，而`v-on:click`作为事件进行绑定，在 JS 中需要修改新的解析函数，判断是作为模型或者是指令，并且绑定该输入框。
```
// exp11
class MVVM {
  //...
  // 处理模板节点
  compileNode(node) {
    let attrsArr = Array.from(node.attributes);
    attrsArr.forEach(attr => {
      if(this.isModel(attr.name)) {
        this.bindModel(node, attr);  // 绑定数据
      }else if(this.isHandle(attr.name)) {
        this.bindHandle(node, attr);  // 绑定指令
      }
    });
  }
  // 初始化输入框的值
  bindModel(node, attr) {
    let key = attr.value;
    node.value = this.vm.$data[key];
    new Observer(this.vm, key, function(newVal){
      node.value= newVal;
    });
    
    // 绑定输入的值作为数据源
    node.oninput = (e) => {
      this.vm.$data[key] = e.target.value;
    };      
  }
  // 解析时间模板，绑定事件。
  bindHandle(node, attr) {
    let startIndex = attr.name.indexOf(':')+1;
    let endIndex = attr.name.length;
    let eventType = attr.name.substring(startIndex, attr.name.length);
    let method = attr.value;
    node.addEventListener(eventType, this.vm.methods[method]);
  }
  
  // 判断数据模板
  isModel(attrName) {
    return (attrName === 'v-model');
  }
  // 判断指令
  isHandle(attrName) {
    return (attrName.indexOf('v-on') > -1);
  }
}
```
上述代码分别对 HTML 中的两个`input`实现了数值和事件的绑定，并且第一步将初始化时候的值作为输入框的初始值，在每个输入框的值改变的时候绑定该事件，并且绑定到上述的 `observe` 的`set`函数中，实现了在视图上修改时，反馈到视图反应中，于是实现了简单的双向绑定。

双向绑定的实现源码和演示地址：
[双向绑定的实现源码](http://js.jirengu.com/sifidakavu/1/edit?html,js)
[MVVM 双向绑定的演示](http://js.jirengu.com/sifidakavu/1)


### 参考阅读
1. [MVVM 框架解析之双向绑定](https://juejin.im/post/5a5f72a0518825732258bce9)，[掘金用户牧云云](https://juejin.im/user/582becad2f301e0059426df2)。

