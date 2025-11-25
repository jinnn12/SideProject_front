<template>
  <div id="app">
    <transition name="fade">
      <div v-if="toast.visible" :class="['toast', toast.type]">
        {{ toast.message }}
      </div>
    </transition>

    <header class="app-header">
      <img class="brand-logo" src="@/assets/OYSOAP logopng2.png" alt="OYSOAP 로고" />
      <p v-if="!showStatsSection">
        QR 스캔으로 굴 패각 비누의 친환경 기여도를<br>
        확인하세요
      </p>
      <p v-else>
        나의 환경 기여도를<br>
        확인해보세요
      </p>
    </header>

    <!-- 스캔 결과 화면 -->
    <div v-if="showResultSection">
      <section class="card card-with-back celebration-card">
        <a href="#" class="back-link light" @click.prevent="showResultSection = false">← 뒤로가기</a>
        <div class="celebration-top no-icon">
          <div class="celebration-text">
            <p class="celebration-label">스캔 완료</p>
            <p class="celebration-message">
              <span>{{ scanPrimaryMessage }}</span>
              <span class="celebration-subtext">{{ scanSecondaryMessage }}</span>
            </p>
          </div>
        </div>
        <div class="celebration-badge">
          <strong>{{ celebrationBadgeText }}</strong>
          <span>{{ celebrationSubtitle }}</span>
        </div>
      </section>

      <section v-if="scanStats" class="card stats-card">
        <h2>나의 기여 현황</h2>
        <div class="stats-grid">
          <div class="stat-item soap">
            <div class="stat-icon">🧼</div>
            <span class="label">기여한 비누</span>
            <strong class="stat-value">{{ scanStats.usedQrCount }}개</strong>
          </div>
          <div class="stat-item seedling">
            <div class="stat-icon">🌱</div>
            <span class="label">나의 묘목</span>
            <strong class="stat-value">{{ scanStats.totalSeedlingTrees.toFixed(2) }} 그루</strong>
          </div>
        </div>
        
        <!-- 전체 목표 진행률 게이지 -->
        <div class="progress-section">
          <h3>전체 목표 진행률</h3>
          <div class="progress-bar-container">
            <div class="progress-bar">
              <div 
                class="progress-fill" 
                :style="{ width: scanProgressPercentage + '%' }"
              ></div>
            </div>
            <div class="progress-text">
              <span>{{ scanStats.totalSeedlingAllUsers.toFixed(2) }} / {{ (scanStats.targetSeedlingTrees || 3600).toFixed(0) }} 그루</span>
              <span class="progress-percentage">{{ scanProgressPercentage.toFixed(1) }}%</span>
            </div>
          </div>
        </div>
      </section>
    </div>

    <!-- QR 스캔 화면 -->
    <div v-else-if="!showStatsSection">
      <section class="card">
        <form @submit.prevent="handleScan">
          <label>
            이메일
            <input
              v-model.trim="scanForm.email"
              type="email"
              placeholder="user@example.com"
              required
            />
          </label>

          <!-- QR 코드 자동 입력 (URL 파라미터에서 읽기) -->
          <div v-if="qrCodeFromUrl && scanForm.qrCode" class="qr-display-section">
            <label>
              QR 코드
              <input
                v-model.trim="scanForm.qrCode"
                type="text"
                readonly
                style="background: #f8fafc; cursor: not-allowed;"
              />
            </label>
            <button type="button" class="clear-qr-btn" @click="scanForm.qrCode = ''; qrCodeFromUrl = false;">변경</button>
          </div>

          <!-- 시리얼 번호 수동 입력 (QR 인식 실패 시 또는 URL에서 온 게 아닐 때) -->
          <div v-else class="qr-input-section">
            <label>
              시리얼 번호
              <input
                v-model.trim="scanForm.qrCode"
                type="text"
                placeholder="시리얼 번호 (예: OY00001-A3B2)"
              />
            </label>
            <p class="input-hint">QR 인식이 안 될 경우 시리얼 번호를 직접 입력하세요</p>
          </div>

          <div class="consent-section">
            <label class="checkbox-label">
              <input
                :checked="scanForm.consentPrivacy"
                type="checkbox"
                required
                @click.prevent="showPrivacyModal = true"
                readonly
              />
              <span @click.prevent="showPrivacyModal = true" style="cursor: pointer;">개인정보보호 동의 <span class="required">(필수)</span></span>
            </label>
            <label class="checkbox-label" @click.prevent="handleMarketingConsentClick">
              <input
                :checked="scanForm.consentMarketing"
                type="checkbox"
                readonly
                tabindex="-1"
              />
              <span style="cursor: pointer;">광고성 수신이벤트 동의 <span class="optional">(선택)</span></span>
            </label>
          </div>

          <button type="submit" :disabled="loading.scan">
            {{ loading.scan ? '처리 중...' : '스캔 등록' }}
          </button>
        </form>
        <p v-if="messages.scan && !showResultSection" :class="{ error: scanError }">{{ messages.scan }}</p>
        
        <p class="stats-link">
          <a href="#" @click.prevent="showStatsSection = true" class="toggle-link">기여도 조회 →</a>
        </p>
      </section>
    </div>

    <!-- 기여도 조회 화면 -->
    <div v-else>
      <section class="card card-with-back">
        <a href="#" class="back-link" @click.prevent="showStatsSection = false">← 뒤로가기</a>
        <h2>기여도 조회</h2>
        <form @submit.prevent="handleStats">
          <label>
            이메일
            <input
              v-model.trim="statsEmail"
              type="email"
              placeholder="user@example.com"
              required
            />
          </label>
          <button type="submit" :disabled="loading.stats">
            {{ loading.stats ? '불러오는 중...' : '기여도 조회' }}
          </button>
        </form>
        <p v-if="messages.stats" :class="{ error: statsError }">{{ messages.stats }}</p>
      </section>

      <section v-if="stats && stats.usedQrCount > 0" class="card stats-card">
        <h2>나의 기여 현황</h2>
        <div class="stats-grid">
          <div class="stat-item soap">
            <div class="stat-icon">🧼</div>
            <span class="label">기여한 비누</span>
            <strong class="stat-value">{{ stats.usedQrCount }}개</strong>
          </div>
          <div class="stat-item seedling">
            <div class="stat-icon">🌱</div>
            <span class="label">나의 묘목</span>
            <strong class="stat-value">{{ stats.totalSeedlingTrees.toFixed(2) }} 그루</strong>
          </div>
        </div>
        
        <!-- 전체 목표 진행률 게이지 -->
        <div class="progress-section">
          <h3>전체 목표 진행률</h3>
          <div class="progress-bar-container">
            <div class="progress-bar">
              <div 
                class="progress-fill" 
                :style="{ width: progressPercentage + '%' }"
              ></div>
            </div>
            <div class="progress-text">
              <span>{{ stats.totalSeedlingAllUsers.toFixed(2) }} / {{ (stats.targetSeedlingTrees || 3600).toFixed(0) }} 그루</span>
              <span class="progress-percentage">{{ progressPercentage.toFixed(1) }}%</span>
            </div>
          </div>
        </div>
      </section>

      <section v-else-if="stats && stats.usedQrCount === 0" class="card empty-stats-card">
        <div class="empty-state-icon">🌱</div>
        <h3>아직 등록된 기여가 없습니다</h3>
        <p>첫 QR을 스캔하면 나만의 묘목이 자라기 시작해요.</p>
        <button type="button" class="secondary-btn" @click="showStatsSection = false">스캔하러 가기</button>
      </section>
    </div>
  </div>

  <!-- 광고성 수신 이벤트 동의 모달 -->
  <div v-if="showMarketingModal" class="modal-overlay" @click="showMarketingModal = false">
    <div class="modal-content" @click.stop>
      <div class="modal-header">
        <h3>광고성 정보 수신 동의 (선택)</h3>
        <button class="modal-close" @click="showMarketingModal = false">×</button>
      </div>
      <div class="modal-body">
        <p>이메일을 통해 아래의 내용을 받아보실 수 있습니다.</p>
        <div class="privacy-info">
          <p>• 신규 제품 출시 소식</p>
          <p>• 이벤트 및 할인 안내</p>
          <p>• 환경 기여 활동 관련 소식</p>
        </div>
        <p class="privacy-note">동의를 하지 않으셔도 기본 서비스 이용에 제한은 없습니다.</p>
      </div>
      <div class="modal-footer">
        <button class="modal-button" @click="confirmMarketingConsent">동의합니다</button>
        <button class="modal-button secondary" @click="showMarketingModal = false">취소</button>
      </div>
    </div>
  </div>

  <!-- 개인정보 처리 동의 모달 -->
  <div v-if="showPrivacyModal" class="modal-overlay" @click="showPrivacyModal = false">
    <div class="modal-content" @click.stop>
      <div class="modal-header">
        <h3>개인정보 수집·이용 동의 (필수)</h3>
        <button class="modal-close" @click="showPrivacyModal = false">×</button>
      </div>
      <div class="modal-body">
        <p>저희는 서비스 제공을 위해 아래의 개인정보를 수집·이용합니다.</p>
        <div class="privacy-info">
          <p><strong>수집 항목:</strong> 이메일 주소</p>
          <p><strong>이용 목적:</strong> 제품 이용 기록 관리, QR 참여 내역 조회</p>
          <p><strong>보유 기간:</strong> 목적 달성 후 즉시 파기 또는 관련 법령에 따른 보관 기간 준수</p>
        </div>
        <p class="privacy-note">위 내용에 동의하셔야 서비스 이용이 가능합니다.</p>
      </div>
      <div class="modal-footer">
        <button class="modal-button" @click="confirmPrivacyConsent">동의합니다</button>
        <button class="modal-button secondary" @click="showPrivacyModal = false">취소</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      apiBase: '/api/user',
      scanForm: {
        email: '',
        qrCode: '',
        consentPrivacy: false,
        consentMarketing: false
      },
      statsEmail: '',
      stats: null,
      scanStats: null,
      showStatsSection: false,
      showResultSection: false,
      showPrivacyModal: false,
      showMarketingModal: false,
      qrCodeFromUrl: false, // URL 파라미터에서 온 QR 코드인지 여부
      loading: {
        scan: false,
        stats: false
      },
      messages: {
        scan: '',
        stats: ''
      },
      scanError: false,
      statsError: false,
      toast: {
        visible: false,
        message: '',
        type: 'info',
        timeoutId: null
      }
    }
  },
  mounted() {
    // URL 파라미터에서 QR 코드 읽기
    const urlParams = new URLSearchParams(window.location.search)
    const qrParam = urlParams.get('qr')
    if (qrParam) {
      this.scanForm.qrCode = qrParam.trim()
      this.qrCodeFromUrl = true // URL에서 온 QR 코드임을 표시
    }
  },
  computed: {
    progressPercentage() {
      return this.calculateProgress(this.stats)
    },
    scanProgressPercentage() {
      return this.calculateProgress(this.scanStats)
    },
    celebrationBadgeText() {
      if (!this.scanStats || typeof this.scanStats.totalSeedlingTrees !== 'number') {
        return '친환경 기여 +0.09'
      }
      return `묘목 ${this.scanStats.totalSeedlingTrees.toFixed(2)} 그루`
    },
    celebrationSubtitle() {
      if (this.scanStats && typeof this.scanStats.usedQrCount === 'number') {
        return `총 ${this.scanStats.usedQrCount}회 참여 중`
      }
      return '지속 가능한 바다를 함께 만들고 있어요'
    },
    scanPrimaryMessage() {
      if (this.scanStats && this.scanStats.email) {
        return `${this.scanStats.email}님, 오늘도 한 번 더 지구에 손을 보태주셨습니다.`
      }
      return '오늘도 한 번 더 지구에 손을 보태주셨습니다.'
    },
    scanSecondaryMessage() {
      if (this.scanStats && typeof this.scanStats.usedQrCount === 'number') {
        return `총 ${this.scanStats.usedQrCount}번의 작은 숲이 쌓이고 있습니다.`
      }
      return '작은 숲이 차곡차곡 쌓이고 있습니다.'
    }
  },
  methods: {
    calculateProgress(source) {
      if (!source || !source.targetSeedlingTrees || !source.totalSeedlingAllUsers) {
        return 0
      }
      const percentage = (source.totalSeedlingAllUsers / source.targetSeedlingTrees) * 100
      return Math.min(percentage, 100)
    },
    showToast(message, type = 'info') {
      if (this.toast.timeoutId) {
        clearTimeout(this.toast.timeoutId)
      }
      this.toast.message = message
      this.toast.type = type
      this.toast.visible = true
      this.toast.timeoutId = setTimeout(() => {
        this.toast.visible = false
        this.toast.timeoutId = null
      }, 3500)
    },
    async handleScan() {
      if (!this.scanForm.consentPrivacy) {
        const message = '개인정보보호 동의(필수)를 완료해주세요.'
        this.scanError = true
        this.messages.scan = message
        this.showPrivacyModal = true
        this.showToast(message, 'error')
        return
      }
      this.loading.scan = true
      this.messages.scan = ''
      this.scanError = false
      let res = null
      try {
        res = await fetch(`${this.apiBase}/qr/scan`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(this.scanForm)
        })
        if (!res.ok) {
          // Rate Limiting 에러 처리 (429 Too Many Requests)
          if (res.status === 429) {
            const errorMessage = '요청이 너무 많습니다. 1분에 15회까지만 가능합니다. 잠시 후 다시 시도해주세요.'
            this.scanError = true
            this.messages.scan = errorMessage
            this.showToast(errorMessage, 'error')
            return
          }
          const message = await this.extractErrorMessage(res)
          throw new Error(message)
        }
        const data = await res.json()
        this.messages.scan = `${data.email}님의 스캔이 반영되었습니다. 총 ${data.usedQrCount}회 참여 중!`
        this.scanForm.qrCode = ''
        this.scanStats = data
        this.showResultSection = true // 결과 화면으로 이동
        this.showToast('스캔이 정상 처리되었습니다.', 'success')
      } catch (err) {
        this.scanError = true
        this.messages.scan = err.message || '알 수 없는 오류가 발생했습니다.'
        this.showToast(this.messages.scan, 'error')
      } finally {
        this.loading.scan = false
      }
    },
    async handleStats() {
      this.loading.stats = true
      this.messages.stats = ''
      this.statsError = false
      this.stats = null
      try {
        const res = await fetch(`${this.apiBase}/stats?email=${encodeURIComponent(this.statsEmail)}`)
        if (!res.ok) {
          const message = await this.extractErrorMessage(res)
          throw new Error(message)
        }
        const data = await res.json()
        this.stats = data
        if (data.usedQrCount > 0) {
          this.messages.stats = `${data.email}님의 기여도를 불러왔습니다.`
          this.showToast('기여도 정보를 불러왔습니다.', 'success')
        } else {
          this.messages.stats = `${data.email}님의 참여 내역이 아직 없습니다.`
          this.showToast('아직 등록된 기여가 없습니다.', 'info')
        }
      } catch (err) {
        this.statsError = true
        this.messages.stats = err.message || '알 수 없는 오류가 발생했습니다.'
        this.showToast(this.messages.stats, 'error')
      } finally {
        this.loading.stats = false
      }
    },
    async extractErrorMessage(res) {
      try {
        const data = await res.clone().json()
        if (data && data.message) {
          return data.message
        }
        if (data && data.error) {
          return data.error
        }
      } catch (jsonErr) {
        // fallthrough to text
      }
      const text = await res.text()
      if (text) {
        try {
          const parsed = JSON.parse(text)
          return parsed.message || parsed.error || '알 수 없는 오류가 발생했습니다.'
        } catch (parseErr) {
          return text.length > 120 ? `${text.slice(0, 117)}...` : text
        }
      }
      return '서버와 통신 중 문제가 발생했습니다.'
    },
    confirmPrivacyConsent() {
      this.scanForm.consentPrivacy = true
      this.showPrivacyModal = false
    },
    handleMarketingConsentClick() {
      // 이미 체크된 상태면 해제
      if (this.scanForm.consentMarketing) {
        this.scanForm.consentMarketing = false
      } else {
        // 체크 안 된 상태면 모달 표시
        this.showMarketingModal = true
      }
    },
    confirmMarketingConsent() {
      this.scanForm.consentMarketing = true
      this.showMarketingModal = false
    }
  }
}

