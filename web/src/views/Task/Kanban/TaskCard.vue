<template>
  <div class="task-card" @dragstart="tryDrag" :draggable="!canDrag">
    <div class="item-input-cloak" v-if="isInputtingNewCard" @click.self="cancel"></div>
    <div class="item-input" v-show="isInputtingNewCard">
      <div class="title-editable-placeholder" v-if="isShowingPlaceholder" @click="focusTitleEditable">
        Enter a title for this card…
      </div>
      <div class="title-editable" ref="titleEditable" contenteditable @keypress.enter.prevent="save" @keyup="calculateShowPlaceholder"
        @input="titleBackbone = $event.target.innerText" @paste="handlePaste">{{itemCopy.title}}</div>
      <div class="item-actions">
        <button class="btn btn-link" @click="cancel">
          <legacy-icon name="close" size="1.5rem" title="Close"/>
        </button>
        <button v-if="isInputtingNewCard" class="btn btn-primary" @click="save" :disabled="!canSave">Add Card</button>
      </div>
    </div>
    <div v-if="!isInputtingNewCard" class="card" @click="openModal()">
      <div class="card-item">
        <div class="header">
          <div class="title" ref="cardTitle">
            {{item.title}}
          </div>
          <div class="date" v-if="itemCopy.dueDate" :content="formatDateReadable(itemCopy.dueDate)" v-tippy>
            {{ formatDate(itemCopy.dueDate) }}
          </div>
        </div>
        <div class="footer" v-if="isHasFooter(item)" :class="{ 'tags-margin': !isTitleMoreThanOneLine}">
          <div class="tags" ref="tagLists">
            <ul>
              <li v-for="(tag, index) in itemCopy.tags" :key="index">
                <div :style="{background: tag.color, color: textColor(tag.color)}" class="tag">
                  {{ tag.label }}
                </div>
              </li>
              <li v-if="item.attachments && item.attachments.length > 0">
                <span class="icon">
                  <legacy-icon name="attachment" viewbox="20" size="20px" :withTitle="false" content="Attachment(s)" v-tippy/>
                </span>
              </li>
              <li v-if="item.taskComments.length > 0">
                <span class="icon">
                  <legacy-icon name="comment" viewbox="20" size="20px" :withTitle="false" content="Comment(s)" v-tippy/>
                </span>
              </li>
            </ul>
          </div>
          <ul class="assignees" v-if="item.assignees && item.assignees.length > 0" :class="{ 'assignees-margin' : isTagMoreThanOneLine }">
            <li v-for="(assignee, index) in item.assignees.slice(0, 10)" :key="index" class="assignee cursor-pointer" @click.stop="openProfile(assignee)">
              <avatar :size="28" :src="assignee.avatar && assignee.avatar.versions ? assignee.avatar.versions.default.location : ''" :content="memberName(assignee)" :username="memberName(assignee)" v-tippy></avatar>
            </li>
            <li class="assignee more-assignee" v-if="hasMoreAssignee">
              <avatar :content="`${countMoreAssignee} More`" :username="`+ ${countMoreAssignee}`" v-tippy></avatar>
            </li>
          </ul>
        </div>
      </div>
    </div>
    <TaskModal v-if="showModal" @close="closeModal" :item="item" :visible="showModal" :archivedView="archivedView"></TaskModal>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Inject, InjectReactive, Prop, Ref, Vue, Watch } from 'vue-property-decorator'
import { TaskBoardResource, TaskItemResource, UserResource } from '@/types/resource'
import { required } from 'vuelidate/lib/validators'
import { Optional } from '@/types/core'
import TaskModal from '@/views/Task/TaskModal.vue'
import moment from 'moment'
import Avatar from 'vue-avatar'
import * as Y from 'yjs'
import { ModalInjectedContext, ProfileModal } from '@/components/modal'
import { ArchivedViewKey, ClientID, FilteredKey, TaskId, YDoc } from '../injectionKeys'

@Component({
  name: 'TaskCard',
  components: {
    TaskModal,
    Avatar
  },
  validations: {
    titleBackbone: { required }
  }
})
export default class TaskCard extends Vue {
    @Prop({ type: Object, required: true })
    private readonly item!: Optional<TaskItemResource, 'updatedAt' | 'createdAt' | 'userId'>

    @Prop({ type: Boolean, default: false })
    private readonly defaultInputting!: boolean

    @Prop({ type: Boolean, default: false })
    private readonly drag!: boolean

