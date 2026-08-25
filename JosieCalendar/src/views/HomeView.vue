<script setup>
import { ref, computed, watch, nextTick } from 'vue'

const currentDate = ref(new Date())
const weekdays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']

// --- View mode (month grid vs. weekly time-grid) ---
const viewMode = ref('month')
const weekBodyEl = ref(null)

const setViewMode = async (mode) => {
  viewMode.value = mode
  if (mode === 'week') {
    await nextTick()
    if (weekBodyEl.value) {
      weekBodyEl.value.scrollTop = HOUR_HEIGHT * 7
    }
  }
}

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

// --- Week view ---
const HOUR_HEIGHT = 48 // px per hour in the weekly time grid

const weekStart = computed(() => {
  const d = new Date(currentDate.value)
  d.setDate(d.getDate() - d.getDay())
  d.setHours(0, 0, 0, 0)
  return d
})

const weekDays = computed(() => {
  const days = []
  for (let i = 0; i < 7; i++) {
    const d = new Date(weekStart.value)
    d.setDate(d.getDate() + i)
    days.push({ day: d.getDate(), month: d.getMonth(), year: d.getFullYear() })
  }
  return days
})

const weekRangeLabel = computed(() => {
  const start = weekStart.value
  const end = new Date(weekStart.value)
  end.setDate(end.getDate() + 6)
  const startStr = start.toLocaleString('default', { month: 'short', day: 'numeric' })
  const endStr = end.toLocaleString('default', { month: 'short', day: 'numeric', year: 'numeric' })
  return `${startStr} – ${endStr}`
})

const prevWeek = () => {
  const d = new Date(currentDate.value)
  d.setDate(d.getDate() - 7)
  currentDate.value = d
}

const nextWeek = () => {
  const d = new Date(currentDate.value)
  d.setDate(d.getDate() + 7)
  currentDate.value = d
}

const goPrev = () => (viewMode.value === 'week' ? prevWeek() : prevMonth())
const goNext = () => (viewMode.value === 'week' ? nextWeek() : nextMonth())
const headerLabel = computed(() =>
  viewMode.value === 'week' ? weekRangeLabel.value : `${monthName.value} ${year.value}`,
)

// --- Time helpers (24h "HH:MM" storage <-> 12h AM/PM display) ---
const formatTimeLabel = (timeStr) => {
  if (!timeStr) return ''
  const [h, m] = timeStr.split(':').map(Number)
  const period = h >= 12 ? 'PM' : 'AM'
  const h12 = h % 12 === 0 ? 12 : h % 12
  return `${h12}:${String(m).padStart(2, '0')} ${period}`
}

const hourLabel = (hour) => {
  const period = hour >= 12 ? 'PM' : 'AM'
  const h12 = hour % 12 === 0 ? 12 : hour % 12
  return `${h12} ${period}`
}

const timeToMinutes = (timeStr) => {
  if (!timeStr) return null
  const [h, m] = timeStr.split(':').map(Number)
  return h * 60 + m
}

