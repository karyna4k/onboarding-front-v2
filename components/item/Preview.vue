<template>
  <div class="card-preview">
    <video
      v-if="data.isVideo"
      :playsinline="true"
      :autoplay="true"
      :loop="true"
      :muted="true"
    >
      <source :src="data.preview">
    </video>
    <img v-else :src="data.preview" :alt="title">
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'nuxt-property-decorator'
import { IMediaModel } from '~/models'

@Component
export default class ItemPreview extends Vue {
  @Prop() readonly data!: IMediaModel
  @Prop(String) readonly title: string | undefined
}
</script>

<style lang="scss" scoped>
.card {
  @apply tw-space-y-4;
  &-preview {
    @apply tw-rounded tw-overflow-hidden tw-aspect-w-1 tw-aspect-h-1;
    > * {
      @apply tw-object-cover tw-object-center;
    }
  }
}
</style>
