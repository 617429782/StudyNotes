
<img src="https://vuex.vuejs.org/vuex.png" height="350" >
<img src="https://awps-assets.meituan.net/mit-x/blog-images-bundle-2017/280a9260.jpg" height="350" >


```
new Store({
	name: 'test',
  state: {
    _obj: {},
  },
  getters: {
    obj(state) {
      return state._obj;
    },
  },
  mutations: {
    setObj (state, val) {
      state._obj = val
    },
  },
})

Store {
	commit: ƒ boundCommit(type, payload, options)
	dispatch: ƒ boundDispatch(type, payload)

	_mutations: {
		setObj: [ wrappedMutationHandler(payload) ]
	}

	_actionSubscribers: []
	_actions: {}
	_makeLocalGettersCache: {}
	_modules: ModuleCollection {root: Module}
	_modulesNamespaceMap: {}
	
	_subscribers: []
	_wrappedGetters: {obj: ƒ}

	state: {
		_obj: {
			__ob__: Observer {
				dep: {
					id: 9,
					subs: [Watcher]
				},
				value: __ob__,
			}
		},
		__ob__: Observer {
			dep: {
				id: 7,
				subs: [Watcher]
			},
			value: __ob__,
		}
	},

  getters: {
		obj: {
			__ob__: Observer {
				dep: {
					id: 9,
					subs: [Watcher]
				},
				value: __ob__,
			}
		}
	},

}
```