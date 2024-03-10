## part 1 背景 & 目的

- 热卖推荐项目的两次问题  
  数据更新没有触发 watch, 因而采用了 trick  
	```
		state: {
			list: {
				map: {
					0: { age: 1 }
				}
			}
		}

		mutations: {
			update(state, val) {
				const newMap = { ...state.list.map };
				val.forEach(item => {
					newMap[item.index] = item
				})
				state.list.map = newMap
			}
		}

		actions: {
			toUpdate({ state, commit }, fromBackVal) {
				const newItem = state.list.map[fromBackVal[0].index]
				newItem.name = '1'
				// oldItem === newItem
				commit(update, [newItem])
			}
		}
	```
  直播间串商品问题

- 日常开发中搜集的一些疑惑  
  对象类型的 属性变化 能否触发 watch ？  
  新增/删除 属性能否触发 watch ？  
  多层嵌套的属性变化后, 能否触发上层对象的 watch ？

- 分析 Vanx 特性及实现原理, 为上文提到的问题找到解释
- 对比 Wex 与 Vanx, 解释 Vanx 部分痛点产生的原因, 以及 Wex 是如何解决的

## part 2 Vanx 原理

### Vanx 功能

- 核心功能: mutations, actions, getters, modules
- 扩展功能: computed, watch, mapData
- 辅助函数: mapState, mapGetters, mapActions, mapMutations

### mutations、actions 的实现: 发布订阅机制

```
const s = new Store({
	state: {
		count: 1,
	},
	mutations: {
		add (state) {
			state.count ++
		},
	},
	actions: {
		toAdd (ctx) {
			ctx.commit('add');
		}
	}
})

s: Store {
	state: {
		count: 1,
		__ob__: Observer {
			ep: Dep { id: 7, subs: [] },
			value: __ob__
		}
	},
	_mutations: {
		add: [ wrappedMutationHandler(payload) ]
	},
	_actions: {
		toAdd: [ wrappedActionHandler(payload) ]
	},
	commit (type, payload)
}

s.commit('add')
s.state.count // 2
s.dispatch('toAdd')
s.state.count // 3
```

- 上述代码中有几点疑问

  1. 通过 commit 提交了 mutation 后, state.count + 1, commit 做了什么, 能不能 直接修改 state.count
  2. 为什么要在 action 里去提交 mutation
  3. Observer 及其下的 Dep 是什么, 有什么用

- Vanx 对传入的 mutations 和 actions 的处理可以简化为

  ```
  // mutations
  function registerMutation (store, type, handler, local) {
  	const entry = store._mutations[type] || (store._mutations[type] = []);
  	entry.push(function wrappedMutationHandler (payload) {
  		handler.call(store, local.state, payload);
  	});
  }

  function commit (type, payload) {
  	const entry = this._mutations[type];
  	entry.forEach((handler) => {
  		handler(payload);
  	});
  };

  // actions
  function registerAction (store, type, handler, local) {
  	const entry = store._actions[type] || (store._actions[type] = []);
  	entry.push((payload, cb) => {
  		let res = handler.call(store, {
  			dispatch: local.dispatch,
  			commit: local.commit,
  			getters: local.getters,
  			state: local.state,
  			rootGetters: store.getters,
  			rootState: store.state
  		}, payload, cb);
  		if (!isPromise(res)) {
  			res = Promise.resolve(res);
  		}
  		return res;
  	});
  }

  function dispatch (type, payload) {
  	const entry = this._actions[type];
  	return entry.length > 1
  		? Promise.all(entry.map(handler => handler(payload)))
  		: entry[0](payload);
  }
  ```

  事实上, 在 mutation 的 handler 中同步修改 state、在 action 中异步提交 mutation 都只是一种约定, 为的是让数据操作的意图更加明确  
  在 vuex 中, 因为 devtools 的需要, 严格模式下 state 被限制为只能在 mutation 中同步修改

