<template>
  <div
    v-if="isOpen"
    class="fixed inset-0 z-[100] flex items-center justify-center p-4 bg-black bg-opacity-70 backdrop-blur-sm"
  >
    <div
      class="bg-gray-800 rounded-2xl shadow-2xl w-full max-w-md border border-gray-700 overflow-hidden"
    >
      <!-- Modal Header -->
      <div
        class="px-6 py-4 bg-gray-700 border-b border-gray-600 flex justify-between items-center bg-gradient-to-r from-gray-700 to-gray-800"
      >
        <h2 class="text-xl font-bold text-white">
          {{ isEditMode ? 'Kamera bearbeiten' : 'Kamera hinzufügen' }}
        </h2>
        <button
          @click="$emit('close')"
          class="text-gray-400 hover:text-white transition-colors focus:outline-none"
        >
          <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M6 18L18 6M6 6l12 12"
            />
          </svg>
        </button>
      </div>

      <!-- Modal Body -->
      <div class="px-6 py-6 space-y-6 max-h-[70vh] overflow-y-auto">
        <!-- Camera Name -->
        <div>
          <label class="block text-sm font-medium text-gray-300 mb-2">Kamera Name</label>
          <input
            v-model="editName"
            type="text"
            class="w-full px-4 py-2.5 bg-gray-900 border border-gray-700 rounded-lg text-white placeholder-gray-500 focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition-all"
            placeholder="z.B. Süd-Himmel, Allsky..."
          />
        </div>

        <!-- Snapshot URL -->
        <div>
          <label class="block text-sm font-medium text-gray-300 mb-2">Snapshot-URL</label>
          <input
            v-model="editUrl"
            type="text"
            class="w-full px-4 py-2.5 bg-gray-900 border border-gray-700 rounded-lg text-white placeholder-gray-500 focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition-all"
            placeholder="http://192.168.1.100/snapshot.jpg"
          />
        </div>

        <!-- Auto-Refresh Toggle -->
        <div class="flex items-center justify-between">
          <div>
            <label class="block text-sm font-medium text-gray-300">Auto-Refresh</label>
            <p class="text-[10px] text-gray-500 mt-0.5">Bild automatisch aktualisieren</p>
          </div>
          <button
            @click="editAutoRefresh = !editAutoRefresh"
            class="relative inline-flex h-6 w-11 items-center rounded-full transition-colors"
            :class="editAutoRefresh ? 'bg-indigo-600' : 'bg-gray-600'"
          >
            <span
              class="inline-block h-4 w-4 transform rounded-full bg-white transition-transform"
              :class="editAutoRefresh ? 'translate-x-6' : 'translate-x-1'"
            />
          </button>
        </div>

        <!-- Refresh Interval -->
        <div v-if="editAutoRefresh">
          <label class="block text-sm font-medium text-gray-300 mb-2">
            Aktualisierungsintervall
          </label>
          <div class="flex items-center space-x-3">
            <input
              v-model.number="editIntervalSec"
              type="number"
              min="1"
              step="1"
              class="w-32 px-4 py-2.5 bg-gray-900 border border-gray-700 rounded-lg text-white placeholder-gray-500 focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition-all text-center"
              @change="editIntervalSec = Math.max(1, editIntervalSec)"
            />
            <span class="text-sm text-gray-400">Sekunden</span>
          </div>
          <p class="text-[10px] text-gray-600 mt-1.5">Minimum: 1 Sekunde</p>
        </div>
      </div>

      <!-- Modal Footer -->
      <div class="px-6 py-4 bg-gray-900 flex justify-between">
        <button
          v-if="isEditMode"
          @click="handleRemove"
          class="px-4 py-2 text-sm font-semibold text-red-400 hover:bg-red-900/30 rounded-lg transition-colors border border-red-900/50"
        >
          Löschen
        </button>
        <div v-else></div>

        <div class="flex space-x-3">
          <button
            @click="$emit('close')"
            class="px-5 py-2.5 text-sm font-semibold text-gray-300 bg-gray-800 hover:bg-gray-700 rounded-lg transition-colors border border-gray-700"
          >
            Abbrechen
          </button>
          <button
            @click="handleSave"
            :disabled="!editName || !editUrl"
            class="px-6 py-2.5 text-sm font-semibold text-white bg-indigo-600 hover:bg-indigo-500 disabled:opacity-30 disabled:cursor-not-allowed rounded-lg shadow-lg shadow-indigo-900/40 transition-all"
          >
            {{ isEditMode ? 'Speichern' : 'Hinzufügen' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { useImageMonitorStore } from '../store/imageMonitorStore';

const props = defineProps({
  isOpen: Boolean,
  cameraId: { type: String, default: null },
});

const emit = defineEmits(['close', 'remove', 'created']);
const store = useImageMonitorStore();

const isEditMode = computed(() => !!props.cameraId);
const camera = computed(() => store.getCameraById(props.cameraId));

const editName = ref('');
const editUrl = ref('');
const editIntervalSec = ref(60);
const editAutoRefresh = ref(true);

watch(
  () => props.isOpen,
  (newVal) => {
    if (!newVal) return;
    if (isEditMode.value && camera.value) {
      editName.value = camera.value.name;
      editUrl.value = camera.value.url;
      editIntervalSec.value = Math.round(camera.value.interval / 1000);
      editAutoRefresh.value = camera.value.autoRefresh ?? true;
    } else {
      editName.value = '';
      editUrl.value = '';
      editIntervalSec.value = 60;
      editAutoRefresh.value = true;
    }
  },
  { immediate: true }
);

const handleSave = () => {
  if (!editName.value || !editUrl.value) return;
  if (isEditMode.value) {
    store.updateCamera(props.cameraId, {
      name: editName.value,
      url: editUrl.value,
      interval: Math.max(1, editIntervalSec.value) * 1000,
      autoRefresh: editAutoRefresh.value,
    });
  } else {
    const id = store.addCamera({
      name: editName.value,
      url: editUrl.value,
      interval: Math.max(1, editIntervalSec.value) * 1000,
      autoRefresh: editAutoRefresh.value,
    });
    emit('created', id);
  }
  emit('close');
};

const handleRemove = () => {
  if (confirm(`Möchtest du die Kamera "${camera.value.name}" wirklich löschen?`)) {
    emit('remove', props.cameraId);
  }
};
</script>
