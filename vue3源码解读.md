# Vue3原始值的响应式原理

vue3是基于proxy来实现数据劫持的，然而proxy它只能用于对象，不能用于基础类型数据。因此vue3对基础类型的数据进行了一层包裹。通过使用ref函数，生成RefImpl对象，内部再通过toReactive函数实现响应式。



## ref函数

```ts
// packages/reactivity/src/ref.ts

export function ref<T extends object>(
  value: T
): [T] extends [Ref] ? T : Ref<UnwrapRef<T>>
export function ref<T>(value: T): Ref<UnwrapRef<T>>
export function ref<T = any>(): Ref<T | undefined>
export function ref(value?: unknown) {
  return createRef(value, false)
}

// packages/reactivity/src/ref.ts

function createRef(rawValue: unknown, shallow: boolean) {
  // 如果已经是 ref 对象，直接返回
  if (isRef(rawValue)) {
    return rawValue
  }
  // 创建一个 ref 对象实例
  return new RefImpl(rawValue, shallow)
}

```

ref接收一个原始值作为参数，内部通过`createRef`方法，生成一个 `RefImpl`实例对象

> rawValue：即传入的原始值
>
> shallow：是一个boolean值，表示创建的响应式变量是深层次还是浅层次。



## RefImpl

```ts
// packages/reactivity/src/ref.ts

// ref 的实现类  ref函数返回的其实是RefImpl类的实例对象
class RefImpl<T> {
  private _value: T
  private _rawValue: T  //传入的原始值

  public dep?: Dep = undefined  //调度中心的dep是一个map，键名为监听的对象，键值是个子map。子map中，键名是对应的key，键值是一个set，用来存放watch对应的update方法。而这里的dep只是收集副作用函数
    
  // ref实例下都有一个 __v_isRef 的只读属性，标识它是一个ref
  public readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean) {
    // 如果是 浅层响应，则直接将 _rawValue 置为 value，否则通过toRaw()获取其原始值，因为它可能传入的是响应式变量
    this._rawValue = __v_isShallow ? value : toRaw(value)
    // 如果是 浅层响应，则直接将 _value 置为 value，否则将 value 转换成深响应
    this._value = __v_isShallow ? value : toReactive(value)
  }

  // 拦截 读取 操作
  get value() {
    // 通过 trackEffects 收集 value 依赖 将副作用函数推入dep中
    trackRefValue(this)
    // 返回 该ref 对应的 value 属性值，实现自动脱 ref 能力 
    return this._value
  }

  // 拦截 设置 操作
  set value(newVal) {
    newVal = this.__v_isShallow ? newVal : toRaw(newVal)
    // 比较新值和旧值
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      // 转换成响应式数据
      this._value = this.__v_isShallow ? newVal : toReactive(newVal)
      // 调用 triggerRefValue， 通过 triggerEffects 派发 value 更新
      triggerRefValue(this, newVal)
    }
  }
}
```

过程：

- 定义两个私有变量，`_value`用来存储`原始值变成响应式变量后的数据`， `_rawvalue`用来存放`原始值`
- 在constructor中，判断其是否是深层次响应式，如果是的话，就调用`toReative`函数，将其转化为`响应式变量`
- 在get中收集副作用函数。
- 在set中判断新值和旧值是否发生改变，如果改变，就调用trigger通知调度中心，触发对应订阅者的update方法

**总结：RefImpl类本质上还是通过toReactive函数，将其转化为响应式变量**



## 响应式丢失问题

在刚接触vue3的时候，我们对reactive创建的响应式变量解构，发现再次调用时，不会触发响应式。这是因为{...obj}得到的是一个 **新的对象**，而且这个新的对象 是 `普通对象`，(不是通过reactive创建的)，自然没有响应式。为了方便我们解构使用，vue封装了toRef

