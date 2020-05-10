<template>
	<!-- 不固定高度 -->
	<!-- height: 40px;   -->
	<div style=" border: 1px solid #ccc; padding: 20px 0;">{{item.id}}-------{{item.value}}</div>
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
