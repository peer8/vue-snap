<template>
  <div class="vs-carousel">
    <component
      :is="tag"
      ref="vsWrapper"
      class="vs-carousel__wrapper"
      :class="{ 'vs-carousel--vertical': isVertical }"
    >
      <!-- @slot Slot for Slides -->
      <slot />
    </component>

    <!-- Slide indicators -->
    <div
      v-if="indicators"
      :class="`slide-indicators ${indicators}`"
      @click.stop
    >
      <span
        v-for="i in slidesDim.length"
        :key="i"
        :class="{ 'slide-indicator-active': currentPage === i - 1 }"
        class="slide-indicator-icon"
        @click="handleIndicator(i - 1)"
      />
    </div>

    <!-- @slot Slot for Arrows -->
    <slot
      v-if="!hideArrows"
      name="arrows"
      :change-slide="changeSlide"
      :bound-left="boundLeft"
      :bound-right="boundRight"
    >
      <button
        v-show="hideArrowsOnBound ? !boundLeft : true"
        type="button"
        :aria-label="i18n.slideLeft"
        :disabled="boundLeft"
        class="
          vs-carousel__arrows
          vs-carousel__arrows--left
        "
        @click="changeSlide(-1)"
      >
        ←
      </button>

      <button
        v-show="hideArrowsOnBound ? !boundRight : true"
        type="button"
        :aria-label="i18n.slideRight"
        :disabled="boundRight"
        class="
          vs-carousel__arrows
          vs-carousel__arrows--right
        "
        @click="changeSlide(1)"
      >
        →
      </button>
    </slot>
  </div>
</template>

<script>
import debounce from 'lodash/debounce'
import { approximatelyEqual, isClient } from '../../utils'

const SCROLL_DEBOUNCE = 100
const RESIZE_DEBOUNCE = 410

