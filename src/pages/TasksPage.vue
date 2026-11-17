
<script setup>
import { computed, nextTick, ref } from 'vue';
import AppTabbar from '../components/AppTabbar.vue';
import TaskCard from '../components/TaskCard.vue';
import TaskForm from '../components/TaskForm.vue';
import TaskListControls from '../components/TaskListControls.vue';

import {
  taskState,
  addTask,
  updateTask,
  deleteTask,
  setTaskStatus,
  setTaskPriority,
} from '../store/tasks';

import '../css/task-components.css';
import '../css/task-form.css';
import '../css/task-list.css';

// Пошук, фільтри та спосіб сортування.
const searchText = ref('');
const selectedStatus = ref('all');
const selectedPriority = ref('all');
const sortOrder = ref('newest');

// Керування формою створення та редагування.
const showForm = ref(false);
const editingId = ref(null);
const formAnchor = ref(null);
const feedback = ref('');

// Знаходимо завдання, яке зараз редагується.
const editingTask = computed(() =>
  taskState.tasks.find(
    (task) => task.id === editingId.value
  ) ?? null
);

// Формуємо список з урахуванням пошуку, фільтрів та сортування.
const visibleTasks = computed(() => {
  const term = searchText.value.trim().toLocaleLowerCase('uk-UA');

  const priorityWeight = {
    high: 3,
    medium: 2,
    low: 1,
  };

  // Шукаємо текст у назві та описі завдання.
  const result = taskState.tasks.filter((task) => {
    const taskText =
      `${task.title} ${task.description ?? ''}`
        .toLocaleLowerCase('uk-UA');

    const matchesText =
      !term || taskText.includes(term);

    const matchesStatus =
      selectedStatus.value === 'all' ||
      task.status === selectedStatus.value;

    const matchesPriority =
      selectedPriority.value === 'all' ||
      task.priority === selectedPriority.value;

    return matchesText && matchesStatus && matchesPriority;
  });

  // Сортуємо результат, не змінюючи вихідний список завдань.
  return result.sort((a, b) => {
    if (sortOrder.value === 'title') {
      return a.title.localeCompare(b.title, 'uk-UA') || a.id - b.id;
    }

    if (sortOrder.value === 'priority') {
      return priorityWeight[b.priority] -
        priorityWeight[a.priority] || b.id - a.id;
    }

    if (sortOrder.value === 'dueDate') {
      const first = a.dueDate || '9999-12-31';
      const second = b.dueDate || '9999-12-31';

      return first.localeCompare(second) || b.id - a.id;
    }

    // За замовчуванням спочатку показуємо нові завдання.
    return b.id - a.id;
  });
});

// Повертаємо всі параметри пошуку до початкових значень.
function resetFilters() {
  searchText.value = '';
  selectedStatus.value = 'all';
  selectedPriority.value = 'all';
  sortOrder.value = 'newest';
}

// Після відкриття форми прокручуємо сторінку до неї.
async function scrollToForm() {
  await nextTick();

  formAnchor.value?.scrollIntoView({
    behavior: 'smooth',
    block: 'start',
  });
}

// Відкриваємо форму створення нового завдання.
async function startCreate() {
  editingId.value = null;
  showForm.value = true;
  feedback.value = '';

  await scrollToForm();
}

// Відкриваємо форму редагування вибраного завдання.
async function startEdit(id) {
  editingId.value = id;
  showForm.value = true;
  feedback.value = '';

  await scrollToForm();
}

// Закриваємо форму без збереження змін.
function cancelForm() {
  showForm.value = false;
  editingId.value = null;
}

// Створюємо нове завдання або оновлюємо наявне.
function saveTask(values) {
  const isEditing = values.id !== null;

  const saved = isEditing
    ? updateTask(values)
    : addTask(values);

  if (!saved) {
    feedback.value =
      'Не вдалося зберегти завдання. Перевірте введені дані.';
    return;
  }

  feedback.value = isEditing
    ? `Завдання «${saved.title}» оновлено.`
    : `Завдання «${saved.title}» створено.`;

  cancelForm();

  // Після збереження показуємо повний список.
  resetFilters();
}

// Змінюємо статус завдання.
function handleStatusChange(payload) {
  setTaskStatus(payload);
}

// Змінюємо пріоритет завдання.
function handlePriorityChange(payload) {
  setTaskPriority(payload);
}

// Перед видаленням обов'язково запитуємо підтвердження.
function handleDelete(id) {
  const task = taskState.tasks.find(
    (item) => item.id === Number(id)
  );

  if (!task) return;

  const confirmed = window.confirm(
    `Видалити завдання «${task.title}»?`
  );

  if (!confirmed) return;

  const deleted = deleteTask(id);

  if (deleted) {
    feedback.value = `Завдання «${task.title}» видалено.`;

    if (editingId.value === id) {
      cancelForm();
    }
  }
}
</script>

<template>
  <f7-page name="tasks" class="tf3-page">
    <f7-navbar title="Мої завдання" />

    <div class="tf3-container">
      <section class="tf3-hero">
        <div class="tf3-eyebrow">
          TASKFLOW / TASK MANAGER
        </div>

        <h1>Керування завданнями</h1>

        <p>
          Створюйте, редагуйте, знаходьте, сортуйте
          та видаляйте завдання.
        </p>

        <button
          type="button"
          class="tf5-btn tf5-btn--primary tf5-new"
          @click="startCreate"
        >
          + Додати завдання
        </button>
      </section>

      <p
        v-if="feedback"
        class="tf4-feedback"
        role="status"
      >
        {{ feedback }}
      </p>

      <section
        v-if="showForm"
        ref="formAnchor"
        class="tf4-form-section"
      >
        <TaskForm
          :key="editingId ?? 'new'"
          :task="editingTask"
          @save="saveTask"
          @cancel="cancelForm"
        />
      </section>

      <TaskListControls
        v-model:search="searchText"
        v-model:status="selectedStatus"
        v-model:priority="selectedPriority"
        v-model:sort="sortOrder"
        @reset="resetFilters"
      />

      <section class="tf3-section">
        <div class="tf3-section-heading">
          <div>
            <div class="tf3-eyebrow">
              TASK LIST
            </div>

            <h2>Список завдань</h2>
          </div>

          <div class="tf3-counter">
            Показано: {{ visibleTasks.length }}
            із {{ taskState.tasks.length }}
          </div>
        </div>

        <div
          v-if="visibleTasks.length"
          class="tf3-task-grid"
        >
          <TaskCard
            v-for="task in visibleTasks"
            :key="task.id"
            :task="task"
            :editable="true"
            @status-change="handleStatusChange"
            @priority-change="handlePriorityChange"
            @edit="startEdit"
            @delete="handleDelete"
          />
        </div>

        <div v-else class="tf5-empty">
          <p>
            За поточними умовами завдань не знайдено.
          </p>

          <button
            type="button"
            class="tf5-btn"
            @click="resetFilters"
          >
            Показати всі завдання
          </button>
        </div>
      </section>
    </div>

    <AppTabbar active="tasks" />
  </f7-page>
</template>
