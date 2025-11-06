<script setup lang='ts'>
import type {Ref} from 'vue'
import {computed, onMounted, onUnmounted, ref} from 'vue'
import {useRoute} from 'vue-router'
import {storeToRefs} from 'pinia'
import {NAutoComplete, NButton, NInput, useDialog, useMessage} from 'naive-ui'
import html2canvas from 'html2canvas'
import {Message} from './components'
import {useScroll} from './hooks/useScroll'
import {useChat} from './hooks/useChat'
import {useUsingContext} from './hooks/useUsingContext'
import HeaderComponent from './components/Header/index.vue'
import {HoverButton, SvgIcon} from '@/components/common'
import {useBasicLayout} from '@/hooks/useBasicLayout'
import {useChatStore, usePromptStore} from '@/store'
import {fetchChatAPIProcess} from '@/api'
import {t} from '@/locales'
import axios from "axios";

let controller = new AbortController()

const openLongReply = import.meta.env.VITE_GLOB_OPEN_LONG_REPLY === 'true'

const route = useRoute()
const dialog = useDialog()
const ms = useMessage()

const chatStore = useChatStore()

const {isMobile} = useBasicLayout()
const {addChat, updateChat, updateChatSome, getChatByUuidAndIndex} = useChat()
const {scrollRef, scrollToBottom, scrollToBottomIfAtBottom} = useScroll()
const {usingContext, toggleUsingContext} = useUsingContext()

const {uuid} = route.params as { uuid: string }

const dataSources = computed(() => chatStore.getChatByUuid(+uuid))
const conversationList = computed(() => dataSources.value.filter(item => (!item.inversion && !!item.conversationOptions)))

const prompt = ref<string>('')
const loading = ref<boolean>(false)
const inputRef = ref<Ref | null>(null)
const fileNames = ref<string[]>([])
const showFileList = ref(true)
const USER_UUID_KEY = 'chat_user_uuid'
const userUuid = ref<string>(localStorage.getItem(USER_UUID_KEY) ?? '')
const SERVICE_HTTP = import.meta.env.VITE_SERVICE_HTTP

if (!userUuid.value) {
	const gen = (typeof crypto !== 'undefined' && (crypto as any).randomUUID) ? (crypto as any).randomUUID() : `u_${Date.now()}_${Math.random().toString(36).slice(2, 10)}`
	userUuid.value = gen
	try {
		localStorage.setItem(USER_UUID_KEY, userUuid.value)
	} catch (e) {
		// ignore storage errors
	}
}

function toggleFileList() {
	showFileList.value = !showFileList.value
}

function removeFileName(filename: string) {
	const formData = new FormData()
	formData.set("name", filename)
	formData.set("userId", userUuid.value)

	axios.post(`${SERVICE_HTTP}/chatExcel/delete`, formData, {
		headers: { 'Content-Type': 'multipart/form-data' },
	})

	// 找到文件名在数组中的索引并删除
	const idx = fileNames.value.findIndex(name => name === filename)
	if (idx !== -1) {
		fileNames.value.splice(idx, 1)
	}
}

// 添加PromptStore
const promptStore = usePromptStore()

// 使用storeToRefs，保证store修改后，联想部分能够重新渲染
const refs = storeToRefs(promptStore) as unknown as { promptList: Ref<any[]> }
const promptTemplate = refs.promptList


// 未知原因刷新页面，loading 状态不会重置，手动重置
dataSources.value.forEach((item, index) => {
	if (item.loading)
		updateChatSome(+uuid, index, {loading: false})
})

function handleSubmit() {
	onConversation()
}

