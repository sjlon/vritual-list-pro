<template>
	<div id="app">
		<!-- 需要只显示可视区域 -->
		<!-- 需要知道我的列表每一项多高，因为需要弄出一个滚动条来 -->
		<!--  -->
		<VirtualList :size="100" :remain="5" :items="items" :variable="true">
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
import mockjs from 'mockjs'
let items = []
for (let i = 0; i < 10000; i++) {
	items.push({ id: i, value: mockjs.Random.sentence(10, 60) })
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
<style lang="scss">
* {
	margin: 0;
	padding: 0;
	box-sizing: border-box;
}
</style>
