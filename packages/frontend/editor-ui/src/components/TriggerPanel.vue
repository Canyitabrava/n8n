<script setup lang="ts">
import { computed, ref } from 'vue';
import {
	CHAT_TRIGGER_NODE_TYPE,
	VIEWS,
	WEBHOOK_NODE_TYPE,
	WORKFLOW_SETTINGS_MODAL_KEY,
	FORM_TRIGGER_NODE_TYPE,
} from '@/constants';
import type { INodeUi } from '@/Interface';
import type { INodeTypeDescription } from 'n8n-workflow';
import { getTriggerNodeServiceName } from '@/utils/nodeTypesUtils';
import NodeExecuteButton from '@/components/NodeExecuteButton.vue';
import CopyInput from '@/components/CopyInput.vue';
import NodeIcon from '@/components/NodeIcon.vue';
import { useUIStore } from '@/stores/ui.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { useNDVStore } from '@/stores/ndv.store';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';
import { createEventBus } from '@n8n/utils/event-bus';
import { useRouter } from 'vue-router';
import { useWorkflowHelpers } from '@/composables/useWorkflowHelpers';
import { isTriggerPanelObject } from '@/utils/typeGuards';
import { useI18n } from '@n8n/i18n';
import { useTelemetry } from '@/composables/useTelemetry';

const props = withDefaults(
	defineProps<{
		nodeName: string;
		pushRef: string;
	}>(),
	{
		pushRef: '',
	},
);

const emit = defineEmits<{
	activate: [];
	execute: [];
}>();

const nodesTypeStore = useNodeTypesStore();
const uiStore = useUIStore();
const workflowsStore = useWorkflowsStore();
const ndvStore = useNDVStore();

const router = useRouter();
const workflowHelpers = useWorkflowHelpers();
const i18n = useI18n();
const telemetry = useTelemetry();

const executionsHelpEventBus = createEventBus();

const help = ref<HTMLElement | null>(null);

const node = computed<INodeUi | null>(() => workflowsStore.getNodeByName(props.nodeName));

const nodeType = computed<INodeTypeDescription | null>(() => {
	if (node.value) {
		return nodesTypeStore.getNodeType(node.value.type, node.value.typeVersion);
	}

	return null;
});

const triggerPanel = computed(() => {
	const panel = nodeType.value?.triggerPanel;
	if (isTriggerPanelObject(panel)) {
		return panel;
	}
	return undefined;
});

const hideContent = computed(() => {
	const hideContent = triggerPanel.value?.hideContent;
	if (typeof hideContent === 'boolean') {
		return hideContent;
	}

	if (node.value) {
		const hideContentValue = workflowHelpers
			.getCurrentWorkflow()
			.expression.getSimpleParameterValue(node.value, hideContent, 'internal', {});

		if (typeof hideContentValue === 'boolean') {
			return hideContentValue;
		}
	}

	return false;
});

const hasIssues = computed(() => {
	return Boolean(
		node.value?.issues && (node.value.issues.parameters ?? node.value.issues.credentials),
	);
});

const serviceName = computed(() => {
	if (nodeType.value) {
		return getTriggerNodeServiceName(nodeType.value);
	}

	return '';
});

const displayChatButton = computed(() => {
	return Boolean(
		node.value &&
			node.value.type === CHAT_TRIGGER_NODE_TYPE &&
			node.value.parameters.mode !== 'webhook',
	);
});

const isWebhookNode = computed(() => {
	return Boolean(node.value && node.value.type === WEBHOOK_NODE_TYPE);
});

const webhookHttpMethod = computed(() => {
	if (!node.value || !nodeType.value?.webhooks?.length) {
		return undefined;
	}

	const httpMethod = workflowHelpers.getWebhookExpressionValue(
		nodeType.value.webhooks[0],
		'httpMethod',
		false,
	);

	if (Array.isArray(httpMethod)) {
		return httpMethod.join(', ');
	}

	return httpMethod;
});

