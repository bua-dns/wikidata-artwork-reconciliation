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
      // mappedItem['provenanceStations'] = getProvenanceStations(item['o:id']);
      mappedItem['provenanceStations'] =getProvenanceStations(item['o:id']);
      return mappedItem;
  })
  .sort((a, b) => a.label.localeCompare(b.label));
});

function getSortDate(station) {
  let date = '';
  if (station['pro:date'] && station['pro:date'][0]) {
    date = station['pro:date'][0]['@value'];
  } else if (station['pro:dateLatest'] && station['pro:dateLatest'][0]) {
    date = station['pro:dateLatest'][0]['@value'];
  }
  return date;
}

function getOwner(id) {
  const source = itemsData.value
    .find(item => item['o:id'] === id);
  if (!source) {
    return 'Owner not found';
  }
  // return source['pro:owner'][0]['@value'];
  return source['pro:caOwner'] && source['pro:caOwner'][0]
    ? source['pro:caOwner'][0]['@value']
    : 'Owner not found';
}

function getStationInfo(station) {
  let info = {
    'id': station['o:id'],
    'dbLink': getDBLink(station['o:id']),
    'infoSourceDate': station['pro:dateDisplay'][0]['@value'],
    'owner': getOwner(station['pro:infoSource'][0]['value_resource_id']),
  };
  return info;
}

function getProvenanceStations(id) {
  let stations = itemsData.value
    .filter(item => item['o:resource_template']['o:id'] === 8)
    .filter(item => item['pro:relatesToCulturalAsset'][0]['value_resource_id'] === id)
  if (!stations.length) {
    return 'No provenance stations found';
  }
  let mappedStations = stations.map(station => {
    return {
      'label': station['dcterms:title'][0]['@value'],
      'dbLink': getDBLink(station['o:id']),
      'sortDate': getSortDate(station),
      'stationInfo': getStationInfo(station),
    }
  });
    return mappedStations;
}

function getRelatedInformationUnits(id) {
  let relatedItems = itemsData.value
    .filter(item => item['o:resource_template']['o:id'] === 7)
    .filter(item => item['pro:relatesToCulturalAsset'][0]['value_resource_id'] === id)
    .map(item => {
      let mappedItem = {};
      mappedItem['id'] = item['o:id'];
      mappedItem['dbLInk'] = getDBLink(item['o:id']);
      mappedItem['infoSourceDate'] = item['pro:infoSourceDate'][0]['@value'];
      mappedItem['label'] = item['o:title'];
      const provenanceProps = Object.keys(item).filter(key => key.startsWith('pro:'));
      mappedItem['claims'] = provenanceProps
        .filter(prop => prop.startsWith('pro:ca'))
        .map(prop => {
          return {
            label: item[prop][0]['property_label'],
            value: item[prop][0]['@value'],
          };
        });
      return mappedItem;
    })
    .sort((a, b) => a.infoSourceDate.localeCompare(b.infoSourceDate));
  let claims = {};
  for (let item of relatedItems) {
    for (let claim of item.claims) {
      if (!claims[claim.label]) {
        claims[claim.label] = [];
      }
      claims[claim.label].push({
        value: claim.value,
        infoSourceDate: item.infoSourceDate,
        dbLink: item.dbLInk,
      }) ;
    }
  }
  for (let claim in claims) {
    claims[claim].sort((a, b) => a.infoSourceDate.localeCompare(b.infoSourceDate));
  }

  return claims
  // return relatedItems;
}
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
const currentInput = ref('');
const suggestions = computed(() => {
  if (!currentInput.value || currentInput.value.length < 2) {
    return [];
  }
  return itemsData.value
    .filter(item => item['@type'].includes('pro:CulturalAsset'))
    .filter(item => item['o:title'].toLowerCase().includes(currentInput.value.toLowerCase()))
    .map(item => {
      return {
        label: item['o:title'],
        id: item['o:id'],
      };
    });
});

const selectedItem = ref(null);
function selectItem(id) {
  const item = items.value.find(item => item.id === id);
  if (item) {
    selectedItem.value = item; // Set the selected item
    currentInput.value = ''; // Clear the input
  }
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
</script>

<template>
  <main>
    <div class="headings content-element">
      <h1>Provenienzforschung zur Sammlung Sultan</h1>
      <h2>Data Inspector</h2>
    </div>
    <div class="controls content-element">
      <div class="search-control">
        <input type="text" class="search" v-model="currentInput" 
          placeholder="Werk suchen..."
        />
        <img src="@/assets/icons/x-circle.svg" alt="Edit" class="icon close-icon"
          @click="currentInput = ''"
          v-if="currentInput"
        />
      </div>
      <div class="suggestions" v-if="currentInput && suggestions?.length">
        <div class="suggestion"
          v-for="item in suggestions"
          :key="item.label"
          @click="selectItem(item.id)"
        >
          {{ item.label }}
        </div>
      </div>
    </div>
    <div class="display">
      <div v-if="loading">Loading...</div>
      <div v-else-if="error">{{ error }}</div>
      <div v-else>
        <div class="item content-element" v-if="selectedItem">
          <h3>{{ selectedItem.label }} <a :href="getDBLink(selectedItem.id)" target="_blank" rel="noopener">
              <img src="@/assets/icons/edit.svg" alt="Edit" class="icon edit-icon"/>
            </a>
          </h3>
          <div class="content">
            <template v-if="selectedItem?.related">
              <div class="field"
                v-for="(values, label) in selectedItem.related"
                :key="label"
              >
                <strong>{{ label }}</strong>
                <div class="value"
                  v-for="value in values"
                  :key="value.value"
                >
                  {{ value.value }}
                  <a :href="value.dbLink" target="_blank" rel="noopener">
                    <img src="@/assets/icons/edit.svg" alt="Edit" class="icon edit-icon"/>
                  </a>
                </div>
              </div>
            </template>
          </div>
          <div class="provenance-stations">
            <h4>Provenienzkette</h4>
            <div v-if="selectedItem.provenanceStations === 'No provenance stations found'">
              {{ selectedItem.provenanceStations }}
            </div>
            <div v-else>
              <div class="station value"
                v-for="station in selectedItem.provenanceStations"
                :key="station.label"
              >
                {{ station.stationInfo.infoSourceDate }}: {{ station.stationInfo.owner }}
                <a :href="station.stationInfo.dbLink" target="_blank" rel="noopener">
                    <img src="@/assets/icons/edit.svg" alt="Edit" class="icon edit-icon"/>
                  </a>
              </div>
            </div>
          </div>
          <pre v-if="false">{{ selectedItem.provenanceStations }}</pre>
        </div>
      </div>
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
  </div>
  <div class="data-output" v-if="false">
    <pre>{{ items }}</pre>
  </div>
</main>
</template>

<style scoped>
.controls {
  position: relative;
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
  background-color: #fff;
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
</style>


