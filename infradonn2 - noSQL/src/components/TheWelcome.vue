<script setup lang="ts">
import { onMounted, onUnmounted, ref, computed, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// On ajoute le plugin find pour pouvoir faire des requetes plus complexes
// genre rechercher par type, trier, etc. Sans ca on peut juste faire get() par ID
PouchDB.plugin(PouchDBFind)

// ============================================================================
// TYPES
// ============================================================================

// Un commentaire dans la base de donnees
interface CommentDocument {
  _id: string // ID unique genere par nous
  _rev?: string // Revision geree par CouchDB (pour les conflits)
  _conflicts?: string[] // Liste des revisions en conflit
  type: 'comment' // Pour differencier des posts
  postId: string // Reference vers le post parent (comme une foreign key en SQL)
  author: string
  content: string
  date: string // Date ISO, plus facile a trier qu'un timestamp
}

// Un post dans la base de donnees
interface PostDocument {
  _id: string
  _rev?: string
  _conflicts?: string[]
  _attachments?: Record<string, { content_type: string }> // Pour les images/PDF
  type: 'post'
  title: string
  author: string
  content: string
  likes: number
  date: string
}

// Version enrichie du post pour l'affichage
// On rajoute les commentaires et autres infos utiles pour le rendu
interface DisplayPost extends PostDocument {
  lastComment: CommentDocument | null // Dernier commentaire pour l'apercu
  allComments: CommentDocument[] // Tous les commentaires charges
  commentsCount: number // Nombre total (pour afficher "voir les X commentaires")
  showAllComments: boolean // Est-ce qu'on affiche tous les commentaires?
  commentsLoaded: boolean // Est-ce que les commentaires ont ete charges?
  attachmentUrl?: string // URL blob pour afficher l'image
  hasConflict: boolean // Y a-t-il un conflit a resoudre?
}

// Structure pour gerer les conflits
// Quand 2 personnes modifient le meme doc en meme temps, on a un conflit
interface ConflictInfo {
  type: 'post' | 'comment'
  documentId: string
  localVersion: PostDocument | CommentDocument // Notre version
  remoteVersion: PostDocument | CommentDocument // Version du serveur
  conflictRev: string // Revision en conflit
}

// ============================================================================
// CONFIGURATION
// Toutes les constantes sont ici, facile a modifier si besoin
// ============================================================================

// URL de CouchDB - en prod faudrait mettre ca dans des variables d'environnement
const REMOTE_DB_URL = 'http://admin:admin@localhost:5984/lucaleone_infradonn2'
const LOCAL_DB_NAME = 'posts_local' // Nom de la base IndexedDB locale

const POSTS_PER_PAGE = 10 // Pagination: on charge 10 posts a la fois
const MAX_RETRY_ATTEMPTS = 3 // Nombre de tentatives si erreur 409 (conflit)

// Donnees pour generer des posts de test
const AUTHORS = ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve', 'Frank']
const TITLES = [
  'Introduction a Vue.js',
  'Guide PouchDB',
  'NoSQL vs SQL',
  'Sync offline',
  'Performance web',
  'Architecture moderne',
]

// ============================================================================
// STATE (variables reactives)
// ref() permet a Vue de detecter les changements et mettre a jour l'interface
// ============================================================================

// Connexions aux bases de donnees
const localDB = ref<PouchDB.Database | null>(null) // Base locale (IndexedDB)
const remoteDB = ref<PouchDB.Database | null>(null) // Base distante (CouchDB)
const syncHandler = ref<PouchDB.Replication.Sync<object> | null>(null) // Gestionnaire de sync
const changesHandler = ref<PouchDB.Core.Changes<object> | null>(null) // Ecouteur de changements

// Donnees des posts
const posts = ref<DisplayPost[]>([]) // Posts affiches a l'ecran (mode normal)
const allPostsSorted = ref<PostDocument[]>([]) // Cache de tous les posts tries
const currentPage = ref(0) // Page actuelle pour la pagination
const hasMorePosts = ref(true) // Y a-t-il encore des posts a charger?
const totalPostsCount = ref(0) // Nombre total de posts

// Pour la recherche: on garde une liste separee des resultats
// Comme ca on peut chercher dans TOUTE la base, pas juste les posts charges
const searchResults = ref<DisplayPost[]>([])
const isSearching = ref(false) // Indicateur de recherche en cours
const searchResultsCount = ref(0) // Nombre de resultats trouves

// Etats de l'interface
const isLoading = ref(false) // Afficher le spinner?
const error = ref<string | null>(null) // Message d'erreur a afficher

// Etat de la connexion
const isOnline = ref(true) // Mode en ligne ou hors ligne?
const syncMessage = ref('Initialisation...') // Message de statut de sync
const lastSync = ref<Date | null>(null) // Derniere synchronisation

// Recherche et tri
const searchQuery = ref('') // Texte de recherche
const sortType = ref<'likes' | 'date'>('likes') // Trier par likes ou date
const sortDirection = ref<'desc' | 'asc'>('desc') // Ordre croissant ou decroissant

// Formulaire de creation de post
const newPostForm = ref({ title: '', author: '', content: '' })
const selectedFile = ref<File | null>(null) // Fichier selectionne pour upload

// Gestion des conflits
const showConflictModal = ref(false) // Afficher la modale de conflit?
const currentConflict = ref<ConflictInfo | null>(null) // Conflit en cours
const pendingConflicts = ref<ConflictInfo[]>([]) // File d'attente des conflits

// URLs des pieces jointes (faut les liberer a la fin pour eviter les fuites memoire)
const objectUrls = ref<string[]>([])

// Timer pour le debounce de la recherche
// Ca evite de faire une requete a chaque lettre tapee, on attend que l'utilisateur
// ait fini de taper (300ms sans frappe)
let searchDebounceTimer: ReturnType<typeof setTimeout> | null = null

// ============================================================================
// COMPUTED (valeurs calculees)
// Ces valeurs sont recalculees automatiquement quand leurs dependances changent
// C'est plus propre que de les recalculer manuellement partout
// ============================================================================

// On affiche soit les resultats de recherche, soit les posts normaux
// Ca permet d'avoir une seule liste dans le template
const displayedPosts = computed(() => {
  if (searchQuery.value.trim()) {
    return searchResults.value
  }
  return posts.value
})

// Raccourcis pratiques
const hasConflicts = computed(() => pendingConflicts.value.length > 0)
const displayedCount = computed(() => displayedPosts.value.length)

// On peut charger plus seulement si on n'est pas en mode recherche
// Parce qu'en mode recherche on affiche tous les resultats d'un coup
const canLoadMore = computed(() => hasMorePosts.value && !searchQuery.value.trim())

// ============================================================================
// FONCTIONS UTILITAIRES
// Petites fonctions helper utilisees un peu partout
// ============================================================================

// Genere un ID unique avec prefix + timestamp + random
// Ex: "post_1699123456789_abc123"
const generateId = (prefix: string) =>
  `${prefix}_${Date.now()}_${Math.random().toString(36).slice(2, 9)}`

// Prend un element au hasard dans un tableau
const getRandomItem = (arr: string[]) => arr[Math.floor(Math.random() * arr.length)] || 'Anonyme'

// Affiche une erreur pendant 5 secondes puis la cache
const showError = (msg: string) => {
  error.value = msg
  setTimeout(() => {
    if (error.value === msg) error.value = null
  }, 5000)
}

// Libere une URL blob pour eviter les fuites memoire
// Faut toujours faire ca quand on utilise URL.createObjectURL
const revokeObjectUrl = (url: string) => {
  const i = objectUrls.value.indexOf(url)
  if (i > -1) {
    URL.revokeObjectURL(url)
    objectUrls.value.splice(i, 1)
  }
}

// ============================================================================
// INITIALISATION
// ============================================================================

// Cree les index pour accelerer les requetes
// Sans index, PouchDB doit scanner tous les documents a chaque requete
// C'est comme les index en SQL, ca accelere les SELECT mais ralentit les INSERT
const createIndexes = async () => {
  if (!localDB.value) return
  const indexes = [
    { fields: ['type'] }, // Pour filtrer par type (post ou comment)
    { fields: ['type', 'likes'] }, // Pour trier les posts par likes
    { fields: ['type', 'postId'] }, // Pour trouver les commentaires d'un post
    { fields: ['postId', 'date'] }, // Pour trier les commentaires par date
    { fields: ['type', 'postId', 'date'] }, // Index composite pour les commentaires
  ]

  for (const index of indexes) {
    try {
      await localDB.value.createIndex({ index })
    } catch (e) {
      // L'index existe deja, c'est pas grave on continue
    }
  }
}

// Fonction principale d'initialisation de la base
const initDatabase = async () => {
  try {
    // On cree les deux bases: locale (stockee dans IndexedDB du navigateur)
    // et distante (CouchDB sur le serveur)
    localDB.value = new PouchDB(LOCAL_DB_NAME)
    remoteDB.value = new PouchDB(REMOTE_DB_URL)

    await createIndexes()
    setupChangesListener() // Ecouter les changements en temps reel

    // Si on est en ligne, on lance la synchronisation
    if (isOnline.value) await startSync()

    // Chargement initial des donnees
    await loadPostsCache()
    await loadPage(0)
    await checkForConflicts()
  } catch (e) {
    console.error('Erreur initialisation:', e)
    showError("Erreur d'initialisation")
  }
}

// ============================================================================
// SYNCHRONISATION
// C'est le coeur du mode offline-first
// Les donnees sont d'abord ecrites en local puis synchronisees avec le serveur
// ============================================================================

// Demarre la synchronisation bidirectionnelle avec CouchDB
const startSync = async () => {
  if (!localDB.value || !remoteDB.value) return
  stopSync() // On arrete l'ancienne sync si elle existe

  try {
    // Sync initiale pour recuperer les donnees du serveur
    await localDB.value.sync(remoteDB.value)
    lastSync.value = new Date()
  } catch (e) {
    console.warn('Erreur sync initiale:', e)
  }

  // Sync continue en arriere-plan
  // live: true = reste connecte et sync en temps reel
  // retry: true = reessaie automatiquement si la connexion est perdue
  syncHandler.value = localDB.value
    .sync(remoteDB.value, { live: true, retry: true })
    .on('change', async () => {
      // Quand il y a des changements, on recharge les donnees
      lastSync.value = new Date()
      await loadPostsCache()
      // Si on est en mode recherche, on relance la recherche pour avoir les nouveaux resultats
      if (searchQuery.value.trim()) {
        await performSearch(searchQuery.value)
      } else {
        await loadPage(0, true)
      }
      await checkForConflicts()
    })
    .on('paused', () => {
      // Sync en pause (plus de changements a synchroniser)
      syncMessage.value = 'Synchronise'
      checkForConflicts()
    })
    .on('active', () => {
      // Sync en cours
      syncMessage.value = 'Synchronisation...'
    })
    .on('error', () => {
      // Erreur de connexion
      syncMessage.value = 'Erreur connexion'
    })
}

// Arrete la synchronisation
const stopSync = () => {
  if (syncHandler.value) {
    syncHandler.value.cancel()
    syncHandler.value = null
  }
}

// Bascule entre mode en ligne et hors ligne
// En mode hors ligne, on peut toujours utiliser l'app grace a la base locale
const toggleOnlineMode = async (online: boolean) => {
  isOnline.value = online
  if (online) {
    syncMessage.value = 'Connexion...'
    await startSync()
  } else {
    stopSync()
    syncMessage.value = 'Hors ligne'
  }
}

// ============================================================================
// ECOUTE DES CHANGEMENTS
// On ecoute les changements de la base locale pour reagir en temps reel
// ============================================================================

const setupChangesListener = () => {
  if (!localDB.value) return

  // On ecoute les changements depuis maintenant (since: 'now')
  // include_docs: true = on veut le contenu du document, pas juste l'ID
  // conflicts: true = on veut savoir s'il y a des conflits
  changesHandler.value = localDB.value
    .changes({ since: 'now', live: true, include_docs: true, conflicts: true })
    .on('change', async (change) => {
      const doc = change.doc as (PostDocument | CommentDocument) | undefined

      // Si le document a des conflits, on les gere
      if (doc?._conflicts?.length) await handleConflictDetected(doc)

      // On met a jour l'affichage selon le type de changement
      if (change.deleted || doc?.type === 'post') {
        // Si c'est un post ou une suppression, on recharge tout
        await loadPostsCache()
        if (searchQuery.value.trim()) {
          await performSearch(searchQuery.value)
        } else {
          await loadPage(0, true)
        }
      } else if (doc?.type === 'comment') {
        // Si c'est un commentaire, on met a jour juste le post concerne
        // Faut le faire dans les deux listes (posts normaux et resultats de recherche)
        const post = posts.value.find((p) => p._id === (doc as CommentDocument).postId)
        if (post) await updatePostComments(post)

        const searchPost = searchResults.value.find(
          (p) => p._id === (doc as CommentDocument).postId,
        )
        if (searchPost) await updatePostComments(searchPost)
      }
    })
    .on('error', (err) => console.error('Erreur changes:', err))
}

// ============================================================================
// RECHERCHE
// La recherche se fait directement dans la base de donnees,
// pas seulement sur les posts charges a l'ecran
// C'etait le bug principal: avant on cherchait que dans les 10 premiers posts
// ============================================================================

// Fonction de recherche qui interroge directement la base
const performSearch = async (query: string) => {
  if (!localDB.value || !query.trim()) {
    searchResults.value = []
    searchResultsCount.value = 0
    return
  }

  isSearching.value = true

  try {
    // On recupere TOUS les posts de la base pour la recherche
    // C'est pas optimal pour des grosses bases (genre des millions de posts)
    // mais pour un TP ca marche bien
    const result = await localDB.value.find({
      selector: { type: 'post' },
      limit: 10000,
    })

    const allPosts = result.docs as PostDocument[]
    const queryLower = query.toLowerCase()

    // Filtrage cote client sur titre et auteur
    // On pourrait faire ca cote serveur avec une vue CouchDB mais c'est plus complique
    const matchingPosts = allPosts.filter(
      (p) =>
        p.title.toLowerCase().includes(queryLower) || p.author.toLowerCase().includes(queryLower),
    )

    // Tri selon le critere choisi (meme logique que pour les posts normaux)
    if (sortType.value === 'likes') {
      matchingPosts.sort((a, b) => {
        const diff = b.likes - a.likes
        return sortDirection.value === 'desc' ? diff : -diff
      })
    } else {
      matchingPosts.sort((a, b) => {
        const diff = new Date(b.date).getTime() - new Date(a.date).getTime()
        return sortDirection.value === 'desc' ? diff : -diff
      })
    }

    searchResultsCount.value = matchingPosts.length

    // On convertit les posts en DisplayPost avec leurs commentaires
    // C'est un peu long mais necessaire pour avoir toutes les infos
    const displayPosts: DisplayPost[] = []

    for (const postDoc of matchingPosts) {
      const { lastComment, count, allComments } = await loadCommentsForPost(postDoc._id)
      const attachmentUrl = await loadAttachmentUrl(postDoc)

      let hasConflict = false
      try {
        const d = (await localDB.value!.get(postDoc._id, { conflicts: true })) as PostDocument
        hasConflict = (d._conflicts?.length ?? 0) > 0
      } catch {}

      displayPosts.push({
        ...postDoc,
        lastComment,
        allComments,
        commentsCount: count,
        showAllComments: false,
        commentsLoaded: true,
        attachmentUrl,
        hasConflict,
      })
    }

    // On nettoie les anciennes URLs des resultats precedents pour pas avoir de fuite memoire
    searchResults.value.forEach((p) => p.attachmentUrl && revokeObjectUrl(p.attachmentUrl))
    searchResults.value = displayPosts
  } catch (e) {
    console.error('Erreur recherche:', e)
    showError('Erreur de recherche')
  } finally {
    isSearching.value = false
  }
}

// Watcher sur la recherche avec debounce
// On attend 300ms apres la derniere frappe avant de lancer la recherche
// Ca evite de faire une requete a chaque lettre tapee (genre si tu tapes "hello"
// ca ferait 5 requetes sans debounce, la ca en fait qu'une)
watch(searchQuery, (newQuery) => {
  // On annule le timer precedent si l'utilisateur tape encore
  if (searchDebounceTimer) {
    clearTimeout(searchDebounceTimer)
  }

  if (!newQuery.trim()) {
    // Si la recherche est vide, on efface les resultats
    searchResults.value.forEach((p) => p.attachmentUrl && revokeObjectUrl(p.attachmentUrl))
    searchResults.value = []
    searchResultsCount.value = 0
    return
  }

  // Debounce de 300ms
  searchDebounceTimer = setTimeout(() => {
    performSearch(newQuery)
  }, 300)
})

// ============================================================================
// CHARGEMENT DES DONNEES
// On utilise un systeme de pagination pour pas tout charger d'un coup
// ============================================================================

// Charge tous les posts en memoire et les trie
// Le tri se fait cote client car PouchDB-find gere mal le tri descendant
const loadPostsCache = async () => {
  if (!localDB.value) return
  try {
    const result = await localDB.value.find({
      selector: { type: 'post' },
      limit: 10000, // On charge tout, c'est pas ouf mais ca marche pour un TP
    })

    let postDocs = result.docs as PostDocument[]

    // Tri cote client selon le critere choisi
    if (sortType.value === 'likes') {
      postDocs.sort((a, b) => {
        const diff = b.likes - a.likes
        return sortDirection.value === 'desc' ? diff : -diff
      })
    } else {
      postDocs.sort((a, b) => {
        const diff = new Date(b.date).getTime() - new Date(a.date).getTime()
        return sortDirection.value === 'desc' ? diff : -diff
      })
    }

    allPostsSorted.value = postDocs
    totalPostsCount.value = postDocs.length
    hasMorePosts.value = postDocs.length > POSTS_PER_PAGE
  } catch (e) {
    console.error('Erreur cache posts:', e)
  }
}

// Charge une page de posts (10 par defaut)
const loadPage = async (page: number, reset = false) => {
  if (!localDB.value) return
  isLoading.value = true

  try {
    // On calcule quels posts charger
    const start = page * POSTS_PER_PAGE
    const end = start + POSTS_PER_PAGE
    const pagePostDocs = allPostsSorted.value.slice(start, end)

    const displayPosts: DisplayPost[] = []

    // Pour chaque post, on charge ses commentaires et sa piece jointe
    for (const postDoc of pagePostDocs) {
      const { lastComment, count, allComments } = await loadCommentsForPost(postDoc._id)
      const attachmentUrl = await loadAttachmentUrl(postDoc)

      // On verifie s'il y a des conflits sur ce post
      let hasConflict = false
      try {
        const d = (await localDB.value!.get(postDoc._id, { conflicts: true })) as PostDocument
        hasConflict = (d._conflicts?.length ?? 0) > 0
      } catch {}

      displayPosts.push({
        ...postDoc,
        lastComment,
        allComments,
        commentsCount: count,
        showAllComments: false,
        commentsLoaded: true,
        attachmentUrl,
        hasConflict,
      })
    }

    if (reset) {
      // Si on reset, on nettoie les anciennes URLs pour liberer la memoire
      posts.value.forEach((p) => p.attachmentUrl && revokeObjectUrl(p.attachmentUrl))
      posts.value = displayPosts
      currentPage.value = 0
    } else {
      // Sinon on ajoute a la liste existante (infinite scroll)
      posts.value = [...posts.value, ...displayPosts]
    }

    currentPage.value = page
    hasMorePosts.value = end < allPostsSorted.value.length
  } catch (e) {
    console.error('Erreur chargement page:', e)
    showError('Erreur de chargement')
  } finally {
    isLoading.value = false
  }
}

// Charge les commentaires d'un post
// J'ai eu un bug avec le tri de PouchDB qui marchait pas, du coup je trie en JS
const loadCommentsForPost = async (
  postId: string,
): Promise<{
  lastComment: CommentDocument | null
  count: number
  allComments: CommentDocument[]
}> => {
  if (!localDB.value) return { lastComment: null, count: 0, allComments: [] }

  try {
    const result = await localDB.value.find({
      selector: {
        type: 'comment',
        postId: postId,
      },
      limit: 10000,
    })

    const comments = result.docs as CommentDocument[]

    if (comments.length === 0) {
      return { lastComment: null, count: 0, allComments: [] }
    }

    // Tri par date decroissante (plus recent en premier)
    // J'ai du faire ca en JS car PouchDB-find galere avec le tri
    comments.sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime())

    return {
      lastComment: comments[0] || null,
      count: comments.length,
      allComments: comments,
    }
  } catch (e) {
    console.error('Erreur chargement commentaires:', e)
    return { lastComment: null, count: 0, allComments: [] }
  }
}

