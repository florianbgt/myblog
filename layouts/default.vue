<template>
  <div class="text-dark">
    <TheHeader />
      <Nuxt />
    <TheFooter />
  </div>
</template>

<style>
.page-enter-active,
.page-leave-active {
  transition: opacity 0.4s;
}
.page-enter,
.page-leave-to {
  opacity: 0;
}
</style>