- 小结:
  1. mutations 和 actions 的 注册 和 触发 可以简单理解为对 store 实例私有属性的 写 和 读, 注册的时候对传入的 handler 做了一层包装, 以便在 handler 中获取 state 和 module
  2. 功能上而言, 直接修改 state 在 Vanx 中是可以的 (但不建议这么做), 在 vuex 的严格模式下该行为将会抛出异常, vuex 判断修改是否来自 mutation 的 方式是  
     调用 commit 时设置 commiting 变量,
     深度监听 state, 在其改变时判断 commiting 并决定是否抛出异常

### 响应式原理: 观察者模式

- Vanx ≈ vuex@3.x + 响应式设计  
  vuex 的响应式部分依赖于 Vue.set  
  Vanx 删掉了 vuex 的部分功能, 如 plugin

- 响应式设计的实现  
  响应式设计的本质是 观察者模式的自动化
  观察者模式的基础实现是 被观察者(主题) 维护 观察者的列表, 并在合适的时机通知 所有观察者

  ```
  // 被观察者类 伪代码
  class Subject {
  	watchers: []       // 观察该 Subject 的所有 Watcher 实例
  	addOb() {}
  	removeOb() {}
  	notify() {}        // 通知所有的 Watcher 实例, 调用 update 方法
  }

  // 观察者类 伪代码
  class Watcher {
  	update() {}
  }
  ```

  在 Vue@2.x 和 Vanx 中, 被观察者 和 观察者 分别对应 Dep 和 Watcher; 在 Wex 中, 对应 Subject 和 Watcher  
  自动化 主要是三个步骤, 即 将一个对象转为响应式(调用 defineReactive)时, 自动为其

  - 创建被观察者实例
  - 添加 Watcher(依赖收集)
  - 通知 Watcher

  ```
  function defineReactive(target, key, value) {
  	const dep = new Dep();  // 创建 Subject 实例 的自动化
  	Object.defineProperty(target, key, {
  		get() {
  			dep.depend();       // 添加 Watcher 的自动化
  			return value;
  		},
  		set(newVal) {
  			value = newVal;
  			dep.notify();       // 通知 Watcher 的自动化
  		}
  	})
  }
  ```

- 在 Vue@2.x、 Vanx、Wex 中, 响应式设计都是基于 Object.defineProperty  
  上文中提到的 Observer 的主要作用, 就是为 Object.defineProperty 提供 属性描述符 (主要是 get 和 set)  
  上文中, dep.depend 的作用是 为 Dep 实例添加 Watcher, 并将 Dep 自身添加到 Watcher 下
  那么又有一个问题, target.get 在被读取时就会调用, 但并不是每次读取都要 为 Dep 添加 Watcher, 如何区分？

  ```
  class Dep {
  	depend() {
  		Dep.target && Dep.target.addDep(this)		// Dep.target 在什么时候存在？
  	}
  }

  class Watcher {
  	constructor (expOrFn, cb, options) {
  		this.getter = parsePath(expOrFn);
  		this.cb = cb
  		this.value = this.get()
  	}
  	addDep(dep) {
  		dep.addOb(this)
  	}
  	get() {
  		Dep.target = this;
  		// 通过调用 getter 来触发 target 的 get, 完成 依赖搜集
  		let value = this.getter.call(null)
  		Dep.target = null;
  		return value;
  	}
  }

  const o = { age: 10 }

  new Watcher(() => {
  	return o.age
  }, (val, old) => {
  	console.log(val, old)
  })

  new Watcher('o.age', (val, old) => {
  	console.log(val, old)
  })
  ```

  可以看出, 调用 Watcher.get 期间 Dep.target 会被设置成 Watcher 本身, 通过调用 Watcher.getter 来触发 target.get 来完成依赖收集

- 小结
  1.  响应式设计的基础是 观察者模式的自动化, 核心在于 Object.defineProperty 的 get/set
  2.  每个被转成响应式的 对象, 都有属于自己的 Dep 实例, 用于管理观察者 Watchers
  3.  并不是每次触发 get 都会进行依赖收集, 只有特定的时机 如 new Watcher、Watcher.update 才需要进行收集