// Met a jour les commentaires d'un post specifique
const updatePostComments = async (post: DisplayPost) => {
  const { lastComment, count, allComments } = await loadCommentsForPost(post._id)
  post.lastComment = lastComment
  post.commentsCount = count
  post.allComments = allComments
  post.commentsLoaded = true
}

// Charge l'URL de la piece jointe d'un post
// On cree un blob URL pour pouvoir l'afficher dans une balise img
const loadAttachmentUrl = async (doc: PostDocument): Promise<string | undefined> => {
  if (!localDB.value || !doc._attachments) return undefined
  const name = Object.keys(doc._attachments)[0]
  if (!name) return undefined

  try {
    const blob = (await localDB.value.getAttachment(doc._id, name)) as Blob
    const url = URL.createObjectURL(blob)
    objectUrls.value.push(url) // On garde une trace pour liberer plus tard
    return url
  } catch {
    return undefined
  }
}

// Charge les 10 posts suivants (pagination)
const loadMorePosts = async () => {
  if (!hasMorePosts.value || isLoading.value) return
  await loadPage(currentPage.value + 1)
}

// Change le tri et recharge les donnees
const changeSortAndReload = async () => {
  await loadPostsCache()
  // Si on est en mode recherche, on relance la recherche avec le nouveau tri
  if (searchQuery.value.trim()) {
    await performSearch(searchQuery.value)
  } else {
    await loadPage(0, true)
  }
}

