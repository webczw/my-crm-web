<script setup lang="ts">
import { useRouter, RouterView } from 'vue-router'
import { ref } from 'vue'

const activeIndex = ref('/project')
const router = useRouter()
const handleSelect = (key: string, keyPath: string[]) => {
  console.log(key, keyPath)
}
</script>

<template>
  <div class="common-layout">
    <el-container>
      <el-header style="padding: 0;">
        <el-menu
    :default-active="activeIndex"
    class="el-menu-demo"
    mode="horizontal"
    router
    @select="handleSelect"
  >
    <el-menu-item index="/project">项目管理</el-menu-item>
    <el-menu-item index="/finance">财经管理</el-menu-item>
  </el-menu>

      </el-header>
      <el-main style="padding: 10px;">
        <RouterView />
      </el-main>
    </el-container>
  </div>
</template>

<style scoped>

</style>
