# computed以及watch相关面试题

## 1. computed 什么时候初始化

在`vue`创建实例的时候会初始化各种选项，其中initState方法中会初始化`props、methods、data、computed、以及watch`，其中`initComputed`就是初始化`computed`

## 2. computed 的缓存是怎么做的

computed本身就是一个计算属性类型的watcher，通过控制`watcher`实例对象的`dirty`属性做到的，`dirty`属性默认值是`false`，`computed`将 `lazy` 赋值给 `dirty`，就是给一个初始值，让`watcher`控制缓存的任务开始，如果 `computed` 依赖的数据变化，`dirty` 会变成`true`，调用 `evalute` 重新计算，然后更新缓存值 `watcher.value`。

如果computed的数据`A`调用了data的数据`B`，即`A依赖B`，那么就会`B`会收集到`A`的`watcher`，当`B`发生改变了，就会通知`A`进行更新，即调用 `A-watcher.update`，当通知 `computed` 更新的时候，就只是把 `dirty` 设置为 `true`，从而读取 `comptued` 时，便会调用 `evalute` 重新计算。

## 3. computed是怎么可以直接使用实例访问到的

在defineComputed函数当中通过`Object.defineProperty(target, key, sharedPropertyDefinition)`将computed的所有属性都定义在了vm上面，所以可以直接使用实例访问到的

## 4. computed是怎样进行计算的

computed在初始化watcher实例时，并没有计算值，`watcher.value = undefined`，只有调用了`computed`的属性时，控制`watcher`实例对象的`dirty`设置为true才会发生计算。在计算的过程中，computed-watcher.evaluted 被调用，进而 computed-watcher.get 被调用，Dep.target 被设置为 computed-watcher，旧值 页面 watcher 被缓存起来。computed 计算会读取 data，此时 data 就收集到 computed-watcher，同时 computed-watcher 也会保存到 data 的依赖收集器 dep。computed 计算完毕，释放Dep.target，并且Dep.target 恢复上一个watcher（页面watcher），返回value的更新值

## 5. computed 的 月老身份的来源

1. 页面更新，读取 computed 的时候，Dep.target 会设置为 页面 watcher。

2. computed 被读取，createComputedGetter 包装的函数触发，第一次会进行计算

3. computed-watcher.evaluted 被调用，进而 computed-watcher.get 被调用，Dep.target 被设置为 computed-watcher，旧值 页面 watcher 被缓存起来。

computed 计算会读取 data，此时 data 就收集到 computed-watcher

同时 computed-watcher 也会保存到 data 的依赖收集器 dep（用于下一步）。

computed 计算完毕，释放Dep.target，并且Dep.target 恢复上一个watcher（页面watcher）

4. 手动 watcher.depend， 让 data 再收集一次 Dep.target，于是 data 又收集到 恢复了的页面watcher

从而使得页面A与数据C关联上


## 5. computed与watch的区别

**区别**

+ 计算属性computed和监听器watcher都可以观察属性的变化从而做出响应
+ computed本质上就是一个watcher，与watch最大的区别就在于computed更多的是作为缓存功能的观察者，它可以将一个或者多个data的属性进行复杂的计算生成一个新的值，提供给渲染函数使用，当依赖发生变化时，computed不会立即重新计算生成新的值，而是先标记为脏数据，当下次computed被获取的时候，才会重新计算并返回；而监听器watch并不具备缓存性，监听器watch提供一个监听函数，类似于某些数据的监听回调，每当监听的数据变化时都会执行回调进行后续操作

**运用的场景**

+ 当我们需要进行数值计算，并且依赖于其它数据时，应该使用 computed，因为可以利用 computed 的缓存特性，避免每次获取值时，都要重新计算；
  
+ 当我们需要在数据变化时执行异步或开销较大的操作时，应该使用 watch，使用 watch 选项允许我们执行异步操作 ( 访问一个 API )，限制我们执行该操作的频率，并在我们得到最终结果前，设置中间状态。这些都是计算属性无法做到的。

## 6. 在使用计算属性时，函数名和 data 数据源中的数据可以同名吗？

不可以重名，不仅仅是计算属性和 data，其他的如 props，methods 都不可以重名，因为 Vue 会把这些属性挂载在组件实例上，直接使用 this 就可以访问，如果重名就会导致冲突。

在组件初始化的时候会执行 `_init` 函数, 这里面执行了一个 `initState` 的方法，这里初始化了很多数据，有 `props，methods，data，computed，watch` 这几个,在这个`initComputed`方法内部，做了一个判断，会去查你定义的 `computed` 的 key 值在 `data `和 `props` 中有没有存在，这个时候 props 和 data 都已经初始化完成了，且已经挂载到了组件实例上，你的 `computed` 如果有冲突的话就会报错了，其实这几个初始化数据的方法内部都有做这些检测。

## 7. watch的原理

watch用于监控用户的data变化，数据变化后会触发对应的watch的回调方法,vue当中的watch属性本质上其实调用的是vue暴露在实例上面的`$watch`方法，`$watch`内部就是创建一个`watcher`的过程，如果用户传进来的还有一些特殊的操作做一些特殊的处理（如`immediate`立刻执行回调函数），然后借助vue响应式原理，默认在取值时会将这个旧值保存下来，然后将watcher存放到对应属性的dep中，当数据发生变化时通知对应`的watcher`重新执行`getter`函数拿到用户返回的结果作为旧值，然后执行用户传进来的回调函数，将新旧值作为参数

## 8. vue 和 react 里的key的作用是什么? 为什么不能用Index？用了会怎样? 如果不加key会怎样?