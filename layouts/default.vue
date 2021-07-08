<template>
  <div id="root" class="text-dark">
    <TheHeader />
      <Nuxt id="main-content" />
    <TheFooter />
  </div>
</template>

<style>
#root {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
#main-content {
  flex: 1;
}

.page-enter-active,
.page-leave-active {
  transition: opacity 0.4s;
}
.page-enter,
.page-leave-to {
  opacity: 0;
}
</style>