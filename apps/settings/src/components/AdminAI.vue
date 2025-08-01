<!--
  - SPDX-FileCopyrightText: 2023 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="ai-settings">
		<NcSettingsSection :name="t('settings', 'Unified task processing')"
			:description="t('settings', 'AI tasks can be implemented by different apps. Here you can set which app should be used for which task.')">
			<NcCheckboxRadioSwitch v-model="settings['ai.taskprocessing_guests']"
				type="switch"
				@update:modelValue="saveChanges">
				{{ t('settings', 'Allow AI usage for guest users') }}
			</NcCheckboxRadioSwitch>
			<template v-for="type in taskProcessingTaskTypes">
				<div :key="type">
					<h3>{{ t('settings', 'Task:') }} {{ type.name }}</h3>
					<p>{{ type.description }}</p>
					<NcCheckboxRadioSwitch v-model="settings['ai.taskprocessing_type_preferences'][type.id]"
						type="switch"
						@update:modelValue="saveChanges">
						{{ t('settings', 'Enable') }}
					</NcCheckboxRadioSwitch>
					<NcSelect v-model="settings['ai.taskprocessing_provider_preferences'][type.id]"
						class="provider-select"
						:clearable="false"
						:disabled="!settings['ai.taskprocessing_type_preferences'][type.id]"
						:options="taskProcessingProviders.filter(p => p.taskType === type.id).map(p => p.id)"
						@input="saveChanges">
						<template #option="{label}">
							{{ taskProcessingProviders.find(p => p.id === label)?.name }}
						</template>
						<template #selected-option="{label}">
							{{ taskProcessingProviders.find(p => p.id === label)?.name }}
						</template>
					</NcSelect>
					<p>&nbsp;</p>
				</div>
			</template>
			<template v-if="!hasTaskProcessing">
				<NcNoteCard type="info">
					{{ t('settings', 'None of your currently installed apps provide Task processing functionality') }}
				</NcNoteCard>
			</template>
		</NcSettingsSection>
		<NcSettingsSection :name="t('settings', 'Machine translation')"
			:description="t('settings', 'Machine translation can be implemented by different apps. Here you can define the precedence of the machine translation apps you have installed at the moment.')">
			<draggable v-model="settings['ai.translation_provider_preferences']" @change="saveChanges">
				<div v-for="(providerClass, i) in settings['ai.translation_provider_preferences']" :key="providerClass" class="draggable__item">
					<DragVerticalIcon /> <span class="draggable__number">{{ i + 1 }}</span> {{ translationProviders.find(p => p.class === providerClass)?.name }}
					<NcButton aria-label="Move up" type="tertiary" @click="moveUp(i)">
						<template #icon>
							<ArrowUpIcon />
						</template>
					</NcButton>
					<NcButton aria-label="Move down" type="tertiary" @click="moveDown(i)">
						<template #icon>
							<ArrowDownIcon />
						</template>
					</NcButton>
				</div>
			</draggable>
		</NcSettingsSection>
		<NcSettingsSection :name="t('settings', 'Image generation')"
			:description="t('settings', 'Image generation can be implemented by different apps. Here you can set which app should be used.')">
			<template v-for="provider in text2imageProviders">
				<NcCheckboxRadioSwitch :key="provider.id"
					:checked.sync="settings['ai.text2image_provider']"
					:value="provider.id"
					name="text2image_provider"
					type="radio"
					@update:checked="saveChanges">
					{{ provider.name }}
				</NcCheckboxRadioSwitch>
			</template>
			<template v-if="!hasText2ImageProviders">
				<NcNoteCard type="info">
					{{ t('settings', 'None of your currently installed apps provide image generation functionality') }}
				</NcNoteCard>
			</template>
		</NcSettingsSection>
		<NcSettingsSection :name="t('settings', 'Text processing')"
			:description="t('settings', 'Text processing tasks can be implemented by different apps. Here you can set which app should be used for which task.')">
			<template v-for="type in tpTaskTypes">
				<div :key="type">
					<h3>{{ t('settings', 'Task:') }} {{ getTextProcessingTaskType(type).name }}</h3>
					<p>{{ getTextProcessingTaskType(type).description }}</p>
					<p>&nbsp;</p>
					<NcSelect v-model="settings['ai.textprocessing_provider_preferences'][type]"
						class="provider-select"
						:clearable="false"
						:options="textProcessingProviders.filter(p => p.taskType === type).map(p => p.class)"
						@input="saveChanges">
						<template #option="{label}">
							{{ textProcessingProviders.find(p => p.class === label)?.name }}
						</template>
						<template #selected-option="{label}">
							{{ textProcessingProviders.find(p => p.class === label)?.name }}
						</template>
					</NcSelect>
					<p>&nbsp;</p>
				</div>
			</template>
			<template v-if="tpTaskTypes.length === 0">
				<NcNoteCard type="info">
					<!-- TRANSLATORS Text processing is the name of a Nextcloud-internal API -->
					{{ t('settings', 'None of your currently installed apps provide text processing functionality using the Text Processing API.') }}
				</NcNoteCard>
			</template>
		</NcSettingsSection>
	</div>
