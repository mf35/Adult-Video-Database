<template>
  <vuescroll ref="mainContainer" @handle-scroll="handleScroll">
    <div class="website-header-wrapper">
      <div class="all-thumbs-website">
        <v-img v-for="(imgUrl, i) in videoThumbImgUrls" :key="i" 
          :src="imgUrl" :aspect-ratio="16/9" width="20%" />
      </div>
      <div class="website-header" >
        <div class="gradient" :style="gradient"></div>
        <div class="background" :style="headerColor"></div>
        <div class="website-logo">
          <v-img :src="getImgUrl(websiteId)" max-width="300" class="ma-3 logo"/>
        </div>
        <div class="website-info">
          <div class="text-h2 pa-4 website-name"> {{website.name}} 
            <span class="text-h4">{{website.network?'(network)':''}}</span>
            <span v-if="isChildWebsite" class="body-2">is a child of
              <span @click="$router.push(websiteLink(isChildWebsite))" 
                @click.middle="addNewTabWebsite(isChildWebsite)" class="network-title">
                {{isChildWebsite}}</span> network
            </span>
          </div>
          
          <div v-if="website.network && website.childWebsites.length" class="child-websites px-4">
            <div class="mb-2">Child websites</div>
            <div class="grid mb-4">
              <v-card v-for="child in website.childWebsites" :key="child" class="child-card" hover 
                :to="websiteLink(child)" @click.middle="addNewTabWebsite(child)">
                <v-img :src="getImgUrl(getWebsiteId(child))" aspect-ratio="1.5"/>
                <v-card-title>{{ child }}</v-card-title>
              </v-card>
            </div>
          </div>
          <div class="px-4 performers-title"> Performers of website 
            <v-btn v-if="activePerformers.length" @click="showAllPerformers" 
              x-small rounded class="mx-4">
              <v-icon left>mdi-eye-outline</v-icon> Show all
            </v-btn>
          </div>
          <v-chip-group v-model="activePerformers" active-class="active-chip" 
            multiple column class="px-4 pt-2 chips-performers">
            <v-chip v-for="performer in performersOfWebsite" :key="performer"
              outlined small class="mr-1 mb-1 px-2"
              @mouseover.stop="showImage($event, getPerformerId(performer), 'performer')" 
              @mouseleave.stop="$store.state.hoveredImage=false"
              @click="$store.state.hoveredImage=false"
              @click.middle="addNewTabPerformer(performer)"
            > {{ performer }} </v-chip>
          </v-chip-group>
        </div>
      </div>
    </div>

    <v-container fluid
      v-if="!$store.state.Videos.filteredEmpty" 
      class="pagination-container my-6" 
    >
      <v-overflow-btn v-model="videosPerPage" hint="items per page" persistent-hint
        :items="videosPerPagePreset" dense height="36" solo disable-lookup hide-no-data
        class="items-per-page-dropdown"
      ></v-overflow-btn>
      <v-spacer></v-spacer>
      <v-pagination 
        v-model="videosCurrentPage"
        :length="videosPagesSum"
        :total-visible="getNumberOfPagesLimit"
      ></v-pagination>
      <v-spacer></v-spacer>
      <v-overflow-btn v-if="videosPagesSum > 5"
        v-model="videosCurrentPage" :items="pages" dense height="36" solo
        class="items-per-page-dropdown jump-to-page-menu" 
        disable-lookup hint="jump to page" persistent-hint hide-no-data
        :menu-props="{ 
          auto:true, 
          contentClass:'jump-to-page-menu',
          nudgeBottom: -110,
          origin:'center center', 
          transition:'scale-transition'
        }"
      ></v-overflow-btn>
      <div v-else style="min-width:80px;"></div>
    </v-container>

    <div v-if="$store.state.Videos.filteredEmpty" class="text-center pt-10">
      <div><v-icon size="100" class="ma-10">mdi-close</v-icon></div>
      There are no matching videos for the selected filters.
    </div>

    <Loading />

    <v-container fluid class="videos-grid" :class="[cardSize, gapSize]">
      <!-- Video Blocks parsing -->
      <VideoCard v-for="(video) in videosFromWebsite" :key="video.id" :video="video"/>
    </v-container>

    <v-pagination class="pt-10 pb-16"
      v-if="!$store.state.Videos.filteredEmpty"
      v-model="videosCurrentPage"
      :length="videosPagesSum"
      :total-visible="getNumberOfPagesLimit"
    ></v-pagination>
    
    <div v-show="$store.state.Settings.navigationSide=='2'" class="py-6"></div>
    
    <v-btn @click="scrollToTop" v-show="isScrollToTopVisible" 
      class="scroll-to-top" fixed fab color="primary">
      <v-icon>mdi-chevron-up</v-icon>
    </v-btn>

    <DialogEditWebsite v-if="$store.state.Websites.dialogEditWebsite"/>
  </vuescroll>
