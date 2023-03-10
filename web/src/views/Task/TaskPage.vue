<template>
  <section class="board">
    <header
      class="header"
      v-if="board || boardCache"
    >
      <div class="title">
        <h3>{{title}}</h3>
        <div class="pill" v-if="filtered">
          <mono-icon name="eye" class="pill-icon"/> View only
        </div>
      </div>

      <div class="actions">
        <div class="action-group">
          <label
            class="action action--search"
            :class="{ 'action__active': searchVisible }"
            @click="searchVisible = true"
            v-click-outside="closeSearch"
          >
            <mono-icon
              name="search"
              class="action--icon"
            />

            <div
              v-if="searchVisible"
              class="action--body"
            >
              <input
                type="text"
                placeholder="Search"
                v-model="search"
                @input="lazyFetchTask"
                @keydown.esc="clearSearch"
              >

              <button
                v-if="search"
                class="action--search--close"
                @click.stop="clearSearch"
              >
                <mono-icon name="close" />
              </button>
            </div>
          </label>
        </div>
        <div class="action-group">
          <div
            class="action action--filter"
            :class="{ 'action__active': filterVisible }"
            @click="filterVisible = !filterVisible"
          >
            <mono-icon
              name="filter"
              class="action--icon"
            />
            <div class="action--body">
              Filter
            </div>
          </div>

          <div
            class="action action--archive"
            :class="{ 'action__active': filters.archived }"
            @click="toggleArchived"
          >
            <mono-icon
              name="archive"
              class="action--icon"
            />
            <div class="action--body">
              Archived
            </div>
          </div>
        </div>
        <div class="action-group action-group--view">
          <Tip
            :active="shouldShowTip"
            @close="markTipAsSeen"
          >
            <template #tip>
              You can change your view preference between Kanban Board or List using this button
            </template>

            <div
              class="action"
              :class="{ 'action__active': isKanban }"
              @click="viewAsBoard"
            >
              <mono-icon
                name="board"
                class="action--icon"
              />
              <div class="action-body">
                Board
              </div>
            </div>

            <div
              class="action"
              :class="{ 'action__active': !isKanban }"
              @click="viewAsList"
            >
              <mono-icon
                name="menu"
                class="action--icon"
              />
              <div class="action--body">
                List
              </div>
            </div>
          </Tip>
        </div>
      </div>
    </header>

    <filter-bar
      v-if="filterVisible"
      v-model="filters"
      :member-list="memberList"
      :tag-list="tags"
      @input="lazyFetchTask"
    />

    <div :class="isKanban ? 'view--kanban' : 'view--list'">
      <component
        :is="isKanban ? 'BoardManager' : 'ListManager'"
        :loading="firstLoad"
        :can-drag="!filtered"
        :board="board"
      />
    </div>

    <loading :loading="isFetching">
      Loading...
    </loading>
  </section>
</template>

<script lang="ts">
import { Component, Mixins, ProvideReactive, Watch } from 'vue-property-decorator'
import { debounce } from 'helpful-decorators'
import { some } from 'lodash'
import VSelect from 'vue-select'
import Avatar from 'vue-avatar'

import {
  TagResource,
  TaskBoardResource,
  TaskBoardType,
  UserResource
} from '@/types/resource'

import SpaceService from '@/services/space'
import SpaceMixin from '@/mixins/SpaceMixin'
import PageMixin from '@/mixins/PageMixin'
import TaskObserverMixin from '@/mixins/TaskObserverMixin'
import { TaskSettings } from '@/store/modules/task/settings'
import EventBus from '@/utils/eventBus'
import FilterBar from './FilterBar.vue'

import BoardManager from '@/views/Task/Kanban/BoardManager.vue'
import ListManager from '@/views/Task/List/ListManager.vue'
import Popover from '@/components/Popover.vue'
import VField from '@/components/Field.vue'
import ButtonSwitch from '@/components/ButtonSwitch.vue'
import Tip from '@/components/Tip.vue'
import Loading from '@/components/Loading.vue'

import { FilteredKey, ArchivedViewKey, TaskId, YDoc, ClientID } from './injectionKeys'
import * as encoding from 'lib0/encoding.js'
import * as decoding from 'lib0/decoding.js'
import { WebsocketProvider } from 'y-websocket'
import * as Y from 'yjs'

const wsMessageType = {
  authenticate: 10,
  unauthenticated: 11,
  unauthorized: 12,
  wait: 13,
  initCollaboration: 14,
  restore: 15
}