async function onConversation() {
	let message = prompt.value

	if (loading.value)
		return

	if (!message || message.trim() === '')
		return

	controller = new AbortController()

	addChat(
		+uuid,
		{
			dateTime: new Date().toLocaleString(),
			text: message,
			inversion: true,
			error: false,
			conversationOptions: null,
			requestOptions: {prompt: message, options: null},
		},
	)
	scrollToBottom()

	loading.value = true
	prompt.value = ''

	let options: Chat.ConversationRequest = {}
	const lastContext = conversationList.value[conversationList.value.length - 1]?.conversationOptions

	if (lastContext && usingContext.value)
		options = {...lastContext}

	addChat(
		+uuid,
		{
			dateTime: new Date().toLocaleString(),
			text: '',
			loading: true,
			inversion: false,
			error: false,
			conversationOptions: null,
			requestOptions: {prompt: message, options: {...options}},
		},
	)
	scrollToBottom()

	try {
		if (fileNames.value && fileNames.value.length > 0) {
			const resp = await fetch(`${SERVICE_HTTP}/chatExcel/query`, {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
				},
				body: JSON.stringify({
					query: message,
					userId: userUuid.value,
				}),
			})

			const data = await resp.json()
			const fullText = data

			updateChat(
				+uuid,
				dataSources.value.length - 1,
				{
					dateTime: new Date().toLocaleString(),
					text: fullText,
					inversion: false,
					error: false,
					loading: false, // 一次性返回，不需要流式状态
					conversationOptions: null,
					requestOptions: {prompt: message, options: {...options}},
				},
			)

			scrollToBottomIfAtBottom()
			return
		}

		let lastText = ''
			// attach user uuid to options so backend can identify requester
		;(options as any).userUuid = userUuid.value
		const fetchChatAPIOnce = async () => {
			await fetchChatAPIProcess<Chat.ConversationResponse>({
				prompt: message,
				options,
				signal: controller.signal,
				onDownloadProgress: ({event}) => {
					const xhr = event.target
					const {responseText} = xhr
					// Always process the final line
					const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
					let chunk = responseText
					if (lastIndex !== -1)
						chunk = responseText.substring(lastIndex)
					try {
						const data = JSON.parse(chunk)
						updateChat(
							+uuid,
							dataSources.value.length - 1,
							{
								dateTime: new Date().toLocaleString(),
								text: lastText + (data.text ?? ''),
								inversion: false,
								error: false,
								loading: true,
								conversationOptions: {conversationId: data.conversationId, parentMessageId: data.id},
								requestOptions: {prompt: message, options: {...options}},
							},
						)

						if (openLongReply && data.detail.choices[0].finish_reason === 'length') {
							options.parentMessageId = data.id
							lastText = data.text
							message = ''
							return fetchChatAPIOnce()
						}

						scrollToBottomIfAtBottom()
					} catch (error) {
						//
					}
				},
			})
			updateChatSome(+uuid, dataSources.value.length - 1, {loading: false})
		}

		await fetchChatAPIOnce()
	} catch (error: any) {
		const errorMessage = error?.message ?? t('common.wrong')

		if (error.message === 'canceled') {
			updateChatSome(
				+uuid,
				dataSources.value.length - 1,
				{
					loading: false,
				},
			)
			scrollToBottomIfAtBottom()
			return
		}

		const currentChat = getChatByUuidAndIndex(+uuid, dataSources.value.length - 1)

		if (currentChat?.text && currentChat.text !== '') {
			updateChatSome(
				+uuid,
				dataSources.value.length - 1,
				{
					text: `${currentChat.text}\n[${errorMessage}]`,
					error: false,
					loading: false,
				},
			)
			return
		}

		updateChat(
			+uuid,
			dataSources.value.length - 1,
			{
				dateTime: new Date().toLocaleString(),
				text: errorMessage,
				inversion: false,
				error: true,
				loading: false,
				conversationOptions: null,
				requestOptions: {prompt: message, options: {...options}},
			},
		)
		scrollToBottomIfAtBottom()
	} finally {
		loading.value = false
	}
}