### watch 和 getter 的实现

    Vanx 中, watch、getter 会为响应式对象的 Dep 创建 Watcher, 区别在于
    - watch 的回调会在每一次触发依赖时被执行
    - getter 具有缓存特性, 其回调会在 getter 值下一次被使用时(触发其 get 时)执行

- 这里有两个问题

  1. getter 回调的延迟执行, 或者说缓存, 是如何实现的
  2. 当 Object.defineProperty 劫持的属性 key 对应的值(key-value 的 value) 是对象类型时, value 下的属性变化不会触发对 key 的监听, 如何解决

  上述两个问题分别对应 Watcher 的 lazy 和 deep 配置项  
  先说 getter, 既然是缓存, 那么就应该考虑

  - 为什么要缓存
  - 缓存在哪里
  - 什么时候更新

  ```
  // Vanx getters
  function registerGetter (store, type, rawGetter, local) {
  	if (store._wrappedGetters[type]) {
  		return;
  	}

  	const watcher = new Watcher(store, (() => rawGetter(
  		local.state, // local state
  		local.getters, // local getters
  		store.state, // root state
  		store.getters // root getters
  	)), noop, { lazy: true });

  	store._wrappedGetters[type] = function wrappedGetter() {
  		return watcher.evaluate();
  	};
  }

  function computedGetters () {
  	this.getters = {};
  	const wrappedGetters = this._wrappedGetters;

  	forEachValue(wrappedGetters, (fn, key) => {
  		Object.defineProperty(this.getters, key, {
  			get: () => fn(this),
  			enumerable: true
  		});
  	});
  }
  ```

  可以看出, Object.defineProperty 重新定义了 store.getters, 用于在访问 getter 的 key 时触发其 wrappedGetter

  ```
  // Vanx watcher.evaluate
  evaluate() {
  	// 如果存在 Dep.target 表示还有别的数据依赖这个值的变化
  	// 所以需要把这个监听者加入到监听列表, 等这个数据值改变的时候通知监听者
  	if (Dep.target) {
  		this.dep.depend();
  	}

  	// 如果 dirty 标识为 true 表示数据有变化, 需要重新获取
  	if (this.dirty) {
  		this.value = this.get();
  		this.dirty = false;
  	}

  	return this.value;
  }

	// Vanx Dep.notify
	notify() {
		this.subs.forEach((sub) => sub.update());
		this.subs.forEach(sub => sub.done())
	}

	// Vanx watcher.update
	update(immediate) {
    this.dirty = true;

    // 对于主动获取的, 不需要立即计算, 如果有依赖的需要及时通知
    if (this.lazy) {
      this.dep.notify();
      return;
    }

    this.oldValue = this.value;
    this.value = this.get();
    
    // 初始化 watch 的时候, 直接执行
    immediate && this.done()
  }

	// Vanx watcher.done
	done() {
    if (this.dirty && !this.lazy) {
      this.dirty = false;
      this.cb.call(this.ctx, this.value, this.oldValue);
    }
  }
  ```
  wrappedGetter 调用了 watcher.evaluate(), 可以得出结论: getter 的值缓存于 watcher.value  
	需要注意的是, optison.getters 传入的 handler 并不是作为 watcher.cb 传入, 而是 watcher.expOrFn, 这意味着其调用时机 不是target.set触发依赖之后, 而是在 watcher.get 调用时  
	当响应式数据(target)被修改后, 首先执行 target.set 触发依赖, 然后调用 Watcher.update 和 Watcher.done, 将 新旧值 传递给 handler (watcher.cb)  
	但是在 lazy 模式下, 创建 watcher 并不会同时进行依赖收集, 也就是说, registerGetter 时虽然为 target 创建了 watcher, 但是没有把 watcher 添加到 target.dep 下
	因此 target 被修改的时候, 不需要关注 getter 相关的计算, lazy-watcher 被添加到 target.dep 下的时机是 getter.handler 调用时, 即, 读取 getter 属性值的时候

  ```
  // Watcher.constructor
  constructor (expOrFn, cb, options) {
  	this.getter = parsePath(expOrFn);
  	this.cb = cb
  	this.lazy = !!options.lazy;
		this.dirty = !!options.lazy;
  	this.value = this.lazy ? undefined : this.get()
  }
  ```

  deep 的实现相对简单, 因为 Dep 和 Watcher 各自维护了对方的列表, 因此只需要在 收集依赖的过程中将 递归其子对象 并将其转为响应式, 同时将子对象的 watcher 添加到自身的 Dep.subs 中

  - 小结
    1. getter 和 watch 实现的原理都是 Watcher, 区别在于 lazy-watcher 在 target.set 时仅更新 value, 不触发 handler, 而是在 getter.get 时触发

