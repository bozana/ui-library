<template>
	<li class="pkpListPanelItem" :class="{'-isOrdering': isOrdering}">
		<list-panel-item-orderer
			v-if="isOrdering"
			@itemOrderUp="itemOrderUp"
			@itemOrderDown="itemOrderDown"
			:itemTitle="item.title"
			:i18n="i18n"
		/>
		<div class="pkpListPanelItem__item">
			{{ item.title }}
		</div>
	</li>
</template>

<script>
import ListPanelItem from '@/components/ListPanel/ListPanelItem.vue';
import ListPanelItemOrderer from '@/components/ListPanel/ListPanelItemOrderer.vue';

export default {
	extends: ListPanelItem,
	name: 'WithOrdering_OrderableListPanelItem',
	props: ['item', 'i18n', 'isOrdering'],
	components: {
		ListPanelItemOrderer,
	},
};
</script>
