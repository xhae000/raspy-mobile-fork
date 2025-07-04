<!--
TODO: Game 상태에 따라 다른 승인자나타남, 게임 진행됨 등 표시.
-->

<template>
  <div>

    <div v-if="games.length" class="space-y-5">
      <div v-for="game in games" :key="game.id" class="p-5 border bg-white rounded-2xl shadow space-y-5">

        <div class="flex justify-between items-start">
          <div class="space-y-2">
            <p class="text-xs text-gray-400">{{ game.matchLocation || '장소 미정' }} · {{ formatDate(game.matchDate) }}</p>
            <h3 class="text-lg font-bold text-gray-800">{{ game.rule.ruleTitle }}</h3>
            <p class="text-sm text-gray-500">{{ game.rule.majorCategory }} > {{ game.rule.minorCategory }}</p>
            <p class="text-sm text-gray-700 leading-snug">{{ game.rule.ruleDescription }}</p>

          </div>

          <span class="text-xs font-semibold px-1 w-[40%] text-center py-2 rounded-full"
            :class="{
              'bg-blue-100 text-blue-600': game.status === 'APPROVED',
              'bg-gray-100 text-gray-500': game.status === 'REQUESTED',
              'bg-red-100 text-red-500': game.status === 'REJECTED'
            }">
            {{ translateStatus(game.status) }}
          </span>
        </div>
        <button @click="game.showRuleDetail = !game.showRuleDetail" class="mt-2 w-full py-3 text-xs bg-gray-100 rounded-[5px] text-gray-600 flex items-center justify-center gap-2">
              <i :class="game.showRuleDetail ? 'fas fa-chevron-up' : 'fas fa-chevron-down'"></i>
              {{ game.showRuleDetail ? '간략히 보기' : '상세 보기' }}
        </button>
        <transition name="fade">
          <div v-if="game.showRuleDetail" class="mt-3 space-y-3 p-4 rounded-xl bg-gray-50 border border-gray-200">
            <h4 class="text-base font-bold text-gray-600">세부 규칙</h4>
            <div class="text-sm text-gray-700 space-y-2">
              <div class="flex justify-between">
                <span class="text-gray-500">세트 승리 기준</span>
                <span>{{ game.rule.winBy === 'SETS_HALF_WIN' ? '점수 달성' : '시간 도달' }}</span>
              </div>
              <div class="flex justify-between">
                <span class="text-gray-500">세트 승리 점수</span>
                <span>{{ game.rule.pointsToWin === -1 ? '제한 없음' : game.rule.pointsToWin + '점' }}</span>
              </div>
              <div class="flex justify-between">
                <span class="text-gray-500">총 세트 수</span>
                <span>{{ game.rule.setsToWin }}세트</span>
              </div>
              <div class="flex justify-between">
                <span class="text-gray-500">세트 제한 시간</span>
                <span>{{ game.rule.duration == -1 ? '제한 없음' :  game.rule.duration >= 60 ? Math.floor(game.rule.duration / 60) + '분 ' + (game.rule.duration % 60) + '초' : game.rule.duration + '초' }}</span>
              </div>
            </div>
          </div>
        </transition>

        <div class="flex justify-between items-center pt-3 border-t">
          <div>
            <p class="text-sm font-semibold text-gray-800 flex items-center gap-2">
              @{{ game.ownerNickname }}
              <span class="text-xs flex items-center gap-1">
                <i :class="[
                  (game.ownerStatistics.manner || 4.5) >= 4 ? 'fas fa-face-smile text-green-500' : 
                  (game.ownerStatistics.manner || 4.5) >= 2 ? 'fas fa-face-meh text-orange-500' : 
                  (game.ownerStatistics.manner || 4.5) > 0 ? 'fas fa-face-frown text-red-500' : 
                  'fas fa-user-slash text-gray-400'
                ]"></i>
                {{ (game.ownerStatistics.manner || 4.5).toFixed(1) }}
              </span>
            </p>
            <p class="text-xs text-gray-500">
              {{ game.ownerStatistics.wins }}승 {{ game.ownerStatistics.draws }}무 {{ game.ownerStatistics.losses }}패 · 승률 {{ getWinRate(game.ownerStatistics) }}%
            </p>
          </div>
          <div>
            <champion-badge v-if="game.championId == game.ownerId"></champion-badge>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-3 pt-1">
          <button v-if="game.status === 'REQUESTED'" @click="cancelRequest(game.id)"
            class="flex items-center justify-center gap-2 text-red-400  text-sm text-red-400 border-[1px] border-red-400 py-3 rounded-[8px]">
            <i class="fas fa-xmark "></i> 신청 취소
          </button>
          <button v-else-if="game.status === 'APPROVED'"
            class="flex items-center justify-center gap-2 bg-[#afafaf] text-sm text-white border-[1px]  py-3 rounded-[8px]">
            승인 완료
          </button>
          <button v-else-if="game.status === 'REJECTED'"
            class="flex items-center justify-center gap-2 bg-[#afafaf] text-sm text-white border-[1px]  py-3 rounded-[8px]">
            승인 거부
          </button>

          <button class="flex items-center justify-center gap-2 text-sm text-white bg-blue-400 py-3 rounded-[8px]">
            <i class="fas fa-paper-plane"></i> DM
          </button>
        </div>

      </div>
    </div>

    <div v-else class="text-center text-gray-400 text-sm py-10">신청한 게임이 없습니다.</div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import client from '../../api/api'
import ChampionBadge from '../ChampionBadge.vue'

const games = ref([])

onMounted(async () => {
  const res = await client.get('/api/games/my-requests')
  games.value = res.data.map(g => ({ ...g, showRuleDetail: false }))
})

function formatDate(dateStr) {
  if (!dateStr) return '일정 미정'
  const date = new Date(dateStr)
  return date.toLocaleString('ko-KR', { dateStyle: 'short', timeStyle: 'short' })
}

function translateStatus(status) {
  switch (status) {
    case 'APPROVED': return '승인됨'
    case 'REQUESTED': return '대기 중'
    case 'REJECTED': return '거절됨'
    default: return '알 수 없음'
  }
}

function getWinRate(stat) {
  const total = stat.wins + stat.draws + stat.losses
  if (total === 0) return 0
  return Math.round((stat.wins / total) * 100)
}

async function cancelRequest(gameId) {
  await client.post('/api/games/cancel-request', { gameId })
  games.value = games.value.filter(g => g.id !== gameId)
}
</script>

<style scoped>
@import url("https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css");

.fade-enter-active, .fade-leave-active {
  transition: all 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
  transform: translateY(-4px);
}
</style>
