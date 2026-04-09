<script setup>
import { ref, computed, onMounted, watch } from 'vue'

// --- СОСТОЯНИЕ (DATA) ---
const activeTab = ref('today')
const isCollapsed = ref(false)
const isVisible = ref(true)
const isAdding = ref(false)

const newTaskText = ref('')
const newTaskDate = ref(new Date().toISOString().substr(0, 10))

// ИСПРАВЛЕНО: Добавлен пустой массив [] для корректной работы синтаксиса ||
const todos = ref(JSON.parse(localStorage.getItem('my-todos')) || [])

// Автосохранение при любых изменениях
watch(todos, (newVal) => {
    localStorage.setItem('my-todos', JSON.stringify(newVal))
}, { deep: true })

// --- ЛОГИКА ---
const filteredTodos = computed(() => {
    return todos.value.filter(todo => todo.category === activeTab.value)
})

const setActive = (tabId) => {
    activeTab.value = tabId
    isAdding.value = false
    // Безопасное закрытие мобильного меню
    const offcanvasElement = document.getElementById('mobileSidebar')
    if (offcanvasElement && typeof bootstrap !== 'undefined') {
        const instance = bootstrap.Offcanvas.getInstance(offcanvasElement)
        if (instance) instance.hide()
    }
}

const saveTask = () => {
    if (!newTaskText.value.trim()) return
    todos.value.push({
        id: Date.now(),
        text: newTaskText.value.trim(),
        date: newTaskDate.value,
        category: activeTab.value,
        done: false
    })
    newTaskText.value = ''
    isAdding.value = false
}

const deleteTask = (id) => {
    todos.value = todos.value.filter(t => t.id !== id)
}

const toggleDesktop = () => isCollapsed.value = !isCollapsed.value
const toggleCat = () => isVisible.value = false

onMounted(() => {
    const myOffcanvas = document.getElementById('mobileSidebar')
    if (myOffcanvas) {
        myOffcanvas.addEventListener('hidden.bs.offcanvas', () => isVisible.value = true)
    }
})
</script>

<template>
    <div class="d-flex w-100 min-vh-100 bg-white main-app-container">
        
        <!-- КНОПКА КОТИК (ПОЛНОСТЬЮ ПЛОСКАЯ И ПРОЗРАЧНАЯ) -->
        <button 
            v-if="isVisible" 
            class="btn d-md-none headerBtn position-fixed border-0 bg-transparent shadow-none p-0" 
            type="button"
            data-bs-toggle="offcanvas" 
            data-bs-target="#mobileSidebar" 
            @click="toggleCat"
        >
            <img src="/CatHeader.png" alt="Menu" class="cat-trigger-img">
        </button>

        <!-- САЙДБАР ПК -->
        <header class="d-none d-md-flex flex-column vh-100 p-3 bg-light side-menu border-right" :class="{ 'collapsed': isCollapsed }">
            <div class="d-flex justify-content-between align-items-center mb-4">
                <h4 v-if="!isCollapsed" class="m-0 fw-bold">Меню</h4>
                <button @click="toggleDesktop" class="btn btn-sm p-0 border-0 bg-transparent shadow-none">
                    <img src="/CatHeader.png" style="max-width: 30px;">
                </button>
            </div>
            <nav class="nav flex-column">
                <a class="nav-link mb-1" :class="{ 'active-item': activeTab === 'today' }" @click.prevent="setActive('today')" href="#">
                    <span class="icon">☀️</span>
                    <span v-if="!isCollapsed" class="ms-3">Сегодня</span>
                </a>
                <a class="nav-link mb-1" :class="{ 'active-item': activeTab === 'upcoming' }" @click.prevent="setActive('upcoming')" href="#">
                    <span class="icon">📅</span>
                    <span v-if="!isCollapsed" class="ms-3">На потом</span>
                </a>
            </nav>
        </header>

        <!-- МОБИЛЬНОЕ МЕНЮ -->
        <div class="offcanvas offcanvas-start" id="mobileSidebar" tabindex="-1">
            <div class="offcanvas-header border-bottom">
                <h5 class="offcanvas-title fw-bold">Список дел</h5>
                <button type="button" class="btn-close shadow-none" data-bs-dismiss="offcanvas"></button>
            </div>
            <div class="offcanvas-body">
                <div class="d-grid gap-3">
                    <button class="btn mobile-nav-btn shadow-none" :class="{ 'active-item': activeTab === 'today' }" @click="setActive('today')">
                        <span class="fs-1 mb-2">☀️</span>
                        <span class="fw-bold">Сегодня</span>
                    </button>
                    <button class="btn mobile-nav-btn shadow-none" :class="{ 'active-item': activeTab === 'upcoming' }" @click="setActive('upcoming')">
                        <span class="fs-1 mb-1">📅</span>
                        <span class="fw-bold">Предстоящие</span>
                    </button>
                </div>
            </div>
        </div>

        <!-- ОСНОВНОЙ КОНТЕНТ -->
        <main class="flex-grow-1 p-4 p-md-5 overflow-auto bg-white">
            <div class="container-narrow">
                <div class="d-flex justify-content-between align-items-center mb-4 main-header-row">
                    <h2 class="fw-bold m-0 text-dark">
                        {{ activeTab === 'today' ? '☀️ Сегодня' : '📅 На потом' }}
                    </h2>
                    <button @click="isAdding = !isAdding" class="btn btn-primary rounded-circle add-btn shadow-sm">
                        <span class="fs-3">{{ isAdding ? '×' : '+' }}</span>
                    </button>
                </div>

                <!-- ФОРМА ДОБАВЛЕНИЯ -->
                <div v-if="isAdding" class="add-task-form p-4 mb-4 rounded-4 border shadow-sm bg-white">
                    <input v-model="newTaskText" type="text" class="form-control mb-3 border-0 fs-5 ps-0 shadow-none bg-transparent" placeholder="Напишите задачу...">
                    <div class="d-flex justify-content-between align-items-center">
                        <input v-model="newTaskDate" type="date" class="form-control form-control-sm w-auto border-0 bg-light rounded-pill px-3 shadow-none">
                        <button @click="saveTask" class="btn btn-primary px-4 rounded-pill fw-bold" :disabled="!newTaskText.trim()">Добавить</button>
                    </div>
                </div>

                <!-- СПИСОК -->
                <div class="todo-list">
                    <div v-for="todo in filteredTodos" :key="todo.id" class="todo-item d-flex align-items-center p-3 mb-2 rounded-4 bg-white border">
                        <input type="checkbox" v-model="todo.done" class="form-check-input me-3 custom-check shadow-none">
                        <div class="flex-grow-1">
                            <div :class="{ 'text-decoration-line-through text-muted opacity-50': todo.done }" class="fs-5 text-dark">{{ todo.text }}</div>
                            <small class="text-muted opacity-75">{{ todo.date }}</small>
                        </div>
                        <button @click="deleteTask(todo.id)" class="btn btn-sm btn-delete text-danger shadow-none ms-2">
                            <span class="fs-3">&times;</span>
                        </button>
                    </div>
                    <div v-if="filteredTodos.length === 0" class="text-center mt-5 py-5 text-muted fs-5">Пока задач нет 🐾</div>
                </div>
            </div>
        </main>
    </div>
