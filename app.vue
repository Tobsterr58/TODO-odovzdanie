<script setup lang="ts">
import { ref, onMounted } from 'vue'

// Definition of Todos
type TodoStatus = 'todo' | 'in-progress' | 'completed'

interface Todo {
  id: number
  text: string
  status: TodoStatus
  isEditing?: boolean
  createdAt: string
}

const API_URL = 'http://localhost:3001/todos'


// This will hold ALL our todos from the database
const allTodos = ref<Todo[]>([])

// These will hold the todos for each column, filtered from `allTodos`
const todoItems = ref<Todo[]>([])
const inProgressItems = ref<Todo[]>([])
const completedItems = ref<Todo[]>([])

// For adding a new todo
const newTodoText = ref('')

type SortOrder = 'default' | 'asc' | 'desc'
const currentSortOrder = ref<SortOrder>('default') // Starts with no specific sorting

const isLoading = ref(true)
const hasError = ref(false)


// Function to fetch all todos from the backend
const fetchTodos = async () => {
  isLoading.value = true
  hasError.value = false
  try {
    const response = await fetch(API_URL) // Make the request
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }
    const data: Todo[] = await response.json()
    allTodos.value = data
    applySorting()
  } catch (e) {
    console.error('Failed to fetch todos:', e)
    hasError.value = true
  } finally {
    isLoading.value = false
  }
}

// Function to put todos into their correct columns and apply sorting
const categorizeAndSortTodos = () => {
  let sortedTodos = [...allTodos.value];

  // Apply sorting based on currentSortOrder
  if (currentSortOrder.value === 'asc') {
    sortedTodos.sort((a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime());
  } else if (currentSortOrder.value === 'desc') {
    sortedTodos.sort((a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime());
  }

  todoItems.value = sortedTodos.filter(t => t.status === 'todo')
  inProgressItems.value = sortedTodos.filter(t => t.status === 'in-progress')
  completedItems.value = sortedTodos.filter(t => t.status === 'completed')
}

// Function to handle the sort button click
const toggleSortOrder = () => {
  if (currentSortOrder.value === 'default') {
    currentSortOrder.value = 'asc';
  } else if (currentSortOrder.value === 'asc') {
    currentSortOrder.value = 'desc';
  } else {
    currentSortOrder.value = 'default';
  }
  applySorting();
}

// Helper function to apply the current sorting and re-categorize
const applySorting = () => {
  categorizeAndSortTodos();
}


// Function to create a new todo
const createTodo = async () => {
  const text = newTodoText.value.trim()
  if (!text) return // Don't create empty todos

  const newTodo: Omit<Todo, 'id'> = {
    text,
    status: 'todo',
    createdAt: new Date().toISOString(),
  }

  try {
    // Send POST request to json-server
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(newTodo),
    })
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }
    const createdTodo: Todo = await response.json()

    allTodos.value.push(createdTodo)
    applySorting()
    newTodoText.value = ''
  } catch (e) {
    console.error('Failed to create todo:', e)
  }
}

// Function to delete a todo
const deleteTodo = async (itemToDelete: Todo) => {
  try {
    // Send DELETE request
    const response = await fetch(`${API_URL}/${itemToDelete.id}`, {
      method: 'DELETE',
    })
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }

    allTodos.value = allTodos.value.filter(t => t.id !== itemToDelete.id)
    applySorting()
  } catch (e) {
    console.error('Failed to delete todo:', e)
  }
}

// Function to toggle edit mode for a todo
const toggleEdit = (item: Todo) => {
  item.isEditing = !item.isEditing
}

