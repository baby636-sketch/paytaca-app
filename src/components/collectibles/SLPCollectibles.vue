<template>
  <div :class="[darkMode ? 'pt-dark-label' : 'text-grey-8']">
    <div v-if="fetchingCollectibles" class="row items-center justify-center">
      <ProgressLoader/>
    </div>
    <template v-if="collectibles.length > 0">
      <div v-if="groupedCollectibles.length" class="q-mx-md row justify-end">
        <q-btn-toggle
          flat
          v-model="viewType"
          toggle-color="brandblue"
          :options="[
            {icon: 'view_stream', value: 'list'},
            {icon: 'window', value: 'grid'},
          ]"
        />
      </div>
      <template v-if="groupedCollectibles.length && groupedView">
        <div v-for="group in groupedCollectibles" class="q-mt-md">
          <div
            class="text-h6 row no-wrap items-center q-px-md"
            
            v-ripple style="position:relative;"
            @click="() => toggleExpandGroup(group.token.token_id)"
          >
            <div class="ellipsis q-space" :class="darkMode ? 'text-grad' : 'text-grey-8'">
              <span v-if="group.token.ungrouped_tokens">
                Ungrouped Collectibles
              </span>
              <span v-else >
                {{ group.token.name || `Group #${group.token.token_id}` }}
              </span>
            </div>
            <q-icon
              name="expand_more"
              :class="{'upsidedown': expandedGroupIds.indexOf(group.token.token_id) >= 0}"
              style="transition: 0.25s ease-out all;"
            />
          </div>
          <q-slide-transition>
            <div v-if="expandedGroupIds.indexOf(group.token.token_id) >= 0" class="q-px-sm row items-start">
              <SLPCollectiblesItem
                v-for="collectible in group.collectibles"
                :key="collectible.token_id"
                :collectible="collectible"
                @click="showDetails(collectible)"
              />
            </div>
          </q-slide-transition>
          <q-separator :dark="darkMode" inset/>
        </div>
      </template>
      <div v-else class="q-pa-md row items-start">
        <SLPCollectiblesItem
          v-for="collectible in collectibles"
          :key="collectible.token_id"
          :collectible="collectible"
          @click="showDetails(collectible)"
        />
      </div>
    </template>
    <template v-if="collectibles.length === 0 && !fetchingCollectibles">
      <p style="font-size: 18px; color: gray; text-align: center; margin-top: 50px;" :class="{'pt-dark-label': $store.getters['darkmode/getStatus']}">
        {{ $t('NoCollectibles') }}
      </p>
    </template>
    <Collectible v-model="collectibleDetail.show" :collectible="collectibleDetail.collectible"/>
  </div>
</template>
<script>
import ProgressLoader from 'components/ProgressLoader'
import Collectible from 'components/collectibles/SLPCollectibleDetail'
import SLPCollectiblesItem from 'components/collectibles/SLPCollectiblesItem.vue'

export default {
  name: 'SLPCollectibles',

  components: { ProgressLoader, Collectible, SLPCollectiblesItem },

  props: {
    wallet: {}
  },

  data () {
    return {
      collectibleDetail: {
        show: false,
        collectible: null
      },
      viewType: 'grid', // grid = ungrouped, list = grouped
      expandedGroupIds: [],
      collectibleGroups: [],
      fetchingCollectibles: false,
      collectibles: []
    }
  },

  computed: {
    darkMode() {
      return this.$store.getters['darkmode/getStatus']
    },
    groupedView () {
      return this.viewType === 'list'
    },
    groupedCollectibles() {
      if (!Array.isArray(this.collectibleGroups)) return []
      const filteredGroups = this.collectibleGroups.filter(
        group => group?.token_id || group.ungrouped_tokens
      )
      if (!filteredGroups.length) return []

      const groupTokenIds = filteredGroups.map(group => group?.token_id).filter(Boolean)

      return filteredGroups.map(group => {
        const collectibles = this.collectibles.filter(collectible => {
          const nftTokenGroup = collectible?.nft_token_group
          const hasGroup = Boolean(nftTokenGroup) && groupTokenIds.indexOf(nftTokenGroup) >= 0

          if (group.ungrouped_tokens && !hasGroup) return true
          return nftTokenGroup == group.token_id
        })
        return { token: group, collectibles }
      })
    }
  },

  methods: {
    toggleExpandGroup(groupId) {
      const index = this.expandedGroupIds.indexOf(groupId)
      if (index >= 0) this.expandedGroupIds.splice(index, 1)
      else this.expandedGroupIds.push(groupId)
    },
    getImageUrl (collectible) {
      if (!collectible) return ''
      return collectible.thumbnail_image_url ||
            collectible.thumbnail_image_url ||
            collectible.original_image_url
    },
    showDetails (collectible) {
      this.collectibleDetail.show = true
      this.collectibleDetail.collectible = collectible
    },
    fetchCollectibles () {
      if (!this.wallet) return

      this.fetchingCollectibles = true
      this.wallet.SLP.getCollectibles()
        .finally(() => {
          this.fetchingCollectibles = false
        })
        .then(collectibles => {
          if (Array.isArray(collectibles)) this.collectibles = collectibles
          else this.collectibles = []
        })
    },
    fetchCollectibleGroups() {
      if (!this.wallet) return

      this.wallet.SLP.getCollectibleGroups()
        .then(collectibleGroups => {
          if (Array.isArray(collectibleGroups)) this.collectibleGroups = collectibleGroups
          else this.collectibleGroups = []
        })
    }
  },

  watch: {
    wallet () {
      this.fetchCollectibles()
      this.fetchCollectibleGroups()
    }
  },

  mounted () {
    if (this.wallet) {
      this.fetchCollectibles()
      this.fetchCollectibleGroups()
    }
  }
}
</script>
<style scoped>
.collectible-card {
  width: 100%;
  max-width: 130px;
}
.upsidedown {
  transform: rotate(180deg);
}
</style>