// Rafraichit un seul post (apres une modification par exemple)
const refreshPost = async (postId: string) => {
  if (!localDB.value) return

  try {
    const doc = (await localDB.value.get(postId, { conflicts: true })) as PostDocument
    const { lastComment, count, allComments } = await loadCommentsForPost(postId)
    const attachmentUrl = await loadAttachmentUrl(doc)

    // Mettre a jour dans la liste principale
    const index = posts.value.findIndex((p) => p._id === postId)
    if (index >= 0) {
      const existing = posts.value[index]
      if (existing?.attachmentUrl) revokeObjectUrl(existing.attachmentUrl)

      posts.value[index] = {
        ...doc,
        lastComment,
        allComments,
        commentsCount: count,
        showAllComments: existing?.showAllComments || false,
        commentsLoaded: true,
        attachmentUrl,
        hasConflict: (doc._conflicts?.length ?? 0) > 0,
      }
    }

    // Mettre a jour aussi dans les resultats de recherche si le post y est
    const searchIndex = searchResults.value.findIndex((p) => p._id === postId)
    if (searchIndex >= 0) {
      const existing = searchResults.value[searchIndex]
      if (existing?.attachmentUrl) revokeObjectUrl(existing.attachmentUrl)

      searchResults.value[searchIndex] = {
        ...doc,
        lastComment,
        allComments,
        commentsCount: count,
        showAllComments: existing?.showAllComments || false,
        commentsLoaded: true,
        attachmentUrl,
        hasConflict: (doc._conflicts?.length ?? 0) > 0,
      }
    }
  } catch (e) {
    console.error('Erreur refresh:', e)
  }
}

