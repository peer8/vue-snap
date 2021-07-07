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
        v-if="hideArrowsOnBound ? !boundLeft : true"
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
import { onMounted, ref, onBeforeUnmount, watch } from 'vue'
import debounce from 'lodash/debounce'
import { approximatelyEqual, isClient } from '../../utils'

const SCROLL_DEBOUNCE = 100
const RESIZE_DEBOUNCE = 410

const props = {
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
    // TODO: isVertical ?
    default: () => ({ slideLeft: 'Slide left', slideRight: 'Slide right' }),
    validator: config => {
      const translations = ['slideLeft', 'slideRight']
      return translations.every(key => key in config)
    }
  }
}

export default {
  name: 'Carousel',
  props,
  setup(_, { emit }) {
    const vsWrapper = ref(null)
    const boundLeft = ref(true)
    const boundRight = ref(false)
    const slidesDim = ref([])
    const wrapperScrollDim = ref(0)
    const wrapperVisibleDim = ref(0)
    const currentPage = ref(0)
    const currentPos = ref(0)
    const maxPages = ref(0)
    const onResizeFn = ref(null)
    const onScrollFn = ref(null)

    // Watchers
    watch(currentPage, (current, previous) => {
      if (current !== previous) {
        /**
         * Page changed
         * @event page
         * @type {Event}
         */
        emit('page', { current, previous })
      }
    })

    // Calculations
    const calcBounds = () => {
      // Find the closest point, with 5px approximate.
      const isBoundLeft = approximatelyEqual(currentPos.value, 0, 5)
      const isBoundRight = approximatelyEqual(
        wrapperScrollDim.value - wrapperVisibleDim.value,
        currentPos.value,
        5
      )

      if (isBoundLeft) {
        /**
         * Reach first item
         * @event bound-left
         * @type {Event}
         */
        emit('bound-left', true)
        boundLeft.value = true
      } else {
        boundLeft.value = false
      }

      if (isBoundRight) {
        /**
         * Reach last item
         * @event bound-right
         * @type {Event}
         */
        emit('bound-right', true)
        boundRight.value = true
      } else {
        boundRight.value = false
      }
    }
    const calcWrapperDim = () => {
      wrapperScrollDim.value = vsWrapper.value.scrollWidth
      wrapperVisibleDim.value = vsWrapper.value.offsetWidth
    }
    const calcSlidesDim = () => {
      const childNodes = [ ...vsWrapper.value.children ]

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
    }
    const calcNextDim = direction => {
      const nextSlideIndex = direction > 0 ? currentPage.value : currentPage.value + direction
      const dim = slidesDim.value[nextSlideIndex][this.isVertical ? 'height' : 'width'] || 0

      if (!dim) {
        return
      }

      return dim * direction
    }
    const calcCurrentPage = () => {
      const getCurrentPage = slidesDim.value.findIndex(slide => {
        // Find the closest point, with 5px approximate.
        return approximatelyEqual(this.isVertical ? slide.offsetTop : slide.offsetLeft, currentPos.value, 5)
      })

      if (getCurrentPage !== -1 && getCurrentPage !== -2) {
        currentPage.value = getCurrentPage || 0
      }
    }
    const calcCurrentPosition = () => {
      currentPos.value = vsWrapper.value[this.isVertical ? 'scrollTop' : 'scrollLeft'] || 0
    }
    const calcMaxPages = () => {
      const maxPos = wrapperScrollDim.value - wrapperVisibleDim.value
      if (this.isVertical) {
        maxPages.value = slidesDim.value.findIndex(({ offsetTop }) => offsetTop > maxPos) - 1
      } else {
        maxPages.value = slidesDim.value.findIndex(({ offsetLeft }) => offsetLeft > maxPos) - 1
      }
    }
    const calcOnInit = () => {
      if (!vsWrapper.value) {
        return
      }

      calcWrapperDim()
      calcSlidesDim()
      calcCurrentPosition()
      calcCurrentPage()
      calcBounds()
      calcMaxPages()
    }
    const calcOnScroll = () => {
      if (!vsWrapper.value) {
        return
      }

      calcCurrentPosition()
      calcCurrentPage()
      calcBounds()
    }

    const changeSlide = direction => {
      const nextSlideDim = calcNextDim(direction)

      if (nextSlideDim) {
        if (this.isVertical) {
          vsWrapper.value.scrollBy({ top: nextSlideDim, behavior: 'smooth' })
        } else {
          vsWrapper.value.scrollBy({ left: nextSlideDim, behavior: 'smooth' })
        }
      }
    }

    onMounted(() => {
      calcOnInit()

      if (isClient) {
        // Assign to new variable and keep reference for removeEventListener (Avoid Memory Leaks)
        onScrollFn.value = debounce(calcOnScroll, SCROLL_DEBOUNCE)
        onResizeFn.value = debounce(calcOnInit, RESIZE_DEBOUNCE)

        // Events
        vsWrapper.value.addEventListener('scroll', onScrollFn.value)
        window.addEventListener('resize', onResizeFn.value)
      }
    })
    onBeforeUnmount(() => {
      if (isClient) {
        // Events
        vsWrapper.value.removeEventListener('scroll', onScrollFn.value)
        window.removeEventListener('resize', onResizeFn.value)
      }
    })

    return { boundLeft, boundRight, changeSlide, vsWrapper }
  }
}
</script>
