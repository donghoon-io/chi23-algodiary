<script>
	import { getPosts, getFromKeywords, predictNextSentence } from '$lib/clova'
	import { experimentID, nickname, temperature } from '$lib/store'
	import Switch from '$lib/misc/toggle.svelte';
    import RangeSlider from "svelte-range-slider-pips";
	import LoadingIndicator from '$lib/misc/LoadingIndicator.svelte'
	import { onMount } from "svelte";
	import { getNotificationsContext } from 'svelte-notifications';
	import { mapRange } from 'fractils'
	import { openModal } from 'svelte-modals'
	import { Modals, closeModal } from 'svelte-modals'
	import Modal from '$lib/Modal.svelte'
	import CaptureModal from '$lib/CaptureModal.svelte'
	import Tooltip from '$lib/misc/tooltip/Tooltip.svelte';
	import { tooltip } from '$lib/misc/tooltip/tooltip';
	import { tooltip as tooltipv1 } from '$lib/misc/tooltip/tooltip.v1';
	import { Tabs, Tab, TabList, TabPanel } from 'svelte-tabs';
	import { doc, onSnapshot, updateDoc, query, orderBy, setDoc, collection, addDoc, getFirestore, Timestamp } from "firebase/firestore";



	const { addNotification } = getNotificationsContext();

	let db = getFirestore();
	
	let isSharing = true;
	var today = new Date();

	var recommendedKeywordPhrase = [];
	var recommendedPhrase = [];

	const delay = 1000;

	var prevData = [];
	var highlightedData;

	let diaryTitle = "";
	let diaryContent = "";

	let tags = "";

	$: range = [$temperature]
	$: temperature.set(range[0])

	$: tempMapped = mapRange($temperature, 0, 100, 0.0, 1.0)

	function clickPrevDiary(idx) {
		highlightedData = prevData[idx];
		addDoc(collection(db, "data", String($experimentID), "prev_diary_click_event"), {"content": prevData[idx].content, "feedback": prevData[idx].feedback, "is_shared": prevData[idx].is_shared, "accessed_timestamp": Timestamp.fromDate(new Date()), "created_timestamp": prevData[idx].timestamp, "name": prevData[idx].name, "title": prevData[idx].title});
	}

	function diaryComplete() {
		if (diaryTitle == "") {
			addNotification({
				text: '제목을 입력해주세요',
				type: 'danger',
				position: 'top-center',
				removeAfter: 3000,
			})
		}
		if (diaryContent == "") {
			addNotification({
				text: '내용을 입력해주세요',
				type: 'danger',
				position: 'top-center',
				removeAfter: 3000,
			})
		}
		if (diaryTitle != "" && diaryContent != "") {
			// save and populate here
			getPosts(diaryContent).then(result => {
				addDoc(collection(db, "data", String($experimentID), "diary"), {"content": diaryContent, "feedback": result, "is_shared": isSharing, "timestamp": Timestamp.fromDate(new Date()), "name": $nickname, "title": diaryTitle}).then(doc => openModal(Modal, { message: result, id: doc.id }));
			});
		}
	}

	function addText(text, id, isKeyword) {
		if (isKeyword) {
			updateDoc(doc(db, "data", String($experimentID), "keyword_request", id), {
				"selection": text
			})
			recommendedKeywordPhrase = [];
			tags = "";
		} else {
			updateDoc(doc(db, "data", String($experimentID), "next_sentence_request", id), {
				"selection": text
			})
			recommendedPhrase = [];
		}
		if (diaryContent != "" && diaryContent != null) {
			if (diaryContent.substr(diaryContent.length - 1) == " ") {
				diaryContent += text;
			} else {
				diaryContent += " " + text;
			}
		} else {
			diaryContent = text;
		}
	}

	var currentKeywordID = "";
	var currentNextID = "";

	function addKeywordPhrase(text) {
		if (text != "Error" && text != "E") {
			recommendedKeywordPhrase.push(text);
			recommendedKeywordPhrase = recommendedKeywordPhrase;
		}
	}
	function addNextPhrase(text) {
		if (text != "Error" && text != "E") {
			recommendedPhrase.push(text);
			recommendedPhrase = recommendedPhrase;
		}
	}

	let loading = false;
	
	function keywordComplete(text) {
		if (tags == "" || tags == null) {
			addNotification({
				text: '키워드를 입력해주세요',
				type: 'danger',
				position: 'top-center',
				removeAfter: 3000,
			})
		} else {
			// save and populate here
			recommendedKeywordPhrase = [];
			loading = true;
			getFromKeywords(tags, tempMapped).then(result => {
				addKeywordPhrase(result);
				addDoc(collection(db, "data", String($experimentID), "keyword_request"), {"original_content": diaryContent, "timestamp": Timestamp.fromDate(new Date()), "name": $nickname, "title": diaryTitle, "tags": tags, "temperature": tempMapped, "recommendation": recommendedKeywordPhrase}).then(docu => {
					currentKeywordID = docu.id;
					setTimeout(() => {
						getFromKeywords(tags, tempMapped).then(result1 => {
							addKeywordPhrase(result1);
							setTimeout(() => {
								getFromKeywords(tags, tempMapped).then(result2 => {
									addKeywordPhrase(result2);
									setTimeout(() => {
										getFromKeywords(tags, tempMapped).then(result3 => {
											addKeywordPhrase(result3);
											setTimeout(() => {
												getFromKeywords(tags, tempMapped).then(result4 => {
													addKeywordPhrase(result4);
													loading = false;
													updateDoc(doc(db, "data", String($experimentID), "keyword_request", docu.id), {
														"recommendation": recommendedKeywordPhrase
													})
												});
											}, delay);
										});
									}, delay);
								});
							}, delay);
						});
					}, delay);
				})
			});
		}
	}

	function nextComplete() {
		if (diaryContent == "") {
			addNotification({
				text: '일기란에 최소 한 문장을 입력해주세요',
				type: 'danger',
				position: 'top-center',
				removeAfter: 3000,
			})
		} else {
			// save and populate here
			recommendedPhrase = [];
			loading = true;
			predictNextSentence(diaryContent, tempMapped).then(result => {
				addNextPhrase(result);
				addDoc(collection(db, "data", String($experimentID), "next_sentence_request"), {"original_content": diaryContent, "timestamp": Timestamp.fromDate(new Date()), "name": $nickname, "temperature": tempMapped, "title": diaryTitle, "recommendation": recommendedPhrase}).then(docu => {
					currentNextID = docu.id;
					setTimeout(() => {
						predictNextSentence(diaryContent, tempMapped).then(result1 => {
							addNextPhrase(result1);
							setTimeout(() => {
								predictNextSentence(diaryContent, tempMapped).then(result2 => {
									addNextPhrase(result2);
									setTimeout(() => {
										predictNextSentence(diaryContent, tempMapped).then(result3 => {
											addNextPhrase(result3);
											setTimeout(() => {
												predictNextSentence(diaryContent, tempMapped).then(result4 => {
													addNextPhrase(result4);
													loading = false;
													updateDoc(doc(db, "data", String($experimentID), "next_sentence_request", docu.id), {
														"recommendation": recommendedPhrase
													})
												});
											}, delay);
										});
									}, delay);
								});
							}, delay);
						});
					}, delay);
				})
			});
		}
	}
	
	function logout() {
		$experimentID = 0;
		$nickname = "";
	}
	function changeName() {
		let person = prompt("변경할 이름을 입력해주세요:", $nickname);
		if (person == "") {
			alert("최소 1자 이상으로 입력해주세요");
		} else if (person != null) {
			$nickname = person;
		}
	}
	
	const q = query(collection(db, "data", String($experimentID), "diary"), orderBy("timestamp", "desc"))

	onSnapshot(q, (snapshot) => {
		prevData = snapshot.docs.map(doc => doc.data());
	})



	onMount(async () => {
		addDoc(collection(db, "data", String($experimentID), "access_event"), {"timestamp": Timestamp.fromDate(new Date())});
	});
