<template>
  <div>
    <a class="add-list  mb-4" @click="$emit('link')">
      <legacy-icon
        name="link"
        size="3.5em"
        class="icon-list"
      />

      <div class="desc-list">
        <h5>Link</h5>
        <p>ADD A LINK TO ANYTHING YOU’D LIKE.</p>
      </div>
    </a>

    <a class="add-list mb-4" @click="$emit('document')">
      <legacy-icon
        name="file"
        size="3.5em"
        class="icon-list"
      />

      <div class="desc-list">
        <h5>Document</h5>
        <p>WRITE NOTES, THOUGHTS OR IDEAS.</p>
      </div>
    </a>

    <a class="add-list" @click="$emit('task')">
      <legacy-icon
        name="edit"
        size="3.5em"
        class="icon-list"
      />

      <div class="desc-list">
        <h5>Task</h5>
        <p>CREATE BOARD, LIST OR IDEAS.</p>
      </div>
    </a>
  </div>
</template>

<script lang="ts">

import { Component, Vue } from 'vue-property-decorator'

@Component({
  name: 'SelectLinkType'
})
export default class SelectLinkType extends Vue {
}
</script>

<style lang="postcss" scoped>
.add-list {
  @apply flex flex-row border p-3 rounded;

  border-color: theme("colors.gray.400");
  cursor: pointer;

  &:hover {
    background-color: rgba(theme('colors.gray.100'), 0.5);
  }

  .icon-list {
    @apply rounded-full text-white p-2;

    background: theme("colors.primary.default");
  }
  .desc-list {
    @apply pl-2;

    font-size: 12px;
  }
}
</style>
