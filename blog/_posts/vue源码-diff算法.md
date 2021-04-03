---
date: 2021-04-03
tag: 
  - vue源码
author: chengwx
location: ShenZhen  
---

# vue源码

## 响应式系统

### Dep
```js
class Dep {
  constructor() {
    this.subs = [];
  }
  addSub (sub) {
    this.subs.push(sub);
  }
  notify () {
    this.subs.forEach((sub) => {
      sub.update();
    })
  }
}

```
### Watcher

```js
class Watcher {
  constructor() {
    Dep.target = this;
  }
  update () {
    // do something
  }
}
```

### Observer

```js
function observe (value) {
  if (!value || typeof value !== 'object') return;
  Object.keys(value).forEach(key => {
    defineReactive(value, key, value[key])
  })
}
```

```js
function defineReactive (data, key, val) {
  observe(val);
  const dep = new Dep();
  Object.defineProperty(data, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      return val;
    },
    set: function reactiveSetter (newVal) {
      if (newValue === val) return;
      observe(newValue);
      val = newVal;
      Dep.notify();
    }
  })
}
```
```js
function Vue (options) {
  this._data = options.data;
  observe(this._data);
  new Watcher();
}
```

## nextTick
```js
let callbacks = [];
let timerFunc;
function nextTick (cb) {
  callbacks.push(cb);
  timerFunc();
}
function flushCallbacks () {
  callbacks.forEach(cb => cb());
}

if (Promise) {
  timerFunc = () => {
    Promise.resolve().then(flushCallbacks);
  }
} else if (MutationObserver) {
  let observer = new MutationObserver(flushCallbacks);
  let textNode = document.createTextNode(1);
  observe.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    textNode.textContent = 2;
  }
} else if (setImmediate) {
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

## patch
```js
function patch (oldVnode, vnode, parentElm) {
  // 首先在 oldVnode不存在的时候，相当于新的 VNode 替代原本没有的节点，所以直接用 addVnodes 将这些节点批量添加到 parentElm 上。
  if (!oldVnode) {
    addVnode(parentElm, null, vnode, 0, vnode.length - 1);
    // 在 vnode（新 VNode 节点）不存在的时候，相当于要把老的节点删除，所以直接使用 removeVnodes 进行批量的节点删除即可
  } else if (!vnode) {
    removeVnode(parentElm, oldVnode, oldVnode.length - 1);
  } else {
    // 当 oldVNode 与 vnode 都存在的时候，需要判断它们是否属于 sameVnode（相同的节点）
    if (sameVnode(oldVnode, vnode)) {
      // 如果是相同节点则进行patchVnode（比对 VNode ）操作
      patchVnode(oldVnode, vnode);
    } else {
      // 如果不是相同节点，则删除老节点，增加新节点
      removeVnodes(parentElm, oldVnode, 0, oldVnode.length - 1);
      addVnodes(parentEle, null, vnode, 0, vnode.length - 1);
    }
  }
}

function patchVnode (oldVnode, vnode) {
  // 新老 VNode 节点相同的情况下，直接 return 掉
  if (oldVnode === vnode) return;

  // 当新老 VNode 节点都是 isStatic（静态的），并且 key 相同时，
  // 只要将 componentInstance 与 elm 从老 VNode 节点“拿过来”即可
  if (vnode.isStatic && oldVnode.isStatic && vnode.key === oldVnode.key) {
    vnode.elm = oldVnode.elm;
    vnode.componentInstance = oldVnode.componentInstance;
    return;
  }

  const elm = vnode.elm = oldVnode.elm;
  const oldCh = oldVnode.children;
  const ch = vnode.children;

  // 当新 VNode 节点是文本节点的时候，直接用 setTextContent 来设置 text，
  // 这里的 nodeOps 是一个适配层，根据不同平台提供不同的操作平台 DOM 的方法，实现跨平台
  if (vnode.text) {
    nodeOps.setTextContent(elm, vnode.text);
  } else {
    // 当新 VNode 节点是非文本节点当时候
    // oldCh 与 ch 都存在且不相同时，使用 updateChildren 函数来更新子节点
    if (oldCh && ch && (oldCh !== ch)) {
      updateChildren(elm, oldCh, ch);
      // 如果只有 ch 存在的时候，如果老节点是文本节点则先将节点的文本清除，然后将 ch 批量插入插入到节点elm下。
    } else if (ch) {
      if (oldVnode.text) nodeOps.setTextContent(elm, '');
      addVnodes(elm, null, ch, 0, ch.length - 1);
      // 同理当只有 oldch 存在时，说明需要将老节点通过 removeVnodes 全部清除
    } else if (oldCh) {
      removeVnodes(elm, oldCh, 0, oldCh.length - 1);
      // 最后一种情况是当只有老节点是文本节点的时候，清除其节点文本内容
    } else if (oldVnode.text) {
      nodeOps.setTextContent(elm, '');
    }
  }
}

