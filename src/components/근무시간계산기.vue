<script setup>
import { ref, computed, watch, watchEffect } from 'vue'
import { VueDatePicker } from '@vuepic/vue-datepicker'
import '@vuepic/vue-datepicker/dist/main.css'
import { 월소정근로일수조회, 남은근무일수조회, 남은금요일수조회, 급여일조회 } from '../utils/근무시간'
import { 월별공휴일조회, 공휴일데이터존재여부 } from '../utils/공휴일'
import { 시분파싱, 시분변환 } from '../utils/시간포맷'
import { 테마사용 } from '../composables/테마'
import 달성현황 from './달성현황.vue'
import 공휴일목록 from './공휴일목록.vue'
import 다음달미리보기 from './다음달미리보기.vue'

const { 테마, 토글: 테마토글 } = 테마사용()
const 다크모드 = computed(() => 테마.value === 'dark')

// TODO: 테스트용 금요일 강제 고정 — 배포 전 `const 오늘 = new Date()`로 되돌릴 것
const 오늘 = new Date()
const 현재연도 = 오늘.getFullYear()
const 현재월 = 오늘.getMonth() + 1

const 요일이름 = ['일', '월', '화', '수', '목', '금', '토']
const 오늘요일 = 오늘.getDay()
const 오늘금요일여부 = 오늘요일 === 5
const 오늘표시 = `${현재연도}년 ${현재월}월 ${오늘.getDate()}일 ${요일이름[오늘요일]}요일`
const 재택안내 =
  오늘요일 === 5 ? '오늘은 재택근무' :
  오늘요일 === 4 ? '내일은 재택근무' : ''

// 이번 주(일~토)에 월급날이 포함되는지 확인
const 급여일 = 급여일조회(현재연도, 현재월)
const 주시작 = new Date(현재연도, 오늘.getMonth(), 오늘.getDate() - 오늘요일)
const 주끝 = new Date(주시작)
주끝.setDate(주끝.getDate() + 6)
const 급여주여부 = 급여일 >= 주시작 && 급여일 <= 주끝

const 선택연도 = ref(현재연도)
const 선택월 = ref(현재월)
const 고정연장시간 = ref('10:00')
const 입력근무시간 = ref('')
const 오늘예상시간 = ref('0:00')
const 오늘입력모드 = ref('출퇴근')
const 오늘재택근무 = ref(false)
const 출근시각 = ref('09:00')
const 퇴근시각 = ref('18:00')
const 휴게자동 = ref(true)
const 휴게수동분 = ref(60)

function 시각분리(시각) {
  const 매칭 = String(시각 ?? '').match(/^(\d{1,2}):(\d{2})$/)
  if (!매칭) return { 시: 9, 분: 0 }
  return { 시: Number(매칭[1]), 분: Number(매칭[2]) }
}
function 시각조립(시, 분) {
  return `${String(시).padStart(2, '0')}:${String(분).padStart(2, '0')}`
}
function 시각객체(시각) {
  if (!시각) return null
  const { 시, 분 } = 시각분리(시각)
  return { hours: 시, minutes: 분, seconds: 0 }
}
const 출근객체 = computed({
  get: () => 시각객체(출근시각.value),
  set: (값) => { 출근시각.value = 값 ? 시각조립(값.hours, 값.minutes) : '' },
})
const 퇴근객체 = computed({
  get: () => 시각객체(퇴근시각.value),
  set: (값) => { 퇴근시각.value = 값 ? 시각조립(값.hours, 값.minutes) : '' },
})
const 입사한달여부 = ref(false)
const 입사일 = ref(오늘.getDate())
const 재택근무여부 = ref(false)
const 재택근무일수 = ref(0)
const 연차여부 = ref(false)
const 연차일수 = ref(0)
const 반차수 = ref(0)
const 반반차수 = ref(0)

const 하루근무분 = 8 * 60

function 입력근무정규화() {
  const 결과 = 시분파싱(입력근무시간.value)
  if (!결과.유효) return
  입력근무시간.value = 결과.비어있음 ? '' : 시분변환(결과.분)
}

function 고정연장정규화() {
  const 결과 = 시분파싱(고정연장시간.value)
  if (!결과.유효) return
  고정연장시간.value = 시분변환(Math.max(0, 결과.분))
}

function 오늘예상정규화() {
  const 결과 = 시분파싱(오늘예상시간.value)
  if (!결과.유효) return
  오늘예상시간.value = 결과.비어있음 ? '0:00' : 시분변환(Math.max(0, 결과.분))
}

function 지금시각() {
  const 지금 = new Date()
  const 시 = String(지금.getHours()).padStart(2, '0')
  const 분 = String(지금.getMinutes()).padStart(2, '0')
  return `${시}:${분}`
}
function 출근지금() { 출근시각.value = 지금시각() }
function 퇴근지금() { 퇴근시각.value = 지금시각() }

const 입력결과 = computed(() => 시분파싱(입력근무시간.value))
const 고정연장결과 = computed(() => 시분파싱(고정연장시간.value))
const 오늘예상결과 = computed(() => 시분파싱(오늘예상시간.value))
const 입력분 = computed(() => Math.max(0, 입력결과.value.분))
const 고정연장분 = computed(() => Math.max(0, 고정연장결과.value.분))
const 입력유효 = computed(() => 입력결과.value.유효)
const 고정연장유효 = computed(() => 고정연장결과.value.유효)
const 오늘예상유효 = computed(() => 오늘예상결과.value.유효)

// 출퇴근 자동 계산
function 시각문자열을분으로(문자열) {
  const 매칭 = String(문자열 ?? '').match(/^(\d{1,2}):(\d{2})$/)
  if (!매칭) return null
  const 시 = Number(매칭[1])
  const 분 = Number(매칭[2])
  if (시 < 0 || 시 > 23 || 분 < 0 || 분 > 59) return null
  return 시 * 60 + 분
}
const 출근분 = computed(() => 시각문자열을분으로(출근시각.value))
const 퇴근분 = computed(() => 시각문자열을분으로(퇴근시각.value))
const 출퇴근유효 = computed(
  () => 출근분.value !== null && 퇴근분.value !== null,
)
const 자정넘김여부 = computed(() => {
  if (!출퇴근유효.value) return false
  return 퇴근분.value < 출근분.value
})
const 총체류분 = computed(() => {
  if (!출퇴근유효.value) return 0
  let 차 = 퇴근분.value - 출근분.value
  if (차 < 0) 차 += 24 * 60
  return Math.max(0, 차)
})
const 휴게자동분 = computed(() => {
  const 체류 = 총체류분.value
  if (체류 > 5 * 60) return 60
  return 0
})
const 휴게분 = computed(() => {
  if (!출퇴근유효.value) return 0
  return 휴게자동.value ? 휴게자동분.value : Math.max(0, Number(휴게수동분.value) || 0)
})
const 출퇴근근무분 = computed(() => {
  if (!출퇴근유효.value) return 0
  return Math.max(0, 총체류분.value - 휴게분.value)
})

const 오늘예상분 = computed(() => {
  // 오늘이 재택근무일이면 누적 근무시간에 이미 반영되므로 오늘 시간은 0으로 계산(중복 방지)
  if (오늘재택근무.value) return 0
  if (오늘입력모드.value === '출퇴근') return 출퇴근근무분.value
  return Math.max(0, 오늘예상결과.value.분)
})
// 재택 토글: 켜면 직접 입력값을 백업하고 0으로 리셋, 끄면 이전 값 복원
const 오늘예상시간_백업 = ref('')
watch(오늘재택근무, (켜짐) => {
  if (켜짐) {
    오늘예상시간_백업.value = 오늘예상시간.value
    오늘예상시간.value = '0:00'
  } else {
    오늘예상시간.value = 오늘예상시간_백업.value || '0:00'
  }
})
const 반영분 = computed(() => 입력분.value + 오늘예상분.value)

const 연도목록 = computed(() => {
  const 목록 = []
  for (let 연도 = 현재연도 - 1; 연도 <= 현재연도 + 2; 연도++) {
    목록.push(연도)
  }
  return 목록
})

const 월목록 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]

const 월말일 = computed(() =>
  new Date(선택연도.value, 선택월.value, 0).getDate(),
)
const 일목록 = computed(() => {
  const 목록 = []
  for (let 일 = 1; 일 <= 월말일.value; 일++) 목록.push(일)
  return 목록
})
const 유효입사일 = computed(() => {
  if (!입사한달여부.value) return 1
  return Math.max(1, Math.min(월말일.value, Number(입사일.value) || 1))
})
const 소정근로일 = computed(() =>
  월소정근로일수조회(선택연도.value, 선택월.value, 유효입사일.value),
)
const 의무근로분 = computed(() => 소정근로일.value * 하루근무분)
const 최대근로분 = computed(() => 의무근로분.value + 고정연장분.value)
const 남은근무일 = computed(() =>
  남은근무일수조회(선택연도.value, 선택월.value, 유효입사일.value),
)
const 경과근무일 = computed(() => 소정근로일.value - 남은근무일.value)

