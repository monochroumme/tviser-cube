<template>
  <div class="index-page">
    <Cube />
  </div>
</template>

<script>
export default {
  name: 'IndexPage',

  components: {
    Cube: () => import('@/components/Cube')
  }
}
</script>