async function onRegenerate(index: number) {
	if (loading.value)
		return

	controller = new AbortController()

	const {requestOptions} = dataSources.value[index]

	let message = requestOptions?.prompt ?? ''

	let options: Chat.ConversationRequest = {}

	if (requestOptions.options)
		options = {...requestOptions.options}
		// ensure regenerate requests also carry user uuid
		;
	(options as any).userUuid = userUuid.value

	loading.value = true

	updateChat(
		+uuid,
		index,
		{
			dateTime: new Date().toLocaleString(),
			text: '',
			inversion: false,
			error: false,
			loading: true,
			conversationOptions: null,
			requestOptions: {prompt: message, options: {...options}},
		},
	)

	try {
		let lastText = ''
		const fetchChatAPIOnce = async () => {
			await fetchChatAPIProcess<Chat.ConversationResponse>({
				prompt: message,
				options,
				signal: controller.signal,
				onDownloadProgress: ({event}) => {
					const xhr = event.target
					const {responseText} = xhr
					// Always process the final line
					const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
					let chunk = responseText
					if (lastIndex !== -1)
						chunk = responseText.substring(lastIndex)
					try {
						const data = JSON.parse(chunk)
						updateChat(
							+uuid,
							index,
							{
								dateTime: new Date().toLocaleString(),
								text: lastText + (data.text ?? ''),
								inversion: false,
								error: false,
								loading: true,
								conversationOptions: {conversationId: data.conversationId, parentMessageId: data.id},
								requestOptions: {prompt: message, options: {...options}},
							},
						)

						if (openLongReply && data.detail.choices[0].finish_reason === 'length') {
							options.parentMessageId = data.id
							lastText = data.text
							message = ''
							return fetchChatAPIOnce()
						}
					} catch (error) {
						//
					}
				},
			})
			updateChatSome(+uuid, index, {loading: false})
		}
		await fetchChatAPIOnce()
	} catch (error: any) {
		if (error.message === 'canceled') {
			updateChatSome(
				+uuid,
				index,
				{
					loading: false,
				},
			)
			return
		}

		const errorMessage = error?.message ?? t('common.wrong')

		updateChat(
			+uuid,
			index,
			{
				dateTime: new Date().toLocaleString(),
				text: errorMessage,
				inversion: false,
				error: true,
				loading: false,
				conversationOptions: null,
				requestOptions: {prompt: message, options: {...options}},
			},
		)
	} finally {
		loading.value = false
	}
}

function handleExport() {
	if (loading.value)
		return

	const d = dialog.warning({
		title: t('chat.exportImage'),
		content: t('chat.exportImageConfirm'),
		positiveText: t('common.yes'),
		negativeText: t('common.no'),
		onPositiveClick: async () => {
			try {
				d.loading = true
				const ele = document.getElementById('image-wrapper')
				const canvas = await html2canvas(ele as HTMLDivElement, {
					useCORS: true,
				})
				const imgUrl = canvas.toDataURL('image/png')
				const tempLink = document.createElement('a')
				tempLink.style.display = 'none'
				tempLink.href = imgUrl
				tempLink.setAttribute('download', 'chat-shot.png')
				if (typeof tempLink.download === 'undefined')
					tempLink.setAttribute('target', '_blank')

				document.body.appendChild(tempLink)
				tempLink.click()
				document.body.removeChild(tempLink)
				window.URL.revokeObjectURL(imgUrl)
				d.loading = false
				ms.success(t('chat.exportSuccess'))
				Promise.resolve()
			} catch (error: any) {
				ms.error(t('chat.exportFailed'))
			} finally {
				d.loading = false
			}
		},
	})
}

async function handleUpload() {
	if (loading.value) return

	const input = document.createElement('input')
	input.type = 'file'
	input.accept = '.xls,.xlsx'
	input.multiple = true // ✅ 支持多文件

	input.onchange = async (e: Event) => {
		const target = e.target as HTMLInputElement | null
		const files = Array.from(target?.files ?? []) as File[]
		const updatedFiles = files.map(file => {
			const newName = `${file.name}`
			return new File([file], newName, {type: file.type})
		})
		if (!files.length) return

		const maxSizeMB = 100
		for (const file of updatedFiles) {
			if (file.size / 1024 / 1024 > maxSizeMB) {
				window.$message?.error(`文件 "${file.name}" 大小超过 ${maxSizeMB}MB`)
				return
			}
		}

		loading.value = true
		try {
			const formData = new FormData()
			formData.append('userId', userUuid.value)
			updatedFiles.forEach(file => {
				formData.append('files', file)
			})
			const res = await axios.post(`${SERVICE_HTTP}/chatExcel/upload`, formData, {
				headers: {'Content-Type': 'multipart/form-data'},
			})
			if (res.data?.success) {
				window.$message?.success('上传成功')
				if (Array.isArray(res.data.result)) {
					fileNames.value = [...fileNames.value, ...res.data.result]
				}
			} else {
				window.$message?.error(res.data?.message || '上传失败')
			}
		} catch (err) {
			console.error('上传出错:', err)
			window.$message?.error('上传失败')
		} finally {
			loading.value = false
		}
	}
	input.click()
	showFileList.value = true
}