</template>


<script>
const fs = require("fs")
const path = require("path")
import VideosGrid from '@/mixins/VideosGrid'
import CropImage from '@/mixins/CropImage'
import vuescroll from 'vuescroll'
import ShowImageFunction from '@/mixins/ShowImageFunction'
import LabelFunctions from '@/mixins/LabelFunctions'

export default {
  name: 'WebsitePage',
  components: {
    vuescroll,
    DialogEditWebsite: () => import("@/components/pages/websites/DialogEditWebsite.vue"),
    Loading: () => import('@/components/elements/Loading.vue'),
  },
  mixins: [VideosGrid, ShowImageFunction, LabelFunctions],
  mounted() {
    this.$nextTick(function () {
      this.initFilters()
    })
  },
  data: () => ({
    activePerformers: [],
    isScrollToTopVisible: false,
  }),
  computed: {
    website() {
      return this.$store.getters.websites.find({ id: this.websiteId }).value()    
    },
    websiteId() {
      return this.$route.params.id.replace(/:/g, '')    
    },
    videosFromWebsite: {
      get() {
        return this.$store.getters.videosOnPage
      },
      set(value) {
        // this.$store.dispatch('rearrangeVideos', filteredVideos)
      },
    },
    cardSize() {
      return `card-size-${this.$store.state.Settings.videoCardSize}`
    },
    headerColor() {
      return `background: ${this.website.color}`
    },
    gradient() {
      let color = this.$vuetify.theme.isDark ? '#121212' : '#fff'
      let gradient = `linear-gradient(to top, ${color}, rgba(0,0,0,.0) 35%)`
      return `background: ${gradient}`
    },
    performersOfWebsite() {
      let videos = this.videosOfWebsite.value()
      let performers = []
      for (let i=0; i<videos.length; i++) {
        for (let p=0; p<videos[i].performers.length; p++) {
          performers.push(videos[i].performers[p])
        }
      }
      let performersUnique = [...new Set(performers)]
      performersUnique = performersUnique.slice(0,100)
      return performersUnique.sort((a, b) => a.localeCompare(b))
    },
    videosOfWebsite() {
      return this.$store.getters.videos.filter(video=>
        (video.website == this.website.name)
      )
    },
    videoThumbImgUrls() {
      let imgUrls = this.videosOfWebsite.orderBy('rating',['desc']).take(40).value().map(v=>{
        let imgPath = path.join(this.pathToUserData, `/media/thumbs/${v.id}.jpg`)
        if (fs.existsSync(imgPath)) {
          return imgPath
        } else return false
      })
      return imgUrls
    },
    pathToUserData() {
      return this.$store.getters.getPathToUserData
    },
    tabId() {
      return this.$route.query.tabId
    },
    filtersTab() {
      if (this.tabId === 'default') {
        return undefined
      } else {
        return this.$store.getters.tabsDb.find({id:this.tabId}).value().filters    
      }
    },
    gapSize() {
      return `gap-size-${this.$store.state.Settings.gapSize}`
    },
    isChildWebsite() {
      let network = this.$store.getters.websites.filter(web=>
          (web.network&&web.childWebsites.includes(this.website.name))).value()
      if (network.length) {
        network = network[0].name
      } else network = ''
      return network
    },
  },
  methods: {
    scrollToTop() {
      this.$refs.mainContainer.scrollTo({y: 0},500,"easeInQuad")
    },
    handleScroll(vertical) {
      if (vertical.scrollTop > 500) {
        this.isScrollToTopVisible = true
      } else this.isScrollToTopVisible = false
    },
    showAllPerformers() {
      this.activePerformers = []
    },
    initFilters() {
      if (this.tabId === 'default' || typeof this.filtersTab === 'undefined') {
        this.$store.state.Settings.videoFilters = [{
          param: 'websites',
          cond: 'all',
          val: [this.website.name],
          type: 'array',
          lock: true,
        },{
          param: 'performers',
          cond: 'one of',
          val: this.getFilteredPerformers(),
          type: 'array',
          lock: true,
        }]
        this.$store.dispatch('filterVideos', true)
      } else {
        this.$store.state.Settings.videoFilters = _.cloneDeep(this.filtersTab)
        this.$store.dispatch('filterVideos')
      }
    },
    updateFiltersOfWebsitesTab() {
      if (this.tabId !== 'default') {
        let newFilters = _.cloneDeep(this.$store.state.Settings.videoFilters)
        this.$store.getters.tabsDb.find({id:this.tabId}).assign({filters:newFilters}).write()
      }
    },
    getImgUrl(websiteId) {
      let imgPath = path.join(this.pathToUserData, `/media/websites/${websiteId}_.jpg`)
      return this.checkImageExist(imgPath)
    },
    checkImageExist(imgPath) {
      if (fs.existsSync(imgPath)) {
        return imgPath
      } else {
        return path.join(this.pathToUserData, '/img/templates/website.png')
      }
    },
    getWebsite(itemName) {
      return this.$store.getters.websites.find({name: itemName}).value()
    },
    getFilteredPerformers() {
      let active = this.activePerformers
      let all = this.performersOfWebsite
      let filtered = [] 
      for (let i=0; i<active.length; i++) {
        filtered.push(all[active[i]])
      }
      return filtered
    },
    updateFilters() {
      const defaults = [{
        param: 'websites',
        cond: 'all',
        val: [this.website.name],
        type: 'array',
        lock: true,
      },{
        param: 'performers',
        cond: 'one of',
        val: this.getFilteredPerformers(),
        type: 'array',
        lock: true,
      }]
      const others = _.filter(this.$store.state.Settings.videoFilters, {lock: false})
      this.$store.state.Settings.videoFilters = [...defaults, ...others]
      this.$store.dispatch('filterVideos')
      this.updateFiltersOfWebsitesTab()
    },
  },
  watch: {
    activePerformers() {
      this.updateFilters()
    },
    $route(newRoute) {
      if (!this.$route.path.includes('/website/:')) return
      this.initFilters()
    },
  }
}
</script>


