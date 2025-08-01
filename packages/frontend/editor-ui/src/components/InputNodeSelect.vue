<script lang="ts" setup>
import { useI18n } from '@n8n/i18n';
import { useNDVStore } from '@/stores/ndv.store';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { isPresent } from '@/utils/typesUtils';
import type { IConnectedNode, Workflow } from 'n8n-workflow';
import { computed } from 'vue';
import NodeIcon from './NodeIcon.vue';
import { truncate } from '@n8n/utils/string/truncate';

type Props = {
	nodes: IConnectedNode[];
	workflow: Workflow;
	modelValue: string | null;
};

const props = defineProps<Props>();

const emit = defineEmits<{
	'update:model-value': [value: string];
}>();

const i18n = useI18n();
const workflowsStore = useWorkflowsStore();
const nodeTypesStore = useNodeTypesStore();
const ndvStore = useNDVStore();

const selectedInputNode = computed(() => workflowsStore.getNodeByName(props.modelValue ?? ''));

const selectedInputNodeType = computed(() => {
	const node = selectedInputNode.value;
	if (!node) return null;

	return nodeTypesStore.getNodeType(node.type, node.typeVersion);
});

const inputNodes = computed(
	() =>
		props.nodes
			?.map((node) => {
				const fullNode = workflowsStore.getNodeByName(node.name);
				if (!fullNode) return null;

				return {
					node: fullNode,
					type: nodeTypesStore.getNodeType(fullNode.type, fullNode.typeVersion),
					depth: node.depth,
				};
			})
			.filter(isPresent) ?? [],
);

const activeNode = computed(() => ndvStore.activeNode);

const activeNodeType = computed(() => {
	const node = activeNode.value;
	if (!node) return null;

	return nodeTypesStore.getNodeType(node.type, node.typeVersion);
});

const isMultiInputNode = computed(() => {
	const nodeType = activeNodeType.value;
	return nodeType !== null && nodeType.inputs.length > 1;
});

const connectedTo = (nodeName: string) => {
	const connections = ndvStore.ndvNodeInputNumber[nodeName];
	if (!connections) return '';
	if (connections.length === 1) {
		return `Input ${ndvStore.ndvNodeInputNumber[nodeName]}`;
	}
	return `Inputs ${ndvStore.ndvNodeInputNumber[nodeName].join(', ')}`;
};

function getMultipleNodesText(nodeName: string): string {
	if (
		!nodeName ||
		!isMultiInputNode.value ||
		!activeNode.value ||
		!activeNodeType.value?.inputNames
	)
		return '';

	const activeNodeConnections =
		workflowsStore.connectionsByDestinationNode[activeNode.value.name].main || [];
	// Collect indexes of connected nodes
	const connectedInputIndexes = activeNodeConnections.reduce((acc: number[], node, index) => {
		if (node?.[0] && node[0].node === nodeName) return [...acc, index];
		return acc;
	}, []);

	// Match connected input indexes to their names specified by active node
	const connectedInputs = connectedInputIndexes.map(
		(inputIndex) => activeNodeType.value?.inputNames?.[inputIndex],
	);

	if (connectedInputs.length === 0) return '';

	return `(${connectedInputs.join(' & ')})`;
}

function title(nodeName: string, length = 30) {
	return truncate(nodeName, length);
}

function subtitle(nodeName: string, depth: number) {
	const multipleNodesText = getMultipleNodesText(nodeName);
	if (multipleNodesText) return multipleNodesText;
	return i18n.baseText('ndv.input.nodeDistance', { adjustToNumber: depth });
}

function onInputNodeChange(value: string) {
	emit('update:model-value', value);
}
</script>

<template>
	<n8n-select
		:model-value="modelValue"
		:no-data-text="i18n.baseText('ndv.input.noNodesFound')"
		:placeholder="i18n.baseText('ndv.input.parentNodes')"
		:class="$style.select"
		teleported
		size="small"
		filterable
		data-test-id="ndv-input-select"
		@update:model-value="onInputNodeChange"
	>
		<template #prefix>
			<NodeIcon
				:disabled="selectedInputNode?.disabled"
				:node-type="selectedInputNodeType"
				:size="14"
				:shrink="false"
			/>
		</template>

		<n8n-option
			v-for="{ node, type, depth } of inputNodes"
			:key="node.name"
			:value="node.name"
			:class="[$style.node, { [$style.disabled]: node.disabled }]"
			:label="`${title(node.name)} ${getMultipleNodesText(node.name)}`"
			data-test-id="ndv-input-option"
		>
			<NodeIcon
				:disabled="node.disabled"
				:node-type="type"
				:size="14"
				:shrink="false"
				:class="$style.icon"
			/>
			<span :class="$style.title">
				{{ title(node.name) }}
				<span v-if="node.disabled">({{ i18n.baseText('node.disabled') }})</span>
			</span>

			<span :class="$style.subtitle">{{
				connectedTo(node.name) ? connectedTo(node.name) : subtitle(node.name, depth)
			}}</span>
		</n8n-option>
	</n8n-select>
</template>

<style lang="scss" module>
.select {
	--max-select-width: 224px;
	max-width: var(--max-select-width);

	:global(.el-input--suffix .el-input__inner) {
		padding-left: calc(var(--spacing-l) + var(--spacing-4xs));
		padding-right: var(--spacing-l);
	}
}

.node {
	--select-option-padding: 0 var(--spacing-s);
	display: flex;
	align-items: center;
	font-size: var(--font-size-2xs);
	gap: var(--spacing-4xs);
}

.icon {
	padding-right: var(--spacing-4xs);
}

.title {
	color: var(--color-text-dark);
	font-weight: var(--font-weight-regular);
	max-width: var(--max-select-width);
	overflow: hidden;
	white-space: nowrap;
	text-overflow: ellipsis;
}

.disabled .title {
	color: var(--color-text-light);
}

.subtitle {
	margin-left: auto;
	padding-left: var(--spacing-2xs);
	color: var(--color-text-light);
	font-weight: var(--font-weight-regular);
}
</style>
