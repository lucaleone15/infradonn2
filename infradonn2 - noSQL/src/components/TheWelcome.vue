<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

// --- TYPES ---
interface Comment {
  _id?: string
  _rev?: string
  type: string
  postId: string
  author: string
  content: string
  date: string
  _conflicts?: string[]
}

interface Post {
  _id?: string
  _rev?: string
  type: string
  title: string
  author: string
  content: string
  likes: number
  date: string
  _conflicts?: string[]
  comments?: Comment[]
  showComments?: boolean
}

// --- STATE ---
const localDB = ref<any>(null)
const remoteDB = ref<any>(null)
const posts = ref<Post[]>([])

// Sync & Status
const isOnline = ref(true)
const statusMsg = ref('Initialisation...')
const syncHandler = ref<any>(null)

// Recherche & Tri
const searchQuery = ref('')
const sortType = ref<'date' | 'likes'>('likes')
const sortDirection = ref<'asc' | 'desc'>('desc')

// Form
const newPost = ref({ title: '', author: '', content: '' })

// --- INIT ---
onMounted(async () => {
  localDB.value = new PouchDB('posts_local')
  remoteDB.value = 'http://admin:admin@localhost:5984/test_infradonn2'

  try {
    await localDB.value.createIndex({
      index: { fields: ['likes'] },
    })
    await localDB.value.createIndex({
      index: { fields: ['postId'] },
    })
  } catch (e) {
    console.error('Erreur Index', e)
  }

  updateSyncState()
  fetchData()

  localDB.value
    .changes({ since: 'now', live: true, include_docs: true, conflicts: true })
    .on('change', () => fetchData())
})

// --- SYNCHRONISATION ---
const updateSyncState = () => {
  if (syncHandler.value) {
    syncHandler.value.cancel()
    syncHandler.value = null
  }

  if (isOnline.value) {
    statusMsg.value = 'En ligne (Synchronisation active)'
    syncHandler.value = localDB.value
      .sync(remoteDB.value, {
        live: true,
        retry: true,
        conflicts: true,
      })
      .on('error', (err: any) => {
        console.error('Erreur sync:', err)
        statusMsg.value = 'Erreur connexion serveur'
      })
      .on('paused', () => (statusMsg.value = 'En ligne (En attente)'))
      .on('active', () => (statusMsg.value = 'Synchronisation en cours...'))
      .on('change', (info: any) => {
        console.log('Changement sync:', info)
        if (info.direction === 'pull') {
          fetchData()
        }
      })
  } else {
    statusMsg.value = 'Hors ligne (Mode local uniquement)'
  }
}

// --- RECHERCHE & TRI ---
const fetchData = async () => {
  if (!localDB.value) return

  try {
    let postsList: Post[] = []

    if (sortType.value === 'likes') {
      const selector: any = {
        type: 'post',
        likes: { $gte: 0 },
      }

      if (searchQuery.value) {
        selector.title = { $regex: RegExp(searchQuery.value, 'i') }
      }

      const result = await localDB.value.find({
        selector: selector,
        sort: [{ likes: sortDirection.value }],
        limit: 10,
        conflicts: true,
      })

      postsList = result.docs
    } else {
      const result = await localDB.value.allDocs({
        include_docs: true,
        conflicts: true,
        startkey: sortDirection.value === 'desc' ? 'post_\ufff0' : 'post_',
        endkey: sortDirection.value === 'desc' ? 'post_' : 'post_\ufff0',
        descending: sortDirection.value === 'desc',
      })

      let docs = result.rows
        .map((row: any) => row.doc)
        .filter((doc: any) => doc && doc.type === 'post')

      if (searchQuery.value) {
        const regex = new RegExp(searchQuery.value, 'i')
        docs = docs.filter((doc: any) => regex.test(doc.title))
      }

      postsList = docs
    }

    for (const post of postsList) {
      const comments = await fetchCommentsForPost(post._id!)
      post.comments = comments

      const existing = posts.value.find((p) => p._id === post._id)
      post.showComments = existing ? existing.showComments : false
    }

    posts.value = postsList
  } catch (e) {
    console.error('Erreur Find', e)
  }
}

const fetchCommentsForPost = async (postId: string): Promise<Comment[]> => {
  try {
    const result = await localDB.value.find({
      selector: {
        type: 'comment',
        postId: postId,
      },
      conflicts: true,
    })
    return result.docs
  } catch (e) {
    console.error('Erreur récupération commentaires:', e)
    return []
  }
}

watch([searchQuery, sortType, sortDirection], fetchData)