## part 3 对比 vuex / Vanx / Wex

### Vanx 的痛点是如何产生的

- [x] 1. 对数组赋值, 会触发两次 watcher
	```
		state: {
			_arr: [1]
		}
		mutations: {
			setArr1 (state, val) { state._arr = [{ p: 1 }] },
			setArr2 (state, val) { state._arr.push(1) },
			setArr3 (state, val) { state._arr.pop() },
		}
		watch: {
			arr(v) { console.log('监听到了 arr 变化') }
		}
		setArr1() // 触发2次
		setArr2() // 触发1次
		setArr3() // 触发0次

		set(newVal) {
			if (equal(newVal, this._value)) return;
			this._setter ? this._setter(newVal) : this.observe(newVal);
			this.dep.notify();
		}
	```
	
- [x] 2. 同一个调用栈内, 多次赋值操作触发多次相同的 watcher
	set时若新旧值不一致则触发watcher, 没有去重
  引入 watcher 队列, 合并相同的 watcher, 放到 nextTick 执行

- [x] 3. 修改对象的属性值, 触发了对象对应的 watcher
  应该在 deep 配置下才触发
	vanx 在 set 时深度匹配了对象树，使得属性变化也会通知到对象本身
	```
		state: { _obj: { age: 1 } }
		mutations: {
			setObj (state) {
				state._obj.age ++
			},
		},
		watch: {
			'data._obj': () => {
				console.log('监听到了 data._obj 变化')
			}
		}
		setObj() // 修改了 _obj.age，应不应该通知 _obj 的观察者呢
	```

- [ ] 4. 修改数组内对象类型元素的属性值, 不会触发数组对应的 watcher -- 在 vue 中会触发

- [ ] 5. watcher 对象属性值 相同仍然触发的情况 -- vue 中不会触发

