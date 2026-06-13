<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

// --- СОСТОЯНИЕ (DATA) ---
const isCollapsed = ref(false)
const isVisible = ref(true)
const isAdding = ref(false)

const newTaskText = ref('')
const newTaskDate = ref(new Date().toISOString().substr(0, 10))
const newTaskImage = ref(null)

const MAX_CHARS = 2000
const openedImage = ref(null)

// Хранилище задач
const todos = ref(JSON.parse(localStorage.getItem('my-todos')) || [])

// ДИНАМИЧЕСКИЕ КАТЕГОРИИ
const categories = ref(JSON.parse(localStorage.getItem('my-categories')) || [
    { id: 'today', name: 'Сегодня', icon: '☀️', deletable: false }
])

// Активная вкладка
const activeTab = ref(categories.value?.id || 'today')

// Состояние для создания новой категории
const newCategoryName = ref('')
const selectedIcon = ref('📁') 
const isCreatingCategory = ref(false)

// Список доступных эмодзи
const availableIcons = ['📁', '☀️', '📅', '💼', '🏠', '🎓', '🍕', '🛒', '🐾', '🎯', '🛠️', '❤️']

// Автосохранение задач и категорий
watch(todos, (newVal) => {
    localStorage.setItem('my-todos', JSON.stringify(newVal))
}, { deep: true })

watch(categories, (newVal) => {
    localStorage.setItem('my-categories', JSON.stringify(newVal))
}, { deep: true })

// --- ЛОГИКА ---
const filteredTodos = computed(() => {
    return todos.value.filter(todo => todo.category === activeTab.value)
})

const currentTabLabel = computed(() => {
    const current = categories.value.find(cat => cat.id === activeTab.value)
    return current ? `${current.icon} ${current.name}` : '📋 Задачи'
})

const addCategory = () => {
    if (!newCategoryName.value.trim()) return
    const id = 'cat_' + Date.now()
    categories.value.push({
        id: id,
        name: newCategoryName.value.trim(),
        icon: selectedIcon.value,
        deletable: true
    })
    activeTab.value = id
    newCategoryName.value = ''
    selectedIcon.value = '📁'
    isCreatingCategory.value = false
}

const deleteCategory = (id) => {
    if (confirm('Вы уверены, что хотите удалить эту вкладку и все задачи в ней?')) {
        todos.value = todos.value.filter(todo => todo.category !== id)
        categories.value = categories.value.filter(cat => cat.id !== id)
        activeTab.value = categories.value?.id || 'today'
    }
}

// Функция для сжатия изображения перед сохранением в localStorage
const compressImage = (base64Str, maxWidth = 800, quality = 0.7) => {
    return new Promise((resolve) => {
        const img = new Image()
        img.src = base64Str
        img.onload = () => {
            const canvas = document.createElement('canvas')
            let width = img.width
            let height = img.height

            if (width > maxWidth) {
                height = Math.round((height * maxWidth) / width)
                width = maxWidth
            }

            canvas.width = width
            canvas.height = height

            const ctx = canvas.getContext('2d')
            ctx.drawImage(img, 0, 0, width, height)

            const compressedBase64 = canvas.toDataURL('image/jpeg', quality)
            resolve(compressedBase64)
        }
    })
}

// Загрузка файла: читает данные и сжимает их на лету для телефонов
const handleImageUpload = (event) => {
    const file = event.target.files[0]
    if (!file) return

    const reader = new FileReader()
    reader.onload = async (e) => {
        const originalBase64 = e.target.result
        newTaskImage.value = await compressImage(originalBase64, 800, 0.7)
    }
    reader.readAsDataURL(file)
}

const setActive = (tabId) => {
    activeTab.value = tabId
    isAdding.value = false
    const offcanvasElement = document.getElementById('mobileSidebar')
    if (offcanvasElement && typeof bootstrap !== 'undefined') {
        const instance = bootstrap.Offcanvas.getInstance(offcanvasElement)
        if (instance) instance.hide()
    }
}

const saveTask = () => {
    if (!newTaskText.value.trim() || newTaskText.value.length > MAX_CHARS) return
    todos.value.push({
        id: Date.now(),
        text: newTaskText.value.trim(),
        date: newTaskDate.value,
        category: activeTab.value,
        done: false,
        image: newTaskImage.value
    })
    newTaskText.value = ''
    newTaskImage.value = null
    isAdding.value = false
}