// ============================================================================
// GESTION DES CONFLITS
// Les conflits arrivent quand 2 personnes modifient le meme doc en meme temps
// CouchDB garde les deux versions et c'est a nous de choisir laquelle garder
// ============================================================================

// Parcourt tous les documents pour trouver ceux qui ont des conflits
const checkForConflicts = async () => {
  if (!localDB.value) return

  try {
    const result = await localDB.value.find({ selector: { type: { $in: ['post', 'comment'] } } })
    for (const doc of result.docs) {
      try {
        const d = (await localDB.value.get(doc._id, { conflicts: true })) as
          | PostDocument
          | CommentDocument
        if (d._conflicts?.length) await handleConflictDetected(d)
      } catch {}
    }
  } catch (e) {
    console.error('Erreur check conflits:', e)
  }
}

// Gere la detection d'un nouveau conflit
const handleConflictDetected = async (doc: PostDocument | CommentDocument) => {
  if (!localDB.value || !doc._conflicts?.length) return
  const conflictRev = doc._conflicts[0]
  // On evite d'ajouter un conflit deja dans la file
  if (!conflictRev || pendingConflicts.value.some((c) => c.documentId === doc._id)) return

  try {
    // On recupere la version en conflit
    const remote = (await localDB.value.get(doc._id, { rev: conflictRev })) as typeof doc
    pendingConflicts.value.push({
      type: doc.type,
      documentId: doc._id,
      localVersion: doc,
      remoteVersion: remote,
      conflictRev,
    })
    // On affiche la modale si c'est le premier conflit
    if (!showConflictModal.value) showNextConflict()

    // On marque le post comme ayant un conflit (dans les deux listes)
    if (doc.type === 'post') {
      const p = posts.value.find((x) => x._id === doc._id)
      if (p) p.hasConflict = true
      const sp = searchResults.value.find((x) => x._id === doc._id)
      if (sp) sp.hasConflict = true
    }
  } catch (e) {
    console.error('Erreur detection conflit:', e)
  }
}

// Affiche le prochain conflit a resoudre
const showNextConflict = () => {
  const next = pendingConflicts.value[0]
  if (next) {
    currentConflict.value = next
    showConflictModal.value = true
  } else {
    showConflictModal.value = false
    currentConflict.value = null
  }
}

// Resout le conflit en gardant soit la version locale soit la version serveur
const resolveCurrentConflict = async (keepLocal: boolean) => {
  if (!localDB.value || !currentConflict.value) return
  const c = currentConflict.value

  try {
    if (keepLocal) {
      // On garde notre version, on supprime la version en conflit
      await localDB.value.remove({ _id: c.documentId, _rev: c.conflictRev })
    } else {
      // On garde la version serveur
      const local = (await localDB.value.get(c.documentId)) as PostDocument | CommentDocument
      if (!local._rev) throw new Error('No rev')

      // Petit truc malin: pour les posts, on garde le max des likes
      // Comme ca on perd pas de likes si quelqu'un a like pendant qu'on editait
      const merged =
        c.type === 'post'
          ? {
              ...c.remoteVersion,
              _id: c.documentId,
              _rev: local._rev,
              likes: Math.max(
                (c.localVersion as PostDocument).likes,
                (c.remoteVersion as PostDocument).likes,
              ),
            }
          : { ...c.remoteVersion, _id: c.documentId, _rev: local._rev }

      await localDB.value.put(merged as PouchDB.Core.PutDocument<typeof merged>)
      await localDB.value.remove({ _id: c.documentId, _rev: c.conflictRev })
    }

    // On retire le conflit de la file
    pendingConflicts.value.shift()

    // On met a jour l'affichage
    if (c.type === 'post') {
      await loadPostsCache()
      await refreshPost(c.documentId)
    } else {
      const comment = c.localVersion as CommentDocument
      const post = posts.value.find((p) => p._id === comment.postId)
      if (post) await updatePostComments(post)
      const searchPost = searchResults.value.find((p) => p._id === comment.postId)
      if (searchPost) await updatePostComments(searchPost)
    }

    // On passe au conflit suivant
    showNextConflict()
  } catch (e) {
    console.error('Erreur resolution:', e)
    showError('Erreur resolution conflit')
  }
}

// ============================================================================
// CRUD POSTS
// Les operations de base: Create, Read, Update, Delete
// ============================================================================

// Cree un nouveau post
const createPost = async () => {
  if (!localDB.value) return
  if (!newPostForm.value.title.trim() || !newPostForm.value.author.trim()) {
    showError('Titre et auteur requis')
    return
  }

  try {
    const postDoc: PostDocument = {
      _id: generateId('post'),
      type: 'post',
      title: newPostForm.value.title.trim(),
      author: newPostForm.value.author.trim(),
      content: newPostForm.value.content.trim(),
      likes: 0,
      date: new Date().toISOString(),
    }

    const res = await localDB.value.put(postDoc as PouchDB.Core.PutDocument<PostDocument>)

    // Si on a selectionne un fichier, on l'ajoute comme piece jointe
    if (selectedFile.value) {
      await localDB.value.putAttachment(
        postDoc._id,
        selectedFile.value.name,
        res.rev,
        selectedFile.value,
        selectedFile.value.type,
      )
    }

    await loadPostsCache()

    // Si on est en mode recherche et que le nouveau post correspond, on relance la recherche
    if (searchQuery.value.trim()) {
      await performSearch(searchQuery.value)
    } else {
      await loadPage(0, true)
    }

    // On reset le formulaire
    newPostForm.value = { title: '', author: '', content: '' }
    selectedFile.value = null
  } catch (e) {
    console.error('Erreur creation:', e)
    showError('Erreur creation')
  }
}