function handleDelete(index: number) {
	if (loading.value)
		return

	dialog.warning({
		title: t('chat.deleteMessage'),
		content: t('chat.deleteMessageConfirm'),
		positiveText: t('common.yes'),
		negativeText: t('common.no'),
		onPositiveClick: () => {
			chatStore.deleteChatByUuid(+uuid, index)
		},
	})
}

function handleClear() {
	if (loading.value)
		return

	dialog.warning({
		title: t('chat.clearChat'),
		content: t('chat.clearChatConfirm'),
		positiveText: t('common.yes'),
		negativeText: t('common.no'),
		onPositiveClick: () => {
			chatStore.clearChatByUuid(+uuid)
		},
	})
}

function handleEnter(event: KeyboardEvent) {
	if (!isMobile.value) {
		if (event.key === 'Enter' && !event.shiftKey) {
			event.preventDefault()
			handleSubmit()
		}
	} else {
		if (event.key === 'Enter' && event.ctrlKey) {
			event.preventDefault()
			handleSubmit()
		}
	}
}

function handleStop() {
	if (loading.value) {
		controller.abort()
		loading.value = false
	}
}

// 可优化部分
// 搜索选项计算，这里使用value作为索引项，所以当出现重复value时渲染异常(多项同时出现选中效果)
// 理想状态下其实应该是key作为索引项,但官方的renderOption会出现问题，所以就需要value反renderLabel实现
const searchOptions = computed(() => {
	if (prompt.value.startsWith('/')) {
		return promptTemplate.value.filter((item: {
			key: string
		}) => item.key.toLowerCase().includes(prompt.value.substring(1).toLowerCase())).map((obj: { value: any }) => {
			return {
				label: obj.value,
				value: obj.value,
			}
		})
	} else {
		return []
	}
})

// value反渲染key
const renderOption = (option: { label: string }) => {
	for (const i of promptTemplate.value) {
		if (i.value === option.label)
			return [i.key]
	}
	return []
}

const placeholder = computed(() => {
	if (isMobile.value)
		return t('chat.placeholderMobile')
	return t('chat.placeholder')
})

const buttonDisabled = computed(() => {
	return loading.value || !prompt.value || prompt.value.trim() === ''
})

const footerClass = computed(() => {
	let classes = ['p-4']
	if (isMobile.value)
		classes = ['sticky', 'left-0', 'bottom-0', 'right-0', 'p-2', 'pr-3', 'overflow-hidden']
	return classes
})

onMounted(() => {
	scrollToBottom()
	if (inputRef.value && !isMobile.value)
		inputRef.value?.focus()
})

onUnmounted(() => {
	if (loading.value)
		controller.abort()
})
</script>

