<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

interface Post {
  _id?: string
  _rev?: string
  title: string
  author: string
  content: string
  date: string
}

const localDB = ref<any>(null)
const remoteDB = ref<any>(null)
const posts = ref<Post[]>([])
const syncStatus = ref<string>('Non synchronisé')
const isSyncing = ref<boolean>(false)
const isOnline = ref<boolean>(true)

const newPost = ref<Post>({
  title: '',
  author: '',
  content: '',
  date: new Date().toLocaleDateString(),
})

// --- INIT DB ---
const initDB = async () => {
  localDB.value = new PouchDB('posts_local')
  remoteDB.value = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
  await fetchPosts()
  listenToChanges()
  // Si on démarre en online, on tente une réplication initiale
  if (isOnline.value) {
    syncFromRemote()
  }
}

// --- FETCH posts depuis la base locale ---
const fetchPosts = async () => {
  if (!localDB.value) return
  try {
    const result = await localDB.value.allDocs({ include_docs: true })
    posts.value = result.rows
      .map((r: any) => r.doc as Post)
      // trier par date (si date au format lisible) ou par _id pour stabilité
      .sort((a: any, b: any) => {
        if (a.date && b.date) return a.date < b.date ? 1 : -1
        return (a._id || '').localeCompare(b._id || '')
      })
  } catch (err) {
    console.error('fetchPosts error', err)
  }
}

// --- SYNC from remote (pull) ---
const syncFromRemote = async () => {
  if (!isOnline.value || !localDB.value || !remoteDB.value) return
  syncStatus.value = 'Téléchargement...'
  isSyncing.value = true
  try {
    await localDB.value.replicate.from(remoteDB.value)
    syncStatus.value = 'Synchronisé ✓'
    await fetchPosts()
  } catch (e) {
    console.error('replicate.from error', e)
    syncStatus.value = 'Erreur ✗'
  } finally {
    isSyncing.value = false
  }
}

// --- SYNC bidirectionnel (push+pull) ---
const syncBidirectional = async () => {
  if (!isOnline.value || !localDB.value || !remoteDB.value) {
    syncStatus.value = isOnline.value
      ? 'DB non initialisée'
      : 'Hors ligne - impossible de synchroniser'
    return
  }
  syncStatus.value = 'Synchronisation bidirectionnelle...'
  isSyncing.value = true
  try {
    await localDB.value.sync(remoteDB.value)
    syncStatus.value = 'Synchronisé ✓'
    await fetchPosts()
  } catch (e) {
    console.error('sync error', e)
    syncStatus.value = 'Erreur ✗'
  } finally {
    isSyncing.value = false
  }
}

// --- CRUD ---
const createPost = async () => {
  if (!localDB.value) return
  const doc: Post & { _id: string } = {
    _id: `post_${Date.now()}_${Math.floor(Math.random() * 1000)}`,
    title: newPost.value.title || '(sans titre)',
    author: newPost.value.author || 'Anonyme',
    content: newPost.value.content || '',
    date: newPost.value.date || new Date().toLocaleDateString(),
  }
  try {
    await localDB.value.put(doc)
    newPost.value = { title: '', author: '', content: '', date: new Date().toLocaleDateString() }
    await fetchPosts()
    syncStatus.value = 'Modifications locales non synchronisées'
  } catch (err) {
    console.error('createPost error', err)
  }
}

const updatePost = async (post: Post) => {
  if (!localDB.value || !post._id) return
  const title = prompt('Nouveau titre :', post.title)
  if (title === null) return
  const content = prompt('Nouveau contenu :', post.content)
  if (content === null) return
  try {
    const toPut = {
      _id: post._id,
      _rev: post._rev,
      title,
      author: post.author,
      content,
      date: post.date,
    }
    await localDB.value.put(toPut)
    await fetchPosts()
    syncStatus.value = 'Modifications locales non synchronisées'
  } catch (err) {
    console.error('updatePost error', err)
  }
}

const deletePost = async (post: Post) => {
  if (!localDB.value || !post._id || !post._rev) return
  if (!confirm(`Supprimer "${post.title}" ?`)) return
  try {
    await localDB.value.remove(post._id, post._rev)
    await fetchPosts()
    syncStatus.value = 'Modifications locales non synchronisées'
  } catch (err) {
    console.error('deletePost error', err)
  }
}

