<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { snakeCase } from 'lodash-es';
import { useSessionStorage } from '@vueuse/core';

import { N8nButton, N8nInput, N8nTooltip } from 'n8n-design-system/components';
import type { CodeExecutionMode, INodeExecutionData } from 'n8n-workflow';

import type { BaseTextKey } from '@/plugins/i18n';
import type { INodeUi, Schema } from '@/Interface';
import { generateCodeForPrompt } from '@/api/ai';
import { useDataSchema, useI18n, useMessage, useToast, useTelemetry } from '@/composables';
import { useNDVStore, usePostHog, useRootStore, useWorkflowsStore } from '@/stores';
import { executionDataToJson } from '@/utils';
import {
	ASK_AI_EXPERIMENT,
	ASK_AI_MAX_PROMPT_LENGTH,
	ASK_AI_MIN_PROMPT_LENGTH,
	ASK_AI_LOADING_DURATION_MS,
} from '@/constants';

const emit = defineEmits<{
	(e: 'submit', code: string): void;
	(e: 'replaceCode', code: string): void;
	(e: 'startedLoading'): void;
	(e: 'finishedLoading'): void;
}>();

const props = defineProps<{
	hasChanges: boolean;
}>();

const { getSchemaForExecutionData, getInputDataWithPinned } = useDataSchema();
const i18n = useI18n();

const loadingPhraseIndex = ref(0);
const loaderProgress = ref(0);

const isLoading = ref(false);
const prompt = ref('');
const parentNodes = ref<INodeUi[]>([]);

const isSubmitEnabled = computed(() => {
	return (
		!isEachItemMode.value &&
		prompt.value.length >= ASK_AI_MIN_PROMPT_LENGTH &&
		hasExecutionData.value
	);
});
const hasExecutionData = computed(() => (useNDVStore().ndvInputData || []).length > 0);
const loadingString = computed(() =>
	i18n.baseText(`codeNodeEditor.askAi.loadingPhrase${loadingPhraseIndex.value}` as BaseTextKey),
);
const isEachItemMode = computed(() => {
	const mode = useNDVStore().activeNode?.parameters.mode as CodeExecutionMode;

	return mode === 'runOnceForEachItem';
});

function getErrorMessageByStatusCode(statusCode: number) {
	const errorMessages: Record<number, string> = {
		400: i18n.baseText('codeNodeEditor.askAi.generationFailedUnknown'),
		413: i18n.baseText('codeNodeEditor.askAi.generationFailedTooLarge'),
		429: i18n.baseText('codeNodeEditor.askAi.generationFailedRate'),
		500: i18n.baseText('codeNodeEditor.askAi.generationFailedUnknown'),
	};

	return errorMessages[statusCode] || i18n.baseText('codeNodeEditor.askAi.generationFailedUnknown');
}

function getParentNodes() {
	const activeNode = useNDVStore().activeNode;
	const { getCurrentWorkflow, getNodeByName } = useWorkflowsStore();
	const workflow = getCurrentWorkflow();

	if (!activeNode || !workflow) return [];

	return workflow
		.getParentNodesByDepth(activeNode?.name)
		.filter(({ name }, i, nodes) => {
			return name !== activeNode.name && nodes.findIndex((node) => node.name === name) === i;
		})
		.map((n) => getNodeByName(n.name))
		.filter((n) => n !== null) as INodeUi[];
}

function getSchemas() {
	const parentNodesNames = parentNodes.value.map((node) => node?.name);
	const parentNodesSchemas: Array<{ nodeName: string; schema: Schema }> = parentNodes.value
		.map((node) => {
			const inputData: INodeExecutionData[] = getInputDataWithPinned(node);

			return {
				nodeName: node?.name || '',
				schema: getSchemaForExecutionData(executionDataToJson(inputData), true),
			};
		})
		.filter((node) => node.schema?.value.length > 0);

	const inputSchema = parentNodesSchemas.shift();

	return {
		parentNodesNames,
		inputSchema,
		parentNodesSchemas,
	};
}

function startLoading() {
	emit('startedLoading');
	loaderProgress.value = 0;
	isLoading.value = true;

	triggerLoadingChange();
}

function stopLoading() {
	loaderProgress.value = 100;
	emit('finishedLoading');

	setTimeout(() => {
		isLoading.value = false;
	}, 200);
}

