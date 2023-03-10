<template>
  <component
    :is="wrapper.tag"
    v-bind="wrapper.attrs"
    v-on="wrapper.listeners"
    active-class="is-active"
    :class="{
      'tree-node-content': true,
      'is-renaming': isRenaming,
      'is-context-open': isContextOpen,
      'is-touched': touched
    }"
  >
    <div
      class="tree-node-arrow"
      :class="{
        'is-hidden': !hasChildren,
        'is-folded': folded
      }"
      @click.stop.prevent="toggleFold"
    >
      <legacy-icon name="down2" viewbox="16" />
    </div>

    <div class="tree-node-label overflow-hidden" >
      <div class="tree-node-icon">
        <legacy-icon viewbox="16" :name="iconName" />
      </div>
      <label-editable
        class="tree-node-text truncate"
        v-model="title"
        v-if="!isRenaming"
      />
      <input ref="input" type="text" class="field node-input" v-show="isRenaming" placeholder="Node name" :value="title"
        @blur="saveTitle" @keydown.enter="$event.target.blur()" @keydown.esc="isRenaming = false">
    </div>

    <div class="tree-node-actions" v-if="!isRenaming">
      <Popover top="38px" :with-close="false" :borderless="true" @trigger="isContextOpen = $event" v-if="value.contentId !== 0">
        <template #default="{ hide }">
          <div class="action-line" @click.prevent.stop="hide();addNew()">
            <mono-icon class="action-icon" name="plus"/>
            <div class="action-line-text">
              Add new
            </div>
          </div>
          <div v-if="!isFavorited && !isFolder" class="action-line" @click.prevent.stop="hide();addToFavorites();">
            <mono-icon class="action-icon" name="star"/>
            <div class="action-line-text">
              Add to Favorites
            </div>
          </div>
          <div v-if="!isFocusMode" class="action-line" @click.prevent.stop="hide();addToFocusedList();">
            <mono-icon class="action-icon" name="crosshair"></mono-icon>
            <div class="action-line-text">
              Focus View
            </div>
          </div>
          <div v-if="isFavorited" class="action-line" @click.prevent.stop="hide();removeFromFavorites();">
            <mono-icon class="action-icon" name="remove-circle"/>
            <div class="action-line-text">
              Remove from Favorites
            </div>
          </div>
          <div>
            <div class="action-separator"></div>
            <div class="action-line" @click.prevent.stop="hide();rename();">
              <mono-icon class="action-icon" name="pencil"/>
              <div class="action-line-text">
                Rename
              </div>
            </div>
            <div class="action-line" v-if="canEdit" @click.prevent.stop="hide();updateContent();">
              <mono-icon class="action-icon" name="pencil"/>
              <div class="action-line-text">
                Edit
              </div>
            </div>
            <div class="action-line" @click.prevent.stop="hide();archive()">
              <mono-icon class="action-icon" name="archive"/>
              <div class="action-line-text">
                Archive
              </div>
            </div>
          </div>
        </template>
        <template #trigger="{ visible }">
          <button class="btn btn-link" :class="{'btn-link-primary': visible}">
            <legacy-icon name="vertical-ellipsis" viewbox="20" size="20px"/>
          </button>
        </template>
      </Popover>
    </div>
  </component>
</template>

<script lang="ts">
import { Vue, Component, Prop, Watch, Ref } from 'vue-property-decorator'
import { Node } from '@adityapurwa/he-tree-vue'
import { Dictionary, Location } from 'vue-router/types/router'
import LabelEditable from '@/components/LabelEditable.vue'

import {
  NodeResource
} from '@/types/resource'
import Popover from '@/components/Popover.vue'

// Interfaces

interface WrapperConfig {
  tag: string;
  attrs?: object;
  listeners?: object;
}

// Enums

export enum NodeType {
  Link = 'link',
  Doc = 'doc',
  Task = 'taskBoard',
  Folder = 'folder',
  Embed = 'embed',
  Storage = 'storage'
}

export const nodeRouteNames = {
  link: 'Link',
  doc: 'Document',
  novaDoc: 'Novadoc',
  taskBoard: 'TaskPage',
  embed: 'Embed',
  storage: 'Storage',
  folder: ''
} as const

export const nodeIconNames = {
  link: 'link2',
  doc: 'file-empty',
  taskBoard: 'layout',
  embed: 'code',
  folder: 'folder-closed',
  archive: 'folder-closed',
  storage: 'file-empty'
} as const

// Types

type NodeData = Node & NodeResource

@Component({
  name: 'SidebarTreeNode',
  components: {
    LabelEditable,
    Popover
  }
})
export default class SidebarTreeNode extends Vue {
  // Props

  @Prop(Object)
  private readonly value!: NodeData

  @Prop(Array)
  private readonly path!: number[]

  @Prop(Boolean)
  private readonly editable!: boolean

  @Prop({ type: Boolean, default: false })
  private readonly hideSecondaryMenu!: boolean

  @Ref('input')
  private readonly inputRef!: HTMLInputElement;

  // State

  private payload: Partial<NodeData> = {}
  private isRenaming = false

  private isContextOpen = false

  // Computed

  get type (): NodeType {
    return this.payload.type as NodeType
  }

