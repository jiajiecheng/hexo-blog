---
title: vue学习笔记（一）
date: 2023-06-27 07:58:05
tags: Vue
---

## ref系列

### ref

以下是ref的基本使用：

```TypeScript
type Person = {
  name:string,
  age:number
}

const person =  ref<Person>({
  name:"Jessie",
  age:18
})
```

### shallowRef

作为浅层次的响应，数据较大的时候建议使用浅层次响应，以节约性能：

```TypeScript
const person =  shallowRef<Person>({
  name:"Jessie",
  age:18
})
person.value = {
  name:"Chan",
  age:20
}
```

注意: ref与shallowRef会相互产生影响，shallowRef和ref不能够同时使用。因为*ref*底层已经调用了*triggerRef*导致依赖收集更新。

### customRef

自定义ref

```TypeScript
function MyRef<T>(value:T){
  return customRef((track,trigger) => {
    return {
      get() {
        track()
        return value
      },
      set(newVal){
        value = newVal
        trigger()
      }
    }
  })
} 
```

### ref获取dom元素

```TypeScript
....
<div ref="dom"></div>
....

const dom  = ref<HTMLDivElement>()
dom.value?.innerHTML
```

### 原理解析

ref底层其实调用的还是reactive，如果是普通值则直接返回值，如果是对象则调用reactive，导致shallowRef更新的原因是因为*triggerRefValue*这个函数，ref和triggerRef这两个函数底层都调用了这个函数。

## Reactive系列

### reactive

基本使用

```TypeScript
let form = reactive({
   name:'Jessie',
   age:40
})
form.name 
```

注意： 数组不能直接赋值，否则破坏响应式对象

```TypeScript
let arr = [1,2,3,4]
let list = reactive([])
// list = arr 错误
list.push(...arr)

// 或者
let arr = [1,2,3,4]
let list = reactive<{
   arr:string[]
}>({
 arr:[]
})
list.arr = arr
```

### readonly

```TypeScript
let form = reactive({
   name:'Jessie',
   age:40
})
// 只读对象
let read = readonly(form)
```

注意：此时read对象仍旧收到原来form对象的影

### shallowReactive

基本使用：

```TypeScript
let form = shallowReactive({
  // 此时只能监听到foo层
   foo:{
     name:{
        xing:'Chan',
        ming:'Jessie'
     }
   }
})
```

注意：shallowReactive与reactive同样不能一起用，否则会出现与上述*ref*相同的问题。

## to系列

### toRef

只能修改响应式数据，常用于解构，然后作为函数的参数。并且单独取值不影响原来的对象。

```TypeScript
const person = reactive({ name:"Jessie" , age:20 })
let name = toRef(person,"name")

const test(name:Ref<string>){
   ......
}
test(name)
```

### toRefs

源码

```TypeScript
const  toRefs = <T extends object>(object:T){
  const map:any = {}
  for(let key in object){
    map[key] = toRef(object,key)
  }
  return map
}
const {name,age} = toRefs(person)
```

注意： reactive一旦解构就失去了响应式，所以解构赋值请使用toRef或者toRefs

### toRaw

取消响应式

```TypeScript
toRaw(person)
```