@Component({
  name: 'TaskPage',
  components: {
    ListManager,
    Tip,
    BoardManager,
    Popover,
    VSelect,
    VField,
    ButtonSwitch,
    Avatar,
    FilterBar,
    Loading
  }
})
export default class TaskPage extends Mixins(SpaceMixin, PageMixin, TaskObserverMixin) {
  private search = ''
  private filters = {
    tags: [],
    assignees: [],
    unassigned: false,
    archived: false
  }

  private boardCache: TaskBoardResource | null = null
  private memberList: Array<UserResource> = []
  private firstLoad = true
  private isFetching = false
  private filterVisible = false
  private searchVisible = false
  private provider?: WebsocketProvider
  private ydoc = new Y.Doc()

  @ProvideReactive(FilteredKey)
  get filtered () {
    return some([
      this.search.length,
      this.filters.tags.length,
      this.filters.assignees.length,
      this.filters.archived
    ])
  }

  @ProvideReactive(ArchivedViewKey)
  get archivedView () {
    return this.filters.archived
  }

  @ProvideReactive(YDoc)
  get doc (): Y.Map<any> {
    if (!this.ydoc) {
      this.ydoc = new Y.Doc()
    }

    const doc = this.ydoc.getMap(this.taskId)

    return doc
  }

  @ProvideReactive(ClientID)
  get clientID (): Number {
    return this.currentUser().id
  }

  @ProvideReactive(TaskId)
  get taskId (): string {
    return 'task_' + this.boardId
  }

  private currentUser () {
    return this.$store.state.auth.user
  }

  get title (): string {
    if (this.board) {
      return this.board.title
    } else if (this.boardCache) {
      return this.boardCache.title
    } else {
      return 'Untitled'
    }
  }

  get tags (): TagResource[] | null {
    return this.$store.state.task.tag.data
  }

  get board (): TaskBoardResource | null {
    return this.$store.state.task.board.current
  }

  get isKanban (): boolean {
    return this.prefferedView === TaskBoardType.Kanban
  }

  get boardId (): number {
    return +(this.$route.params.id)
  }

  get prefferedView (): TaskBoardType {
    if (!this.$store.state.task.settings.viewAs[this.boardId]) {
      return this.board?.type || TaskBoardType.Kanban
    }
    return this.$store.state.task.settings.viewAs[this.boardId] ?? TaskBoardType.Kanban
  }

  get shouldShowTip (): boolean {
    return !this.$store.state.task.settings.seenViewTip
  }

  @Watch('boardId', { immediate: true })
  async load () {
    await this.getSpaceMember()
    await this.fetchOnSwitchBoard()
    EventBus.$on('BUS_TASKBOARD_UPDATE', this.syncTaskAttr)

    this.connectWebsocket()
    this.observeBoard()
  }

  async fetchOnSwitchBoard () {
    this.firstLoad = true

    this.searchVisible = false
    this.filterVisible = false
    this.search = ''
    this.filters = {
      tags: [],
      assignees: [],
      unassigned: false,
      archived: false
    }
    await this.fetchTask()
  }

  async getSpaceMember () {
    try {
      const id = this.activeSpace.id
      const viewUserAtSpace = await SpaceService.spaceUsers(id)

      this.memberList = viewUserAtSpace.data
    } catch { }
  }

  viewAsList () {
    this.$store.commit('task/settings/setData', (state: TaskSettings) => {
      if (!isNaN(state.viewAs as any)) {
        state.viewAs = {}
      }

      state.viewAs = {
        ...state.viewAs,
        [this.boardId]: TaskBoardType.List
      }
    })
  }

  markTipAsSeen () {
    this.$store.commit('task/settings/setData', (state: TaskSettings) => {
      state.seenViewTip = true
    })
  }

  viewAsBoard () {
    this.$store.commit('task/settings/setData', (state: TaskSettings) => {
      if (!isNaN(state.viewAs as any)) {
        state.viewAs = {}
      }

      state.viewAs = {
        ...state.viewAs,
        [this.boardId]: TaskBoardType.Kanban
      }
    })
  }

  clearSearch () {
    this.search = ''
    this.fetchTask()
  }

  closeSearch () {
    if (this.search) return

    this.searchVisible = false
  }

  idExistsOn (array: number[], id: number) {
    return array.findIndex(v => v === id) !== -1
  }