export default {
  props: {
    /**
     * Disable arrows
     */
    hideArrows: {
      type: Boolean,
      default: false
    },
    /**
     * Disable arrows on bound
     */
    hideArrowsOnBound: {
      type: Boolean,
      default: false
    },
    /**
     * Make a vertical carousel (scroller)
     */
    isVertical: {
      type: Boolean,
      default: false
    },
    /**
     * Show slide indicators
     */
    indicators: {
      type: String,
      default: 'center'
    },
    /**
     * Custom tag
     */
    tag: {
      type: String,
      default: 'ul'
    },
    /**
     * Translations
     */
    i18n: {
      type: Object,
      // TODO: isVertical ? :
      default: () => ({ slideLeft: 'Slide left', slideRight: 'Slide right' }),
      validator: config => {
        const translations = ['slideLeft', 'slideRight']
        return translations.every(key => key in config)
      }
    }
  },
  data: () => ({
    boundLeft: true,
    boundRight: false,
    slidesDim: [],
    wrapperScrollDim: 0,
    wrapperVisibleDim: 0,
    currentPage: 0,
    previousPage: 0,
    currentPos: 0,
    step: 1,
    observer: null,
    onResizeFn: null,
    onScrollFn: null
  }),
  watch: {
    currentPage(currentPage, previousPage) {
      if (currentPage === previousPage) {
        return
      }

      /**
       * Page changed
       * @event page
       * @type {Event}
       */
      this.$emit('page', { currentPage, previousPage })
    },
    boundLeft(state) {
      if (!state) {
        return
      }

      /**
       * Reach first item
       * @event bound-left
       * @type {Event}
       */
      this.$emit('bound-left', true)
    },
    boundRight(state) {
      if (!state) {
        return
      }

      /**
       * Reach last item
       * @event bound-right
       * @type {Event}
       */
      this.$emit('bound-right', true)
    },
  },
  mounted() {
    this.calcOnInit()

    if (isClient) {
      // Assign to new variable and keep reference for removeEventListener (Avoid Memory Leaks)
      this.onResizeFn = debounce(this.calcOnInit, RESIZE_DEBOUNCE)
      this.onScrollFn = debounce(this.calcOnScroll, SCROLL_DEBOUNCE)

      // MutationObserver
      this.attachMutationObserver()

      // Events
      this.$refs.vsWrapper.addEventListener('scroll', this.onScrollFn)
      window.addEventListener('resize', this.onResizeFn, false)

      this.$on('go-to-page', index => this.goToSlide(index))

      /**
       * Carousel mounted
       * @event mounted
       * @type {Event}
       */
      this.$emit('mounted', true)
    }
  },
  beforeDestroy() {
    if (isClient) {
      // MutationObserver
      this.observer.disconnect()

      // Events
      this.$refs.vsWrapper.removeEventListener('scroll', this.onScrollFn)
      window.removeEventListener('resize', this.onResizeFn, false)
    }
  },
  methods: {
    calcOnInit() {
      if (!this.$refs.vsWrapper) {
        return
      }

      this.calcWrapperDim()
      this.calcSlidesDim()
      this.calcCurrentPosition()
      this.setCurrentPage()
      this.calcBounds()
    },
    calcOnScroll() {
      if (!this.$refs.vsWrapper) {
        return
      }

      this.calcCurrentPosition()
      this.setCurrentPage()
      this.calcBounds()
    },
    calcBounds() {
      // Find the closest point, with 5px approximate.
      const isBoundLeft = approximatelyEqual(this.currentPos, 0, 5)
      const isBoundRight = approximatelyEqual(
        this.wrapperScrollDim - this.wrapperVisibleDim,
        this.currentPos,
        5
      )

      if (isBoundLeft) {
        this.boundLeft = true
      } else {
        this.boundLeft = false
      }

      if (isBoundRight) {
        this.boundRight = true
      } else {
        this.boundRight = false
      }
    },
    calcWrapperDim() {
      if (this.isVertical) {
        this.wrapperScrollDim = this.$refs.vsWrapper.scrollHeight
        this.wrapperVisibleDim = this.$refs.vsWrapper.offsetHeight
      } else {
        this.wrapperScrollDim = this.$refs.vsWrapper.scrollWidth
        this.wrapperVisibleDim = this.$refs.vsWrapper.offsetWidth
      }
    },
    calcSlidesDim() {
      const childNodes = [ ...this.$refs.vsWrapper.children ]

      if (this.isVertical) {
        this.slidesDim = childNodes.map(node => ({
          offsetTop: node.offsetTop,
          height: node.offsetHeight
        }))
      } else {
        this.slidesDim = childNodes.map(node => ({
          offsetLeft: node.offsetLeft,
          width: node.offsetWidth
        }))
      }
    },
    getCurrentPage() {
      // Last element if there is nothing left to scroll
      if (approximatelyEqual(
        this.currentPos + this.wrapperVisibleDim,
        this.wrapperScrollDim,
        5
      )) {
        return this.slidesDim.length - 1
      }

      // Find slide closest to scroll position, with 5px approximate
      return this.slidesDim.findIndex(slide => {
        return approximatelyEqual(slide.offsetLeft, this.currentPos, 5)
      })
    },
    setCurrentPage(index) {
      const newPage = index !== undefined ? index : this.getCurrentPage()

      if (newPage < 0) {
        return
      }

      this.previousPage = this.currentPage
      this.currentPage = newPage > 0 ? newPage : 0
    },
    calcCurrentPosition() {
      this.currentPos = this.$refs.vsWrapper.scrollLeft || 0
    },
    attachMutationObserver() {
      this.observer = new MutationObserver(() => {
        this.calcOnInit()
      })

      this.observer.observe(
        this.$refs.vsWrapper,
        { attributes: true, childList: true, characterData: true, subtree: true }
      )
    },
    changeSlide(direction) {
      this.goToSlide(this.currentPage + direction)
    },
    goToSlide(index) {
      if (!this.slidesDim[index]) {
        return
      }

      if (this.isVertical) {
        this.$refs.vsWrapper.scrollTo({
          top: this.slidesDim[index].offsetTop,
          behavior: 'smooth'
        })
      } else {
        this.$refs.vsWrapper.scrollTo({
          left: this.slidesDim[index].offsetLeft,
          behavior: 'smooth'
        })
      }

      this.setCurrentPage(index)
    },
    handleIndicator(index) {
      console.log('handleIndicator', index)
      this.goToSlide(index)
    }
  }
}
</script>