// 재택근무: 남은 금요일 중 신청 일수만큼은 8시간이 자동 인정되므로
// 일평균 목표 계산에서 제외하고 '출근일'만 분모로 사용한다.
const 남은금요일 = computed(() =>
  남은금요일수조회(선택연도.value, 선택월.value, 유효입사일.value),
)
const 재택일수 = computed(() => {
  if (!재택근무여부.value) return 0
  return Math.max(0, Math.min(남은금요일.value, Number(재택근무일수.value) || 0))
})
// 연차: 연차(8h)·반차(4h)·반반차(2h)는 '현재까지 근무시간'에 이미 포함된 시간이다.
// 따라서 반영분에 다시 더하지 않고(중복 방지), '현재까지 근무시간'을 한도로 둔다.
// 대신 사용한 만큼 환산 일수(연차 1d·반차 0.5d·반반차 0.25d)를 출근 남은일에서 제외한다.
const 연차분요청 = computed(() =>
  (Number(연차일수.value) || 0) * 480 +
  (Number(반차수.value) || 0) * 240 +
  (Number(반반차수.value) || 0) * 120,
)
const 연차예산분 = computed(() => 입력분.value) // 현재까지 근무시간 = 연차 상한
const 연차잔여분 = computed(() => Math.max(0, 연차예산분.value - 연차분요청.value))
const 연차분 = computed(() =>
  연차여부.value ? Math.min(연차분요청.value, 연차예산분.value) : 0,
)
const 연차일수환산 = computed(() => 연차분.value / 480)
const 연차초과여부 = computed(
  () => 연차여부.value && 연차분요청.value > 연차예산분.value,
)
const 출근남은일 = computed(() =>
  Math.max(0, 남은근무일.value - 재택일수.value - 연차일수환산.value),
)

// 재택근무를 처음 켜면 남은 금요일 전체를 기본 선택
watch(재택근무여부, (켜짐) => {
  if (켜짐 && 재택근무일수.value === 0) {
    재택근무일수.value = 남은금요일.value
  }
})
// 월/입사일 변경 등으로 남은 금요일이 줄면 선택값을 자동 보정
watchEffect(() => {
  if (재택근무일수.value > 남은금요일.value) {
    재택근무일수.value = 남은금요일.value
  }
})
// '현재까지 근무시간'이 줄면 연차 합계가 한도를 넘지 않도록 반반차→반차→연차 순으로 보정
watchEffect(() => {
  const 예산 = 연차예산분.value
  let 연 = Number(연차일수.value) || 0
  let 반 = Number(반차수.value) || 0
  let 반반 = Number(반반차수.value) || 0
  let 합 = 연 * 480 + 반 * 240 + 반반 * 120
  while (합 > 예산 && 반반 > 0) { 반반--; 합 -= 120 }
  while (합 > 예산 && 반 > 0) { 반--; 합 -= 240 }
  while (합 > 예산 && 연 > 0) { 연--; 합 -= 480 }
  if (연 !== 연차일수.value) 연차일수.value = 연
  if (반 !== 반차수.value) 반차수.value = 반
  if (반반 !== 반반차수.value) 반반차수.value = 반반
})
// 증감: 잔여 예산(현재까지 근무시간 − 이미 지정한 연차) 안에서만 증가 허용
function 연차증감(필드, 델타) {
  const 단위 = 필드 === '연차' ? 480 : 필드 === '반차' ? 240 : 120
  if (델타 > 0 && 연차잔여분.value < 단위) return
  const 대상 = 필드 === '연차' ? 연차일수 : 필드 === '반차' ? 반차수 : 반반차수
  대상.value = Math.max(0, (Number(대상.value) || 0) + 델타)
}
const 남은의무분 = computed(() =>
  Math.max(0, 의무근로분.value - 반영분.value),
)
const 남은최대분 = computed(() =>
  Math.max(0, 최대근로분.value - 반영분.value),
)
const 의무달성일평균분 = computed(() => {
  if (출근남은일.value === 0) return 0
  return Math.round(남은의무분.value / 출근남은일.value)
})
const 최대달성일평균분 = computed(() => {
  if (출근남은일.value === 0) return 0
  return Math.round(남은최대분.value / 출근남은일.value)
})
const 달성률 = computed(() => {
  if (의무근로분.value === 0) return 0
  return Math.min(100, Math.floor((반영분.value / 의무근로분.value) * 100))
})
const 초과분 = computed(() =>
  Math.max(0, 반영분.value - 의무근로분.value),
)
const 의무달성여부 = computed(() => 반영분.value >= 의무근로분.value)
const 의무대비차 = computed(() =>
  Math.abs(반영분.value - 의무근로분.value),
)
const 최대대비차 = computed(() =>
  Math.abs(반영분.value - 최대근로분.value),
)
const 이달공휴일 = computed(() =>
  월별공휴일조회(선택연도.value, 선택월.value),
)
const 공휴일데이터있음 = computed(() => 공휴일데이터존재여부(선택연도.value))

// 다음 달
const 다음달 = computed(() => {
  const 월 = 선택월.value === 12 ? 1 : 선택월.value + 1
  const 연도 = 선택월.value === 12 ? 선택연도.value + 1 : 선택연도.value
  return { 연도, 월 }
})
const 다음달표시 = computed(
  () => `${다음달.value.연도}년 ${다음달.value.월}월`,
)
const 다음달소정근로일 = computed(() =>
  월소정근로일수조회(다음달.value.연도, 다음달.value.월),
)
const 다음달의무근로분 = computed(
  () => 다음달소정근로일.value * 하루근무분,
)
const 다음달최대근로분 = computed(
  () => 다음달의무근로분.value + 고정연장분.value,
)
const 다음달공휴일 = computed(() =>
  월별공휴일조회(다음달.value.연도, 다음달.value.월),
)

const 진행바색상 = computed(() => {
  if (달성률.value >= 100) return '#10b981'
  if (달성률.value >= 70) return '#f59e0b'
  return '#3b82f6'
})

const 선택월표시 = computed(
  () => `${선택연도.value}년 ${선택월.value}월`,
)

const 지난달여부 = computed(() => {
  const 선택 = new Date(선택연도.value, 선택월.value - 1, 1)
  const 이번달 = new Date(현재연도, 현재월 - 1, 1)
  return 선택 < 이번달
})

const 이번달여부 = computed(
  () => 선택연도.value === 현재연도 && 선택월.value === 현재월,
)

watchEffect(() => {
  document.title = `${선택월표시.value} 근무시간 계산기`
})

watchEffect(() => {
  if (!입사한달여부.value) return
  const 기본 = 이번달여부.value ? 오늘.getDate() : 1
  if (입사일.value < 1 || 입사일.value > 월말일.value) {
    입사일.value = Math.min(월말일.value, 기본)
  }
})
</script>

