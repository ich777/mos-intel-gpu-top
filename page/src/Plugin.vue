<template>
  <div>
    <h2 class="mb-4">Intel-GPU-TOP Plugin</h2>
    <v-skeleton-loader v-if="loading" :loading="true" type="card" />
    <v-card v-else-if="!isConfigured" class="mb-4 pa-0">
      <v-card-text class="pa-4">
        Please configure the plugin first to display GPU data.
      </v-card-text>
    </v-card>
    <div v-else style="margin-bottom: 80px">
      <v-card v-for="(gpu, index) in settings.gpus" :key="index" class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <span>{{ getGpuName(gpu.pci) }}</span>
          <v-spacer />
          <v-chip size="small">{{ gpu.pci }}</v-chip>
        </v-card-title>
        <v-card-text v-if="gpuData[gpu.pci]" class="pa-4">
          <v-row dense>
            <v-col v-if="gpuData[gpu.pci].frequency?.requested !== undefined" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>Frequency Requested</strong></div>
              <div class="text-body-2">{{ Math.round(gpuData[gpu.pci].frequency.requested) }} MHz</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci].frequency?.actual !== undefined" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>Frequency Actual</strong></div>
              <div class="text-body-2">{{ Math.round(gpuData[gpu.pci].frequency.actual) }} MHz</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci].interrupts" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>Interrupts</strong></div>
              <div class="text-body-2">{{ Math.round(gpuData[gpu.pci].interrupts.count) }}/s</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci].power?.['GPU']" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>Power GPU</strong></div>
              <div class="text-body-2">{{ (gpuData[gpu.pci].power['GPU'].power || 0).toFixed(2) }} W</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci].power?.['Package']" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>Power Package</strong></div>
              <div class="text-body-2">{{ (gpuData[gpu.pci].power['Package'].power || 0).toFixed(2) }} W</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci]['imc-bandwidth']?.reads !== undefined" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>IMC Reads</strong></div>
              <div class="text-body-2">{{ Math.round(gpuData[gpu.pci]['imc-bandwidth'].reads) }} MiB/s</div>
            </v-col>
            <v-col v-if="gpuData[gpu.pci]['imc-bandwidth']?.writes !== undefined" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>IMC Writes</strong></div>
              <div class="text-body-2">{{ Math.round(gpuData[gpu.pci]['imc-bandwidth'].writes) }} MiB/s</div>
            </v-col>
          </v-row>
          <v-divider class="mt-2 mb-2" />
          <v-row v-if="gpuData[gpu.pci].rc6" dense>
            <v-col cols="auto" class="d-flex align-center text-caption" style="width: 60px">
              <strong>RC6:</strong>
            </v-col>
            <v-col class="d-flex align-center">
              <v-progress-linear
                :model-value="gpuData[gpu.pci].rc6.value || 0"
                height="16"
                color="green"
                style="border-radius: 7px; overflow: hidden"
              >
                <template #default>
                  <span><small>{{ (gpuData[gpu.pci].rc6.value || 0).toFixed(1) }}%</small></span>
                </template>
              </v-progress-linear>
            </v-col>
          </v-row>
          <v-row v-if="gpuData[gpu.pci].engines" dense class="mt-2">
            <v-col cols="12">
              <div class="text-caption text-medium-emphasis mb-1"><strong>Engines</strong></div>
              <v-row v-for="(engine, engineName) in gpuData[gpu.pci].engines" :key="engineName" dense>
                <v-col>
                  <div style="min-width: 0; display: flex; align-items: center; gap: 6px">
                    <div class="text-body-2" style="width: 100px"><small><b>{{ engineName }}</b></small></div>
                    <div style="flex: 1 1 auto; min-width: 0">
                      <v-progress-linear
                        :model-value="engine.busy || 0"
                        height="12"
                        :color="engine.busy >= 90 ? 'red' : engine.busy >= 75 ? 'orange' : 'green'"
                        style="border-radius: 6px; overflow: hidden"
                      >
                        <template #default>
                          <span><small>{{ (engine.busy || 0).toFixed(1) }}%</small></span>
                        </template>
                      </v-progress-linear>
                    </div>
                  </div>
                </v-col>
              </v-row>
            </v-col>
          </v-row>
          <v-row v-if="gpuData[gpu.pci].clients && Object.keys(gpuData[gpu.pci].clients).length > 0" dense class="mt-2">
            <v-col cols="12">
              <details>
                <summary style="cursor: pointer; color: var(--v-theme-primary); text-decoration: underline" class="text-body-2 mb-1">Clients</summary>
                <div v-for="(client, clientId) in gpuData[gpu.pci].clients" :key="clientId" class="mb-3">
                  <div class="d-flex align-center mb-1">
                    <span class="text-body-2"><b>{{ client.name || 'Unknown' }}</b></span>
                    <span class="text-caption text-medium-emphasis ml-2">PID: {{ client.pid || clientId }}</span>
                  </div>
                  <v-row v-if="client['engine-classes']" dense>
                    <v-col v-for="(eng, engName) in client['engine-classes']" :key="engName" cols="6">
                      <div style="display: flex; align-items: center; gap: 6px">
                        <span class="text-caption" style="width: 100px; flex-shrink: 0">{{ engName }}</span>
                        <v-progress-linear
                          :model-value="parseFloat(eng.busy) || 0"
                          height="10"
                          :color="parseFloat(eng.busy) >= 90 ? 'red' : parseFloat(eng.busy) >= 75 ? 'orange' : 'green'"
                          style="border-radius: 4px; overflow: hidden"
                        />
                        <span class="text-caption" style="width: 45px; text-align: right; flex-shrink: 0">{{ parseFloat(eng.busy || 0).toFixed(1) }}%</span>
                      </div>
                    </v-col>
                  </v-row>
                </div>
              </details>
            </v-col>
          </v-row>
        </v-card-text>
        <v-card-text v-else class="text-center pa-8 text-grey">
          Loading data...
        </v-card-text>
      </v-card>
    </div>
    <v-dialog v-model="settingsDialog.value" max-width="600">
      <v-card class="pa-0">
        <v-card-title>Settings</v-card-title>
        <v-card-text>
          <v-form>
            <v-text-field
              v-model.number="settingsDialog.interval"
              label="Update interval (seconds)"
              type="number"
              min="1"
              @blur="validateInterval"
            />
            <v-select
              v-model="settingsDialog.selectedGpus"
              :items="availableGpusItems"
              item-title="title"
              item-value="value"
              label="GPUs"
              multiple
              chips
              clearable
            />
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn color="onPrimary" @click="settingsDialog.value = false">Cancel</v-btn>
          <v-btn color="onPrimary" @click="saveSettings" :loading="settingsDialog.saving">Save</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-fab color="primary" style="position: fixed; bottom: 32px; right: 32px; z-index: 1000" size="large" icon @click="openSettingsDialog">
      <v-icon>mdi-cog</v-icon>
    </v-fab>
  </div>
