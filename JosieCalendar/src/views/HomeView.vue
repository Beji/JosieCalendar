<script setup>
import { ref, computed, watch } from 'vue'

const currentDate = ref(new Date())
const weekdays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']

// Get current year and month
const year = computed(() => currentDate.value.getFullYear())
const month = computed(() => currentDate.value.getMonth())

const monthName = computed(() => {
  return currentDate.value.toLocaleString('default', { month: 'long' })
})

// Number of days in the current month
const daysInMonth = computed(() => {
  return new Date(year.value, month.value + 1, 0).getDate()
})

// Number of days in the previous month
const daysInPrevMonth = computed(() => {
  return new Date(year.value, month.value, 0).getDate()
})

// Starting day offset (0 = Sunday, 1 = Monday, etc.)
const startDayOfWeek = computed(() => {
  return new Date(year.value, month.value, 1).getDay()
})

// Full grid of cells (including leading/trailing days from adjacent months
// so the first and last weeks are always complete rows)
const calendarCells = computed(() => {
  const cells = []

  let prevMonth = month.value - 1
  let prevYear = year.value
  if (prevMonth < 0) {
    prevMonth = 11
    prevYear -= 1
  }
  const leading = startDayOfWeek.value
  for (let i = 0; i < leading; i++) {
    const day = daysInPrevMonth.value - leading + 1 + i
    cells.push({ day, month: prevMonth, year: prevYear, inCurrentMonth: false })
  }

  for (let day = 1; day <= daysInMonth.value; day++) {
    cells.push({ day, month: month.value, year: year.value, inCurrentMonth: true })
  }

  let nextMonth = month.value + 1
  let nextYear = year.value
  if (nextMonth > 11) {
    nextMonth = 0
    nextYear += 1
  }
  const trailing = (7 - (cells.length % 7)) % 7
  for (let day = 1; day <= trailing; day++) {
    cells.push({ day, month: nextMonth, year: nextYear, inCurrentMonth: false })
  }

  return cells
})

// Navigation methods
const prevMonth = () => {
  currentDate.value = new Date(year.value, month.value - 1, 1)
}

const nextMonth = () => {
  currentDate.value = new Date(year.value, month.value + 1, 1)
}

// Check if a cell is today
const isToday = (cell) => {
  const today = new Date()
  return (
    cell.day === today.getDate() &&
    cell.month === today.getMonth() &&
    cell.year === today.getFullYear()
  )
}

// --- Events (persisted to localStorage so state survives app restarts) ---
const STORAGE_KEY = 'josiecalendar.events'

const loadEvents = () => {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    const parsed = raw ? JSON.parse(raw) : []
    return Array.isArray(parsed) ? parsed : []
  } catch {
    return []
  }
}

const events = ref(loadEvents())
let nextEventId = events.value.reduce((max, event) => Math.max(max, event.id), 0) + 1

watch(
  events,
  (value) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(value))
  },
  { deep: true },
)

const dateKey = (y, m, day) => {
  return `${y}-${String(m + 1).padStart(2, '0')}-${String(day).padStart(2, '0')}`
}

const eventsByDay = computed(() => {
  const map = {}
  for (const event of events.value) {
    if (!map[event.date]) map[event.date] = []
    map[event.date].push(event)
  }
  return map
})

const eventsForDay = (cell) => {
  return eventsByDay.value[dateKey(cell.year, cell.month, cell.day)] || []
}

// Events not yet placed on a day
const bankEvents = computed(() => {
  return events.value.filter((event) => !event.date)
})

// --- Create event form ---
const showForm = ref(false)
const formCell = ref(null)
const formIsBank = ref(false)
const formTitle = ref('')
const formStart = ref('')
const formEnd = ref('')

const formMonthName = computed(() => {
  if (!formCell.value) return ''
  return new Date(formCell.value.year, formCell.value.month).toLocaleString('default', {
    month: 'long',
  })
})

const openCreateForm = (cell) => {
  formCell.value = cell
  formIsBank.value = false
  formTitle.value = ''
  formStart.value = ''
  formEnd.value = ''
  showForm.value = true
}

const openBankForm = () => {
  formCell.value = null
  formIsBank.value = true
  formTitle.value = ''
  formStart.value = ''
  formEnd.value = ''
  showForm.value = true
}

const closeForm = () => {
  showForm.value = false
  formCell.value = null
  formIsBank.value = false
}

const saveEvent = () => {
  if (!formTitle.value.trim()) return
  if (!formIsBank.value && !formCell.value) return
  events.value.push({
    id: nextEventId++,
    title: formTitle.value.trim(),
    date: formIsBank.value ? null : dateKey(formCell.value.year, formCell.value.month, formCell.value.day),
    start: formStart.value,
    end: formEnd.value,
  })
  closeForm()
}

// --- Drag and drop ---
const onEventDragStart = (event, domEvent) => {
  domEvent.dataTransfer.effectAllowed = 'move'
  domEvent.dataTransfer.setData('text/plain', String(event.id))
}

const onDayDrop = (cell, domEvent) => {
  domEvent.preventDefault()
  const id = Number(domEvent.dataTransfer.getData('text/plain'))
  const event = events.value.find((e) => e.id === id)
  if (event) {
    event.date = dateKey(cell.year, cell.month, cell.day)
  }
}