// --- GESTION CONFLITS ---
const resolveConflict = async (post: Post) => {
  if (!post._conflicts || post._conflicts.length === 0) return

  try {
    const conflictRev = post._conflicts[0]
    const versionDistante = await localDB.value.get(post._id, { rev: conflictRev })

    const choice = confirm(
      `CONFLIT DÉTECTÉ\n\n` +
        `Deux versions différentes existent pour "${post.title}":\n\n` +
        `VERSION 1 (Locale):\n` +
        `${post.content.substring(0, 100)}...\n` +
        `Likes: ${post.likes}\n\n` +
        `VERSION 2 (Distante):\n` +
        `${versionDistante.content.substring(0, 100)}...\n` +
        `Likes: ${versionDistante.likes}\n\n` +
        `OK = Garder Version 1 (Locale)\n` +
        `ANNULER = Garder Version 2 (Distante)`,
    )

    if (choice) {
      await localDB.value.remove(post._id, conflictRev)
      statusMsg.value = '✓ Version locale conservée'
    } else {
      const versionLocale = await localDB.value.get(post._id)

      const nouvelleVersion = {
        ...versionLocale,
        title: versionDistante.title,
        content: versionDistante.content,
        author: versionDistante.author,
        likes: versionDistante.likes,
        date: versionDistante.date,
      }

      await localDB.value.put(nouvelleVersion)
      await localDB.value.remove(post._id, conflictRev)
      statusMsg.value = '✓ Version distante conservée'
    }

    setTimeout(() => {
      statusMsg.value = isOnline.value ? 'En ligne (En attente)' : 'Hors ligne'
    }, 2000)
  } catch (e) {
    console.error('Erreur résolution conflit:', e)
    alert('Erreur lors de la résolution du conflit')
  }
}

const resolveCommentConflict = async (comment: Comment) => {
  if (!comment._conflicts || comment._conflicts.length === 0) return

  try {
    const conflictRev = comment._conflicts[0]
    const versionDistante = await localDB.value.get(comment._id, { rev: conflictRev })

    const choice = confirm(
      `CONFLIT COMMENTAIRE\n\n` +
        `VERSION 1 (Locale):\n` +
        `${comment.content}\n\n` +
        `VERSION 2 (Distante):\n` +
        `${versionDistante.content}\n\n` +
        `OK = Garder Version 1 (Locale)\n` +
        `ANNULER = Garder Version 2 (Distante)`,
    )

    if (choice) {
      await localDB.value.remove(comment._id, conflictRev)
    } else {
      const versionLocale = await localDB.value.get(comment._id)
      const nouvelleVersion = {
        ...versionLocale,
        author: versionDistante.author,
        content: versionDistante.content,
        date: versionDistante.date,
      }
      await localDB.value.put(nouvelleVersion)
      await localDB.value.remove(comment._id, conflictRev)
    }

    await fetchData()
  } catch (e) {
    console.error('Erreur résolution conflit commentaire:', e)
    alert('Erreur lors de la résolution du conflit')
  }
}

// --- CRUD POSTS ---
const createPost = async () => {
  if (!newPost.value.title || !newPost.value.author) return alert('Champs requis')

  const doc: Post = {
    _id: `post_${Date.now()}`,
    type: 'post',
    title: newPost.value.title,
    author: newPost.value.author,
    content: newPost.value.content,
    likes: 0,
    date: new Date().toLocaleString(),
  }

  try {
    await localDB.value.put(doc)
    newPost.value = { title: '', author: '', content: '' }
  } catch (e) {
    console.error(e)
  }
}

const updatePost = async (post: Post) => {
  const doc = await localDB.value.get(post._id)
  const newTxt = prompt('Modifier le contenu :', doc.content)
  if (newTxt !== null) {
    doc.content = newTxt
    await localDB.value.put(doc)
  }
}

const deletePost = async (post: Post) => {
  if (confirm('Supprimer ce post et tous ses commentaires ?')) {
    try {
      const doc = await localDB.value.get(post._id)
      await localDB.value.remove(doc)

      const comments = await fetchCommentsForPost(post._id!)
      for (const comment of comments) {
        await localDB.value.remove(comment._id, comment._rev)
      }
    } catch (e) {
      console.error('Erreur suppression:', e)
    }
  }
}

const addLike = async (post: Post) => {
  const doc = await localDB.value.get(post._id)
  doc.likes++
  await localDB.value.put(doc)
}

// --- CRUD COMMENTAIRES ---
const addComment = async (post: Post) => {
  const author = prompt('Votre Nom :')
  if (!author) return
  const content = prompt('Commentaire :')
  if (!content) return

  const commentDoc: Comment = {
    _id: `comment_${Date.now()}`,
    type: 'comment',
    postId: post._id!,
    author,
    content,
    date: new Date().toLocaleTimeString(),
  }

  try {
    await localDB.value.put(commentDoc)

    const p = posts.value.find((x) => x._id === post._id)
    if (p) p.showComments = true
  } catch (e) {
    console.error('Erreur ajout commentaire:', e)
  }
}

