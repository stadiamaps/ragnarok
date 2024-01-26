<script setup>
import ValhallaResponseVisualizer
  from "@/components/ValhallaResponseVisualizer.vue";
import {ref} from "vue";

const data = ref('');
</script>

<template>
  <main>
    <ValhallaResponseVisualizer :data="data"/>
    <p>
      Paste a Valhalla API response:
    </p>
    <textarea v-model="data"></textarea>
    <p>
      Check the <a href="https://github.com/stadiamaps/ragnarok">repository on GitHub</a>
      for usage instructions.
    </p>
  </main>
</template>

<style scoped>
textarea {
  min-height: 250px;
  width: 100%;
}
</style>