<template>
  <div class="calculator">
    <header class="calc-header">
      <h1>⏱ 근무시간 계산기</h1>
      <p class="subtitle">소정근로일 기준 의무·최대 근로시간과 일평균 목표를 확인하세요</p>
      <div class="today-chip">
        <span class="chip" :class="급여주여부 ? 'chip--pay' : 'chip--date'">
          <span class="chip-ico">{{ 급여주여부 ? '💸' : '📆' }}</span>
          <span class="chip-txt">{{ 오늘표시 }}</span>
        </span>
        <span v-if="재택안내" class="chip chip--wfh">
          <span class="chip-ico">🏠</span>
          <span class="chip-txt">{{ 재택안내 }}</span>
        </span>
      </div>
    </header>

    <button
      type="button"
      class="theme-fab"
      :aria-label="다크모드 ? '라이트 모드로 전환' : '다크 모드로 전환'"
      :title="다크모드 ? '라이트 모드로' : '다크 모드로'"
      @click="테마토글"
    >
      <span class="theme-fab-icon">{{ 다크모드 ? '☀️' : '🌙' }}</span>
    </button>

    <!-- 월 선택 -->
    <section class="card month-selector">
      <div class="selector-row">
        <div class="select-group">
          <label for="연도선택">연도</label>
          <select id="연도선택" v-model="선택연도">
            <option v-for="연도 in 연도목록" :key="연도" :value="연도">{{ 연도 }}년</option>
          </select>
        </div>
        <div class="select-group">
          <label for="월선택">월</label>
          <select id="월선택" v-model="선택월">
            <option v-for="월 in 월목록" :key="월" :value="월">{{ 월 }}월</option>
          </select>
        </div>
        <div class="month-badge">
          <span>{{ 선택월표시 }}</span>
          <span v-if="이번달여부" class="badge current">이번 달</span>
          <span v-else-if="지난달여부" class="badge past">지난 달</span>
          <span v-else class="badge future">다음 달</span>
        </div>
      </div>
      <div class="join-inline">
        <label class="join-checkbox">
          <input type="checkbox" v-model="입사한달여부" />
          <span>이 달에 입사했어요</span>
        </label>
        <div v-if="입사한달여부" class="join-date">
          <label for="입사일" class="join-date-label">입사일</label>
          <select id="입사일" v-model.number="입사일" class="join-date-select">
            <option v-for="일 in 일목록" :key="일" :value="일">{{ 일 }}일</option>
          </select>
        </div>
        <p v-if="입사한달여부" class="input-hint join-hint">
          <strong>{{ 유효입사일 }}일</strong>부터 월말까지 근무일로 계산
          <span class="hint-extra">(입사일도 포함)</span>
        </p>
      </div>
    </section>

    <!-- 공휴일 데이터 부재 알림 -->
    <div v-if="!공휴일데이터있음" class="warn-notice" role="alert">
      ⚠ {{ 선택연도 }}년 공휴일 데이터가 없습니다. 근무일 계산에서 공휴일이 평일로 간주되어 부정확할 수 있습니다.
    </div>

    <!-- 근무일 요약 -->
    <section class="summary-grid" aria-live="polite">
      <div class="summary-card blue">
        <div class="summary-icon">📅</div>
        <div class="summary-label">이달 근무일 <span class="label-aside">(소정 근로일)</span></div>
        <div class="summary-value">{{ 소정근로일 }}<span class="unit">일</span></div>
        <div class="summary-sub">
          <template v-if="입사한달여부">{{ 유효입사일 }}일부터 · 주말·공휴일 제외</template>
          <template v-else>주말·공휴일 제외</template>
        </div>
      </div>
      <div class="summary-card green">
        <div class="summary-icon">✅</div>
        <div class="summary-label">의무 근로시간</div>
        <div class="summary-value">{{ 시분변환(의무근로분) }}</div>
        <div class="summary-sub">8:00 × {{ 소정근로일 }}일</div>
      </div>
      <div class="summary-card purple">
        <div class="summary-icon">⏰</div>
        <div class="summary-label">최대 근로시간</div>
        <div class="summary-value">{{ 시분변환(최대근로분) }}</div>
        <div class="summary-sub">8:00 × {{ 소정근로일 }}일 + {{ 시분변환(고정연장분) }}</div>
      </div>
    </section>

    <!-- 입력 설정 -->
    <section class="card input-section">
      <h2 class="section-title">⚙️ 근무 설정</h2>
      <div v-if="반영분 > 0" class="reflected-summary" aria-live="polite">
        <span class="reflected-label">총 반영 시간</span>
        <span class="reflected-value">{{ 시분변환(반영분) }}</span>
        <span class="reflected-formula">
          누적 {{ 시분변환(입력분) }}<template v-if="연차분 > 0"> (연차 {{ 시분변환(연차분) }} 포함)</template><template v-if="오늘예상분 > 0"> + 오늘 {{ 시분변환(오늘예상분) }}</template>
        </span>
      </div>
      <div class="input-grid">
        <div class="input-group">
          <label for="고정연장">월 고정 연장근무 (시:분)</label>
          <div class="input-with-unit">
            <input
              id="고정연장"
              v-model="고정연장시간"
              @blur="고정연장정규화"
              :class="{ error: !고정연장유효 }"
              :aria-invalid="!고정연장유효"
              type="text"
              inputmode="numeric"
              placeholder="10:00"
              pattern="[0-9:]*"
            />
          </div>
          <p v-if="!고정연장유효" class="input-error">
            ⚠ 형식이 올바르지 않습니다. 예: <code>10:00</code> 또는 <code>1000</code>
          </p>
          <p v-else class="input-hint">
            <strong>형식</strong>: <code>10:00</code>, <code>7:30</code>
            <span class="hint-extra">(콜론 없이 <code>1000</code>도 가능)</span>
          </p>
        </div>
        <div class="input-group">
          <label for="근무입력">현재까지 근무시간 (시:분)</label>
          <div class="input-with-unit">
            <input
              id="근무입력"
              v-model="입력근무시간"
              @blur="입력근무정규화"
              :class="{ error: !입력유효 }"
              :aria-invalid="!입력유효"
              type="text"
              inputmode="numeric"
              placeholder="0:00"
              pattern="[0-9:]*"
            />
          </div>
          <p v-if="!입력유효" class="input-error">
            ⚠ 형식이 올바르지 않습니다. 예: <code>23:30</code> 또는 <code>2330</code>
          </p>
          <p v-else class="input-hint">
            <strong>형식</strong>: <code>23:30</code>, <code>137:30</code>
            <span class="hint-extra">(콜론 없이 <code>2330</code>도 가능)</span>
          </p>
        </div>
        <div v-if="!지난달여부" class="input-group input-today">
          <div class="today-header">
            <label>🏠 금요일 재택근무</label>
            <label
              class="join-checkbox today-wfh"
              :class="{ disabled: 남은금요일 === 0 }"
            >
              <input type="checkbox" v-model="재택근무여부" :disabled="남은금요일 === 0" />
              <span>사용</span>
            </label>
          </div>
          <div v-if="재택근무여부 && 남은금요일 > 0" class="join-date">
            <label for="재택일수" class="join-date-label">재택 일수</label>
            <select id="재택일수" v-model.number="재택근무일수" class="join-date-select">
              <option v-for="n in (남은금요일 + 1)" :key="n - 1" :value="n - 1">{{ n - 1 }}일</option>
            </select>
          </div>
          <p v-if="남은금요일 === 0" class="input-hint join-hint">
            남은 금요일이 없어 재택근무를 신청할 수 없습니다.
          </p>
          <p v-else-if="재택근무여부" class="input-hint join-hint">
            남은 금요일 <strong>{{ 남은금요일 }}일</strong> 중 <strong>{{ 재택일수 }}일</strong>을 재택근무로 반영
            <span class="hint-extra">(재택일은 8시간이 자동 인정되어 일평균 목표 계산에서 제외)</span>
          </p>
        </div>
        <div class="input-group input-today">
          <div class="today-header">
            <label>🌴 연차 / 반차</label>
            <label
              class="join-checkbox today-wfh"
              :class="{ disabled: 연차예산분 === 0 }"
              :title="연차예산분 === 0 ? '현재까지 근무시간을 먼저 입력하세요' : undefined"
            >
              <input type="checkbox" v-model="연차여부" :disabled="연차예산분 === 0" />
              <span>사용</span>
            </label>
          </div>
          <div v-if="연차여부 && 연차예산분 > 0" class="annual-panel">
            <div v-if="연차분 > 0" class="annual-hero">
              <div class="annual-hero-main">
                <span class="annual-hero-label">🌴 연차로 지정한 시간</span>
                <span class="annual-hero-value">{{ 시분변환(연차분) }}</span>
              </div>
              <span class="annual-hero-badge">출근일 −{{ 연차일수환산 }}일</span>
            </div>
            <p v-else class="annual-hero-empty">
              연차·반차를 지정하면 그만큼 출근일이 줄어요
            </p>

            <div class="annual-steppers">
              <div
                v-for="항목 in [
                  { 키: '연차', 이름: '연차', 시간: '8h', 값: 연차일수, 단위: 480 },
                  { 키: '반차', 이름: '반차', 시간: '4h', 값: 반차수, 단위: 240 },
                  { 키: '반반차', 이름: '반반차', 시간: '2h', 값: 반반차수, 단위: 120 },
                ]"
                :key="항목.키"
                class="annual-stepper"
                :class="{ filled: 항목.값 > 0 }"
              >
                <div class="annual-stepper-top">
                  <span class="annual-stepper-name">{{ 항목.이름 }}</span>
                  <span class="annual-stepper-hour">{{ 항목.시간 }}</span>
                </div>
                <div class="annual-stepper-ctrl">
                  <button
                    type="button"
                    class="annual-btn"
                    :disabled="항목.값 <= 0"
                    :aria-label="`${항목.이름} 줄이기`"
                    @click="연차증감(항목.키, -1)"
                  >−</button>
                  <span class="annual-count">{{ 항목.값 }}</span>
                  <button
                    type="button"
                    class="annual-btn"
                    :disabled="연차잔여분 < 항목.단위"
                    :aria-label="`${항목.이름} 늘리기`"
                    @click="연차증감(항목.키, 1)"
                  >+</button>
                </div>
              </div>
            </div>

            <p class="input-hint annual-hint">
              <span class="hint-extra">현재까지 근무시간 {{ 시분변환(연차예산분) }} 중 지정 · 남은 한도 {{ 시분변환(연차잔여분) }}</span>
            </p>
          </div>
          <p v-else-if="연차예산분 === 0" class="input-hint">
            현재까지 근무시간을 먼저 입력하면 그 안에서 연차를 지정할 수 있어요.
          </p>
          <p v-else class="input-hint">
            현재까지 근무시간 중 연차를 지정하면 그만큼 출근일이 줄어요.
            <span class="hint-extra">연차 −1일 · 반차 −0.5일 · 반반차 −0.25일</span>
          </p>
        </div>
        <div class="input-group input-today">
          <div class="today-header">
            <label>오늘 예상 근무시간</label>
            <div class="today-controls">
              <label v-if="오늘금요일여부" class="join-checkbox today-wfh">
                <input type="checkbox" v-model="오늘재택근무" />
                <span>🏠 재택</span>
              </label>
              <div v-if="!오늘재택근무" class="mode-switch" role="tablist" aria-label="입력 방식">
                <button
                  type="button"
                  role="tab"
                  :aria-selected="오늘입력모드 === '출퇴근'"
                  :class="{ active: 오늘입력모드 === '출퇴근' }"
                  @click="오늘입력모드 = '출퇴근'"
                >출·퇴근으로 계산</button>
                <button
                  type="button"
                  role="tab"
                  :aria-selected="오늘입력모드 === '직접'"
                  :class="{ active: 오늘입력모드 === '직접' }"
                  @click="오늘입력모드 = '직접'"
                >직접 입력</button>
              </div>
            </div>
          </div>

          <!-- 재택근무 활성 상태: 입력 영역을 대체 -->
          <div v-if="오늘재택근무" class="wfh-active-card">
            <span class="wfh-active-icon">🏠</span>
            <div class="wfh-active-body">
              <p class="wfh-active-title">오늘 재택근무 적용됨 · <strong>8:00</strong></p>
              <p class="wfh-active-sub">
                오늘 근무시간은 <strong>8:00</strong>으로 계산됩니다.
                재택 근무시간은 위 ‘현재까지 근무시간’에 포함해 주세요.
              </p>
            </div>
          </div>

          <template v-if="!오늘재택근무 && 오늘입력모드 === '출퇴근'">
            <div class="commute-grid">
              <div class="commute-field">
                <label>출근</label>
                <div class="time-input-wrap">
                  <VueDatePicker
                    v-model="출근객체"
                    time-picker
                    :is-24="true"
                    auto-apply
                    :clearable="false"
                    :minutes-increment="5"
                    :minutes-grid-increment="5"
                    :dark="다크모드"
                    placeholder="출근 시각"
                    class="dp-wrap"
                  />
                  <button type="button" class="now-btn" @click="출근지금" title="현재 시각으로">📍 지금</button>
                </div>
              </div>
              <div class="commute-field">
                <label>퇴근 예상</label>
                <div class="time-input-wrap">
                  <VueDatePicker
                    v-model="퇴근객체"
                    time-picker
                    :is-24="true"
                    auto-apply
                    :clearable="false"
                    :minutes-increment="5"
                    :minutes-grid-increment="5"
                    :dark="다크모드"
                    placeholder="퇴근 시각"
                    class="dp-wrap"
                  />
                  <button type="button" class="now-btn" @click="퇴근지금" title="현재 시각으로">📍 지금</button>
                </div>
              </div>
              <div class="commute-field commute-break">
                <label for="휴게수동">휴게시간</label>
                <div class="break-row">
                  <select
                    id="휴게수동"
                    v-model.number="휴게수동분"
                    :disabled="휴게자동"
                    class="break-select"
                  >
                    <option :value="0">0분</option>
                    <option :value="30">30분</option>
                    <option :value="45">45분</option>
                    <option :value="60">1시간</option>
                    <option :value="90">1시간 30분</option>
                    <option :value="120">2시간</option>
                  </select>
                  <label class="auto-toggle">
                    <input type="checkbox" v-model="휴게자동" />
                    <span>자동</span>
                  </label>
                </div>
              </div>
            </div>

            <div class="commute-result" :class="{ midnight: 자정넘김여부 }" aria-live="polite">
              <span class="result-tag">오늘 예상</span>
              <span class="result-time">{{ 시분변환(오늘예상분) }}</span>
              <span class="result-formula">
                체류 {{ 시분변환(총체류분) }} − 휴게 {{ 시분변환(휴게분) }}
                <template v-if="휴게자동">(자동)</template>
              </span>
              <span v-if="자정넘김여부" class="midnight-badge" title="퇴근이 출근보다 빠르거나 같음">
                🌙 자정 넘김
              </span>
            </div>
          </template>

          <template v-else-if="!오늘재택근무">
            <div class="input-with-unit">
              <input
                id="오늘예상"
                v-model="오늘예상시간"
                @blur="오늘예상정규화"
                :class="{ error: !오늘예상유효 }"
                :aria-invalid="!오늘예상유효"
                type="text"
                inputmode="numeric"
                placeholder="0:00"
                pattern="[0-9:]*"
              />
            </div>
            <p v-if="!오늘예상유효" class="input-error">
              ⚠ 형식이 올바르지 않습니다. 예: <code>8:00</code> 또는 <code>800</code>
            </p>
            <p v-else class="input-hint">
              오늘 추가로 일할 시간 · <strong>현재까지에 더해</strong> 합산
              <span class="hint-extra">(기본 <code>0:00</code>)</span>
            </p>
          </template>
        </div>
      </div>
    </section>

    <!-- 진행 상황 -->
    <component
      :is="달성현황"
      :입력분="반영분"
      :의무근로분="의무근로분"
      :초과분="초과분"
      :의무달성여부="의무달성여부"
      :경과근무일="경과근무일"
      :소정근로일="소정근로일"
      :달성률="달성률"
      :진행바색상="진행바색상"
    />

    <!-- 결과 -->
    <section class="card result-section" aria-live="polite">
      <h2 class="section-title">📋 {{ 지난달여부 ? '지난 달 결과 요약' : '남은 근무 계획' }}</h2>

      <div v-if="지난달여부" class="notice past-notice">
        ℹ️ 지난 달입니다. 입력한 누적 시간으로 의무·최대 대비 결과만 표시합니다.
      </div>

      <div v-if="지난달여부 && 반영분 === 0" class="empty-banner">
        💡 위에서 해당 달의 <strong>실제 근무시간</strong>을 입력하면 의무·최대 달성 결과를 확인할 수 있습니다.
      </div>

      <div v-if="지난달여부 && 반영분 > 0" class="result-grid">
        <div class="result-item">
          <div class="result-label">실제 근무시간</div>
          <div class="result-value highlight-blue">{{ 시분변환(반영분) }}</div>
          <div class="result-sub">달성률 {{ 달성률 }}%</div>
        </div>
        <div class="result-item">
          <div class="result-label">의무 대비</div>
          <div class="result-value" :class="의무달성여부 ? 'highlight-green' : 'highlight-red'">
            {{ 의무달성여부 ? '+' : '−' }}{{ 시분변환(의무대비차) }}
          </div>
          <div class="result-sub">
            <template v-if="의무달성여부">의무 {{ 시분변환(의무근로분) }} 초과 달성</template>
            <template v-else>의무 {{ 시분변환(의무근로분) }} 미달</template>
          </div>
        </div>
        <div class="result-item">
          <div class="result-label">최대 대비</div>
          <div class="result-value highlight-purple">
            {{ 반영분 >= 최대근로분 ? '+' : '−' }}{{ 시분변환(최대대비차) }}
          </div>
          <div class="result-sub">최대 {{ 시분변환(최대근로분) }}</div>
        </div>
      </div>

      <div v-if="!지난달여부 && 반영분 === 0" class="empty-banner">
        💡 위에서 <strong>현재까지 근무시간</strong>을 입력하면 남은 시간과 일평균 목표가 계산됩니다.
      </div>

      <div v-if="!지난달여부" class="result-grid">
        <div class="result-item">
          <div class="result-label">남은 근무일</div>
          <div class="result-value highlight-blue">
            {{ 남은근무일 }}<span class="unit">일</span>
          </div>
          <div class="result-sub">
            <template v-if="재택일수 > 0 || 연차일수환산 > 0">
              출근 {{ 출근남은일 }}일<template v-if="재택일수 > 0"> · 재택 {{ 재택일수 }}일</template><template v-if="연차일수환산 > 0"> · 연차 {{ 연차일수환산 }}일</template> · 오늘 제외
            </template>
            <template v-else>오늘 제외 · 내일부터</template>
          </div>
        </div>
        <div class="result-item">
          <div class="result-label">남은 의무 근무시간</div>
          <div class="result-value highlight-green">
            <template v-if="반영분 > 0">{{ 시분변환(남은의무분) }}</template>
            <span v-else class="placeholder-dash">—</span>
          </div>
          <div v-if="반영분 > 0" class="result-sub">
            의무 {{ 시분변환(의무근로분) }} − 누적 {{ 시분변환(입력분) }}<template v-if="오늘예상분 > 0"> − 오늘 {{ 시분변환(오늘예상분) }}</template>
          </div>
          <div v-else class="result-sub">의무 {{ 시분변환(의무근로분) }}</div>
        </div>
        <div class="result-item">
          <div class="result-label">남은 최대 근무시간</div>
          <div class="result-value highlight-purple">
            <template v-if="반영분 > 0">{{ 시분변환(남은최대분) }}</template>
            <span v-else class="placeholder-dash">—</span>
          </div>
          <div v-if="반영분 > 0" class="result-sub">
            최대 {{ 시분변환(최대근로분) }} − 누적 {{ 시분변환(입력분) }}<template v-if="오늘예상분 > 0"> − 오늘 {{ 시분변환(오늘예상분) }}</template>
          </div>
          <div v-else class="result-sub">최대 {{ 시분변환(최대근로분) }}</div>
        </div>
      </div>

      <div v-if="!지난달여부 && 출근남은일 > 0 && 반영분 > 0" class="avg-section">
        <h3 class="avg-title">일평균 목표 근무시간</h3>
        <p v-if="재택일수 > 0 || 연차일수환산 > 0" class="avg-note">
          <template v-if="재택일수 > 0">재택 {{ 재택일수 }}일</template><template v-if="재택일수 > 0 && 연차일수환산 > 0"> · </template><template v-if="연차일수환산 > 0">연차 {{ 연차일수환산 }}일</template>(8시간 자동 인정)을 제외한 <strong>출근 {{ 출근남은일 }}일</strong> 기준입니다.
        </p>
        <div class="avg-grid">
          <div class="avg-card">
            <span class="avg-tag tag-mandatory">의무</span>
            <div class="avg-value">{{ 시분변환(의무달성일평균분) }}</div>
            <div class="avg-sub">출근 {{ 출근남은일 }}일 동안 매일</div>
          </div>
          <div class="avg-card">
            <span class="avg-tag tag-max">최대</span>
            <div class="avg-value">{{ 시분변환(최대달성일평균분) }}</div>
            <div class="avg-sub">출근 {{ 출근남은일 }}일 동안 매일</div>
          </div>
        </div>
      </div>

      <div v-if="!지난달여부 && 남은근무일 === 0" class="notice">
        🎊 남은 근무일이 없습니다!
      </div>
      <div
        v-else-if="!지난달여부 && 출근남은일 === 0 && (재택일수 > 0 || 연차일수환산 > 0)"
        class="notice"
      >
        🏠 남은 근무일 {{ 남은근무일 }}일이 모두 재택·연차입니다. 출근일이 없어 일평균 목표를 표시하지 않습니다.
      </div>
    </section>

    <!-- 공휴일 목록 -->
    <component :is="공휴일목록" :선택월표시="선택월표시" :이달공휴일="이달공휴일" />

    <!-- 다음 달 미리보기 -->
    <component
      :is="다음달미리보기"
      :다음달표시="다음달표시"
      :다음달소정근로일="다음달소정근로일"
      :다음달의무근로분="다음달의무근로분"
      :다음달최대근로분="다음달최대근로분"
      :다음달공휴일="다음달공휴일"
    />
  </div>
