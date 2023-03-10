<template>
  <div class="doc-history">
    <header class="header">
      <h4 class="title">
        History
      </h4>
      <div class="close">
        <button class="btn btn-icon" @click="$emit('close')">
          <legacy-icon name="close2" size="20px" viewbox="20" title="Close"/>
        </button>
      </div>
    </header>
    <HistoryGhost active v-if="loading"></HistoryGhost>
    <main class="content" v-if="history && !loading">
      <div class="history-entry" :key="'current'"
           :class="{current: !preview || !preview.id}"
           @click="$emit('preview', null)" v-if="doc">
        <div class="entry-left">
          <div class="entry-metadata">
            <time class="datetime" :title="doc.contentUpdatedAt">{{ doc.contentUpdatedAt | formatSimpleDateTime }}
            </time>
            <div class="current">Current Version</div>
          </div>
          <div class="entry-author">
            <div class="author-avatar">
              <avatar :size="24"
                      :src="currentUser.avatar && currentUser.avatar.versions ? currentUser.avatar.versions.default.location : ''"
                      :username="`${currentUser.firstName} ${currentUser.lastName}`"></avatar>
            </div>
            <div class="author-name">
              {{ currentUser.firstName }} {{ currentUser.lastName }}
            </div>
          </div>
        </div>
      </div>

      <div class="history-entry" v-for="(entry) in history" :key="entry.id"
           :class="{current: preview && preview.id === entry.id}"
           @click="$emit('preview', entry)">
        <div class="entry-left">
          <div class="entry-metadata">
            <time class="datetime" :title="entry.revisionAt">{{ entry.revisionAt | formatSimpleDateTime }}</time>
          </div>
          <div class="entry-author">
            <div class="author-avatar">
              <avatar :size="24"
                      :src="entry.revisionUser.avatar && entry.revisionUser.avatar.versions ? entry.revisionUser.avatar.versions.default.location : ''"
                      :username="`${entry.revisionUser.firstName} ${entry.revisionUser.lastName}`"></avatar>
            </div>
            <div class="author-name">
              {{ entry.revisionUser.firstName }} {{ entry.revisionUser.lastName }}
            </div>
          </div>
        </div>
        <div class="entry-right">
          <div v-if="!isReadonly" class="restore">
            <button class="btn btn-icon" @click="$emit('restore', entry)" v-tippy="{ placement : 'top',  arrow: true }"
                    content="Restore">
              <legacy-icon name="restore" size="16px" viewbox="16" title="Restore"/>
            </button>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator'
import Avatar from 'vue-avatar'
import DocumentService from '@/services/document'
import { DocRevisionResource, DocumentResource, UserResource } from '@/types/resource'
import { formatAsSimpleDateTime } from '@/utils/date'
import Ghost from '@/components/Ghost.vue'
import HistoryGhost from '@/components/HistoryGhost.vue'

@Component({
  components: {
    HistoryGhost,
    Ghost,
    Avatar
  },
  filters: {
    formatSimpleDateTime (date: string | Date) {
      return formatAsSimpleDateTime(date instanceof Date ? date : new Date(date), true)
    }
  }
})
export default class DocHistory extends Vue {
  @Prop({ type: Number, required: true })
  private readonly id!: number;

  @Prop({ type: Object, required: false })
  private readonly doc!: DocumentResource;

  @Prop({ type: Boolean, required: false, default: false })
  private readonly isReadonly!: boolean;

  @Prop(Object)
  private readonly preview?: DocRevisionResource;

  private loading = false;

  get currentUser (): UserResource {
    return this.$store.state.auth.user
  }

  private history: DocRevisionResource[] = [];

  async mounted () {
    await this.loadHistory()
  }

  public async refresh () {
    await this.loadHistory()
  }

  private async loadHistory () {
    this.loading = true
    try {
      if (this.id) {
        const res = await DocumentService.history(this.id)
        this.history = res.data
      }
    } catch (e) {
      // Error loading history
    }
    this.loading = false
  }
}
</script>

<style lang="postcss" scoped>
.doc-history {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.header {
  padding: 24px 16px;
  display: flex;
  align-items: center;
  flex: 0 0 auto;
}

.content {
  flex: 1 1 auto;
  overflow-y: auto;
}

.title {
  margin: 0;
  font-weight: bold;
  font-size: 18px;
  line-height: 21px;
  color: #2C2B35;
  flex: 1 1 auto;
  margin-right: 16px;
}

.close {
  .btn-icon {
    background: transparent;
    color: #444754;
    padding: 8px;
    height: auto;

    &:hover {
      background: #eee;
      color: #444754;
    }
  }
}

.restore {
  .btn-icon {
    background: transparent;
    color: #444754;
    padding: 8px;
    height: auto;
    border-radius: 100%;

    &:hover {
      background: #EFF1F6;
      color: #444754;
    }

    .fill-current {
      fill: transparent;
    }
  }
}

.group-title {
  font-weight: 500;
  font-size: 12px;
  line-height: 14px;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  color: #AAB1C5;
  margin-bottom: 12px;
  padding: 0 16px;
}

.history-entry {
  margin: 16px;
  padding: 16px;
  cursor: pointer;
  display: flex;
  align-items: center;
  transition: all 0.15s ease;
  border-bottom: 1px solid transparent;

  &:first-child {
    border-bottom: 1px solid transparent;
  }

  &:not(:last-child) {
    border-bottom: 1px solid #E0E2E7;
  }

  &.current, &:hover {
    background: #fff;
    border-radius: 6px;
    border-bottom: 1px solid transparent;
  }
}

.entry-left {
  flex: 1 1 auto;
}

.entry-right {
  flex: 0 0 auto;
  margin-left: 8px;
}

.entry-metadata {
  display: flex;
  align-items: center;
  margin-bottom: 8px;

  .datetime {
    flex: 0 0 auto;
    font-weight: bold;
    font-size: 14px;
    line-height: 17px;
    color: #2C2B35;
  }

  .current {
    margin-left: 8px;
    font-size: 10px;
    font-weight: bold;
    line-height: 12px;
    text-transform: uppercase;
    color: #146493;
    padding: 4px 8px;
    border-radius: 13px;
    background: #DDF3FF;
    flex: 0 1 auto;
  }
}

.entry-author {
  display: flex;
  align-items: center;

  .author-name {
    margin-left: 8px;
    font-style: normal;
    font-weight: normal;
    font-size: 13px;
    line-height: 16px;
    color: #2C2B35;
  }
}
</style>
