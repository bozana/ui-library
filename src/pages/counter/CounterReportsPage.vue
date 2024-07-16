<template>
	<h1 class="text-default">Example page</h1>
	<Panel>
		<PanelSection>
			<CounterReportsListPanel
				v-bind="components.counterReportsListPanel"
				@set="set"
			/>
		</PanelSection>
	</Panel>
</template>

<script setup>
import Panel from '@/components/Panel/Panel.vue';
import PanelSection from '@/components/Panel/PanelSection.vue';
import CounterReportsListPanel from '@/components/ListPanel/counter/CounterReportsListPanel.vue';
</script>
