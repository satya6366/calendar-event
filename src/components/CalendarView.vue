<template>
  <div class="container mx-auto p-4">
    <!-- Search and Filter Bar -->
    <div class="mb-4 flex flex-col sm:flex-row gap-2">
      <input
        v-model="searchQuery"
        placeholder="Search events..."
        class="border p-2 rounded w-full sm:w-1/3"
      />
      <select
        v-model="filterColor"
        class="border p-2 rounded w-full sm:w-1/4"
      >
        <option value="">All Categories</option>
        <option value="bg-blue-500">Blue</option>
        <option value="bg-green-500">Green</option>
        <option value="bg-red-500">Red</option>
        <option value="bg-yellow-500">Yellow</option>
      </select>
    </div>

    <!-- Month Navigation -->
    <div class="flex justify-between items-center mb-4">
      <button
        @click="changeMonth(-1)"
        class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
      >
        Prev
      </button>
      <h2 class="text-xl font-semibold">{{ format(currentMonth, 'MMMM yyyy') }}</h2>
      <button
        @click="changeMonth(1)"
        class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
      >
        Next
      </button>
    </div>

    <!-- Calendar Grid -->
    <div class="grid grid-cols-7 gap-1 text-center sm:grid-cols-7">
      <div class="font-bold hidden sm:block" v-for="day in weekDays" :key="day">
        {{ day }}
      </div>
      <div
        v-for="date in calendarDays"
        :key="date.toISOString()"
        class="h-24 border p-1 relative cursor-pointer overflow-auto"
        :class="{
          'bg-blue-100': isToday(date),
          'bg-gray-100': !isSameMonth(date, currentMonth),
          'drop-target': true
        }"
        @click="openAddEventModal(date)"
        @drop="handleDrop(date)"
        @dragover.prevent
        @dragenter.prevent
      >
        <div class="text-sm">{{ format(date, 'd') }}</div>
        <div
          v-for="event in filteredEvents(date)"
          :key="event.id"
          class="text-xs text-white rounded px-1 mt-1 cursor-pointer truncate draggable"
          :class="event.color || 'bg-blue-500'"
          :draggable="true"
          @dragstart="handleDragStart(event)"
          @click.stop="openEditEventModal(event)"
        >
          {{ event.title }}
          <span v-if="hasConflict(event)" class="text-red-200 font-bold">!</span>
        </div>
      </div>
    </div>

    <!-- Event Modal -->
    <EventModal
      :event-date="selectedDate"
      :event-to-edit="editingEvent"
      @save="saveEvent"
      @delete="deleteEvent"
      @close="closeModal"
      v-if="isModalOpen"
    />

    <!-- Conflict Warning Modal -->
    <div
      v-if="conflictWarning"
      class="fixed inset-0 bg-black bg-opacity-30 flex justify-center items-center z-50"
    >
      <div class="bg-white p-4 rounded shadow max-w-md w-full">
        <h3 class="text-lg font-bold">Event Conflict</h3>
        <p>{{ conflictWarning }}</p>
        <div class="flex justify-end gap-2 mt-4">
          <button
            @click="conflictWarning = ''"
            class="px-3 py-1 bg-gray-300 rounded"
          >
            Cancel
          </button>
          <button
            @click="confirmConflict"
            class="px-3 py-1 bg-green-500 text-white rounded"
          >
            Proceed
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import {
  format,
  startOfMonth,
  endOfMonth,
  startOfWeek,
  endOfWeek,
  addDays,
  addMonths,
  isSameDay,
  addWeeks,
  isSameMonth,
  parseISO,
  isWithinInterval,
} from 'date-fns';
import interact from 'interactjs';
import EventModal from './EventModal.vue';

const events = ref([]);
const today = new Date();
const currentMonth = ref(startOfMonth(today));
const isModalOpen = ref(false);
const selectedDate = ref(null);
const editingEvent = ref(null);
const searchQuery = ref('');
const filterColor = ref('');
const conflictWarning = ref('');
const pendingEvent = ref(null);

const loadEvents = () => {
  const saved = localStorage.getItem('events');
  if (saved) events.value = JSON.parse(saved);
};

const saveToStorage = () => {
  localStorage.setItem('events', JSON.stringify(events.value));
};

const changeMonth = (offset) => {
  currentMonth.value = addMonths(currentMonth.value, offset);
};

const weekDays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

const calendarDays = computed(() => {
  const start = startOfWeek(startOfMonth(currentMonth.value));
  const end = endOfWeek(endOfMonth(currentMonth.value));
  const days = [];
  let date = start;
  while (date <= end) {
    days.push(date);
    date = addDays(date, 1);
  }
  return days;
});

const isToday = (date) => isSameDay(date, today);