</template>

<style scoped>
.calculator {
  max-width: 860px;
  margin: 0 auto;
  padding: 24px 16px 48px;
  font-family: 'Pretendard', 'Noto Sans KR', system-ui, sans-serif;
  color: #1e293b;
}

/* Header */
.calc-header {
  text-align: center;
  margin-bottom: 32px;
}
.calc-header h1 {
  font-size: 1.75rem;
  font-weight: 800;
  margin: 0 0 6px;
  color: #191f28;
  letter-spacing: -0.03em;
}
.subtitle {
  color: #8b95a1;
  font-size: 0.9rem;
  margin: 0;
}
.today-chip {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 8px;
  margin-top: 16px;
}
.chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 14px 6px 7px;
  border-radius: 999px;
  border: 1px solid transparent;
  font-size: 0.82rem;
  font-weight: 700;
  letter-spacing: -0.01em;
  position: relative;
  overflow: hidden;
  isolation: isolate;
  animation: chip-in 0.55s cubic-bezier(0.22, 1, 0.36, 1) both;
}
.today-chip .chip:nth-child(2) {
  animation-delay: 0.09s;
}
.chip-ico {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  font-size: 0.74rem;
  line-height: 1;
}
.chip-ico,
.chip-txt {
  position: relative;
  z-index: 1;
}
@keyframes chip-in {
  from { opacity: 0; transform: translateY(7px); }
  to { opacity: 1; transform: translateY(0); }
}
.chip--date {
  background: #f4f6f8;
  border-color: #e6e9ee;
  color: #3f4b5b;
}
.chip--date .chip-ico {
  background: #fff;
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.1);
}
.chip--wfh {
  background: #eef4ff;
  border-color: #cfe0ff;
  color: #1d4ed8;
}
.chip--wfh .chip-ico {
  background: #fff;
  box-shadow: 0 1px 2px rgba(29, 78, 216, 0.14);
}
.chip--pay {
  background: linear-gradient(135deg, #fdeaa6 0%, #f6c945 52%, #efb429 100%);
  border-color: #e0a100;
  color: #6a4905;
  box-shadow:
    0 4px 16px rgba(239, 180, 41, 0.42),
    inset 0 1px 0 rgba(255, 255, 255, 0.55);
}
.chip--pay .chip-ico {
  background: rgba(255, 255, 255, 0.6);
  box-shadow: 0 1px 2px rgba(120, 80, 0, 0.22);
}
.chip--pay::before {
  content: '';
  position: absolute;
  top: 0;
  left: -60%;
  width: 42%;
  height: 100%;
  z-index: 2;
  background: linear-gradient(100deg, transparent, rgba(255, 255, 255, 0.75), transparent);
  transform: skewX(-20deg);
  animation: chip-shine 5s ease-in-out 1.2s infinite;
}
@keyframes chip-shine {
  0% { left: -60%; }
  16% { left: 135%; }
  100% { left: 135%; }
}
@media (prefers-reduced-motion: reduce) {
  .chip { animation: none; }
  .chip--pay::before { animation: none; opacity: 0; }
}

/* Theme toggle FAB */
.theme-fab {
  position: fixed;
  right: 20px;
  bottom: max(20px, env(safe-area-inset-bottom));
  z-index: 50;
  width: 52px;
  height: 52px;
  border-radius: 50%;
  background: #fff;
  border: 1px solid #ebedf0;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  font-size: 1.35rem;
  color: #4e5968;
  box-shadow: 0 6px 20px rgba(15, 23, 42, 0.12), 0 2px 4px rgba(15, 23, 42, 0.06);
  transition: background 0.15s, border-color 0.15s, transform 0.1s, box-shadow 0.15s;
}
.theme-fab:hover {
  background: #f7f8fa;
  border-color: #d1d6db;
  transform: translateY(-1px);
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.15), 0 3px 6px rgba(15, 23, 42, 0.08);
}
.theme-fab:active {
  transform: translateY(0) scale(0.95);
}
.theme-fab:focus-visible {
  outline: none;
  border-color: #06c755;
  box-shadow: 0 0 0 3px rgba(6, 199, 85, 0.22), 0 6px 20px rgba(15, 23, 42, 0.12);
}
.theme-fab-icon {
  display: inline-block;
  line-height: 1;
}