  removeTag (tag: TagResource) {
    this.filters.tags = this.filters.tags.filter((t: number) => t !== tag.id)
  }

  removeMember (member: UserResource) {
    this.filters.assignees = this.filters.assignees.filter((m: number) => m !== member.id)
  }

  async fetchTask () {
    this.isFetching = true

    try {
      await this.$store.dispatch('task/board/search', { boardId: this.boardId, search: this.search, filters: this.filters })
      await this.$store.dispatch('task/tag/fetch', null)

      if (this.board) {
        this.boardCache = this.board

        if (!this.pageReady) {
          await this.activateSpace(this.board.spaceId)
        }

        this.pageTitle = this.board.title
        this.pageReady = true
      }
    } catch { }

    this.isFetching = false
    this.firstLoad = false
  }

  @debounce(100)
  async lazyFetchTask () {
    await this.fetchTask()
  }

  async resetFilters () {
    this.filters.assignees = []
    this.filters.tags = []
    await this.fetchTask()
  }

  filterMember (option: UserResource, label: string, search: string) {
    const rx = new RegExp(search, 'gi')
    return rx.test(option.firstName) || rx.test(option.lastName)
  }

  syncTaskAttr (payload: any) {
    this.pageTitle = payload.title

    if (this.board) {
      this.board.title = payload.title
    }

    if (this.boardCache) {
      this.boardCache.title = payload.title
    }
  }

  get colors () {
    return ['#DEFFD9', '#FFE8E8', '#FFEAD2', '#DBF8FF', '#F6DDFF', '#FFF2CC', '#FFDDF1', '#DFE7FF', '#D5D1FF', '#D2E4FF']
  }

  beforeDestroy () {
    this.ydoc.destroy()

    EventBus.$off('BUS_TASKBOARD_UPDATE', this.syncTaskAttr)
  }

  textColor (bgColor: string) {
    const textColor = ['#3A932C', '#C94747', '#DD8435', '#588f9c', '#9C3DBF', '#8c7940', '#883b68', '#394c84', '#47408c', '#2D6FD6']
    const getBgPosition = this.colors.indexOf(bgColor)

    return textColor[getBgPosition]
  }

  async toggleArchived () {
    this.filters.archived = !this.filters.archived

    await this.lazyFetchTask()
  }

  async observeBoard () {
    this.ydoc.on('update', (event: Uint8Array, origin: any, ydoc: Y.Doc) => {
      const doc = ydoc.getMap(this.taskId)
      const data = doc.toJSON()

      if (origin !== this.clientID) {
        this.doTransact(data)
        Y.applyUpdate(ydoc, event)
      }
    })
  }

  private async doTransact (data: any) {
    if (data?.[this.taskId]) {
      const newData = data[this.taskId]

      // only operate mutation on peers, so the broadcaster don't have to do mutation again
      switch (newData.action) {
        case 'addedToLane':
        case 'movedToLane':
        case 'updateListItem':
        case 'updateTaskItem':
        case 'updateTaskItemTitle':
        case 'updateTaskItemDescription':
        case 'addDueDate':
        case 'removeDueDate':
        case 'uploadFile':
        case 'deleteUploadFile':
          await this.updateTaskItem(newData)
          break
        case 'listLaneMoved':
        case 'taskLaneMoved':
          await this.$store.dispatch('task/list/update', { ...newData })
          break
        case 'createNewListItem':
        case 'createNewTaskItem':
          await this.createTaskItem(newData)
          break
        case 'createComment':
          await this.createComment(newData)
          break
        case 'archiveTaskItem':
          await this.archiveTaskItem(newData)
          break
        case 'restoreTaskItem':
          await this.restoreTaskItem(newData)
          break
        case 'addAssigneeToTask':
          await this.addAssignee(newData)
          break
        case 'removeAssigneeFromTask':
          await this.removeAssignee(newData)
          break
        case 'addTagToTask':
          await this.addTagToTask(newData)
          break
        case 'removeTagFromTask':
          await this.removeTagFromTask(newData)
          break
        case 'createListLane':
        case 'createTaskLane':
          await this.createTaskLane(newData)
          break
        case 'updateListLane':
        case 'updateTaskLane':
        case 'updateColorListLane':
        case 'updateColorTaskLane':
          await this.updateTaskLane(newData)
          break
        case 'archiveListLane':
        case 'archiveTaskLane':
          await this.archiveTaskLane(newData)
          break
        case 'createTag':
          await this.createTag(newData)
          break
        case 'updateTag':
          await this.updateTag(newData)
          break
        default:
          break
      }
    }
  }