const webhookTestUrl = computed(() => {
	if (!node.value || !nodeType.value?.webhooks?.length) {
		return undefined;
	}

	return workflowHelpers.getWebhookUrl(nodeType.value.webhooks[0], node.value, 'test');
});

const isWebhookBasedNode = computed(() => {
	return Boolean(nodeType.value?.webhooks?.length);
});

const isPollingNode = computed(() => {
	return Boolean(nodeType.value?.polling);
});

const isListeningForEvents = computed(() => {
	const waitingOnWebhook = workflowsStore.executionWaitingForWebhook;
	const executedNode = workflowsStore.executedNode;
	return (
		!!node.value &&
		!node.value.disabled &&
		isWebhookBasedNode.value &&
		waitingOnWebhook &&
		(!executedNode || executedNode === props.nodeName)
	);
});

const workflowRunning = computed(() => workflowsStore.isWorkflowRunning);

const isActivelyPolling = computed(() => {
	const triggeredNode = workflowsStore.executedNode;

	return workflowRunning.value && isPollingNode.value && props.nodeName === triggeredNode;
});

const isWorkflowActive = computed(() => {
	return workflowsStore.isWorkflowActive;
});

const listeningTitle = computed(() => {
	return nodeType.value?.name === FORM_TRIGGER_NODE_TYPE
		? i18n.baseText('ndv.trigger.webhookNode.formTrigger.listening')
		: i18n.baseText('ndv.trigger.webhookNode.listening');
});

const listeningHint = computed(() => {
	switch (nodeType.value?.name) {
		case CHAT_TRIGGER_NODE_TYPE:
			return i18n.baseText('ndv.trigger.webhookBasedNode.chatTrigger.serviceHint');
		case FORM_TRIGGER_NODE_TYPE:
			return i18n.baseText('ndv.trigger.webhookBasedNode.formTrigger.serviceHint');
		default:
			return i18n.baseText('ndv.trigger.webhookBasedNode.serviceHint', {
				interpolate: { service: serviceName.value },
			});
	}
});

const header = computed(() => {
	if (isActivelyPolling.value) {
		return i18n.baseText('ndv.trigger.pollingNode.fetchingEvent');
	}

	if (triggerPanel.value?.header) {
		return triggerPanel.value.header;
	}

	if (isWebhookBasedNode.value) {
		return i18n.baseText('ndv.trigger.webhookBasedNode.action', {
			interpolate: { name: serviceName.value },
		});
	}

	return '';
});

const subheader = computed(() => {
	if (isActivelyPolling.value) {
		return i18n.baseText('ndv.trigger.pollingNode.fetchingHint', {
			interpolate: { name: serviceName.value },
		});
	}

	return '';
});

const executionsHelp = computed(() => {
	if (triggerPanel.value?.executionsHelp) {
		if (typeof triggerPanel.value.executionsHelp === 'string') {
			return triggerPanel.value.executionsHelp;
		}
		if (!isWorkflowActive.value && triggerPanel.value.executionsHelp.inactive) {
			return triggerPanel.value.executionsHelp.inactive;
		}
		if (isWorkflowActive.value && triggerPanel.value.executionsHelp.active) {
			return triggerPanel.value.executionsHelp.active;
		}
	}

	if (isWebhookBasedNode.value) {
		if (isWorkflowActive.value) {
			return i18n.baseText('ndv.trigger.webhookBasedNode.executionsHelp.active', {
				interpolate: { service: serviceName.value },
			});
		} else {
			return i18n.baseText('ndv.trigger.webhookBasedNode.executionsHelp.inactive', {
				interpolate: { service: serviceName.value },
			});
		}
	}

	if (isPollingNode.value) {
		if (isWorkflowActive.value) {
			return i18n.baseText('ndv.trigger.pollingNode.executionsHelp.active', {
				interpolate: { service: serviceName.value },
			});
		} else {
			return i18n.baseText('ndv.trigger.pollingNode.executionsHelp.inactive', {
				interpolate: { service: serviceName.value },
			});
		}
	}

	return '';
});