const deleteTask = (id) => {
    if (openedImage.value && todos.value.find(t => t.id === id)?.image === openedImage.value) {
        openedImage.value = null
    }
    todos.value = todos.value.filter(t => t.id !== id)
}

const toggleDesktop = () => {
    isCollapsed.value = !isCollapsed.value
    if (isCollapsed.value) {
        isCreatingCategory.value = false
    }
}
const toggleCat = () => isVisible.value = false

const handleKeyDown = (event) => {
    if (event.key === 'Escape' || event.key === 'Esc') {
        openedImage.value = null
        isAdding.value = false
        isCreatingCategory.value = false
    }
}

onMounted(() => {
    window.addEventListener('keydown', handleKeyDown)
    const myOffcanvas = document.getElementById('mobileSidebar')
    if (myOffcanvas) {
        myOffcanvas.addEventListener('hidden.bs.offcanvas', () => isVisible.value = true)
    }
})

onUnmounted(() => {
    window.removeEventListener('keydown', handleKeyDown)
})
</script>







<template>
    <div class="d-flex w-100 min-vh-100 bg-white main-app-container">
        
        <!-- КНОПКА КОТИК -->
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
                <h4 v-if="!isCollapsed" class="m-0 fw-bold">Списки</h4>
                <button @click="toggleDesktop" class="btn btn-sm p-0 border-0 bg-transparent shadow-none">
                    <img src="/CatHeader.png" style="max-width: 30px;">
                </button>
            </div>
            
            <!-- Список вкладок ПК -->
            <nav class="nav flex-column flex-grow-1 overflow-auto app-categories-list">
                <div v-for="cat in categories" :key="cat.id" class="position-relative category-nav-wrapper mb-1">
                    <a class="nav-link pe-5" :class="{ 'active-item': activeTab === cat.id }" @click.prevent="setActive(cat.id)" href="#">
                        <span class="icon">{{ cat.icon }}</span>
                        <span v-if="!isCollapsed" class="ms-3 text-truncate d-inline-block style-cat-text">{{ cat.name }}</span>
                    </a>
                    <button v-if="cat.deletable && !isCollapsed" @click.stop="deleteCategory(cat.id)" class="btn btn-sm position-absolute end-0 top-50 translate-middle-y me-2 p-1 border-0 bg-transparent text-muted btn-del-cat">×</button>
                </div>
            </nav>

            <!-- Создание списка на ПК -->
            <div v-if="!isCollapsed" class="mt-auto pt-3 border-top block-create-cat">
                <div v-if="isCreatingCategory" class="mb-2 p-2 bg-white rounded-3 border">
                    <!-- Сетка выбора смайликов -->
                    <div class="d-flex flex-wrap gap-1 mb-2 justify-content-center emoji-picker-grid">
                        <button 
                            v-for="emoji in availableIcons" 
                            :key="emoji" 
                            @click="selectedIcon = emoji"
                            type="button"
                            class="btn btn-sm p-1 emoji-select-btn"
                            :class="{ 'selected-emoji': selectedIcon === emoji }"
                        >
                            {{ emoji }}
                        </button>
                    </div>
                    
                    <div class="input-group input-group-sm">
                        <span class="input-group-text bg-light border-end-0">{{ selectedIcon }}</span>
                        <input v-model="newCategoryName" @keyup.enter="addCategory" type="text" class="form-control shadow-none border-start-0" placeholder="Название...">
                        <button @click="addCategory" class="btn btn-primary" :disabled="!newCategoryName.trim()">+</button>
                    </div>
                </div>
                <button @click="isCreatingCategory = !isCreatingCategory" class="btn btn-sm btn-light w-100 rounded-3 text-muted fw-bold">
                    {{ isCreatingCategory ? 'Отмена' : '+ Новый список' }}
                </button>
            </div>
        </header>

        <!-- МОБИЛЬНОЕ МЕНЮ -->
        <div class="offcanvas offcanvas-start" id="mobileSidebar" tabindex="-1">
            <div class="offcanvas-header border-bottom">
                <h5 class="offcanvas-title fw-bold">Списки дел</h5>
                <button type="button" class="btn-close shadow-none" data-bs-dismiss="offcanvas"></button>
            </div>
            <div class="offcanvas-body d-flex flex-column justify-content-between h-100">
                <!-- Вкладки на мобилках -->
                <div class="d-grid gap-2 overflow-auto app-categories-list">
                    <div v-for="cat in categories" :key="cat.id" class="position-relative mobile-cat-row">
                        <button class="btn mobile-nav-btn shadow-none w-100 d-flex align-items-center p-3 text-start pe-5" :class="{ 'active-item': activeTab === cat.id }" @click="setActive(cat.id)">
                            <span class="fs-4 me-3">{{ cat.icon }}</span>
                            <span class="fw-bold text-truncate">{{ cat.name }}</span>
                        </button>
                        <button v-if="cat.deletable" @click.stop="deleteCategory(cat.id)" class="btn position-absolute end-0 top-50 translate-middle-y me-3 fs-4 text-muted border-0 bg-transparent p-1">×</button>
                    </div>
                </div>

                <!-- Создание списка на мобилках -->
                <div class="mt-4 pt-3 border-top bg-light p-3 rounded-4">
                    <div class="d-flex flex-wrap gap-2 mb-3 justify-content-center">
                        <button 
                            v-for="emoji in availableIcons" 
                            :key="emoji" 
                            @click="selectedIcon = emoji"
                            type="button"
                            class="btn p-2 fs-4 border-0 emoji-select-btn"
                            :class="{ 'selected-emoji': selectedIcon === emoji }"
                        >
                            {{ emoji }}
                        </button>
                    </div>
                    <div class="input-group">
                        <span class="input-group-text bg-white border-end-0 fs-4">{{ selectedIcon }}</span>
                        <input v-model="newCategoryName" @keyup.enter="addCategory" type="text" class="form-control shadow-none border-start-0 py-2" placeholder="Новый список...">
                        <button @click="addCategory" class="btn btn-primary px-3" :disabled="!newCategoryName.trim()">Создать</button>
                    </div>
                </div>
            </div>
        </div>
        <!-- ОСНОВНОЙ КОНТЕНТ -->
        <main class="flex-grow-1 p-4 p-md-5 overflow-auto bg-white">
            <div class="container-narrow">
                <div class="d-flex justify-content-between align-items-center mb-4 main-header-row">
                    <h2 class="fw-bold m-0 text-dark text-truncate pe-3">
                        {{ currentTabLabel }}
                    </h2>
                    <button @click="isAdding = !isAdding" class="btn btn-primary rounded-circle add-btn shadow-sm">
                        <span class="fs-3">{{ isAdding ? '×' : '+' }}</span>
                    </button>
                </div>

                <!-- ФОРМА ДОБАВЛЕНИЯ -->
                <div v-if="isAdding" class="add-task-form p-4 mb-4 rounded-4 border shadow-sm bg-white">
                    <textarea 
                        v-model="newTaskText" 
                        :maxlength="MAX_CHARS"
                        rows="3"
                        class="form-control mb-1 border-0 fs-5 ps-0 shadow-none bg-transparent" 
                        placeholder="Напишите задачу или развернутую заметку..."
                    ></textarea>
                    
                    <div class="text-end small text-muted mb-3">
                        {{ newTaskText.length }} / {{ MAX_CHARS }}
                    </div>

                    <div v-if="newTaskImage" class="mb-3 position-relative d-inline-block">
                        <img :src="newTaskImage" class="rounded-3" style="max-height: 100px; object-fit: cover;">
                        <button @click="newTaskImage = null" class="btn btn-danger btn-sm rounded-circle position-absolute top-0 end-0 m-1">×</button>
                    </div>

                    <div class="d-flex flex-wrap justify-content-between align-items-center gap-2">
                        <div class="d-flex align-items-center gap-2">
                            <input v-model="newTaskDate" type="date" class="form-control form-control-sm w-auto border-0 bg-light rounded-pill px-3 shadow-none">
                            <label class="btn btn-light btn-sm rounded-pill px-3 m-0 cursor-pointer">
                                📎 Фото
                                <input type="file" accept="image/*" @change="handleImageUpload" class="d-none">
                            </label>
                        </div>
                        <button @click="saveTask" class="btn btn-primary px-4 rounded-pill fw-bold btn-submit-task" :disabled="!newTaskText.trim() || newTaskText.length > MAX_CHARS">Добавить</button>
                    </div>
                </div>
                <!-- СПИСОК ЗАДАЧ -->
                <div class="todo-list">
                    <div v-for="todo in filteredTodos" :key="todo.id" class="todo-item d-flex align-items-center p-3 mb-2 rounded-4 bg-white border">
                        <input type="checkbox" v-model="todo.done" class="form-check-input me-3 custom-check shadow-none">
                        
                        <div class="flex-grow-1">
                            <div :class="{ 'text-decoration-line-through text-muted opacity-50': todo.done }" class="fs-5 text-dark text-break">
                                {{ todo.text }}
                            </div>
                            
                            <div v-if="todo.image" class="mt-2">
                                <img 
                                    :src="todo.image" 
                                    class="rounded-3 mw-100 cursor-pointer img-preview" 
                                    style="max-height: 150px; object-fit: cover;"
                                    @click="openedImage = todo.image"
                                >
                            </div>
                            <small class="text-muted opacity-75 d-block mt-1">{{ todo.date }}</small>
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

    <!-- ПОЛНОЭКРАННОЕ ОКНО ПРОСМОТРА ФОТО -->
    <div v-if="openedImage" class="image-modal-overlay" @click="openedImage = null">
        <button class="btn-close btn-close-white position-absolute top-0 end-0 m-4 shadow-none fs-4" title="Закрыть (Esc)"></button>
        <div class="image-modal-wrapper" @click.stop>
            <img :src="openedImage" class="image-modal-content">
        </div>
    </div>