</script>

<svelte:head>
	<title>Home</title>
</svelte:head>


<Modals>
	<div
		slot="backdrop"
		class="backdrop"
		on:click={closeModal}
	/>
</Modals>

<section>
	<div class="flex h-screen divide-x divide-slate-200">
		<div class="w-4/12 h-full divide-y divide-slate-200 bg-zinc-100">
			<div class="h-1/5 text-center flex justify-center items-center bg-white">
				<div>
					<p class="text-sm pb-1">인공지능과 함께 나에대해 알아보기</p>
					<p class="text-3xl font-medium" style="font-family: 'Gentium Book Plus' !important">DiaryMate</p>
					<div class="flex items-center justify-center mt-3">
						<p class="text-sm pr-4">참가자 ID: {$experimentID}</p>
						<button class="hover:bg-gray-100 text-gray-800 py-1 px-2 border border-gray-400 rounded shadow inline-flex items-center justify-center">
							<p on:click={logout} class="text-sm">로그아웃</p>
						</button>
					</div>
				</div>
			</div>
			<div class="{highlightedData != null ? 'hidden px-6 py-4 text-center overflow-scroll bg-white' : 'h-4/5 px-6 py-4 text-center overflow-scroll bg-white'}">
				{#if prevData.length != 0}
				<Tooltip title="제목을 누르면, 기존에 작성했던 일기를 다시 볼 수 있어요">
				<table class="table">
					<caption>표 제목</caption>
					<tr class="text-sm" style="border-bottom: 2px solid #999;"><th>일기 쓴 날짜</th><th>제목</th></tr>
					{#each prevData as data, idx}
					<tr class="text-sm cursor-pointer" on:click={() => clickPrevDiary(idx)}><td>{new Date(data.timestamp.seconds * 1000).getMonth()+1}월 {new Date(data.timestamp.seconds * 1000).getDate()}일</td><td>{data.title}</td></tr>
					{/each}
				</table>  
				</Tooltip>
				{:else}
				<div class="flex h-full items-center justify-center">
					<p class="text-center text-sm text-gray-500 leading-6">아직 작성하신 일기가 없습니다.<br>오른쪽 화면에서 오늘의 일기를 작성해보세요!</p>
				</div>
				{/if}  
			</div>
			{#if highlightedData != null}
			<div class="p-4 h-2/5 overflow-scroll">
				<div class="flex">
					<div>
						<p class="text-xl pb-3">{highlightedData.title}</p>
					</div>
					<div class="ml-auto">
						<button type="button" class="hover:bg-gray-100 focus:ring-4 focus:outline-none focus:ring-blue-300 py-2 text-center inline-flex items-center" on:click={() => {highlightedData = null}}>
							<img src="./x.png" class="h-5">
						  </button>
					</div>
				</div>
				<p class="text-sm pt-2 pb-2">{highlightedData.content}</p>
				<div class="pt-8 flex items-center">	
					<img src="./book.png" class="h-12">
					<blockquote class="mx-8">
					<p class="text-sm">{highlightedData.feedback}</p>
					</blockquote>     
				</div>
			</div>
			{/if}
		</div>
		<div class="w-5/12 divide-y divide-slate-200">
				<div class="h-1/6 text-center flex justify-center items-center">
					<div class="w-full px-3">
						<textarea bind:value={diaryTitle} class="
							form-control
							w-full
							font-normal
							text-center
							text-gray-700
							bg-clip-padding
							px-3
							py-2
							transition
							ease-in-out
							text-xl
						"
						id="titleArea"
						rows="1"
						placeholder="제목을 입력해주세요"
						></textarea>
						<p class="text-sm">by {$nickname} <button on:click={changeName}><img class="w-4 h-4" style="margin-bottom: -.2rem !important" src="./pencil.png"></button> - {String(today.getMonth() + 1)}월 {String(today.getDate())}일</p>
					</div>
				</div>
				<div class="h-3/5 p-4">
					<textarea bind:value={diaryContent}
						class="
							form-control
							w-full
							h-full
							font-normal
							text-gray-700
							bg-clip-padding
							bg-transparent
							transition
							ease-in-out
							m-0
							p-1
							text-sm
						"
						id="diaryTextArea"
						placeholder="여기에 일기를 적어주세요"
						></textarea>
				</div>
				<div class="h-1/5 px-4 pt-4 text-center flex justify-center items-center">
					<div>
						<div class="flex justify-center items-center space-x-3 pb-3">
							<p class="text-sm">이 글을 연구진과 공유합니다</p>
							<Switch bind:checked={isSharing}></Switch>
						</div>
						<div class="flex justify-center items-center ml-8">
							<button class="bg-white mt-1 hover:bg-gray-100 text-gray-800 font-medium py-2 px-3 border border-gray-400 rounded shadow inline-flex items-center justify-center">
								<p class="text-sm" on:click={diaryComplete}>오늘의 일기쓰기 완료</p>
							</button>
							<button class="bg-white mt-1 bg-blue-400 hover:bg-gray-600 text-gray-800 font-medium ml-6 p-2 rounded-full drop-shadow-lg inline-flex items-center justify-center" on:click={() => openModal(CaptureModal, { diaryTitle: diaryTitle, diaryContent: diaryContent, keywordPhrases: recommendedKeywordPhrase, nextPhrases: recommendedPhrase, tags: tags })}>
								<img src="./photo.png" class="h-6 p-0.5">
							</button>
						</div>
					</div>
				</div>
			</div>
			
			<div class="w-4/12 h-full divide-slate-200" style="position: relative !important;">
				<div class="p-6 text-center flex justify-center items-center">	
					<img src="./pencil_cute.png" class="h-10 mr-1">
					<p class="text-md font-medium">이런 문장은 어때?</p>
				</div>
			<Tabs>
				<TabList>
				  <Tab>키워드로 부탁하기</Tab>
				  <Tab>다음문장 부탁하기</Tab>
				</TabList>
			  
				<TabPanel>
					<div class="p-4">
						<div class="flex mt-6">	
							<p class="text-sm text-slate-600">어떤 문장을 제안받고 싶은가요?</p>
						</div>
						<div class="text-center">
						<div class="px-5 pb-5">
							<RangeSlider bind:values={range} pips first='label' last='label' formatter={ v => "" } />
							{#if $temperature < 20}
							<div class="flex space-x-1 pt-4 items-center justify-center">
								<svg class="w-5 h-5 pt-0.5 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
								<p class="text-xs text-red-500 font-bold">슬라이더를 낮게 설정하면 너무 일반적인 문장이 나올 수 있어요!</p>
							</div>
							{:else if $temperature > 80}
							<div class="flex space-x-1 pt-4 items-center justify-center">
								<svg class="w-5 h-5 pt-0.5 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
								<p class="text-xs text-red-500 font-bold">슬라이더를 높게 설정하면 너무 독특한 문장이 나올 수 있어요!</p>
							</div>
							{/if}
						</div>
						
						<div class="flex mt-6 mb-5">	
							<p class="text-sm text-slate-600">인공지능이 참고할만한 키워드를 알려주세요</p>
						</div>
						<textarea class="
							form-control
							w-full
							font-normal
							text-gray-700
							bg-clip-padding
							bg-gray-100
							p-3
							transition
							ease-in-out
							m-0
							text-xs
						"
						id="suggestionArea"
						rows="1"
						placeholder="키워드1, 키워드2, 키워드3, ..."
						bind:value={tags}
						></textarea>
						<button class="bg-white mt-6 hover:bg-gray-100 text-gray-800 font-medium py-1.5 border border-gray-400 rounded shadow inline-flex items-center justify-center px-3" on:click={() => keywordComplete(tags)}>
							<img src="./next_line.png" class="w-6 p-1 mr-2"><p class="text-sm">문장을 만들어줘</p>
						</button>
						{#if loading}
						<LoadingIndicator/>
						{/if}
					</div>
					<div class="text-left mt-6 overflow-scroll">
						{#if recommendedKeywordPhrase.length != 0}
						{#each recommendedKeywordPhrase as phrase}
						<button class="tag text-left" on:click={() => addText(phrase, currentKeywordID, true)}>{phrase}</button>
						{/each}
						{:else}
						<p class="text-center text-sm text-gray-500 leading-6 mt-6">“문장을 만들어줘” 버튼을 누르면<br>인공지능이 키워드를 참고하여<br>일기에 들어갈만한 문장을 제안해줘요!</p>
						{/if}
					</div>
				</div>
				</TabPanel>
			  
				<TabPanel>
					
				<div class="p-4">
					<div class="flex mt-6">	
						<p class="text-sm text-slate-600">어떤 문장을 제안받고 싶은가요?</p>
					</div>
					<div class="text-center">
					<div class="px-5 pb-5">
						<RangeSlider bind:values={range} pips first='label' last='label' formatter={ v => "" } />
							{#if $temperature < 20}
							<div class="flex space-x-1 pt-4 items-center justify-center">
								<svg class="w-5 h-5 pt-0.5 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
								<p class="text-xs text-red-500 font-bold">슬라이더를 낮게 설정하면 너무 일반적인 문장이 나올 수 있어요!</p>
							</div>
							{:else if $temperature > 80}
							<div class="flex space-x-1 pt-4 items-center justify-center">
								<svg class="w-5 h-5 pt-0.5 text-red-500" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
								<p class="text-xs text-red-500 font-bold">슬라이더를 높게 설정하면 너무 독특한 문장이 나올 수 있어요!</p>
							</div>
							{/if}
					</div>
					<div class="p-4 text-center">
						<button class="hover:bg-gray-200 text-gray-800 py-1 px-2 border border-gray-400 rounded shadow inline-flex items-center justify-center" on:click={nextComplete}>
							<img src="./next_line.png" class="w-6 p-1 mr-2"><p class="text-sm">다음문장을 만들어줘</p>
						</button>
						{#if loading}
						<LoadingIndicator/>
						{/if}
					</div>
					<div class="text-left mt-6 pt-6">
						{#if recommendedPhrase.length != 0}
						{#each recommendedPhrase as phrase}
						<button class="tag text-left" on:click={() => addText(phrase, currentNextID, false)}>{phrase}</button>
						{/each}
						{:else}
						<p class="text-center text-sm text-gray-500 leading-6">“다음문장을 만들어줘” 버튼을 누르면<br>인공지능이 키워드를 참고하여<br>일기에 들어갈만한 문장을 제안해줘요!</p>
						{/if}
					</div>
				</div>
				</TabPanel>
			  </Tabs>
			</div>
	</div>
</section>

<style>
	.tag {
		border-radius: 15px;
		font-size: .9rem;
		font-weight: 300;
    	line-height: 1.25rem;
		padding-top: 0.4rem;
		padding-bottom: 0.4rem;
		padding-left: 0.4rem;
		padding-right: 0.4rem;
		display: inline-block;	
		margin-bottom: 0.25rem !important;
	}
	.tag:hover {
		background-color: #ddd;
		font-weight: 700;
	}
	.tag:last-of-type {
		margin-bottom: 0 !important;
	}
	
	:global(.pip.first) {
		font-size: .7rem !important;
	}
	:global(.pip.last) {
		font-size: .7rem !important;
	}
	:global(.pip.first > .pipVal:after) {
  		content: '전형적인 문장' !important;
		white-space: pre !important;
	}
	:global(.pip.last > .pipVal:after) {
  		content: '다양한 문장' !important;
		white-space: pre !important;
	}

    .table {
      border-collapse: collapse;
      border-top: 0 !important;
	  border-bottom: 0 !important;
	  width: 100% !important
    }  
    .table th {
		border-top: 0 !important;
      color: #333;
      text-align: center;
    }
    .table th, .table td {
      padding: 10px;
      border: 1px solid #ddd;
	  border-bottom: 0 !important
    }
    .table th:first-child, .table td:first-child {
      border-left: 0;
	  width: 25% !important
    }
    .table th:last-child, .table td:last-child {
      border-right: 0;
    }
    .table tr td:first-child{
      text-align: center;
    }
    .table caption{caption-side: bottom; display: none;}

	blockquote {
    position: relative;
		/* background: #ddd; */
	}
	blockquote {
		position: relative;
		/* background: #ddd; */
	}
	blockquote:before {
		position: absolute;
		content: open-quote;
		font-size: 2em;
		margin-left: -0.6em;
		margin-top: -0.4em;
		
	}
	blockquote:after {
		position: absolute;
		content: close-quote;
		font-size: 2em;
		bottom: 0;
		right: 0;
		margin-right: -0.6em;
		margin-bottom: -0.8em;
	}
	blockquote p {
		display: inline;
		font-style: italic;
	}
	.backdrop {
		position: fixed;
		top: 0;
		z-index: 10 !important;
		bottom: 0;
		right: 0;
		left: 0;
		background: rgba(0,0,0,0.50)
	}
	:global(.svelte-tabs__selected) {
		color: #333 !important;
		font-weight: 500 !important;
		outline: none !important;
		border-bottom: 2px solid #CCC !important;
	}
	:global(.svelte-tabs__selected:focus) {
		outline: none !important;
	}
	:global(.svelte-tabs__tab) {
		color: #666 !important;
	}
	:global(.svelte-tabs__tab:focus) {
		outline: none !important;
	}
	:global(.svelte-tabs__tab-list) {
		text-align: center !important;
	}
  </style>