- [ ] 6. 页面 unload 后, 由一些异步任务触发的提交 mutation 行为会报错

  上文说到并不是每次 get 都要收集依赖, 同样, 也不是每次 set 都要触发依赖, Vanx 的做法是在 set 中判断新旧值是否相等, 实现如下

  ```
  	set(newVal) {
  		if (equal(newVal, this._value)) {
  			return;
  		}
  		this._setter ? this._setter(newVal) : this.observe(newVal);
  		this.dep.notify();
  	}

  	function equal(a, b) {
  		// 基本类型 和 同一对象
  		if (a === b) return true;

  		// a 和 b 都是 数组
  		a.length === b.length
  		遍历 equal a 和 b 的每个数据项

  		// a 和 b 都是 对象且不为 null
  		Object.keys(a).length === Object.keys(b).length
  		Object.keys(b).every(k => k in a)
  		递归 equal a 和 b 的每个属性值

  		// 其他情况, 针对 NaN
  		return a!==a && b!==b;
  	}

  	observe(value) {
  		if (Array.isArray(value)) {
  			// attachArrayProps 作用是对数组原型方法做劫持
  			this.attachArrayProps(this._value) && this.observeArray(value);
  		}
  	}

  	observeArray(value) {
  		value.forEach((item, index) => {
  			defineReactive(this._value, index, item);
  		});
  		this.dep.notify();
  	}
  ```

  在 vue 中, 仅仅 比对了对象本身, 而不会对属性进行递归比较

  ```
	set (newVal) {
		if (newVal === value || (newVal !== newVal && value !== value)) {
			return
		}
		setter.call(obj, newVal)
		observe(newVal)
		dep.notify()
	}
  ```

  可以看出, Vanx 中对触发依赖没有做什么处理, 这也是多次赋值操作会触发多次的原因, 在 vue 和 Wex 中, watcher 会被添加到队列, 在 nextTick 中执行
  在 set 中, 会对传入的新值做 递归劫持, 即 this.observe(newVal), 这个过程中, 若 newVal 为数组, 则进行一次 notify()  
  那么 为什么需要为数组类型多执行一次 notify(), notify() 做了什么事情？

  ```
	notify() {
		this.subs.forEach((sub) => sub.update());
		this.subs.forEach(sub => sub.done())
	}

	update(immediate) {
		this.oldValue = this.value;
		this.value = this.get();
	}

	done() {
		this.cb.call(this.ctx, this.value, this.oldValue);
	}
  ```

  可以看出, update 中完成了 新值 和 旧值 的更新, 用于在 done 中执行创建 watcher 时传入的回调
  此外, 因为获取新值时, 调用了 watcher.get, 重新进行了依赖收集

  ```
  	// Vanx Watcher.get
  	get() {
  		// 每次获取时, 设置依赖的ID标识为空, 在获取值过程中会设置新的依赖的ID
  		// 获取完值后, 根据新的依赖ID标识删除那些旧的依赖
  		this.depIds = new Set();

  		pushDepTarget(this); // 理解成 Dep.target = this

  		const value = this.getter();	// 理解成调用了 addDep

  		// 监听嵌套数据的改变
  		// 通过遍历值, 可以对所有的嵌套数据添加到依赖中, 当值改变时, 可触发 update
  		this.deep && traverse(value);

  		popDepTarget(); // 理解成 Dep.target = null

  		this.cleanDeps(); // 在当前流程中, 可以认为无效
  		return value;
  	}

  	addDep(dep) {
  		const { id } = dep;

  		if (!this.deps.has(dep)) {
  			this.deps.add(dep);
  			dep.addSub(this);
  		}

  		this.depIds.add(id);
  	}

  	cleanDeps() {
  		for (const dep of this.deps) {
  			if (!this.depIds.has(dep.id)) {
  				this.deps.delete(dep);
  				dep.removeSub(this);
  			}
  		}
  	}
  ```

  众所周知, Object.defineProperty 无法感知对象类型的属性修改, 具体表现为:

  - 数组  
    调用 push、pop 等可以修改数组本身的方法  
    通过索引修改数组  
    通过修改 length 修改数组

  - 对象  
    删除/增加/修改 属性

  以上操作不会触发 set 方法, 在 Vanx 中, 针对数组采用了函数劫持的方式

  ```
  	attachArrayProps(value) {
  		// arrayKeys 是被劫持的方法列表
  		arrayKeys.forEach(key => {
  			// attachMethods 理解为数组原型方法的扩展
  			def(value, key, attachMethods[key]);
  		});
  	}

  	// Vanx attachMethods
  	[
  		'push',
  		'pop',
  		'shift',
  		'unshift',
  		'splice',
  		'sort',
  		'reverse'
  	].forEach((method) => {
  		// cache original method
  		const original = arrayProto[method];
  		def(arrayMethods, method, function mutator(...args) {
  			const result = original.apply(this, args);
  			const ob = this.__ob__;
  			let inserted;
  			switch (method) {
  				case 'push':
  				case 'unshift':
  					inserted = args;
  					break;
  				case 'splice':
  					inserted = args.slice(2);
  					break;
  			}
  			if (inserted) {
  				ob.observeArray(this);
  			}

  			// 这里缺少了 ob.dep.notify(), 被放到了 observeArray 中

  			return result;
  		});
  	});
  ```

  这里应该是 Vanx 的设计失误, 缺少了 ob.dep.notify() 意味着 pop 等方法无法触发 watcher

### 相对 Vanx, Wex 在哪些地方做了升级

- 异步触发依赖, 将多个 watcher 合并成一个

## part 3 总结

- 大致介绍了 Vanx 流程
- 简单实现了响应式设计
- 解释了热卖推荐项目出现问题的原因
- 解释了 Vanx 部分痛点产生的原因, 以及 Wex 是如何解决的