// Modifie le contenu d'un post
const updatePost = async (post: DisplayPost, newContent: string) => {
  if (!localDB.value) return
  try {
    // On recupere la derniere version pour avoir le bon _rev
    const doc = (await localDB.value.get(post._id)) as PostDocument
    await localDB.value.put({
      ...doc,
      content: newContent,
      date: new Date().toISOString(),
    } as PouchDB.Core.PutDocument<PostDocument>)
    await refreshPost(post._id)
  } catch (e) {
    console.error('Erreur update:', e)
    showError('Erreur modification')
  }
}

// Supprime un post et tous ses commentaires
const deletePost = async (post: DisplayPost) => {
  if (!localDB.value) return
  try {
    // D'abord on supprime tous les commentaires du post
    // En NoSQL y a pas de CASCADE DELETE comme en SQL, faut le faire a la main
    const comments = await localDB.value.find({ selector: { type: 'comment', postId: post._id } })
    for (const c of comments.docs) {
      try {
        const d = await localDB.value.get(c._id)
        if (d._rev) await localDB.value.remove({ _id: d._id, _rev: d._rev })
      } catch {}
    }

    // Ensuite on supprime le post
    const doc = await localDB.value.get(post._id)
    if (doc._rev) await localDB.value.remove({ _id: doc._id, _rev: doc._rev })

    // On libere l'URL de la piece jointe
    if (post.attachmentUrl) revokeObjectUrl(post.attachmentUrl)

    // Supprimer des deux listes
    posts.value = posts.value.filter((p) => p._id !== post._id)
    searchResults.value = searchResults.value.filter((p) => p._id !== post._id)

    await loadPostsCache()
  } catch (e) {
    console.error('Erreur suppression:', e)
    showError('Erreur suppression')
  }
}

// Like un post avec gestion des conflits
// Si 2 personnes likent en meme temps, on peut avoir une erreur 409
const likePost = async (post: DisplayPost) => {
  if (!localDB.value) return

  // On essaie plusieurs fois en cas de conflit
  for (let i = 0; i < MAX_RETRY_ATTEMPTS; i++) {
    try {
      const doc = (await localDB.value.get(post._id)) as PostDocument
      doc.likes++
      await localDB.value.put(doc as PouchDB.Core.PutDocument<PostDocument>)

      // Mise a jour locale immediate pour un feedback rapide (dans les deux listes)
      const p = posts.value.find((x) => x._id === post._id)
      if (p) p.likes = doc.likes

      const sp = searchResults.value.find((x) => x._id === post._id)
      if (sp) sp.likes = doc.likes

      const cached = allPostsSorted.value.find((x) => x._id === post._id)
      if (cached) cached.likes = doc.likes

      return
    } catch (e: unknown) {
      // Erreur 409 = conflit, on reessaie avec un delai exponentiel
      if ((e as { status?: number }).status === 409 && i < MAX_RETRY_ATTEMPTS - 1) {
        await new Promise((r) => setTimeout(r, 100 * Math.pow(2, i)))
      } else {
        showError('Erreur like')
        return
      }
    }
  }
}

// ============================================================================
// CRUD COMMENTAIRES
// ============================================================================

// Ajoute un commentaire a un post
const addComment = async (post: DisplayPost, author: string, content: string) => {
  if (!localDB.value) return
  try {
    const commentDoc: CommentDocument = {
      _id: generateId('comment'),
      type: 'comment',
      postId: post._id, // Reference vers le post parent
      author: author.trim(),
      content: content.trim(),
      date: new Date().toISOString(),
    }

    await localDB.value.put(commentDoc as PouchDB.Core.PutDocument<CommentDocument>)
    await updatePostComments(post)

    // Si le post est aussi dans les resultats de recherche, le mettre a jour la aussi
    const searchPost = searchResults.value.find((p) => p._id === post._id)
    if (searchPost && searchPost !== post) {
      await updatePostComments(searchPost)
    }
  } catch (e) {
    console.error('Erreur commentaire:', e)
    showError('Erreur commentaire')
  }
}

// Modifie un commentaire
const updateComment = async (comment: CommentDocument, newContent: string) => {
  if (!localDB.value) return
  try {
    const doc = (await localDB.value.get(comment._id)) as CommentDocument
    await localDB.value.put({
      ...doc,
      content: newContent,
      date: new Date().toISOString(),
    } as PouchDB.Core.PutDocument<CommentDocument>)

    // Mettre a jour dans les deux listes
    const post = posts.value.find((p) => p._id === comment.postId)
    if (post) await updatePostComments(post)

    const searchPost = searchResults.value.find((p) => p._id === comment.postId)
    if (searchPost) await updatePostComments(searchPost)
  } catch (e) {
    console.error('Erreur update comment:', e)
    showError('Erreur modification')
  }
}

// Supprime un commentaire
const deleteComment = async (comment: CommentDocument) => {
  if (!localDB.value) return
  try {
    const doc = await localDB.value.get(comment._id)
    if (doc._rev) await localDB.value.remove({ _id: doc._id, _rev: doc._rev })

    // Mettre a jour dans les deux listes
    const post = posts.value.find((p) => p._id === comment.postId)
    if (post) await updatePostComments(post)

    const searchPost = searchResults.value.find((p) => p._id === comment.postId)
    if (searchPost) await updatePostComments(searchPost)
  } catch (e) {
    console.error('Erreur delete comment:', e)
    showError('Erreur suppression')
  }
}

// ============================================================================
// PIECES JOINTES (images, PDF)
// CouchDB permet d'attacher des fichiers binaires aux documents
// ============================================================================

// Ajoute une piece jointe a un post
const addAttachment = async (post: DisplayPost, file: File) => {
  if (!localDB.value) return
  try {
    const doc = await localDB.value.get(post._id)
    if (doc._rev) await localDB.value.putAttachment(post._id, file.name, doc._rev, file, file.type)
    await refreshPost(post._id)
  } catch (e) {
    console.error('Erreur attachment:', e)
    showError('Erreur media')
  }
}

// Supprime la piece jointe d'un post
const removeAttachment = async (post: DisplayPost) => {
  if (!localDB.value || !post._attachments) return
  const name = Object.keys(post._attachments)[0]
  if (!name) return
  try {
    const doc = await localDB.value.get(post._id)
    if (doc._rev) await localDB.value.removeAttachment(post._id, name, doc._rev)
    if (post.attachmentUrl) revokeObjectUrl(post.attachmentUrl)
    await refreshPost(post._id)
  } catch (e) {
    console.error('Erreur remove attachment:', e)
    showError('Erreur suppression media')
  }
}

// ============================================================================
// GENERATION DE DONNEES DE TEST
// Super pratique pour tester l'application rapidement
// ============================================================================

