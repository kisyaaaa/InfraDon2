<script setup lang="ts">
import { onMounted, ref, watch } from 'vue';
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

// "factory"
const generatePosts = async (count = 10) => {
  if (!storage.value) return;

  const docs = Array.from({ length: count }, (_, i) => ({
    _id: `post_${Date.now()}_${i}`,  
    title: `Random ${i + 1}`,
    post_content: `Contenu du post ${i + 1}`,
    attributes: { creation_date: new Date().toISOString() }
  }));

  try {
    await storage.value.bulkDocs(docs);
    console.log(`${count} documents générés !`);
    fetchData();
  } catch (err) {
    console.error("Erreur lors de la génération :", err);
  }
};


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

// recherche 
const searchQuery = ref('') 

// déclaration de la DB locale
const localDB = new PouchDB('infra_don2_local')

// déclaration de la DB distante
const remoteDB = new PouchDB('http://lilianakmkv:8260@localhost:5984/infra_don2')

// activer/désactiver le offline
const isOffline = ref(false)

// initialisation de la Database utilise isOffline pour la base
const initDatabase = () => {
  console.log('=> Connexion à la base de données');

  storage.value = isOffline.value ? localDB : remoteDB;
  console.log("Mode actif :", isOffline.value ? "Offline (localDB)" : "Online (remoteDB)");

//création de l'index
const createIndex = async () => {
  if (!storage.value) return
  try {
    await storage.value.createIndex({ index: { fields: ['title'] } })
    console.log('Index créé sur le champ "title"')
  } catch (err) {
    console.error('Erreur lors de la création de l’index', err)
  }
}



  // synchronisation automatique locale <-> distante
  if (!isOffline.value) {
    localDB.sync(remoteDB, {
      live: false,
      retry: true
    })
      .on('complete', () => console.log("Synchronisation initiale réussie"))
      .on('error', (err) => console.error("Oups, erreur de synchronisation", err));
  }
}

// création de l'index 
const createIndex = async () => {
  if (!storage.value) return;
  try {
    await storage.value.createIndex({
      index: { fields: ['title'] }
    });
    console.log('Index créé sur le champ "title"');
  } catch (err) {
    console.error('Erreur lors de la création de l’index', err);
  }
}


// Récupération des données
const fetchData = (): any => {
  if (!storage.value) return;
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

// fonction recherche 
const searchPosts = async () => {
  if (!storage.value) return;
  const query = String(searchQuery.value || '');
  try {
    const result = await storage.value.find({
      selector: {
        title: { $regex: query }
      }
    });
    postsData.value = result.docs
    console.log('Résultats de la recherche :', result.docs)
  } catch (err) {
    console.error('Erreur lors de la recherche', err)
  }
}

// Ajout du document 
const addDocument = () => {
  storage.value.post({
    title: 'Ajouter votre nouveau mot ' + new Date().toLocaleTimeString() //identification du moment
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
  localDB.replicate.to(remoteDB)
    .on('complete', () => {
      console.log('Réplication locale -> distante réussie');
      fetchData();
    })
    .on('error', (err) => console.error("Oups, erreur de réplication", err));
};

watch(isOffline, (newVal) => {
  console.log("Mode basculé :", newVal ? "Offline" : "Online");
  storage.value = newVal ? localDB : remoteDB;

  if (!newVal) {
    // quand on repasse online, synchroniser
    localDB.sync(remoteDB, { live: false, retry: true })
      .on('complete', () => {
        console.log("Synchronisation après retour online réussie");
        fetchData();
      })
      .on('error', (err) => console.error("Erreur de sync :", err));
  } else {
    fetchData();
  }
});

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  createIndex() 
  fetchData()
});
console.log(postsData.value)

</script>

<template>
  <h1>Fetch Data</h1>
  <button v-if="!isOffline" @click="isOffline = true">Passer en Offline</button>
  <button v-else @click="isOffline = false">Revenir en Online</button>
  <button @click="addDocument">Clique ici</button>
  <button @click="replicateNow">Synchroniser maintenant</button>
   <div>
   <button @click="generatePosts(10)">Générer 10 posts</button>
    <input v-model="searchQuery" placeholder="Rechercher par titre" />
    <button @click="searchPosts">Rechercher</button>
    <button @click="fetchData">Réinitialiser</button>
  </div>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <input v-model="post.title" />
    <button @click="updateDocument(post)">Sauvegarder</button>
    <button @click="deleteDocument(post)">Supprimer</button>
  </article>
</template>
