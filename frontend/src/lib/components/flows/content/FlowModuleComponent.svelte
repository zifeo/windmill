<script lang="ts">
	import { Pane, Splitpanes } from 'svelte-splitpanes'
	import Tab from '$lib/components/common/tabs/Tab.svelte'
	import Tabs from '$lib/components/common/tabs/Tabs.svelte'
	import Editor from '$lib/components/Editor.svelte'
	import EditorBar from '$lib/components/EditorBar.svelte'
	import ModulePreview from '$lib/components/ModulePreview.svelte'
	import { createScriptFromInlineScript, fork } from '$lib/components/flows/flowStateUtils'

	import { RawScript, type FlowModule } from '$lib/gen'
	import FlowCard from '../common/FlowCard.svelte'
	import FlowModuleHeader from './FlowModuleHeader.svelte'
	import { getLatestHashForScript, scriptLangToEditorLang } from '$lib/scripts'
	import PropPickerWrapper from '../propPicker/PropPickerWrapper.svelte'
	import { afterUpdate, getContext, tick } from 'svelte'
	import type { FlowEditorContext } from '../types'
	import { loadSchemaFromModule } from '../utils'
	import FlowModuleScript from './FlowModuleScript.svelte'
	import FlowModuleEarlyStop from './FlowModuleEarlyStop.svelte'
	import FlowModuleSuspend from './FlowModuleSuspend.svelte'
	import FlowRetries from './FlowRetries.svelte'
	import { getStepPropPicker } from '../previousResults'
	import { deepEqual } from 'fast-equals'

	import Button from '$lib/components/common/button/Button.svelte'
	import Alert from '$lib/components/common/alert/Alert.svelte'
	import FlowModuleSleep from './FlowModuleSleep.svelte'
	import FlowPathViewer from './FlowPathViewer.svelte'
	import InputTransformSchemaForm from '$lib/components/InputTransformSchemaForm.svelte'
	import { schemaToObject } from '$lib/schema'

	const { selectedId, previewArgs, flowStateStore, flowStore, saveDraft } =
		getContext<FlowEditorContext>('FlowEditorContext')

	export let flowModule: FlowModule
	export let failureModule: boolean = false

	export let parentModule: FlowModule | undefined = undefined
	export let previousModule: FlowModule | undefined

	let editor: Editor
	let modulePreview: ModulePreview
	let websocketAlive = {
		pyright: false,
		black: false,
		deno: false,
		go: false,
		ruff: false,
		shellcheck: false
	}
	let selected = 'inputs'
	let advancedSelected = 'retries'
	let wrapper: HTMLDivElement
	let panes: HTMLElement
	let totalTopGap = 0
	let validCode = true
	let width = 1200

	$: stepPropPicker = failureModule
		? {
				pickableProperties: {
					flow_input: schemaToObject($flowStore.schema as any, $previewArgs),
					priorIds: {},
					previousId: undefined,
					hasResume: false
				},
				extraLib: ''
		  }
		: getStepPropPicker(
				$flowStateStore,
				parentModule,
				previousModule,
				flowModule.id,
				$flowStore,
				$previewArgs,
				false
		  )

	function onKeyDown(event: KeyboardEvent) {
		if ((event.ctrlKey || event.metaKey) && event.key == 'Enter') {
			event.preventDefault()
			selected = 'test'
			modulePreview?.runTestWithStepArgs()
		}
	}

	let inputTransformSchemaForm: InputTransformSchemaForm | undefined = undefined
	async function reload(flowModule: FlowModule) {
		try {
			const { input_transforms, schema } = await loadSchemaFromModule(flowModule)
			validCode = true

			inputTransformSchemaForm?.setArgs(input_transforms)

			await tick()
			if (!deepEqual(schema, $flowStateStore[flowModule.id]?.schema)) {
				if (!$flowStateStore[flowModule.id]) {
					$flowStateStore[flowModule.id] = { schema }
				} else {
					$flowStateStore[flowModule.id].schema = schema
				}
			}
		} catch (e) {
			validCode = false
		}
	}

	function selectAdvanced(subtab: string) {
		selected = 'advanced'
		advancedSelected = subtab
	}

	afterUpdate(() => {
		totalTopGap = 0
		if (!(wrapper && panes)) return

		const wrapperTop = wrapper.getBoundingClientRect().top
		const panesTop = panes.getBoundingClientRect().top
		totalTopGap = panesTop - wrapperTop
	})

	let forceReload = 0
</script>

<svelte:window on:keydown={onKeyDown} />