```ts
// obj 是响应式数据
const obj = reactive({ foo: 1, bar: 2 })

// 将响应式数据展开到一个新的对象 newObj 中
const newObj = {
  ...obj
}

effect(() => {
  // 在副作用函数内通过新的对象 newObj 读取 foo 属性值
  console.log(newObj.foo)
})

// 很显然，此时修改 obj.foo 并不会触发响应
obj.foo = 100

```



## toRef

```ts
//用来为源响应式对象上的某个 property 新创建一个 ref
// 第一个参数 obj 是一个响应式数据，第二个参数是 obj 对象的一个健
export function toRef<T extends object, K extends keyof T>(
  object: T,
  key: K,
  defaultValue?: T[K]
): ToRef<T[K]> {
  const val = object[key]
  // 返回 ref 对象
  return isRef(val)
    ? val
    : (new ObjectRefImpl(object, key, defaultValue) as any)
}

```

**toRef的原理：** 

- 第一个参数是响应式对象，第二个参数是响应式对象中的键名。首先通过isRef判断该键名对应的value是否是响应式对象，如果是直接返回，如果不是，就通过`ObjectRefImpl类`创建一个`响应式变量返回`

![image-20231225152552852](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231225152552852.png) ![image-20231225152623386](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231225152623386.png)

## toRefs

toRefs其实就是遍历响应式对象里的key，判断其键值是否是响应式变量，如果不是，调用toRef，创建响应式变量。最后放在一个对象中返回。



> 因此，toRef本质上是`将响应式对象的第一层属性值转化为ref`(通过ObjectRefImpl类)，如果我们想要使用属性值，那么需要通过 **.value**来进行访问

```ts
const obj = reactive({ foo: 1, bar: 2 })
const { foo, bar } = { ...toRefs(obj) }
console.log('foo', foo)
console.log('bar', bar)
```

![image-20231225152250714](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231225152250714.png) 





## 模板中自动脱ref(省略.value)

在模板编写中，我们会遇到一个很奇怪的事情，就是ref创建的响应式变量，我们无需`.value`来获取其值。

`setup`函数是有返回值的，返回的数据会交给`proxyRefs`来处理，它会再次创建一个proxy对象，在get中直接返回.value。

```js
            let obj = {
                a: 1,
                b: 2,
            }
            let _obj1 = new Proxy(obj, {
                get(target, key) {
                    console.log('第一次get调用')
                    //track
                    return target[key]
                },
                set(target, key, value) {
                    console.log('第一次set调用')

                    target[key] = {
                        value: value,
                    }
                    //trigger
                },
            })
            console.log(_obj1.a)
            let _obj2 = new Proxy(_obj1, {
                get(target, key) {
                    console.log('第二次get调用')
                    return target[key].value
                },
                set(target, key, value) {
                    console.log('第二次set调用')
                    target[key] = value
                },
            })
            _obj2.a = 1111
            console.log(_obj2.a)
```







# Vue非原始值的响应式原理

> vue3非原始值的响应式是基于proxy实现的，proxy除了可以代理对象、数组，还可以代理map、set、weakmap、weakset等集合类型。

**vue3非原始值响应式通过调用reactive()，基于proxy返回一个代理对象，在get中进行依赖收集，在set中进行通知调度中心，分发依赖，从而实现视图更新**



## 代理Object

### 	属性读取的拦截

一个普通对象的属性读取有：①直接读取obj.key  ②for in 循环读取 ③key in obj

对于这些属性的读取，vue都会进行拦截操作，以便数据发生变更时，视图能跟着变化。即 `track()`

**过程：**

- 如果被拦截的`key` 是一个symbol或者原型链上的属性，那么`不进行依赖收集`，**直接返回属性的读取结果**(即不做响应式处理)
- 实现get()方法，在get中调用 `track()`方法收集依赖
- 实现set()方法，在set中调用 `trigger()`方法触发对应副作用，实现视图更新



**简单实现reactive:**

