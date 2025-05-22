<template>
	<div class="floating_wrap" :style="draggingStyle" ref="floatingWrap" @mousedown="startDrag">
		<div class="floating_cont">
			<div class="floating_item" ref="floatingItem">
				<SuppPopover
					v-model:open="isFloatingMenuOpen"
					placement="topLeft"
					trigger="manual"
					:destroyTooltipOnHide="true"
					overlayClassName="chatbot_dropdown"
					@click="handleClick"
				>
					<SuppButton class="btn_chatbot"><i class="ico chatbot"></i></SuppButton>
					<template #content>
						<div class="chatbot_cont">
							<div class="chatbot_top_area">
								<p class="chatbot_top_title">AI 챗봇 <strong>지피</strong></p>
							</div>
							<div class="chatbot_cont_inner">
								<div ref="scrollRef" class="chatbot_msg_area">
									<!-- 초기 챗봇 메세지 : S -->
									<div class="chat_msg chatbot start">
										<div class="msg">안녕하세요? 👋</div>
									</div>
									<div class="chat_msg chatbot start">
										<div class="msg">
											{{ userInfo.userNm }}님! <br />
											GPRO AI Chatbot 지피입니다. <br />
											무엇을 도와드릴까요?
										</div>
									</div>
									<div
										class="chat_msg chatbot start"
										v-if="commCodes.chatbotServiceDsceList.length > 0"
									>
										<div class="msg">자주 이용하신 서비스를 보여드릴게요.</div>
										<div class="chat_btn_area">
											<button
												type="button"
												v-for="(commCode, index) in commCodes.chatbotServiceDsceList"
												:key="index"
												@click="reqChat(commCode.label)"
											>
												{{ commCode.label }}
											</button>
										</div>
									</div>
									<!-- 초기 챗봇 메세지 : E -->

									<div
										v-for="(message, index) in chatbotMessage"
										:key="index"
										:class="['chat_msg', message.type]"
									>
										<div class="msg">{{ message.msg }}</div>
										<div class="time">{{ formatTime(message.time) }}</div>
									</div>

									<!-- 챗봇 메세지 로딩 : S -->
									<div class="chat_msg chatbot" v-if="isSendingMessage">
										<div class="msg">
											<div class="chat_loading_area">
												<div class="chat_loading"></div>
											</div>
										</div>
										<div class="time"></div>
									</div>
									<!-- 챗봇 메세지 로딩 : E -->
								</div>
							</div>
							<!-- //chatbot_cont_inner -->
							<div :class="['chatbot_btm_area', { on: isVoiceActive }]">
								<SuppButton class="btn_voice" @click="toggleVoice"
									><i class="ico voice"></i
								></SuppButton>
								<div class="chatbot_ipt_area">
									<div class="voice_record_area">
										<div class="voice_record">
											<div class="voice_record_bar">
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
											</div>
											<span class="voice_record_text">듣고 있어요</span>
											<div class="voice_record_bar">
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
												<div class="bar"></div>
											</div>
										</div>
										<SuppButton class="btn_voice_stop" @click="toggleVoice"
											><i class="ico voice_stop"></i
										></SuppButton>
									</div>
									<SuppInput
										v-model:value="searchValue"
										inputType="search"
										placeholder="궁금한점을 입력하세요."
										@search="clickSave"
										:disabled="isVoiceActive || isSendingMessage"
									/>
								</div>
							</div>
							<!-- //chatbot_btm_area -->
						</div>
					</template>
				</SuppPopover>
			</div>
		</div>
	</div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch } from 'vue';
const authStore = useAuthStore();
const alertStore = useAlertStore();
const { userInfo } = storeToRefs(authStore);

// 공통코드
const commCodes = reactive({
	chatbotServiceDsceList: [],
});

const scrollRef = ref(null);

// 하단 음성인식 버튼 클릭
let recognition = null;
const searchValue = ref('');
const isVoiceActive = ref(false);

const isSendingMessage = ref(false);
const chatbotMessage = ref([]);

const formatTime = date => {
	let hours = date.getHours();
	const minutes = date.getMinutes();
	const isPM = hours >= 12;
	const period = isPM ? '오후' : '오전';

	hours = hours % 12;
	if (hours === 0) hours = 12;

	return `${period} ${hours}:${minutes.toString().padStart(2, '0')}`;
};

const clickSave = async () => {
	if (!searchValue.value) {
		// 포커스 된 행 없음
		alertStore.openAlert({
			// text: $t('alert.i.admin.menu.message6'),
			text: '메시지를 입력해 주세요',
		});
		return;
	}

	await reqChat(searchValue.value);
	searchValue.value = '';
};