</template>

<style scoped>
.container-narrow { max-width: 700px; margin: 0 auto; }

/* КНОПКА КОТИК: ПОЛНОСТЬЮ ЧИСТАЯ */
.headerBtn {
    z-index: 2000;
    top: 15px;
    left: 15px;
    background: transparent !important;
    border: none !important;
    box-shadow: none !important;
    outline: none !important;
    padding: 0 !important;
}
.cat-trigger-img {
    max-width: 40px;
    display: block;
    filter: drop-shadow(0 0 0 transparent); /* Убираем любые системные тени */
}

/* ИСПРАВЛЕНИЕ: Отступ сверху на мобилках, чтобы котик не закрывал текст */
@media (max-width: 767.98px) {
    main {
        padding-top: 75px !important;
    }
}

/* Сайдбар */
.side-menu { width: 280px; border-right: 1px solid #f0f0f0; transition: all 0.3s ease; white-space: nowrap; overflow: hidden; }
.side-menu.collapsed { width: 75px; padding: 15px !important; }

.nav-link { color: #555; display: flex; align-items: center; padding: 12px 15px; border-radius: 12px; text-decoration: none; cursor: pointer; transition: 0.2s; }
.nav-link:hover { background-color: #f5f5f5; }
.active-item { background-color: #e8f0fe !important; color: #1a73e8 !important; font-weight: 600; }

/* Мобильное меню */
.mobile-nav-btn { display: flex; flex-direction: column; align-items: center; padding: 30px; border-radius: 20px; background: #f8f9fa; border: 1px solid transparent; transition: 0.2s; }
.mobile-nav-btn.active-item { border-color: #1a73e8; }

/* Карточки задач */
.todo-item { transition: 0.2s; border: 1px solid #f0f0f0; }
.todo-item:hover { transform: translateX(5px); border-color: #ddd; }

.btn-delete { opacity: 0; transition: 0.2s; border: none; background: none; }
.todo-item:hover .btn-delete { opacity: 1; }

.add-btn { width: 52px; height: 52px; display: flex; align-items: center; justify-content: center; line-height: 0; }
.custom-check { width: 22px; height: 22px; border-radius: 50%; cursor: pointer; border-color: #ccc; }
.icon { font-size: 1.3rem; min-width: 24px; text-align: center; }
</style>