async function onSubmit() {
	const { getRestApiContext } = useRootStore();
	const { activeNode } = useNDVStore();
	const { showMessage } = useToast();
	const { alert } = useMessage();
	if (!activeNode) return;
	const schemas = getSchemas();

	useTelemetry().trackAskAI('ask.generationClicked', {
		prompt: prompt.value,
	});

	if (props.hasChanges) {
		const confirmModal = await alert(i18n.baseText('codeNodeEditor.askAi.areYouSureToReplace'), {
			title: i18n.baseText('codeNodeEditor.askAi.replaceCurrentCode'),
			confirmButtonText: i18n.baseText('codeNodeEditor.askAi.generateCodeAndReplace'),
			showClose: true,
			showCancelButton: true,
		});

		if (confirmModal === 'cancel') {
			return;
		}
	}

	startLoading();

	try {
		const version = useRootStore().versionCli;
		const model =
			usePostHog().getVariant(ASK_AI_EXPERIMENT.name) === ASK_AI_EXPERIMENT.gpt4
				? 'gpt-4'
				: 'gpt-3.5-turbo-16k';

		const { code, usage } = await generateCodeForPrompt(getRestApiContext, {
			question: prompt.value,
			context: { schema: schemas.parentNodesSchemas, inputSchema: schemas.inputSchema! },
			model,
			n8nVersion: version,
		});

		stopLoading();
		emit('replaceCode', code);
		showMessage({
			type: 'success',
			title: i18n.baseText('codeNodeEditor.askAi.generationCompleted'),
		});

		useTelemetry().trackAskAI('askAi.generationFinished', {
			prompt: prompt.value,
			code,
			tokensCount: usage?.total_tokens,
			hasErrors: false,
			error: '',
		});
	} catch (error) {
		showMessage({
			type: 'error',
			title: i18n.baseText('codeNodeEditor.askAi.generationFailed'),
			message: getErrorMessageByStatusCode(error.httpStatusCode || error?.response.status),
		});

		useTelemetry().trackAskAI('askAi.generationFinished', {
			prompt: prompt.value,
			code: '',
			tokensCount: 0,
			hasErrors: true,
			error: getErrorMessageByStatusCode(error.httpStatusCode || error?.response.status),
		});
		stopLoading();
	}
}
function triggerLoadingChange() {
	const loadingPhraseUpdateMs = 2000;
	const loadingPhrasesCount = 8;
	let start: number | null = null;
	let lastPhraseChange = 0;
	const step = (timestamp: number) => {
		if (!start) start = timestamp;

		// Loading phrase change
		if (!lastPhraseChange || timestamp - lastPhraseChange >= loadingPhraseUpdateMs) {
			loadingPhraseIndex.value = Math.floor(Math.random() * loadingPhrasesCount);
			lastPhraseChange = timestamp;
		}

		// Loader progress change
		const elapsed = timestamp - start;
		loaderProgress.value = Math.min((elapsed / ASK_AI_LOADING_DURATION_MS) * 100, 100);

		if (!isLoading.value) return;
		if (loaderProgress.value < 100 || lastPhraseChange + loadingPhraseUpdateMs > timestamp) {
			window.requestAnimationFrame(step);
		}
	};

	window.requestAnimationFrame(step);
}

function getSessionStoragePrompt() {
	const codeNodeName = (useNDVStore().activeNode?.name as string) ?? '';
	const hashedCode = snakeCase(codeNodeName);

	return useSessionStorage(`ask_ai_prompt__${hashedCode}`, '');
}

function onPromptInput(inputValue: string) {
	getSessionStoragePrompt().value = inputValue;
}

onMounted(() => {
	// Restore prompt from session storage(with empty string fallback)
	prompt.value = getSessionStoragePrompt().value;
	parentNodes.value = getParentNodes();
});
</script>

