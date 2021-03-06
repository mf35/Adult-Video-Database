<template>
  <v-lazy>
    <v-card @click="$store.state.Performers.bottomSheet = true" 
      @mousedown="stopSmoothScroll($event)" @contextmenu="showContextMenu"
      class="performer-card" :class="{favorite: isFavorite}"
      :data-id="performer.id" hover outlined height="100%" v-ripple="{ class: 'accent--text' }"
    >
      <div class="img-container" :class="{hidden: isFavoriteHidden}">
        <div v-if="!isNationalityHidden" @click="filterByNationality"
          @click.middle="addNewTabWithNationality" class="flag-icon">
          <country-flag :country='findCountryCode(performer.nation)' 
            size='normal' :title="performer.nation" />
        </div>

        <v-img @click.middle="addNewTabPerformer()"
          class="performer-card-img" :src="imgMain" :key="imgMainKey"
          @click="openPerformerPage" :aspect-ratio="5/8" position="top"
        >
          <span class="template-text" v-if="imgMain.includes('performer.png')">Main image</span>
        </v-img>
        
        <v-img @click.middle="addNewTabPerformer()"
          class="secondary-img" :src="imgAlt" :aspect-ratio="5/8" position="top"
          @click="openPerformerPage" :key="imgAltKey" :title="`Open ${performer.name} page`"
        >
          <span class="template-text" v-if="imgAlt.includes('performer_back.png')">Alternate image</span>
        </v-img>

        <div v-if="!isRatingHidden && !$store.state.Settings.ratingAndFavoriteInCard" class="rating-wrapper">
          <v-rating :value="performer.rating" @input="changeRating($event, performer.id)"
            color="yellow darken-2" background-color="grey darken-1"
            empty-icon="mdi-star-outline" half-icon="mdi-star-half-full"
            dense half-increments hover size="18" clearable />
        </div>

        <v-btn v-if="!isFavoriteHidden && !$store.state.Settings.ratingAndFavoriteInCard" 
          @click="toggleFavorite" icon absolute 
          :color="isFavorite===false ? 'white' : 'pink'" class="fav-btn"
        > <v-icon :color="isFavorite===false ? 'grey' : 'pink'"> mdi-heart-outline </v-icon>
        </v-btn>

        <div class="custom1-img-wrapper" v-if="!isCustomImgExist(imgCustom1)">
          <v-img @click="openPerformerPage" @click.middle="addNewTabPerformer()"
            class="custom1-img" :key="imgCustom1Key"
            :src="isCustomImgExist(imgCustom1) ? '' : imgCustom1"
          />
        </div>

        <div class="custom2-img-wrapper" v-if="!isCustomImgExist(imgCustom2)">
          <v-img @click="openPerformerPage" @click.middle="addNewTabPerformer()"
            class="custom2-img" :key="imgCustom2Key"
            :src="isCustomImgExist(imgCustom2) ? '' : imgCustom2"
          /> 
        </div>

        <v-progress-circular v-if="!isProfileProgressHidden" class="profile-complete-progress"
          :value="profileCompleteProgress" size="28" rotate="270" title="Profile complete"
          :color="profileCompleteProgress>=99?'green':'white'" width="2"
        >
          <v-icon v-if="profileCompleteProgress>=99" color="green">mdi-check</v-icon>
          <div v-else class="caption">
            {{profileCompleteProgress}}<span class="percent">%</span>
          </div>
        </v-progress-circular>
        
        <v-icon v-if="performer.bookmark" class="bookmark" color="red" size="28" :title="bookmark">
          mdi-bookmark
        </v-icon>
      </div>

      <v-progress-linear v-if="!isMeterHidden" class="performer-meter" :value="meter" :height="meterHeight"/>
      
      <v-card-title v-if="!isNameHidden" class="performer-card-title pa-2"> 
        <div> {{performerName}} ({{performer.videos}})</div>
        <div v-if="performerAliases.length != 0 && !isAliasesHidden" class="aliases mt-1">
          <span class="aka">aka</span>{{performerAliases}}
        </div>
      </v-card-title>

      <v-card-actions v-if="$store.state.Settings.ratingAndFavoriteInCard" class="px-1 pt-0">
        <v-rating :value="performer.rating" @input="changeRating($event, performer.id)"
          color="yellow darken-2" background-color="grey"
          empty-icon="mdi-star-outline" half-icon="mdi-star-half-full"
          dense half-increments hover size="18" clearable />
        <v-spacer></v-spacer>
        <v-btn @click="isFavorite = !isFavorite" small icon color="pink"> 
          <v-icon :color="isFavorite===false?'grey':'pink'">mdi-heart</v-icon>
        </v-btn>
      </v-card-actions>

      <v-card-text v-if="!isCareerStatusHidden" class="pa-2 performer-career-status">
        <span class="caption">Career status</span> 
        <v-chip class="px-4" label small dark :color="careerStatusColor">
          {{careerStatus}}
        </v-chip>
      </v-card-text>

      <v-card-text v-if="performer.tags.length>0 && !isTagsHidden" class="px-1 py-0">
        <v-chip-group column>
          <v-chip v-for="tag in performer.tags" :key="tag" :to="tagLink(tag)"
            outlined :color="getTagColor(tag)"
            @mouseover.stop="showImage($event, getTagId(tag), 'tag')" 
            @mouseleave.stop="$store.state.hoveredImage=false"
            @click="$store.state.hoveredImage=false"
            @click.middle="addNewTabTag(tag)"
          > {{ tag }}
          </v-chip>
        </v-chip-group>
      </v-card-text>
      
      <v-card-text v-if="tagsFromVideos.length>0 && !isVideoTagsHidden" class="pa-1 py-0">
        <div class="caption px-1">Tags from videos</div>
        <v-chip-group column>
          <v-chip v-for="tag in tagsFromVideos" :key="tag" :to="tagLink(tag)"
            outlined :color="getTagColor(tag)" 
            @mouseover.stop="showImage($event, getTagId(tag), 'tag')" 
            @mouseleave.stop="$store.state.hoveredImage=false"
            @click="$store.state.hoveredImage=false"
            @click.middle="addNewTabTag(tag)"
          > {{ tag }}
          </v-chip>
        </v-chip-group>
      </v-card-text>
      
      <v-card-text v-if="performer.websites.length>0 && !isWebsitesHidden" class="pa-1 py-0">
        <div class="caption px-1">Websites</div>
        <v-chip-group column>
          <v-chip v-for="website in performer.websites" :key="website" :to="websiteLink(website)"
            outlined label :color="getWebsiteColor(website)" 
            @mouseover.stop="showImage($event, getWebsiteId(website), 'website')" 
            @mouseleave.stop="$store.state.hoveredImage=false"
            @click="$store.state.hoveredImage=false"
            @click.middle="addNewTabWebsite(website)"
          > {{ website }}
          </v-chip>
        </v-chip-group>
      </v-card-text>
      
      <v-btn v-if="!isEditBtnHidden" @click="$store.state.Performers.dialogEditPerformerImages=true"
        color="secondary" fab x-small class="btn-edit-images">
        <v-icon>mdi-image-edit-outline</v-icon>
      </v-btn>
      <v-btn v-if="!isEditBtnHidden" @click="$store.state.Performers.dialogEditPerformerInfo=true"
        color="secondary" fab x-small class="btn-edit">
        <v-icon>mdi-pencil</v-icon>
      </v-btn>
    </v-card>
  </v-lazy>