// Function to save edited todo text
const editTodo = async (itemToEdit: Todo) => {
  const newText = itemToEdit.text.trim()
  if (!newText) {
    console.warn("Todo text cannot be empty. Reverting edit.");
    fetchTodos();
    itemToEdit.isEditing = false;
    return;
  }

  try {
    // Send PATCH request to update text
    const response = await fetch(`${API_URL}/${itemToEdit.id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: newText }),
    })
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }
    itemToEdit.isEditing = false
  } catch (e) {
    console.error('Failed to edit todo:', e)
  }
}

const getNextStatus = (currentStatus: TodoStatus, direction: 'left' | 'right'): TodoStatus | null => {
  if (direction === 'left') {
    if (currentStatus === 'completed') return 'in-progress'
    if (currentStatus === 'in-progress') return 'todo'
  } else if (direction === 'right') {
    if (currentStatus === 'todo') return 'in-progress'
    if (currentStatus === 'in-progress') return 'completed'
  }
  return null
}

// Function to update a todo's status (when an arrow is clicked)
const updateStatusViaArrow = async (itemToMove: Todo, direction: 'left' | 'right') => {
  const newStatus = getNextStatus(itemToMove.status, direction)
  if (!newStatus) {
    console.warn(`Cannot move "${itemToMove.text}" ${direction} from "${itemToMove.status}" status.`);
    return; // Don't do anything if move is invalid
  }

  if (itemToMove.status === newStatus) return

  try {
    // Send PATCH request to update status
    const response = await fetch(`${API_URL}/${itemToMove.id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: newStatus }),
    })
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }

    const index = allTodos.value.findIndex(t => t.id === itemToMove.id)
    if (index !== -1) {
      allTodos.value[index].status = newStatus
    }
    applySorting() // Re-categorize and sort the todos
  } catch (e) {
    console.error('Failed to update status via arrow:', e)
  }
}

onMounted(() => {
  fetchTodos() // Load todos when the app starts
})


const columns = [
  { status: 'todo', title: '📝 To Do', items: todoItems },
  { status: 'in-progress', title: '🚧 In Progress', items: inProgressItems },
  { status: 'completed', title: '✅ Completed', items: completedItems },
]

const sortButtonText = ref('Sort: Default');
watchEffect(() => {
  if (currentSortOrder.value === 'default') {
    sortButtonText.value = 'Sort: Default (by creation)';
  } else if (currentSortOrder.value === 'asc') {
    sortButtonText.value = 'Sort: Oldest First (Asc)';
  } else {
    sortButtonText.value = 'Sort: Newest First (Desc)';
  }
});

</script>

<template>
  <div class="kanban-board">
    <header class="board-header">
      <h1>My Simple ToDo Board</h1>
    </header>

    <div class="add-todo-section">
      <input
        v-model="newTodoText"
        placeholder="Add a new task..."
        @keyup.enter="createTodo"
        class="new-todo-input"
      />
      <button @click="createTodo" class="create-button">Add Task</button>
    </div>

    <div class="filter-sort-section">
      <button @click="toggleSortOrder" class="sort-button">{{ sortButtonText }}</button>
    </div>

    <div v-if="isLoading" class="status-message">Loading tasks...</div>
    <div v-else-if="hasError" class="status-message error">
      Error loading tasks. Is the server running on {{ API_URL }}?
    </div>
    <main v-else class="board-columns">
      <div v-for="column in columns" :key="column.status" class="column">
        <h2 class="column-title">{{ column.title }}</h2>

        <div class="card-list">
          <div v-for="element in column.items.value" :key="element.id" class="card">
            <template v-if="element.isEditing">
              <input
                v-model="element.text"
                @blur="editTodo(element)"
                @keyup.enter="editTodo(element)"
                autofocus
                class="edit-input"
              />
            </template>
            <template v-else>
              <span class="todo-text">{{ element.text }}</span>
            </template>

            <div class="card-controls">
              <button
                v-if="column.status === 'in-progress' || column.status === 'completed'"
                @click="updateStatusViaArrow(element, 'left')"
                title="Move Left"
                class="arrow-button left-arrow"
              >
                ◀
              </button>

              <div class="card-actions">
                <button @click="toggleEdit(element)" title="Edit" class="action-button">✏️</button>
                <button @click="deleteTodo(element)" title="Delete" class="action-button">🗑️</button>
              </div>

              <button
                v-if="column.status === 'in-progress' || column.status === 'todo'"
                @click="updateStatusViaArrow(element, 'right')"
                title="Move Right"
                class="arrow-button right-arrow"
              >
                ▶
              </button>
            </div>
            <div class="card-date" v-if="element.createdAt">
              Created: {{ new Date(element.createdAt).toLocaleDateString() }}
            </div>
          </div>
          <p v-if="column.items.value.length === 0" class="empty-column-message">No tasks here yet!</p>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>

.kanban-board {
  display: flex;
  flex-direction: column;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  background-color: #f0f2f5;
  min-height: 100vh;
  padding: 1rem;
  box-sizing: border-box;
}

.board-header {
  padding: 1rem;
  text-align: center;
}

.board-header h1 {
  font-size: clamp(1.5rem, 5vw, 2.5rem);
  color: #172b4d;
  margin-bottom: 0;
}

