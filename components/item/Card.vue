<template>
  <div class="card">
    <item-preview :title="data.title" :data="data.file" />
    <div class="card-body">
      <div class="card-title" :title="data.title">
        {{ data.title }}
      </div>
      <p class="card-description" :title="data.description">
        {{ data.description }}
      </p>
      <div class="card-collections">
        <div
          v-for="collection in data.collections"
          :key="collection.id"
          class="card-collections-item"
        >
          <div class="card-collections-avatar">
            <img :src="collection.avatar.preview" :alt="collection.title">
          </div>
          <h4>{{ collection.title }}</h4>
          <p v-if="collection.description">
            {{ collection.description }}
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'nuxt-property-decorator'
import { IItemModel } from '~/models'

@Component
export default class ItemCard extends Vue {
  @Prop() readonly data!: IItemModel
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
  &-body {
    @apply tw-space-y-2;
  }
  &-title {
    @apply tw-h4 tw-line-clamp-2;
  }
  &-description {
    @apply tw-text-base tw-text-sm tw-line-clamp-2;
  }
}
</style>
