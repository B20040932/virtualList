<template>
  <div>
    <!-- <virtualList/> -->
    <div style="margin:20px;"></div>
    <virtualList2/>
  </div>
</template>

<script>
import virtualList from '@/components/virtualList.vue'
import virtualList2 from '@/components/virtualList2.vue'

export default {
    components:{
        virtualList,
        virtualList2
    }
}
</script>

<style>

</style>