const generateRecurringEvents = (event) => {
  const result = [];
  let current = parseISO(event.date);
  const rangeEnd = endOfMonth(currentMonth.value);
  const recurrenceDays = event.recurrenceDays || [];

  while (current <= rangeEnd) {
    if (
      !event.recurrence ||
      event.recurrence === 'daily' ||
      (event.recurrence === 'weekly' &&
        recurrenceDays.includes(format(current, 'EEE'))) ||
      (event.recurrence === 'monthly' &&
        format(current, 'd') === format(parseISO(event.date), 'd')) ||
      (event.recurrence === 'custom' && current.getDate() % event.customInterval === 0)
    ) {
      result.push({ ...event, date: current.toISOString() });
    }
    if (event.recurrence === 'daily') current = addDays(current, 1);
    else if (event.recurrence === 'weekly') current = addDays(current, 1);
    else if (event.recurrence === 'monthly') current = addMonths(current, 1);
    else if (event.recurrence === 'custom') current = addDays(current, 1);
    else break;
  }
  return result;
};

const getEvents = (date) => {
  const dayEvents = [];
  for (const event of events.value) {
    if (event.recurrence) {
      dayEvents.push(
        ...generateRecurringEvents(event).filter((e) => isSameDay(parseISO(e.date), date))
      );
    } else if (isSameDay(parseISO(event.date), date)) {
      dayEvents.push(event);
    }
  }
  return dayEvents;
};

const filteredEvents = computed(() => {
  return (date) => {
    let dayEvents = getEvents(date);
    if (searchQuery.value) {
      dayEvents = dayEvents.filter(
        (event) =>
          event.title.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
          event.description.toLowerCase().includes(searchQuery.value.toLowerCase())
      );
    }
    if (filterColor.value) {
      dayEvents = dayEvents.filter((event) => event.color === filterColor.value);
    }
    return dayEvents;
  };
});

const hasConflict = (event) => {
  const eventDateTime = parseISO(event.date);
  const dayEvents = getEvents(eventDateTime).filter((e) => e.id !== event.id);
  return dayEvents.some((e) =>
    isWithinInterval(parseISO(e.date), {
      start: addDays(parseISO(event.date), -1),
      end: addDays(parseISO(event.date), 1),
    })
  );
};

const checkConflicts = (event) => {
  const conflicts = getEvents(parseISO(event.date)).filter(
    (e) => e.id !== event.id && isSameDay(parseISO(e.date), parseISO(event.date))
  );
  if (conflicts.length > 0) {
    return `Event "${event.title}" conflicts with "${conflicts[0].title}" on ${format(
      parseISO(event.date),
      'MMMM d, yyyy'
    )}. Proceed?`;
  }
  return '';
};

const openAddEventModal = (date) => {
  selectedDate.value = date;
  editingEvent.value = null;
  isModalOpen.value = true;
};

const openEditEventModal = (event) => {
  editingEvent.value = event;
  selectedDate.value = parseISO(event.date);
  isModalOpen.value = true;
};

const saveEvent = (event) => {
  const conflictMessage = checkConflicts(event);
  if (conflictMessage && !editingEvent.value) {
    conflictWarning.value = conflictMessage;
    pendingEvent.value = event;
    return;
  }
  const index = events.value.findIndex((e) => e.id === event.id);
  if (index !== -1) events.value[index] = event;
  else events.value.push(event);
  saveToStorage();
  isModalOpen.value = false;
  conflictWarning.value = '';
  pendingEvent.value = null;
};

const confirmConflict = () => {
  if (pendingEvent.value) {
    events.value.push(pendingEvent.value);
    saveToStorage();
    isModalOpen.value = false;
    conflictWarning.value = '';
    pendingEvent.value = null;
  }
};

const deleteEvent = (id) => {
  events.value = events.value.filter((e) => e.id !== id);
  saveToStorage();
  isModalOpen.value = false;
};

const closeModal = () => {
  isModalOpen.value = false;
  conflictWarning.value = '';
  pendingEvent.value = null;
};

const handleDragStart = (event) => {
  window.draggedEvent = event;
};

const handleDrop = (date) => {
  if (window.draggedEvent) {
    const updatedEvent = { ...window.draggedEvent, date: date.toISOString() };
    const conflictMessage = checkConflicts(updatedEvent);
    if (conflictMessage) {
      conflictWarning.value = conflictMessage;
      pendingEvent.value = updatedEvent;
      return;
    }
    const index = events.value.findIndex((e) => e.id === window.draggedEvent.id);
    if (index !== -1) {
      events.value[index] = updatedEvent;
      saveToStorage();
    }
    window.draggedEvent = null;
  }
};

onMounted(() => {
  loadEvents();
  interact('.draggable').draggable({
    listeners: {
      start(event) {
        event.target.classList.add('opacity-50');
      },
      end(event) {
        event.target.classList.remove('opacity-50');
      },
    },
  });
  interact('.drop-target').dropzone({
    ondrop: (event) => {
      const dateStr = event.target.getAttribute('data-date');
      if (dateStr) {
        handleDrop(new Date(dateStr));
      }
    },
  });
});
</script>

<style scoped>
@media (max-width: 640px) {
  .grid-cols-7 {
    grid-template-columns: repeat(7, minmax(0, 1fr));
  }
  .h-24 {
    height: 100px;
  }
  .text-sm {
    font-size: 0.75rem;
  }
  .text-xs {
    font-size: 0.65rem;
  }
}
</style>