/* Label aside */
.label-aside {
  font-weight: 400;
  color: #94a3b8;
  font-size: 0.78rem;
  margin-left: 4px;
}

/* Month selector */
.selector-row {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}
.select-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.select-group label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.select-group select {
  padding: 10px 36px 10px 14px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 500;
  color: #0f172a;
  background: #f8fafc url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E") no-repeat right 12px center;
  appearance: none;
  cursor: pointer;
  transition: border-color 0.2s;
}
.select-group select:focus-visible {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}
.join-group {
  flex-direction: row;
  align-items: center;
  gap: 8px;
}
.join-checkbox {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  height: 36px;
  padding: 0 12px;
  border-radius: 10px;
  font-size: 0.88rem;
  font-weight: 600;
  color: #475569;
  background: #f8fafc;
  border: 1.5px solid #e2e8f0;
  cursor: pointer;
  text-transform: none;
  letter-spacing: normal;
  user-select: none;
  transition: background 0.15s, border-color 0.15s;
}
.join-checkbox:hover {
  background: #f1f5f9;
  border-color: #cbd5e1;
}
.join-checkbox:has(input:checked) {
  background: #eff6ff;
  border-color: #93c5fd;
  color: #1d4ed8;
}
.join-checkbox input[type='checkbox'] {
  width: 16px;
  height: 16px;
  accent-color: #3b82f6;
  cursor: pointer;
  margin: 0;
}
.join-checkbox.disabled {
  opacity: 0.55;
  cursor: not-allowed;
}
.join-checkbox.disabled input[type='checkbox'] {
  cursor: not-allowed;
}
.join-inline {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  row-gap: 10px;
  column-gap: 12px;
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px dashed #e2e8f0;
}
.join-date {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}
.join-date-label {
  font-size: 0.85rem;
  font-weight: 600;
  color: #475569;
}
.join-date-select {
  height: 36px;
  padding: 0 32px 0 12px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  font-size: 0.9rem;
  font-weight: 600;
  color: #0f172a;
  background: #f8fafc url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E") no-repeat right 10px center;
  appearance: none;
  cursor: pointer;
  transition: border-color 0.2s, box-shadow 0.2s;
}
.join-date-select:focus-visible {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}
.join-hint {
  margin: 0;
  flex-basis: 100%;
  font-size: 0.8rem;
  color: #64748b;
}
.month-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
  font-size: 1.1rem;
  font-weight: 600;
  color: #0f172a;
}
.badge {
  font-size: 0.75rem;
  font-weight: 600;
  padding: 3px 10px;
  border-radius: 20px;
}
.badge.current {
  background: #dbeafe;
  color: #1d4ed8;
}
.badge.past {
  background: #f1f5f9;
  color: #64748b;
}
.badge.future {
  background: #fef3c7;
  color: #92400e;
}

