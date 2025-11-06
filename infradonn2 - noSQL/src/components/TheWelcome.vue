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

const localDB = ref<any>()
const remoteDB = ref<any>()
const posts = ref<Post[]>([])
const syncStatus = ref<string>('Non synchronisé')
const isSyncing = ref<boolean>(false)

const newPost = ref<Post>({
  title: '',
  author: '',
  content: '',
  date: new Date().toLocaleDateString(),
})

const initDB = () => {
  localDB.value = new PouchDB('posts_local')
  remoteDB.value = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
  syncFromRemote()
}

const fetchPosts = async () => {
  if (!localDB.value) return
  const result = await localDB.value.allDocs({ include_docs: true })
  posts.value = result.rows.map((r: any) => r.doc as Post)
}

const syncFromRemote = async () => {
  if (!localDB.value || !remoteDB.value) return
  syncStatus.value = 'Téléchargement...'
  isSyncing.value = true
  try {
    await localDB.value.replicate.from(remoteDB.value)
    syncStatus.value = 'Synchronisé ✓'
    await fetchPosts()
  } catch (e) {
    syncStatus.value = 'Erreur ✗'
  } finally {
    isSyncing.value = false
  }
}

const syncBidirectional = async () => {
  if (!localDB.value || !remoteDB.value) return
  syncStatus.value = 'Synchronisation bidirectionnelle...'
  isSyncing.value = true
  try {
    await localDB.value.sync(remoteDB.value)
    syncStatus.value = 'Synchronisé ✓'
    await fetchPosts()
  } catch (e) {
    syncStatus.value = 'Erreur ✗'
  } finally {
    isSyncing.value = false
  }
}

const createPost = async () => {
  if (!localDB.value) return
  await localDB.value.post(newPost.value)
  newPost.value = { title: '', author: '', content: '', date: new Date().toLocaleDateString() }
  await fetchPosts()
  syncStatus.value = 'Modifications locales non synchronisées'
}

const updatePost = async (post: Post) => {
  const title = prompt('Nouveau titre:', post.title)
  if (!title) return
  const content = prompt('Nouveau contenu:', post.content)
  if (!content || !localDB.value) return
  await localDB.value.put({
    _id: post._id,
    _rev: post._rev,
    title,
    author: post.author,
    content,
    date: post.date,
  })
  await fetchPosts()
  syncStatus.value = 'Modifications locales non synchronisées'
}

const deletePost = async (post: Post) => {
  if (!localDB.value || !post._id || !post._rev) return
  await localDB.value.remove(post._id, post._rev)
  await fetchPosts()
  syncStatus.value = 'Modifications locales non synchronisées'
}

onMounted(() => initDB())
</script>

<template>
  <div class="container">
    <h1>Gestion des Posts - Réplication</h1>

    <div class="sync-panel">
      <h3>Synchronisation</h3>
      <p>
        Statut:
        <strong
          :class="{
            success: syncStatus.includes('✓'),
            error: syncStatus.includes('✗'),
            pending: !syncStatus.includes('✓') && !syncStatus.includes('✗'),
          }"
        >
          {{ syncStatus }}
        </strong>
      </p>
      <div class="buttons">
        <button @click="syncBidirectional" :disabled="isSyncing">🔄 Synchroniser</button>
      </div>
      <p class="note">
        💡 Les modifications sont d'abord enregistrées localement. Cliquez sur "Synchroniser" pour
        mettre à jour le serveur distant.
      </p>
    </div>

    <h2>Posts ({{ posts.length }})</h2>
    <div v-if="posts.length === 0" class="empty">Aucun post pour le moment</div>

    <article v-for="post in posts" :key="post._id" class="post">
      <h3>{{ post.title }}</h3>
      <p class="content">{{ post.content }}</p>
      <p class="meta">
        <strong>{{ post.author }}</strong> – {{ post.date }}
      </p>
      <div class="buttons">
        <button @click="updatePost(post)">✏️ Modifier</button>
        <button @click="deletePost(post)">🗑️ Supprimer</button>
      </div>
    </article>

    <hr />

    <h2>Créer un nouveau post</h2>
    <div class="form">
      <input v-model="newPost.title" placeholder="Titre" />
      <input v-model="newPost.author" placeholder="Auteur" />
      <textarea v-model="newPost.content" rows="5" placeholder="Contenu"></textarea>
      <button @click="createPost">✅ Publier (localement)</button>
    </div>
  </div>
</template>

<style scoped>
.container {
  padding: 2rem;
  max-width: 800px;
  margin: auto;
  color: #000;
}
h1,
h2 {
  color: #f0f0f0;
}
.sync-panel {
  background: #f5f5f5;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 2rem;
}
.sync-panel h3 {
  margin-top: 0;
  color: #000;
}
.buttons {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-top: 1rem;
}
button {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  background: #1976d2;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.note {
  font-size: 0.85rem;
  color: #333;
  margin-top: 0.5rem;
}
.success {
  color: green;
}
.error {
  color: red;
}
.pending {
  color: orange;
}
.post {
  border: 1px solid #ddd;
  padding: 1rem;
  margin: 1rem 0;
  border-radius: 4px;
  background: white;
}
.post h3 {
  margin-top: 0;
  color: #000;
}
.post .content {
  white-space: pre-wrap;
}
.post .meta {
  color: #555;
  font-size: 0.9rem;
  margin-bottom: 0.5rem;
}
.empty {
  padding: 1rem;
  background: #f0f0f0;
  border-radius: 4px;
}
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.form input,
.form textarea {
  padding: 0.5rem;
  font-size: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}
.form textarea {
  resize: vertical;
}
</style>
