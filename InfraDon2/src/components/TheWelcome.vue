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

// déclaration de la DB locale
const localDB = new PouchDB('infra_don2_local')

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


 // synchronisation automatique locale <-> distante
  localDB.sync(db, {
    live: false,
    retry: true
  })
  .on('complete', () => console.log("Synchronisation initiale réussie"))
  .on('error', (err) => console.error("Oups, erreur de synchronisation", err));
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

// Ajout du document 
const addDocument = () => {
  storage.value.post({
    title: 'Ajouter votre nouveau mot'
  }).then(() => {
    console.log("Ça marche");
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur :", error);
  });
};

// Delete document
const deleteDocument = (post: any) => {
  storage.value.remove(post._id, post._rev).then(() => {
    console.log("Document supprimé");
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur lors de la suppression :", error);
  });
};

// Update document

const updateDocument = (post: any) => {
  storage.value.put(post)
    .then(() => {
      console.log("Document mis à jour");
      fetchData();
    })
    .catch((error: any) => {
      console.error("Erreur lors de la mise à jour :", error);
    });
};

// TODO Récupération des données
// https://pouchdb.com/api.html#batch_fetch
// Regarder l'exemple avec function allDocs
// Remplir le tableau postsData avec les données récupérée



 //bouton manuel de réplication
const replicateNow = () => {
  localDB.replicate.to(storage.value)
    .on('complete', () => {
      console.log('Réplication locale -> distante réussie');
      fetchData();
    })
    .on('error', (err) => console.error("Oups, erreur de réplication", err));
};

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
  <button @click="replicateNow">Synchroniser maintenant</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
  <input v-model="post.title" />
  <button @click="updateDocument(post)">Sauvegarder</button>
  <button @click="deleteDocument(post)">Supprimer</button>
</article>
</template>




