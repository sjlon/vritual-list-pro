#### 什么是进程？什么是线程？

* 进程是系统资源分配和调查的一个独立单位，一个进程内可以包含多个线程

#### 渲染进程

* GUI渲染线程-页面渲染
* JS引擎线程-执行js脚本
* 事件触发线程-EventLoop轮询线程

> GUI 渲染线程和JS引擎线程是互斥的



#### 浏览器中的EventLoop

1. 执行JS脚本执行当前宏任务，执行栈自上向下执行，碰到宏任务将宏任务放到宏任务队列中，碰到微任务将微任务的回调放到微任务队列中
2. 当前宏任务执行完毕，就会去清空微任务队列
3. 微任务队列执行完毕，调用GUI渲染线程渲染页面
4. 页面渲染完毕就会去宏任务队列中，取出第一个宏任务执行，重复上面的步骤

#### 超长列表渲染性能优化

* 分片渲染（通过浏览器事件循环机制、分割渲染时间）

* 虚拟列表（只渲染可视区域）



> 分片渲染

```js
<body>
    <div id="ul">

    </div>
    <script>
        let start = Date.now()

        for (let i = 0; i <= 100000; i++) {
            let li = document.createElement('li')
            li.innerHTML = i
            ul.appendChild(li)
        }
        let end = Date.now()
        console.log(end - start, '执行时间') // 569 执行时间
        setTimeout(() => {
            console.log(Date.now() - end, '渲染时间') // 3785 渲染时间
        }, 0)
        // 分片-根据数据大小每一次加载固定的数量
    </script>
</body>


```

```js
<body>
    <div id="ul">

    </div>
    <script>
        let total = 100000
        // 偏移量
        let index = 0
        let id = 0
        let start = Date.now()
        function load() {
            index += 50
            if (index < total) {
                requestAnimationFrame(() => {
                    let fragment = document.createDocumentFragment()
                    for (let i = 0; i < 50; i++) {
                        let li = document.createElement('li')
                        li.innerHTML = id++
                        fragment.appendChild(li)
                    }
                    ul.appendChild(fragment)
                    load()
                })
            }
        }
        load()

        let end = Date.now()
        console.log(end - start, '执行时间')
        setTimeout(() => {
            console.log(Date.now() - end, '渲染时间')
        }, 0)
        // 分片加载会导致也看dom过多，造成页面卡顿
        // 虚拟列表优化
    </script>
</body>
```



> 虚拟列表优化

```js
// #app.vue
<template>
	<div id="app">
		<!-- 需要只显示可视区域 -->
		<!-- 需要知道我的列表每一项多高，因为需要弄出一个滚动条来 -->
		<!--  -->
		<VirtualList :size="40" :remain="8" :items="items">
			<!-- 作用域插槽 -->
			<template #default="{item}">
				<Itme :item="item" />
			</template>
		</VirtualList>
	</div>
</template>
<script>
import VirtualList from './views/virtualList'
import Itme from './views/item'
let items = []
for (let i = 0; i < 10000; i++) {
	items.push({ id: i, value: i })
}
export default {

	components: {
		VirtualList, Itme
	},
	data () {
		return {
			items: items
		}
	},


}
</script>
```

```js
// virtualList.vue
<template>
	<!-- 可滚动的盒子 -->
	<div class="viewport" ref="viewport" @scroll="handleScroll">
		<!-- 滚动条 -->
		<div class="scroll-bar" ref="scrollBar"></div>
		<!-- 真实渲染的内容 -->
		<div class="scroll-list" ref="scrollList" :style="{transform: `translate3d(0, ${offset}px, 0)`}">
			<div v-for="item in visibleData " :key="item.id" :vid="item.id">
				<slot :item="item"></slot>
			</div>
		</div>
	</div>
</template>

<script>
export default {
	name: 'virtualList',
	props: {
		size: Number, // 当前每一项的高度
		remain: Number, // 可见个数
		items: Array

	},
	data () {
		return {
			start: 0,
			end: this.remain, // 默认的显示8个
			offset: 0
		}
	},
	computed: {
		// 可见的数据
		visibleData () {
			let start = this.start - this.preCount
			let end = this.end + this.nextCount
			// return this.items.slice(this.start, this.end)
			return this.items.slice(start, end)
		},
		// 前面预留几个
		preCount () {
			return Math.min(this.start, this.remain)
		},
		// 后面预留几个
		nextCount () {
			return Math.min(this.remain, this.items.length - this.end)
		}
	},
	mounted () {
		this.$refs.viewport.style.height = this.size * this.remain + 'px'
		this.$refs.scrollBar.style.height = this.items.length * this.remain + 'px'
	},
	methods: {
		handleScroll () {
			// 1. 计算当前应该从第几个开始
			let scrollTop = this.$refs.viewport.scrollTop
			console.log(scrollTop)
			this.start = Math.floor(scrollTop / this.size)
			this.end = this.start + this.remain // 当前可渲染的区域
			// 定位当前的可视区域,调整可视区域的位置
			// 如果有预留渲染，应该把位置再向上移动那么多
			this.offset = this.start * this.size - this.size * this.preCount

		}
	}
}
</script>

<style lang="scss">
.viewport {
	overflow-y: scroll;
	position: relative;
	border: 1px solid red;
}
.scroll-list {
	position: absolute;
	left: 0;
	top: 0;
	width: 100%;
}
</style>

```

