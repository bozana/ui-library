<template>
	<SideModalBody>
		<template #title>
			{{ title }}
		</template>
		<SideModalLayoutBasic>
			<PkpForm
				v-bind="activeForm"
				@set="(...args) => emit('updateForm', ...args)"
				@success="onSuccess"
			/>
		</SideModalLayoutBasic>
	</SideModalBody>
</template>

<script setup>
import SideModalBody from '@/components/Modal/SideModalBody.vue';
import SideModalLayoutBasic from '@/components/Modal/SideModalLayoutBasic.vue';
import PkpForm from '@/components/Form/Form.vue';
//import {useForm} from '@/composables/useForm';

defineProps({
	title: {type: String, required: true},
	activeForm: {type: Object, required: true},
	//reportId: {type: String, required: false},
	reportFields: {type: Array, required: true},
	earliestDate: {type: String, required: true},
	lastDate: {type: String, required: true},
});
const emit = defineEmits(['updateForm', 'formSuccess']);
</script>
