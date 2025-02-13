<template>
  <div>
    <div v-if="propsToSelect.length" class="relative flex border border-gray-200 dark:border-gray-700 rounded-t-md overflow-hidden not-prose">
      <div v-for="prop in propsToSelect" :key="prop.name" class="flex flex-col gap-0.5 justify-between py-1.5 font-medium bg-gray-50 dark:bg-gray-800 border-r border-r-gray-200 dark:border-r-gray-700">
        <label :for="prop.name" class="block text-xs px-3 font-medium text-gray-400 dark:text-gray-500 -my-px">{{ prop.label }}</label>
        <UCheckbox
          v-if="prop.type === 'boolean'"
          v-model="componentProps[prop.name]"
          :name="prop.name"
          variant="none"
          class="justify-center"
        />
        <USelectMenu
          v-else-if="prop.type === 'string' && prop.options.length"
          v-model="componentProps[prop.name]"
          :options="prop.options"
          :name="prop.name"
          :label="componentProps[prop.name]"
          variant="none"
          class="inline-flex"
          :ui="{ width: 'w-32 !-mt-px', rounded: 'rounded-b-md' }"
          :ui-select="{ custom: '!py-0' }"
          :popper="{ strategy: 'fixed', placement: 'bottom-start' }"
        />
        <UInput
          v-else
          :model-value="componentProps[prop.name]"
          :type="prop.type === 'number' ? 'number' : 'text'"
          :name="prop.name"
          variant="none"
          autocomplete="off"
          :ui="{ custom: '!py-0' }"
          @update:model-value="val => componentProps[prop.name] = prop.type === 'number' ? Number(val) : val"
        />
      </div>
    </div>

    <div class="flex border border-b-0 border-gray-200 dark:border-gray-700 relative not-prose" :class="[{ 'p-4': padding }, propsToSelect.length ? 'border-t-0' : 'rounded-t-md', backgroundClass]">
      <component :is="name" v-model="vModel" v-bind="fullProps">
        <ContentSlot v-if="$slots.default" :use="$slots.default" />
      </component>
    </div>

    <ContentRenderer :value="ast" class="[&>div>pre]:!rounded-t-none" />
  </div>
</template>

<script setup lang="ts">
// @ts-expect-error
import { transformContent } from '@nuxt/content/transformers'

// eslint-disable-next-line vue/no-dupe-keys
const props = defineProps({
  slug: {
    type: String,
    default: null
  },
  padding: {
    type: Boolean,
    default: true
  },
  props: {
    type: Object,
    default: () => ({})
  },
  code: {
    type: String,
    default: null
  },
  baseProps: {
    type: Object,
    default: () => ({})
  },
  ui: {
    type: Object,
    default: () => ({})
  },
  excludedProps: {
    type: Array,
    default: () => []
  },
  extraColors: {
    type: Array,
    default: () => []
  },
  backgroundClass: {
    type: String,
    default: 'bg-white dark:bg-gray-900'
  }
})

// eslint-disable-next-line vue/no-dupe-keys
const baseProps = reactive({ ...props.baseProps })
const componentProps = reactive({ ...props.props })

const appConfig = useAppConfig()
const route = useRoute()
// eslint-disable-next-line vue/no-dupe-keys
const slug = props.slug || route.params.slug[1]
const camelName = useCamelCase(slug)
const name = `U${useUpperFirst(camelName)}`

const meta = await fetchComponentMeta(name)

// Computed

// eslint-disable-next-line vue/no-dupe-keys
const ui = computed(() => ({ ...appConfig.ui[camelName], ...props.ui }))

const fullProps = computed(() => ({ ...props.baseProps, ...componentProps }))
const vModel = computed({
  get: () => baseProps.modelValue,
  set: (value) => {
    baseProps.modelValue = value
  }
})

const propsToSelect = computed(() => Object.keys(componentProps).map((key) => {
  if (props.excludedProps.includes(key)) {
    return null
  }

  const prop = meta?.meta?.props?.find((prop: any) => prop.name === key)
  const dottedKey = useKebabCase(key).replaceAll('-', '.')
  const keys = useGet(ui.value, dottedKey, {})
  let options = typeof keys === 'object' && Object.keys(keys)
  if (key.toLowerCase().endsWith('color')) {
    // @ts-ignore
    options = [...appConfig.ui.colors, ...props.extraColors]
  }

  return {
    type: prop?.type || 'string',
    name: key,
    label: key === 'modelValue' ? 'value' : useCamelCase(key),
    options
  }
}).filter(Boolean))

// eslint-disable-next-line vue/no-dupe-keys
const code = computed(() => {
  let code = `\`\`\`html
<${name}`
  for (const [key, value] of Object.entries(componentProps)) {
    if (value === 'undefined' || value === null) {
      continue
    }

    const prop = meta?.meta?.props?.find((prop: any) => prop.name === key)

    code += ` ${(prop?.type === 'boolean' && value !== true) || typeof value === 'object' ? ':' : ''}${key === 'modelValue' ? 'value' : useKebabCase(key)}${prop?.type === 'boolean' && !!value && key !== 'modelValue' ? '' : `="${typeof value === 'object' ? renderObject(value) : value}"`}`
  }
  if (props.code) {
    const lineBreaks = (props.code.match(/\n/g) || []).length
    if (lineBreaks > 1) {
      code += `>
  ${props.code}</${name}>`
    } else {
      code += `>${props.code}</${name}>`
    }
  } else {
    code += ' />'
  }
  code += `
\`\`\`
`
  return code
})

function renderObject (obj: any) {
  if (Array.isArray(obj)) {
    return `[${obj.map(renderObject).join(', ')}]`
  }

  if (typeof obj === 'object') {
    return `{ ${Object.entries(obj).map(([key, value]) => `${key}: ${renderObject(value)}`).join(', ')} }`
  }

  if (typeof obj === 'string') {
    return `'${obj}'`
  }

  return obj
}

const { data: ast } = await useAsyncData(`${name}-ast-${JSON.stringify(componentProps)}`, () => transformContent('content:_markdown.md', code.value, {
  highlight: {
    theme: {
      light: 'material-lighter',
      dark: 'material-palenight'
    }
  }
}), { watch: [code] })
</script>