/* Summary grid */
.summary-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-bottom: 20px;
}
.summary-card {
  border-radius: 16px;
  padding: 20px;
  text-align: center;
  border: 1px solid transparent;
}
.summary-card.blue {
  background: #eff6ff;
  border-color: #bfdbfe;
}
.summary-card.green {
  background: #f0fdf4;
  border-color: #bbf7d0;
}
.summary-card.purple {
  background: #faf5ff;
  border-color: #e9d5ff;
}
.summary-icon {
  font-size: 1.75rem;
  margin-bottom: 8px;
}
.summary-label {
  font-size: 0.82rem;
  font-weight: 600;
  color: #64748b;
  margin-bottom: 8px;
}
.summary-value {
  font-size: 2.2rem;
  font-weight: 800;
  color: #0f172a;
  line-height: 1;
  margin-bottom: 6px;
}
.summary-value .unit {
  font-size: 1rem;
  font-weight: 500;
  margin-left: 2px;
  color: #64748b;
}
.summary-sub {
  font-size: 0.78rem;
  color: #94a3b8;
}

/* Input section */
.reflected-summary {
  display: flex;
  align-items: baseline;
  flex-wrap: wrap;
  gap: 8px 12px;
  padding: 12px 16px;
  margin-bottom: 18px;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 10px;
}
.reflected-label {
  font-size: 0.82rem;
  font-weight: 600;
  color: #1e40af;
}
.reflected-value {
  font-size: 1.1rem;
  font-weight: 800;
  color: #1d4ed8;
}
.reflected-formula {
  font-size: 0.78rem;
  color: #64748b;
  margin-left: auto;
}
.input-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}
.input-today {
  grid-column: 1 / -1;
}
.today-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px 12px;
}
.mode-switch {
  display: inline-flex;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 3px;
  gap: 2px;
}
.mode-switch button {
  appearance: none;
  border: none;
  background: transparent;
  padding: 6px 12px;
  font-size: 0.82rem;
  font-weight: 600;
  color: #64748b;
  border-radius: 7px;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}
.mode-switch button:hover {
  color: #334155;
}
.mode-switch button.active {
  background: #fff;
  color: #1d4ed8;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06);
}
.today-controls {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}
.today-wfh {
  height: 38px;
}
/* 재택근무 활성 상태 카드 */
.wfh-active-card {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 14px 16px;
  background: #eff6ff;
  border: 1.5px solid #bfdbfe;
  border-radius: 12px;
}
.wfh-active-icon {
  font-size: 1.5rem;
  line-height: 1.2;
}
.wfh-active-body {
  flex: 1;
  min-width: 0;
}
.wfh-active-title {
  font-size: 0.92rem;
  font-weight: 700;
  color: #1d4ed8;
  margin: 0 0 4px;
}
.wfh-active-sub {
  font-size: 0.8rem;
  color: #475569;
  margin: 0;
  line-height: 1.55;
}
.dp-wrap {
  flex: 1;
  min-width: 0;
}
.dp-wrap :deep(.dp__input) {
  height: 40px;
  border-radius: 10px;
  border: 1.5px solid #e2e8f0;
  background: #f8fafc;
  font-size: 0.95rem;
  font-weight: 700;
  color: #0f172a;
  padding-left: 36px;
}
.dp-wrap :deep(.dp__input:focus),
.dp-wrap :deep(.dp__input_focus) {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}
.commute-grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 12px;
}
.commute-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.commute-field label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
.time-input-wrap {
  display: flex;
  gap: 6px;
}
.now-btn {
  appearance: none;
  border: 1.5px solid #e2e8f0;
  background: #fff;
  border-radius: 10px;
  padding: 0 10px;
  font-size: 0.78rem;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.15s, border-color 0.15s, color 0.15s;
}
.now-btn:hover {
  background: #eff6ff;
  border-color: #93c5fd;
  color: #1d4ed8;
}
.break-row {
  display: flex;
  gap: 8px;
  align-items: center;
}
.break-select {
  flex: 1;
  min-width: 0;
  height: 40px;
  padding: 0 32px 0 12px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  font-size: 0.9rem;
  font-weight: 600;
  color: #0f172a;
  background: #f8fafc url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E") no-repeat right 10px center;
  appearance: none;
  cursor: pointer;
  transition: border-color 0.2s, box-shadow 0.2s;
}
.break-select:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}
.break-select:focus-visible {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}
.auto-toggle {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.8rem;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  user-select: none;
  white-space: nowrap;
}
.auto-toggle input[type='checkbox'] {
  width: 14px;
  height: 14px;
  accent-color: #3b82f6;
  cursor: pointer;
  margin: 0;
}
.commute-result {
  display: flex;
  flex-wrap: wrap;
  align-items: baseline;
  gap: 6px 12px;
  margin-top: 12px;
  padding: 12px 16px;
  background: #e6f9f0;
  border: 1px solid #b7e8c8;
  border-radius: 12px;
}
.commute-result .result-tag {
  font-size: 0.72rem;
  font-weight: 700;
  color: #06873e;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.commute-result .result-time {
  font-size: 1.15rem;
  font-weight: 800;
  color: #04632d;
  letter-spacing: -0.02em;
}
.commute-result .result-formula {
  font-size: 0.78rem;
  color: #4b5563;
  margin-left: auto;
}
.commute-result.midnight {
  background: #eef2ff;
  border-color: #c7d2fe;
}
.midnight-badge {
  flex-basis: 100%;
  font-size: 0.78rem;
  color: #4338ca;
  font-weight: 600;
}
.input-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.input-group label {
  font-size: 0.88rem;
  font-weight: 600;
  color: #374151;
}
.input-with-unit {
  position: relative;
  display: flex;
  align-items: center;
}
.input-with-unit input {
  width: 100%;
  padding: 12px 44px 12px 14px;
  border: 1.5px solid #e2e8f0;
  border-radius: 10px;
  font-size: 1.05rem;
  font-weight: 600;
  color: #0f172a;
  background: #f8fafc;
  box-sizing: border-box;
  transition: border-color 0.2s, box-shadow 0.2s;
}
.input-with-unit input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
  background: #fff;
}
.input-with-unit input.error {
  border-color: #ef4444;
  background: #fef2f2;
}
.input-with-unit input.error:focus {
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.15);
}
.input-hint {
  font-size: 0.82rem;
  color: #475569;
  margin: 0;
}
.input-hint code {
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  padding: 1px 6px;
  font-family: 'SF Mono', ui-monospace, Menlo, Consolas, monospace;
  font-size: 0.78rem;
  color: #0f172a;
}
.input-hint .hint-extra {
  color: #94a3b8;
  margin-left: 6px;
}
.input-hint .hint-extra.hint-warn {
  color: #b45309;
  font-weight: 600;
}

