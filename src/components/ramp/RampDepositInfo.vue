<template>
  <div
    class="br-15 q-pt-sm q-mx-md"
    :class="[ darkMode ? 'text-white' : 'text-black',]"
    v-if="isloaded"
  >
    <div>
      <q-btn
        flat
        padding="md"
        icon="arrow_back"
        @click="$emit('retry')"
      />
    </div>
    <div v-if="!sendBCH">
      <div v-if="!shiftExpired">
        <div class="text-center justify-center text-h5" style="font-size:20px;">
          Please send exactly <br><b style="letter-spacing: 1px;">{{ parseFloat(shiftInfo.shift_info.deposit.amount) }} {{ shiftInfo.shift_info.deposit.coin }} ({{ getNetwork(shiftInfo.ramp_type, shiftInfo.shift_info.deposit.network) }})</b> to...
        </div>

        <div class="row q-pt-md">
          <div class="col qr-code-container">
            <div class="col col-qr-code q-pl-sm q-pr-sm q-pt-md">
              <div class="row text-center">
                <div class="col row justify-center q-pt-md" @click="copyToClipboard(shiftInfo.shift_info.deposit.address)">
                  <div style="height: 60px; width: 60px; border-radius: 50%;" v-html="shiftInfo.shift_info.deposit.icon" class="receive-icon-asset"></div>
                  <qr-code :text="depositAddress" color="#253933" :size="200" error-level="H" class="q-mb-sm"></qr-code>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="row">
          <div class="col" style="padding: 20px 40px 0px 40px; overflow-wrap: break-word;"  @click="copyToClipboard(shiftInfo.shift_info.deposit.address)">
            <span class="qr-code-text text-weight-light text-center">
              <div style="letter-spacing: 1px" :class="darkMode ? 'text-white' : 'pp-text'">
                {{ shiftInfo.shift_info.deposit.address }}
                <p style="font-size: 12px; margin-top: 7px;">{{ $t('ClickToCopyAddress') }}</p>
              </div>
            </span>
          </div>
        </div>
        <div class="text-center q-pt-md text-h2 text-blue-5">
          {{ countDown }}
        </div>
        <div class="text-center q-pt-md">
          <span style="font-size:13px;">Send amount before the timer ends...</span>
        </div>
      </div>
      <div class="text-center" v-if="shiftExpired">
        <div class="q-pt-md text-h2 text-red-5 q-py-lg">
          Expired
        </div>
        <div class="q-pt-lg">
          <q-btn color="blue-9" label="Try Again" @click="$emit('retry')"></q-btn>
        </div>
      </div>
    </div>
    <div v-if="sendBCH">
      <div v-if="processing">
        <div class="text-center text-h5 q-px-lg" style="margin-top: 100px; font-size: 20px; overflow-wrap: break-word;">
          Sending <b>{{ shiftInfo.shift_info.deposit.amount }}</b> BCH to <b>{{ shiftInfo.shift_info.settle.address }}</b>
        </div>
        <div class="row justify-center q-py-lg">
          <ProgressLoader/>
        </div>
      </div>
      <div v-if="!sendFailed && !processing">
        <div class="text-center text-h5 q-px-lg" style="margin-top: 100px; font-size: 20px; overflow-wrap: break-word;">
          <b>{{ shiftInfo.shift_info.deposit.amount }} BCH</b> Sent!
        </div>
        <div class="q-pt-lg text-center">
          <q-btn color="blue-9" label="Back" @click="$emit('done')"></q-btn>
        </div>
      </div>
      <div v-if="sendFailed && !processing">
        <div class="text-center text-h5 q-px-lg" style="margin-top: 100px; font-size: 20px; overflow-wrap: break-word;">
          Sorry, failed to send BCH...
        </div>
        <div class="q-pt-lg text-center">
          <q-btn color="blue-9" label="Try Again" @click="$emit('retry')"></q-btn>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import ProgressLoader from '../ProgressLoader.vue'
import { getMnemonic, Wallet } from '../../wallet'
import { getMemoedVNodeCall } from '@vue/compiler-core'
import { getNetwork } from '@ethersproject/networks'