const minutesToTime = (mins) => {
  const clamped = Math.max(0, Math.min(24 * 60 - 1, Math.round(mins)))
  const h = Math.floor(clamped / 60)
  const m = clamped % 60
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`
}

const timedEventsForDay = (cell) => eventsForDay(cell).filter((e) => e.start)
const allDayEventsForDay = (cell) => eventsForDay(cell).filter((e) => !e.start)

const eventBlockStyle = (event) => {
  const startMin = timeToMinutes(event.start) ?? 0
  const endMin = event.end ? timeToMinutes(event.end) : startMin + 30
  const top = (startMin / 60) * HOUR_HEIGHT
  const height = Math.max(((endMin - startMin) / 60) * HOUR_HEIGHT, 18)
  return {
    top: `${top}px`,
    height: `${height}px`,
    background: event.color || DEFAULT_COLOR,
  }
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

// --- Create / edit event form ---
const DEFAULT_COLOR = '#3182ce'

const showForm = ref(false)
const formCell = ref(null)
const formIsBank = ref(false)
const formEditingId = ref(null)
const formTitle = ref('')
const formColor = ref(DEFAULT_COLOR)
const formStart = ref('')
const formEnd = ref('')
const formQuantity = ref(1)

const formMonthName = computed(() => {
  if (!formCell.value) return ''
  return new Date(formCell.value.year, formCell.value.month).toLocaleString('default', {
    month: 'long',
  })
})

const resetFormFields = () => {
  formTitle.value = ''
  formColor.value = DEFAULT_COLOR
  formStart.value = ''
  formEnd.value = ''
  formQuantity.value = 1
}

const openCreateForm = (cell) => {
  formCell.value = cell
  formIsBank.value = false
  formEditingId.value = null
  resetFormFields()
  showForm.value = true
}

const openBankForm = () => {
  formCell.value = null
  formIsBank.value = true
  formEditingId.value = null
  resetFormFields()
  showForm.value = true
}

const openEditForm = (event) => {
  formCell.value = null
  formIsBank.value = !event.date
  formEditingId.value = event.id
  formTitle.value = event.title
  formColor.value = event.color || DEFAULT_COLOR
  formStart.value = event.start
  formEnd.value = event.end
  formQuantity.value = 1
  showForm.value = true
}

const closeForm = () => {
  showForm.value = false
  formCell.value = null
  formIsBank.value = false
  formEditingId.value = null
}

const saveEvent = () => {
  if (!formTitle.value.trim()) return

  if (formEditingId.value !== null) {
    const event = events.value.find((e) => e.id === formEditingId.value)
    if (event) {
      event.title = formTitle.value.trim()
      event.color = formColor.value
      event.start = formStart.value
      event.end = formEnd.value
    }
    closeForm()
    return
  }

  if (!formIsBank.value && !formCell.value) return

  const date = formIsBank.value
    ? null
    : dateKey(formCell.value.year, formCell.value.month, formCell.value.day)
  const count = formIsBank.value ? Math.min(Math.max(Math.round(formQuantity.value) || 1, 1), 20) : 1

  for (let i = 0; i < count; i++) {
    events.value.push({
      id: nextEventId++,
      title: formTitle.value.trim(),
      date,
      start: formStart.value,
      end: formEnd.value,
      color: formColor.value,
    })
  }
  closeForm()
}

const deleteEvent = (id) => {
  events.value = events.value.filter((e) => e.id !== id)
}

// --- Right-click context menu ---
const contextMenu = ref({ visible: false, x: 0, y: 0, eventId: null })

const openContextMenu = (event, domEvent) => {
  domEvent.preventDefault()
  contextMenu.value = { visible: true, x: domEvent.clientX, y: domEvent.clientY, eventId: event.id }
}

const closeContextMenu = () => {
  contextMenu.value = { visible: false, x: 0, y: 0, eventId: null }
}

const deleteContextMenuEvent = () => {
  if (contextMenu.value.eventId !== null) deleteEvent(contextMenu.value.eventId)
  closeContextMenu()
}

// --- Drag and drop ---
// Drag payload carries the event id plus where within the block it was grabbed,
// so dropping onto the week time-grid can position the event under the cursor.
const onEventDragStart = (event, domEvent) => {
  domEvent.dataTransfer.effectAllowed = 'move'
  domEvent.dataTransfer.setData(
    'text/plain',
    JSON.stringify({ id: event.id, grabOffsetY: domEvent.offsetY || 0 }),
  )
}

const getDragPayload = (domEvent) => {
  try {
    return JSON.parse(domEvent.dataTransfer.getData('text/plain'))
  } catch {
    return null
  }
}

const onDayDrop = (cell, domEvent) => {
  domEvent.preventDefault()
  const payload = getDragPayload(domEvent)
  if (!payload) return
  const event = events.value.find((e) => e.id === payload.id)
  if (event) {
    event.date = dateKey(cell.year, cell.month, cell.day)
  }
}

const onBankDrop = (domEvent) => {
  domEvent.preventDefault()
  const payload = getDragPayload(domEvent)
  if (!payload) return
  const event = events.value.find((e) => e.id === payload.id)
  if (event) {
    event.date = null
  }
}

const onDayDragOver = (domEvent) => {
  domEvent.preventDefault()
  domEvent.dataTransfer.dropEffect = 'move'
}

// Dropping in the week time-grid: the drop Y position (adjusted for where the
// event was grabbed) sets its new start time, snapped to 15-minute increments,
// while preserving its duration. Dropping on a different day's column also
// moves it to that day.
const onWeekColumnDrop = (cell, domEvent) => {
  domEvent.preventDefault()
  const payload = getDragPayload(domEvent)
  if (!payload) return
  const event = events.value.find((e) => e.id === payload.id)
  if (!event) return

  const rect = domEvent.currentTarget.getBoundingClientRect()
  const offsetY = domEvent.clientY - rect.top - (payload.grabOffsetY || 0)
  const rawMinutes = (offsetY / HOUR_HEIGHT) * 60
  const snapped = Math.round(rawMinutes / 15) * 15

  const startMin = timeToMinutes(event.start)
  const endMin = timeToMinutes(event.end)
  const duration = startMin !== null && endMin !== null ? endMin - startMin : 30

  const clampedStart = Math.max(0, Math.min(24 * 60 - duration, snapped))
  event.date = dateKey(cell.year, cell.month, cell.day)
  event.start = minutesToTime(clampedStart)
  event.end = minutesToTime(clampedStart + duration)
}

// Dropping on the all-day strip clears the time, moving the event out of the hourly grid.
const onAllDayDrop = (cell, domEvent) => {
  domEvent.preventDefault()
  const payload = getDragPayload(domEvent)
  if (!payload) return
  const event = events.value.find((e) => e.id === payload.id)
  if (!event) return
  event.date = dateKey(cell.year, cell.month, cell.day)
  event.start = ''
  event.end = ''
}
</script>

<template>
  <div class="calendar">
    <!-- Header -->
    <div class="header">
      <button @click="goPrev">&lt;</button>
      <h2>{{ headerLabel }}</h2>
      <button @click="goNext">&gt;</button>
      <div class="view-toggle">
        <button :class="{ active: viewMode === 'month' }" @click="setViewMode('month')">Month</button>
        <button :class="{ active: viewMode === 'week' }" @click="setViewMode('week')">Week</button>
      </div>
    </div>

    <div class="body">
      <!-- Event Bank -->
      <div class="bank" @dragover="onDayDragOver" @drop="onBankDrop">
        <div class="bank-header">
          <h3 style="color: black">Event Bank</h3>
          <button class="add-event-btn" @click="openBankForm">+</button>
        </div>
        <div class="bank-events">
          <div
            v-for="event in bankEvents"
            :key="event.id"
            class="event-item"
            :style="{ background: event.color || '#3182ce' }"
            draggable="true"
            @dragstart="onEventDragStart(event, $event)"
            @click="openEditForm(event)"
            @contextmenu="openContextMenu(event, $event)"
          >
            <div class="event-title">{{ event.title }}</div>
            <div v-if="event.start || event.end" class="event-time">
              {{ formatTimeLabel(event.start) }}<span v-if="event.start && event.end"> - </span>{{ formatTimeLabel(event.end) }}
            </div>
          </div>
        </div>
      </div>

      <!-- Main Calendar -->
      <div class="main">
        <!-- Month Grid View -->
        <template v-if="viewMode === 'month'">
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
              <div class="day-number" >{{ cell.day }}</div>

              <div class="day-events">
                <div
                  v-for="event in eventsForDay(cell)"
                  :key="event.id"
                  class="event-item"
                  :style="{ background: event.color || '#3182ce' }"
                  draggable="true"
                  @dragstart="onEventDragStart(event, $event)"
                  @click="openEditForm(event)"
                  @contextmenu="openContextMenu(event, $event)"
                >
                  <div class="event-title">{{ event.title }}</div>
                  <div v-if="event.start || event.end" class="event-time">
                    {{ formatTimeLabel(event.start) }}<span v-if="event.start && event.end"> - </span>{{ formatTimeLabel(event.end) }}
                  </div>
                </div>
              </div>

              <button class="add-event-btn" @click="openCreateForm(cell)">+</button>
            </div>
          </div>
        </template>

        <!-- Weekly Time-Grid View -->
        <div v-else class="week-view">
          <div class="week-header-row">
            <div class="week-gutter-spacer"></div>
            <div
              v-for="(cell, index) in weekDays"
              :key="`h-${cell.year}-${cell.month}-${cell.day}`"
              class="week-day-header"
              :class="{ today: isToday(cell) }"
            >
              <span class="week-day-name">{{ weekdays[index] }}</span>
              <span class="week-day-number">{{ cell.day }}</span>
              <button class="add-event-btn" @click="openCreateForm(cell)">+</button>
            </div>
          </div>

          <div class="week-allday-row">
            <div class="week-gutter-spacer week-allday-label">All-day</div>
            <div
              v-for="cell in weekDays"
              :key="`a-${cell.year}-${cell.month}-${cell.day}`"
              class="week-allday-cell"
              @dragover="onDayDragOver"
              @drop="onAllDayDrop(cell, $event)"
            >
              <div
                v-for="event in allDayEventsForDay(cell)"
                :key="event.id"
                class="event-item"
                :style="{ background: event.color || '#3182ce' }"
                draggable="true"
                @dragstart="onEventDragStart(event, $event)"
                @click="openEditForm(event)"
                @contextmenu="openContextMenu(event, $event)"
              >
                <div class="event-title">{{ event.title }}</div>
              </div>
            </div>
          </div>

          <div class="week-body" ref="weekBodyEl">
            <div class="week-gutter">
              <div
                v-for="hour in 24"
                :key="hour"
                class="hour-label"
                :style="{ height: HOUR_HEIGHT + 'px' }"
              >
                {{ hourLabel(hour - 1) }}
              </div>
            </div>
            <div class="week-grid">
              <div
                v-for="cell in weekDays"
                :key="`g-${cell.year}-${cell.month}-${cell.day}`"
                class="week-day-column"
                :style="{ height: HOUR_HEIGHT * 24 + 'px' }"
                @dragover="onDayDragOver"
                @drop="onWeekColumnDrop(cell, $event)"
              >
                <div
                  v-for="hour in 24"
                  :key="hour"
                  class="hour-line"
                  :style="{ top: HOUR_HEIGHT * (hour - 1) + 'px' }"
                ></div>
                <div
                  v-for="event in timedEventsForDay(cell)"
                  :key="event.id"
                  class="week-event"
                  :style="eventBlockStyle(event)"
                  draggable="true"
                  @dragstart="onEventDragStart(event, $event)"
                  @click="openEditForm(event)"
                  @contextmenu="openContextMenu(event, $event)"
                >
                  <div class="event-title">{{ event.title }}</div>
                  <div class="event-time">
                    {{ formatTimeLabel(event.start) }}<span v-if="event.end"> - {{ formatTimeLabel(event.end) }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Create / Edit Event Modal -->
    <div v-if="showForm" class="modal-overlay" @click.self="closeForm">
      <div class="modal">
        <h3 v-if="formEditingId !== null">Edit Event</h3>
        <h3 v-else-if="formIsBank">New Event</h3>
        <h3 v-else>New Event &mdash; {{ formMonthName }} {{ formCell?.day }}, {{ formCell?.year }}</h3>
        <input
          type="text"
          v-model="formTitle"
          class="form-control"
          placeholder="Enter event title"
          @keyup.enter="saveEvent"
        />
        <label class="color-field">
          Color
          <input type="color" v-model="formColor" class="color-input" />
        </label>
        <input type="time" v-model="formStart" class="form-control" placeholder="Start time" />
        <input type="time" v-model="formEnd" class="form-control" placeholder="End time" />
        <label v-if="formIsBank && formEditingId === null" class="quantity-field">
          Number of events
          <input type="number" v-model.number="formQuantity" class="form-control" min="1" max="20" />
        </label>
        <div class="modal-actions">
          <button @click="closeForm">Cancel</button>
          <button @click="saveEvent">{{ formEditingId !== null ? 'Save Changes' : 'Create Event' }}</button>
        </div>
      </div>
    </div>

    <!-- Event Context Menu -->
    <div
      v-if="contextMenu.visible"
      class="context-menu-overlay"
      @click="closeContextMenu"
      @contextmenu.prevent="closeContextMenu"
    >
      <div class="context-menu" :style="{ top: contextMenu.y + 'px', left: contextMenu.x + 'px' }" @click.stop>
        <button @click="deleteContextMenuEvent">Delete</button>
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
  gap: 0.75rem;
}
.header h2 {
  flex: 1;
  text-align: center;
}
.view-toggle {
  display: flex;
  gap: 4px;
}
.view-toggle button {
  padding: 4px 10px;
  border: 1px solid #ccc;
  background: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.85rem;
}
.view-toggle button.active {
  background: #3182ce;
  color: white;
  border-color: #3182ce;
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
  color: #555;
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
.color-field, .quantity-field {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 0.85rem;
  gap: 0.5rem;
}
.color-input {
  width: 48px;
  height: 28px;
  padding: 0;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
}
.quantity-field .form-control {
  width: 70px;
}
.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 0.5rem;
}
.context-menu-overlay {
  position: fixed;
  inset: 0;
  z-index: 200;
}
.context-menu {
  position: fixed;
  background: white;
  color: #222;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.25);
  overflow: hidden;
  min-width: 120px;
}
.context-menu button {
  display: block;
  width: 100%;
  text-align: left;
  padding: 8px 12px;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 0.85rem;
  color: #c0392b;
}
.context-menu button:hover {
  background: #f4f4f4;
}

/* --- Weekly time-grid view --- */
.week-view {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
}
.week-header-row {
  display: flex;
}
.week-gutter-spacer {
  width: 52px;
  flex-shrink: 0;
}
.week-day-header {
  flex: 1;
  position: relative;
  text-align: center;
  padding: 4px 0 6px;
  border-radius: 4px;
}
.week-day-header.today {
  background: #d6e9ff;
  font-weight: bold;
}
.week-day-name {
  display: block;
  font-size: 0.7rem;
  color: #666;
}
.week-day-number {
  display: block;
  font-size: 1rem;
}
.week-day-header .add-event-btn {
  position: absolute;
  top: 0;
  right: 2px;
}
.week-allday-row {
  display: flex;
  min-height: 26px;
  border-top: 1px solid #ddd;
  border-bottom: 1px solid #ddd;
  padding: 2px 0;
}
.week-allday-label {
  font-size: 0.65rem;
  color: #888;
  padding-top: 4px;
  text-align: right;
  padding-right: 6px;
  box-sizing: border-box;
}
.week-allday-cell {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding: 0 3px;
  border-left: 1px solid #eee;
}
.week-body {
  flex: 1;
  display: flex;
  overflow-y: auto;
  min-height: 0;
}
.week-gutter {
  width: 52px;
  flex-shrink: 0;
}
.hour-label {
  font-size: 0.65rem;
  color: #888;
  text-align: right;
  padding-right: 6px;
  padding-top: 2px;
  box-sizing: border-box;
}
.week-grid {
  flex: 1;
  display: flex;
  position: relative;
}
.week-day-column {
  flex: 1;
  position: relative;
  border-left: 1px solid #eee;
}
.hour-line {
  position: absolute;
  left: 0;
  right: 0;
  border-top: 1px solid #eee;
}
.week-event {
  position: absolute;
  left: 2px;
  right: 2px;
  color: white;
  border-radius: 3px;
  padding: 2px 4px;
  font-size: 0.7rem;
  overflow: hidden;
  cursor: grab;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.25);
}
</style>