const reqChat = async msg => {
	chatbotMessage.value.push({
		type: 'user',
		msg: msg,
		time: new Date(),
	});

	isSendingMessage.value = true;

	await nextTick();
	scrollRef.value.scrollTop = scrollRef.value.scrollHeight;

	await aiChatbotApi
		.reqChat({
			user_id: authStore.getLoginInfo().value.localLoginId,
			message: msg,
		})
		.then(async response => {
			let answer = '';
			if (response.status === 200 && response.data.resultCode === 200) {
				const resultInfo = JSON.parse(response.data.answer);
				answer = resultInfo.answer;
			} else {
				answer = '서비스 요청 중 오류가 발생했습니다.';
			}

			chatbotMessage.value.push({
				type: 'chatbot',
				msg: answer,
				time: new Date(),
			});

			// TTS 음성 응답
			const synth = window.speechSynthesis;
			const utter = new SpeechSynthesisUtterance(answer);
			utter.lang = 'ko-KR';
			synth.speak(utter);
		})
		.catch(() => {
			chatbotMessage.value.push({
				type: 'chatbot',
				msg: '서비스 요청 중 오류가 발생했습니다.',
				time: new Date(),
			});
		});
	isSendingMessage.value = false;
	scrollRef.value.scrollTop = scrollRef.value.scrollHeight;
};

// 챗봇 popover show/hide
const isFloatingMenuOpen = ref(false);

function toggleVoice() {
	if (!recognition) return;

	if (isVoiceActive.value) {
		recognition.stop();
		isVoiceActive.value = false;
	} else {
		recognition.start();
		isVoiceActive.value = true;
	}
}

// 챗봇 드래그
const dragging = ref(false);
const wasDragged = ref(false);
const useAbsolute = ref(false);
const position = reactive({ x: 0, y: 0 });
const offset = reactive({ x: 0, y: 0 });
const floatingWrap = ref(null);
const floatingItem = ref(null);

function startDrag(e) {
	const rect = floatingWrap.value.getBoundingClientRect();
	position.x = rect.left;
	position.y = rect.top;
	offset.x = e.clientX - rect.left;
	offset.y = e.clientY - rect.top;

	useAbsolute.value = true;
	dragging.value = true;
	wasDragged.value = false;

	window.addEventListener('mousemove', onDrag);
	window.addEventListener('mouseup', stopDrag);
}

function onDrag(e) {
	if (!dragging.value) return;
	wasDragged.value = true;

	const el = floatingWrap.value;
	const elWidth = el.offsetWidth;
	const elHeight = el.offsetHeight;

	let newX = e.clientX - offset.x;
	let newY = e.clientY - offset.y;

	newX = Math.max(0, Math.min(newX, window.innerWidth - elWidth));
	newY = Math.max(0, Math.min(newY, window.innerHeight - elHeight));

	position.x = newX;
	position.y = newY;
}

function stopDrag() {
	dragging.value = false;
	window.removeEventListener('mousemove', onDrag);
	window.removeEventListener('mouseup', stopDrag);
}

// 버튼 클릭 시 팝업 토글, 드래그 후 클릭은 무시
function handleClick(e) {
	if (wasDragged.value) {
		e.preventDefault();
		e.stopPropagation();
		return;
	}
	isFloatingMenuOpen.value = !isFloatingMenuOpen.value;
}

// 외부 클릭 시 Popover 닫기
function handleOutsideClick(e) {
	const target = e.target;
	if (floatingItem.value && !floatingItem.value.contains(target)) {
		// isFloatingMenuOpen.value = false;
	}
}

watch(isFloatingMenuOpen, open => {
	if (open) {
		document.addEventListener('click', handleOutsideClick);
	} else {
		document.removeEventListener('click', handleOutsideClick);
	}
});

onBeforeUnmount(() => {
	document.removeEventListener('click', handleOutsideClick);
});

const draggingStyle = computed(() => {
	if (useAbsolute.value) {
		return {
			position: 'fixed',
			left: position.x + 'px',
			right: 'auto',
			bottom: 'auto',
			top: position.y + 'px',
			zIndex: 1111,
		};
	} else {
		return {};
	}
});

// 공통코드 조회
const getCommCodeList = async code => {
	await codeApi.selectGlobalMultiCodeList({ cdGrpIds: code }).then(response => {
		const multiCodeList = response.data;
		const selectOptionsList = $comm.convertToSelectOptions(multiCodeList, null, false);
		Object.assign(commCodes, selectOptionsList);
	});
};

onMounted(async () => {
	//검색분류구분코드,내외자구분코드
	await getCommCodeList('CHATBOT_SERVICE_DSCE');

	const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
	if (!SpeechRecognition) {
		alert('이 브라우저는 음성 인식을 지원하지 않습니다.');
		return;
	}

	recognition = new SpeechRecognition();
	recognition.lang = 'ko-KR';
	recognition.continuous = false;
	recognition.interimResults = false;

	recognition.onresult = event => {
		const transcript = event.results[0][0].transcript;
		reqChat(transcript);
	};

	recognition.onerror = event => {
		console.error('Recognition error:', event.error);
		isVoiceActive.value = false;
	};

	recognition.onend = () => {
		isVoiceActive.value = false;
	};
});
</script>
