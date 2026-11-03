<template>
  <div class="app">
    <aside class="sidebar">
      <div class="sidebar-brand">
        <div class="brand-mark" aria-hidden="true">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor"
               stroke-width="3.5" stroke-linecap="square" stroke-linejoin="miter">
            <path d="M7 4l10 8-10 8"/>
          </svg>
        </div>
        <div class="brand-text">
          <h1>{{ t('nav.companyName') }}</h1>
          <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
        </div>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          :title="item.label"
          :class="['nav-item', { active: $route.path === item.path }]"
        >
          <svg class="nav-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor"
               stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path v-for="(d, i) in item.icon" :key="i" :d="d"/>
          </svg>
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <span class="env-badge">Demo</span>
      </div>
    </aside>

    <div class="app-body">
      <header class="topbar">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </header>

      <FilterBar />

      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar entries. Each icon is a list of SVG path `d` strings so the
    // markup stays a single reusable <svg> in the template.
    const navItems = computed(() => [
      {
        path: '/',
        label: t('nav.overview'),
        icon: ['M3 3h7v7H3z', 'M14 3h7v4h-7z', 'M14 11h7v10h-7z', 'M3 14h7v7H3z']
      },
      {
        path: '/inventory',
        label: t('nav.inventory'),
        icon: ['M21 8l-9-5-9 5 9 5 9-5z', 'M3 12l9 5 9-5', 'M3 16l9 5 9-5']
      },
      {
        path: '/orders',
        label: t('nav.orders'),
        icon: ['M9 3h6v4H9z', 'M5 5h2M17 5h2M5 5v16h14V5', 'M9 11h6', 'M9 15h6']
      },
      {
        path: '/spending',
        label: t('nav.finance'),
        icon: ['M12 2v20', 'M17 6H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6']
      },
      {
        path: '/demand',
        label: t('nav.demandForecast'),
        icon: ['M3 17l6-6 4 4 7-7', 'M17 7h4v4']
      },
      {
        path: '/restocking',
        label: t('nav.restocking'),
        icon: ['M21 2v6h-6', 'M3 12a9 9 0 0 1 15-6.7L21 8', 'M3 22v-6h6', 'M21 12a9 9 0 0 1-15 6.7L3 16']
      },
      {
        path: '/reports',
        label: 'Reports',
        icon: ['M3 3v18h18', 'M7 15V9', 'M12 15V5', 'M17 15v-4']
      }
    ])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/* ==========================================================================
   Design tokens - Accenture brand language
   Palette, type and shape taken from accenture.com's production stylesheets:
   #a100ff is the signature purple, grays are warm (not slate), and
   border-radius is 0 almost everywhere - the brand reads square, not rounded.
   ========================================================================== */
:root {
  --sidebar-width: 248px;
  --sidebar-width-collapsed: 76px;
  --topbar-height: 60px;

  /* Purple ramp */
  --purple: #a100ff;
  --purple-deep: #7500c0;
  --purple-dark: #460073;
  --purple-darkest: #39005e;
  --purple-light: #be82ff;
  --purple-pale: #dcafff;

  /* Neutrals - warm, per brand */
  --black: #000000;
  --ink: #202020;
  --ink-2: #2b2b2b;
  --gray: #616160;
  --gray-warm: #a2a2a0;
  --line: #e3e3df;
  --line-soft: #f1f1ef;
  --canvas: #f1f1ef;
  --surface: #ffffff;

  /* Accent / status - the brand palette has no green, so purple carries
     the positive signal and the warm ramp carries warning through danger. */
  --coral: #ff5757;
  --orange: #ff7800;
  --amber: #feb149;
  --pink: #ff50a0;

  /* Square by default */
  --radius: 0;
  --radius-sm: 0;

  --shadow-sm: 0 1px 2px 0 rgba(32, 32, 32, 0.06);
  --shadow-md: 0 6px 20px -6px rgba(32, 32, 32, 0.16);

  /* Signature easing from the brand's button transition */
  --ease-brand: cubic-bezier(0.85, 0, 0, 1);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  /* Graphik is the brand face; Inter is the closest widely-available stand-in */
  font-family: 'Graphik', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  background: var(--canvas);
  color: var(--ink);
  letter-spacing: -0.005em;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Brand focus treatment: purple ring, generously offset */
:focus-visible {
  outline: 2px solid var(--purple);
  outline-offset: 4px;
}

/* ==========================================================================
   Shell - fixed left sidebar, scrolling content column
   ========================================================================== */
.app {
  min-height: 100vh;
}

.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  width: var(--sidebar-width);
  background: var(--black);
  display: flex;
  flex-direction: column;
  z-index: 200;
}