</template>


<script>
const fs = require("fs")
const path = require("path")
const shortid = require("shortid")

import CountryFlag from 'vue-country-flag'
import Countries from '@/mixins/Countries'
import ShowImageFunction from '@/mixins/ShowImageFunction'
import LabelFunctions from '@/mixins/LabelFunctions'

export default {
  name: "PerformerCard",
  props: {
    performer: Object,
  },
  components: {
    CountryFlag,
	},
  mixins: [Countries, ShowImageFunction, LabelFunctions], 
  mounted () {
    this.$nextTick(function () {
      this.performerAliases = this.performer.aliases.join(", ")
      this.performerName = this.performer.name
      this.isFavorite = this.performer.favorite
      this.imgMain = this.getImg(this.performer.id, 'main')
      this.imgAlt = this.getImg(this.performer.id, 'alt')
      this.imgCustom1 = this.getImg(this.performer.id, 'custom1')
      this.imgCustom2 = this.getImg(this.performer.id, 'custom2')
      this.updateMeter()
    })
  },
  data: () => ({
    performerName: '',
    performerAliases: '',
    imgMain: '',
    imgMainKey: Date.now()+0,
    imgAlt: '',
    imgAltKey: Date.now()+1,
    imgCustom1: '',
    imgCustom1Key: Date.now()+2,
    imgCustom2: '',
    imgCustom2Key: Date.now()+3,
    isFavorite: false,
    meter: 0,
    percentValue: 5.263,
  }),
  computed: {
    isChipsColored() {
      return this.$store.state.Settings.performerChipsColored
    },
    isEditBtnHidden() {
      return this.$store.state.Settings.performerEditBtnHidden 
    },
    isRatingHidden() {
      return this.$store.state.Settings.performerRatingHidden
    },
    isNationalityHidden() {
      return this.$store.state.Settings.performerNationalityHidden
    },
    isFavoriteHidden() {
      return this.$store.state.Settings.performerFavoriteHidden
    },
    isProfileProgressHidden() {
      return this.$store.state.Settings.performerProfileProgressHidden
    },
    isNameHidden() {
      return this.$store.state.Settings.performerNameHidden
    },
    isAliasesHidden() {
      return this.$store.state.Settings.performerAliasesHidden
    },
    isMeterHidden() {
      return this.$store.state.Settings.performerMeterHidden
    },
    isCareerStatusHidden() {
      return this.$store.state.Settings.performerCareerStatusHidden
    },
    isTagsHidden() {
      return this.$store.state.Settings.performerTagsHidden
    },
    isVideoTagsHidden() {
      return this.$store.state.Settings.performerVideoTagsHidden
    },
    isWebsitesHidden() {
      return this.$store.state.Settings.performerWebsitesHidden
    },
    aliases: {
      get() {
        let filteredAliases = JSON.parse(JSON.stringify(this.performer.aliases))
        this.performerAliases = filteredAliases.join(", ")
      },
      set(newAliases) {
        if (typeof newAliases === 'object') {
          let filteredAliases = JSON.parse(JSON.stringify(newAliases))
          this.performerAliases = filteredAliases.join(", ")
        }
        if (typeof newAliases === 'string') {
          this.performerAliases = newAliases
        }
      },
    },
    name: {
      get() {
        this.performerName = this.performer.name
      },
      set(newName) {
        if (typeof newName === 'string') {
          this.performerName = newName
        }
      },
    },
    updateImagesData() {
      return this.$store.state.Performers.updateImages
    },
    updateInfoData() {
      return this.$store.state.Performers.updateInfo
    },
    tagsFromVideos() {
      return this.performer.videoTags
    },
    meterHeight() {
      return this.$store.state.Settings.meterHeight
    },
    meterMultiplier() {
      return this.$store.state.Settings.meterMultiplier
    },
    careerStatus() {
      let status
      if (this.performer.start !== "" && this.performer.end === "") {
        status = 'Active'
      }
      if (this.performer.start !== "" && this.performer.end !== "") {
        status = 'Retired'
      }
      if (this.performer.start === "" && this.performer.end === "") {
        status = 'Unknown'
      }
      return status
    },
    careerStatusColor() {
      let status
      if (this.performer.start !== "" && this.performer.end === "") {
        status = 'green'
      }
      if (this.performer.start !== "" && this.performer.end !== "") {
        status = 'orange'
      }
      if (this.performer.start === "" && this.performer.end === "") {
        status = 'grey darken-2'
      }
      return status
    },
    pathToUserData() {
      return this.$store.getters.getPathToUserData
    },
    profileCompleteProgress() {
      let progress = 0
      if (this.performer.name.length) {
        progress += this.percentValue 
      }
      if (this.performer.birthday.length) {
        progress += this.percentValue 
      }
      if (this.performer.nation.length) {
        progress += this.percentValue 
      }
      if (this.performer.ethnicity.length) {
        progress += this.percentValue 
      }
      if (this.performer.hair.length) {
        progress += this.percentValue 
      }
      if (this.performer.eyes.length) {
        progress += this.percentValue 
      }
      if (this.performer.height.length) {
        progress += this.percentValue 
      }
      if (this.performer.weight.length) {
        progress += this.percentValue 
      }
      if (this.performer.bra.length) {
        progress += this.percentValue 
      }
      if (this.performer.waist.length) {
        progress += this.percentValue 
      }
      if (this.performer.hip.length) {
        progress += this.percentValue 
      }
      if (this.performer.boobs.length) {
        progress += this.percentValue 
      }
      if (this.performer.cup.length) {
        progress += this.percentValue 
      }
      if (this.performer.category.length) {
        progress += this.percentValue 
      }
      if (this.performer.body.length) {
        progress += this.percentValue 
      }
      if (this.performer.pussy.length) {
        progress += this.percentValue 
      }
      if (this.performer.pussyLips.length) {
        progress += this.percentValue 
      }
      if (this.performer.pussyHair.length) {
        progress += this.percentValue 
      }
      if (this.performer.start.length) {
        progress += this.percentValue 
      }
      if (progress > 100) progress = 100
      return Math.ceil(progress)
    },
    bookmark() {
      return this.$store.getters.bookmarks.get('performers').find({itemId:this.performer.id}).value().text
    },
    tabId() {
      return this.$route.query.tabId
    },
  },
  methods: {
    stopSmoothScroll(event) {
      if(event.button != 1) return
      event.preventDefault()
      event.stopPropagation()
    },
    addNewTabWithNationality() {
      let key = 'nation'
      let value = [this.performer.nation]
      this.$store.commit('updateFiltersOfPerformers', {key, value})
      this.$store.dispatch('filterPerformers')
      let tabId = shortid.generate()
      let tab = { 
        name: this.performer.nation, 
        link: `/performers/:${tabId}?tabId=${tabId}`,
        id: tabId,
        filters: _.cloneDeep(this.$store.state.Performers.filters),
        icon: 'account-outline'
      }
      this.$store.dispatch('addNewTab', tab)
    },
    updateTabFiltersForNationality() {
      let newFilters = _.cloneDeep(this.$store.state.Performers.filters)
      if (this.tabId === 'default') {
        this.$store.state.Performers.filtersReserved = newFilters
      } else {
        this.$store.getters.tabsDb.find({id: this.tabId}).assign({
          name: this.$store.getters.performersFilters,
          filters: newFilters,
        }).write()
        this.$store.commit('getTabsFromDb')
      }
    },
    filterByNationality() {
      let key = 'nation'
      let value = [this.performer.nation]
      this.$store.commit('updateFiltersOfPerformers', {key, value})
      this.$store.dispatch('filterPerformers')
      this.updateTabFiltersForNationality()
    },
    openPerformerPage() {
      this.$router.push(`/performer/:${this.performer.id}?tabId=default`)
    },
    isCustomImgExist (imgPath) {
      return imgPath.includes('not_exist')
    },
    updateInfo(info) { //TODO:make update only for current card
      if (this.performer.id != info.id) return
      setTimeout(() => {
        if (info.aliases !== undefined) {
          this.aliases = info.aliases
        }
        if (info.name !== undefined) {
          this.name = info.name
        }
        // console.warn('name typeof: '+ typeof info.name)
        // console.warn('name is: '+ info.name)
        console.log(`info updated`)
      }, 1000)
    },
    updateImages(imageData) {
      setTimeout(() => {
        if (imageData.type === 'main') {
          this.imgMain = this.getImg(this.performer.id, imageData.type)
          this.imgMainKey = imageData.key
          console.log(`main img updated`)
        }
        if (imageData.type === 'alt') {
          this.imgAlt = this.getImg(this.performer.id, imageData.type)
          this.imgAltKey = imageData.key
          console.log(`alt img updated`)
        }
        if (imageData.type === 'custom1') {
          this.imgCustom1 = this.getImg(this.performer.id, imageData.type)
          this.imgCustom1Key = imageData.key
          console.log(`custom1 img updated`)
        }
        if (imageData.type === 'custom2') {
          this.imgCustom2 = this.getImg(this.performer.id, imageData.type)
          this.imgCustom2Key = imageData.key
          console.log(`custom2 img updated`)
        }
      }, 3000)
    },
    updateMeter() {
      if (this.tagsFromVideos.length>0 && !this.isMeterHidden) {
        this.meter = 0
        for (let i=0; i < this.tagsFromVideos.length; i++) {
          this.meter += this.tagInPercentsOfMeter(this.getTagValue(this.tagsFromVideos[i]))
        }
        this.meter = this.meter * (1 + this.meterMultiplier / 50)
      }
    },
    getImg(performerId, imageType) {
      let imgMainPath = this.getImgUrl(performerId) + `_${imageType}.jpg`
      return this.checkImageExist(imgMainPath, imageType)
    },
    checkImageExist(imgPath, imgType) {
      if (fs.existsSync(imgPath)) {
        return imgPath
      } else if (imgType === 'alt') {
        return path.join(this.pathToUserData, '/img/templates/performer_back.png')
      } else if (imgType === 'custom1' || imgType === 'custom2') {
        return 'not_exist'
      } else {
        return path.join(this.pathToUserData, '/img/templates/performer.png')
      }
    },
    getImgUrl(img) {
      return  path.join(this.pathToUserData, `/media/performers/${img}`)
    },
    addTag(tagName, videoID) {
      console.log(tagName, videoID);
    },
    changeRating(stars, performerID) {
      console.log('rating changed: ' +stars);
      this.$store.getters.performers
        .find({ id: performerID })
        .assign({ rating: stars })
        .write();
      this.$store.commit('updatePerformers')
    },
    toggleFavorite() {
      if (this.isFavorite) {
        this.isFavorite = false
        console.log('remove from favorite')
      } else {
        this.isFavorite = true
        console.log('added to favorite')
      }

      this.$store.getters.performers
        .find({ id: this.performer.id })
        .assign({ favorite: this.isFavorite })
        .write();
      this.$store.commit('updatePerformers')
    },
    getTagColor(itemName) {
      if (this.isChipsColored) {
        return this.$store.getters.tags.find({name:itemName}).value().color
      } else return ''
    },
    getTagValue(itemName) {
      return this.$store.getters.tags.find({name:itemName}).value().value
    },
    getWebsiteColor(itemName) {
      if (this.isChipsColored) {
        return this.$store.getters.websites.find({name:itemName}).value().color
      } else return ''
    },
    tagInPercentsOfMeter(tagValue) {
      if (tagValue != 0) {
        return (tagValue / this.$store.getters.sumOfTagsValue) * 100
      } else return 0
    },
    findCountryCode(country) {
      if (!country == '') {
        let countryName = country.toLowerCase()
        let code = _.filter(this.countries,
          country => (country.name.toLowerCase().includes(countryName))
        )[0]
        if (code == undefined) {
          return ''
        } else return code.code
      } else return ""
    },
    showContextMenu(e) {
      e.preventDefault()
      this.$store.state.menuTabs = false
      this.$store.state.Performers.menuCard = false
      this.$store.state.x = e.clientX
      this.$store.state.y = e.clientY
      this.$nextTick(() => {
        this.$store.state.Performers.menuCard = true
      })
    },
  },
  watch: {
    updateImagesData(imgData) {
      if (this.performer.id == imgData.id) {
        this.updateImages(imgData)
      }
    },
    updateInfoData(info) {
      this.updateInfo(info)
    },
    isMeterHidden() {
      this.updateMeter()
    },
    meterMultiplier() {
      this.updateMeter()
    },
  },
}
</script>


