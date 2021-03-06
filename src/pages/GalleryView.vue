<template>
  <div class="gallery-view">
    <div class="photo-list-outer">
      <div v-show="!isContentLoaded" class="loading">
        LOADING...
        <loading-bar :colors="appColors"></loading-bar>
      </div>
      <photo-list v-show="true" ref="list"
        :metas="imgMetas"
        @changed="itemChanged"
        @click="bookSlideshow"
        ></photo-list>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.gallery-view {
  margin: 20px;
  padding: 0px;
  box-sizing: border-box;
  overflow: visible;
}
.photo-list-outer {
  position: absolute;
  box-sizing: border-box;
  width: calc(100% - 40px);
  height: calc(100% - 20px);
}
.loading {
  height: 120px;
  padding-top: 50px;
}
</style>

<script>
import LoadingBar from '@/components/LoadingBar'
import PhotoList from '@/components/PhotoList'
import Time from '@/core/Time'
import MathUtil from '@/core/MathUtil'
const SLIDESHOW_ACTIVATE_SEC = 40
const SLIDESHOW_INTERVAL_SEC = 15

export default {
  name: 'GalleryView',
  components: { LoadingBar, PhotoList },
  data () {
    return {
      /** current selected item id */
      selId: null,
      /** selId that specified with URL when mounted. Keep until imgMetas are loaded */
      initialSelId: null,
      /* slide show ... */
      slideshowActivateTimerId: null,
      isSlideshowMode: false
    }
  },
  computed: {
    imgMetas () {
      return this.$store.state.gallery.imgMetas
    },
    isContentLoaded () {
      return this.imgMetas && this.imgMetas.length > 0
    },
    appColors () {
      return [...this.$store.state.common.colors, '#006987', '#020751', '#acd', '#99aacc', '#f8d899']
    }
  },
  watch: {
    '$route' (to, from) {
      const newSelId = this.$route.params.filename
      if (this.selId !== newSelId) {
        this.$refs.list.selectItemWithId(newSelId)
      }
    },
    async imgMetas (val) {
      // this watch will be fired only once when DB documents are loaded.
      if (this.initialSelId) {
        // if the item to select is specified (see this.mounted), select that after item list is constructed.
        await Time.wait(1)
        // select item without smooth-scrolling to avoid animation delay in page loading phase
        this.$refs.list.selectItemWithId(this.initialSelId, false)
        this.initialSelId = null
      }
    }
  },
  mounted () {
    // keep gallery item specified in url onto local data object
    // NOTE: gallery items metadata (=imgMetas) may not be loaded yet.
    this.initialSelId = this.$route.params.filename || null
    // init slideshow auto activation
    document.addEventListener('visibilitychange', this.onvisibilitychange, false)
    this.onvisibilitychange()
  },
  destroyed () {
    window.clearTimeout(this.slideshowActivateTimerId)
    this.suspendSlideshow()
    document.removeEventListener('visibilitychange', this.onvisibilitychange, false)
  },
  methods: {
    async itemChanged (id, item) {
      this.selId = id
      await Time.wait(200)
      if (this.selId !== id) { return }
      if (item) {
        const colors = item.meta.colors.colorset.colors
          .map(rgb => `rgb(${rgb.map(v => Math.round(v)).join(',')})`)
        this.$store.commit('setAppColors', { colors })
      } else {
        this.$store.commit('setAppColors', { colors: null })
      }
      this.$router.replace(`/gallery/${item ? item.id : ''}`)
    },
    async runSlideshow () {
      console.log('Slideshow start')
      const LIST_DISPLAY_SEC = 2
      this.isSlideshowMode = true
      while (this.isSlideshowMode) {
        const crId = this.selId
        const list = this.$refs.list
        list.selectItemWithId(null)
        list.scrollToItem(crId)
        await Time.wait(LIST_DISPLAY_SEC * 1000)
        const meta = this.imgMetas[MathUtil.intBetween(0, this.imgMetas.length - 1)]
        list.selectItemWithId(meta.filename)
        await Time.wait(SLIDESHOW_INTERVAL_SEC * 1000)
      }
      console.log('Slideshow ended')
    },
    bookSlideshow () {
      // Quit (or cancel booked) slideshow (if running)
      this.suspendSlideshow()
      // Book slideshow activation after SLIDESHOW_ACTIVATE_SEC sec
      this.slideshowActivateTimerId = window.setTimeout(() => {
        this.runSlideshow()
      }, SLIDESHOW_ACTIVATE_SEC * 1000)
    },
    suspendSlideshow () {
      // If runing ... quit
      // If booked ... cancel
      // to resume, call bookSlideshow method.
      this.isSlideshowMode = false
      window.clearTimeout(this.slideshowActivateTimerId)
    },
    onvisibilitychange () {
      if (document.webkitHidden) {
        console.log('Detect app deactivated: suspen slideshow')
        this.suspendSlideshow()
      } else {
        console.log('Detect app activated: book next slideshow')
        this.bookSlideshow()
      }
    }
  }
}
</script>