</script>

<style>
@import url('https://cdn.jsdelivr.net/npm/pretendard@1.3.9/dist/web/static/pretendard.css');

:root {
  font-family: 'Pretendard', 'Noto Sans KR', 'Segoe UI', system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
  background-color: #f7f5ef;
  color: #103b2c;
}

#app {
  max-width: 640px;
  margin: 0 auto;
  padding: 1.5rem 1rem 3rem;
}

.app-header {
  text-align: center;
  margin-bottom: 1.5rem;
}

.brand-logo {
  width: 180px;
  height: 180px;
  object-fit: contain;
  margin-bottom: 1rem;
}

.card {
  background: #fff;
  border-radius: 1rem;
  padding: 1.25rem;
  margin-bottom: 1.25rem;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.12);
}

.celebration-card {
  background: linear-gradient(140deg, #f1fff4 0%, #e0f4ff 55%, #fef6f0 100%);
  border: 1px solid rgba(255, 255, 255, 0.6);
  overflow: hidden;
  position: relative;
  animation: cardGlow 4s ease-in-out infinite;
}

.celebration-card::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: radial-gradient(circle at 20% 20%, rgba(255, 255, 255, 0.6), transparent 60%), radial-gradient(circle at 80% 10%, rgba(255, 255, 255, 0.4), transparent 55%);
  pointer-events: none;
  opacity: 0.6;
  mix-blend-mode: screen;
}

