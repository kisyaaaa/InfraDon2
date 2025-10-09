<script setup lang="ts">

import PouchDB from 'pouchdb'
import { onMounted } from 'vue';
import { ref } from 'vue';


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const storage = ref()

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://lilianakmkv:8260@localhost:5984/infra_53_1527')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}
 
</script>

<template>
  <p>InfraDon</p>
</template>
