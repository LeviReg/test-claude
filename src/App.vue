<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import TaskNode from './components/TaskNode.vue'

const STORAGE_KEY = 'work-day-notes-vue'

const notes = ref([])
const selectedDate = ref(new Date().toISOString().slice(0, 10))
const noteText = ref('')
const selectedNoteId = ref(null)
const newTaskText = ref('')

const selectedNote = computed(() => notes.value.find((n) => n.id === selectedNoteId.value) || null)

function loadFromStorage() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) {
      notes.value = JSON.parse(raw)
      if (notes.value.length) {
        selectedNoteId.value = notes.value[0].id
      }
    }
  } catch (error) {
    console.warn('Could not load storage', error)
  }
}

function saveToStorage() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(notes.value))
}

watch(notes, saveToStorage, { deep: true })

onMounted(loadFromStorage)

function createId() {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`
}

function addNote() {
  const note = {
    id: createId(),
    date: selectedDate.value,
    text: noteText.value.trim(),
    keyPoints: [],
    todos: [],
  }

  if (!note.text) {
    alert('Enter a note first')
    return
  }

  notes.value.unshift(note)
  selectedNoteId.value = note.id
  noteText.value = ''
}

function selectNote(id) {
  selectedNoteId.value = id
}

function deleteNote(id) {
  notes.value = notes.value.filter((note) => note.id !== id)
  if (notes.value.length) {
    selectedNoteId.value = notes.value[0].id
  } else {
    selectedNoteId.value = null
  }
}

function extractKeyPoints(text) {
  const lines = text
    .split(/\n+/)
    .map((s) => s.trim())
    .filter(Boolean)

  const verbs = /\b(fix|deploy|release|review|test|document|write|plan|meet|schedule|follow up|update|refactor|implement|create|design|analyze)\b/i

  const points = []

  lines.forEach((line) => {
    if (verbs.test(line) || /\b(need|should|must|todo|asap|for|by)\b/i.test(line)) {
      points.push(line.replace(/^[\-\*\d\.\s]+/, '').trim())
    }
  })

  if (points.length === 0) {
    points.push(...lines.slice(0, 3))
  }

  return Array.from(new Set(points)).filter(Boolean)
}

function generateTodos() {
  if (!selectedNote.value) return

  const keyPoints = extractKeyPoints(selectedNote.value.text)
  selectedNote.value.keyPoints = keyPoints

  const todos = keyPoints.map((point) => ({
    id: createId(),
    parentId: null,
    text: point,
    done: false,
    priority: 'normal',
  }))

  selectedNote.value.todos = todos
  newTaskText.value = ''
}

function addTask() {
  if (!selectedNote.value) return
  if (!newTaskText.value.trim()) return

  selectedNote.value.todos.push({
    id: createId(),
    parentId: null,
    text: newTaskText.value.trim(),
    done: false,
    priority: 'normal',
  })
  newTaskText.value = ''
}

function addSubtask(parentId) {
  const text = prompt('Subtask description')
  if (!text) return
  selectedNote.value.todos.push({
    id: createId(),
    parentId,
    text: text.trim(),
    done: false,
    priority: 'normal',
  })
}

function toggleDone(task) {
  task.done = !task.done
}

function updateTaskText(task, value) {
  task.text = value
}

function deleteTask(taskId) {
  if (!selectedNote.value) return

  const toRemove = new Set([taskId])
  let changed

  do {
    changed = false
    selectedNote.value.todos.forEach((task) => {
      if (task.parentId && toRemove.has(task.parentId) && !toRemove.has(task.id)) {
        toRemove.add(task.id)
        changed = true
      }
    })
  } while (changed)

  selectedNote.value.todos = selectedNote.value.todos.filter((task) => !toRemove.has(task.id))
}

function cyclePriority(task) {
  const order = ['low', 'normal', 'high']
  const next = order[(order.indexOf(task.priority) + 1) % order.length]
  task.priority = next
}

function buildTree(tasks) {
  const map = {}
  tasks.forEach((task) => {
    map[task.id] = task
    // attach children array to the actual task object so updates stay reactive
    map[task.id].children = []
  })

  const roots = []

  tasks.forEach((task) => {
    if (task.parentId && map[task.parentId]) {
      map[task.parentId].children.push(map[task.id])
    } else if (!task.parentId) {
      roots.push(map[task.id])
    }
  })

  function sortNodes(nodes) {
    nodes.sort((a, b) => (a.done === b.done ? 0 : a.done ? 1 : -1))
    nodes.forEach((n) => sortNodes(n.children))
  }

  sortNodes(roots)
  return roots
}

const treeTasks = computed(() => {
  if (!selectedNote.value) return []
  return buildTree(selectedNote.value.todos)
})
</script>

<template>
  <div class="container">
    <header>
      <h1>Daily Work Notes + Todo Builder</h1>
      <p>Capture your day, extract action points, and manage interactive subtasks.</p>
    </header>

    <section class="note-creation">
      <label>
        Date:
        <input type="date" v-model="selectedDate" />
      </label>
      <textarea v-model="noteText" rows="6" placeholder="Write your daily notes here... (e.g. what you accomplished and next steps)"></textarea>
      <div class="buttons">
        <button @click="addNote">Add Note</button>
      </div>
    </section>

    <section class="notes-list">
      <h2>Notes</h2>
      <ul>
        <li v-for="note in notes" :key="note.id" :class="{active: note.id === selectedNoteId}">
          <button @click="selectNote(note.id)">{{ note.date }} — {{ note.text.slice(0, 40) || '(empty)' }}</button>
          <button class="danger" @click="deleteNote(note.id)">Delete</button>
        </li>
      </ul>
    </section>

    <section v-if="selectedNote" class="note-detail">
      <h2>Selected Note: {{ selectedNote.date }}</h2>
      <textarea v-model="selectedNote.text" rows="6"></textarea>

      <div class="buttons">
        <button @click="generateTodos">Extract Key Points + Generate Todos</button>
        <button @click="selectedNote.keyPoints = extractKeyPoints(selectedNote.text)">Extract Key Points Only</button>
      </div>

      <div class="keypoints">
        <h3>Key Points</h3>
        <ul>
          <li v-for="(point, index) in selectedNote.keyPoints" :key="index">{{ point }}</li>
          <li v-if="selectedNote.keyPoints.length === 0">No key points extracted yet.</li>
        </ul>
      </div>

      <div class="todo-input">
        <input v-model="newTaskText" placeholder="Add a new root todo" @keydown.enter.prevent="addTask" />
        <button @click="addTask">Add Todo</button>
      </div>

      <div class="todo-tree">
        <h3>Todo Tree</h3>

        <ul>
          <TaskNode
            v-for="task in treeTasks"
            :key="task.id"
            :task="task"
            :depth="0"
            @toggleDone="toggleDone"
            @updateTaskText="updateTaskText"
            @cyclePriority="cyclePriority"
            @addSubtask="addSubtask"
            @deleteTask="deleteTask"
          />
          <li v-if="treeTasks.length === 0">No todos yet. Generate from key points or add one.</li>
        </ul>
      </div>
    </section>
  </div>
</template>



<style scoped>
.container {
  max-width: 980px;
  margin: 0 auto;
  padding: 24px;
  background: #000000;
  color: #0088ff;
  font-family: 'Courier New', monospace;
  min-height: 100vh;
  border: 1px solid #0088ff;
  box-shadow: 0 0 20px rgba(0, 136, 255, 0.3);
}

header h1 {
  margin-bottom: 4px;
  color: #0088ff;
  text-shadow: 0 0 10px #0088ff;
  font-weight: bold;
}

header p {
  margin-top: 0;
  color: #0066aa;
}

.note-creation,
.notes-list,
.note-detail {
  margin-bottom: 30px;
  background: #111111;
  border: 1px solid #0088ff;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 0 15px rgba(0, 136, 255, 0.2);
}

.note-creation textarea,
.note-detail textarea,
.todo-input input {
  width: 100%;
  margin-top: 12px;
  padding: 12px;
  font-size: 15px;
  border: 1px solid #0088ff;
  border-radius: 6px;
  resize: vertical;
  background: #000000;
  color: #0088ff;
  font-family: 'Courier New', monospace;
  box-shadow: inset 0 0 10px rgba(0, 136, 255, 0.1);
  margin-bottom: 10px;
}

.note-creation textarea:focus,
.note-detail textarea:focus,
.todo-input input:focus {
  outline: none;
  box-shadow: 0 0 10px rgba(0, 136, 255, 0.5);
}

.buttons {
  margin-top: 15px;
}

button {
  cursor: pointer;
  border: 1px solid #0088ff;
  border-radius: 5px;
  font-size: 14px;
  padding: 10px 14px;
  margin-right: 10px;
  margin-bottom: 8px;
  background: #000000;
  color: #0088ff;
  font-family: 'Courier New', monospace;
  transition: box-shadow 0.3s;
}

button:hover {
  box-shadow: 0 0 10px rgba(0, 136, 255, 0.5);
}

button.danger {
  border-color: #ff4444;
  color: #ff4444;
}

button.danger:hover {
  box-shadow: 0 0 10px rgba(255, 68, 68, 0.5);
}

.notes-list ul,
.todo-tree ul,
.keypoints ul {
  list-style: none;
  padding-left: 0;
}

.notes-list li,
.task-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  border-bottom: 1px solid #333333;
  padding-bottom: 8px;
}

.notes-list li.active button:first-child {
  font-weight: bold;
  color: #0088ff;
  text-shadow: 0 0 5px #0088ff;
}

.task-row .task-text {
  flex: 1;
  border: 1px solid #0088ff;
  border-radius: 4px;
  padding: 8px;
  background: #000000;
  color: #0088ff;
  font-family: 'Courier New', monospace;
  margin-bottom: 5px;
}

.task-text.done {
  text-decoration: line-through;
  color: #666666;
}

.priority {
  border-radius: 4px;
  padding: 2px 8px;
  border: 1px solid #0088ff;
  font-size: 12px;
  cursor: pointer;
  background: #000000;
  color: #0088ff;
  font-family: 'Courier New', monospace;
  margin-left: 5px;
}

.priority.low {
  border-color: #0066aa;
  color: #0066aa;
}

.priority.normal {
  border-color: #0088ff;
  color: #0088ff;
}

.priority.high {
  border-color: #ff4444;
  color: #ff4444;
}

.child {
  margin-left: 30px;
  border-left: 1px dashed #0088ff;
  padding-left: 20px;
}

.todo-input {
  margin-top: 15px;
}

/* ensure safe selector syntax for environments that reject nested a-rule grammars */
:where(.notes-list, .todo-tree, .task-row) :is(a) {
  color: #0088ff;
  text-decoration: underline;
}

.todo-input input {
  width: calc(100% - 140px);
  margin-right: 10px;
  background: #000000;
  color: #0088ff;
  border: 1px solid #0088ff;
  font-family: 'Courier New', monospace;
  padding: 8px;
  margin-bottom: 5px;
}

.todo-input input:focus {
  box-shadow: 0 0 10px rgba(0, 136, 255, 0.5);
}
</style>
