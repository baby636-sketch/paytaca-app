<template>
  <div
    style="background-color: #ECF3F3; min-height: 100vh;padding-top:70px;padding-bottom:50px;"
    :class="{'pt-dark': darkMode}"
  >
    <HeaderNav
      :title="$t('Bridge')"
      backnavpath="/apps"
      style="position: fixed; top: 0; background: #ECF3F3; width: 100%; z-index: 100 !important;"
    />

    <q-icon class="context-menu" size="35px" name="more_vert" :style="{ 'margin-top': $q.platform.is.ios ? '42px' : '0px'}">
      <q-menu>
        <q-list :class="{'pt-dark': darkMode}" style="min-width: 100px">
          <q-item
            :disable="waiting"
            clickable
            v-close-popup
            :class="[darkMode ? 'pt-dark-label' : 'pp-text']"
            :active="mode.active === mode.opts.main"
            @click="mode.active = mode.opts.main"
          >
            <q-item-section>
              {{ $t('BchBridge') }}
            </q-item-section>
          </q-item>
          <q-item
            :disable="waiting"
            clickable
            v-close-popup
            :class="[darkMode ? 'pt-dark-label' : 'pp-text']"
            :active="mode.active === mode.opts.spicebotSlp2Sep20"
            @click="mode.active = mode.opts.spicebotSlp2Sep20"
          >
            <q-item-section>
              {{ $t('SLP_to_SEP20') }}
            </q-item-section>
          </q-item>
        </q-list>
      </q-menu>
    </q-icon>

    <div class="q-px-md" :style="{ 'margin-top': $q.platform.is.ios ? '30px' : '0px'}">
      <template v-if="mode.active === mode.opts.main">
        <HopCashSwapForm v-if="!waiting" @new-incoming="onNewIncoming" :darkMode="darkMode" />
        <HopCashSwapWait v-else v-bind="parsedWaitInfo" :darkMode="darkMode">
          <template v-slot:after="ctx">
            <q-btn
              v-if="ctx && ctx.outgoingTxFound"
              no-caps
              color="brandblue"
              :label="$t('SwapAgain')"
              class="q-mt-md full-width"
              @click="clearWaitInfo()"
            />
          </template>
        </HopCashSwapWait>
      </template>
      <template v-if="mode.active === mode.opts.spicebotSlp2Sep20">
        <SpicebotBridgeForm/>
      </template>
    </div>
  </div>
</template>
<script>
import HeaderNav from '../../components/header-nav'
import HopCashSwapForm from '../../components/bridge/HopCashSwapForm.vue'
import HopCashSwapWait from '../../components/bridge/HopCashSwapWait.vue'
import SpicebotBridgeForm from '../../components/bridge/SpicebotBridgeForm.vue'

export default {
  name: 'Bridge',
  components: { HeaderNav, HopCashSwapForm, HopCashSwapWait, SpicebotBridgeForm },
  data () {
    return {
      mode: {
        // active: 'spicebot-slp-2-sep20',
        active: 'main',
        opts: {
          main: 'main',
          spicebotSlp2Sep20: 'spicebot-slp-2-sep20'
        }
      },
      waitInfo: {
        transferType: 'c2s',
        incomingTxid: '',
        // incomingTxid: '0xb2c6383e9e96a51171658f8b08113fbe54f873136fcd47add2307692e47bc9a5',
        // incomingTxid: '6f4f020193146fb711e64977917753d1f927da2815a6e350ce470c114b123b57',
        amount: '0.02306408',
        expectedAmount: '0.02295788'
      },
      darkMode: this.$store.getters['darkmode/getStatus']
    }
  },
  computed: {
    waiting () {
      return Boolean(this.parsedWaitInfo.incomingTxid)
    },
    parsedWaitInfo () {
      return {
        transferType: this.waitInfo && this.waitInfo.transferType || 'c2s',
        incomingTxid: this.waitInfo && this.waitInfo.incomingTxid || '',
        amount: this.waitInfo && this.waitInfo.amount || '',
        expectedAmount: this.waitInfo && this.waitInfo.expectedAmount || ''
      }
    }
  },
  methods: {
    onNewIncoming (info) {
      let message = this.$t('TransactionSent') + '! '
      if (info && info.transferType === 'c2s') message += this.$t('WaitingSmartBchTransaction')
      if (info && info.transferType === 's2c') message += this.$t('WaitingBchTransaction')
      const dialogStyleClass = this.darkMode ? 'text-white pt-dark-card' : 'text-black'
      this.$q.dialog({
        title: this.$t('SwapUpdate'),
        message: message,
        class: dialogStyleClass
      })
      this.waitInfo = info
    },
    clearWaitInfo () {
      this.waitInfo = {
        transferType: '',
        incomingTxid: '',
        amount: '',
        expectedAmount: ''
      }
    }
  },
  beforeRouteLeave (to, from, next) {
    if (!this.waiting) return next()
    const dialogStyleClass = this.darkMode ? 'text-white pt-dark-card' : 'text-black'
    this.$q.dialog({
      title: this.$t('LeavingPage'),
      message: this.$t('BridgeLeavingPageMsg'),
      cancel: true,
      persistent: true,
      class: dialogStyleClass
    }).onOk(() => {
      next()
    })
  }
}
</script>
<style scoped>
.context-menu {
  position: fixed;
  top: 16px;
  right: 10px;
  z-index: 150 !important;
  color: #3b7bf6;
}
</style>
