<template>
  <div ref="wrapper" class="list-wrapper">
    <div class="scroll-content">
      <div ref="listWrapper">
        <slot class="list-content"></slot>
      </div>
      <!-- 上拉加载 -->
      <slot name="pullup" :pullUpLoad="pullUpLoad" :isPullUpLoad="isPullUpLoad">
        <div class="pullup-wrapper" v-if="pullUpLoad">
          <div class="before-trigger" v-if="!isPullUpLoad">
            <span>{{pullUpTxt}}</span>
          </div>
          <div class="after-trigger" v-else>
            <loading></loading>
          </div>
        </div>
      </slot>
    </div>
    <!-- 下拉刷新 -->
    <slot
      name="pulldown"
      :pullDownRefresh="pullDownRefresh"
      :pullDownStyle="pullDownStyle"
      :beforePullDown="beforePullDown"
      :isPullingDown="isPullingDown"
      :bubbleY="bubbleY">
      <div ref="pulldown" class="pulldown-wrapper" :style="pullDownStyle" v-if="pullDownRefresh">
        <div class="before-trigger" v-if="beforePullDown">
          <bubble :y="bubbleY"></bubble>
        </div>
        <div class="after-trigger" v-else>
          <div v-if="isPullingDown" class="loading">
            <loading></loading>
          </div>
          <div v-else><span>{{refreshTxt}}</span></div>
        </div>
      </div>
    </slot>
  </div>
</template>

<script>
/**
 * 详细参数配置，http://ustbhuangyi.github.io/better-scroll/doc/zh-hans/
 * 参考链接，http://www.imooc.com/article/18232
 */
import BScroll from 'better-scroll'
import Bubble from './pullDown/bubble'
import { getRect } from './dom'
import Loading from './pullDown/loading'

const DIRECTION_H = 'horizontal'
const DIRECTION_V = 'vertical'