const activationHint = computed(() => {
	if (isActivelyPolling.value || !triggerPanel.value) {
		return '';
	}

	if (triggerPanel.value.activationHint) {
		if (typeof triggerPanel.value.activationHint === 'string') {
			return triggerPanel.value.activationHint;
		}
		if (!isWorkflowActive.value && typeof triggerPanel.value.activationHint.inactive === 'string') {
			return triggerPanel.value.activationHint.inactive;
		}
		if (isWorkflowActive.value && typeof triggerPanel.value.activationHint.active === 'string') {
			return triggerPanel.value.activationHint.active;
		}
	}

	if (isWebhookBasedNode.value) {
		if (isWorkflowActive.value) {
			return i18n.baseText('ndv.trigger.webhookBasedNode.activationHint.active', {
				interpolate: { service: serviceName.value },
			});
		} else {
			return i18n.baseText('ndv.trigger.webhookBasedNode.activationHint.inactive', {
				interpolate: { service: serviceName.value },
			});
		}
	}

	if (isPollingNode.value) {
		if (isWorkflowActive.value) {
			return i18n.baseText('ndv.trigger.pollingNode.activationHint.active', {
				interpolate: { service: serviceName.value },
			});
		} else {
			return i18n.baseText('ndv.trigger.pollingNode.activationHint.inactive', {
				interpolate: { service: serviceName.value },
			});
		}
	}

	return '';
});

const expandExecutionHelp = () => {
	if (help.value) {
		executionsHelpEventBus.emit('expand');
	}
};

const openWebhookUrl = () => {
	telemetry.track('User clicked ndv link', {
		workflow_id: workflowsStore.workflowId,
		push_ref: props.pushRef,
		pane: 'input',
		type: 'open-chat',
	});
	window.open(webhookTestUrl.value, '_blank', 'noreferrer');
};

const onLinkClick = (e: MouseEvent) => {
	if (!e.target) {
		return;
	}
	const target = e.target as HTMLElement;
	if (target.localName !== 'a') return;

	if (target.dataset?.key) {
		e.stopPropagation();
		e.preventDefault();

		if (target.dataset.key === 'activate') {
			emit('activate');
		} else if (target.dataset.key === 'executions') {
			telemetry.track('User clicked ndv link', {
				workflow_id: workflowsStore.workflowId,
				push_ref: props.pushRef,
				pane: 'input',
				type: 'open-executions-log',
			});
			ndvStore.activeNodeName = null;
			void router.push({
				name: VIEWS.EXECUTIONS,
			});
		} else if (target.dataset.key === 'settings') {
			uiStore.openModal(WORKFLOW_SETTINGS_MODAL_KEY);
		}
	}
};

const onTestLinkCopied = () => {
	telemetry.track('User copied webhook URL', {
		pane: 'inputs',
		type: 'test url',
	});
};

const onNodeExecute = () => {
	emit('execute');
};
</script>

