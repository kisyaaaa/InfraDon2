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
  _attachments?: any;
  title: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
  likes: number;
  comments: Comment[];
  category: 'vacances' | 'lifestyle' | 'éducatif' | 'général';
  image_url?: string;
}

declare interface User {
  _id?: string;
  _rev?: string;
  username: string;
  email: string;
  followers: number;
  following: number;
}

// "factory"
const generatePosts = async (count = 10) => {
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
const usersData = ref<User[]>([])

// recherche
const searchQuery = ref('')
//tri
const sortField = ref<'creation_date' | 'likes' | 'attributes.creation_date'>('likes');
const sortDirection = ref<'desc' | 'asc'>('desc');
// filtre par catégorie
const selectedCategory = ref<'all' | 'vacances' | 'lifestyle' | 'éducatif' | 'général'>('all');
// pagination
const displayLimit = ref(10);

// posts affichés (limités)
const displayedPosts = ref<Post[]>([])

// état d'affichage des commentaires (map postId -> boolean)
const showAllComments = ref<Record<string, boolean>>({})

// fonction pour voir plus de posts (affiche les 10 suivants)
const showMore = () => {
  displayLimit.value += 10;
  updateDisplayedPosts();
}

// fonction pour mettre à jour les posts affichés
const updateDisplayedPosts = () => {
  displayedPosts.value = postsData.value.slice(0, displayLimit.value);
}

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

  // synchronisation automatique locale <-> distante en temps réel
  if (!isOffline.value) {
    localDB.sync(remoteDB, {
      live: true,
      retry: true
    })
      .on('change', (info) => {
        console.log("Changement détecté:", info);
        fetchData();
      })
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
    await storage.value.createIndex({
  index: { fields: ['username'] }
    });
    console.log('Index créé avec succès!');
  } catch (err) {
    console.error("Erreur lors de la création de l'index", err);
  }
}


