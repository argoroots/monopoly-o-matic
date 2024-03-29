<script setup>
import { computed, defineEmits, defineProps } from 'vue'

const emit = defineEmits(['update:modelValue'])

const props = defineProps({
  modelValue: {
    type: Number,
    default: null
  },
  properties: {
    type: Array,
    required: true
  }
})

const selectedId = computed({
  get () {
    return props.modelValue
  },
  set (val) {
    if (val === props.modelValue || !props.properties[val].type) {
      emit('update:modelValue')
    } else {
      emit('update:modelValue', val)
    }
  }
})
</script>

<template>
  <div
    v-for="p in properties"
    :key="p.id"
    class="border border-white"
    :class="{
      'bg-emerald-50': selectedId === p.id,
      'cursor-pointer hover:bg-emerald-50': p.type
    }"
    :style="{
      'grid-row': p.row,
      'grid-column': p.col,
    }"
    @click="selectedId = p.id"
  >
    <div
      class="w-full h-full p-2 flex justify-center items-center text-center text-sm text-stone-900/80 tracking-wide"
      :class="[p.group, p.class]"
    >
      {{ p.title }}
    </div>
  </div>
</template>

<style scoped>
.corner {
  @apply font-extrabold uppercase;
}

.utility {
  @apply border-white;
}
.brown {
  @apply border-t-8 border-amber-900;
}

.station {
  @apply border-stone-800;
}

.sky {
  @apply border-t-8 border-sky-300;
}

.purple {
  @apply border-r-8 border-fuchsia-700;
}

.orange {
  @apply border-r-8 border-orange-400;
}

.red {
  @apply border-b-8 border-red-600;
}

.yellow {
  @apply border-b-8 border-yellow-300;
}

.green {
  @apply border-l-8 border-green-600;
}

.blue {
  @apply border-l-8 border-blue-800;
}
</style>
