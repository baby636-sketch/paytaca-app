<template>
  <q-dialog
    ref="dialog"
    @hide="onDialogHide"
    @before-show="function(){
      startPoller()
      updateSwapRequest()
    }"
    full-width
    persistent>
    <q-card :class="darkMode ? 'pt-dark' : 'text-black'">
      <div class="row no-wrap items-center justify-center q-pl-md">
        <div class="text-subtitle1 q-space">Fulfilling swap</div>
        <q-btn
          flat
          padding="sm"
          icon="close"
          v-close-popup
        />
      </div>
      <q-card-section>
        <q-banner rounded :class="'text-white bg-red-4'">
          You can close this dialog and the swap will proceed.
          However you might not be able to see the progress.
        </q-banner>
        <q-item class="q-px-none">
          <q-item-section avatar>
            <img
              height="40"
              :src="tokenImgUrl"
            />
          </q-item-section>
          <q-item-section>
            <q-item-label>
              Fulfulling {{ swapRequest && swapRequest.token && swapRequest.token.name }} swap for
              {{ swapRequest && swapRequest.amount }}
            </q-item-label>
          </q-item-section>
        </q-item>
        <div v-if="waiting" class="row items-center justify-center">
          <ProgressLoader/>
        </div>
        <div v-else-if="fulfillmentTxDetails.hash" class="q-mt-md">
          <div class="ellipsis">
            <a
              :href="`https://sonar.cash/tx/${fulfillmentTxDetails.hash}`"
              target="_blank"
            >
              {{ fulfillmentTxDetails.hash }}
            </a>
          </div>
        </div>
        <div v-else class="text-center">
          <div class="q-my-md" :class="[darkMode ? 'pt-dark-label' : 'pp-text']">Outgoing transaction not found</div>
          <q-btn
            no-caps
            color="brandblue"
            :label="$t('Retry')"
            class="full-width"
            :class="[darkMode ? 'pt-dark-label' : 'pp-text']"
            @click="startPoller()"
          />
        </div>
      </q-card-section>
    </q-card>
  </q-dialog>
</template>
<script>
import { getSwapRequestDetails } from '../../wallet/spicebot-bridge'
import ProgressLoader from '../ProgressLoader.vue'

export default {
  name: 'SpicebotBridgeSwapListenerDialog',
  emits: [
    // REQUIRED
    'ok', 'hide'
  ],
  components: {
    ProgressLoader
  },
  props: {
    swapRequestId: {
      type: Number,
      required: true
    },
    tokenImagesMap: {
      type: Object // Map(<slp_token_id>: <image_url>)
    },
    darkMode: {
      type: Boolean,
      defualt: false
    }
  },
  data () {
    return {
      updatingSwapRequest: false,
      tokenImgUrl: '',
      swapRequest: { id: 0 }, // for watchter reactivity
      pollerId: null,
      pollInterval: 5000
    }
  },
  computed: {
    waiting () {
      return Boolean(this.pollerId)
    },
    fulfillmentTxDetails () {
      return {
        hash: this.swapRequest?.fulfillment?.send_transaction_hash || '',
        success: this.swapRequest?.fulfillment?.success || false,
        dateFulfilled: this.swapRequest?.date_fulfilled || null
      }
    }
  },
  methods: {
    async updateTokenImgUrl () {
      if (this.tokenImagesMap?.[this.swapRequest?.token?.slp_token_id]) {
        this.tokenImgUrl = this.tokenImagesMap?.[this.swapRequest?.token?.slp_token_id]
        return
      }

      if (this.swapRequest?.token?.slp_token_id) {
        try {
          const slpTokenId = this.swapRequest.token.slp_token_id
          const { data: token } = await this.$axios.get(`https://watchtower.cash/api/tokens/${slpTokenId}/`)
          if (token?.image_url) {
            this.tokenImgUrl = token?.image_url
            return
          }
        } catch (err) { console.error(err) }
      }

      if (this.swapRequest?.token?.sep20_contract) {
        try {
          const sep20ContractAddress = this.swapRequest.token.sep20_contract
          const { data: token } = await this.$axios.get(`https://watchtower.cash/api/smartbch/token-contracts/${sep20ContractAddress}/`)
          if (token?.image_url) {
            this.tokenImgUrl = token.image_url
            return
          }
        } catch (err) { console.error(err) }
      }
    },
    stopPoller () {
      clearInterval(this.pollerId)
      this.pollerId = null
    },
    startPoller () {
      this.stopPoller()

      this.pollerId = setInterval(() => {
        this.updateSwapRequest(false)
      }, this.pollInterval)
    },
    updateSwapRequest (silent = false) {
      if (!this.swapRequestId) return

      this.updatingSwapRequest = Boolean(silent)
      getSwapRequestDetails(this.swapRequestId)
        .then(response => {
          if (response && response.swapRequest) {
            this.swapRequest = response.swapRequest
            if (this.swapRequest?.date_fulfilled) this.stopPoller()
          }
        })
        .finally(() => {
          this.updatingSwapRequest = false
        })
    },
    // following method is REQUIRED
    // (don't change its name --> "show")
    show () {
      this.$refs.dialog.show()
    },

    // following method is REQUIRED
    // (don't change its name --> "hide")
    hide () {
      this.$refs.dialog.hide()
    },

    onDialogHide () {
      this.stopPoller()
      // required to be emitted
      // when QDialog emits "hide" event
      if (this.fulfillmentTxDetails.hash) this.$emit('ok')

      this.$emit('hide')
    },

    onOKClick () {
      // on OK, it is REQUIRED to
      // emit "ok" event (with optional payload)
      // before hiding the QDialog
      this.$emit('ok')
      // or with payload: this.$emit('ok', { ... })

      // then hiding dialog
      this.hide()
    },

    onCancelClick () {
      // we just need to hide dialog
      this.hide()
    }
  },
  watch: {
    'swapRequest.id': {
      handler () {
        this.updateTokenImgUrl()
      }
    }
  }
}
</script>