<style lang="less" scoped>
.child-websites {
  .child-card {
    .v-card__title {
      padding: 0 5px;
      font-size: 10px;
      line-height: 2;
    }
  }
  .grid {
    display: grid;
    grid-gap: 8px;
    grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  }
}
.website-header {
  display: flex;
  align-items: flex-start;
  padding-bottom: 100px;
  z-index: 2;
  &-wrapper {
    position: relative;
    overflow: hidden;
  }
  .website-name {
    z-index: 2;
    font-weight: bold;
  }
  .logo,
  .chips-performers,
  .performers-title {
    z-index: 2;
  }
  .website-info {
    display: flex;
    flex-direction: column;
    z-index: 2;
  }
  .background {
    position: absolute;
    opacity: 0.75;
    top: 0;
    bottom: 0;
    right: 0;
    left: 0;
    pointer-events: none;
  }
  .gradient {
    position: absolute;
    top: 0;
    bottom: 0;
    right: 0;
    left: 0;
    z-index: 1;
    pointer-events: none;
  }
  .v-chip.active-chip {
    background: white !important;
    color: #000;
  }
}
.all-thumbs-website {
  display: flex;
  flex-wrap: wrap;
  position: absolute;
  opacity: 0.7;
  top: 0;
  left: 0;
  width: 100%;
  pointer-events: none;
}
.network-title {
  font-weight: bold;
  font-size: 2rem;
  cursor: pointer;
}
</style>