.celebration-card::before {
  content: '';
  position: absolute;
  inset: -40%;
  background: conic-gradient(from 0deg, rgba(34, 197, 94, 0.2), rgba(59, 130, 246, 0.25), rgba(249, 115, 22, 0.2), rgba(34, 197, 94, 0.2));
  animation: auroraSpin 12s linear infinite;
  filter: blur(60px);
  opacity: 0.6;
  pointer-events: none;
}

.celebration-top {
  display: flex;
  align-items: center;
  gap: 1rem;
  position: relative;
  z-index: 1;
}

.celebration-top.no-icon {
  align-items: flex-start;
  justify-content: center;
  text-align: center;
}

.celebration-icon {
  width: 64px;
  height: 64px;
  border-radius: 20px;
  background: linear-gradient(160deg, #22c55e, #15803d);
  color: #fff;
  font-size: 2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 12px 25px rgba(34, 197, 94, 0.35);
  animation: floatIcon 3.5s ease-in-out infinite;
}

.celebration-text {
  flex: 1;
  color: #0f2f21;
}

.celebration-label {
  font-size: 1rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  margin-bottom: 0.25rem;
  color: #0f766e;
}

.celebration-message {
  font-size: 1.1rem;
  font-weight: 600;
  line-height: 1.45;
  color: #032915;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  text-align: center;
}

.celebration-subtext {
  font-size: 0.95rem;
  font-weight: 500;
  color: #065f46;
}

.celebration-badge {
  margin-top: 1.25rem;
  padding: 0.85rem 1rem;
  border-radius: 0.85rem;
  background: rgba(255, 255, 255, 0.8);
  box-shadow: inset 0 1px 4px rgba(15, 23, 42, 0.08);
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  position: relative;
  z-index: 1;
  animation: badgeGlow 3s ease-in-out infinite;
}

.celebration-badge strong {
  font-size: 1.05rem;
  color: #0f172a;
}

.celebration-badge span {
  font-size: 0.9rem;
  color: #0f766e;
}

@keyframes auroraSpin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

@keyframes floatIcon {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-6px);
  }
}

