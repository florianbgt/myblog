<template>
    <img :src="require(`~/content${this.$route.path}/${this.src}`)"/>
</template>

<script>
export default {
  props: ['src']
}
</script>