<template>
	<div :class="$style.container">
		<transition name="fade" mode="out-in">
			<div v-if="hasIssues || hideContent" key="empty"></div>
			<div v-else-if="isListeningForEvents" key="listening">
				<n8n-pulse>
					<NodeIcon :node-type="nodeType" :size="40"></NodeIcon>
				</n8n-pulse>
				<div v-if="isWebhookNode">
					<n8n-text tag="div" size="large" color="text-dark" class="mb-2xs" bold>{{
						i18n.baseText('ndv.trigger.webhookNode.listening')
					}}</n8n-text>
					<div :class="[$style.shake, 'mb-xs']">
						<n8n-text>
							{{
								i18n.baseText('ndv.trigger.webhookNode.requestHint', {
									interpolate: { type: webhookHttpMethod ?? '' },
								})
							}}
						</n8n-text>
					</div>
					<CopyInput
						:value="webhookTestUrl"
						:toast-title="i18n.baseText('ndv.trigger.copiedTestUrl')"
						class="mb-2xl"
						size="medium"
						:collapse="true"
						:copy-button-text="i18n.baseText('generic.clickToCopy')"
						@copy="onTestLinkCopied"
					></CopyInput>
					<NodeExecuteButton
						data-test-id="trigger-execute-button"
						:node-name="nodeName"
						size="medium"
						telemetry-source="inputs"
						@execute="onNodeExecute"
					/>
				</div>
				<div v-else>
					<n8n-text tag="div" size="large" color="text-dark" class="mb-2xs" bold>{{
						listeningTitle
					}}</n8n-text>
					<div :class="[$style.shake, 'mb-xs']">
						<n8n-text tag="div">
							{{ listeningHint }}
						</n8n-text>
					</div>
					<div v-if="displayChatButton">
						<n8n-button class="mb-xl" @click="openWebhookUrl()">
							{{ i18n.baseText('ndv.trigger.chatTrigger.openChat') }}
						</n8n-button>
					</div>

					<NodeExecuteButton
						data-test-id="trigger-execute-button"
						:node-name="nodeName"
						size="medium"
						telemetry-source="inputs"
						@execute="onNodeExecute"
					/>
				</div>
			</div>
			<div v-else key="default">
				<div v-if="isActivelyPolling" class="mb-xl">
					<n8n-spinner type="ring" />
				</div>

				<div :class="$style.action">
					<div :class="$style.header">
						<n8n-heading v-if="header" tag="h1" bold>
							{{ header }}
						</n8n-heading>
						<n8n-text v-if="subheader">
							<span v-text="subheader" />
						</n8n-text>
					</div>

					<NodeExecuteButton
						data-test-id="trigger-execute-button"
						:node-name="nodeName"
						size="medium"
						telemetry-source="inputs"
						@execute="onNodeExecute"
					/>
				</div>

				<n8n-text v-if="activationHint" size="small" @click="onLinkClick">
					<span v-n8n-html="activationHint"></span>&nbsp;
				</n8n-text>
				<n8n-link
					v-if="activationHint && executionsHelp"
					size="small"
					@click="expandExecutionHelp"
					>{{ i18n.baseText('ndv.trigger.moreInfo') }}</n8n-link
				>
				<n8n-info-accordion
					v-if="executionsHelp"
					ref="help"
					:class="$style.accordion"
					:title="i18n.baseText('ndv.trigger.executionsHint.question')"
					:description="executionsHelp"
					:event-bus="executionsHelpEventBus"
					@click:body="onLinkClick"
				></n8n-info-accordion>
			</div>
		</transition>
	</div>
</template>

<style lang="scss" module>
.container {
	position: relative;
	width: 100%;
	height: 100%;
	background-color: var(--color-run-data-background);
	display: flex;
	flex-direction: column;

	align-items: center;
	justify-content: center;
	padding: var(--spacing-s) var(--spacing-s) var(--spacing-xl) var(--spacing-s);
	text-align: center;
	overflow: hidden;

	> * {
		width: 100%;
	}
}

.header {
	margin-bottom: var(--spacing-s);

	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.action {
	margin-bottom: var(--spacing-2xl);
}

.shake {
	animation: shake 8s infinite;
}

@keyframes shake {
	90% {
		transform: translateX(0);
	}
	92.5% {
		transform: translateX(6px);
	}
	95% {
		transform: translateX(-6px);
	}
	97.5% {
		transform: translateX(6px);
	}
	100% {
		transform: translateX(0);
	}
}

.accordion {
	position: absolute;
	bottom: 0;
	left: 0;
	width: 100%;
}
</style>

<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
	transition: opacity 200ms;
}
.fade-enter,
.fade-leave-to {
	opacity: 0;
}
</style>