<template>
	<div class="flex flex-col w-full h-full">
		<HeaderComponent
			v-if="isMobile"
			:using-context="usingContext"
			@export="handleExport"
			@toggle-using-context="toggleUsingContext"
		/>
		<main class="flex-1 overflow-hidden">
			<div id="scrollRef" ref="scrollRef" class="h-full overflow-hidden overflow-y-auto">
				<div
					id="image-wrapper"
					class="w-full max-w-screen-xl m-auto dark:bg-[#101014]"
					:class="[isMobile ? 'p-2' : 'p-[50px]']"
				>
					<template v-if="!dataSources.length">
						<div class="flex items-center justify-center mt-4 text-center text-neutral-300">
							<SvgIcon icon="ri:bubble-chart-fill" class="mr-2 text-3xl"/>
							<span>Aha~</span>
						</div>
					</template>
					<template v-else>
						<div>
							<Message
								v-for="(item, index) of dataSources"
								:key="index"
								:date-time="item.dateTime"
								:text="item.text"
								:inversion="item.inversion"
								:error="item.error"
								:loading="item.loading"
								@regenerate="onRegenerate(index)"
								@delete="handleDelete(index)"
							/>
							<div class="sticky bottom-[50px] left-0 flex justify-center">
								<NButton v-if="loading" type="warning" @click="handleStop">
									<template #icon>
										<SvgIcon icon="ri:stop-circle-line"/>
									</template>
									Stop Responding
								</NButton>
							</div>
						</div>
					</template>
				</div>
			</div>
		</main>
		<footer :class="footerClass">
			<div class="w-full max-w-screen-xl m-auto">
				<div class="flex items-center justify-between space-x-2">
					<HoverButton @click="handleClear">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:delete-bin-line"/>
            </span>
					</HoverButton>
					<HoverButton v-if="!isMobile" @click="handleExport">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:download-2-line"/>
            </span>
					</HoverButton>
					<HoverButton v-if="!isMobile" @click="toggleUsingContext">
            <span class="text-xl" :class="{ 'text-[#4b9e5f]': usingContext, 'text-[#a8071a]': !usingContext }">
              <SvgIcon icon="ri:chat-history-line"/>
            </span>
					</HoverButton>
					<!-- 文件列表弹窗组件，显示在上传按钮上方 -->
					<div v-if="fileNames.length > 0 && showFileList && !isMobile"
							 style="position: absolute; bottom: 60px; left: 0; right: 0; margin: auto; z-index: 1000; max-width: 800px; width: 100%; background: #fff; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15); border: 1px solid #eee; ">
						<div
							style="display: flex; flex-wrap: nowrap; gap: 8px; align-items: center; min-height: 40px; border: 1px solid #d9d9d9; border-radius: 6px; padding: 6px; background: #fafafa; overflow-x: auto; max-width: 100%;">
                <span v-for="name in fileNames" :key="name"
											style="display: flex; align-items: center; background: #f5f5f5; border: 1px solid #e0e0e0; border-radius: 4px; padding: 2px 8px; font-size: 13px; color: #333; margin-right: 0; margin-bottom: 0; white-space: nowrap;">
                  <span style="max-width: 160px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;">{{
											name
										}}</span>
                  <NButton size="tiny" type="error" style="margin-left: 6px;"
													 @click="removeFileName(name)">删除</NButton>
                </span>
						</div>
					</div>
					<HoverButton v-if="!isMobile" @click="handleUpload">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:upload-2-line"/>
            </span>
					</HoverButton>
					<HoverButton v-if="fileNames.length > 0 && !isMobile" @click="toggleFileList">
            <span class="text-xl text-[#4f555e] dark:text-white">
              <SvgIcon icon="ri:file-2-line"/>
            </span>
					</HoverButton>
					<NAutoComplete v-model:value="prompt" :options="searchOptions" :render-label="renderOption">
						<template #default="{ handleInput, handleBlur, handleFocus }">
							<NInput
								ref="inputRef"
								v-model:value="prompt"
								type="textarea"
								:placeholder="placeholder"
								:autosize="{ minRows: 1, maxRows: isMobile ? 4 : 8 }"
								@input="handleInput"
								@focus="handleFocus"
								@blur="handleBlur"
								@keypress="handleEnter"
							/>
						</template>
					</NAutoComplete>
					<NButton type="primary" :disabled="buttonDisabled" @click="handleSubmit">
						<template #icon>
              <span class="dark:text-black">
                <SvgIcon icon="ri:send-plane-fill"/>
              </span>
						</template>
					</NButton>
				</div>
			</div>
		</footer>
	</div>
</template>