const generateTestData = async (count = 10) => {
  if (!localDB.value) return
  isLoading.value = true

  try {
    const docs: (PostDocument | CommentDocument)[] = []
    const ts = Date.now()

    for (let i = 0; i < count; i++) {
      const postId = `post_${ts}_${i}`

      // On cree le post
      docs.push({
        _id: postId,
        type: 'post',
        title: `${getRandomItem(TITLES)} #${i + 1}`,
        author: getRandomItem(AUTHORS),
        content: `Contenu du post ${i + 1}. Lorem ipsum dolor sit amet.`,
        likes: Math.floor(Math.random() * 100),
        date: new Date(ts + i * 60000).toISOString(),
      })

      // On ajoute quelques commentaires aleatoires (entre 1 et 5)
      const numComments = Math.floor(Math.random() * 5) + 1
      for (let j = 0; j < numComments; j++) {
        docs.push({
          _id: `comment_${ts}_${i}_${j}`,
          type: 'comment',
          postId,
          author: getRandomItem(AUTHORS),
          content: `Commentaire ${j + 1} - Super post ! Tres interessant.`,
          date: new Date(ts + i * 60000 + (j + 1) * 1000).toISOString(),
        })
      }
    }

    // bulkDocs permet d'inserer plusieurs documents en une seule operation
    // C'est beaucoup plus rapide que de faire des put() un par un
    await localDB.value.bulkDocs(docs as PouchDB.Core.PutDocument<PostDocument | CommentDocument>[])

    await loadPostsCache()

    // Si on est en mode recherche, on relance la recherche
    if (searchQuery.value.trim()) {
      await performSearch(searchQuery.value)
    } else {
      await loadPage(0, true)
    }
  } catch (e) {
    console.error('Erreur generation:', e)
    showError('Erreur generation')
  } finally {
    isLoading.value = false
  }
}

// ============================================================================
// HANDLERS UI
// Fonctions appelees par les evenements du template
// ============================================================================

// Selection d'un fichier pour upload
const handleFileSelect = (e: Event) => {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) selectedFile.value = file
}

// Modification d'un post via prompt (pas ideal mais simple pour un TP)
const handleUpdatePost = (post: DisplayPost) => {
  const content = prompt('Modifier:', post.content)
  if (content !== null && content !== post.content) updatePost(post, content)
}

// Suppression d'un post avec confirmation
const handleDeletePost = (post: DisplayPost) => {
  if (confirm('Supprimer ce post ?')) deletePost(post)
}

// Ajout d'un commentaire via prompt
const handleAddComment = (post: DisplayPost) => {
  const author = prompt('Votre nom:')
  if (!author?.trim()) return
  const content = prompt('Commentaire:')
  if (content?.trim()) addComment(post, author, content)
}

// Modification d'un commentaire
const handleUpdateComment = (comment: CommentDocument) => {
  const content = prompt('Modifier:', comment.content)
  if (content !== null && content !== comment.content) updateComment(comment, content)
}

// Suppression d'un commentaire
const handleDeleteComment = (comment: CommentDocument) => {
  if (confirm('Supprimer ?')) deleteComment(comment)
}

// Ajout d'une piece jointe via input file
const handleAddAttachment = (post: DisplayPost) => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'image/*,application/pdf'
  input.onchange = (e) => {
    const file = (e.target as HTMLInputElement).files?.[0]
    if (file) addAttachment(post, file)
  }
  input.click()
}

// Suppression d'une piece jointe
const handleRemoveAttachment = (post: DisplayPost) => {
  if (confirm('Supprimer ce media ?')) removeAttachment(post)
}

// Toggle pour afficher/masquer tous les commentaires d'un post
const toggleAllComments = (post: DisplayPost) => {
  post.showAllComments = !post.showAllComments
}

// Quand on change le tri, on recharge les donnees
const onSortChange = async () => {
  await changeSortAndReload()
}

// ============================================================================
// LIFECYCLE HOOKS
// onMounted = quand le composant est monte dans le DOM
// onUnmounted = quand le composant est demonte (pour nettoyer)
// ============================================================================

onMounted(() => {
  initDatabase()
})

onUnmounted(() => {
  // Nettoyage important pour eviter les fuites memoire
  stopSync()
  if (changesHandler.value) changesHandler.value.cancel()
  if (searchDebounceTimer) clearTimeout(searchDebounceTimer)
  objectUrls.value.forEach((u) => URL.revokeObjectURL(u))
})
</script>