</template>

<script setup>
import { ref, computed, reactive, onMounted, onUnmounted } from 'vue';

const loading = ref(true);
const settings = ref({ gpus: [], interval: 2 });
const gpuData = ref({});
const availableGpus = ref([]);
let pollInterval = null;

const settingsDialog = reactive({
  value: false,
  interval: 2,
  selectedGpus: [],
  saving: false,
});

const isConfigured = computed(() => {
  return settings.value.gpus && settings.value.gpus.length > 0 && settings.value.gpus.some((g) => g.pci);
});

const availableGpusItems = computed(() => {
  return availableGpus.value.map((gpu) => ({
    title: `${gpu.name} (${gpu.pci})`,
    value: gpu.pci,
  }));
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const getGpuName = (pci) => {
  const gpu = availableGpus.value.find((g) => g.pci === pci);
  return gpu ? gpu.name : `GPU ${pci}`;
};

const validateInterval = () => {
  if (settingsDialog.interval < 1) {
    settingsDialog.interval = 1;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/settings/intel-gpu-top', {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      settings.value = {
        gpus: Array.isArray(data.gpus) ? data.gpus : [],
        interval: data.interval || 2,
      };
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const fetchAvailableGpus = async () => {
  try {
    const res = await fetch('/api/v1/system/gpus', {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.Intel && Array.isArray(data.Intel)) {
        availableGpus.value = data.Intel.map((gpu) => ({
          pci: gpu.pci,
          name: gpu.name,
        }));
      }
    }
  } catch (e) {
    console.error('Failed to fetch GPUs:', e);
  }
};

const fetchGpuData = async (gpu) => {
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'intel_gpu_top',
        args: ['-J', '-n', '1', `sys:/sys/devices/pci0000:00/0000:${gpu.pci}`],
        timeout: 2,
        parse_json: true,
      }),
    });

    if (res.ok) {
      const data = await res.json();
      gpuData.value[gpu.pci] = Array.isArray(data.output) ? data.output[0] : data.output;
    }
  } catch (e) {
    console.error(`Failed to fetch data for GPU ${gpu.pci}:`, e);
  }
};

const fetchAllGpuData = async () => {
  if (!isConfigured.value) return;

  for (const gpu of settings.value.gpus) {
    if (gpu.pci) {
      await fetchGpuData(gpu);
    }
  }
};

const startPolling = () => {
  stopPolling();
  if (!isConfigured.value) return;

  fetchAllGpuData();
  const interval = Math.max(1, settings.value.interval) * 1000;
  pollInterval = setInterval(fetchAllGpuData, interval);
};

const stopPolling = () => {
  if (pollInterval) {
    clearInterval(pollInterval);
    pollInterval = null;
  }
};

const openSettingsDialog = () => {
  settingsDialog.interval = settings.value.interval || 2;
  settingsDialog.selectedGpus = settings.value.gpus.map((g) => g.pci).filter(Boolean);
  settingsDialog.saving = false;
  settingsDialog.value = true;
};

const saveSettings = async () => {
  settingsDialog.saving = true;
  validateInterval();

  try {
    const gpus = settingsDialog.selectedGpus.map((pci) => ({ pci }));

    const res = await fetch('/api/v1/mos/plugins/settings/intel-gpu-top', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        gpus: gpus,
        interval: settingsDialog.interval,
      }),
    });

    if (res.ok) {
      settings.value = {
        gpus: gpus,
        interval: settingsDialog.interval,
      };
      settingsDialog.value = false;
      startPolling();
    }
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    settingsDialog.saving = false;
  }
};

onMounted(async () => {
  try {
    await Promise.all([fetchSettings(), fetchAvailableGpus()]);
    if (isConfigured.value) {
      startPolling();
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  stopPolling();
});
</script>