<template>
	<div>
		<p :class="$style.intro" v-text="i18n.baseText('codeNodeEditor.askAi.intro')" />
		<div :class="$style.inputContainer">
			<div :class="$style.meta">
				<span
					v-show="prompt.length > 1"
					:class="$style.counter"
					v-text="`${prompt.length} / ${ASK_AI_MAX_PROMPT_LENGTH}`"
					data-test-id="ask-ai-prompt-counter"
				/>
				<a href="https://docs.n8n.io/code-examples/ai-code" target="_blank" :class="$style.help">
					<n8n-icon icon="question-circle" color="text-light" size="large" />{{
						i18n.baseText('codeNodeEditor.askAi.help')
					}}
				</a>
			</div>
			<N8nInput
				v-model="prompt"
				@input="onPromptInput"
				:class="$style.input"
				type="textarea"
				:rows="6"
				:maxlength="ASK_AI_MAX_PROMPT_LENGTH"
				:placeholder="i18n.baseText('codeNodeEditor.askAi.placeholder')"
				data-test-id="ask-ai-prompt-input"
			/>
		</div>
		<div :class="$style.controls">
			<div :class="$style.loader" v-if="isLoading">
				<transition name="text-fade-in-out" mode="out-in">
					<div v-text="loadingString" :key="loadingPhraseIndex" />
				</transition>
				<n8n-circle-loader :radius="8" :progress="loaderProgress" :stroke-width="3" />
			</div>
			<n8n-tooltip :disabled="isSubmitEnabled" v-else>
				<div>
					<N8nButton
						:disabled="!isSubmitEnabled"
						@click="onSubmit"
						size="small"
						data-test-id="ask-ai-cta"
					>
						{{ i18n.baseText('codeNodeEditor.askAi.generateCode') }}
					</N8nButton>
				</div>
				<template #content>
					<span
						v-if="!hasExecutionData"
						v-text="i18n.baseText('codeNodeEditor.askAi.noInputData')"
						data-test-id="ask-ai-cta-tooltip-no-input-data"
					/>
					<span
						v-else-if="prompt.length === 0"
						v-text="i18n.baseText('codeNodeEditor.askAi.noPrompt')"
						data-test-id="ask-ai-cta-tooltip-no-prompt"
					/>
					<span
						v-else-if="isEachItemMode"
						v-text="i18n.baseText('codeNodeEditor.askAi.onlyAllItemsMode')"
						data-test-id="ask-ai-cta-tooltip-only-all-items-mode"
					/>
					<span
						v-else-if="prompt.length < ASK_AI_MIN_PROMPT_LENGTH"
						data-test-id="ask-ai-cta-tooltip-prompt-too-short"
						v-text="
							i18n.baseText('codeNodeEditor.askAi.promptTooShort', {
								interpolate: { minLength: ASK_AI_MIN_PROMPT_LENGTH.toString() },
							})
						"
					/>
				</template>
			</n8n-tooltip>
		</div>
	</div>
</template>

<style scoped>
.text-fade-in-out-enter-active,
.text-fade-in-out-leave-active {
	transition:
		opacity 0.5s ease-in-out,
		transform 0.5s ease-in-out;
}
.text-fade-in-out-enter,
.text-fade-in-out-leave-to {
	opacity: 0;
	transform: translateX(10px);
}
.text-fade-in-out-enter-to,
.text-fade-in-out-leave {
	opacity: 1;
}
</style>

<style module lang="scss">
.input * {
	border: 0 !important;
}
.input textarea {
	font-size: var(--font-size-2xs);
	padding-bottom: var(--spacing-2xl);
	font-family: var(--font-family);
	resize: none;
}
.intro {
	font-weight: var(--font-weight-bold);
	font-size: var(--font-size-2xs);
	color: var(--color-text-dark);
	padding: var(--spacing-2xs) var(--spacing-xs) 0;
}
.loader {
	font-size: var(--font-size-2xs);
	color: var(--color-text-dark);
	display: flex;
	align-items: center;
	gap: var(--spacing-2xs);
}
.inputContainer {
	position: relative;
}
.help {
	text-decoration: underline;
	margin-left: auto;
	color: #909399;
}
.meta {
	display: flex;
	justify-content: space-between;
	position: absolute;
	bottom: var(--spacing-2xs);
	left: var(--spacing-xs);
	right: var(--spacing-xs);
	z-index: 1;

	* {
		font-size: var(--font-size-2xs);
		line-height: 1;
	}
}
.counter {
	color: var(--color-text-light);
}
.controls {
	padding: var(--spacing-2xs) var(--spacing-xs);
	display: flex;
	justify-content: flex-end;
	border-top: 1px solid var(--border-color-base);
}
</style>