export default {
  data () {
    return {
      darkMode: this.$store.getters['darkmode/getStatus'],
      shiftInfo: {},
      countDown: '',
      shiftExpired: false,
      sendBCH: false,
      processing: false,
      sendFailed: false,
      depositAddress: '',
      state: '',
      baseUrl: process.env.ANYHEDGE_BACKEND_BASE_URL,
      error: false,
      isloaded: false
    }
  },
  props: {
    shiftData: Object,
    refundAddress: {
      type: String,
      default: ''
    },
    type: {
      type: String,
      default: 'created'
    }
  },
  emits: ['retry', 'done'],
  components: {
    ProgressLoader
  },
  methods: {
    copyToClipboard (value) {
      this.$copyText(value)
      this.$q.notify({
        message: this.$t('CopiedToClipboard'),
        timeout: 800,
        color: 'blue-9',
        icon: 'mdi-clipboard-check'
      })
    },
    getNetwork (coin, network) {
      const token = { coin: coin, network: network }
      return this.$parent.getNetwork(token).toLowerCase()
    },
    countingDown () {
      const vm = this

      const expire = vm.shiftInfo.shift_info.shift_expiration
      const expireDate = new Date(expire).getTime()

      const x = setInterval(function () {
        const now = new Date().getTime()
        // find distance
        const distance = expireDate - now

        const minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60))
        let seconds = Math.floor((distance % (1000 * 60)) / 1000)
        if (seconds.toString().length < 2) {
          seconds = '0' + seconds
        }

        vm.countDown = minutes + ':' + seconds

        if (distance < 0) {
          clearInterval(x)
          vm.countDown = 'Expired'
          vm.shiftExpired = true
        }
      }, 1000)
    },
    async sendingBCH () {
      const vm = this
      vm.processing = true
      vm.sendFailed = false
      const mnemonic = await getMnemonic()
      const wallet = new Wallet(mnemonic)

      const amount = parseFloat(vm.shiftInfo.shift_info.deposit.amount)

      await wallet.BCH.sendBch(amount, vm.shiftInfo.shift_info.deposit.address).then(function (result, err) {
        if (result.success) {
          console.log('success')
        } else {
          vm.sendFailed = true
          vm.expireShift()
        }
      }).catch((error) => {
        console.log(error)
        vm.sendFailed = true
        vm.expireShift()
      })
      vm.processing = false
    },
    async expireShift () {
      const vm = this
      const shiftId = vm.shiftData.id

      const response = await vm.$axios.post(
        vm.baseUrl + '/ramp/expire',
        {
          shift_id: shiftId
        }
      ).catch(function () {
        vm.error = true
      })
    }
  },
  async mounted () {
    const vm = this

    vm.shiftInfo = vm.shiftData

    vm.depositAddress = vm.shiftInfo.shift_info.deposit.address
    vm.state = vm.type
    if (vm.state === 'created') {
      if (vm.shiftInfo.shift_info.deposit.coin === 'BCH' && vm.refundAddress === vm.$store.getters['global/getAddress']('bch')) {
        // console.log('this wallet')
        vm.sendBCH = true
        await vm.sendingBCH()
      } else {
        // console.log('others')
        vm.countingDown()
      }
    } else if (vm.state === 'history') {
      if (this.shiftData.status === 'expired') {
        vm.shiftExpired = true
      } else {
        vm.countingDown()
      }
    }

    vm.isloaded = true
  }
}
</script>
<style lang="scss" scoped>
 .address-text {
    font-size: 18px;
    color: #000;
 }
 .qr-code-container {
    padding-left: 28px;
    padding-right: 28px;
  }
  .col-qr-code {
    margin-left: auto;
    margin-right: auto;
    text-align: center;
    width: 300px;
    border-radius: 16px;
    border: 4px solid #ed5f59;
    padding: 25px 10px 32px 10px;
    background: white;
  }
  .qr-code-text {
    font-size: 18px;
    color: #000;
  }
  .receive-icon-asset {
    position: absolute;
    margin-top: 73px;
    background: white;
    border-radius: 50%;
    padding: 4px;
  }
  .text-subtitle1 {
  font-size: 14px;
}
.text-nowrap {
  white-space: nowrap;
}
.pp-text {
  color: #000 !important;
}
.pt-internet-required {
  text-align: center;
  width: 100%;
  font-size: 24px;
  padding: 30px;
  color: gray;
}
</style>