.status-message {
  text-align: center;
  font-size: clamp(1rem, 3vw, 1.2rem);
  color: #5e6c84;
  padding: 2rem;
}
.status-message.error {
  color: #de350b;
}

.add-todo-section {
  margin-bottom: 1.5rem;
  text-align: center;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.8rem;
}

.new-todo-input {
  padding: 0.75rem 1rem;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  width: 100%;
  max-width: 400px;
  border: 1px solid #c1c7d0;
  border-radius: 6px;
  box-sizing: border-box;
}

.create-button {
  padding: 0.75rem 1.5rem;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  cursor: pointer;
  background-color: #0052cc;
  color: white;
  border: none;
  border-radius: 6px;
  transition: background-color 0.2s ease;
  flex-shrink: 0;
}

.create-button:hover {
  background-color: #0043b3;
}

.filter-sort-section {
  margin-bottom: 1.5rem;
  text-align: center;
}

.sort-button {
  padding: 0.75rem 1.5rem;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  cursor: pointer;
  background-color: #5e6c84;
  color: white;
  border: none;
  border-radius: 6px;
  transition: background-color 0.2s ease;
}

.sort-button:hover {
  background-color: #42526e;
}

.board-columns {
  display: flex;
  gap: 1rem;
  flex-grow: 1;
  /* Make columns stack on smaller screens */
  flex-direction: column;
}

/* For larger screens, make columns side-by-side */
@media (min-width: 768px) {
  .board-columns {
    flex-direction: row;
  }
}

.column {
  flex: 1; /* Each column takes equal space */
  background-color: #ebecf0;
  border-radius: 8px;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  max-height: 85vh;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  margin-bottom: 1rem;
}

@media (min-width: 768px) {
  .column {
    margin-bottom: 0;
  }
}

.column-title {
  font-size: clamp(1.1rem, 3.5vw, 1.25rem);
  font-weight: 600;
  margin-bottom: 1rem;
  padding-bottom: 0.5rem;
  border-bottom: 1px solid #c1c7d0;
  color: #172b4d;
}

.card-list {
  flex-grow: 1;
  overflow-y: auto;
  padding-right: 5px;
}

.card-list::-webkit-scrollbar {
  width: 8px;
}

.card-list::-webkit-scrollbar-track {
  background: #f0f2f5;
  border-radius: 4px;
}

.card-list::-webkit-scrollbar-thumb {
  background: #c1c7d0;
  border-radius: 4px;
}

.card-list::-webkit-scrollbar-thumb:hover {
  background: #97a0af;
}

.card {
  background-color: white;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  box-shadow: 0 1px 2px rgba(9, 30, 66, 0.1);
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
  transition: box-shadow 0.2s ease;
  min-height: 48px;
  box-sizing: border-box;
}

.card:hover {
  box-shadow: 0 4px 8px rgba(9, 30, 66, 0.15);
}

.todo-text {
  flex-grow: 1;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  word-break: break-word;
  min-width: 0;
  padding-right: 0.5rem;
}

.edit-input {
  flex-grow: 1;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  border: 1px solid #ccc;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  box-sizing: border-box;
}

.card-controls {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  flex-shrink: 0;
  margin-left: auto;
}

.card-actions {
  display: flex;
  gap: 0.5rem;
}

.action-button,
.arrow-button {
  background: none;
  border: none;
  font-size: clamp(1rem, 3vw, 1.1rem);
  cursor: pointer;
  padding: 0.2rem;
  line-height: 1;
  opacity: 0.7;
  transition: opacity 0.2s ease, transform 0.2s ease;
  color: #5e6c84;
}

.action-button:hover,
.arrow-button:hover {
  opacity: 1;
  color: #172b4d;
  transform: scale(1.1);
}

.arrow-button {
  font-size: clamp(1.2rem, 4vw, 1.5rem);
  color: #0052cc;
}

.arrow-button.left-arrow {
  margin-right: 0.5rem;
}

.arrow-button.right-arrow {
  margin-left: 0.5rem;
}

.card-date {
  width: 100%;
  font-size: clamp(0.7rem, 2vw, 0.8rem);
  color: #8c9aac;
  text-align: right;
  margin-top: 0.5rem;
}

.empty-column-message {
  color: #5e6c84;
  text-align: center;
  padding: 2rem;
  font-style: italic;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
}
</style>