</template>

<script>
import axios from '@nextcloud/axios'
import NcCheckboxRadioSwitch from '@nextcloud/vue/components/NcCheckboxRadioSwitch'
import NcSettingsSection from '@nextcloud/vue/components/NcSettingsSection'
import NcSelect from '@nextcloud/vue/components/NcSelect'
import NcButton from '@nextcloud/vue/components/NcButton'
import NcNoteCard from '@nextcloud/vue/components/NcNoteCard'
import draggable from 'vuedraggable'
import DragVerticalIcon from 'vue-material-design-icons/DragVertical.vue'
import ArrowDownIcon from 'vue-material-design-icons/ArrowDown.vue'
import ArrowUpIcon from 'vue-material-design-icons/ArrowUp.vue'
import { loadState } from '@nextcloud/initial-state'
import { nextTick } from 'vue'
import { generateUrl } from '@nextcloud/router'

export default {
	name: 'AdminAI',
	components: {
		NcCheckboxRadioSwitch,
		NcSettingsSection,
		NcSelect,
		draggable,
		DragVerticalIcon,
		ArrowDownIcon,
		ArrowUpIcon,
		NcButton,
		NcNoteCard,
	},
	data() {
		return {
			loading: false,
			dirty: false,
			groups: [],
			loadingGroups: false,
			sttProviders: loadState('settings', 'ai-stt-providers'),
			translationProviders: loadState('settings', 'ai-translation-providers'),
			textProcessingProviders: loadState('settings', 'ai-text-processing-providers'),
			textProcessingTaskTypes: loadState('settings', 'ai-text-processing-task-types'),
			text2imageProviders: loadState('settings', 'ai-text2image-providers'),
			taskProcessingProviders: loadState('settings', 'ai-task-processing-providers'),
			taskProcessingTaskTypes: loadState('settings', 'ai-task-processing-task-types'),
			settings: loadState('settings', 'ai-settings'),
		}
	},
	computed: {
		hasTextProcessing() {
			return Object.keys(this.settings['ai.textprocessing_provider_preferences']).length > 0 && Array.isArray(this.textProcessingTaskTypes)
		},
		tpTaskTypes() {
			const builtinTextProcessingTypes = [
				'OCP\\TextProcessing\\FreePromptTaskType',
				'OCP\\TextProcessing\\HeadlineTaskType',
				'OCP\\TextProcessing\\SummaryTaskType',
				'OCP\\TextProcessing\\TopicsTaskType',
			]
			return Object.keys(this.settings['ai.textprocessing_provider_preferences'])
				.filter(type => !!this.getTextProcessingTaskType(type))
				.filter(type => !builtinTextProcessingTypes.includes(type))
		},
		hasText2ImageProviders() {
		  return this.text2imageProviders.length > 0
		},
		hasTaskProcessing() {
			return Object.keys(this.settings['ai.taskprocessing_provider_preferences']).length > 0 && Array.isArray(this.taskProcessingTaskTypes)
		},
	},
	methods: {
	  moveUp(i) {
			this.settings['ai.translation_provider_preferences'].splice(
			  Math.min(i - 1, 0),
				0,
				...this.settings['ai.translation_provider_preferences'].splice(i, 1),
			)
			this.saveChanges()
		},
		moveDown(i) {
			this.settings['ai.translation_provider_preferences'].splice(
				i + 1,
				0,
				...this.settings['ai.translation_provider_preferences'].splice(i, 1),
			)
			this.saveChanges()
		},
		async saveChanges() {
			this.loading = true
			await nextTick()
			const data = { settings: this.settings }
			try {
				await axios.put(generateUrl('/settings/api/admin/ai'), data)
			} catch (err) {
				console.error('could not save changes', err)
			}
			this.loading = false
		},
		getTextProcessingTaskType(type) {
		  if (!Array.isArray(this.textProcessingTaskTypes)) {
				return null
			}
			return this.textProcessingTaskTypes.find(taskType => taskType.class === type)
		},
	},
}
</script>
<style scoped>
.draggable__item {
	margin-bottom: 5px;
  display: flex;
  align-items: center;
}

.draggable__item,
.draggable__item * {
  cursor: grab;
}

.draggable__number {
	border-radius: 20px;
	border: 2px solid var(--color-primary-element);
	color: var(--color-primary-element);
	padding: 0px 7px;
	margin-inline-end: 3px;
}

.drag-vertical-icon {
  float: left;
}

.ai-settings h3 {
	font-size: 16px; /* to offset against the 20px section heading */
}

.provider-select {
	min-width: 350px !important;
}
</style>