.app-body {
  margin-left: var(--sidebar-width);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* --- Brand --- */
.sidebar-brand {
  display: flex;
  align-items: center;
  gap: 0.875rem;
  padding: 1.5rem 1.5rem 1.75rem;
}

.brand-mark {
  flex-shrink: 0;
  width: 26px;
  height: 26px;
  color: var(--purple);
  display: flex;
  align-items: center;
  justify-content: center;
}

.brand-mark svg {
  width: 100%;
  height: 100%;
}

.brand-text {
  min-width: 0;
}

.brand-text h1 {
  font-size: 0.938rem;
  font-weight: 500;
  color: #fff;
  letter-spacing: -0.02em;
  /* Wrap rather than truncate - the product name must stay readable */
  line-height: 1.25;
}

.brand-subtitle {
  display: block;
  margin-top: 0.125rem;
  font-size: 0.688rem;
  color: var(--gray-warm);
  letter-spacing: -0.005em;
  line-height: 1.3;
}

/* --- Vertical nav --- */
.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 0;
  overflow-y: auto;
}

.nav-item {
  position: relative;
  display: flex;
  align-items: center;
  gap: 0.875rem;
  padding: 0.75rem 1.5rem;
  color: var(--gray-warm);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 400;
  letter-spacing: -0.005em;
  transition: color 0.2s var(--ease-brand), background-color 0.2s var(--ease-brand);
}

/* Purple wipe on hover, sweeping in from the left */
.nav-item::before {
  content: '';
  position: absolute;
  inset: 0;
  background: var(--purple);
  transform: scaleX(0);
  transform-origin: left center;
  transition: transform 0.55s var(--ease-brand);
}

.nav-item:hover::before {
  transform: scaleX(1);
}

.nav-item:hover {
  color: #fff;
}

.nav-item.active {
  color: #fff;
  font-weight: 500;
}

.nav-item.active::before {
  transform: scaleX(1);
  background: var(--purple);
}

.nav-icon,
.nav-label {
  position: relative;
  z-index: 1;
}

.nav-icon {
  flex-shrink: 0;
  width: 18px;
  height: 18px;
}

.nav-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-footer {
  padding: 1.25rem 1.5rem;
}

.env-badge {
  display: inline-block;
  padding: 0.25rem 0.5rem;
  background: transparent;
  border: 1px solid var(--gray);
  color: var(--gray-warm);
  font-size: 0.688rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.02em;
}

/* --- Top utility bar --- */
.topbar {
  height: var(--topbar-height);
  flex-shrink: 0;
  background: var(--surface);
  border-bottom: 1px solid var(--line);
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1rem;
  padding: 0 1.75rem;
  position: sticky;
  top: 0;
  z-index: 100;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: 1600px;
  padding: 2rem 1.75rem;
}

/* ==========================================================================
   Page furniture
   ========================================================================== */
.page-header {
  margin-bottom: 2rem;
}

.page-header h2 {
  font-size: 2rem;
  font-weight: 400;
  color: var(--ink);
  margin-bottom: 0.5rem;
  /* Brand headings run large, light and tightly tracked */
  letter-spacing: -0.03em;
  line-height: 1.15;
}

.page-header p {
  color: var(--gray);
  font-size: 0.938rem;
  letter-spacing: -0.005em;
}

/* --- Stat cards --- */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(230px, 1fr));
  gap: 1px;
  margin-bottom: 1.5rem;
  background: var(--line);
  border: 1px solid var(--line);
}

/* Hairline grid: cards butt together, separated by the container background */
.stat-card {
  background: var(--surface);
  padding: 1.25rem 1.375rem;
  border-radius: var(--radius);
  border: none;
  position: relative;
  transition: background-color 0.2s var(--ease-brand);
}

.stat-card::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  height: 3px;
  background: var(--purple);
  transform: scaleX(0);
  transform-origin: left center;
  transition: transform 0.45s var(--ease-brand);
}

.stat-card:hover::after {
  transform: scaleX(1);
}

.stat-label {
  color: var(--gray);
  font-size: 0.688rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.02em;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 1.875rem;
  font-weight: 400;
  color: var(--ink);
  letter-spacing: -0.03em;
  line-height: 1.1;
}

.stat-card.warning .stat-value {
  color: var(--orange);
}

.stat-card.success .stat-value {
  color: var(--purple);
}

.stat-card.danger .stat-value {
  color: var(--coral);
}

.stat-card.info .stat-value {
  color: var(--purple-deep);
}

/* --- Cards --- */
.card {
  background: var(--surface);
  border-radius: var(--radius);
  padding: 1.5rem;
  border: 1px solid var(--line);
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1.25rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid var(--line);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 400;
  color: var(--ink);
  letter-spacing: -0.02em;
}

