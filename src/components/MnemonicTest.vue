<template>
  <div>
    <div
      class="bg-white q-pa-sm rounded-borders q-ma-sm"
      style="min-height:10rem;"
    >
      <q-btn
        v-if="rearrangedSeedphrase.length"
        icon="clear"
        round
        size="sm"
        text-color="black"
        class="float-right"
        @click="rearrangedSeedphrase = []"
      />
      <div class="row items-start q-gutter-sm">
        <q-btn
          v-for="(word, index) in rearrangedSeedphrase"
          :key="word"
          no-caps
          :label="`${index+1}. ${word}`"
          padding="xs sm"
          color="brandblue"
          rounded
          @click="toggleRearrangedSeedphraseWord(word)"
        />
      </div>
    </div>
    <div class="text-grey text-center">
      {{ $t('VerifyMnemonicBackupPhrase') }}
    </div>
    <div class="q-mx-sm q-mt-sm">
      <div class="row justify-around q-gutter-sm">
        <q-btn
          v-for="word in shuffledSeedphrase"
          :key="word"
          :color="rearrangedSeedphrase.indexOf(word) >= 0 ? 'grey' : 'brandblue'"
          rounded
          no-caps
          :label="word"
          padding="xs sm"
          color="brandblue"
          @click="toggleRearrangedSeedphraseWord(word)"
        />
      </div>
    </div>
  </div>  
</template>
<script setup>
import { computed, ref, onMounted, watch } from 'vue'
const $emit = defineEmits(['matched'])
const props = defineProps({
  mnemonic: String,
})

const seedPhrase = computed(() => props.mnemonic.trim().split(' '))

const shuffledSeedphrase = ref([])
const rearrangedSeedphrase = ref([])
onMounted(() => shuffleSeedphrase())
function shuffleSeedphrase() {
  shuffledSeedphrase.value = [...seedPhrase.value]
  shuffleArray(shuffledSeedphrase.value)
  rearrangedSeedphrase.value = rearrangedSeedphrase.value.filter(word => shuffledSeedphrase.value.indexOf(word) < 0)
}
/**
 * Fisher-Yates shuffle
 * @param {String[]} array 
 * @link https://stackoverflow.com/questions/2450954/how-to-randomize-shuffle-a-javascript-array
 */
function shuffleArray(array) {
  let currentIndex = array.length,  randomIndex;
  // While there remain elements to shuffle.
  while (currentIndex != 0) {

    // Pick a remaining element.
    randomIndex = Math.floor(Math.random() * currentIndex);
    currentIndex--;

    // And swap it with the current element.
    [array[currentIndex], array[randomIndex]] = [
      array[randomIndex], array[currentIndex]];
  }

  return array;
}

function toggleRearrangedSeedphraseWord(word='') {
  if (rearrangedSeedphrase.value.indexOf(word) >= 0) {
    rearrangedSeedphrase.value = rearrangedSeedphrase.value.filter(w => w !== word)
  } else {
    rearrangedSeedphrase.value.push(word)
  }
}

const matched = computed(() => {
  if (rearrangedSeedphrase.value.length !== seedPhrase.value.length) return false

  for (var i = 0; i < seedPhrase.value.length; i++) {
    if (seedPhrase.value[i] !== rearrangedSeedphrase.value[i]) return false
  }
  return true
})
watch(matched, () => matched.value ? $emit('matched') : null)
</script>