function updateChildren (parentElm, oldCh, newCh) {
  // oldStartIdx、newStartIdx、oldEndIdx 以及 newEndIdx 分别是新老两个 VNode 的两边的索引
  let oldStartIdx = 0;
  let newStartIdx = 0;
  let oldEndIdx = oldCh.length - 1;
  let newEndIdx = newCh.length - 1;
  // oldStartVnode、newStartVnode、oldEndVnode 以及 newEndVnode 分别指向这几个索引对应的 VNode 节点
  let oldStartVnode = oldCh[0];
  let oldEndVnode = oldCh[oldEndIdx];
  let newStartVnode = newCh[0];
  let newEndVnode = newCh[newEndIdx];
  let oldKeyToIdx, idxInOld, elmToMove, refElm;

  // oldStartIdx、newStartIdx、oldEndIdx 以及 newEndIdx 会逐渐向中间靠拢
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    // 当 oldStartVnode 或者 oldEndVnode 不存在的时候，oldStartIdx 与 oldEndIdx 继续向中间靠拢，
    // 并更新对应的 oldStartVnode 与 oldEndVnode 的指向
    if (!oldStartVnode) {
      oldStartVnode = oldCh[++oldStartIdx];
    } else if (!oldEndVnode) {
      oldEndVnode = oldCh[--oldEndIdx];

      // oldStartVnode 与 newStartVnode 符合 sameVnode 时，
      // 说明老 VNode 节点的头部与新 VNode 节点的头部是相同的 VNode 节点，直接进行 patchVnode
      // 同时 oldStartIdx 与 newStartIdx 向后移动一位
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode);
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];

      // 其次是 oldEndVnode 与 newEndVnode 符合 sameVnode，也就是两个 VNode 的结尾是相同的 VNode，
      // 同样进行 patchVnode 操作并将 oldEndVnode 与 newEndVnode 向前移动一位。
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode);
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];

      // 两种交叉的情况
      // oldStartVnode 与 newEndVnode 符合 sameVnode 的时候，也就是老 VNode 节点的头部与新 VNode 节点的尾部是同一节点的时候，
      // 将 oldStartVnode.elm 这个节点直接移动到 oldEndVnode.elm 这个节点的后面即可。然后 oldStartIdx 向后移动一位，newEndIdx 向前移动一位。
    } else if (sameVnode(oldStartVnode, newEndVnode)) {
      patchVnode(oldStartVnode, newEndVnode);
      nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm));
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];

      // oldEndVnode 与 newStartVnode 符合 sameVnode 时，也就是老 VNode 节点的尾部与新 VNode 节点的头部是同一节点的时候，
      // 将 oldEndVnode.elm 插入到 oldStartVnode.elm 前面。同样的，oldEndIdx 向前移动一位，newStartIdx 向后移动一位。
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      patchVnode(oldEndVnode, newStartVnode);
      nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else {
      // createKeyToOldIdx 的作用是产生 key 与 index 索引对应的一个 map 表
      // 们可以根据某一个 key 的值，快速地从 oldKeyToIdx（createKeyToOldIdx 的返回值）中
      // 获取相同 key 的节点的索引 idxInOld，然后找到相同的节点。
      let elmToMove = oldCh[idxInOld];
      if (!oldKeyToIdx) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
      idxInOld = newStartVnode.key ? oldKeyToIdx[newStartVnode.key] : null;
      // 如果没有找到相同的节点，则通过 createElm 创建一个新节点，并将 newStartIdx 向后移动一位
      if (!idxInOld) {
        createElm(newStartVnode, parentElm);
        newStartVnode = newCh[++newStartIdx];

        // 如果找到了节点
      } else {
        elmToMove = oldCh[idxInOld];
        // 同时它符合 sameVnode,则将这两个节点进行 patchVnode
        // 将该位置的老节点赋值 undefined（之后如果还有新节点与该节点key相同可以检测出来提示已有重复的 key ），
        // 同时将 newStartVnode.elm 插入到 oldStartVnode.elm 的前面，newStartIdx 往后移动一位
        if (sameVnode(elmToMove, newStartVnode)) {
          patchVnode(elmToMove, newStartVnode);
          oldCh[idxInOld] = undefined;
          nodeOps.insertBefore(parentElm, newStartVnode.elm, oldStartVnode.elm);
          newStartVnode = newCh[++newStartIdx];

          // 如果不符合 sameVnode，只能创建一个新节点插入到 parentElm 的子节点中，newStartIdx 往后移动一位。
        } else {
          createElm(newStartVnode, parentElm);
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
  }

  // 当 while 循环结束以后，如果 oldStartIdx > oldEndIdx，说明老节点比对完了，但是新节点还有多的，
  // 需要将新节点插入到真实 DOM 中去，调用 addVnodes 将这些节点插入即可
  if (oldStartIdx > oldEndIdx) {
    refElm = (newCh[newEndIdx + 1]) ? newCh[newEndIdx + 1].elm : null;
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx);

    // 如果满足 newStartIdx > newEndIdx 条件，说明新节点比对完了，老节点还有多，
    // 将这些无用的老节点通过 removeVnodes 批量删除即可
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}
```

