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