const updateComment = async (comment: Comment) => {
  try {
    const doc = await localDB.value.get(comment._id)
    const newTxt = prompt('Modifier commentaire :', doc.content)
    if (newTxt !== null) {
      doc.content = newTxt
      await localDB.value.put(doc)
    }
  } catch (e) {
    console.error('Erreur modification commentaire:', e)
  }
}

const deleteComment = async (comment: Comment) => {
  if (confirm('Supprimer ce commentaire ?')) {
    try {
      const doc = await localDB.value.get(comment._id)
      await localDB.value.remove(doc)
    } catch (e) {
      console.error('Erreur suppression commentaire:', e)
    }
  }
}

const toggleComments = (post: Post) => {
  post.showComments = !post.showComments
}

// --- FACTORY ---
const runFactory = async () => {
  const docs = []
  const timestamp = Date.now()

  for (let i = 0; i < 10; i++) {
    const postId = `post_${timestamp + i}`
    docs.push({
      _id: postId,
      type: 'post',
      title: `Article Généré ${i}`,
      author: 'Bot Factory',
      content: 'Contenu de test généré automatiquement.',
      likes: Math.floor(Math.random() * 100),
      date: new Date().toLocaleString(),
    })

    const numComments = Math.floor(Math.random() * 3) + 1
    for (let j = 0; j < numComments; j++) {
      docs.push({
        _id: `comment_${timestamp + i}_${j}`,
        type: 'comment',
        postId: postId,
        author: `User${j + 1}`,
        content: `Commentaire test ${j + 1} pour l'article ${i}`,
        date: new Date().toLocaleTimeString(),
      })
    }
  }

  await localDB.value.bulkDocs(docs)
}
</script>

<template>
  <div class="container">
    <h1>TP InfraDonn2 - Blog NoSQL</h1>

    <div class="panel small">
      <label class="inline-toggle">
        <input type="checkbox" v-model="isOnline" @change="updateSyncState" />
        <span class="label-text">{{ isOnline ? 'En Ligne' : 'Hors Ligne' }}</span>
      </label>
      <span :class="{ success: isOnline, error: !isOnline }" style="font-weight: bold">
        {{ statusMsg }}
      </span>
    </div>

    <div class="panel controls">
      <button @click="runFactory">Générer Données (Factory)</button>

      <div class="search-row">
        <input v-model="searchQuery" placeholder="Rechercher un titre..." />
      </div>

      <div class="search-row" style="justify-content: space-between">
        <label>Trier par :</label>
        <select v-model="sortType" style="padding: 5px; border-radius: 4px">
          <option value="date">Date (Chronologique)</option>
          <option value="likes">Top 10 Likes</option>
        </select>

        <select v-model="sortDirection" style="padding: 5px; border-radius: 4px">
          <option value="desc">Décroissant</option>
          <option value="asc">Croissant</option>
        </select>
      </div>
    </div>

    <div class="panel">
      <h3>Nouveau Message</h3>
      <div class="form">
        <input v-model="newPost.title" placeholder="Titre" />
        <input v-model="newPost.author" placeholder="Auteur" />
        <textarea v-model="newPost.content" placeholder="Contenu..."></textarea>
        <div class="form-actions">
          <button @click="createPost">Publier</button>
        </div>
      </div>
    </div>

    <hr />

    <div v-if="posts.length === 0" class="empty">Aucun post trouvé.</div>

    <div v-for="post in posts" :key="post._id" class="post">
      <div v-if="post._conflicts && post._conflicts.length > 0" class="conflict-banner">
        <div>
          <strong>Conflit Post !</strong>
          <span style="font-size: 0.9rem; margin-left: 10px">
            Ce post a été modifié en même temps à deux endroits
          </span>
        </div>
        <button @click="resolveConflict(post)" class="resolve-btn">Résoudre</button>
      </div>

      <div class="post-head">
        <h3>{{ post.title }}</h3>
        <div class="meta">
          {{ post.date }} | Par {{ post.author }} |
          <span class="pending">Likes: {{ post.likes }}</span>
        </div>
      </div>

      <p class="content">{{ post.content }}</p>

      <div class="post-actions">
        <button @click="addLike(post)">Like</button>
        <button @click="updatePost(post)">Modifier</button>
        <button @click="deletePost(post)">Supprimer</button>
        <button @click="addComment(post)">Commenter</button>
      </div>

      <div
        v-if="post.comments && post.comments.length > 0 && post.comments[0]"
        class="first-comment"
      >
        <div
          v-if="post.comments[0]._conflicts && post.comments[0]._conflicts.length > 0"
          class="conflict-banner-small"
        >
          <strong>Conflit commentaire</strong>
          <button
            style="padding: 2px 8px; font-size: 0.75rem"
            @click="resolveCommentConflict(post.comments[0])"
          >
            Résoudre
          </button>
        </div>

        <div class="comment-item">
          <div style="display: flex; justify-content: space-between; align-items: start">
            <div>
              <strong>{{ post.comments[0]?.author }}:</strong> {{ post.comments[0]?.content }}
              <br />
              <small style="color: var(--muted)">{{ post.comments[0]?.date }}</small>
            </div>
            <div>
              <button
                style="padding: 2px 5px; font-size: 0.7rem; margin-right: 5px"
                @click="post.comments[0] && updateComment(post.comments[0])"
              >
                Modifier
              </button>
              <button
                style="padding: 2px 5px; font-size: 0.7rem; background: var(--error)"
                @click="post.comments[0] && deleteComment(post.comments[0])"
              >
                Supprimer
              </button>
            </div>
          </div>
        </div>
      </div>

      <div v-if="post.comments && post.comments.length > 0" style="margin-top: 10px">
        <a
          href="#"
          @click.prevent="toggleComments(post)"
          style="color: var(--accent); text-decoration: none; font-weight: 600"
        >
          {{ post.showComments ? '▼ Masquer' : '▶ Voir tous' }} les commentaires ({{
            post.comments.length
          }})
        </a>

        <div v-if="post.showComments" class="comments">
          <ul>
            <li v-for="c in post.comments.slice(1)" :key="c._id">
              <div v-if="c._conflicts && c._conflicts.length > 0" class="conflict-banner-small">
                <strong>Conflit</strong>
                <button
                  style="padding: 2px 8px; font-size: 0.75rem"
                  @click="resolveCommentConflict(c)"
                >
                  Résoudre
                </button>
              </div>

              <div style="display: flex; justify-content: space-between; align-items: start">
                <div>
                  <strong>{{ c.author }}:</strong> {{ c.content }}
                  <br />
                  <small style="color: var(--muted)">{{ c.date }}</small>
                </div>
                <div>
                  <button
                    style="padding: 2px 5px; font-size: 0.7rem; margin-right: 5px"
                    @click="updateComment(c)"
                  >
                    Modifier
                  </button>
                  <button
                    style="padding: 2px 5px; font-size: 0.7rem; background: var(--error)"
                    @click="deleteComment(c)"
                  >
                    Supprimer
                  </button>
                </div>
              </div>
            </li>
          </ul>
        </div>
      </div>

      <div v-else style="margin-top: 10px; color: var(--muted); font-size: 0.9rem">
        Aucun commentaire pour le moment
      </div>
    </div>
  </div>
