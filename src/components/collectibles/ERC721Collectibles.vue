<template>
  <div>
    <div v-if="fetchingCollectibles" class="row items-center justify-center">
      <ProgressLoader/>
    </div>
    <template v-if="collectibles.length > 0">
      <div class="q-pa-md row items-start">
        <q-card
          v-for="(collectible, index) in collectibles"
          :key="index"
          class="collectible-card q-ma-sm"
          @click="showDetails(collectible)"
        >
          <q-img
            :src="getImageUrl(collectible)"
            :alt="`#${collectible.id}`"
            fit="fill"
          />
        </q-card>
      </div>
      <div class="row q-mt-sm items-center justify-center">
        <q-pagination
          v-if="
            pageNumberPaginationData && pageNumberPaginationData.pageCount > 1
          "
          :modelValue="pageNumberPaginationData.currentPage"
          :max="pageNumberPaginationData.pageCount"
          :max-pages="6"
          :dark="$store.getters['darkmode/getStatus']"
          unelevated
          padding="xs sm"
          input
          boundary-numbers
          @update:modelValue="
            (val) =>
              fetchCollectibles({
                limit: pageNumberPaginationData.pageSize,
                offset: pageNumberPaginationData.pageSize * (val - 1)
              })
          "
        />
      </div>
    </template>
    <template v-if="collectibles.length === 0 && !fetchingCollectibles">
      <p style="font-size: 18px; color: gray; text-align: center;" class="q-py-md" :class="{'pt-dark-label': $store.getters['darkmode/getStatus']}">
        {{ $t('NoCollectibles') }}
      </p>
    </template>
    <ERC721CollectibleDetail v-model="collectibleDetail.show" :darkMode="$store.getters['darkmode/getStatus']" :collectible="collectibleDetail.collectible"/>
  </div>
</template>
<script>
import ERC721CollectibleDetail from 'components/collectibles/ERC721CollectibleDetail.vue'
import ProgressLoader from 'components/ProgressLoader'

export default {
  name: 'ERC721Collectibles',

  components: { ERC721CollectibleDetail, ProgressLoader },

  props: {
    wallet: { },
    contractAddress: {
      type: String,
      default: ''
    }
  },

  data () {
    return {
      collectibleDetail: {
        show: false,
        collectible: null
      },
      fetchingCollectibles: false,
      collectibles: [],
      pagination: {
        limit: 10,
        offset: 0,
        count: 0
      }
    }
  },

  computed: {
    pageNumberPaginationData () {
      if (
        !this.pagination ||
        !Number.isSafeInteger(this.pagination.count) ||
        !Number.isSafeInteger(this.pagination.limit) ||
        !Number.isSafeInteger(this.pagination.offset)
      ) return

      return {
        pageCount: Math.ceil(this.pagination.count / this.pagination.limit),
        currentPage: Math.ceil(this.pagination.offset / this.pagination.limit) + 1,
        pageSize: this.pagination.limit
      }
    }
  },

  methods: {
    getImageUrl (collectible) {
      if (!collectible || !collectible.metadata) return ''
      return collectible.metadata.image || ''
    },

    showDetails (collectible) {
      this.collectibleDetail.show = true
      this.collectibleDetail.collectible = collectible
    },

    fetchCollectibles (opts = {}) {
      if (!this.wallet) return
      const _opts = opts || {}
      if (!Number.isSafeInteger(_opts.limit)) _opts.limit = 6
      _opts.includeMetadata = true
      _opts.asyncMetadata = true
      _opts.metadataCallback = (/* collectible, index */) => {
        this.$forceUpdate()
      }
      this.fetchingCollectibles = true
      // this.wallet.sBCH.getNFTs(this.contractAddress, _opts)
      const address = this.$store.getters['global/getAddress']('sbch')
      this.wallet.sBCH.getOwnedNFTs(this.contractAddress, address, _opts)
        .finally(() => {
          this.fetchingCollectibles = false
        })
        .then(response => {
          if (Array.isArray(response && response.tokens)) this.collectibles = response.tokens
          else this.collectibles = []
          if (response.pagination) {
            this.pagination.limit = response.pagination.limit
            this.pagination.offset = response.pagination.offset
            this.pagination.count = response.pagination.count
          }
        })
    }
  },

  watch: {
    wallet () {
      this.fetchCollectibles()
    }
  },

  mounted () {
    this.fetchCollectibles()
  }
}
</script>
<style scoped>
.collectible-card {
  width: 100%;
  max-width: 130px;
}
</style>