@keyframes badgeGlow {
  0%,
  100% {
    box-shadow: inset 0 1px 4px rgba(15, 23, 42, 0.08), 0 0 0 rgba(34, 197, 94, 0);
  }
  50% {
    box-shadow: inset 0 1px 4px rgba(15, 23, 42, 0.12), 0 0 18px rgba(34, 197, 94, 0.45);
  }
}

@keyframes cardGlow {
  0%,
  100% {
    box-shadow: 0 15px 40px rgba(15, 23, 42, 0.15);
  }
  50% {
    box-shadow: 0 20px 55px rgba(15, 23, 42, 0.25);
  }
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1rem;
}

.form-block h2 {
  font-size: 1.1rem;
  margin-bottom: 0.75rem;
}

form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 1rem;
}

label {
  display: flex;
  flex-direction: column;
  font-weight: 600;
  gap: 0.35rem;
}

input {
  border: 1px solid #cbd5f5;
  border-radius: 0.6rem;
  padding: 0.65rem 0.8rem;
  font-size: 1rem;
}

button {
  background: linear-gradient(120deg, #22c55e, #16a34a);
  color: #fff;
  border: none;
  border-radius: 0.6rem;
  padding: 0.75rem;
  font-size: 1rem;
  cursor: pointer;
  transition: opacity 0.2s ease;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 1rem;
}

.empty-stats-card {
  text-align: center;
  padding: 2rem 1.5rem;
  background: linear-gradient(135deg, #f8fafc 0%, #eef2ff 100%);
  border: 1px dashed #cbd5f5;
}

.empty-state-icon {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
}

.secondary-btn {
  background: transparent;
  color: #0f766e;
  border: 1px solid #0f766e;
  margin-top: 1rem;
}

.secondary-btn:hover {
  background: rgba(15, 118, 110, 0.08);
}

.stat-item {
  background: #f8fafc;
  border-radius: 0.75rem;
  padding: 1.25rem 1rem;
  text-align: center;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  position: relative;
  overflow: hidden;
}

.stat-item:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 12px rgba(16, 59, 44, 0.15);
}

.stat-item.seedling {
  background: linear-gradient(135deg, #f0fdf4 0%, #dcfce7 100%);
  border: 2px solid #86efac;
}

.progress-section {
  margin-top: 2rem;
  padding-top: 2rem;
  border-top: 2px solid #e2e8f0;
}

.progress-section h3 {
  font-size: 1.1rem;
  color: #103b2c;
  margin-bottom: 1rem;
  text-align: center;
}

.progress-bar-container {
  width: 100%;
}

.progress-bar {
  width: 100%;
  height: 30px;
  background-color: #e2e8f0;
  border-radius: 15px;
  overflow: hidden;
  position: relative;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #22c55e 0%, #16a34a 100%);
  border-radius: 15px;
  transition: width 0.5s ease;
  box-shadow: 0 2px 4px rgba(34, 197, 94, 0.3);
}

.progress-text {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.75rem;
  font-size: 0.9rem;
  color: #64748b;
}

.progress-percentage {
  font-weight: 700;
  color: #103b2c;
  font-size: 1rem;
}

.stat-item.soap {
  background: linear-gradient(135deg, #fef3c7 0%, #fde68a 100%);
  border: 2px solid #fbbf24;
}

.stat-icon {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  display: inline-block;
  animation: float 3s ease-in-out infinite;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.1));
}

.stat-item.seedling .stat-icon {
  animation: grow 2s ease-in-out infinite;
}


.stat-item.soap .stat-icon {
  animation: bubble 2.5s ease-in-out infinite;
}

@keyframes bubble {
  0%, 100% {
    transform: scale(1) rotate(0deg);
  }
  25% {
    transform: scale(1.05) rotate(-5deg);
  }
  75% {
    transform: scale(1.05) rotate(5deg);
  }
}

@keyframes float {
  0%, 100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-8px);
  }
}