const onBankDrop = (domEvent) => {
  domEvent.preventDefault()
  const id = Number(domEvent.dataTransfer.getData('text/plain'))
  const event = events.value.find((e) => e.id === id)
  if (event) {
    event.date = null
  }
}

const onDayDragOver = (domEvent) => {
  domEvent.preventDefault()
  domEvent.dataTransfer.dropEffect = 'move'
}
</script>

<template>
  <div class="calendar">
    <!-- Header -->
    <div class="header">
      <button @click="prevMonth">&lt;</button>
      <h2>{{ monthName }} {{ year }}</h2>
      <button @click="nextMonth">&gt;</button>
    </div>

    <div class="body">
      <!-- Event Bank -->
      <div class="bank" @dragover="onDayDragOver" @drop="onBankDrop">
        <div class="bank-header">
          <h3>Event Bank</h3>
          <button class="add-event-btn" @click="openBankForm">+</button>
        </div>
        <div class="bank-events">
          <div
            v-for="event in bankEvents"
            :key="event.id"
            class="event-item"
            draggable="true"
            @dragstart="onEventDragStart(event, $event)"
          >
            <div class="event-title">{{ event.title }}</div>
            <div v-if="event.start || event.end" class="event-time">
              {{ event.start }}<span v-if="event.start && event.end"> - </span>{{ event.end }}
            </div>
          </div>
        </div>
      </div>

      <!-- Main Calendar -->
      <div class="main">
        <!-- Weekday Labels -->
        <div class="weekdays">
          <div v-for="day in weekdays" :key="day" class="weekday">{{ day }}</div>
        </div>

        <!-- Date Grid -->
        <div class="grid">
          <div
            v-for="cell in calendarCells"
            :key="`${cell.year}-${cell.month}-${cell.day}`"
            class="day"
            :class="{ today: isToday(cell), outside: !cell.inCurrentMonth }"
            @dragover="onDayDragOver"
            @drop="onDayDrop(cell, $event)"
          >
            <div class="day-number">{{ cell.day }}</div>

            <div class="day-events">
              <div
                v-for="event in eventsForDay(cell)"
                :key="event.id"
                class="event-item"
                draggable="true"
                @dragstart="onEventDragStart(event, $event)"
              >
                <div class="event-title">{{ event.title }}</div>
                <div v-if="event.start || event.end" class="event-time">
                  {{ event.start }}<span v-if="event.start && event.end"> - </span>{{ event.end }}
                </div>
              </div>
            </div>

            <button class="add-event-btn" @click="openCreateForm(cell)">+</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Create Event Modal -->
    <div v-if="showForm" class="modal-overlay" @click.self="closeForm">
      <div class="modal">
        <h3 v-if="formIsBank">New Event</h3>
        <h3 v-else>New Event &mdash; {{ formMonthName }} {{ formCell?.day }}, {{ formCell?.year }}</h3>
        <input
          type="text"
          v-model="formTitle"
          class="form-control"
          placeholder="Enter event title"
          @keyup.enter="saveEvent"
        />
        <input type="time" v-model="formStart" class="form-control" placeholder="Start time" />
        <input type="time" v-model="formEnd" class="form-control" placeholder="End time" />
        <div class="modal-actions">
          <button @click="closeForm">Cancel</button>
          <button @click="saveEvent">Create Event</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.calendar {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding: 1rem;
  font-family: sans-serif;
}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}
.body {
  flex: 1;
  display: flex;
  gap: 1rem;
  min-height: 0;
}
.bank {
  width: 220px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  background: #eef2f7;
  border-radius: 4px;
  padding: 8px;
  overflow-y: auto;
}
.bank-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.bank-header h3 {
  font-size: 0.9rem;
  margin: 0;
}
.bank-events {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.main {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
}
.weekdays, .grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4px;
}
.grid {
  flex: 1;
  grid-auto-rows: 1fr;
}
.weekday {
  text-align: center;
  font-weight: bold;
  padding: 8px 0;
}
.day {
  min-height: 0;
  display: flex;
  flex-direction: column;
  padding: 4px;
  background: #f4f4f4;
  border-radius: 4px;
  position: relative;
}
.day.today {
  background: #d6e9ff;
  font-weight: bold;
}
.day.outside {
  opacity: 0.5;
}
.day-number {
  align-self: flex-end;
  font-size: 0.8rem;
}
.day-events {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 2px;
  margin-top: 2px;
}
.event-item {
  background: #3182ce;
  color: white;
  border-radius: 3px;
  padding: 2px 4px;
  font-size: 0.7rem;
  cursor: grab;
}
.event-title {
  font-weight: bold;
}
.event-time {
  font-size: 0.65rem;
  opacity: 0.85;
}
.add-event-btn {
  align-self: flex-end;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 0.9rem;
  line-height: 1;
  padding: 2px 4px;
}
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}
.modal {
  background: white;
  color: #222;
  padding: 1.5rem;
  border-radius: 8px;
  width: 280px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.form-control {
  padding: 6px 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
}
.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 0.5rem;
}
</style>