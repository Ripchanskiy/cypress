<template>
  <button
    class="h-full grid gap-8px grid-cols-[14px,16px,auto] items-center focus:outline-none"
    :data-cy="`row-directory-depth-${depth}`"
    :aria-expanded="expanded"
  >
    <i-cy-chevron-down-small_x16
      class="
        mr-8px text-sm icon-dark-gray-300
        group-hocus:(icon-dark-gray-700)
      "
      :class="{'transform rotate-270': !expanded}"
    />
    <component
      :is="IconFolder"
      class="icon-dark-white icon-light-gray-200"
    />
    <HighlightedText
      :text="props.name"
      :indexes="props.indexes"
      class="font-medium text-gray-600"
      highlight-classes="text-gray-1000"
    />
    <span class="sr-only">{{ expanded ? 'collapse' : 'expand' }}</span>
  </button>
</template>

<script lang="ts" setup>
import IconFolder from '~icons/cy/folder_x16.svg'
import HighlightedText from './HighlightedText.vue'

const props = withDefaults(defineProps<{ name: string, expanded: boolean, indexes: number[], depth: number }>(), {
  name: '',
  expanded: false,
  indexes: () => [],
})
</script>