// --- TOGGLE connexion (ne pas inverser deux fois la valeur) ---
// Le checkbox met déjà isOnline.value à jour grâce au v-model.
// Ici on réagit à la nouvelle valeur, sans toggler à nouveau.
const toggleConnection = () => {
  if (isOnline.value) {
    // passé en ligne
    syncStatus.value = 'Reconnexion : synchronisation en cours...'
    // lancement de la sync bidirectionnelle en tâche immédiate
    syncBidirectional()
  } else {
    // passé en offline
    syncStatus.value = 'Mode offline activé 📴'
    // On n'annule pas les listeners locaux ; on bloque simplement les réplications
  }
}

// --- Écoute des changements locaux (live) ---
let changesFeed: any = null
const listenToChanges = () => {
  if (!localDB.value) return
  // Si un feed existait, cancel
  if (changesFeed && typeof changesFeed.cancel === 'function') {
    try {
      changesFeed.cancel()
    } catch (e) {
      /* ignore */
    }
  }
  changesFeed = localDB.value
    .changes({
      since: 'now',
      live: true,
      include_docs: true,
    })
    .on('change', async (ch: any) => {
      // Un changement local ou provenant d'une réplication -> raffraîchir la liste
      await fetchPosts()
    })
    .on('error', (err: any) => {
      console.error('changes feed error', err)
    })
}

// --- FACTORY pour générer des documents de test ---
const generateFakePosts = async (count = 50) => {
  if (!localDB.value) return
  const now = Date.now()
  const bulk = Array.from({ length: count }).map((_, i) => ({
    _id: `post_${now}_${i}`,
    title: `Post ${i + 1}`,
    author: `Auteur ${Math.ceil(Math.random() * 6)}`,
    content: `Texte de test pour le post ${i + 1}\nLigne supplémentaire.`,
    date: new Date().toLocaleDateString(),
  }))
  try {
    await localDB.value.bulkDocs(bulk)
    await fetchPosts()
    syncStatus.value = `Généré ${count} posts localement`
  } catch (err) {
    console.error('generateFakePosts error', err)
  }
}

// --- RECHERCHE (sans plugin) : filtrage côté client ---
const searchQuery = ref('')
const searchResults = ref<Post[]>([])

const searchLocal = () => {
  const q = searchQuery.value.trim().toLowerCase()
  if (!q) {
    searchResults.value = []
    return
  }
  // recherche sur author et title (insensible à la casse)
  searchResults.value = posts.value.filter((p) => {
    return (
      (p.author && p.author.toLowerCase().includes(q)) ||
      (p.title && p.title.toLowerCase().includes(q))
    )
  })
}

// nettoyage du feed quand le composant est détruit (bonne pratique)
onMounted(() => initDB())
</script>

<template>
  <div class="container">
    <h1>Gestion des Posts — Réplication</h1>

    <!-- Online / Offline -->
    <div class="panel small">
      <label class="inline-toggle">
        <input type="checkbox" v-model="isOnline" @change="toggleConnection" />
        <span class="slider"></span>
        <span class="label-text">{{ isOnline ? '🟢 Online' : '🔴 Offline' }}</span>
      </label>
      <p class="status">
        Statut sync :
        <strong
          :class="{
            success: syncStatus.includes('✓'),
            error: syncStatus.includes('✗'),
            pending: !syncStatus.includes('✓') && !syncStatus.includes('✗'),
          }"
          >{{ syncStatus }}</strong
        >
      </p>
    </div>

    <!-- Sync controls -->
    <div class="panel">
      <div class="controls">
        <button @click="syncBidirectional" :disabled="isSyncing || !isOnline">
          🔄 Synchroniser
        </button>
        <button @click="syncFromRemote" :disabled="isSyncing || !isOnline">
          ⬇️ Télécharger depuis serveur
        </button>
        <button @click="generateFakePosts(10)">🧰 Générer 10 posts</button>
      </div>
      <p class="note">
        Les modifications sont d'abord enregistrées localement. Cliquez sur
        <em>Synchroniser</em> pour pousser/puller vers CouchDB.
      </p>
    </div>

    <!-- Recherche -->
    <div class="panel">
      <div class="search-row">
        <input
          v-model="searchQuery"
          @input="searchLocal"
          placeholder="Rechercher par auteur ou titre..."
        />
        <button @click="searchLocal">🔍</button>
      </div>
      <div v-if="searchResults.length > 0" class="search-results">
        <h4>Résultats ({{ searchResults.length }})</h4>
        <ul>
          <li v-for="p in searchResults" :key="p._id">
            <strong>{{ p.title }}</strong> — <em>{{ p.author }}</em>
          </li>
        </ul>
      </div>
    </div>

    <!-- Liste des posts -->
    <h2>Posts ({{ posts.length }})</h2>
    <div v-if="posts.length === 0" class="empty">Aucun post pour le moment</div>

    <article v-for="post in posts" :key="post._id" class="post">
      <div class="post-head">
        <h3>{{ post.title }}</h3>
        <div class="meta">
          <span class="author">{{ post.author }}</span>
          <span class="date">— {{ post.date }}</span>
        </div>
      </div>
      <p class="content">{{ post.content }}</p>
      <div class="post-actions">
        <button @click="updatePost(post)">✏️ Modifier</button>
        <button @click="deletePost(post)">🗑️ Supprimer</button>
      </div>
    </article>

    <hr />

    <!-- Form création -->
    <div class="panel">
      <h3>Créer un nouveau post</h3>
      <div class="form">
        <input v-model="newPost.title" placeholder="Titre" />
        <input v-model="newPost.author" placeholder="Auteur" />
        <textarea v-model="newPost.content" rows="5" placeholder="Contenu"></textarea>
        <div class="form-actions">
          <button @click="createPost">✅ Publier (localement)</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