export default {
  name: 'scroll',
  props: {
    data: {
      default: function() {
        return []
      }
    },
    /**
     * 1 滚动的时候会派发scroll事件，会截流。
     * 2 滚动的时候实时派发scroll事件，不会截流。
     * 3 除了实时派发scroll事件，在swipe的情况下仍然能实时派发scroll事件
     */
    probeType: {
      type: Number,
      default: 1
    },
    // 点击列表是否派发click事件
    click: {
      type: Boolean,
      default: true
    },
    // 是否派发滚动事件
    listenScroll: {
      type: Boolean,
      default: false
    },
    // 是否派发 滚动开始之前 事件
    listenBeforeScroll: {
      type: Boolean,
      default: false
    },
    // 是否派发 滚动结束 事件
    listenScrollEnd: {
      type: Boolean,
      default: false
    },
    // 滚动方向
    direction: {
      type: String,
      default: DIRECTION_V
    },
    // 是否 开启滚动条
    scrollbar: {
      type: null,
      default: false
    },
    // 下拉刷新配置
    pullDownRefresh: {
      type: null,
      default: false
    },
    // 上拉加载配置
    pullUpLoad: {
      type: null,
      default: false
    },
    // 当数据更新时，滚动条重新刷新的延时
    refreshDelay: {
      type: Number,
      default: 20
    },
    // 是否 支持横向和纵向同时滚动
    freeScroll: {
      type: Boolean,
      default: false
    },
    // PC 端的鼠标滚轮配置
    mouseWheel: {
      type: Boolean,
      default: false
    },
    // 是否回弹
    bounce: {
      default: true
    },
    // 滚动内容的缩放配置
    zoom: {
      default: false
    },
    // 额外配置
    options: {
      type: Object,
      default() { return {} }
    }
  },
  data() {
    return {
      beforePullDown: true,
      isRebounding: false,
      isPullingDown: false,
      isPullUpLoad: false,
      pullUpDirty: true,
      pullDownStyle: '',
      bubbleY: 0
    }
  },
  computed: {
    pullUpTxt() {
      // 上拉加载时的文本提示
      const moreTxt = (this.pullUpLoad && this.pullUpLoad.txt && this.pullUpLoad.txt.more) || this.$t('prompt.loadMore')
      // 没有更多数据时的文本提示
      const noMoreTxt = (this.pullUpLoad && this.pullUpLoad.txt && this.pullUpLoad.txt.noMore) || this.$t('prompt.noMore')

      return this.pullUpDirty ? moreTxt : noMoreTxt
    },
    refreshTxt() {
      // 下拉刷新文本提示
      return (this.pullDownRefresh && this.pullDownRefresh.txt) || this.$t('prompt.refreshSuccess')
    }
  },
  created() {
    this.pullDownInitTop = -50
  },
  mounted() {
    setTimeout(() => {
      this.initScroll()
    }, 20)
  },
  destroyed() {
    this.$refs.scroll && this.$refs.scroll.destroy()
  },
  methods: {
    initScroll() {
      if (!this.$refs.wrapper) {
        return
      }
      if (this.$refs.listWrapper && (this.pullDownRefresh || this.pullUpLoad)) {
        this.$refs.listWrapper.style.minHeight = `${getRect(this.$refs.wrapper).height + 1}px`
      }
      const options = Object.assign({
        probeType: this.probeType,
        click: this.click,
        scrollY: this.freeScroll || this.direction === DIRECTION_V,
        scrollX: this.freeScroll || this.direction === DIRECTION_H,
        scrollbar: this.scrollbar,
        pullDownRefresh: this.pullDownRefresh,
        pullUpLoad: this.pullUpLoad,
        freeScroll: this.freeScroll,
        mouseWheel: this.mouseWheel,
        bounce: this.bounce,
        zoom: this.zoom
      }, this.options)
      console.log(options)
      this.scroll = new BScroll(this.$refs.wrapper, options)

      if (this.listenScroll) {
        this.scroll.on('scroll', (pos) => {
          this.$emit('scroll', pos)
        })
      }

      if (this.listenScrollEnd) {
        this.scroll.on('scrollEnd', (pos) => {
          this.$emit('scroll-end', pos)
        })
      }

      if (this.listenBeforeScroll) {
        this.scroll.on('beforeScrollStart', () => {
          this.$emit('beforeScrollStart')
        })

        this.scroll.on('scrollStart', () => {
          this.$emit('scroll-start')
        })
      }

      if (this.pullDownRefresh) {
        this._initPullDownRefresh()
      }

      if (this.pullUpLoad) {
        this._initPullUpLoad()
      }
    },
    disable() {
      this.scroll && this.scroll.disable()
    },
    enable() {
      this.scroll && this.scroll.enable()
    },
    refresh() {
      this.scroll && this.scroll.refresh()
    },
    scrollTo() {
      this.scroll && this.scroll.scrollTo.apply(this.scroll, arguments)
    },
    scrollToElement() {
      this.scroll && this.scroll.scrollToElement.apply(this.scroll, arguments)
    },
    destroy() {
      this.scroll.destroy()
    },
    forceUpdate(dirty) {
      if (this.pullDownRefresh && this.isPullingDown) {
        this.isPullingDown = false
        this._reboundPullDown().then(() => {
          this._afterPullDown()
        })
      } else if (this.pullUpLoad && this.isPullUpLoad) {
        this.isPullUpLoad = false
        this.scroll.finishPullUp()
        this.pullUpDirty = dirty
        this.refresh()
      } else {
        this.refresh()
      }
    },
    _initPullDownRefresh() {
      this.scroll.on('pullingDown', () => {
        this.beforePullDown = false
        this.isPullingDown = true
        this.$emit('pullingDown')
      })

      this.scroll.on('scroll', (pos) => {
        if (!this.pullDownRefresh) {
          return
        }
        if (this.beforePullDown) {
          this.bubbleY = Math.max(0, pos.y + this.pullDownInitTop)
          this.pullDownStyle = `top:${Math.min(pos.y + this.pullDownInitTop, 10)}px`
        } else {
          this.bubbleY = 0
        }

        if (this.isRebounding) {
          this.pullDownStyle = `top:${10 - (this.pullDownRefresh.stop || 40 - pos.y)}px`
        }
      })
    },
    _initPullUpLoad() {
      this.scroll.on('pullingUp', () => {
        this.isPullUpLoad = true
        this.$emit('pullingUp')
      })
    },
    _reboundPullDown() {
      const { stopTime = 600 } = this.pullDownRefresh
      return new Promise((resolve) => {
        setTimeout(() => {
          this.isRebounding = true
          this.scroll.finishPullDown()
          resolve()
        }, stopTime)
      })
    },
    _afterPullDown() {
      setTimeout(() => {
        this.pullDownStyle = `top:${this.pullDownInitTop}px`
        this.beforePullDown = true
        this.isRebounding = false
        this.refresh()
      }, this.scroll.options.bounceTime)
    }
  },
  watch: {
    data() {
      setTimeout(() => {
        this.forceUpdate(true)
      }, this.refreshDelay)
    }
  },
  components: {
    Bubble,
    Loading
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
  .list-wrapper {
    position: relative;
    height: 100%;
    overflow: hidden;
    /*background: #fff;*/
    .scroll-content {
      position: relative;
      z-index: 1;
      .list-content {
        position: relative;
        z-index: 10;
        /*background: #ffffff;*/
      }
      .pullup-wrapper {
        width: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        padding: 16px 0;
      }
    }
    .pulldown-wrapper {
      position: absolute;
      width: 100%;
      left: 0;
      display: flex;
      justify-content: center;
      align-items: center;
      transition: all;
      .after-trigger {
        margin-top: 10px;
      }
    }
  }
</style>