</template>

<style scoped>
:root {
  --bg: #0f172a;
  --panel: #1e293b;
  --accent: #3b82f6;
  --muted: #9ca3af;
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

.panel {
  background: var(--panel);
  padding: 1rem;
  border-radius: 10px;
  margin-bottom: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.08);
}
.panel.small {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

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

.search-row {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  width: 100%;
  margin-top: 10px;
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

.first-comment {
  margin-top: 15px;
  padding: 10px;
  background: rgba(59, 130, 246, 0.1);
  border-left: 3px solid var(--accent);
  border-radius: 4px;
}

.comment-item {
  font-size: 0.95rem;
}

.comments {
  margin-top: 0.5rem;
  padding-left: 1rem;
  border-top: 1px solid #334155;
  padding-top: 10px;
}
.comments li {
  margin-bottom: 0.5rem;
  font-size: 0.95rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  padding-bottom: 5px;
}

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

.conflict-banner {
  background: #7f1d1d;
  padding: 12px;
  margin-bottom: 12px;
  border-radius: 6px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border: 2px solid #ef4444;
}

.conflict-banner strong {
  color: #fca5a5;
  font-size: 1rem;
}

.conflict-banner span {
  color: #fecaca;
}

.conflict-banner-small {
  background: #7f1d1d;
  padding: 6px 10px;
  margin-bottom: 8px;
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border: 1px solid #ef4444;
  font-size: 0.85rem;
}

.conflict-banner-small strong {
  color: #fca5a5;
}

.resolve-btn {
  background: #ef4444 !important;
  padding: 8px 16px !important;
  font-size: 0.9rem !important;
  white-space: nowrap;
}

.resolve-btn:hover {
  background: #dc2626 !important;
}

@media (max-width: 640px) {
  .controls {
    flex-direction: column;
  }
  .post-head {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.25rem;
  }
  .conflict-banner {
    flex-direction: column;
    gap: 10px;
    align-items: stretch;
  }
}
</style>