<style lang="less">
.performer-card {
  box-shadow: 0px 3px 3px 2px rgba(0, 0, 0, 0.12);
  border: none !important;
  cursor: default;
  .v-image {
    cursor: pointer;
  }
  &:hover {
    .btn-edit,
    .btn-edit-images {
      opacity: 0.5;
      &:hover {
        opacity: 1;
      }
    }
    .custom2-img-wrapper,
    .custom1-img-wrapper {
      opacity: 0.7;
      transform: translateX(0);
      &:hover {
        opacity: 1;
      }
    }
    .bookmark {
      opacity: 0.7;
      &:hover {
        opacity: 1;
      }
    }
  }
  &-title {
    font-weight: 400;
    font-size: 16px;
    line-height: 1.2;
  }
  &-img {
    transition: .3s opacity ease-out, 0.3s height ease;
    opacity: 1;
    .v-image__image {
      background-color: rgba(0, 0, 0, 0.5);
    }
  }
  &-inner {
    height: 100%;
  }
  .v-rating .v-icon::before {
    font-size: 1.3em !important;
  }
  &.favorite {
    &:before {
      content: '';
      position: absolute;
      width: 100%;
      height: 100%;
      left: 0;
      top: 0;
      border-radius: 4px;
      pointer-events: none;
      background: none;
      opacity: 1;
      box-shadow: 0px 2px 8px 3px rgba(255, 0, 75, 0.25), 0 0 0 1px rgba(255, 0, 75, 1);
    }
    .img-container:before {
      opacity: 1;
    }
    .hidden:before {
      opacity: 0;
    }
  }
  .fav-btn .v-icon:before {
    font-size: 1.5em !important;
  }
  .rating-wrapper {
    position: absolute;
    bottom: 0;
    left: 0;
    z-index: 1;
    border-top-right-radius: 8px;
  }
  .img-container {
    position: relative;
    overflow: hidden;
    border-radius: 4px 4px 0 0;
    &:hover {
      .btn-open-performer-page {
        opacity: 0.4;
        &:hover {
          opacity: 1;
          .v-btn:before {
            opacity: 0 !important;
          }
        }
      }
      .secondary-img {
        opacity: 1;
      }
      .performer-card-img {
        opacity: 0;
      }
    }
    &:before {
      content: '';
      position: absolute;
      left: 0;
      right: 0;
      top: 0;
      bottom: 0;
      z-index: 1;
      opacity: 0;
      background-image: linear-gradient(225deg, rgba(255, 0, 75, 1) 0%, rgba(0, 0, 0, 0) 12%, rgba(0, 0, 0, 0));
      transition: 1s all ease;
      pointer-events: none;
    }
  }
  .v-input__append-inner {
    display: none;
  }
  .v-chip {
    margin: 0 2px 2px !important;
    padding: 3px 4px;
    line-height: 1;
    height: auto !important;
  }
  .flag-icon {
    position: absolute;
    top: 5px;
    left: 5px;
    z-index: 1;
    width: 24px;
    height: 16px;
    overflow: hidden;
    cursor: pointer;
    .flag {
      position: absolute;
      margin: auto;
      left: -100%;
      right: -100%;
      top: -100%;
      bottom: -100%;
    }
  }
  .aka {
    font-size: 12px;
    font-style: italic;
    margin-left: 7px;
    margin-right: 10px;
  }
  .aliases {
    width: 100%;
    font-size: 12px;
    line-height: 1.2;
    word-break: keep-all;
  }
  .template-text {
    pointer-events: none;
    font-size: 20px;
    font-weight: 300;
    color: #fff;
    opacity: .2;
    position: absolute;
    transform: translateY(-100px);
    top: 0;
    right: 0;
    left: 0;
    bottom: 0;
    margin: auto;
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .btn-edit-images,
  .btn-edit {
    position: absolute;
    right: 5px;
    opacity: 0;
    z-index: 4;
  }
  .btn-edit-images  {
    bottom: 45px;
  }
  .btn-edit {
    bottom: 5px;
  }
  .tags-from-videos {
    font-size: 12px;
    font-weight: 300;
    margin-bottom: 0;
    padding-left: 3px;
  }
  .performer-career-status {
    display: flex;
    justify-content: space-between;
  }
  .bookmark {
    position: absolute;
    top: -4px;
    right: 50px;
    opacity: 0.4;
    text-shadow: 0px 0px 1px #000;
  }
  .secondary-img {
    position: absolute;
    top: 0;
    bottom: 0;
    right: 0;
    left: 0;
    transition: .3s all ease;
    opacity: 0;
    .v-image__image {
      background-position-y: top !important;
      background-color: rgba(0, 0, 0, 0.5);
    } 
  }
  .custom2-img,
  .custom1-img {
    min-width: 100%;
    min-height: 100%;
    &-wrapper {
      width: 40px;
      height: 40px;
      border: 2px solid rgba(255, 255, 255, 0.4);
      border-radius: 50px;
      position: absolute;
      right: 5px;
      overflow: hidden;
      transition: .3s all ease;
      opacity: 0;
      transform: translateX(50px);
      &:hover {
        width: 100%;
        height: 100%;
        border-width: 0px;
        border-radius: 0px;
        position: absolute;
        bottom: 0px;
        right: 0px;
      }
    }
  }
  .custom1-img {
    &-wrapper {
      bottom: calc(50% - 18px);
    }
  }
  .custom2-img {
    &-wrapper {
      bottom: calc(50% - 62px);
    }
  }
  .profile-complete-progress {
    position: absolute;
    z-index: 1;
    bottom: 5px;
    right: 5px;
    background-color: rgba(90, 90, 90, 0.5);
    border-radius: 50%;
    .percent {
      font-size: 8px;
    }
  }
}
</style>