{#if flowModule.value}
	<div class="h-full" bind:this={wrapper} bind:clientWidth={width}>
		<FlowCard bind:flowModule>
			<svelte:fragment slot="header">
				<FlowModuleHeader
					bind:module={flowModule}
					on:toggleSuspend={() => selectAdvanced('suspend')}
					on:toggleSleep={() => selectAdvanced('sleep')}
					on:toggleRetry={() => selectAdvanced('retries')}
					on:toggleStopAfterIf={() => selectAdvanced('early-stop')}
					on:fork={async () => {
						const [module, state] = await fork(flowModule)
						flowModule = module
						$flowStateStore[module.id] = state
					}}
					on:reload={async () => {
						if (flowModule.value.type == 'script') {
							console.log('reload')
							if (flowModule.value.hash != undefined) {
								flowModule.value.hash = await getLatestHashForScript(flowModule.value.path)
							}
							forceReload++
							await reload(flowModule)
						}
					}}
					on:createScriptFromInlineScript={async () => {
						const [module, state] = await createScriptFromInlineScript(
							flowModule,
							$selectedId,
							$flowStateStore[flowModule.id].schema,
							$flowStore
						)
						flowModule = module
						$flowStateStore[module.id] = state
					}}
				/>
			</svelte:fragment>

			{#if flowModule.value.type === 'rawscript'}
				<div class="border-b-2 shadow-sm px-1">
					<EditorBar
						{validCode}
						{editor}
						lang={flowModule.value['language'] ?? 'deno'}
						{websocketAlive}
						iconOnly={width < 850}
					/>
				</div>
			{/if}

			<div
				bind:this={panes}
				class="h-full"
				style="max-height: calc(100% - {totalTopGap}px) !important;"
			>
				<Splitpanes horizontal>
					<Pane size={flowModule.value.type == 'script' ? 30 : 50} minSize={20}>
						{#if flowModule.value.type === 'rawscript'}
							{#key flowModule.id}
								<Editor
									path={flowModule.value.path}
									bind:websocketAlive
									bind:this={editor}
									class="h-full relative"
									bind:code={flowModule.value.content}
									deno={flowModule.value.language === RawScript.language.DENO}
									lang={scriptLangToEditorLang(flowModule.value.language)}
									automaticLayout={true}
									cmdEnterAction={async () => {
										selected = 'test'
										if ($selectedId == flowModule.id) {
											if (flowModule.value.type === 'rawscript') {
												flowModule.value.content = editor.getCode()
											}
											await reload(flowModule)
											modulePreview?.runTestWithStepArgs()
										}
									}}
									on:change={async (event) => {
										if (flowModule.value.type === 'rawscript') {
											flowModule.value.content = event.detail
										}
										await reload(flowModule)
									}}
									formatAction={() => {
										reload(flowModule)
										saveDraft()
									}}
									fixedOverflowWidgets={true}
								/>
							{/key}
						{:else if flowModule.value.type === 'script'}
							{#key forceReload}
								<FlowModuleScript path={flowModule.value.path} hash={flowModule.value.hash} />
							{/key}
						{:else if flowModule.value.type === 'flow'}
							<FlowPathViewer path={flowModule.value.path} />
						{/if}
					</Pane>
					<Pane size={flowModule.value.type == 'script' ? 70 : 50} minSize={20}>
						<Tabs bind:selected>
							<Tab value="inputs"><span class="font-semibold">Step Input</span></Tab>
							<Tab value="test"><span class="font-semibold text-md">Test this step</span></Tab>
							<Tab value="advanced">Advanced</Tab>
						</Tabs>
						<div class="h-[calc(100%-32px)]">
							{#if selected === 'inputs' && (flowModule.value.type == 'rawscript' || flowModule.value.type == 'script' || flowModule.value.type == 'flow')}
								<div class="h-full overflow-auto">
									<PropPickerWrapper
										pickableProperties={stepPropPicker.pickableProperties}
										error={failureModule}
									>
										<InputTransformSchemaForm
											bind:this={inputTransformSchemaForm}
											schema={$flowStateStore[$selectedId]?.schema ?? {}}
											previousModuleId={previousModule?.id}
											bind:args={flowModule.value.input_transforms}
											extraLib={stepPropPicker.extraLib}
										/>
									</PropPickerWrapper>
								</div>
							{:else if selected === 'test'}
								<ModulePreview
									pickableProperties={stepPropPicker.pickableProperties}
									bind:this={modulePreview}
									mod={flowModule}
									schema={$flowStateStore[$selectedId]?.schema ?? {}}
								/>
							{:else if selected === 'advanced'}
								<Tabs bind:selected={advancedSelected}>
									<Tab value="retries">Retries</Tab>
									{#if !$selectedId.includes('failure')}
										<Tab value="early-stop">Early Stop/Break</Tab>
										<Tab value="suspend">Suspend</Tab>
										<Tab value="sleep">Sleep</Tab>
										<Tab value="same_worker">Shared Directory</Tab>
									{/if}
								</Tabs>
								<div class="h-[calc(100%-32px)] overflow-auto p-4">
									{#if advancedSelected === 'retries'}
										<FlowRetries bind:flowModule />
									{:else if advancedSelected === 'early-stop'}
										<FlowModuleEarlyStop bind:flowModule />
									{:else if advancedSelected === 'suspend'}
										<div>
											<FlowModuleSuspend bind:flowModule />
										</div>
									{:else if advancedSelected === 'sleep'}
										<div>
											<FlowModuleSleep previousModuleId={previousModule?.id} bind:flowModule />
										</div>
									{:else if advancedSelected === 'same_worker'}
										<div>
											<Alert type="info" title="Share a directory between steps">
												If shared directory is set, will share a folder that will be mounted on
												`./shared` for each of them to pass data between each other.
											</Alert>
											<Button
												btnClasses="mt-4"
												on:click={() => {
													$selectedId = 'settings-same-worker'
												}}>Set shared directory in the flow settings</Button
											>
										</div>
									{/if}
								</div>
							{/if}
						</div>
					</Pane>
				</Splitpanes>
			</div>
		</FlowCard>
	</div>
{:else}
	Incorrect flow module type
{/if}