/* --- Tables --- */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--surface);
  border-top: none;
  border-bottom: 1px solid var(--ink);
}

th {
  text-align: left;
  padding: 0.625rem 0.75rem;
  font-weight: 500;
  color: var(--ink);
  font-size: 0.688rem;
  text-transform: uppercase;
  letter-spacing: 0.02em;
  white-space: nowrap;
}

td {
  padding: 0.75rem;
  border-top: 1px solid var(--line);
  color: var(--ink-2);
  font-size: 0.875rem;
  letter-spacing: -0.005em;
}

tbody tr {
  transition: background-color 0.15s var(--ease-brand);
}

tbody tr:hover {
  background: var(--line-soft);
}

/* --- Badges: square, quiet, brand-tinted --- */
.badge {
  display: inline-block;
  padding: 0.25rem 0.563rem;
  border-radius: var(--radius);
  font-size: 0.688rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.02em;
  white-space: nowrap;
  border: 1px solid transparent;
}

.badge.success {
  background: #f4e8ff;
  color: var(--purple-dark);
  border-color: var(--purple-pale);
}

.badge.warning {
  background: #fff4e0;
  color: #8a4b00;
  border-color: var(--amber);
}

.badge.danger {
  background: #ffecec;
  color: #a81f1f;
  border-color: var(--coral);
}

.badge.info {
  background: #f4e8ff;
  color: var(--purple-dark);
  border-color: var(--purple-light);
}

.badge.increasing {
  background: #f4e8ff;
  color: var(--purple-dark);
  border-color: var(--purple-pale);
}

.badge.decreasing {
  background: #ffecec;
  color: #a81f1f;
  border-color: var(--coral);
}

.badge.stable {
  background: var(--line-soft);
  color: var(--gray);
  border-color: var(--line);
}

.badge.high {
  background: #ffecec;
  color: #a81f1f;
  border-color: var(--coral);
}

.badge.medium {
  background: #fff4e0;
  color: #8a4b00;
  border-color: var(--amber);
}

.badge.low {
  background: var(--line-soft);
  color: var(--gray);
  border-color: var(--line);
}

/* --- States --- */
.loading {
  text-align: center;
  padding: 3rem;
  color: var(--gray);
  font-size: 0.938rem;
}

.error {
  background: #ffecec;
  border: 1px solid var(--coral);
  border-radius: var(--radius);
  color: #a81f1f;
  padding: 0.875rem 1rem;
  margin: 1rem 0;
  font-size: 0.875rem;
}

/* ==========================================================================
   Brand button - the signature purple wipe on hover
   ========================================================================== */
.btn-brand {
  display: inline-flex;
  align-items: center;
  gap: 0.625rem;
  background-color: var(--purple);
  background-image: linear-gradient(90deg, var(--purple-deep) 50%, var(--purple) 0);
  background-position-x: 100%;
  background-size: 200% 200%;
  color: #fff;
  border: none;
  border-radius: var(--radius);
  padding: 0.875rem 1.75rem;
  font-family: inherit;
  font-size: 0.938rem;
  font-weight: 500;
  letter-spacing: -0.005em;
  cursor: pointer;
  transition: background-position-x 550ms var(--ease-brand);
}

.btn-brand:hover:not(:disabled) {
  background-position-x: 0%;
}

.btn-brand:disabled {
  background-color: var(--line);
  background-image: none;
  color: var(--gray-warm);
  cursor: not-allowed;
}

/* ==========================================================================
   Responsive - collapse the sidebar to icons, then to a top strip
   ========================================================================== */
@media (max-width: 1100px) {
  .sidebar {
    width: var(--sidebar-width-collapsed);
    align-items: center;
  }

  .app-body {
    margin-left: var(--sidebar-width-collapsed);
  }

  .brand-text,
  .nav-label,
  .sidebar-footer {
    display: none;
  }

  .sidebar-brand {
    padding: 1.5rem 0 1.75rem;
  }

  .sidebar-nav {
    width: 100%;
  }

  .nav-item {
    justify-content: center;
    padding: 0.875rem 0;
  }
}

@media (max-width: 720px) {
  .sidebar {
    position: static;
    width: 100%;
    flex-direction: row;
    align-items: center;
    gap: 0.5rem;
    padding: 0 1rem;
  }

  .app-body {
    margin-left: 0;
  }

  .sidebar-brand {
    padding: 0.75rem 0;
  }

  .sidebar-nav {
    flex-direction: row;
    overflow-x: auto;
  }

  .nav-item {
    padding: 0.875rem 1rem;
  }

  .topbar {
    padding: 0 1rem;
  }

  .main-content {
    padding: 1.25rem 1rem;
  }

  .page-header h2 {
    font-size: 1.625rem;
  }
}
</style>
