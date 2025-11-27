<script setup lang="ts">
import { onMounted, ref, watch } from 'vue';
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

declare interface Comment {
  id: string;            
  content: string;       
  created_at: string;     
}

declare interface Post {
  _id?: string;
  _rev?: string;
  title: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
  likes: number;
  comments: Comment[];
  category: 'vacances' | 'lifestyle' | 'éducatif' | 'général';
}

// "factory"
const generatePosts = async (count = 50) => {
  if (!storage.value) return;

  const categories: ('vacances' | 'lifestyle' | 'éducatif' | 'général')[] = ['vacances', 'lifestyle', 'éducatif', 'général'];
  const docs = Array.from({ length: count }, (_, i) => ({
    _id: `post_${Date.now()}_${i}`,
    title: `Random post ${i + 1}`,
    post_content: `Contenu du post ${i + 1}`,
    attributes: { creation_date: new Date().toISOString() },
    likes: Math.floor(Math.random() * 100),
    comments: [],
    category: categories[Math.floor(Math.random() * categories.length)],
  }));

  try {
    await storage.value.bulkDocs(docs);
    console.log(`${count} documents générés !`);
    fetchData();
  } catch (err) {
    console.error("Erreur lors de la génération :", err);
  }
};

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// recherche
const searchQuery = ref('')
//tri
const sortField = ref<'creation_date' | 'likes' | 'attributes.creation_date'>('likes');
const sortDirection = ref<'desc' | 'asc'>('desc');
// filtre par catégorie
const selectedCategory = ref<'all' | 'vacances' | 'lifestyle' | 'éducatif' | 'général'>('all');

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
    await storage.value.createIndex({
      index: { fields: ['likes'] }
    });
    await storage.value.createIndex({
      index: { fields: ['attributes.creation_date'] }
    });
    await storage.value.createIndex({
      index: { fields: ['category'] }
    });
    console.log('Index créé avec succès!');
  } catch (err) {
    console.error("Erreur lors de la création de l'index", err);
  }
}


// Récupération des données
const fetchData = (): any => {
  if (!storage.value) return;

  const selector: any = { _id: { $exists: true } };

  // Ajouter le filtre de catégorie si une catégorie est sélectionnée
  if (selectedCategory.value !== 'all') {
    selector.category = selectedCategory.value;
  }

  storage.value.find({
    selector: selector,
    sort: [{ [sortField.value]: sortDirection.value }],
  })
    .then((result: any) => {
      console.log('=> Données récupérées et affichées:', result.docs.length);
      postsData.value = result.docs as Post[];
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}


//fonction de tri
const changeSort = (field: 'creation_date' | 'likes') => {
  const fieldPath = field === 'creation_date' ? 'attributes.creation_date' : field;
  if (sortField.value === fieldPath) {
    sortDirection.value = sortDirection.value === 'desc' ? 'asc' : 'desc';
  } else {
    sortField.value = fieldPath;
    sortDirection.value = 'desc';
  }
  fetchData();
};

// fonction recherche 
const searchPosts = async () => {
  if (!storage.value) return;
  const query = String(searchQuery.value || '');
  storage.value.find({
    selector: { title: { $regex: query } }
  })
    .then((result: any) => {
      postsData.value = result.docs as Post[];
      console.log('Résultats de la recherche :', result.docs);
    })
    .catch((err: any) => {
      console.error('Erreur lors de la recherche', err);
    });
};

// Ajout du document
const addDocument = () => {
  storage.value.post({
    title: 'Ajouter votre nouveau mot ' + new Date().toLocaleTimeString(),
    post_content: 'Contenu par défaut',
    attributes: { creation_date: new Date().toISOString() },
    likes: 0,
    comments: [],
    category: 'général',
  }).then(() => {
    console.log("Ça marche");
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur :", error);
  });
};

// Delete document
const deleteDocument = (post: Post) => {
  storage.value.remove(post._id, post._rev).then(() => {
    console.log("Document supprimé");
    fetchData();
  }).catch((error: any) => {
    console.error("Erreur lors de la suppression :", error);
  });
};

// Update document

const updateDocument = (post: Post) => {
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

//likes et commentaires
const toggleLike = (post: Post) => {
  post.likes = (post.likes || 0) + 1;
  updateDocument(post);
};

const addComment = (post: Post, newContent: string) => {
  if (!newContent.trim()) return;
  const newComment: Comment = {
    id: `comment_${Date.now()}_${Math.random()}`,
    content: newContent,
    created_at: new Date().toISOString()
  };
  post.comments = [...(post.comments || []), newComment];
  updateDocument(post);
};

const deleteComment = (post: Post, commentId: string) => {
  post.comments = (post.comments || []).filter(c => c.id !== commentId);
  updateDocument(post);
};

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
  <h1>Amstramgram</h1>
  <button v-if="!isOffline" @click="isOffline = true">Passer en Offline</button>
  <button v-else @click="isOffline = false">Revenir en Online</button>
  <button @click="addDocument">Ajouter un nouveau post</button>
  <button @click="replicateNow">Synchroniser maintenant</button>

  <div>
    <button @click="generatePosts(10)">Générer 50 posts</button>
    <input v-model="searchQuery" placeholder="Rechercher par titre" />
    <button @click="searchPosts">Rechercher</button>
    <button @click="fetchData">Réinitialiser</button>
    <button @click="changeSort('likes')">Trier par likes</button>
    <button @click="changeSort('creation_date')">Trier par date</button>
  </div>

  <div>
    <label>Filtrer par catégorie : </label>
    <select v-model="selectedCategory" @change="fetchData">
      <option value="all">Toutes les catégories</option>
      <option value="vacances">Vacances</option>
      <option value="lifestyle">Lifestyle</option>
      <option value="éducatif">Éducatif</option>
      <option value="général">Général</option>
    </select>
  </div>
  <article v-for="post in postsData" :key="(post as any)._id">
    <input v-model="post.title" />
    <select v-model="post.category">
      <option value="vacances">Vacances</option>
      <option value="lifestyle">Lifestyle</option>
      <option value="éducatif">Éducatif</option>
      <option value="général">Général</option>
    </select>
    <button @click="updateDocument(post)">Sauvegarder</button>
    <button @click="deleteDocument(post)">Supprimer</button>
    <button @click="toggleLike(post)">Like ({{ post.likes || 0 }})</button>

     <div>
      <h4>Commentaires ({{ post.comments?.length || 0 }})</h4>
      <ul v-if="post.comments?.length">
        <li v-for="comment in post.comments" :key="comment.id">
          {{ comment.content }}
          <button @click="deleteComment(post, comment.id)">X</button>
        </li>
      </ul>

      <div>
        <input :id="'comment-input-' + (post as any)._id" placeholder="Ajouter un commentaire" />
        <button
          @click="
            addComment(post, (($event.target as HTMLElement).previousElementSibling as HTMLInputElement)?.value || '');
            ((($event.target as HTMLElement).previousElementSibling as HTMLInputElement).value = '');
          "
        >Ajouter</button>
      </div>
    </div>
  </article>
</template>
