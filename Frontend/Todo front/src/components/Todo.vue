<template>
  <div class="todo-container">
    <h1>Todo List</h1>
    
    <div class="todo-layout">
      <div class="add-todo-section">
        <h2>Add New Task</h2>
        <div class="add-todo">
          <input 
            v-model="newTodo" 
            @keyup.enter="addTodo"
            placeholder="Add a new task..."
            class="todo-input">
          <div class="priority-buttons">
            <button 
              v-for="priority in priorities" 
              :key="priority.value"
              @click="newPriority = priority.value"
              :class="['priority-button', priority.value, { active: newPriority === priority.value }]"
            >
              {{ priority.label }}
              <span v-if="newPriority === priority.value" class="priority-check">✓</span>
            </button>
          </div>
          <button @click="addTodo" class="add-button">Add Task</button>
        </div>


        <div class="statistics-section">
          <h3>Task Statistics</h3>
          <div class="statistics-grid">
            <div class="stat-card total">
              <span class="stat-number">{{ todos.length }}</span>
              <span class="stat-label">Total Tasks</span>
            </div>
            <div class="stat-card completed">
              <span class="stat-number">{{ completedTasks }}</span>
              <span class="stat-label">Completed</span>
            </div>
            <div class="stat-card pending">
              <span class="stat-number">{{ pendingTasks }}</span>
              <span class="stat-label">Pending</span>
            </div>
            <div class="stat-card progress">
              <span class="stat-number">{{ completionRate }}%</span>
              <span class="stat-label">Completion Rate</span>
            </div>
          </div>
        </div>
      </div>


      <div class="todo-list-section">
        <h2>Tasks</h2>
        <div class="todo-list">
          <div v-for="todo in sortedTodos" :key="todo._id" class="todo-item" :class="todo.priority">
            <div class="todo-content">
              <input 
                type="checkbox" 
                v-model="todo.completed"
                @change="updateTodo(todo)"
                class="todo-checkbox"
              >
              <input 
                v-if="editingTodo === todo._id"
                v-model="todo.text"
                @blur="updateTodo(todo)"
                @keyup.enter="updateTodo(todo)"
                class="todo-edit-input"
                ref="editInput"
              >
              <span 
                v-else 
                @click="startEditing(todo)"
                :class="{ completed: todo.completed }"
                class="todo-text"
              >
                {{ todo.text }}
              </span>
              <div v-if="editingTodo === todo._id" class="priority-buttons">
                <button 
                  v-for="priority in priorities" 
                  :key="priority.value"
                  @click="updateTodoPriority(todo, priority.value)"
                  :class="['priority-button', priority.value, { active: todo.priority === priority.value }]"
                >
                  {{ priority.label }}
                </button>
              </div>
              <span v-else class="priority-badge" :class="todo.priority">
                {{ todo.priority.charAt(0).toUpperCase() + todo.priority.slice(1) }}
              </span>
            </div>
            <div class="todo-actions">
              <button @click="startEditing(todo)" class="edit-button">Edit</button>
              <button @click="deleteTodo(todo._id)" class="delete-button" title="Delete">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M3 6h18"></path>
                  <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path>
                  <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path>
                </svg>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

interface Todo {
  _id: string
  text: string
  completed: boolean
  priority: string
  createdAt: string
}

const priorities = [
  { value: 'high', label: 'High' },
  { value: 'medium', label: 'Medium' },
  { value: 'low', label: 'Low' }
]

const todos = ref<Todo[]>([])
const newTodo = ref('')
const newPriority = ref('medium')
const editingTodo = ref<string | null>(null)
const API_URL = 'http://backend/api'

const sortedTodos = computed(() => {
  return [...todos.value].sort((a, b) => {
    if (a.completed !== b.completed) {
      return a.completed ? 1 : -1
    }
    const priorityOrder = { high: 0, medium: 1, low: 2 }
    const priorityDiff = priorityOrder[a.priority as keyof typeof priorityOrder] - priorityOrder[b.priority as keyof typeof priorityOrder]
    if (priorityDiff !== 0) return priorityDiff
    return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
  })
})

const completedTasks = computed(() => todos.value.filter(todo => todo.completed).length)
const pendingTasks = computed(() => todos.value.filter(todo => !todo.completed).length)
const completionRate = computed(() => {
  if (todos.value.length === 0) return 0
  return Math.round((completedTasks.value / todos.value.length) * 100)
})

const fetchTodos = async () => {
  try {
    const response = await fetch(`${API_URL}/todos`)
    todos.value = await response.json()
  } catch (error) {
    console.error('Error fetching todos:', error)
  }
}

const addTodo = async () => {
  if (!newTodo.value.trim()) return
  
  try {
    const response = await fetch(`${API_URL}/todos`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ 
        text: newTodo.value,
        priority: newPriority.value
      }),
    })
    const todo = await response.json()
    todos.value.unshift(todo)
    newTodo.value = ''
    newPriority.value = 'medium'
  } catch (error) {
    console.error('Error adding todo:', error)
  }
}

