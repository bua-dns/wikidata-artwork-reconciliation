<script setup>
import { ref, computed } from 'vue';

const itemsData = ref([]);
const rawData = ref(null);
const items = ref([]);
const loading = ref(false);
const error = ref(null);
const currentInput = ref('');

async function submitSearch() {
  if (currentInput.value.length) {
    loading.value = true;
    const query = `
      SELECT DISTINCT ?artwork ?artworkLabel ?artworkTypeLabel ?image WHERE {
      VALUES ?artist { wd:${currentInput.value} }  # Define the artist here (e.g., Leonardo da Vinci = Q762)
    
      ?artwork wdt:P170 ?artist .  # Filter works created by the artist
      ?artwork wdt:P31 ?artworkType .
    
      OPTIONAL { ?artwork wdt:P18 ?image . }  # Include image if available
    
      SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],de,en". }
      }
    `;
    const url = `https://query.wikidata.org/sparql?query=${encodeURIComponent(
        query
      )}`;
    console.log(url);
    try {
      const response = await fetch(url, {
        headers: { Accept: "application/json" },
      });
      const data = await response.json();
      rawData.value = data;

      // itemsData.value = data.results;
      itemsData.value = data.results.bindings.map((item) => ({
        artwork: item.artwork.value,
        artworkLabel: item.artworkLabel.value,
        artworkTypeLabel: item.artworkTypeLabel.value,
        image: item.image?.value || null,
      }));
    } catch (err) {
      itemsData.value = [];
      error.value = "Failed to fetch data. Please try again.";
    } finally {
      loading.value = false;
    }
  }
} 

</script>

<template>
  <main>
    <div class="headings content-element">
      <h1>Artworks on Wikidata</h1>
      <h2>Lookup By Creator</h2>
    </div>
    <div class="controls content-element">
      <div class="search-control">
        <input type="text" class="search" v-model="currentInput" 
          placeholder="Wikidata ID (Q-Number)"
        />
        <button @click="submitSearch()" v-if="currentInput.length">Submit</button>
        <img src="@/assets/icons/x-circle.svg" alt="Edit" class="icon close-icon"
          @click="currentInput = ''"
          v-if="currentInput"
        />
      </div>
    </div>
    <div class="display content-element">
      <div v-if="loading">Loading...</div>
      <div v-else-if="error">{{ error }}</div>
      <div v-else>
        <div class="table-container">
    <table class="data-table">
      <thead>
        <tr>
          <th>Label</th>
          <th>Type</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item, index) in itemsData" :key="index">
          <td>
            <a :href="item.artwork" target="_blank">{{ item.artworkLabel }}</a></td>
          <td>{{ item.artworkTypeLabel }}</td>
          <td class="image-cell">
            <img
              v-if="item.image"
              :src="item.image"
              :alt="item.artworkLabel"
            />
            <span v-else>No Image</span>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
      </div>
  </div>
  <div class="data-output" v-if="false">
    <h3>Raw Data</h3>
    <pre>{{ rawData }}</pre>
    <pre>{{ itemsData }}</pre>
  </div>
</main>
</template>

<style scoped>
a {
  color: #0078d4;
  text-decoration: none;
}
.controls {
  position: relative;
}
.shortcuts {
  margin-bottom: 1rem;
  display: flex;
  align-items: center;
}
.suggestions {
  position: absolute;
  top: 100%;
  left: 1rem;
  padding: 1rem;
  width: 40rem;
  background-color: #fff;
  border: 1px solid #ccc;
  border-top: 0;
  display: block;
}
.suggestion {
  cursor: pointer;
  letter-spacing: .015rem;
}
.suggestion:hover {
  font-weight: bold;
  letter-spacing: 0;
}
.search-control {
  display: flex;
  gap: .25rem;
  align-items: center;
}
.search {
  width: 24rem;
  padding: 0.5rem;
  font-size: 1rem;
} 

.content-element {
  padding: 1rem;
  margin-bottom: 0.75rem;
  background-color: #fff;
  border-radius: 0.5rem;
}
.display {
  display: block;
  margin-top: 2rem;
}
.data-output {
  margin-top: 2rem;
  background-color: #fff;
  padding: 1rem;
}
.icon {
  display: inline-block;
  width: 1rem;
  height: 1rem;
  margin-left: 0.5rem;
  cursor: pointer;
}
.close-icon {
  height: 2rem;
  width: 2rem;
  opacity: 0.5;
}
h2 {
  margin-bottom: 0.5rem;
  font-size: 1.5rem;
}
h3 {
  margin-top: 1rem;
  margin-bottom: 0.5rem;
  font-size: 1.5rem;
}
h4 {
  margin-bottom: 0.5rem;
  font-size: 1.125rem;
}
.item {
  margin-bottom: 1rem;
}
.provenance-property {
  margin-left: 1rem;
  margin-bottom: 0.25rem;
}
.value-listing {
  margin-left: 1rem;
}
.value {
  display: flex;
  align-items: center;
  margin-left: 1rem;
}
.thumb {
  margin-bottom: 1rem;
}

.table-container {
  overflow-x: auto;
  padding: 16px;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  border: 1px solid #ddd;
}

.data-table th,
.data-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.data-table thead tr {
  background-color: #f9f9f9;
}

.data-table tbody tr:nth-child(even) {
  background-color: #f3f3f3;
}

.data-table .image-cell {
  text-align: center;
}

.data-table .image-cell img {
  max-width: 600px;
  max-height: 400px;
  height: auto;
  display: block;
  margin: auto;
}
</style>