/* 연차 패널 — 청록(teal) 톤으로 의무근로 green 카드와 구분 */
.annual-panel {
  display: flex;
  flex-direction: column;
  gap: 11px;
  padding: 13px;
  border-radius: 14px;
  background: linear-gradient(135deg, #ecfeff 0%, #f0f9ff 100%);
  border: 1.5px solid #a5f3fc;
  animation: annual-in 0.32s cubic-bezier(0.22, 1, 0.36, 1) both;
}
@keyframes annual-in {
  from { opacity: 0; transform: translateY(-6px); }
  to { opacity: 1; transform: translateY(0); }
}
@media (prefers-reduced-motion: reduce) {
  .annual-panel { animation: none; }
  .annual-btn:active:not(:disabled) { transform: none; }
}
.annual-hero {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
}
.annual-hero-main {
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.annual-hero-label {
  font-size: 0.74rem;
  font-weight: 700;
  color: #0891b2;
  letter-spacing: -0.01em;
}
.annual-hero-value {
  font-size: 1.55rem;
  font-weight: 800;
  color: #0e7490;
  line-height: 1;
  letter-spacing: -0.03em;
  font-variant-numeric: tabular-nums;
}
.annual-hero-badge {
  display: inline-flex;
  align-items: center;
  padding: 5px 11px;
  border-radius: 999px;
  background: rgba(8, 145, 178, 0.12);
  color: #0e7490;
  font-size: 0.78rem;
  font-weight: 700;
  white-space: nowrap;
  font-variant-numeric: tabular-nums;
}
.annual-hero-empty {
  margin: 0;
  min-height: 32px;
  display: flex;
  align-items: center;
  font-size: 0.82rem;
  font-weight: 600;
  color: #5b829a;
  letter-spacing: -0.01em;
}
.annual-steppers {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}
.annual-stepper {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px;
  background: rgba(255, 255, 255, 0.72);
  border: 1.5px solid #cffafe;
  border-radius: 12px;
  transition: border-color 0.15s, background 0.15s;
}
.annual-stepper.filled {
  border-color: #67e8f9;
  background: #fff;
}
.annual-stepper-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
}
.annual-stepper-name {
  font-size: 0.86rem;
  font-weight: 700;
  color: #164e63;
}
.annual-stepper-hour {
  font-size: 0.75rem;
  font-weight: 800;
  color: #0e7490;
  background: #cffafe;
  padding: 2px 7px;
  border-radius: 6px;
  letter-spacing: 0.02em;
}
.annual-stepper-ctrl {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
}
.annual-btn {
  width: 36px;
  height: 36px;
  flex: none;
  border-radius: 9px;
  border: 1.5px solid #a5f3fc;
  background: #fff;
  color: #0e7490;
  font-size: 1.15rem;
  font-weight: 700;
  line-height: 1;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: background 0.12s, border-color 0.12s, transform 0.08s;
}
.annual-btn:hover:not(:disabled) {
  background: #ecfeff;
  border-color: #22d3ee;
}
.annual-btn:active:not(:disabled) {
  transform: scale(0.92);
}
.annual-btn:focus-visible {
  outline: none;
  border-color: #06b6d4;
  box-shadow: 0 0 0 3px rgba(6, 182, 212, 0.22);
}
.annual-btn:disabled {
  opacity: 0.38;
  cursor: not-allowed;
}
.annual-count {
  font-size: 1.1rem;
  font-weight: 800;
  color: #0f172a;
  min-width: 2.4ch;
  text-align: center;
  font-variant-numeric: tabular-nums;
}
.annual-hint {
  margin: 0;
}

/* 연차 패널 — 다크 */
.theme-dark .annual-panel {
  background: linear-gradient(135deg, #082f49 0%, #0c1b2e 100%);
  border-color: #0e4d6e;
}
.theme-dark .annual-hero-label { color: #38bdf8; }
.theme-dark .annual-hero-value { color: #67e8f9; }
.theme-dark .annual-hero-badge {
  background: rgba(14, 165, 233, 0.2);
  color: #7dd3fc;
}
.theme-dark .annual-hero-empty { color: #7da9c0; }
.theme-dark .annual-stepper {
  background: rgba(255, 255, 255, 0.04);
  border-color: #0e4d6e;
}
.theme-dark .annual-stepper.filled {
  background: rgba(14, 165, 233, 0.1);
  border-color: #0ea5e9;
}
.theme-dark .annual-stepper-name { color: #c9d1d9; }
.theme-dark .annual-stepper-hour {
  background: #0c3a52;
  color: #7dd3fc;
}
.theme-dark .annual-btn {
  background: #161b22;
  border-color: #0e4d6e;
  color: #7dd3fc;
}
.theme-dark .annual-btn:hover:not(:disabled) {
  background: #0d1117;
  border-color: #0ea5e9;
}
.theme-dark .annual-count { color: #f0f6fc; }
.input-error {
  font-size: 0.82rem;
  color: #b91c1c;
  margin: 0;
  font-weight: 500;
}
.input-error code {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 4px;
  padding: 1px 6px;
  font-family: 'SF Mono', ui-monospace, Menlo, Consolas, monospace;
  font-size: 0.78rem;
  color: #991b1b;
}

/* Result section */
.empty-banner {
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 12px;
  padding: 14px 18px;
  margin-bottom: 20px;
  font-size: 0.92rem;
  color: #1e40af;
  line-height: 1.5;
}
.empty-banner strong {
  font-weight: 700;
}
.result-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 20px;
}
.result-item {
  background: #f9fafb;
  border-radius: 14px;
  padding: 18px 16px;
  text-align: left;
  border: 1px solid #f0f1f3;
}
.result-label {
  font-size: 0.78rem;
  font-weight: 600;
  color: #8b95a1;
  margin-bottom: 12px;
  letter-spacing: -0.01em;
}
.result-value {
  font-size: 1.6rem;
  font-weight: 700;
  color: #191f28;
  line-height: 1.1;
  margin-bottom: 8px;
  letter-spacing: -0.02em;
}
.result-value .unit {
  font-size: 0.85rem;
  font-weight: 500;
  margin-left: 2px;
  color: #4e5968;
}
.highlight-blue {
  color: #3182f6;
}
.highlight-green {
  color: #06c755;
}
.highlight-purple {
  color: #6e3eff;
}
.highlight-red {
  color: #f04452;
}
.result-sub {
  font-size: 0.74rem;
  color: #8b95a1;
  line-height: 1.4;
}
.placeholder-dash {
  color: #d1d6db;
  font-weight: 600;
}

/* Average section */
.avg-section {
  margin-top: 4px;
  padding-top: 20px;
  border-top: 1px solid #f0f1f3;
}
.avg-title {
  font-size: 0.86rem;
  font-weight: 700;
  color: #4e5968;
  margin: 0 0 12px;
  letter-spacing: -0.01em;
}
.avg-note {
  font-size: 0.8rem;
  color: #64748b;
  margin: -4px 0 12px;
  line-height: 1.5;
}
.avg-note strong {
  color: #3182f6;
  font-weight: 700;
}
.avg-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}
.avg-card {
  background: #f9fafb;
  border: 1px solid #f0f1f3;
  border-radius: 14px;
  padding: 18px 16px;
  text-align: left;
}
.avg-tag {
  display: inline-block;
  font-size: 0.7rem;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 6px;
  margin-bottom: 10px;
  letter-spacing: 0.02em;
}
.tag-mandatory {
  background: #e6f9f0;
  color: #06873e;
}
.tag-max {
  background: #efe9ff;
  color: #5b2bd6;
}
.avg-value {
  font-size: 1.6rem;
  font-weight: 700;
  color: #191f28;
  line-height: 1.1;
  margin-bottom: 6px;
  letter-spacing: -0.02em;
}
.avg-value .unit {
  font-size: 0.85rem;
  font-weight: 500;
  color: #4e5968;
  margin-left: 2px;
}
.avg-sub {
  font-size: 0.74rem;
  color: #8b95a1;
  line-height: 1.4;
}

/* Notice */
.notice {
  padding: 14px 18px;
  background: #f8fafc;
  border-radius: 10px;
  font-size: 0.9rem;
  color: #475569;
  border: 1px solid #e2e8f0;
}
.past-notice {
  margin-bottom: 20px;
}
.warn-notice {
  padding: 12px 16px;
  background: #fef3c7;
  border: 1px solid #fcd34d;
  border-radius: 10px;
  font-size: 0.88rem;
  color: #92400e;
  margin-bottom: 20px;
  line-height: 1.5;
}

/* Dark mode (theme-dark class) */
.theme-dark .calculator { color: #c9d1d9; }
.theme-dark .calc-header h1 { color: #f0f6fc; }
.theme-dark .subtitle { color: #8b949e; }
.theme-dark .chip--date {
  background: #1a212b;
  border-color: #2b333f;
  color: #c9d1d9;
}
.theme-dark .chip--date .chip-ico {
  background: #0d1117;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.45);
}
.theme-dark .chip--wfh {
  background: #122440;
  border-color: #27477e;
  color: #8cc2ff;
}
.theme-dark .chip--wfh .chip-ico {
  background: #0d1117;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.45);
}
.theme-dark .chip--pay {
  background: linear-gradient(135deg, #6a4d0a 0%, #9a7615 52%, #c2961c 100%);
  border-color: #d3a525;
  color: #fff2c4;
  box-shadow:
    0 4px 18px rgba(194, 150, 28, 0.5),
    inset 0 1px 0 rgba(255, 255, 255, 0.14);
}
.theme-dark .chip--pay .chip-ico {
  background: rgba(255, 255, 255, 0.18);
  box-shadow: none;
}
.theme-dark .theme-fab {
  background: #161b22;
  border-color: #21262d;
  color: #f0f6fc;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.45), 0 2px 4px rgba(0, 0, 0, 0.3);
}
.theme-dark .theme-fab:hover {
  background: #21262d;
  border-color: #30363d;
  box-shadow: 0 10px 24px rgba(0, 0, 0, 0.55), 0 3px 6px rgba(0, 0, 0, 0.35);
}
.theme-dark .label-aside { color: #6e7681; }
.theme-dark .select-group label { color: #8b949e; }
.theme-dark .select-group select {
  background-color: #0d1117;
  border-color: #21262d;
  color: #f0f6fc;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%238b949e' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E");
}
.theme-dark .join-checkbox {
  color: #c9d1d9;
  background: #0d1117;
  border-color: #21262d;
}
.theme-dark .join-checkbox:hover {
  background: #161b22;
  border-color: #30363d;
}
.theme-dark .join-checkbox:has(input:checked) {
  background: #0a2e1c;
  border-color: #2ea44f;
  color: #56d364;
}
.theme-dark .join-inline { border-top-color: #21262d; }
/* 재택근무 체크/카드는 초록(달성) 대신 파랑(재택) 계열로 통일 */
.theme-dark .today-wfh:has(input:checked) {
  background: #122440;
  border-color: #27477e;
  color: #8cc2ff;
}
.theme-dark .wfh-active-card {
  background: #122440;
  border-color: #27477e;
}
.theme-dark .wfh-active-title { color: #8cc2ff; }
.theme-dark .wfh-active-sub { color: #8b949e; }
.theme-dark .join-date-label { color: #c9d1d9; }
.theme-dark .join-date-select {
  background-color: #0d1117;
  border-color: #21262d;
  color: #f0f6fc;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%238b949e' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E");
}
.theme-dark .month-badge { color: #f0f6fc; }
.theme-dark .summary-card.blue {
  background: #0d1f3a;
  border-color: #1f3a68;
}
.theme-dark .summary-card.green {
  background: #0a2e1c;
  border-color: #155f3a;
}
.theme-dark .summary-card.purple {
  background: #1d1638;
  border-color: #3d2c63;
}
.theme-dark .summary-label { color: #c9d1d9; }
.theme-dark .summary-value { color: #f0f6fc; }
.theme-dark .summary-sub { color: #c9d1d9; }
.theme-dark .input-group label { color: #c9d1d9; }
.theme-dark .input-with-unit input {
  background: #0d1117;
  border-color: #21262d;
  color: #f0f6fc;
}
.theme-dark .input-with-unit input:focus { background: #0d1117; }
.theme-dark .input-with-unit input.error {
  border-color: #f85149;
  background: #2d0f0f;
}
.theme-dark .input-hint { color: #c9d1d9; }
.theme-dark .input-hint code {
  background: #161b22;
  border-color: #21262d;
  color: #f0f6fc;
}
.theme-dark .input-hint .hint-extra { color: #8b949e; }
.theme-dark .input-error { color: #ff7b72; }
.theme-dark .input-error code {
  background: #2d0f0f;
  border-color: #6e1414;
  color: #ffa198;
}
.theme-dark .empty-banner {
  background: #0d1f3a;
  border-color: #1f3a68;
  color: #79b8ff;
}
.theme-dark .mode-switch {
  background: #0d1117;
  border-color: #21262d;
}
.theme-dark .mode-switch button { color: #8b949e; }
.theme-dark .mode-switch button.active {
  background: #161b22;
  color: #56d364;
}
.theme-dark .commute-field label { color: #8b949e; }
.theme-dark .dp-wrap :deep(.dp__input) {
  background: #0d1117;
  border-color: #21262d;
  color: #f0f6fc;
}
.theme-dark .now-btn {
  background: #161b22;
  border-color: #21262d;
  color: #c9d1d9;
}
.theme-dark .now-btn:hover {
  background: #0a2e1c;
  border-color: #2ea44f;
  color: #56d364;
}
.theme-dark .break-select {
  background-color: #0d1117;
  border-color: #21262d;
  color: #f0f6fc;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%238b949e' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E");
}
.theme-dark .auto-toggle { color: #c9d1d9; }
.theme-dark .commute-result {
  background: #0a2e1c;
  border-color: #155f3a;
}
.theme-dark .commute-result .result-tag { color: #56d364; }
.theme-dark .commute-result .result-time { color: #f0f6fc; }
.theme-dark .commute-result .result-formula { color: #c9d1d9; }
.theme-dark .commute-result.midnight {
  background: #161335;
  border-color: #3730a3;
}
.theme-dark .midnight-badge { color: #a5b4fc; }
.theme-dark .reflected-summary {
  background: #0d1f3a;
  border-color: #1f3a68;
}
.theme-dark .reflected-label { color: #79b8ff; }
.theme-dark .reflected-value { color: #c9d1ff; }
.theme-dark .reflected-formula { color: #8b949e; }
.theme-dark .result-item {
  background: #0d1117;
  border-color: #21262d;
}
.theme-dark .result-label { color: #8b949e; }
.theme-dark .result-value { color: #f0f6fc; }
.theme-dark .result-value .unit { color: #c9d1d9; }
.theme-dark .result-sub { color: #8b949e; }
.theme-dark .placeholder-dash { color: #484f58; }
.theme-dark .avg-section { border-top-color: #21262d; }
.theme-dark .avg-title { color: #c9d1d9; }
.theme-dark .avg-note { color: #8b949e; }
.theme-dark .avg-note strong { color: #58a6ff; }
.theme-dark .avg-card {
  background: #0d1117;
  border-color: #21262d;
}
.theme-dark .avg-value { color: #f0f6fc; }
.theme-dark .avg-sub { color: #8b949e; }
.theme-dark .tag-mandatory {
  background: #0a2e1c;
  color: #56d364;
}
.theme-dark .tag-max {
  background: #1d1638;
  color: #c4b5fd;
}
.theme-dark .highlight-blue { color: #58a6ff; }
.theme-dark .highlight-green { color: #56d364; }
.theme-dark .highlight-purple { color: #d2a8ff; }
.theme-dark .highlight-red { color: #ff7b72; }
.theme-dark .notice {
  background: #0d1117;
  border-color: #21262d;
  color: #8b949e;
}
.theme-dark .warn-notice {
  background: #2d1f06;
  border-color: #4a3a08;
  color: #fbbf24;
}

/* Responsive */
@media (max-width: 640px) {
  .summary-grid {
    grid-template-columns: 1fr;
  }
  .result-grid {
    grid-template-columns: 1fr;
  }
  .input-grid {
    grid-template-columns: 1fr;
  }
  .commute-grid {
    grid-template-columns: 1fr;
  }
  .commute-result .result-formula {
    margin-left: 0;
    flex-basis: 100%;
  }
  .avg-grid {
    grid-template-columns: 1fr;
  }
  .calc-header h1 {
    font-size: 1.5rem;
  }
  .summary-value {
    font-size: 1.8rem;
  }
  .selector-row {
    gap: 12px;
  }
  .month-badge {
    margin-left: 0;
    width: 100%;
    justify-content: space-between;
    order: 99;
  }
}
</style>