@keyframes grow {
  0%, 100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
}


.stat-value {
  display: block;
  font-size: 1.25rem;
  color: #103b2c;
  margin-top: 0.5rem;
  font-weight: 700;
}

.label {
  display: block;
  font-size: 0.85rem;
  color: #64748b;
  margin-bottom: 0.35rem;
}

.error {
  color: #dc2626;
}

.placeholder {
  color: #94a3b8;
  font-size: 0.95rem;
  text-align: center;
}

.stats-link {
  margin-top: 1rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

.stats-link a {
  color: #64748b;
  text-decoration: none;
  font-weight: 600;
  margin-left: 0.25rem;
  transition: color 0.2s ease;
}

.stats-link a:hover {
  color: #103b2c;
  text-decoration: underline;
}

.toggle-link {
  color: #64748b;
  text-decoration: none;
  font-weight: 600;
  transition: color 0.2s ease;
}

.toggle-link:hover {
  color: #103b2c;
  text-decoration: underline;
}

.separator {
  margin: 0 0.5rem;
  color: #cbd5e1;
}

.qr-input-section {
  margin-top: 0.5rem;
  margin-bottom: 1rem;
}

.input-hint {
  font-size: 0.85rem;
  color: #64748b;
  margin-top: 0.5rem;
  margin-bottom: 0;
}

.consent-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 0.6rem;
  border: 1px solid #e2e8f0;
}

