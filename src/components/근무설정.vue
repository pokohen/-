<script setup>
import { computed } from 'vue'
import { VueDatePicker } from '@vuepic/vue-datepicker'
import '@vuepic/vue-datepicker/dist/main.css'
import { 시분파싱, 시분변환 } from '../utils/시간포맷'

const 고정연장시간 = defineModel('고정연장시간')
const 입력근무시간 = defineModel('입력근무시간')
const 재택근무여부 = defineModel('재택근무여부')
const 재택근무일수 = defineModel('재택근무일수')
const 연차여부 = defineModel('연차여부')
const 연차일수 = defineModel('연차일수')
const 반차수 = defineModel('반차수')
const 반반차수 = defineModel('반반차수')
const 오늘재택근무 = defineModel('오늘재택근무')
const 오늘입력모드 = defineModel('오늘입력모드')
const 출근시각 = defineModel('출근시각')
const 퇴근시각 = defineModel('퇴근시각')
const 휴게수동분 = defineModel('휴게수동분')
const 휴게자동 = defineModel('휴게자동')
const 오늘예상시간 = defineModel('오늘예상시간')

const props = defineProps({
  반영분: Number,
  입력분: Number,
  연차분: Number,
  오늘예상분: Number,
  남은금요일: Number,
  재택일수: Number,
  연차예산분: Number,
  연차잔여분: Number,
  연차일수환산: Number,
  오늘금요일여부: Boolean,
  다크모드: Boolean,
  고정연장유효: Boolean,
  입력유효: Boolean,
  오늘예상유효: Boolean,
  자정넘김여부: Boolean,
  총체류분: Number,
  휴게분: Number,
  지난달여부: Boolean,
})

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

function 지금시각() {
  const 지금 = new Date()
  const 시 = String(지금.getHours()).padStart(2, '0')
  const 분 = String(지금.getMinutes()).padStart(2, '0')
  return `${시}:${분}`
}
function 출근지금() { 출근시각.value = 지금시각() }
function 퇴근지금() { 퇴근시각.value = 지금시각() }

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

// 증감: 잔여 예산(현재까지 근무시간 − 이미 지정한 연차) 안에서만 증가 허용
function 연차증감(필드, 델타) {
  const 단위 = 필드 === '연차' ? 480 : 필드 === '반차' ? 240 : 120
  if (델타 > 0 && props.연차잔여분 < 단위) return
  const 대상 = 필드 === '연차' ? 연차일수 : 필드 === '반차' ? 반차수 : 반반차수
  대상.value = Math.max(0, (Number(대상.value) || 0) + 델타)
}
</script>

<template>
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
</template>

<style scoped>
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

/* Dark mode */
.theme-dark .reflected-summary {
  background: #0d1f3a;
  border-color: #1f3a68;
}
.theme-dark .reflected-label { color: #79b8ff; }
.theme-dark .reflected-value { color: #c9d1ff; }
.theme-dark .reflected-formula { color: #8b949e; }
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

/* Responsive */
@media (max-width: 640px) {
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
}
</style>
