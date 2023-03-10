<template>
  <Modal
    title="Image"
    :visible="visible"
    noheader
    nosubmit
    nofooter
    cancel-text="Okay"
    portal="secondary"
    :z-index="2000"
    @cancel="close"
    :modalStyle="{ 'background-color': 'rgb(68 71 84 / 0.97)' }"
    :contentStyle="{ 'background-color': 'unset', height: '65%', boxShadow: 'none', overflowY: 'visible' }"
  >
    <div class="task-modal-body">
      <span class="close" @click="close">
        <legacy-icon name="close" title="Close" size="32" viewbox="32" />
      </span>
      <div class="image-container">
        <div class="image-box" v-if="isFileImage">
          <img :src="image.versions.preview.location" />
        </div>
        <iframe
          v-else
          :src="`${image.location}`"
          type="application/pdf"
          class="pdf-preview"
        />
        <div class="title">
          <p v-if="image">
            {{ image.name }}
          </p>
          <Popover
            :z-index="2001"
            :offset="10"
            :with-close="false"
            position="right-start"
            class="modal-action"
          >
            <template #default="{}">
              <div class="action-line" @click="download">
                <legacy-icon
                  class="action-icon"
                  name="download"
                  viewbox="16"
                  size="16px"
                ></legacy-icon>
                <div class="action-line-text">Download</div>
              </div>

              <div class="action-separator"></div>

              <div class="action-line" @click="restore" v-if="image.deletedAt">
                <mono-icon name="restore" class="action-line-icon" />
                <div class="action-line-text">Put Back</div>
              </div>
              <div class="action-line danger" @click="remove" v-else>
                <mono-icon name="trash" class="action-line-icon" />
                <div class="action-line-text">Delete</div>
              </div>
            </template>
            <template #trigger="{ visible }">
              <button
                class="btn btn-link"
                :class="{ 'btn-link-primary': visible }"
              >
                <legacy-icon name="ellipsis" viewbox="20" size="1.25rem" />
              </button>
            </template>
          </Popover>
        </div>
      </div>
    </div>
  </Modal>
</template>

<script lang="ts">
import { Component, Prop, Vue, Model } from 'vue-property-decorator'

import Modal from '@/components/legacy/Modal.vue'
import Popover from '@/components/Popover.vue'
import type { NewUploadResource } from '@/types/resource'

@Component({
  name: 'StorageViewer',
  components: {
    Modal,
    Popover
  }
})
export default class StorageViewer extends Vue {
  @Model('change', { type: Number }) readonly index!: number

  @Prop({ type: Object })
  private readonly image!: NewUploadResource

  close () {
    this.$emit('close', this.image)
  }

  remove () {
    this.$emit('remove', this.image)
  }

  restore () {
    this.$emit('restore', this.image)
  }

  download () {
    this.$emit('download', this.image)
  }

  get visible () {
    return this.index !== null
  }

  get isFileImage () {
    return this.image.mimetype.startsWith('image')
  }
}
</script>

<style lang="postcss" scoped>
.task-modal-body {
  @apply relative;
}

.image {
  @apply rounded shadow-lg;
  max-width: 80vw;
  max-height: 80vh;
}

.close {
  @apply fixed;
  background-color: rgb(255 255 255 / 20%);
  cursor: pointer;
  border-radius: 50px;
  color: #fff;
  width: 48px;
  height: 48px;
  top: 40px;
  right: 40px;
  svg {
    margin: 7px 0 0 8px;
  }
  &:hover {
    background-color: rgb(255 255 255 / 40%);
  }
}

.image-container {
  @apply flex flex-col items-center w-screen;
  .image-box {
    @apply flex;
    img {
      cursor: pointer;
      margin: 0 auto;
      border-radius: 4px;
      max-width: 100%;
      max-height: 600px;

      @media (min-width: 1200px) {
        max-width: 100vw;
        max-height: 75vh;
      }
    }
  }

  .title {
    @apply flex justify-center;

    text-align: center;
    color: #fff;
    margin-top: 8px;
    width: 943px;
    word-break: break-word;
  }
}

.action-line {
  @apply flex items-center py-2 px-4 my-1 relative;
  font-size: 13px;
  line-height: 16px;
  width: 168px;
  color: theme('colors.gray.900');
  stroke-width: 3px;
  cursor: pointer;

  &:hover {
    background: #f0f2f5;
  }
  &.danger {
    color: theme('colors.danger.default');
  }
}

.action-line-icon {
  font-size: 16px;
  stroke-width: 2px;
}

.action-line-text {
  @apply ml-2;
  flex: 1 1 auto;
}
.action-separator {
  @apply my-1;
  height: 1px;
  background: theme('colors.gray.100');
}

.btn-link {
  padding: 0;
  width: 24px;
  height: 24px;
  background-color: rgb(255 255 255 / 20%);
  cursor: pointer;
  border-radius: 50px;
  margin-left: 8px;

  &:hover {
    background-color: rgb(255 255 255 / 40%);
  }
}

.popover-trigger.show button {
  background-color: rgb(255 255 255 / 40%);
  color: #fff;
}

.modal-action {
  text-align: left;
}

.pdf-preview {
  @apply w-screen px-10;
  height: 80vh;
  margin-top: -5rem;
}
</style>
