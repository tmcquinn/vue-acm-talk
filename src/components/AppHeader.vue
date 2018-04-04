<template>
  <div v-if="loaded">
      <nav class="navbar">
        <img width="50" :src="profileImg"/>
        <h2>twitch.tv/{{name}}</h2>
      </nav>
  </div>
</template>
<script>
export default {
  name: 'AppHeader',
  props: ['loaded', 'profileImg', 'name'],
  data() {
      return {
      }
  }
}
</script>
<style>
  .navbar {
    background-color: #e7e7e7;
  }
</style>