  private connectWebsocket () {
    const websocketUrl = process.env.VUE_APP_YWS_URL

    if (!websocketUrl) {
      throw new Error('process.env.VUE_APP_YWS_URL is missing')
    }

    if (this.provider) {
      this.provider.disconnect()
    }

    this.provider = new WebsocketProvider(websocketUrl, this.taskId, this.ydoc)

    if (!(this.provider && this.provider.ws)) { return }

    const token = this.$store.state.auth.token
    const ws = this.provider.ws
    const providerOnOpen = ws.onopen
    const providerOnMessage = ws.onmessage

    const authenticate = () => {
      const encoder = encoding.createEncoder()
      encoding.writeVarUint(encoder, wsMessageType.authenticate)
      encoding.writeVarString(encoder, token)

      ws.send(encoding.toUint8Array(encoder))
    }

    this.provider.ws.onopen = (event) => {
      authenticate()

      if (providerOnOpen) {
        providerOnOpen.call(ws, event)
      }
    }

    this.provider.ws.onmessage = event => {
      const decoder = decoding.createDecoder(new Uint8Array(event.data))
      const messageType = decoding.readVarUint(decoder)

      if (messageType >= 10) {
        return
      }

      if (providerOnMessage) {
        providerOnMessage.call(ws, event)
      }
    }
  }
}
</script>

<style lang="postcss" scoped>
.board {
  @apply flex flex-col h-full;
  flex: 1 1 0;
  width: 0;

  -webkit-touch-callout: none;
    -webkit-user-select: none;
     -khtml-user-select: none;
       -moz-user-select: none;
        -ms-user-select: none;
            user-select: none;
}

.header {
  @apply flex flex-row items-center;
  align-items: center;
  justify-content: space-between;
  margin-left: 72px;
  margin-right: 24px;
  margin-top: 24px;
  margin-bottom: 24px;
  background: #ffffff;
  color: theme("colors.gray.900");
}

.title {
  display: flex;
  align-items: center;

  h3 {
    font-size: 24px;
    font-weight: 700;
  }
}

.pill {
  display: inline-flex;
  align-items: center;
  padding: 0 8px;
  margin-left: 8px;
  border-radius: 8px;
  background: #37D88B;
  color: #ffffff;

  .pill-icon {
    margin-right: 4px;
  }
}

.actions {
  display: flex;
  flex-flow: row;
  align-items: center;
}

.action-group {
  display: flex;
  flex-flow: row;
  padding: 0 8px;

  &:first-child {
    padding-left: 0;
  }

  &:last-child {
    padding-right: 0;
  }
}

.action-group + .action-group {
  border-left: 1px solid #dee2ee;
}

.action-group--view {
  >>> .target {
    display: flex;
    flex-flow: row;
    align-items: center;
  }

  .action__active,
  .action__active:hover {
    background: #444754;
    color: white;
  }
}

.action {
  display: flex;
  flex-flow: row;
  align-items: center;
  padding: 6px 10px;
  border-radius: 4px;
  font-size: 14px;
  line-height: 1;
  cursor: pointer;
  transition: all 300ms;

  &:hover {
    background-color: #f4f5f7;
  }
}

.action + .action {
  margin-left: 4px;
}

.action__active,
.action__active:hover {
  background-color: #ddf3ff;
  color: #146493;
}

.action--icon {
  margin-right: 4px;
  font-size: 20px;
}

.action--search {
  &.action__active {
    width: 304px;
    box-shadow: 0 0 2px 2px #8cd5ff;
    border-radius: 4px;
    background: white;
    color: #444754;
  }

  input {
    width: 100%;
    outline: none;
  }

  .action--body {
    display: flex;
    flex-flow: row;
    position: relative;
    width: 100%;
    z-index: 110;
  }

  .action--search--close {
    display: flex;
    flex: none;
    align-items: center;
    justify-content: center;
    width: 21px;
    height: 21px;
    border-radius: 50%;
    background-color: #F4F5F7;
    outline: none;
    font-size: 12px;
    stroke-width: 2px;
  }
}

.view--kanban {
  flex: 1;
  overflow-x: auto;
  padding-left: 72px;
}

.view--list {
  flex: 1;
  padding-left: 52px;
  padding-bottom: 52px;
}
</style>