    @Prop({ type: Boolean, default: true })
    private readonly canDrag!: boolean

    @Ref('titleEditable')
    private readonly titleEditableRef!: HTMLDivElement;

    @Ref('tagLists')
    private readonly tagListsRef!: HTMLDivElement;

    @Ref('cardTitle')
    private readonly cardTitleRef!: HTMLDivElement;

    @Inject('modal')
    readonly modal!: ModalInjectedContext

    @InjectReactive(FilteredKey)
    private readonly boardFiltered!: boolean

    @InjectReactive(ArchivedViewKey)
    private archivedView!: boolean

    @InjectReactive(YDoc)
    private readonly doc!: Y.Map<any>

    @InjectReactive(TaskId)
    private readonly taskId!: string

    @InjectReactive(ClientID)
    private readonly clientId!: Number

    @Watch('drag')
    private watchDrag () {
      if (this.drag) {
        this.dragged = true
      } else {
        setTimeout(() => {
          this.dragged = false
        })
      }
    }

    private isInputting = this.defaultInputting
    private itemCopy: Optional<TaskItemResource, 'updatedAt' | 'createdAt' | 'userId'> = { ...this.item }
    private showModal = false
    private isShowingPlaceholder = true
    private isTagMoreThanOneLine = false
    private isTitleMoreThanOneLine = false
    private titleBackbone = ''
    private dragged = false

    private get isProcessing () {
      return this.$store.state.task.item.processing
    }

    private get isInputtingNewCard () {
      return this.isInputting && this.itemCopy.id === null
    }

    private get canSave () {
      return !this.$v.$invalid && !this.isProcessing
    }

    tryDrag (e: DragEvent) {
      if (this.boardFiltered) {
        e.preventDefault()
        e.stopPropagation()

        this.$toast.error('Board edit is disabled')
      }
    }

    async save () {
      if (!this.canSave) {
        return
      }

      // create new task item
      if (this.itemCopy.id === null) {
        this.titleEditableRef.blur()
        const newItem = await this.$store.dispatch('task/item/create', { ...this.itemCopy, title: this.titleEditableRef.innerText.trim(), list: undefined })

        this.transact({
          ...newItem,
          title: this.titleEditableRef.innerText.trim(),
          list: undefined,
          action: 'createNewTaskItem'
        })
      } else {
        await this.$store.dispatch('task/item/update', {
          id: this.item.id,
          title: this.itemCopy.title
        })

        this.transact({
          ...this.itemCopy,
          id: this.item.id,
          title: this.itemCopy.title,
          action: 'updateTaskItem'
        })
      }
      this.isInputting = false
      this.$emit('save', this.itemCopy)
      return this.itemCopy
    }

    private transact (data: any) {
      this.doc.doc.transact(() => {
        this.doc.set(this.taskId, {
          ...data,
          clientId: this.clientId
        })
      }, this.clientId)
    }

    @Emit('cancel')
    cancel () {
      this.itemCopy = { ...this.item }
      this.isInputting = false
      return true
    }

    get board (): TaskBoardResource | null {
      return this.$store.state.task.board.current
    }

    get hasMoreAssignee (): boolean {
      return this.item.assignees ? this.item.assignees.length > 10 : false
    }

    get countMoreAssignee (): number {
      return this.item.assignees ? this.item.assignees.length - 10 : 0
    }

    openModal () {
      // To resolve firefox issue. Modal opened after the card dragged.
      if (this.dragged) {
        return
      }

      if (this.board?.id && this.item.id) {
        this.$router.push({
          name: 'TaskPageWithItem',
          params: {
            id: this.board.id.toString(),
            item: this.item.id.toString(),
            slug: this.item.slug || '-'
          }
        })
        this.$store.commit('task/item/setCurrent', this.item)
        this.showModal = true
      }
    }

    closeModal (params: { visibilityOnly?: boolean }) {
      this.showModal = false

      if (params?.visibilityOnly) {
        return
      }

      this.$store.commit('task/item/setCurrent', null)
      if (this.board?.id && this.item.id) {
        this.$router.replace({
          name: 'TaskPage',
          params: {
            id: this.board.id.toString()
          }
        })
      }
    }

    formatDate (taskDate: Date | string) {
      return moment(taskDate).format('DD.MM.YYYY')
    }

    formatDateReadable (taskDate: Date | string) {
      return moment(taskDate).format('MMMM Do, YYYY')
    }

    memberName (member: UserResource) {
      return `${member.firstName} ${member.lastName}`
    }

