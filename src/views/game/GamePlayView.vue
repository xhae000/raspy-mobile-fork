<template>
  <div v-if="game" class="max-w-2xl mx-auto px-4 py-6 space-y-6">
    <!-- 상단 타이머 & 돌아가기 -->
    <div class="fixed top-0 left-0 w-full bg-gray-900 text-white py-2 px-4 text-sm z-50 flex justify-between items-center">
      <span>
        ⏱ 경과 시간: {{ elapsedTimeStr }} /
        제한 시간: 
        <span v-if="game.limitSeconds === -1">없음</span>
        <span v-else>{{ limitTimeStr }}</span>
      </span>
      <button @click="goBack" class="text-white text-xs underline">← 돌아가기</button>
    </div>

    <div class="pt-10 space-y-8">
      <!-- 유저 점수 및 세트 스코어 -->
      <div class="flex justify-between items-center">
        <div class="flex flex-col items-center gap-1">
          <img v-if="game.user1.profileUrl" :src="game.user1.profileUrl" class="w-12 h-12 rounded-full object-cover" />
          <p class="text-sm">{{ game.user1.nickname }}</p>
          <p class="text-2xl font-bold">{{ currentScore1 }}</p>
          <div class="text-xs text-gray-500">세트 승리: {{ user1SetsWon }}/{{ game.setsToWin }}</div>
          <div class="flex gap-1 mt-1">
            <button @click="socket_sendScore(1, 1)" :disabled="isSetOver || isGameOver" class="px-2 rounded bg-gray-100">➕</button>
            <button @click="socket_sendScore(1, -1)" :disabled="isSetOver || isGameOver || currentScore1<=0" class="px-2 rounded bg-gray-100">➖</button>
          </div>
        </div>
        <p class="text-3xl font-bold text-gray-700">VS</p>
        <div class="flex flex-col items-center gap-1">
          <img v-if="game.user2.profileUrl" :src="game.user2.profileUrl" class="w-12 h-12 rounded-full object-cover" />
          <p class="text-sm">{{ game.user2.nickname }}</p>
          <p class="text-2xl font-bold">{{ currentScore2 }}</p>
          <div class="text-xs text-gray-500">세트 승리: {{ user2SetsWon }}/{{ game.setsToWin }}</div>
          <div class="flex gap-1 mt-1">
            <button @click="socket_sendScore(2, 1)" :disabled="isSetOver || isGameOver" class="px-2 rounded bg-gray-100">➕</button>
            <button @click="socket_sendScore(2, -1)" :disabled="isSetOver || isGameOver || currentScore2<=0" class="px-2 rounded bg-gray-100">➖</button>
          </div>
        </div>
      </div>

      <!-- 게임 정보 -->
      <div class="text-center text-sm text-gray-600">
        <p>현재 세트: {{ currentSet }}</p>
        <p class="mt-2">{{ game.majorCategory }} - {{ game.minorCategory }} / {{ game.ruleTitle }}</p>
        <p class="mt-1">{{ game.place }} · {{ formatDate(game.matchDate) }}</p>
        <p class="mt-1">한 세트 승리 조건: {{ game.pointsToWin }}점</p>
      </div>


      <!-- 세트 종료 시: 다음 세트 버튼 -->
      <div v-if="isSetOver && !isGameOver" class="flex justify-center mt-4">
        <button @click="socket_nextSet" class="bg-blue-500 text-white px-4 py-2 rounded-full">다음 세트</button>
      </div>

      <!-- 게임 종료 시: 통계/결과 페이지 이동 -->
      <div v-if="isGameOver" class="flex flex-col items-center mt-4 space-y-2">
        <p class="text-lg font-bold text-green-700">게임이 종료되었습니다!</p>
        <button @click="goToResult" class="bg-blue-500 text-white px-4 py-2 rounded-full">통계/결과 보기</button>
      </div>
    </div>

    <!-- 세트 종료/게임 종료 안내 모달 (optional) -->
    <div v-if="showFinishModal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white p-6 rounded-xl text-center space-y-4">
        <h2 class="text-lg font-bold">세트가 종료되었습니다! 승자 : {{ winner }}</h2>
        <button @click="closeFinishModal" class="bg-blue-500 text-white px-4 py-2 rounded-full">확인</button>
      </div>
    </div>
  </div>

  <div class="p-5">
    <div @click="resetGame" class="bg-red-400 text-white text-center ">
      게임 리셋하기
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import api from '../../api/api'
import { socket } from '../../websocket'

const route = useRoute()
const router = useRouter()
const gameId = route.params.gameId

const game = ref(null)
const chatRoomId = ref(null)
const currentSet = ref(0)
const currentScore1 = ref(0)
const currentScore2 = ref(0)
const user1SetsWon = ref(0)
const user2SetsWon = ref(0)
const isSetOver = ref(false)
const isGameOver = ref(false)
const showFinishModal = ref(false)
const winner = ref('-')
const elapsedSeconds = ref(0)
const elapsedTimeStr = ref('0분 0초')
const limitTimeStr = ref('')
const timerRef = ref(null)

const user1 = ref(null)
const user2 = ref(null)

const delta = 1; // fixed score delta value

const formatDate = (dateStr) => {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  return d.toLocaleString('ko-KR', { dateStyle: 'short', timeStyle: 'short' })
}

function startSet() {
  currentScore1.value = 0
  currentScore2.value = 0
  isSetOver.value = false
  showFinishModal.value = false
  winner.value = '-'
  updateElapsed()
  startTimer()
}

