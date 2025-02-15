<template>
  <transition appear v-bind="ui.transition">
    <div :class="[ui.wrapper, ui.background, ui.rounded, ui.shadow]" @mouseover="onMouseover" @mouseleave="onMouseleave">
      <div :class="[ui.container, ui.rounded, ui.ring]">
        <div :class="ui.padding">
          <div class="flex gap-3" :class="{ 'items-start': description, 'items-center': !description }">
            <UIcon v-if="icon" :name="icon" :class="iconClass" />
            <UAvatar v-if="avatar" v-bind="{ size: ui.avatar.size, ...avatar }" :class="ui.avatar.base" />

            <div class="w-0 flex-1">
              <p :class="ui.title">
                {{ title }}
              </p>
              <p v-if="description" :class="ui.description">
                {{ description }}
              </p>

              <div v-if="description && actions.length" class="mt-3 flex items-center gap-2">
                <UButton v-for="(action, index) of actions" :key="index" v-bind="{ ...ui.default.action, ...action }" @click.stop="onAction(action)" />
              </div>
            </div>
            <div class="flex-shrink-0 flex items-center gap-3">
              <div v-if="!description && actions.length" class="flex items-center gap-2">
                <UButton v-for="(action, index) of actions" :key="index" v-bind="{ ...ui.default.action, ...action }" @click.stop="onAction(action)" />
              </div>

              <UButton v-if="close" v-bind="{ ...ui.default.close, ...close }" @click.stop="onClose" />
            </div>
          </div>
        </div>
        <div v-if="timeout" :class="progressClass" :style="progressStyle" />
      </div>
    </div>
  </transition>
</template>

<script lang="ts">
import { ref, computed, onMounted, onUnmounted, watchEffect, defineComponent } from 'vue'
import type { PropType } from 'vue'
import { defu } from 'defu'
import UIcon from '../elements/Icon.vue'
import UAvatar from '../elements/Avatar.vue'
import UButton from '../elements/Button.vue'
import { useTimer } from '../../composables/useTimer'
import type { NotificationAction } from '../../types'
import type { Avatar} from '../../types/avatar'
import type { Button } from '../../types/button'
import { classNames } from '../../utils'
import { useAppConfig } from '#imports'
// TODO: Remove
// @ts-expect-error
import appConfig from '#build/app.config'

// const appConfig = useAppConfig()

export default defineComponent({
  components: {
    UIcon,
    UAvatar,
    UButton
  },
  props: {
    id: {
      type: [String, Number],
      required: true
    },
    title: {
      type: String,
      required: true
    },
    description: {
      type: String,
      default: null
    },
    icon: {
      type: String,
      default: () => appConfig.ui.notification.default.icon
    },
    avatar: {
      type: Object as PropType<Partial<Avatar>>,
      default: null
    },
    close: {
      type: Object as PropType<Partial<Button>>,
      default: () => appConfig.ui.notification.default.close
    },
    timeout: {
      type: Number,
      default: 5000
    },
    actions: {
      type: Array as PropType<NotificationAction[]>,
      default: () => []
    },
    callback: {
      type: Function,
      default: null
    },
    color: {
      type: String,
      default: () => appConfig.ui.notification.default.color,
      validator (value: string) {
        return ['gray', ...appConfig.ui.colors].includes(value)
      }
    },
    ui: {
      type: Object as PropType<Partial<typeof appConfig.ui.notification>>,
      default: () => appConfig.ui.notification
    }
  },
  emits: ['close'],
  setup (props, { emit }) {
    // TODO: Remove
    const appConfig = useAppConfig()

    const ui = computed<Partial<typeof appConfig.ui.notification>>(() => defu({}, props.ui, appConfig.ui.notification))

    let timer: any = null
    const remaining = ref(props.timeout)

    const progressStyle = computed(() => {
      const remainingPercent = remaining.value / props.timeout * 100

      return { width: `${remainingPercent || 0}%` }
    })

    const progressClass = computed(() => {
      return classNames(
        ui.value.progress.base,
        ui.value.progress.background?.replaceAll('{color}', props.color)
      )
    })

    const iconClass = computed(() => {
      return classNames(
        ui.value.icon.base,
        appConfig.ui.colors.includes(props.color) && ui.value.icon.color?.replaceAll('{color}', props.color)
      )
    })

    function onMouseover () {
      if (timer) {
        timer.pause()
      }
    }

    function onMouseleave () {
      if (timer) {
        timer.resume()
      }
    }

    function onClose () {
      if (timer) {
        timer.stop()
      }

      if (props.callback) {
        props.callback()
      }

      emit('close')
    }

    function onAction (action: NotificationAction) {
      if (timer) {
        timer.stop()
      }

      if (action.click) {
        action.click()
      }

      emit('close')
    }

    onMounted(() => {
      if (!props.timeout) {
        return
      }

      timer = useTimer(() => {
        onClose()
      }, props.timeout)

      watchEffect(() => {
        remaining.value = timer.remaining.value
      })
    })

    onUnmounted(() => {
      if (timer) {
        timer.stop()
      }
    })

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      progressStyle,
      progressClass,
      iconClass,
      onMouseover,
      onMouseleave,
      onClose,
      onAction
    }
  }
})
</script>