    isHasFooter (itemCopy: Optional<TaskItemResource, 'updatedAt' | 'createdAt' | 'userId'>) {
      return (itemCopy.tags && itemCopy.tags.length > 0) ||
        (itemCopy.attachments && itemCopy.attachments.length > 0) ||
      (itemCopy.taskComments && itemCopy.taskComments.length > 0) ||
      (itemCopy.assignees && itemCopy.assignees.length > 0)
    }

    get colors () {
      return ['#DEFFD9', '#FFE8E8', '#FFEAD2', '#DBF8FF', '#F6DDFF', '#FFF2CC', '#FFDDF1', '#DFE7FF', '#D5D1FF', '#D2E4FF']
    }

    textColor (bgColor: string) {
      const textColor = ['#3A932C', '#C94747', '#DD8435', '#588f9c', '#9C3DBF', '#8c7940', '#883b68', '#394c84', '#47408c', '#2D6FD6']
      const getBgPosition = this.colors.indexOf(bgColor)

      return textColor[getBgPosition]
    }

    @Watch('item')
    updateItem (val: Optional<TaskItemResource, 'updatedAt' | 'createdAt' | 'userId'>) {
      this.itemCopy = val

      if (val.tags) {
        Vue.nextTick(() => {
          this.isTagMoreThanOneLine = false
          this.isTitleMoreThanOneLine = false

          if (this.tagListsRef && this.tagListsRef.clientHeight > 35) {
            this.isTagMoreThanOneLine = true
          }

          if (this.cardTitleRef && this.cardTitleRef.clientHeight > 25) {
            this.isTitleMoreThanOneLine = true
          }
        })
      }
    }

    mounted () {
      if (this.$route.name === 'TaskPageWithItem') {
        const itemId = parseInt(this.$route.params.item)
        if (itemId === this.item.id) {
          this.$store.commit('task/item/setCurrent', this.item)
          this.showModal = true
        }
      }

      Vue.nextTick(() => {
        if (this.tagListsRef && this.tagListsRef.clientHeight > 35) {
          this.isTagMoreThanOneLine = true
        }

        if (this.cardTitleRef && this.cardTitleRef.clientHeight > 25) {
          this.isTitleMoreThanOneLine = true
        }
      })
    }

    @Watch('$route')
    watchRoute () {
      if (this.$route.name === 'TaskPageWithItem') {
        const itemId = parseInt(this.$route.params.item)
        if (itemId === this.item.id) {
          this.$store.commit('task/item/setCurrent', this.item)
          this.showModal = true
        }
      }
    }

    created () {
      Vue.nextTick(() => {
        if (this.isInputtingNewCard) {
          this.focusTitleEditable()
        }
      })
    }

    focusTitleEditable () {
      if (this.titleEditableRef) {
        this.titleEditableRef.focus()
      }
    }

    calculateShowPlaceholder () {
      if (this.titleEditableRef) {
        this.isShowingPlaceholder = this.titleEditableRef.innerText.trim().length <= 0
      }
    }

    handlePaste (e: ClipboardEvent) {
      e.preventDefault()
      const data = e.clipboardData?.getData('text/plain')
      if (data && this.titleEditableRef) {
        document.execCommand('insertText', false, data)
        this.titleBackbone = data
      }
    }

    openProfile (user: UserResource) {
      this.modal.open({
        component: ProfileModal,
        attrs: {
          userId: user.id
        }
      })
    }
}
</script>