function addScore(userIdx) {
  if (isSetOver.value || isGameOver.value) return

  if (userIdx == user1.value.id) {
    currentScore1.value = Math.max(0, currentScore1.value + delta)
  } else {
    currentScore2.value = Math.max(0, currentScore2.value + delta)
  }

  checkSetOver()
}

const resetGame = () => {
  if(!confirm("정말로 게임을 재시작 하겠습니까?")) return;
  
  socket_resetGame()
}

function checkSetOver() {
  if (currentScore1.value >= game.value.pointsToWin) {
    user1SetsWon.value++
    currentSet.value ++
    finishSet(1)
    return
  }  if (currentScore2.value >= game.value.pointsToWin) {
    user2SetsWon.value++
    currentSet.value ++
    finishSet(2)
    return
  }
  if (game.value.limitSeconds !== -1 && elapsedSeconds.value >= game.value.limitSeconds) {
    if (currentScore1.value > currentScore2.value) {
      user1SetsWon.value++
      currentSet.value ++
      finishSet(1)
    } else if (currentScore2.value > currentScore1.value) {
      user2SetsWon.value++
      currentSet.value ++
      finishSet(2)
    } else {
      finishSet(0) // 무승부 (점수가 동일한 경우 무승부일것임)
      currentSet.value ++
    } 

    elapsedSeconds.value = 0;

    return
  }
}

function finishSet(who) {
  isSetOver.value = true
  clearInterval(timerRef.value)
  if (who === 1) {
    winner.value = game.value.user1.nickname
  }
  else if (who === 2) { 
    winner.value = game.value.user2.nickname
  }

  showFinishModal.value = true
  if (
    user1SetsWon.value >= game.value.setsToWin ||
    user2SetsWon.value >= game.value.setsToWin
  ) {
  /**
   * 조건 세트 수가 채워져, 게임이 종료됨.
   */
    isGameOver.value = true
    isSetOver.value = false

    socket_finishGame()
  }

}

function closeFinishModal() {
  showFinishModal.value = false
}

function startTimer() {
  if(isGameOver.value) return
  clearInterval(timerRef.value)
  updateElapsed()
  timerRef.value = setInterval(updateElapsed, 1000)
}

function updateElapsed() {
  // 경과 시간: setStartedAt 기준으로 현 시각에서 1초씩 증가
  if (!game.value?.setStartedAt) return
  const startedAt = new Date(game.value.setStartedAt)
  const now = new Date()
  elapsedSeconds.value = Math.floor((now - startedAt) / 1000)
  elapsedTimeStr.value = getTimeStr(elapsedSeconds.value)

  // 제한 시간 표시(최초 1회만)
  if (game.value.limitSeconds !== -1) {
    limitTimeStr.value = getTimeStr(game.value.limitSeconds)
  }
  // 세트 종료 자동 감지
  if (!isSetOver.value && !isGameOver.value) {
    checkSetOver()
  }
}

function getTimeStr(seconds) {
  if (seconds <= 0) return '0분 0초'
  const min = Math.floor(seconds / 60)
  const sec = seconds % 60
  return `${min}분 ${sec}초`
}

function goToResult() {
  router.push(`/games/${chatRoomId.value}/result`)
}

function goBack() {
  router.back()
}

function handleVisibilityChange() {
  if (document.visibilityState === 'visible') {
    updateElapsed()
    if (!isSetOver.value && !isGameOver.value) {
      startTimer()
    }
  }
}


onMounted(async () => {
  const res = await api.get(`/api/games/${gameId}/detail`)
  game.value = res.data
  chatRoomId.value = res.data.chatRoomId

  currentScore1.value = res.data.score1
  currentScore2.value = res.data.score2


  currentSet.value = res.data.set1  + res.data.set2 || 0
  user1SetsWon.value = res.data.set1
  user2SetsWon.value = res.data.set2
  user1.value = res.data.user1
  user2.value = res.data.user2

  isSetOver.value = false
  isGameOver.value = false
  console.log(game.value)
  startSet()
  socket.connect(chatRoomId.value, () => {
    socket.subscribe(`${chatRoomId.value}`)
    socket.onMessage((payload) => {
      console.log(payload)
      if (payload.type === 'SCORE') {
        addScore(payload.userId, payload.score)
      } else if (payload.type === 'SET') {
        startSet()
      } else if (payload.type === 'FINISH') {
        isGameOver.value = true
      }
      else if (payload.type === 'RESET') {
        router.replace({ path: route.fullPath, query: route.query })
      }
    })
  })
  document.addEventListener('visibilitychange', handleVisibilityChange)
})

const socket_sendScore = (userId, scoreDelta) => {
  socket.sendGameEvent(chatRoomId.value, {
    type: 'SCORE',
    userId: userId==1 ? user1.value.id : user2.value.id, // 점수 변화하는 사람 
    setIndex: currentSet.value,
    scoreDelta: scoreDelta
  })
}

 const socket_nextSet = () => {
  console.log("세트 증가함")
  socket.sendGameEvent(chatRoomId.value, {
    type: 'SET',
    setIndex: currentSet.value,
        

    /**
     * no-valuable
     */
    userId: 0,
    scoreDelta: 0
  })
}

const socket_finishGame = () => {
  socket.sendGameEvent(chatRoomId.value, {
    type: 'FINISH'
  })
}

const socket_resetGame = () => {
  socket.sendGameEvent(chatRoomId.value, {
    type: 'RESET'
  })
}

onUnmounted(() => {
  clearInterval(timerRef.value)
  document.removeEventListener('visibilitychange', handleVisibilityChange)
})
</script>
