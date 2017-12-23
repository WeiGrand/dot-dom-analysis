# .dom源码解读

### 前言

.dom 是一款极小(511 byte)的 `虚拟DOM` 模版引擎，分析其源码有助于我们学习 `虚拟DOM` 技术的实现原理。

### 准备工作

clone github仓库

```bash
git clone https://github.com/wavesoft/dot-dom.git
cd dot-dom
```

一般读源码都是由作者的第一次commit开始，此时的代码一般都是实现了基本的功能，没有很多兼容性、逻辑优化的处理，可读性最强。

查看 `commit` 历史

```bash
git log
```

找到最前的 `commit`

```bash
commit 7b743d851446416013c6f5a083228698d181b270 (HEAD)
Author: Ioannis Charalampidis <ioannis.charalampidis@cern.ch>
Date:   Thu Feb 2 02:28:40 2017 +0100

    First version

commit 50a0acaf24e18263860c4a116f5962eaba0121a4
Author: Ioannis Charalampidis <wavesoft@users.noreply.github.com>
Date:   Wed Feb 1 22:09:54 2017 +0100

    Initial commit
```

因为 `Initial commit` 并没有实际的coding，第一版代码是在 `First version` 所以决定 `checkout` 到那里

```bash
git checkout 50a0acaf24e18263860c4a116f5962eaba0121a4
```

可以看到 `dotdom.js` 只有100多行代码

### 正片开始

`dotdom` 由一个IIFE所包围

```javascript
((global, document, Object, vnodeFlag, expandTags, createElement, render) => {
 	//...
})(window, document, Object, Symbol(), 'a.b.button.i.span.div.img.p.h1.h2.h3.h4.table.tr.td.th.ul.ol.li.form.input.select');
```

前三个参数没什么好说的，从第四个参数看起 `vnodeFlag` 是 `dotdom` 所创建的 `虚拟dom` 对象的标识，用于识别一个对象是否是 `虚拟dom` 。[Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) 是 `ES6` 的新增的对象，用于生成唯一的对象属性的标识符。也就是如果

`obj[vnodeFlag] === true` 则 `obj` 就是 `dotdom` 的一个 `虚拟dom`。

```javascript
String.prototype[vnodeFlag] = 1;
```

让字符串能够别识别为 `虚拟dom`。

在 `dotdom` 的实现可以多次看到函数接受的参数个数总是大于实际传递的参数，这样其实是为了防止变量污染，同时起到一个变量声明的作用，从而避免了手写 `var`、 `let`、 `const` 。

纵观整个 `dotdom`， 一共就定义了两个方法，一个是用于创建 `VNODE`，另一个则用于渲染。

### createElement

```javascript
/**
 * 创建VNODE的函数
 */
global.H = createElement = (element, props={}, ...children) => ({
  	[vnodeFlag]: 1,                                                
  	E: element,                                                                                
  	P: props[vnodeFlag] && children.unshift(props) && {C: children} || (props.C = children) && props                               
})
```

`createElement` 返回一个 `VNODE` 对象，除了 `vnodeFlag` 该对象还包括两个属性：

`E` 代表了节点类型，既可以是标签名如 `'div'` ，也可以是一个函数如

```javascript
function Clickable(props, state, setState) {// you can call setState here }
createElement(Clickable);
```

`P` 包含了该虚拟节点的属性值和子元素，这里还判断传入的 `props` 值是否为 `VNODE` 。

这里 `E` 和 `P` 都不是必须的 因为字符串就可以没有。

#### render

```javascript
/**
 * 渲染VNODE 
 * 接受 2个参数 vnode 和 dom
 * @param {VNode} - 由 `createElement` 方法返回的对象（或 字符串）
 * @param {DOMElement} - 原生dom
 * @returns {DOMElement} - 原生dom
 */
global.R = render = (vnode, dom, _render, _element=vnode.E, _props=vnode.P) => {}
```

首先判断 `vnode` 是否为字符串，这里用到的技巧是判断 `vnode` 是否有 `trim` 方法，如果是字符串就用 `createTextNode` 进行。

```javascript
vnode.trim ? dom.appendChild(document.createTextNode(vnode) ) : //...
```

接着要判断 `vnode` 的 `E` 是标签还是函数，函数有 `call` 方法。

```javascript
_element.call ? (_render = (state, _instance) => {} )({}) : //...
```

如果为函数，则执行一个渲染函数 `_render` ，该函数接受一个 `state` 对象，没错，就是类似 `React` 的组件的 `state`, 返回一个 `DOMElement` 实例。

```javascript
_render = (state, _instance) =>  
	_instance = render( 
  		_element( 
          	_props, 
          	state, 
          	(newState) =>  //相对于 React 的 this.setState(newState)
          		dom.replaceChild( 
              		_render( Object.assign( state, newState )), _instance ),
          			_instance
          	)
        ),
  		dom )
```

最后剩下是 `HTML` 标签的情况，也就是最一般的情况：

```javascript
//遍历VNODE.P
Object.keys(_props).reduce((instance, key, index, array, _value=_props[key]) => (
  //key 有 4种情况
  key == 'C' ?  // 子元素
  _value.map((child) => render(child, instance)) // append 到 instance
  : key == 'style' ?  // 样式
  Object.assign(instance[key], _value) // instance.style[key] = _value
  : /^on/.exec(key) ? // 事件/回调
  instance.addEventListener(key.substr(2), _value) //监听
  : instance.setAttribute(key, _value) // 普通的attribute
) && instance || instance //保证返回 instance 用于 reduce 迭代 的 hack
                          //true && instance || instance => instance
                          //false && instance || instance => instance
, dom.appendChild(document.createElement(_element)) // reduce的初始值 
)
```

最后为原生的 `HTML` 标签提供内置的生成 `VNODE` 的方法。

```javascript
expandTags
    .split('.')
    .map(
      (dom) =>
        global[dom] = createElement.bind(global, dom)
    ) // 可以直接使用 div(props, ...children), button(props, ...children)
```