<style lang="postcss" scoped>
  .task-card {
    @apply relative;
    cursor: pointer;
    &.dragged {
      opacity: 0.5;
    }
    &.overed{
    }
  }

  .task-card ~ .task-card {
    margin-top: 8px;
  }

  .item-input-cloak {
    position: fixed;
    top:0;
    left:0;
    width: 100%;
    height: 100vh;
    z-index: 50;
  }
  .item-input {
    @apply relative;
    z-index: 51;
  }

  .btn-link {
    @apply py-1 px-1;
    cursor: default;
  }

  .btn-link .stroke-current {
    stroke-width: 3px;
    color: theme("colors.gray.400");
  }

  .item-actions {
    @apply flex items-center justify-end mt-2;

    .btn {
      @apply px-4;
      flex: 0 0 auto;
    }
  }

  .item-textarea {
    @apply rounded p-2 w-full text-base;
    border: solid thin theme("colors.gray.100");
  }

  .card {
    @apply p-2 flex items-center rounded;

    margin: 0 10px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
    background: theme("colors.white.default");
    transition: all 0.1s ease;

    &:hover {
       background: #F9FAFF;
       box-shadow: 0px 2px 5px rgba(0, 0, 0, 0.2);
    }
  }

  .card-item {
    @apply ml-2 text-base flex flex-col;
    flex: 1 1 auto;
    width: 0;
    font-weight: 500;
    color: theme("colors.gray.900");

    .header {
      @apply flex justify-between flex-1;

      font-size: 14px;
      line-height: 17px;
      padding: 3px 0;

      .title {
        @apply flex-1;

        word-break: break-word;
        width: 168px;
      }

      .date {
        @apply flex-none rounded py-1 self-start;

        background: #EFF1F6;
        font-size: 10px;
        line-height: 12px;
        letter-spacing: 0.03em;
        text-transform: uppercase;
        padding-left: 6px;
        padding-right: 6px;
        font-weight: bold;
      }
    }

    /* .content {
      @apply pt-2 pb-5;

      color: theme("colors.gray.800");
      font-size: .9rem;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      width: 220px;
    } */

    .footer {
      /* @apply flex justify-between flex-1; */
      @apply block;

      &.tags-margin {
        margin-top: 8px;
      }

      .tags {
        @apply flex-1 flex justify-start;

        float: left;
        margin-top: 3px;

        ul {
          @apply block;

          li {
            display: inline-block;
            margin-top: 4px;
          }

          .tag {
            @apply px-2 py-1 rounded self-start;

            display: inline;
            color: #fff;
            margin-right: 8px;
            font-size: 10px;
            line-height: 12px;
            font-weight: bold;
            letter-spacing: 0.03em;
            text-transform: uppercase;
          }
        }

        /* .actions {
          @apply flex self-start;
        } */

        .icon {
          @apply cursor-auto;

          width: 26px;
          height: 26px;
          display: block;
          color: theme("colors.gray.400");

          svg {
            display: inline;
          }
        }
      }

      .assignees {
        @apply flex flex-wrap justify-start flex-row-reverse;

        float: right;

        .assignee {
          margin-top: 5px;
        }

        li {
          .vue-avatar--wrapper {
            width: 28px !important;
            height: 28px !important;
            font: 10px / 13px theme("fontFamily.primary") !important;
            float: left;
            border: 2px solid #FFF;
            letter-spacing: 0.03em;
            margin-left: -7px;
            color: #fff !important;
            background-position: center center !important;
          }
        }

        .more-assignee {
          .vue-avatar--wrapper {
            background-color: theme("colors.gray.900") !important;
            color: #FFFFFF !important;
          }
        }
      }
    }
  }

  .drag-block {
    @apply p-2 fixed z-50 rounded shadow shadow-lg;
    animation: shake 0.5s ease, fadeOut 0.5s 2.5s ease-out;
    background: theme("colors.danger.default");
    color: #fff;
  }

  @keyframes fadeOut {
    from{
      opacity: 1;
    }
    to {
      opacity: 0;
    }
  }

  .title-editable-placeholder {
    @apply absolute text-base;
    user-select: none;
    z-index: 1;
    top: 10px;
    left: 15px;
    color: theme("colors.gray.400");
    font-weight: 500;
    opacity: 0.75;
  }
  .title-editable {
    @apply p-2 px-3 block text-base items-center rounded;
    background: theme("colors.white.default");
    outline: none;
    font-weight: 500;
    color: theme("colors.gray.900");
    border: 1px solid theme("colors.gray.400");
    white-space: -moz-pre-wrap !important;  /* Mozilla, since 1999 */
    white-space: -pre-wrap;      /* Opera 4-6 */
    white-space: -o-pre-wrap;    /* Opera 7 */
    white-space: pre-wrap;       /* css-3 */
    word-wrap: break-word;       /* Internet Explorer 5.5+ */
    white-space: -webkit-pre-wrap; /* Newer versions of Chrome/Safari*/
    word-break: break-word;
    white-space: normal;
    &:focus {
      border: 1px solid rgba(47, 128, 237, 0.75);
      box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2), 0 0 0 2px rgba(47, 128, 237, 0.25);
    }
  }

  @keyframes shake {
    0% {
      transform: translateX(0%)
    }
    25% {
      transform: translateX(-1%)
    }
    50% {
      transform: translateX(1%)
    }
    75% {
      transform: translateX(-1%)
    }
    100% {
      transform: translateX(0%)
    }
  }

  .assignees-margin {
    margin-top: 13px;
  }
</style>