<template>
  <div class="container">
    <!-- HEADER -->
    <header class="header">
      <h1>Blog NoSQL - TP InfraDonn2</h1>
    </header>

    <!-- BARRE DE STATUT -->
    <!-- Affiche l'etat de la connexion et permet de basculer en mode offline -->
    <div class="status-bar">
      <label class="toggle-switch">
        <input type="checkbox" :checked="isOnline" @change="toggleOnlineMode(!isOnline)" />
        <span>{{ isOnline ? 'En ligne' : 'Hors ligne' }}</span>
      </label>
      <span :class="['status-text', isOnline ? 'online' : 'offline']">{{ syncMessage }}</span>
      <span v-if="lastSync" class="last-sync">Sync: {{ lastSync.toLocaleTimeString() }}</span>
      <span v-if="hasConflicts" class="conflict-badge" @click="showNextConflict">
        {{ pendingConflicts.length }} conflit(s)
      </span>
    </div>

    <!-- MESSAGE D'ERREUR -->
    <!-- Cliquable pour le fermer -->
    <div v-if="error" class="error-banner" @click="error = null">{{ error }}</div>

    <!-- CONTROLES -->
    <!-- Bouton pour generer des donnees de test, barre de recherche et options de tri -->
    <div class="controls">
      <button @click="generateTestData(10)" :disabled="isLoading">Generer 10 posts</button>
      <input v-model="searchQuery" placeholder="Rechercher..." class="search-input" />
      <select v-model="sortType" @change="onSortChange">
        <option value="likes">Top Likes</option>
        <option value="date">Date</option>
      </select>
      <select v-model="sortDirection" @change="onSortChange">
        <option value="desc">Desc</option>
        <option value="asc">Asc</option>
      </select>
    </div>

    <!-- FORMULAIRE NOUVEAU POST -->
    <div class="card">
      <h2>Nouveau post</h2>
      <input v-model="newPostForm.title" placeholder="Titre" />
      <input v-model="newPostForm.author" placeholder="Auteur" />
      <textarea v-model="newPostForm.content" placeholder="Contenu..." rows="2"></textarea>
      <div class="file-row">
        <label class="file-label">
          Media
          <input type="file" accept="image/*,application/pdf" @change="handleFileSelect" />
        </label>
        <span v-if="selectedFile">{{ selectedFile.name }}</span>
      </div>
      <button @click="createPost" :disabled="isLoading">Publier</button>
    </div>

    <!-- LOADING -->
    <div v-if="isLoading || isSearching" class="loading">
      {{ isSearching ? 'Recherche...' : 'Chargement...' }}
    </div>

    <!-- INFOS SUR LES POSTS AFFICHES -->
    <!-- On affiche un message different selon qu'on est en mode recherche ou pas -->
    <div class="posts-info">
      <template v-if="searchQuery.trim()">
        <span>{{ searchResultsCount }} resultat(s) pour "{{ searchQuery }}"</span>
      </template>
      <template v-else>
        <span>{{ displayedCount }} post(s) affiches sur {{ totalPostsCount }} total</span>
        <span v-if="sortType === 'likes'">(Tries par likes)</span>
        <span v-else>(Tries par date)</span>
      </template>
    </div>

    <!-- ETAT VIDE -->
    <div v-if="!isLoading && !isSearching && !displayedPosts.length" class="empty">
      <template v-if="searchQuery.trim()">
        <p>Aucun resultat pour "{{ searchQuery }}"</p>
        <button @click="searchQuery = ''">Effacer la recherche</button>
      </template>
      <template v-else>
        <p>Aucun post</p>
        <button @click="generateTestData(5)">Creer des posts</button>
      </template>
    </div>

    <!-- LISTE DES POSTS -->
    <!-- On utilise displayedPosts qui contient soit les posts normaux soit les resultats de recherche -->
    <article
      v-for="post in displayedPosts"
      :key="post._id"
      :class="['card', { conflict: post.hasConflict }]"
    >
      <!-- AVERTISSEMENT CONFLIT -->
      <div v-if="post.hasConflict" class="conflict-warning">
        Conflit non resolu
        <button @click="showNextConflict">Resoudre</button>
      </div>

      <!-- CONTENU DU POST -->
      <h3>{{ post.title }}</h3>
      <div class="meta">
        <span>{{ post.author }}</span>
        <span>{{ new Date(post.date).toLocaleString() }}</span>
        <span class="likes">{{ post.likes }} likes</span>
      </div>
      <p>{{ post.content }}</p>

      <!-- PIECE JOINTE -->
      <div v-if="post.attachmentUrl" class="attachment">
        <img :src="post.attachmentUrl" alt="" />
        <button @click="handleRemoveAttachment(post)">Supprimer media</button>
      </div>

      <!-- BOUTONS D'ACTION -->
      <div class="actions">
        <button @click="likePost(post)">Like</button>
        <button @click="handleUpdatePost(post)">Modifier</button>
        <button @click="handleDeletePost(post)">Supprimer</button>
        <button @click="handleAddComment(post)">Commenter</button>
        <button @click="handleAddAttachment(post)">Media</button>
      </div>

      <!-- SECTION COMMENTAIRES -->
      <div class="comments-section">
        <h4 class="comments-title">Commentaires ({{ post.commentsCount }})</h4>

        <!-- Cas: aucun commentaire -->
        <p v-if="post.commentsCount === 0" class="no-comments">Aucun commentaire</p>

        <!-- Cas: 1 seul commentaire -->
        <template v-else-if="post.commentsCount === 1 && post.lastComment">
          <div class="comment-item">
            <div class="comment-header">
              <strong>{{ post.lastComment.author }}</strong>
              <small>{{ new Date(post.lastComment.date).toLocaleString() }}</small>
            </div>
            <p class="comment-content">{{ post.lastComment.content }}</p>
            <div class="comment-actions">
              <button @click="handleUpdateComment(post.lastComment)">Modifier</button>
              <button @click="handleDeleteComment(post.lastComment)">Supprimer</button>
            </div>
          </div>
        </template>

        <!-- Cas: plusieurs commentaires -->
        <template v-else-if="post.commentsCount > 1">
          <!-- On affiche que le dernier commentaire par defaut -->
          <div v-if="post.lastComment && !post.showAllComments" class="comment-item last">
            <div class="comment-badge">Dernier commentaire</div>
            <div class="comment-header">
              <strong>{{ post.lastComment.author }}</strong>
              <small>{{ new Date(post.lastComment.date).toLocaleString() }}</small>
            </div>
            <p class="comment-content">{{ post.lastComment.content }}</p>
            <div class="comment-actions">
              <button @click="handleUpdateComment(post.lastComment)">Modifier</button>
              <button @click="handleDeleteComment(post.lastComment)">Supprimer</button>
            </div>
          </div>

          <!-- Bouton pour voir/masquer tous les commentaires -->
          <div class="show-all-comments">
            <button class="toggle-btn" @click="toggleAllComments(post)">
              {{
                post.showAllComments
                  ? 'Masquer les commentaires'
                  : `Voir tous les ${post.commentsCount} commentaires`
              }}
            </button>
          </div>

          <!-- Tous les commentaires -->
          <div v-if="post.showAllComments" class="all-comments">
            <div v-for="comment in post.allComments" :key="comment._id" class="comment-item">
              <div class="comment-header">
                <strong>{{ comment.author }}</strong>
                <small>{{ new Date(comment.date).toLocaleString() }}</small>
              </div>
              <p class="comment-content">{{ comment.content }}</p>
              <div class="comment-actions">
                <button @click="handleUpdateComment(comment)">Modifier</button>
                <button @click="handleDeleteComment(comment)">Supprimer</button>
              </div>
            </div>
          </div>
        </template>
      </div>
    </article>

    <!-- BOUTON CHARGER PLUS (seulement si pas en mode recherche) -->
    <!-- En mode recherche on affiche tous les resultats d'un coup -->
    <div v-if="canLoadMore" class="load-more">
      <button @click="loadMorePosts" :disabled="isLoading">Charger les 10 posts suivants</button>
      <p class="load-more-info">{{ posts.length }} / {{ totalPostsCount }} posts charges</p>
    </div>

    <!-- MODALE DE RESOLUTION DE CONFLIT -->
    <div v-if="showConflictModal && currentConflict" class="modal-overlay">
      <div class="modal">
        <h2>Conflit detecte</h2>
        <p>Choisissez la version a conserver:</p>

        <div class="versions">
          <!-- Version 1 -->
          <div class="version local">
            <h4>Version 1</h4>
            <template v-if="currentConflict.type === 'post'">
              <p>
                <strong>Titre:</strong> {{ (currentConflict.localVersion as PostDocument).title }}
              </p>
              <p>
                <strong>Contenu:</strong>
                {{ (currentConflict.localVersion as PostDocument).content }}
              </p>
              <p>
                <strong>Likes:</strong> {{ (currentConflict.localVersion as PostDocument).likes }}
              </p>
            </template>
            <template v-else>
              <p>
                <strong>Contenu:</strong>
                {{ (currentConflict.localVersion as CommentDocument).content }}
              </p>
            </template>
            <p>
              <strong>Date:</strong>
              {{ new Date(currentConflict.localVersion.date).toLocaleString() }}
            </p>
            <button @click="resolveCurrentConflict(true)">Garder cette version</button>
          </div>

          <!-- Version 2 -->
          <div class="version remote">
            <h4>Version 2</h4>
            <template v-if="currentConflict.type === 'post'">
              <p>
                <strong>Titre:</strong> {{ (currentConflict.remoteVersion as PostDocument).title }}
              </p>
              <p>
                <strong>Contenu:</strong>
                {{ (currentConflict.remoteVersion as PostDocument).content }}
              </p>
              <p>
                <strong>Likes:</strong> {{ (currentConflict.remoteVersion as PostDocument).likes }}
              </p>
            </template>
            <template v-else>
              <p>
                <strong>Contenu:</strong>
                {{ (currentConflict.remoteVersion as CommentDocument).content }}
              </p>
            </template>
            <p>
              <strong>Date:</strong>
              {{ new Date(currentConflict.remoteVersion.date).toLocaleString() }}
            </p>
            <button @click="resolveCurrentConflict(false)">Garder cette version</button>
          </div>
        </div>

        <p v-if="pendingConflicts.length > 1" class="remaining">
          {{ pendingConflicts.length - 1 }} autre(s) conflit(s)
        </p>
      </div>
    </div>

    <!-- FOOTER -->
    <footer class="footer">TP Infrastructure de Donnees - PouchDB + CouchDB + Vue.js</footer>
  </div>
</template>

<style>
/* Variables CSS pour le theme sombre
   J'ai utilise des variables pour pouvoir changer facilement les couleurs */
:root {
  --bg: #0f172a;
  --bg2: #1e293b;
  --bg3: #334155;
  --text: #f8fafc;
  --text2: #94a3b8;
  --muted: #64748b;
  --accent: #3b82f6;
  --success: #22c55e;
  --warning: #f59e0b;
  --danger: #ef4444;
  --border: #475569;
  --radius: 8px;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  background: var(--bg);
  color: var(--text);
  font-family: system-ui, sans-serif;
  line-height: 1.5;
}

