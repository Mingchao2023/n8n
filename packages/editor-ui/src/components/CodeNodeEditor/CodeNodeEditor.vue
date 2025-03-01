<template>
	<div
		:class="['code-node-editor', $style['code-node-editor-container'], language]"
		@mouseover="onMouseOver"
		@mouseout="onMouseOut"
		ref="codeNodeEditorContainer"
	>
		<el-tabs
			type="card"
			ref="tabs"
			v-model="activeTab"
			v-if="aiEnabled"
			:before-leave="onBeforeTabLeave"
		>
			<el-tab-pane
				:label="$locale.baseText('codeNodeEditor.tabs.code')"
				name="code"
				data-test-id="code-node-tab-code"
			>
				<div ref="codeNodeEditor" class="code-node-editor-input ph-no-capture code-editor-tabs" />
			</el-tab-pane>
			<el-tab-pane
				:label="$locale.baseText('codeNodeEditor.tabs.askAi')"
				name="ask-ai"
				data-test-id="code-node-tab-ai"
			>
				<!-- Key the AskAI tab to make sure it re-mounts when changing tabs -->
				<AskAI
					@replaceCode="onReplaceCode"
					:has-changes="hasChanges"
					:key="activeTab"
					@started-loading="isLoadingAIResponse = true"
					@finished-loading="isLoadingAIResponse = false"
				/>
			</el-tab-pane>
		</el-tabs>
		<!-- If AskAi not enabled, there's no point in rendering tabs -->
		<div v-else ref="codeNodeEditor" class="code-node-editor-input ph-no-capture" />
	</div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import type { PropType } from 'vue';
import jsParser from 'prettier/parser-babel';
import prettier from 'prettier/standalone';
import { mapStores } from 'pinia';
import type { LanguageSupport } from '@codemirror/language';
import type { Extension, Line } from '@codemirror/state';
import { Compartment, EditorState } from '@codemirror/state';
import type { ViewUpdate } from '@codemirror/view';
import { EditorView } from '@codemirror/view';
import { javascript } from '@codemirror/lang-javascript';
import { json } from '@codemirror/lang-json';
import { python } from '@codemirror/lang-python';
import type { CodeExecutionMode, CodeNodeEditorLanguage } from 'n8n-workflow';
import { CODE_EXECUTION_MODES, CODE_LANGUAGES } from 'n8n-workflow';

import { workflowHelpers } from '@/mixins/workflowHelpers'; // for json field completions
import { ASK_AI_EXPERIMENT, CODE_NODE_TYPE } from '@/constants';
import { codeNodeEditorEventBus } from '@/event-bus';
import { useRootStore, usePostHog } from '@/stores';

import { readOnlyEditorExtensions, writableEditorExtensions } from './baseExtensions';
import { CODE_PLACEHOLDERS } from './constants';
import { linterExtension } from './linter';
import { completerExtension } from './completer';
import { codeNodeEditorTheme } from './theme';
import AskAI from './AskAI/AskAI.vue';
import { useMessage } from '@/composables';