```js
   let bucket = new WeakMap()
            let proxyMap = new WeakMap()
            const data = {
                text: 'hello vue3',
            }

            function reactive(target) {
                const existingProxy = proxyMap.get(target)
                if (existingProxy) {
                    // 已被代理过，直接返回缓存的代理对象
                    // 避免重复被代理
                    return existingProxy
                }

                const proxy = new Proxy(target, baseHandlers)

                proxyMap.set(target, proxy)
                return proxy
            }

            const baseHandlers = {
                get(target, key, receiver) {
                    const res = Reflect.get(target, key, receiver)
                    track(target, key)
                    return res
                },
                set(target, key, value, receiver) {
                    const res = Reflect.set(target, key, value, receiver)
                    trigger(target, key)
                    return res
                },
            }
            const _data = reactive(data)
            //把effect转为用来注册副作用函数的函数
            let activeEffect = null
            function effect(fn) {
                activeEffect = fn
                fn()
            }
            effect(() => (document.body.innerText = _data.text))

            //将activeEffect添入set中
            function track(target, key) {
                if (!activeEffect) return //depsMap   target:map     map中  key:set
                let depsMap = bucket.get(target)
                if (!depsMap) {
                    bucket.set(target, (depsMap = new Map()))
                }
                let deps = depsMap.get(key)
                if (!deps) {
                    depsMap.set(key, (deps = new Set()))
                }
                deps.add(activeEffect)
            }
            //取出set并循环执行
            function trigger(target, key) {
                const depsMap = bucket.get(target)
                if (!depsMap) return
                const effects = depsMap.get(key)
                effects && effects.forEach((effect) => effect())
            }
            setTimeout(() => {
                _data.text = 'hello react'
            }, 1000)
```



### 嵌套属性处理

因为proxy只能实现浅层对象的响应式，如果对象是深层次的，如：

```
let data={
	son:{
		a:1
	}
}
data.son.a=123  //这样是无法触发对应的set
```

因此需要递归对象，为其子对象进行代理。



# computed源码解读

> vue源码中通过effect函数注册副作用，在其options中如果添加了lazy:true,则不立即执行副作用函数。而是将副作用函数返回

```ts
export function effect<T = any>(
  fn: () => T,
  options?: ReactiveEffectOptions
): ReactiveEffectRunner {
  
  // 省略部分代码
  
  // 只有非 lazy 的时，才执行副作用函数
  if (!options || !options.lazy) {
    _effect.run()
  }
  const runner = _effect.run.bind(_effect) as ReactiveEffectRunner
  runner.effect = _effect
  // 将副作用函数作为返回值返回
  return runner
}
```

`computed` **本质上就是一个懒加载的副作用函数，在effect函数中通过lazy进行控制，实现懒执行。**

****



## computed函数调用签名

```js
// packages/reactivity/src/computed.ts

// 只读的
export function computed<T>(
  getter: ComputedGetter<T>,
  debugOptions?: DebuggerOptions
): ComputedRef<T>
// 可写的 
export function computed<T>(
  options: WritableComputedOptions<T>,
  debugOptions?: DebuggerOptions
): WritableComputedRef<T>
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>,
  debugOptions?: DebuggerOptions,
  isSSR = false
)
```

- 第一种情况，computed接收一个`getter`作为参数，返回一个`不可修改`的响应式  `ref`对象

  ```js
  const count = ref(1)
  // computed 接受一个 getter 函数
  const plusOne = computed(() => count.value + 1)
  
  console.log(plusOne.value) // 2
  
  plusOne.value++ // 错误
  
  ```

- 第二种情况，接收一个具有 `set`和 `get`的options对象，返回一个`可修改`的响应式`ref`对象

  ```js
  const count = ref(1)
  const plusOne = computed({
    // computed 函数接受一个具有 get 和 set 函数的 options 对象
    get: () => count.value + 1,
    set: val => {
      count.value = val - 1
    }
  })
  
  plusOne.value = 1
  console.log(count.value) // 0
  
  ```



