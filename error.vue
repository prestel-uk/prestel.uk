<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps({
  error: Object as () => NuxtError
})

const route = useRoute();

let errorMsg;

if (props.error?.statusCode === 404) {
  errorMsg = `MISTAKE? PAGE ${ route.path } DOES NOT EXIST`;
} else {
  errorMsg = "SYSTEM INDEX FAULT";
}
</script>

<template>
  <div class="text-center">
    <p class="text-3xl">{{ error?.statusCode }}</p>
    <p class="text-xl" style="padding: 10px">{{ errorMsg }}</p>
    <NuxtLink to="/">Go back home</NuxtLink>
  </div>
</template>