export default defineComponent({
	name: 'code-node-editor',
	mixins: [linterExtension, completerExtension, workflowHelpers],
	components: {
		AskAI,
	},
	props: {
		aiButtonEnabled: {
			type: Boolean,
			default: false,
		},
		mode: {
			type: String as PropType<CodeExecutionMode>,
			validator: (value: CodeExecutionMode): boolean => CODE_EXECUTION_MODES.includes(value),
		},
		language: {
			type: String as PropType<CodeNodeEditorLanguage>,
			default: 'javaScript' as CodeNodeEditorLanguage,
			validator: (value: CodeNodeEditorLanguage): boolean => CODE_LANGUAGES.includes(value),
		},
		isReadOnly: {
			type: Boolean,
			default: false,
		},
		modelValue: {
			type: String,
		},
	},
	setup() {
		return {
			...useMessage(),
		};
	},
	data() {
		return {
			editor: null as EditorView | null,
			languageCompartment: new Compartment(),
			linterCompartment: new Compartment(),
			isEditorHovered: false,
			isEditorFocused: false,
			tabs: ['code', 'ask-ai'],
			activeTab: 'code',
			hasChanges: false,
			isLoadingAIResponse: false,
		};
	},
	watch: {
		mode(newMode, previousMode: CodeExecutionMode) {
			this.reloadLinter();

			if (
				this.getCurrentEditorContent().trim() === CODE_PLACEHOLDERS[this.language]?.[previousMode]
			) {
				this.refreshPlaceholder();
			}
		},
		language(newLanguage, previousLanguage: CodeNodeEditorLanguage) {
			if (
				this.getCurrentEditorContent().trim() === CODE_PLACEHOLDERS[previousLanguage]?.[this.mode]
			) {
				this.refreshPlaceholder();
			}

			const [languageSupport] = this.languageExtensions;
			this.editor?.dispatch({
				effects: this.languageCompartment.reconfigure(languageSupport),
			});
		},
		aiEnabled: {
			immediate: true,
			async handler(isEnabled) {
				if (isEnabled && !this.modelValue) {
					this.$emit('update:modelValue', this.placeholder);
				}
				await this.$nextTick();
				this.hasChanges = this.modelValue !== this.placeholder;
			},
		},
	},
	computed: {
		...mapStores(useRootStore, usePostHog),
		aiEnabled(): boolean {
			const isAiExperimentEnabled = [ASK_AI_EXPERIMENT.gpt3, ASK_AI_EXPERIMENT.gpt4].includes(
				(this.posthogStore.getVariant(ASK_AI_EXPERIMENT.name) ?? '') as string,
			);

			return (
				isAiExperimentEnabled &&
				this.settingsStore.settings.ai.enabled &&
				this.language === 'javaScript'
			);
		},
		placeholder(): string {
			return CODE_PLACEHOLDERS[this.language]?.[this.mode] ?? '';
		},
		// eslint-disable-next-line vue/return-in-computed-property
		languageExtensions(): [LanguageSupport, ...Extension[]] {
			switch (this.language) {
				case 'json':
					return [json()];
				case 'javaScript':
					return [javascript(), this.autocompletionExtension('javaScript')];
				case 'python':
					return [python(), this.autocompletionExtension('python')];
			}
		},
	},
	methods: {
		getCurrentEditorContent() {
			return this.editor?.state.doc.toString() ?? '';
		},
		async onBeforeTabLeave(_activeName: string, oldActiveName: string) {
			// Confirm dialog if leaving ask-ai tab during loading
			if (oldActiveName === 'ask-ai' && this.isLoadingAIResponse) {
				const confirmModal = await this.alert(
					this.$locale.baseText('codeNodeEditor.askAi.sureLeaveTab'),
					{
						title: this.$locale.baseText('codeNodeEditor.askAi.areYouSure'),
						confirmButtonText: this.$locale.baseText('codeNodeEditor.askAi.switchTab'),
						showClose: true,
						showCancelButton: true,
					},
				);

				if (confirmModal === 'confirm') {
					return true;
				}

				return false;
			}

			return true;
		},
		async onReplaceCode(code: string) {
			const formattedCode = prettier.format(code, {
				parser: 'babel',
				plugins: [jsParser],
			});

			this.editor?.dispatch({
				changes: { from: 0, to: this.getCurrentEditorContent().length, insert: formattedCode },
			});

			this.activeTab = 'code';
			this.hasChanges = false;
		},
		onMouseOver(event: MouseEvent) {
			const fromElement = event.relatedTarget as HTMLElement;
			const ref = this.$refs.codeNodeEditorContainer as HTMLDivElement | undefined;

			if (!ref?.contains(fromElement)) this.isEditorHovered = true;
		},
		onMouseOut(event: MouseEvent) {
			const fromElement = event.relatedTarget as HTMLElement;
			const ref = this.$refs.codeNodeEditorContainer as HTMLDivElement | undefined;

			if (!ref?.contains(fromElement)) this.isEditorHovered = false;
		},
		reloadLinter() {
			if (!this.editor) return;

			const linter = this.createLinter(this.language);
			if (linter) {
				this.editor.dispatch({
					effects: this.linterCompartment.reconfigure(linter),
				});
			}
		},
		refreshPlaceholder() {
			if (!this.editor) return;

			this.editor.dispatch({
				changes: { from: 0, to: this.getCurrentEditorContent().length, insert: this.placeholder },
			});
		},
		line(lineNumber: number): Line | null {
			try {
				return this.editor?.state.doc.line(lineNumber) ?? null;
			} catch {
				return null;
			}
		},
		highlightLine(lineNumber: number | 'final') {
			if (!this.editor) return;

			if (lineNumber === 'final') {
				this.editor.dispatch({
					selection: { anchor: (this.modelValue ?? this.getCurrentEditorContent()).length },
				});
				return;
			}

			const line = this.line(lineNumber);

			if (!line) return;

			this.editor.dispatch({
				selection: { anchor: line.from },
			});
		},
		trackCompletion(viewUpdate: ViewUpdate) {
			const completionTx = viewUpdate.transactions.find((tx) => tx.isUserEvent('input.complete'));

			if (!completionTx) return;

			try {
				// @ts-ignore - undocumented fields
				const { fromA, toB } = viewUpdate?.changedRanges[0];
				const full = this.getCurrentEditorContent().slice(fromA, toB);
				const lastDotIndex = full.lastIndexOf('.');

				let context = null;
				let insertedText = null;

				if (lastDotIndex === -1) {
					context = '';
					insertedText = full;
				} else {
					context = full.slice(0, lastDotIndex);
					insertedText = full.slice(lastDotIndex + 1);
				}

				// TODO: Still has to get updated for Python and JSON
				this.$telemetry.track('User autocompleted code', {
					instance_id: this.rootStore.instanceId,
					node_type: CODE_NODE_TYPE,
					field_name: this.mode === 'runOnceForAllItems' ? 'jsCodeAllItems' : 'jsCodeEachItem',
					field_type: 'code',
					context,
					inserted_text: insertedText,
				});
			} catch {}
		},
	},
	beforeUnmount() {
		if (!this.isReadOnly) codeNodeEditorEventBus.off('error-line-number', this.highlightLine);
	},
	mounted() {
		if (!this.isReadOnly) codeNodeEditorEventBus.on('error-line-number', this.highlightLine);

		const { isReadOnly, language } = this;
		const extensions: Extension[] = [
			...readOnlyEditorExtensions,
			EditorState.readOnly.of(isReadOnly),
			EditorView.editable.of(!isReadOnly),
			codeNodeEditorTheme({ isReadOnly }),
		];

		if (!isReadOnly) {
			const linter = this.createLinter(language);
			if (linter) {
				extensions.push(this.linterCompartment.of(linter));
			}

			extensions.push(
				...writableEditorExtensions,
				EditorView.domEventHandlers({
					focus: () => {
						this.isEditorFocused = true;
					},
					blur: () => {
						this.isEditorFocused = false;
					},
				}),

				EditorView.updateListener.of((viewUpdate) => {
					if (!viewUpdate.docChanged) return;

					this.trackCompletion(viewUpdate);

					this.$emit('update:modelValue', this.editor?.state.doc.toString());
					this.hasChanges = true;
				}),
			);
		}

		const [languageSupport, ...otherExtensions] = this.languageExtensions;
		extensions.push(this.languageCompartment.of(languageSupport), ...otherExtensions);

		const state = EditorState.create({
			doc: this.modelValue ?? this.placeholder,
			extensions,
		});

		this.editor = new EditorView({
			parent: this.$refs.codeNodeEditor as HTMLDivElement,
			state,
		});

		// empty on first load, default param value
		if (!this.modelValue) {
			this.refreshPlaceholder();
			this.$emit('update:modelValue', this.placeholder);
		}
	},
});
</script>

<style scoped lang="scss">
:deep(.el-tabs) {
	.code-editor-tabs .cm-editor {
		border: 0;
	}
}
</style>

<style lang="scss" module>
.code-node-editor-container {
	position: relative;

	& > div {
		height: 100%;
	}
}

.ask-ai-button {
	position: absolute;
	top: var(--spacing-2xs);
	right: var(--spacing-2xs);
}
</style>