  get touched (): boolean {
    return this.$store.state.tree.touched[this.path.join('.')]
  }

  get title (): string {
    if (this.payload.title && this.payload.title.charCodeAt(0) === 1 && this.payload.title.charCodeAt(1) === 2) {
      return 'Untitled'
    }
    return this.payload.title || 'Untitled'
  }

  set title (title: string) {
    this.payload.title = title
  }

  get isFolder (): boolean {
    return this.type === NodeType.Folder
  }

  get isFocusMode (): boolean {
    const tree = this.$store.state.tree
    return tree.focusedList.length ? tree.focusedList[0].id === this.payload.id : false
  }

  get iconName (): string {
    if (!this.folded && this.type === NodeType.Folder) {
      return 'folder-open'
    }
    return nodeIconNames[this.type]
  }

  get to (): Location {
    const { contentId } = this.payload

    let name = nodeRouteNames[this.type] as string
    if (this.payload.type === 'doc') {
      name = 'Novadoc'
    }
    const params: Dictionary<string> = {}

    if (contentId) {
      params.id = contentId.toString()
    }

    return { name, params }
  }

  get wrapper (): WrapperConfig {
    switch (this.type) {
      case NodeType.Folder:
        return {
          tag: 'span',
          listeners: {
            click: this.toggleFold
          }
        }

      default:
        return {
          tag: 'router-link',
          attrs: {
            to: this.to,
            activeClass: 'is-active'
          }
        }
    }
  }

  get hasChildren () {
    const { children } = this.payload

    return children && children.length > 0
  }

  get isFavorited (): boolean {
    return this.$store.getters['tree/isFavorited'](this.value)
  }

  get folded (): boolean {
    const index = this.path.join('.')

    return this.$store.state.tree.folded[index] === true
  }

  get canEdit () {
    const blacklist = [NodeType.Doc, NodeType.Folder]

    return !blacklist.includes(this.type)
  }

  // Methods

  update () {
    this.$emit('node:update', this.path, this.payload)
  }

  saveTitle () {
    const title = this.inputRef.value
    if (title.trim().length === 0) {
      this.isRenaming = false
    } else {
      this.payload = {
        ...this.payload,
        title: this.inputRef.value
      }
      this.update()
      this.isRenaming = false
    }
  }

  addNew () {
    this.$emit('node:addNew', this.path, this.payload)
  }

  rename () {
    this.isRenaming = true
    Vue.nextTick(() => {
      this.inputRef.focus()
    })
  }

  archive () {
    this.$emit('node:archive', this.path, this.payload)
  }

  addToFavorites () {
    this.$emit('node:addToFavorites', this.path, this.payload)
  }

  addToFocusedList () {
    this.$emit('node:addToFocusedList', this.payload)
  }

  removeFromFavorites () {
    this.$emit('node:removeFromFavorites', this.path, this.payload)
  }

  updateContent () {
    this.$emit('content:update', this.path, this.payload)
  }

  toggleFold () {
    this.$emit('node:fold:toggle', this.path)
  }

  // Watchers

  @Watch('value', { immediate: true })
  watchValue (value: NodeData) {
    this.payload = { ...value }
  }
}
</script>
<style lang="postcss" scoped>
.tree-node-icon .fill-current {
  fill: transparent;
}
.tree-node-content .btn-link {
  padding: 8px;
  height: auto;
  transition: none;
}
.tree-node-content.is-renaming {
  padding-top: 0;
  padding-bottom: 0;
}
.action-icon.no-fill {
    fill: transparent;
}
.action-line {
  @apply flex items-center py-2 px-4 my-1 relative;
  min-width: 210px;
  font-weight: 500;
  font-size: 14px;
  line-height: 17px;
  color: #2C2B35;
  cursor: pointer;
  .action-icon {
    font-size: 16px;
    color: #AAB1C5;
  }

  &:hover{
    background: #F0F2F5;
    .action-icon {
      color: #2C2B35;
    }
  }
  &.danger {
    color: theme("colors.danger.default");
  }
}

.action-line-text {
  @apply ml-2;
  flex: 1 1 auto;
}
.action-separator{
  @apply my-1;
  height:1px;
  background: theme("colors.gray.100");
}
.action-arrow{
  @apply ml-2;
  color: theme("colors.gray.400");
}

.node-input {
  outline: none;
  padding: 5px 4px 6px 4px;
  margin: 4px 4px 4px -4px;
  font-weight: 500;
  line-height: 0;
  color: #2C2B35;
  width: 100%;
  box-sizing: border-box;
  border-radius: 4px;
  border: 1px solid #D83750;
  height: 32px;
  &:hover{
    border: 1px solid #D83750;
  }
  &:focus, &:active {
    border: 1px solid #D83750;
    box-shadow: 0 0 0 2px #F9DFE3;
  }
}
.btn-link {
  background-color: unset !important;
  padding: 0;
  height: 20px;

  &:hover {
    background-color: unset;

    .stroke-current {
      color: theme("colors.primary.default");
    }
  }
}
.is-touched {
  animation: touch-anim 1.3s ease;
}
@keyframes touch-anim{
  from{
    background: #D8375033;
  }
  to{
    background: #D8375000;
  }
}
</style>