const updateTodo = async (todo: Todo) => {
  try {
    await fetch(`${API_URL}/todos/${todo._id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(todo),
    })
    editingTodo.value = null
  } catch (error) {
    console.error('Error updating todo:', error)
  }
}

const deleteTodo = async (id: string) => {
  try {
    await fetch(`${API_URL}/todos/${id}`, {
      method: 'DELETE',
    })
    todos.value = todos.value.filter(todo => todo._id !== id)
  } catch (error) {
    console.error('Error deleting todo:', error)
  }
}


const startEditing = (todo: Todo) => {
  editingTodo.value = todo._id
}

const updateTodoPriority = async (todo: Todo, priority: string) => {
  todo.priority = priority
  await updateTodo(todo)
}

onMounted(() => {
  fetchTodos()
})
</script>

<style scoped>
.todo-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f5f5;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
  font-size: 2.5rem;
}

.todo-layout {
  display: flex;
  gap: 40px;
  flex: 1;
  margin: 0 auto;
  width: 100%;
  max-width: 1100px;
}

.add-todo-section {
  flex: 1;
  padding: 30px;
  background-color: #f5f5f5;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  height: fit-content;
}

.todo-list-section {
  flex: 2;
  padding: 30px;
  background-color: #f5f5f5;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

h2 {
  color: #333;
  margin-bottom: 20px;
  font-size: 1.5rem;
}

.add-todo {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.todo-input {
  padding: 12px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 0.3s;
}

.todo-input:focus {
  border-color: #2196F3;
  outline: none;
}

.priority-buttons {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.priority-button {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.3s;
  opacity: 0.7;
  position: relative;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.priority-button.active {
  opacity: 1;
  transform: scale(1.05);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.priority-check {
  position: absolute;
  top: -5px;
  right: -5px;
  background: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.priority-button.high {
  background-color: #ffebee;
  color: #d32f2f;
}

.priority-button.medium {
  background-color: #fff3e0;
  color: #f57c00;
}

.priority-button.low {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.priority-button:hover {
  opacity: 1;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.priority-badge {
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 14px;
  font-weight: 500;
  text-transform: capitalize;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.priority-badge.high {
  background-color: #ffebee;
  color: #d32f2f;
}

.priority-badge.medium {
  background-color: #fff3e0;
  color: #f57c00;
}

.priority-badge.low {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.add-button {
  padding: 12px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: background-color 0.3s;
}

.add-button:hover {
  background-color: #43A047;
}

.todo-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.todo-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  background-color: white;
  border-radius: 8px;
  border: 1px solid #ddd;
  transition: all 0.2s;
  margin-bottom: 10px;
}

.todo-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.todo-content {
  display: flex;
  align-items: center;
  gap: 15px;
  flex: 1;
  min-width: 0;
}

.todo-checkbox {
  width: 22px;
  height: 22px;
  cursor: pointer;
  border: 2px solid #ddd;
  border-radius: 6px;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  background-color: white;
  transition: all 0.2s;
  position: relative;
  margin: 0;
}

.todo-checkbox:checked {
  background-color: #4CAF50;
  border-color: #4CAF50;
}

.todo-checkbox:checked::after {
  content: '✓';
  position: absolute;
  color: white;
  font-size: 14px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.todo-checkbox:hover {
  border-color: #4CAF50;
  box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.2);
}

.todo-checkbox:focus {
  outline: none;
  box-shadow: 0 0 0 3px rgba(76, 175, 80, 0.3);
}

.todo-text {
  font-size: 16px;
  flex: 1;
  color: #333;
  margin: 0 10px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}

.todo-text.completed {
  text-decoration: line-through;
  color: #888;
}

.todo-edit-input {
  flex: 1;
  padding: 8px;
  font-size: 16px;
  border: 2px solid #2196F3;
  border-radius: 6px;
}

.todo-actions {
  display: flex;
  gap: 8px;
  margin-left: 15px;
}

.edit-button, .delete-button {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.edit-button {
  background-color: #2196F3;
  color: white;
}

.edit-button:hover {
  background-color: #1976D2;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(33, 150, 243, 0.3);
}

.delete-button {
  padding: 8px;
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
}

.delete-button:hover {
  background-color: #d32f2f;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(244, 67, 54, 0.3);
}

.delete-button:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.delete-button svg {
  width: 16px;
  height: 16px;
}

.edit-button:active, .delete-button:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.edit-button::before {
  content: '✎';
  font-size: 14px;
}

.delete-button::before {
  content: '×';
  font-size: 18px;
  font-weight: bold;
}

.statistics-section {
  margin-top: 30px;
  padding: 20px;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.statistics-section h3 {
  color: #333;
  margin-bottom: 20px;
  font-size: 1.2rem;
  text-align: center;
}

.statistics-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
}

.stat-card {
  padding: 15px;
  border-radius: 8px;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 5px;
  transition: transform 0.2s;
}

.stat-card:hover {
  transform: translateY(-2px);
}

.stat-number {
  font-size: 24px;
  font-weight: bold;
}

.stat-label {
  font-size: 14px;
  color: #666;
}

.stat-card.total {
  background-color: #e3f2fd;
  color: #1976D2;
}

.stat-card.completed {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.stat-card.pending {
  background-color: #fff3e0;
  color: #f57c00;
}

.stat-card.progress {
  background-color: #f3e5f5;
  color: #7b1fa2;
}

@media (max-width: 768px) {
  .todo-layout {
    flex-direction: column;
  }
  
  .todo-container {
    padding: 10px;
  }
  
  .add-todo-section,
  .todo-list-section {
    padding: 20px;
  }
}
</style> 