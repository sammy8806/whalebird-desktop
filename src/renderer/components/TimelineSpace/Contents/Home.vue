<template>
<div id="home">
  <div class="unread">{{ unread.length > 0 ? unread.length : '' }}</div>
  <div v-shortkey="{linux: ['ctrl', 'r'], mac: ['meta', 'r']}" @shortkey="reload()">
  </div>
  <transition-group name="timeline" tag="div">
    <div class="home-timeline" v-for="message in timeline" :key="message.id">
      <toot :message="message" :filter="filter" v-on:update="updateToot" v-on:delete="deleteToot"></toot>
    </div>
  </transition-group>
  <div class="loading-card" v-loading="lazyLoading" :element-loading-background="backgroundColor">
  </div>
  <div class="upper" v-show="!heading">
    <el-button type="primary" icon="el-icon-arrow-up" @click="upper" circle>
    </el-button>
  </div>
</div>
</template>

<script>
import { mapState } from 'vuex'
import Toot from './Cards/Toot'
import scrollTop from '../../utils/scroll'

export default {
  name: 'home',
  components: { Toot },
  computed: {
    ...mapState({
      backgroundColor: state => state.App.theme.background_color,
      startReload: state => state.TimelineSpace.HeaderMenu.reload,
      timeline: state => state.TimelineSpace.Contents.Home.timeline,
      lazyLoading: state => state.TimelineSpace.Contents.Home.lazyLoading,
      heading: state => state.TimelineSpace.Contents.Home.heading,
      unread: state => state.TimelineSpace.Contents.Home.unreadTimeline,
      filter: state => state.TimelineSpace.Contents.Home.filter
    })
  },
  mounted () {
    this.$store.commit('TimelineSpace/SideMenu/changeUnreadHomeTimeline', false)
    document.getElementById('scrollable').addEventListener('scroll', this.onScroll)
  },
  beforeUpdate () {
    if (this.$store.state.TimelineSpace.SideMenu.unreadHomeTimeline && this.heading) {
      this.$store.commit('TimelineSpace/SideMenu/changeUnreadHomeTimeline', false)
    }
  },
  destroyed () {
    this.$store.commit('TimelineSpace/Contents/Home/changeHeading', true)
    this.$store.commit('TimelineSpace/Contents/Home/mergeTimeline')
    this.$store.commit('TimelineSpace/Contents/Home/archiveTimeline')
    if (document.getElementById('scrollable') !== undefined && document.getElementById('scrollable') !== null) {
      document.getElementById('scrollable').removeEventListener('scroll', this.onScroll)
      document.getElementById('scrollable').scrollTop = 0
    }
  },
  watch: {
    startReload: function (newState, oldState) {
      if (!oldState && newState) {
        this.reload()
          .finally(() => {
            this.$store.commit('TimelineSpace/HeaderMenu/changeReload', false)
          })
      }
    }
  },
  methods: {
    onScroll (event) {
      // for lazyLoading
      if (((event.target.clientHeight + event.target.scrollTop) >= document.getElementById('home').clientHeight - 10) && !this.lazyloading) {
        this.$store.dispatch('TimelineSpace/Contents/Home/lazyFetchTimeline', this.timeline[this.timeline.length - 1])
          .catch(() => {
            this.$message({
              message: this.$t('message.timeline_fetch_error'),
              type: 'error'
            })
          })
      }
      // for unread control
      if ((event.target.scrollTop > 10) && this.heading) {
        this.$store.commit('TimelineSpace/Contents/Home/changeHeading', false)
      } else if ((event.target.scrollTop <= 10) && !this.heading) {
        this.$store.commit('TimelineSpace/Contents/Home/changeHeading', true)
        this.$store.commit('TimelineSpace/Contents/Home/mergeTimeline')
      }
    },
    updateToot (message) {
      this.$store.commit('TimelineSpace/Contents/Home/updateToot', message)
    },
    deleteToot (message) {
      this.$store.commit('TimelineSpace/Contents/Home/deleteToot', message)
    },
    async reload () {
      this.$store.commit('TimelineSpace/changeLoading', true)
      try {
        const account = await this.$store.dispatch('TimelineSpace/localAccount', this.$route.params.id).catch((err) => {
          this.$message({
            message: this.$t('message.account_load_error'),
            type: 'error'
          })
          throw err
        })
        await this.$store.dispatch('TimelineSpace/stopUserStreaming')
        await this.$store.dispatch('TimelineSpace/stopLocalStreaming')

        await this.$store.dispatch('TimelineSpace/Contents/Home/fetchTimeline', account)
          .catch(() => {
            this.$message({
              message: this.$t('message.timeline_fetch_error'),
              type: 'error'
            })
          })
        await this.$store.dispatch('TimelineSpace/Contents/Local/fetchLocalTimeline', account)

        this.$store.dispatch('TimelineSpace/startUserStreaming', account)
          .catch(() => {
            this.$message({
              message: this.$t('message.start_streaming_error'),
              type: 'error'
            })
          })
        this.$store.dispatch('TimelineSpace/startLocalStreaming', account)
      } finally {
        this.$store.commit('TimelineSpace/changeLoading', false)
      }
    },
    upper () {
      scrollTop(
        document.getElementById('scrollable'),
        0
      )
    }
  }
}
</script>

<style lang="scss" scoped>
#home {
  .unread {
    position: fixed;
    right: 24px;
    top: 48px;
    background-color: rgba(0, 0, 0, 0.7);
    color: #ffffff;
    padding: 4px 8px;
    border-radius: 0 0 2px 2px;

    &:empty {
      display: none;
    }
  }

  .loading-card {
    height: 60px;
  }

  .loading-card:empty {
    height: 0;
  }

  .upper {
    position: fixed;
    bottom: 20px;
    right: 20px;
  }
}
</style>
<style src="@/assets/timeline-transition.scss"></style>
