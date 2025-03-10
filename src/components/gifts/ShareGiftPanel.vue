<template>
  <slot v-bind="{ copyToClipboard, giftShareUrl, shareLinks }">
    <div
      class="q-py-sm q-px-md q-px-lg q-my-xs row items-center no-wrap rounded-borders"
      style="border:1px solid grey; position:relative"
      v-ripple
      @click="() => copyToClipboard(giftShareUrl)"
    >
      <div style="overflow-y:auto;white-space:nowrap;">{{ giftShareUrl }}</div>
      <q-icon name="content_copy" size="1.25em" class="q-ml-sm"/>
    </div>
    <div class="q-mt-sm q-pb-sm row no-wrap q-gutter-x-sm" style="overflow-x:auto;">
      <template v-for="shareLink in shareLinks">
        <q-btn
          flat
          rounded
          padding="md"
          size="md"
          :icon="shareLink.icon"
          :href="shareLink.url"
          target="_blank"
        />
      </template>
    </div>
  </slot>
</template>
<script setup>
import { computed, inject } from 'vue'
import { useStore } from 'vuex'
import { useQuasar } from 'quasar'

const $copyText = inject('$copyText')
const $q = useQuasar()
const $store = useStore()
const darkMode = computed(() => $store.getters['darkmode/getStatus'])

const props = defineProps({
  qrShare: String,
  amount: Number,
})
const giftShareUrl = computed(() => {
  if (!props.qrShare) return ''
  return `https://gifts.paytaca.com/claim/?code=${props.qrShare}`
})

const shareLinks = computed(() => {
  const encodedUrl = encodeURI(giftShareUrl.value)
  const fbAppId = 438643061338284 // using a dev app, might have to replace
  const data = {
    fb: { icon: 'fab fa-facebook', url: `https://www.facebook.com/dialog/share?app_id=${fbAppId}&href=${encodedUrl}&display=popup` },
    messenger: { icon: 'fab fa-facebook-messenger', url: `https://facebook.com/share/?url=${encodedUrl}`, },
    twitter: { icon: 'fab fa-twitter', url: `https://twitter.com/intent/tweet?url=${encodedUrl}&text=Get ${props.amount} BCH Gift&via=Paytaca&related=Paytaca` },
    telegram: { icon: 'telegram', url: `https://t.me/share?url=${encodedUrl}&text=Paytaca Gift`, },
    whatsapp: { icon: 'fab fa-whatsapp', url: `https://wa.me/?text=${props.amount} BCH from Paytaca Gifts\n${encodedUrl}`, },
    email: { icon: 'email', url: `mailto:?body=${props.amount} BCH from Paytaca Gifts: ${encodedUrl}` },
  }

  if ($q.platform.is.mobile) {  
    data.telegram.url = `tg://msg_url?url=${encodedUrl}&text=${props.amount} BCH from Paytaca Gifts`
    data.messenger = {
      icon: 'fab fa-facebook-messenger',
      url: `fb-messenger://share/?link=${encodedUrl}&app_id=${fbAppId}`
    }
  } else {
    delete data.messenger
  }
  return data
})

function copyToClipboard (value, message='Copied to clipboard') {
  $copyText(value)
  $q.notify({
    message: message,
    timeout: 800,
    color: 'blue-9',
    icon: 'mdi-clipboard-check'
  })
}
</script>