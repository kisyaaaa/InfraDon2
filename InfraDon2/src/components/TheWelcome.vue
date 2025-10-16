<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb'

declare interface Post {
  title: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://lilianakmkv:8260@localhost:5984/infra_don2')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db;
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données

   const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc);
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

const addDocument = () => {
  storage.value.post({
    title: 'New'
  }).then(() => {
    console.log("Ça marche");
    fetchData();
  }).catch((error:any) => {
    console.error("Erreur :", error);
  });
};

  // TODO Récupération des données
  // https://pouchdb.com/api.html#batch_fetch
  // Regarder l'exemple avec function allDocs
  // Remplir le tableau postsData avec les données récupérée

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});
console.log(postsData.value)

</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addDocument">Clique ici</button>

  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template>
