<template>
  <div
    class="field"
    :class="{
      'is-inline': inline,
      'is-bordered': border,
      'is-disabled': disabled,
      'has-addon': $slots.addon
    }"
  >
    <label
      v-if="label"
      v-text="label"
      class="label"
      :for="name"
    />

    <div
      class="control"
      :class="{
        'has-icon-left': hasIconLeft,
        'has-icon-right': hasIconRight,
        'justify-start': align === 'left',
        'justify-end': align === 'right',
        'justify-center': align === 'center'
      }"
    >
      <slot />
    </div>

    <div v-if="$slots.addon" class="control-addon">
      <slot name="addon" />
    </div>

    <slot name="feedback" />
  </div>
</template>

<script lang="ts">

import { Component, Prop, Vue } from 'vue-property-decorator'

@Component({
  name: 'Field'
})
export default class Field extends Vue {
  @Prop({ type: String })
  private readonly label?: string;

  @Prop({ type: String, default: '' })
  private readonly name?: string;

  @Prop({ type: Boolean })
  private readonly inline?: boolean;

  @Prop({ type: Boolean })
  private readonly border?: boolean;

  @Prop({ type: Boolean })
  private readonly disabled?: boolean;

  @Prop({ type: String })
  private readonly align?: string;

  @Prop({ type: Boolean })
  private readonly hasIconLeft?: boolean;

  @Prop({ type: Boolean })
  private readonly hasIconRight?: boolean;
}
</script>
