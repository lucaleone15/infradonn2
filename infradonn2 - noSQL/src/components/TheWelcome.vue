<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue'
import PouchDB from 'pouchdb'

interface Comment {
  id: string
  author: string
  content: string
  date: string
}

interface Post {
  _id?: string
  _rev?: string
  title: string
  author: string
  content: string
  date: string
  comments?: Comment[]
}

// Référence à la base de données
const storage = ref<any>(null)
// Données stockées
const postsData = ref<Post[]>([])

// Formulaire pour ajouter / modifier post
const formPost = reactive<Omit<Post, "_id" | "_rev">>({
  title: "",
  author: "",
  content: "",
  date: new Date().toISOString(),
  comments: []
})

// Formulaire pour ajouter un commentaire
const commentForm = reactive<Omit<Comment, "id" | "date">>({
  author: "",
  content: ""
})

// Mode édition
const editId = ref<string | null>(null)
const editRev = ref<string | null>(null)

const initDatabase = () => {
  storage.value = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
}

const fetchData = async () => {
  if (!storage.value) return
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .filter((row: any) => !!row.doc)
      .map((row: any) => row.doc as Post)
  } catch (err) {
    console.error("Erreur fetch :", err)
  }
}

const addOrUpdatePost = async () => {
  if (!storage.value) return
  try {
    if (editId.value && editRev.value) {
      await storage.value.put({
        _id: editId.value,
        _rev: editRev.value,
        ...formPost
      })
      editId.value = null
      editRev.value = null
    } else {
      await storage.value.post(formPost)
    }
    formPost.title = ""
    formPost.author = ""
    formPost.content = ""
    formPost.date = new Date().toISOString()
    formPost.comments = []
    fetchData()
  } catch (err) {
    console.error("Erreur ajout/modif :", err)
  }
}

const editPost = (post: Post) => {
  formPost.title = post.title
  formPost.author = post.author
  formPost.content = post.content
  formPost.date = post.date
  formPost.comments = post.comments || []
  editId.value = post._id || null
  editRev.value = post._rev || null
}

const deletePost = async (post: Post) => {
  if (!storage.value || !post._id || !post._rev) return
  try {
    await storage.value.remove(post._id, post._rev)
    fetchData()
  } catch (err) {
    console.error("Erreur suppression :", err)
  }
}

const addComment = async (post: Post) => {
  if (!storage.value) return
  if (!commentForm.author || !commentForm.content) return

  const newComment: Comment = {
    id: Date.now().toString(),
    author: commentForm.author,
    content: commentForm.content,
    date: new Date().toISOString()
  }

  const updatedComments = post.comments ? [...post.comments, newComment] : [newComment]

  try {
    await storage.value.put({
      _id: post._id,
      _rev: post._rev,
      ...post,
      comments: updatedComments
    })
    commentForm.author = ""
    commentForm.content = ""
    fetchData()
  } catch (err) {
    console.error("Erreur ajout commentaire :", err)
  }
}

onMounted(() => {
  initDatabase()
  fetchData()
})
</script>

<template>
  <section class="container">
    <h1>📝 Mes Posts</h1>

    <!-- Formulaire Ajouter / Modifier -->
    <form @submit.prevent="addOrUpdatePost" class="form-post">
      <h2>{{ editId ? "✏️ Modifier le post" : "➕ Ajouter un post" }}</h2>
      <input v-model="formPost.title" placeholder="Titre du post" required />
      <input v-model="formPost.author" placeholder="Auteur" required />
      <textarea v-model="formPost.content" placeholder="Contenu du post" rows="4" required></textarea>
      <button type="submit">{{ editId ? "Mettre à jour" : "Ajouter" }}</button>
    </form>

    <!-- Liste des posts -->
    <div v-if="postsData.length === 0" class="empty">
      Aucun post trouvé.
    </div>

    <div v-else class="posts-list">
      <div v-for="post in postsData" :key="post._id" class="post-card">
        <h2>📌 {{ post.title }}</h2>
        <p class="author">🖋️ {{ post.author }} - <small>📅 {{ new Date(post.date).toLocaleDateString() }}</small></p>
        <p class="content">💬 {{ post.content }}</p>

        <!-- Commentaires -->
        <div v-if="post.comments && post.comments.length" class="comments">
          <h3>💭 Commentaires :</h3>
          <ul>
            <li v-for="comment in post.comments" :key="comment.id">
              <p>🗨️ {{ comment.content }}</p>
              <small>— 👤 {{ comment.author }}, ⏱️ {{ new Date(comment.date).toLocaleString() }}</small>
            </li>
          </ul>
        </div>

        <!-- Ajouter un commentaire -->
        <div class="comment-form">
          <input v-model="commentForm.author" placeholder="Votre nom" />
          <input v-model="commentForm.content" placeholder="Votre commentaire" />
          <button @click.prevent="addComment(post)">➕ Ajouter commentaire</button>
        </div>

        <div class="actions">
          <button @click="editPost(post)">✏️ Modifier post</button>
          <button @click="deletePost(post)">🗑️ Supprimer post</button>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.container {
  max-width: 900px;
  margin: 20px auto;
  font-family: Arial, sans-serif;
  color: #222;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #0077cc;
  font-size: 2em;
}

.form-post {
  border: 1px solid #ccc;
  padding: 15px;
  margin-bottom: 30px;
  border-radius: 8px;
  background: #e0f7fa; /* fond bleu clair */
}

.form-post h2 {
  margin-top: 0;
  color: #0077cc;
}

.form-post input,
.form-post textarea {
  width: 100%;
  margin-bottom: 10px;
  padding: 8px;
  box-sizing: border-box;
  border: 1px solid #aaa;
  border-radius: 4px;
  color: #222;
}

.form-post button {
  padding: 8px 12px;
  cursor: pointer;
  background-color: #0077cc;
  color: white;
  border: none;
  border-radius: 4px;
}

.empty {
  text-align: center;
  color: #555;
  font-size: 1.1em;
}

.posts-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.post-card {
  border: 1px solid #ddd;
  padding: 15px;
  border-radius: 8px;
  background: #fff3e0; /* fond doux orangé */
  box-shadow: 1px 1px 6px rgba(0,0,0,0.05);
}

.post-card h2 {
  margin-top: 0;
  color: #d35400;
}

.post-card .author {
  font-size: 0.9em;
  color: #555;
}

.post-card .content {
  margin: 10px 0;
  color: #333;
}

.comments {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #eee;
  background: #f1f8e9; /* fond doux vert clair pour commentaires */
  border-radius: 6px;
  padding: 10px;
}

.comments h3 {
  margin: 0 0 5px 0;
  color: #388e3c;
}

.comments ul {
  padding-left: 15px;
}

.comments li {
  margin-bottom: 8px;
}

.comments li p {
  margin: 0;
  color: #222;
}

.comments li small {
  color: #555;
}

.comment-form {
  margin-top: 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

.comment-form input {
  flex: 1;
  padding: 6px;
  border: 1px solid #aaa;
  border-radius: 4px;
  color: #222;
}

.comment-form button {
  padding: 6px 10px;
  cursor: pointer;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 4px;
}

.actions {
  margin-top: 10px;
  display: flex;
  gap: 10px;
}

.actions button {
  padding: 6px 10px;
  cursor: pointer;
  background-color: #0077cc;
  color: white;
  border: none;
  border-radius: 4px;
}

/* Hover simple pour boutons */
.actions button:hover,
.comment-form button:hover,
.form-post button:hover {
  opacity: 0.85;
}
</style>
