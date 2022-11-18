<script lang="ts">
import config from '@config'
import { get } from 'lodash-es'
import { computed, defineComponent, onMounted, onBeforeUnmount, ref, watch, nextTick } from 'vue'
import { animationBoxData, animationEl } from '../floating'
import type { Ref, StyleValue } from 'vue'

export default defineComponent({
  name: 'LogoAnimation',
  setup() {
    const logo: Ref<string> = get(config, 'logo')
    const proxyRect: Ref<DOMRect | undefined> = ref()
    const floatingState: Ref<boolean> = ref<boolean>(false)

    watch(animationEl, async (newEl) => {
      if (!newEl) return
      floatingState.value = true
      await new Promise(resolve => window.requestAnimationFrame(resolve))

      const handler = () => {
        const newRect = newEl.getBoundingClientRect()
        if (
          !proxyRect.value ||
          (
            newRect.top === proxyRect.value.top
            && newRect.left === proxyRect.value.left
            && newRect.width === proxyRect.value.width
            && newRect.height === proxyRect.value.height
          )
        ) {
          floatingState.value = false
        }
        proxyRect.value = newRect
      }

      window.addEventListener('scroll', handler)

      const unwatch = watch(floatingState, state => {
        if (!state) {
          console.log('unwatch')
          window.removeEventListener('scroll', handler)
          unwatch()
        }
      })

      handler()
    })

    const styles: Ref<StyleValue | undefined> = computed(() => ({
      width: `${proxyRect.value?.width}px`,
      height: `${proxyRect.value?.height}px`,
      top: `${proxyRect.value?.top}px`,
      left: `${proxyRect.value?.left}px`,
    }))

    onMounted(() => {
      window.addEventListener('resize', resetPosition, false)
    })

    onBeforeUnmount(() => {
      resetTimer && clearTimeout(resetTimer)
      window.removeEventListener('resize', resetPosition)
    })

    const land = () => {
      floatingState.value = false
    }

    let resetTimer: number

    const resetPosition = async () => {
      if (floatingState.value) {
        floatingState.value = false
        await nextTick()
      }
      clearTimeout(resetTimer)
      resetTimer = window.setTimeout(() => {
        const newRect = animationEl.value?.getBoundingClientRect()
        if (newRect) proxyRect.value = newRect
      }, 200)
    }

    return {
      logo,
      animationBoxData,
      styles,
      animationEl,
      floatingState,
      land,
    }
  },
})
</script>

<template>
  <Teleport :to="animationEl || 'body'">
    <img 
      v-if="logo"
      v-bind="animationBoxData.attrs" 
      :class="{ 'varlet-cli-logo-position': floatingState }" 
      :style="styles" 
      :src="logo" 
      class="varlet-cli-logo-entity"
      alt="logo" 
      @transitionend="land"
    />
  </Teleport>
</template>

<style lang="less">
.varlet-cli-logo-transition {
  width: 100%;
  height: 100%;
}

.varlet-cli-logo-entity {
  z-index: 10;
  pointer-events: none;
}

body > .varlet-cli-logo-entity {
  visibility: hidden;
}

.varlet-cli-logo-position {
  position: fixed;
  transition: 0.5s ease-in-out;
  transition-property: left, top, height, width;
}
</style>
