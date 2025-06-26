<template>
  <div class="fixed inset-0 bg-black bg-opacity-30 flex justify-center items-center z-50">
    <div class="bg-white p-4 rounded shadow max-w-md w-full space-y-3">
      <h2 class="text-lg font-bold">{{ eventToEdit ? 'Edit Event' : 'Add Event' }}</h2>
      <div class="space-y-2">
        <input
          v-model="form.title"
          placeholder="Title"
          class="border p-2 w-full rounded"
          required
        />
        <input
          type="datetime-local"
          v-model="form.date"
          class="border p-2 w-full rounded"
          required
        />
        <textarea
          v-model="form.description"
          placeholder="Description"
          class="border p-2 w-full rounded"
        ></textarea>
        <select v-model="form.recurrence" class="border p-2 w-full rounded">
          <option value="">No Recurrence</option>
          <option value="daily">Daily</option>
          <option value="weekly">Weekly</option>
          <option value="monthly">Monthly</option>
          <option value="custom">Custom</option>
        </select>
        <div v-if="form.recurrence === 'weekly'" class="flex flex-wrap gap-2">
          <label v-for="day in weekDays" :key="day" class="flex items-center">
            <input
              type="checkbox"
              v-model="form.recurrenceDays"
              :value="day"
              class="mr-1"
            />
            {{ day }}
          </label>
        </div>
        <div v-if="form.recurrence === 'custom'" class="flex items-center gap-2">
          <label>Every</label>
          <input
            v-model.number="form.customInterval"
            type="number"
            min="1"
            class="border p-2 w-16 rounded"
          />
          <span>days</span>
        </div>
        <select v-model="form.color" class="border p-2 w-full rounded">
          <option value="bg-blue-500">Blue</option>
          <option value="bg-green-500">Green</option>
          <option value="bg-red-500">Red</option>
          <option value="bg-yellow-500">Yellow</option>
        </select>
        <div class="flex justify-end gap-2 pt-2">
          <button
            type="button"
            @click="$emit('close')"
            class="px-3 py-1 bg-gray-300 rounded hover:bg-gray-400"
          >
            Cancel
          </button>
          <button
            v-if="eventToEdit"
            type="button"
            @click="$emit('delete', form.id)"
            class="px-3 py-1 bg-red-500 text-white rounded hover:bg-red-600"
          >
            Delete
          </button>
          <button
            type="button"
            @click="submit"
            class="px-3 py-1 bg-green-500 text-white rounded hover:bg-green-600"
          >
            Save
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue';
import { v4 as uuidv4 } from 'uuid';
import { format } from 'date-fns';

const props = defineProps(['eventDate', 'eventToEdit']);
const emit = defineEmits(['save', 'delete', 'close']);

const weekDays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

const form = ref({
  id: '',
  title: '',
  date: '',
  description: '',
  recurrence: '',
  recurrenceDays: [],
  customInterval: 1,
  color: 'bg-blue-500',
});

watch(
  () => props.eventToEdit,
  (event) => {
    if (event) {
      form.value = {
        ...event,
        recurrenceDays: event.recurrenceDays || [],
        customInterval: event.customInterval || 1,
      };
    } else {
      form.value = {
        id: uuidv4(),
        title: '',
        date: props.eventDate ? format(props.eventDate, "yyyy-MM-dd'T'HH:mm") : '',
        description: '',
        recurrence: '',
        recurrenceDays: [],
        customInterval: 1,
        color: 'bg-blue-500',
      };
    }
  },
  { immediate: true }
);

const submit = () => {
  if (form.value.recurrence !== 'weekly') {
    form.value.recurrenceDays = [];
  }
  if (form.value.recurrence !== 'custom') {
    form.value.customInterval = 1;
  }
  emit('save', form.value);
};
</script>