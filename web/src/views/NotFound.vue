<template>
  <div class="not-found">
    <header class="header">
      <img src="../assets/images/logo-inverse@2x.png" alt="Root Logo" class="logo">
      <span class="logo-text">Root</span>
    </header>
    <div class="content">
      <img src="../assets/images/404.svg" alt="404 Not Found" class="illustration">
      <h3 class="title">
        The page you're looking for does not exist
      </h3>
      <h4 class="subtitle">
        Looks like you stumbled upon the wrong root of the tree
      </h4>
      <div class="actions">
        <button class="btn btn-primary" @click="goBack">
          Go Back
        </button>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator'

@Component
export default class NotFound extends Vue {
  mounted () {
    // Clear last active page to prevent loop
    const spaceId = this.$store.getters['space/activeSpace'].id
    this.$store.dispatch('space/updateSetting', {
      id: spaceId,
      data: { activePage: '/' }
    })
  }

  async goBack () {
    // Clear last active page to prevent loop
    const spaceId = this.$store.getters['space/activeSpace'].id
    await this.$store.dispatch('space/updateSetting', {
      id: spaceId,
      data: { activePage: '/' }
    })
    // Going home is more consistent because the previous page might be another 404 page
    await this.$router.replace('/')
  }
}
</script>

<style lang="postcss" scoped>
.not-found {
  background: #EFF1F6;
  height: 100vh;
  display: flex;
  flex-direction: column;
}
.header {
  box-sizing: border-box;
  flex: 0 0 auto;
  padding: 24px;
  background: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  .logo {
    width: 36px;
  }
  .logo-text {
    margin-left: 16px;
    font-size: 24px;
  }
}
.content {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  flex: 1 1 auto;
}
.title {
  margin-top: 40px;
  font-style: normal;
  font-weight: 500;
  font-size: 32px;
  line-height: 38px;
  text-align: center;

  /* Grayscale/Mako */

  color: #444754;
}
.subtitle {
  margin-top: 8px;
  font-style: normal;
  font-weight: normal;
  font-size: 16px;
  line-height: 19px;
  /* identical to box height */

  text-align: center;

  /* Grayscale/Mako */

  color: #444754;
}
.actions {
  margin-top: 24px;
}
</style>