.checkbox-label {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 0.5rem;
  font-weight: 500;
  cursor: pointer;
  font-size: 0.9rem;
}

.checkbox-label input[type="checkbox"] {
  width: 1.1rem;
  height: 1.1rem;
  cursor: pointer;
  margin: 0;
  flex-shrink: 0;
}

.checkbox-label span {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.required {
  color: #dc2626;
  font-size: 0.85rem;
}

.optional {
  color: #64748b;
  font-size: 0.85rem;
}

.card-with-back {
  position: relative;
}

.card-with-back h2 {
  margin-top: 0;
  padding-right: 6rem;
}

.back-link {
  position: absolute;
  top: 1.25rem;
  right: 1.25rem;
  color: #64748b;
  text-decoration: none;
  font-size: 0.95rem;
  font-weight: 600;
  transition: color 0.2s ease;
}

.back-link:hover {
  color: #103b2c;
  text-decoration: underline;
}

.back-link.light {
  color: rgba(255, 255, 255, 0.9);
  z-index: 2;
}

.back-link.light:hover {
  color: #ffffff;
}

.toast {
  position: fixed;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  padding: 0.85rem 1.25rem;
  border-radius: 999px;
  color: #fff;
  z-index: 50;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.toast.success {
  background: #15803d;
}

.toast.error {
  background: #dc2626;
}

/* 모달 스타일 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 1rem;
}

.modal-content {
  background: #fff;
  border-radius: 1rem;
  max-width: 500px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: modalSlideIn 0.3s ease;
}

@keyframes modalSlideIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-header h3 {
  margin: 0;
  font-size: 1.25rem;
  color: #103b2c;
}

.modal-close {
  background: none;
  border: none;
  font-size: 2rem;
  color: #64748b;
  cursor: pointer;
  padding: 0;
  width: 2rem;
  height: 2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
  transition: color 0.2s ease;
}

.modal-close:hover {
  color: #103b2c;
}

.modal-body {
  padding: 1.5rem;
  color: #334155;
  line-height: 1.6;
}

.privacy-info {
  background: #f8fafc;
  border-radius: 0.6rem;
  padding: 1rem;
  margin: 1rem 0;
  border: 1px solid #e2e8f0;
}

.privacy-info p {
  margin: 0.75rem 0;
}

.privacy-info p:first-child {
  margin-top: 0;
}

.privacy-info p:last-child {
  margin-bottom: 0;
}

.privacy-note {
  margin-top: 1rem;
  font-size: 0.9rem;
  color: #64748b;
  font-weight: 500;
}

.modal-footer {
  display: flex;
  gap: 0.75rem;
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  justify-content: flex-end;
}

.modal-button {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 0.6rem;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s ease;
  background: linear-gradient(120deg, #22c55e, #16a34a);
  color: #fff;
}

.modal-button:hover {
  opacity: 0.9;
}

.modal-button.secondary {
  background: #e2e8f0;
  color: #334155;
}

.modal-button.secondary:hover {
  background: #cbd5e1;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translate(-50%, -10px);
}

@media (max-width: 480px) {
  #app {
    padding: 1rem 0.75rem 2rem;
  }

  .card {
    border-radius: 0.85rem;
    padding: 1rem;
  }

  button {
    font-size: 0.95rem;
  }

  .stats-grid {
    grid-template-columns: repeat(2, minmax(120px, 1fr));
    gap: 0.75rem;
  }
}
</style>