```js
// item.vue
<template>
	<div style="height: 40px; border: 1px solid #ccc;">{{item.value}}</div>
</template>

<script>
export default {
	name: 'item',
	props: {
		item: Object
	}
}
</script>

<style lang="scss" scoped>
</style>

```

###### 不固定高度



> variable: true

```js
<template>
	<!-- 可滚动的盒子 -->
	<div class="viewport" ref="viewport" @scroll="scrollFn">
		<!-- 滚动条 -->
		<div class="scroll-bar" ref="scrollBar"></div>
		<!-- 真实渲染的内容 -->
		<div class="scroll-list" ref="scrollList" :style="{transform: `translate3d(0, ${offset}px, 0)`}">
			<div v-for="item in visibleData " :key="item.id" :vid="item.id" ref="items">
				<slot :item="item"></slot>
			</div>
		</div>
	</div>
</template>

<script>
import throttle from 'lodash/throttle'
export default {
	name: 'virtualList',
	props: {
		size: Number, // 当前每一项的高度
		remain: Number, // 可见个数
		items: Array,
		variable: Boolean, // 高度不一定

	},
	data () {
		return {
			start: 0,
			end: this.remain, // 默认的显示8个
			offset: 0
		}
	},
	computed: {
		// 可见的数据
		visibleData () {
			let start = this.start - this.preCount
			let end = this.end + this.nextCount
			// return this.items.slice(this.start, this.end)
			return this.items.slice(start, end)
		},
		// 前面预留几个
		preCount () {
			return Math.min(this.start, this.remain)
		},
		// 后面预留几个
		nextCount () {
			return Math.min(this.remain, this.items.length - this.end)
		}
	},
	mounted () {
		this.$refs.viewport.style.height = this.size * this.remain + 'px'
		this.$refs.scrollBar.style.height = this.items.length * this.remain + 'px'
		// 当不固定高度，如果加载完毕需要缓存每一项的高度
		// 记录每一项的高度，滚动渲染真实dom获取真实高度，更新缓存中的内容
		this.cacheList()
		// 重新计算滚动条的高度
	},
	created () {
		this.scrollFn = throttle(this.handleScroll, 200, { leading: false })
	},
	updated () {
		// 页面渲染完成，根据当前展示的数据，更新缓存区的内容， height， top, bottom,最终更新滚动条的高度
		this.$nextTick(() => {
			let nodes = this.$refs.items //获取真实的节点
			if (!(nodes && nodes.length)) {
				return
			}
			nodes.forEach(node => {
				let { height } = node.getBoundingClientRect() //真实的高度
				// 更新高度
				let id = node.getAttribute('vid')
				console.log(id)
				let oldHeight = this.positions[id].height
				let val = oldHeight - height // 计算当前高度与之前缓存的高度是否有差
				if (val) {
					this.positions[id].height = height
					this.positions[id].bottom = this.positions[id].bottom - val // 底部增加了
				}
				// 链表
				for (let i = id + 1; i < this.positions.length; i++) {
					this.positions[i].top = this.positions[i - 1].bottom
					this.positions[i].bottom = this.positions[i].bottom - val
				}
			})
			// 更新滚动条的高度
			this.$refs.scrollBar.style.height = this.positions[this.positions.length - 1].bottom + 'px'
		})
	},
	methods: {
		cacheList () {
			// 缓存当前项的高度，top值，bottom
			this.positions = this.items.map((item, index) => ({
				height: this.size,
				top: index * this.size,
				bottom: (index + 1) * this.size,
				id: item.id
			}))
			console.log(this.positions, 'positions')
		},
		handleScroll () {
			console.log('throttle')
			// 1. 计算当前应该从第几个开始
			let scrollTop = this.$refs.viewport.scrollTop

			if (this.variable) {
				// 如果有variable，则使用二分查找，找到对应的高度
				this.start = this.getStartIndex(scrollTop)
				console.log(this.start, 'xxxx')
				this.end = this.start + this.remain
				// 设置偏移量
				this.offset = this.positions[this.start - this.preCount] ? this.positions[this.start - this.preCount].top : 0

			} else {

				this.start = Math.floor(scrollTop / this.size)
				this.end = this.start + this.remain // 当前可渲染的区域
				// 定位当前的可视区域,调整可视区域的位置
				// 如果有预留渲染，应该把位置再向上移动那么多
				this.offset = this.start * this.size - this.size * this.preCount
			}

		},
		getStartIndex (value) {
			let start = 0 // 开始
			let end = this.positions.length
			let temp = null
			while (start <= end) {
				let midIndex = parseInt((start + end) / 2)
				let midValue = this.positions[midIndex].bottom // 找到当前那个人的结尾点
				if (midValue === value) { // 如果找到了，则返回当前的下一个人
					return midIndex
				} else if (midValue < value) { //当前要查找的人在右边
					start = midIndex + 1
				} else if (midValue > value) { // 当前要查找的人在左边
					if (temp === null || temp > midIndex) {
						temp = midIndex // 找范围
					}
					end = midIndex - 1
				}
			}
			return temp


		}
	}
}
</script>

<style lang="scss">
.viewport {
	overflow-y: scroll;
	position: relative;
	border: 1px solid red;
}
.scroll-list {
	position: absolute;
	left: 0;
	top: 0;
	width: 100%;
}
</style>

```