:root {
  --bg: #0f172a; /* fond principal sombre */
  --panel: #1e293b; /* panneau plus clair */
  --accent: #3b82f6; /* bleu vif */
  --muted: #9ca3af; /* texte secondaire */
  --success: #22c55e;
  --error: #ef4444;
  --pending: #facc15;
}

.container {
  padding: 2rem;
  max-width: 900px;
  margin: 0 auto;
  color: #ffffff;
  background: var(--bg);
  font-family:
    system-ui,
    -apple-system,
    'Segoe UI',
    Roboto,
    'Helvetica Neue',
    Arial;
}

h1,
h2,
h3 {
  color: #ffffff;
}

h1 {
  font-size: 1.6rem;
  margin-bottom: 0.75rem;
}
h2 {
  margin-top: 1.25rem;
}
h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.05rem;
}

/* Panels */
.panel {
  background: var(--panel);
  padding: 1rem;
  border-radius: 10px;
  margin-bottom: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.08);
  color: white;
}
.panel.small {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

/* Toggle inline */
.inline-toggle {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
}
.inline-toggle input[type='checkbox'] {
  width: 1.2rem;
  height: 1.2rem;
  margin: 0;
  accent-color: var(--accent);
}
.inline-toggle .label-text {
  font-weight: 600;
  color: white;
}

/* Controls */
.controls {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}
button {
  padding: 0.55rem 0.9rem;
  font-size: 0.95rem;
  background: var(--accent);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  transition: 0.2s;
}
button:hover:not(:disabled) {
  background: #2563eb;
}
button:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* Search */
.search-row {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}
.search-row input {
  flex: 1;
  padding: 0.5rem;
  border-radius: 8px;
  border: 1px solid #334155;
  background: #0f172a;
  color: white;
}
.search-row input::placeholder {
  color: #9ca3af;
}

/* Posts */
.post {
  border: 1px solid #334155;
  padding: 1rem;
  margin: 0.75rem 0;
  border-radius: 8px;
  background: #1e293b;
  color: white;
}
.post-head {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 1rem;
}
.post .meta {
  color: var(--muted);
  font-size: 0.9rem;
}
.post .content {
  white-space: pre-wrap;
  margin: 0.5rem 0 0.75rem;
  color: #e2e8f0;
}
.post-actions {
  display: flex;
  gap: 0.5rem;
}
.post-actions button {
  background: #475569;
}
.post-actions button:hover {
  background: #64748b;
}

/* Form */
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.form input,
.form textarea {
  padding: 0.6rem;
  font-size: 1rem;
  border: 1px solid #334155;
  border-radius: 8px;
  background: #0f172a;
  color: white;
}
.form input::placeholder,
.form textarea::placeholder {
  color: #9ca3af;
}
.form-actions {
  margin-top: 0.25rem;
}

/* Status classes */
.success {
  color: var(--success);
  font-weight: 700;
}
.error {
  color: var(--error);
  font-weight: 700;
}
.pending {
  color: var(--pending);
  font-weight: 700;
}

.empty {
  padding: 1rem;
  background: #1e293b;
  border-radius: 8px;
  color: var(--muted);
}

/* responsive */
@media (max-width: 640px) {
  .controls {
    flex-direction: column;
  }
  .post-head {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.25rem;
  }
}
</style>
