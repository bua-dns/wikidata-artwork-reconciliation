<script setup>
import { ref, computed, onMounted } from 'vue';

const itemsData = ref([]);
const loading = ref(true);
const error = ref(null);

const items = computed(() => {
  return itemsData.value
  .filter(item => item['o:resource_class']['o:id'] === 32)
});


onMounted(async () => {
  try {
    const response = await fetch('https://omeka-s-t1.berlin-university-collections.de/api/items');
    if (!response.ok) {
      throw new Error(`Error: ${response.status} ${response.statusText}`);
    }
    itemsData.value = await response.json();
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Unknown error';
  } finally {
    loading.value = false;
  }
});
/*


*/
</script>

<template>
  <main>
    <h1>Provenienzforschung zur Sammlung Sultan</h1>
    <h2>Data Inspector</h2>
    <div>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
      <ul v-else>
        <li v-for="item in items" :key="item.id">
          <pre>{{ item }}</pre>
        </li>
      </ul>
    </div>
  </main>
</template>