.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 1.5rem;
}

.header {
  text-align: center;
  margin-bottom: 1.5rem;
}

.header h1 {
  font-size: 1.75rem;
  color: var(--accent);
}

.status-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  align-items: center;
  padding: 0.75rem;
  background: var(--bg2);
  border-radius: var(--radius);
  margin-bottom: 1rem;
}

.toggle-switch {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
}

.toggle-switch input {
  width: 36px;
  height: 18px;
  accent-color: var(--accent);
}

.status-text {
  font-weight: 600;
}

.status-text.online {
  color: var(--success);
}

.status-text.offline {
  color: var(--warning);
}

.last-sync {
  color: var(--muted);
  font-size: 0.8rem;
}

.conflict-badge {
  background: var(--danger);
  color: white;
  padding: 0.2rem 0.6rem;
  border-radius: 12px;
  font-size: 0.8rem;
  cursor: pointer;
  margin-left: auto;
}

.error-banner {
  background: rgba(239, 68, 68, 0.2);
  border: 1px solid var(--danger);
  color: #fca5a5;
  padding: 0.75rem;
  border-radius: var(--radius);
  margin-bottom: 1rem;
  cursor: pointer;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
  align-items: center;
}

.search-input {
  flex: 1;
  min-width: 150px;
  padding: 0.5rem;
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: var(--text);
}

select {
  padding: 0.5rem;
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: var(--text);
}

.card {
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1rem;
  margin-bottom: 1rem;
}

.card.conflict {
  border-color: var(--danger);
}

.card h2 {
  font-size: 1.1rem;
  margin-bottom: 0.75rem;
}

.card h3 {
  font-size: 1.15rem;
  margin-bottom: 0.5rem;
}

.card input,
.card textarea {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: var(--text);
  font-family: inherit;
}

.card textarea {
  resize: vertical;
  min-height: 60px;
}

.file-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.5rem;
}

.file-label {
  padding: 0.4rem 0.8rem;
  background: var(--bg3);
  border-radius: var(--radius);
  cursor: pointer;
  font-size: 0.85rem;
}

.file-label input {
  display: none;
}

.file-row span {
  color: var(--text2);
  font-size: 0.8rem;
}

button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: var(--radius);
  font-weight: 600;
  cursor: pointer;
  background: var(--accent);
  color: white;
  font-size: 0.85rem;
}

button:hover:not(:disabled) {
  opacity: 0.9;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.posts-info {
  display: flex;
  gap: 0.5rem;
  color: var(--muted);
  font-size: 0.85rem;
  margin-bottom: 1rem;
  padding: 0.5rem;
  background: var(--bg2);
  border-radius: var(--radius);
}

.meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  color: var(--text2);
  font-size: 0.8rem;
  margin-bottom: 0.5rem;
}

.likes {
  color: var(--danger);
  font-weight: 600;
}

.attachment {
  margin: 0.75rem 0;
}

.attachment img {
  max-width: 100%;
  max-height: 200px;
  border-radius: var(--radius);
  display: block;
  margin-bottom: 0.5rem;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  padding-top: 0.75rem;
  border-top: 1px solid var(--border);
  margin-top: 0.75rem;
}

.actions button {
  background: var(--bg3);
  padding: 0.4rem 0.7rem;
  font-size: 0.8rem;
}

.conflict-warning {
  background: rgba(239, 68, 68, 0.2);
  border: 1px solid var(--danger);
  padding: 0.5rem;
  border-radius: var(--radius);
  margin-bottom: 0.75rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #fca5a5;
  font-size: 0.85rem;
}

.conflict-warning button {
  background: var(--warning);
  color: black;
  padding: 0.25rem 0.5rem;
  font-size: 0.75rem;
}

/* Styles pour la section commentaires */
.comments-section {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid var(--border);
}

.comments-title {
  font-size: 0.95rem;
  color: var(--text);
  margin-bottom: 0.75rem;
}

.comment-item {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 0.75rem;
  margin-bottom: 0.5rem;
}

.comment-item.last {
  border-left: 3px solid var(--accent);
}

.comment-badge {
  background: var(--accent);
  color: white;
  font-size: 0.7rem;
  padding: 0.15rem 0.4rem;
  border-radius: 4px;
  display: inline-block;
  margin-bottom: 0.5rem;
}

.comment-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}

.comment-header strong {
  color: var(--text);
  font-size: 0.9rem;
}

.comment-header small {
  color: var(--muted);
  font-size: 0.75rem;
}

.comment-content {
  color: var(--text2);
  font-size: 0.85rem;
  margin: 0.25rem 0 0.5rem 0;
}

.comment-actions {
  display: flex;
  gap: 0.25rem;
}

.comment-actions button {
  background: var(--bg3);
  padding: 0.2rem 0.5rem;
  font-size: 0.7rem;
}

.show-all-comments {
  margin: 0.5rem 0;
}

.toggle-btn {
  background: transparent;
  color: var(--accent);
  padding: 0.4rem 0;
  font-size: 0.85rem;
  text-decoration: underline;
}

.toggle-btn:hover {
  opacity: 0.8;
}

.all-comments {
  margin-top: 0.5rem;
}

.no-comments {
  color: var(--muted);
  font-style: italic;
  font-size: 0.85rem;
}

.load-more {
  text-align: center;
  padding: 1.5rem;
  background: var(--bg2);
  border-radius: var(--radius);
  margin-top: 1rem;
}

.load-more button {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
}

.load-more-info {
  color: var(--muted);
  font-size: 0.85rem;
  margin-top: 0.75rem;
}

.loading {
  text-align: center;
  padding: 1.5rem;
  color: var(--muted);
}

.empty {
  text-align: center;
  padding: 2rem;
  color: var(--muted);
}

.empty p {
  margin-bottom: 0.75rem;
}

/* Modale de conflit */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 1rem;
}

.modal {
  background: var(--bg2);
  border-radius: var(--radius);
  max-width: 700px;
  width: 100%;
  max-height: 85vh;
  overflow-y: auto;
  padding: 1.5rem;
}

.modal h2 {
  color: var(--danger);
  margin-bottom: 0.5rem;
}

.modal > p {
  color: var(--text2);
  margin-bottom: 1rem;
}

.versions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

/* Sur mobile on met les versions en colonne */
@media (max-width: 600px) {
  .versions {
    grid-template-columns: 1fr;
  }
}

.version {
  background: var(--bg);
  padding: 1rem;
  border-radius: var(--radius);
  border: 2px solid var(--border);
}

.version.local {
  border-color: var(--accent);
}

.version.remote {
  border-color: var(--warning);
}

.version h4 {
  margin-bottom: 0.5rem;
}

.version.local h4 {
  color: var(--accent);
}

.version.remote h4 {
  color: var(--warning);
}

.version p {
  font-size: 0.85rem;
  color: var(--text2);
  margin-bottom: 0.5rem;
}

.version button {
  width: 100%;
  margin-top: 0.5rem;
}

.remaining {
  text-align: center;
  color: var(--muted);
  margin-top: 1rem;
  font-size: 0.85rem;
}

.footer {
  text-align: center;
  padding: 1.5rem;
  color: var(--muted);
  border-top: 1px solid var(--border);
  margin-top: 1.5rem;
  font-size: 0.85rem;
}
</style>
