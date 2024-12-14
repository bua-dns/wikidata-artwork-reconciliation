<script setup>
import { ref, computed, onMounted } from 'vue';

const itemsData = ref([]);
const loading = ref(true);
const error = ref(null);

const items = computed(() => {
  return itemsData.value
  .filter(item => item['o:resource_template']['o:id'] === 6 )
   .map(item => {
        let mappedItem = {};
        mappedItem['id'] = item['o:id'];
        mappedItem['label'] = item['o:title'];
        const provenanceProps = Object.keys(item).filter(key => key.startsWith('pro:'));
        provenanceProps.forEach(prop => {
          mappedItem[prop] = item[prop];
        });
        mappedItem['related'] = getRelatedInformationUnits(item['o:id']);
        return mappedItem;
    })
});

function getRelatedInformationUnits(id) {
  return itemsData.value
    .filter(item => item['o:resource_template']['o:id'] === 7)
    .filter(item => item['pro:relatesToCulturalAsset'][0]['value_resource_id'] === id)
    .map(item => {
      let mappedItem = {};
      mappedItem['id'] = item['o:id'];
      mappedItem['label'] = item['o:title'];
      const provenanceProps = Object.keys(item).filter(key => key.startsWith('pro:'));
      provenanceProps.forEach(prop => {
        mappedItem[prop] = item[prop];
      });
      return mappedItem;
    });
}

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
// extract value for specific property from related resource
function getRelatedValue(prop, resourceId) {
  const relatedItem = itemsData.value.find(item => item['o:id'] === resourceId);
  if (!relatedItem) {
    return 'Related item not found';
  }
  if (!relatedItem[prop] || !relatedItem[prop][0]) {
    return 'Value for property not found';
  }
  return relatedItem[prop][0]['@value'];
}

function getDBLink(id) {
  return `https://omeka-s-t1.berlin-university-collections.de/admin/item/${id}`;
}
</script>

<template>
  <main>
    <h1>Provenienzforschung zur Sammlung Sultan</h1>
    <h2>Data Inspector</h2>
    <pre v-if="true">{{ items }}</pre>
    <div>
    <!-- <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else>
      <div v-for="item in items" :key="item.id" class="item">
        <h3>{{ item.label }} <a :href="getDBLink(item.id)" target="_blank" rel="noopener">
                  <img src="@/assets/icons/edit.svg" alt="Edit" class="icon edit-icon"/>
                </a></h3>
        <div v-for="prop in Object.keys(item).filter(key => key.startsWith('provenance:'))" 
          :key="prop"
          class="provenance-property"  
        >
          <strong>{{ item[prop][0]['property_label'] }}</strong>: 
          <div class="value value-listing "
            v-for="(value, index) in item[prop]"
            :key="index"
          >
            <template v-if="value.type === 'literal'">
              {{ value['@value'] }}
            </template>
            <template v-if="value.type === 'resource'">
              {{ getRelatedValue(prop, value['value_resource_id']) }}
              
                <a :href="getDBLink(value['value_resource_id'])" target="_blank" rel="noopener">
                  <img src="@/assets/icons/edit.svg" alt="Edit" class="icon edit-icon"/>
                </a>
            </template>
          </div> 
        </div>
        <pre v-if="false">{{ item }}</pre>
      </div>
    </div> -->
    </div>
  </main>
</template>

<style scoped>
.icon {
  display: inline-block;
  width: 1rem;
  height: 1rem;
  margin-left: 0.5rem;
  cursor: pointer;
}
h2 {
  margin-bottom: 0.5rem;
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
}
</style>