</template>





<style scoped>
.container-narrow { max-width: 700px; margin: 0 auto; }

/* КНОПКА КОТИК */
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
}

@media (max-width: 767.98px) {
    main { padding-top: 75px !important; }
}

/* Сайдбар */
.side-menu { width: 280px; border-right: 1px solid #f0f0f0; transition: all 0.3s ease; white-space: nowrap; overflow: hidden; }

/* ИСПРАВЛЕНО: Полностью глушим появление скролла при сворачивании */
.side-menu.collapsed { 
    width: 75px; 
    padding: 15px !important; 
    overflow: hidden !important; 
}
.side-menu.collapsed .app-categories-list {
    overflow: hidden !important;
}

/* ИСПРАВЛЕНО: Прячем создание категорий в свернутом меню ПК */
.side-menu.collapsed .block-create-cat {
    display: none !important;
}

/* Навигация */
.nav-link { color: #555; display: flex; align-items: center; padding: 12px 15px; border-radius: 12px; text-decoration: none; }
.nav-link.active-item { background-color: #e8f0fe; color: #1a73e8; font-weight: bold; }
.style-cat-text { max-width: 150px; }

.btn-del-cat { opacity: 0; transition: opacity 0.2s ease; }
.category-nav-wrapper:hover .btn-del-cat { opacity: 1; }

/* Текстовое поле */
textarea.form-control {
    resize: none;
}
/* Стили выбора смайликов */
.emoji-picker-grid {
    max-width: 230px;
}
.emoji-select-btn {
    border-radius: 6px;
    transition: transform 0.1s ease, background 0.1s ease;
}
.emoji-select-btn:hover {
    background-color: #f0f0f0;
    transform: scale(1.15);
}
.emoji-select-btn.selected-emoji {
    background-color: #e8f0fe !important;
    box-shadow: 0 0 0 2px #1a73e8;
}

/* Мобильные категории */
.mobile-nav-btn {
    border: 1px solid #f0f0f0;
    background: #fdfdfd;
    border-radius: 12px;
    color: #333;
}
.mobile-nav-btn.active-item { background-color: #e8f0fe; color: #1a73e8; border-color: #1a73e8; }

.app-categories-list {
    max-height: 60vh;
}

/* Фиксация размеров круглой кнопки добавления */
.add-btn {
    width: 45px !important;
    height: 45px !important;
    min-width: 45px !important;
    min-height: 45px !important;
    display: inline-flex !important;
    align-items: center !important;
    justify-content: center !important;
    padding: 0 !important;
    flex-shrink: 0 !important;
}

.btn-submit-task {
    align-self: center;
}

.cursor-pointer {
    cursor: pointer;
}
.img-preview {
    transition: opacity 0.2s ease;
}
.img-preview:hover {
    opacity: 0.85;
}

/* ПОЛНОЭКРАННОЕ МОДАЛЬНОЕ ОКНО */
.image-modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(0, 0, 0, 0.85);
    z-index: 9999;
    display: flex;
    align-items: center;
    justify-content: center;
}

.image-modal-wrapper {
    max-width: 90%;
    max-height: 90%;
    display: flex;
    align-items: center;
    justify-content: center;
}

.image-modal-content {
    max-width: 100%;
    max-height: 90vh;
    object-fit: contain;
    border-radius: 12px;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.6);
}
</style>