## 实现原理

- 在第一种情况下，它会判断传入的第一个参数是否是函数，如果是函数，赋值给定义好的getter，同时，`setter`赋值为`不进行任何操作 NOOP`

  ```ts
   // 判断 getterOrOptions 参数 是否是一个函数
    const onlyGetter = isFunction(getterOrOptions)
    if (onlyGetter) {
      // getterOrOptions 是一个函数，则将函数赋值给取值函数getter 
      getter = getterOrOptions
      setter = __DEV__
        ? () => {
          	//dev环境下 setter为提示，prod环境下不进行操作
            console.warn('Write operation failed: computed value is readonly')
          }
        : NOOP
    } else {
      // getterOrOptions 是一个 options 选项对象，分别取 get/set 赋值给取值函数getter和赋值函数setter
      getter = getterOrOptions.get
      setter = getterOrOptions.set
    }
  
  ```

- 第二种情况下，传入options对象，就将传入的get和set函数分别赋值给setter和getter

  ```ts
      // getterOrOptions 是一个 options 选项对象，分别取 get/set 赋值给取值函数getter和赋值函数setter
      getter = getterOrOptions.get
      setter = getterOrOptions.set
  ```

- 处理完setter和getter后，会通过 `ComputedRefImpl`类，创建出一个`ref对象`并返回

  ```ts
   // 实例化一个 computed 实例
    const cRef = new ComputedRefImpl(getter, setter, onlyGetter || !setter, isSSR)
  
    if (__DEV__ && debugOptions && !isSSR) {
      cRef.effect.onTrack = debugOptions.onTrack
      cRef.effect.onTrigger = debugOptions.onTrigger
    }
  
    return cRef as any
  
  ```



## ComputedRefImpl类的实现

> 因为computed最后会返回一个 `ComputedRefImpl`类的实例( 即 `ref`响应式变量 )，如果读取了value属性，就会触发 `get value()`。在该方法中手动调用`trackRefValue`，添加依赖。`ComputedRefImpl`内部用 `_value`来缓存上一次的值， `_dirty`标识是否需要重新计算值，`如果为true，则需要重新计算`，如果false。直接返回 `_value`即实现懒加载。当计算属性依赖的响应式数据变更时，手动触发 `triggerRefValue`，触发响应式

```ts
// packages/reactivity/src/computed.ts

export class ComputedRefImpl<T> {
  public dep?: Dep = undefined

  // value 用来缓存上一次计算的值
  private _value!: T
  public readonly effect: ReactiveEffect<T>

  public readonly __v_isRef = true
  public readonly [ReactiveFlags.IS_READONLY]: boolean

  // dirty标志，用来表示是否需要重新计算值，为 true 则意味着具有副作用，需要重新计算
  public _dirty = true
  public _cacheable: boolean

  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean,
    isSSR: boolean
  ) {
    this.effect = new ReactiveEffect(getter, () => {
      // getter的时候，不派发通知
      if (!this._dirty) {
        this._dirty = true
        // 当计算属性依赖响应式数据变化时，手动调用 triggerRefValue 函数 触发响应式
        triggerRefValue(this)
      }
    })
    this.effect.computed = this
    this.effect.active = this._cacheable = !isSSR
    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    // the computed ref may get wrapped by other proxies e.g. readonly() #3376
    // 获取原始对象
    const self = toRaw(this)
    // 当读取 value 时，手动调用 trackRefValue 函数进行追踪  track需要传入一个对象，然后会将activeEffect推入到对应的set中等待触发
    trackRefValue(self)
    // 只有副作用才计算值，并将得到的值缓存到value中
    if (self._dirty || !self._cacheable) {
      // 将dirty设置为 false， 下一次访问直接使用缓存的 value 中的值
      self._dirty = false
      self._value = self.effect.run()!
    }
    // 返回最新的值
    return self._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}

```

