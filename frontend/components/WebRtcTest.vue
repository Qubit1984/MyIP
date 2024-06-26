<template>
  <!-- WebRTC Test -->
  <div class="webrtc-test-section mb-4">
    <div class="jn-title2">
      <h2 id="WebRTC" :class="{ 'mobile-h2': isMobile }">🚥 {{ t('webrtc.Title') }}</h2>
      <button @click="checkAllWebRTC(true)" :class="['btn', isDarkMode ? 'btn-dark dark-mode-refresh' : 'btn-light']"
        aria-label="Refresh WebRTC Test" v-tooltip="t('Tooltips.RefreshWebRTC')">
        <i class="bi" :class="[isStarted ? 'bi-arrow-clockwise' : 'bi-caret-right-fill']"></i>
      </button>
    </div>
    <div class="text-secondary">
      <p>{{ t('webrtc.Note') }}</p>
    </div>
    <div class="row">
      <div v-for="stun in stunServers" :key="stun.id" class="col-lg-3 col-md-6 col-12 mb-4">
        <div class="card jn-card keyboard-shortcut-card"
          :class="{ 'dark-mode dark-mode-border': isDarkMode, 'jn-hover-card': !isMobile }">
          <div class="card-body">
            <p class="card-title jn-con-title"><i class="bi bi-sign-merge-left-fill"></i> {{ stun.name }}</p>
            <p class="card-text text-secondary" style="font-size: 10pt;"><i class="bi bi-hdd-network-fill"></i> {{
              stun.url }}</p>
            <p class="card-text" :class="{
              'text-info': stun.ip === t('webrtc.StatusWait'),
              'text-success': stun.ip.includes('.') || stun.ip.includes(':'),
              'text-danger': stun.ip === t('webrtc.StatusError')
            }">
              <i class="bi"
                :class="[stun.ip === t('webrtc.StatusWait') ? 'bi-hourglass-split' : 'bi-pc-display-horizontal']">&nbsp;</i>
              <span :class="{ 'jn-ip-font': stun.ip.length > 32 }"> {{ stun.ip }}</span>
            </p>
            <div v-if="stun.natType" class="alert" :class="{
              'alert-info': stun.natType === t('webrtc.StatusWait'),
              'alert-success': stun.natType !== t('webrtc.StatusWait'),
            }" :data-bs-theme="isDarkMode ? 'dark' : ''">
              <i class="bi"
                :class="[stun.natType === t('webrtc.StatusWait') ? 'bi-hourglass-split' : ' bi-controller']"></i> {{
              stun.natType }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, reactive, watch } from 'vue';
import { useMainStore } from '@/store';
import { useI18n } from 'vue-i18n';
import { trackEvent } from '@/utils/use-analytics';

const { t } = useI18n();

const store = useMainStore();
const isDarkMode = computed(() => store.isDarkMode);
const isMobile = computed(() => store.isMobile);


const isStarted = ref(false);
const IPArray = ref([]);
const stunServers = reactive([
  {
    id: "google",
    name: "Google",
    url: "stun.l.google.com:19302",
    ip: t('webrtc.StatusWait'),
    natType: t('webrtc.StatusWait'),
  },
  {
    id: "nextcloud",
    name: "NextCloud",
    url: "stun.nextcloud.com:443",
    ip: t('webrtc.StatusWait'),
    natType: t('webrtc.StatusWait'),
  },
  {
    id: "twilio",
    name: "Twilio",
    url: "global.stun.twilio.com",
    ip: t('webrtc.StatusWait'),
    natType: t('webrtc.StatusWait'),
  },
  {
    id: "cloudflare",
    name: "Cloudflare",
    url: "stun.cloudflare.com",
    ip: t('webrtc.StatusWait'),
    natType: t('webrtc.StatusWait'),
  },
]);


// 测试 STUN 服务器
const checkSTUNServer = async (stun) => {
  try {
    const servers = { iceServers: [{ urls: 'stun:' + stun.url }] };
    const pc = new RTCPeerConnection(servers);
    let candidateReceived = false;

    pc.onicecandidate = (event) => {
      if (event.candidate) {
        candidateReceived = true;
        const candidate = event.candidate.candidate;
        const ipMatch = /([0-9a-f]{1,4}(:[0-9a-f]{1,4}){7}|[0-9a-f]{0,4}(:[0-9a-f]{1,4}){0,6}::[0-9a-f]{0,4}|::[0-9a-f]{1,4}(:[0-9a-f]{1,4}){0,6}|[0-9]{1,3}(\.[0-9]{1,3}){3})/i.exec(candidate);
        if (ipMatch) {
          stun.ip = ipMatch[0];
          IPArray.value = [...IPArray.value, stun.ip];
          stun.natType = determineNATType(candidate);
          pc.close();
        }
      }
    };

    pc.createDataChannel("");
    await pc.createOffer().then((offer) => pc.setLocalDescription(offer));

    // 设置一个超时计时器
    await new Promise((resolve, reject) => {
      setTimeout(() => {
        if (!candidateReceived) {
          reject(new Error("Stun Server Test Timeout"));
        } else {
          resolve();
        }
      }, 5000);
    });
  } catch (error) {
    console.error("STUN Server Test Error:", error);
    stun.ip = t('webrtc.StatusError');
  }
};

// 分析ICE候选信息，推断NAT类型
const determineNATType = (candidate) => {
  const parts = candidate.split(' ');
  const type = parts[7];

  if (type === 'host') {
    return t('webrtc.NATType.host');
  } else if (type === 'srflx') {
    return t('webrtc.NATType.srflx');
  } else if (type === 'prflx') {
    return t('webrtc.NATType.prflx');
  } else if (type === 'relay') {
    return t('webrtc.NATType.relay');
  } else {
    return t('webrtc.NATType.unknown');
  }
};

// 测试所有 STUN 服务器
const checkAllWebRTC = (isRefresh) => {
  stunServers.forEach((server) => {
    server.ip = t('webrtc.StatusWait');
    server.natType = t('webrtc.StatusWait');
    checkSTUNServer(server);
  });
  if (isRefresh) {
    trackEvent('Section', 'RefreshClick', 'WebRTC');
  }
  isStarted.value = true;
};
onMounted(() => {
  store.setLoadingStatus('webrtc', true);
});

watch(IPArray, () => {
  store.updateGlobalIpDataCards(IPArray.value);
}, { deep: true });

defineExpose({
  checkAllWebRTC,
  stunServers
});

</script>

<style scoped></style>