// Récupération des données
const fetchData = (resetLimit: boolean = true): any => {
  if (!storage.value) return;

  const selector: any = {
    _id: { $regex: '^post_' } 
  };

  // Ajouter le filtre de catégorie si une catégorie est sélectionnée
  if (selectedCategory.value !== 'all') {
    selector.category = selectedCategory.value;
  }

  storage.value.find({
    selector: selector,
    sort: [{ [sortField.value]: sortDirection.value }],
  })
    .then(async (result: any) => {
      console.log('=> Données récupérées et affichées:', result.docs.length);
      postsData.value = result.docs as Post[];
      // Charger les images pour chaque post
      for (const post of postsData.value) {
        await loadImage(post);
      }
      if (resetLimit) {
        displayLimit.value = postsData.value.length;
      }
      updateDisplayedPosts();
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

const fetchUsers = (): any => {
  if (!storage.value) return;
  
  storage.value.find({
    selector: { 
      _id: { $regex: '^user_' }
    }
  })
    .then((result: any) => {
      console.log('=> Utilisateurs récupérés:', result.docs.length);
      usersData.value = result.docs as User[];
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des utilisateurs :', error)
    })
}


const addUser = () => {
  storage.value.post({
    _id: `user_${Date.now()}`,
    username: 'User_' + Date.now(),
    email: `user${Date.now()}@example.com`,
    followers: 0,
    following: 0,
  }).then(() => {
    console.log("Utilisateur ajouté");
    fetchUsers();
  }).catch((error: any) => {
    console.error("Erreur :", error);
  });
};

const deleteUser = (user: User) => {
  storage.value.remove(user._id, user._rev).then(() => {
    console.log("Utilisateur supprimé");
    fetchUsers();
  }).catch((error: any) => {
    console.error("Erreur lors de la suppression :", error);
  });
};

const updateUser = (user: User) => {
  storage.value.put(user)
    .then(() => {
      console.log("Utilisateur mis à jour");
      fetchUsers();
    })
    .catch((error: any) => {
      console.error("Erreur lors de la mise à jour :", error);
    });
};


//fonction de tri
const changeSort = (field: 'creation_date' | 'likes') => {
  const fieldPath = field === 'creation_date' ? 'attributes.creation_date' : field;
  if (sortField.value === fieldPath) {
    sortDirection.value = sortDirection.value === 'desc' ? 'asc' : 'desc';
  } else {
    sortField.value = fieldPath;
    sortDirection.value = 'desc';
  }
  displayLimit.value = 10; // Limiter à 10 
  fetchData(false);
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
    _id: `post_${Date.now()}`,
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

// basculer l'affichage de tous les commentaires
const toggleComments = (postId: string) => {
  showAllComments.value[postId] = !showAllComments.value[postId];
};

// obtenir les commentaires à afficher (dernier ou tous)
const getDisplayedComments = (post: Post): Comment[] => {
  if (!post.comments?.length) return [];
  if (showAllComments.value[post._id || '']) return post.comments;
  const lastComment = post.comments[post.comments.length - 1];
  return lastComment ? [lastComment] : [];
};

// Gestion des images
const uploadImage = async (post: Post, file: File) => {
  if (!storage.value || !post._id || !post._rev) return;

  try {
    const response = await storage.value.putAttachment(
      post._id,
      'image',
      post._rev,
      file,
      file.type
    );
    post._rev = response.rev;
    console.log('Image uploadée avec succès');
    await loadImage(post);
  } catch (err) {
    console.error('Erreur lors de l\'upload:', err);
  }
};

const loadImage = async (post: Post) => {
  if (!storage.value || !post._id || !post._attachments?.image) return;

  try {
    const blob = await storage.value.getAttachment(post._id, 'image');
    post.image_url = URL.createObjectURL(blob);
  } catch (err) {
    console.error('Erreur lors du chargement de l\'image:', err);
  }
};

const deleteImage = async (post: Post) => {
  if (!storage.value || !post._id || !post._rev) return;

  try {
    const response = await storage.value.removeAttachment(post._id, 'image', post._rev);
    post._rev = response.rev;
    if (post.image_url) {
      URL.revokeObjectURL(post.image_url);
      post.image_url = undefined;
    }
    console.log('Image supprimée avec succès');
    fetchData();
  } catch (err) {
    console.error('Erreur lors de la suppression:', err);
  }
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
    // quand on repasse online, synchroniser en temps réel
    localDB.sync(remoteDB, { live: true, retry: true })
      .on('change', (info) => {
        console.log("Changement détecté après retour online:", info);
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
  fetchUsers()
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
  <h2>Utilisateurs</h2>
  <button @click="addUser" style="background-color: #A4D65E; color: white;">Ajouter un utilisateur</button>

  <article v-for="user in usersData" :key="user._id">
    <input v-model="user.username" />
    <input v-model="user.email" />
    <p>Followers: {{ user.followers }} | Following: {{ user.following }}</p>
    <button @click="updateUser(user)">Sauvegarder</button>
    <button @click="deleteUser(user)">Supprimer</button>
  </article>
</div>
  <div>
    <button @click="generatePosts(10)">Générer 10 posts</button>
    <input v-model="searchQuery" placeholder="Rechercher par titre" />
    <button @click="searchPosts">Rechercher</button>
    <button @click="() => fetchData()">Réinitialiser</button>
  </div>

  <div>
    <label>Filtrer par catégorie : </label>
    <select v-model="selectedCategory" @change="() => fetchData()">
      <option value="all">Toutes les catégories</option>
      <option value="vacances">Vacances</option>
      <option value="lifestyle">Lifestyle</option>
      <option value="éducatif">Éducatif</option>
      <option value="général">Général</option>
    </select>
    <button @click="changeSort('likes')">Trier par likes</button>
    <button @click="changeSort('creation_date')">Trier par date</button>
  </div>

  <article v-for="post in displayedPosts" :key="(post as any)._id">
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
      <h4>Image</h4>
      <img v-if="post.image_url" :src="post.image_url" style="max-width: 300px; display: block; margin: 10px 0;" />
      <input
        type="file"
        accept="image/*"
        @change="(e) => {
          const file = (e.target as HTMLInputElement).files?.[0];
          if (file) uploadImage(post, file);
        }"
      />
      <button v-if="post.image_url" @click="deleteImage(post)">Supprimer l'image</button>
    </div>

     <div>
      <h4>Commentaires ({{ post.comments?.length || 0 }})</h4>
      <ul v-if="post.comments?.length">
        <li v-for="comment in getDisplayedComments(post)" :key="comment.id">
          {{ comment.content }}
          <button @click="deleteComment(post, comment.id)">X</button>
        </li>
      </ul>
      <button
        v-if="post.comments && post.comments.length > 1"
        @click="toggleComments(post._id || '')"
      >
        {{ showAllComments[post._id || ''] ? 'Masquer les commentaires' : 'Voir tous les commentaires' }}
      </button>

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

  <div v-if="displayedPosts.length < postsData.length" style="text-align: center; margin: 20px 0;">
    <button @click="showMore">Voir plus</button>
  </div>
</template>
