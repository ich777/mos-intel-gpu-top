<template>
  <div>
    <div v-if="loading" class="d-flex justify-center pa-4">
      <v-progress-circular indeterminate size="24" width="2" />
    </div>
    <div v-else-if="!isConfigured" class="text-center pa-3">
      <v-icon size="32" color="grey" class="mb-2">mdi-cog-outline</v-icon>
      <div class="text-body-2 text-medium-emphasis mb-2">{{ t('plugin_intel_gpu_top.not_configured') }}</div>
      <v-btn size="small" variant="tonal" color="primary" :to="`/plugins/intel-gpu-top`">
        <v-icon start size="16">mdi-open-in-app</v-icon>
        {{ t('plugin_intel_gpu_top.configure') }}
      </v-btn>
    </div>
    <div v-else>
      <div v-for="(gpu, index) in settings.gpus" :key="index" :class="{ 'mt-3': index > 0 }">
        <div class="d-flex align-center mb-2" style="min-width: 0">
          <span class="text-body-2 font-weight-bold" :title="getGpuName(gpu.pci)" style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis; min-width: 0; flex: 1">{{ getGpuName(gpu.pci) }}</span>
          <v-chip size="x-small" variant="outlined" class="ml-2" style="flex-shrink: 0">{{ gpu.pci }}</v-chip>
        </div>
        <div v-if="gpuData[gpu.pci]">
          <div v-if="gpuData[gpu.pci].frequency?.actual != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ t('plugin_intel_gpu_top.frequency') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ Math.round(gpuData[gpu.pci].frequency.actual) }} MHz</span>
          </div>
          <div v-if="gpuData[gpu.pci].power?.['GPU']?.power != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ t('plugin_intel_gpu_top.power') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ gpuData[gpu.pci].power['GPU'].power.toFixed(1) }} W</span>
          </div>
          <div v-if="gpuData[gpu.pci].rc6?.value != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>RC6</b></span>
            <v-progress-linear
              :model-value="gpuData[gpu.pci].rc6.value"
              height="10"
              color="green"
              style="border-radius: 5px; overflow: hidden; flex: 1"
            >
              <template #default>
                <span style="font-size: 9px">{{ gpuData[gpu.pci].rc6.value.toFixed(0) }}%</span>
              </template>
            </v-progress-linear>
          </div>
          <template v-if="gpuData[gpu.pci].engines">
            <template v-for="(engine, engineName) in gpuData[gpu.pci].engines" :key="engineName">
              <div v-if="engine.busy != null" class="d-flex align-center mb-1" style="gap: 6px">
                <span class="text-caption" style="width: 80px; flex-shrink: 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis" :title="engineName"><small>{{ engineName }}</small></span>
                <v-progress-linear
                  :model-value="engine.busy"
                  height="10"
                  :color="engine.busy >= 90 ? 'red' : engine.busy >= 75 ? 'orange' : 'green'"
                  style="border-radius: 5px; overflow: hidden; flex: 1"
                >
                  <template #default>
                    <span style="font-size: 9px">{{ engine.busy.toFixed(0) }}%</span>
                  </template>
                </v-progress-linear>
              </div>
            </template>
          </template>
          <div v-if="getClientCount(gpu.pci) > 0" class="text-caption text-medium-emphasis mt-1">
            <v-icon size="12">mdi-application-outline</v-icon>
            {{ getClientCount(gpu.pci) }} {{ getClientCount(gpu.pci) === 1 ? t('plugin_intel_gpu_top.process') : t('plugin_intel_gpu_top.processes') }}
          </div>
        </div>
        <div v-else class="text-caption text-medium-emphasis text-center pa-2">
          {{ t('plugin_intel_gpu_top.loading') }}
        </div>
        <v-divider v-if="index < settings.gpus.length - 1" class="mt-2" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import { useI18n } from 'vue-i18n';

const { t } = useI18n();

const loading = ref(true);
const settings = ref({ gpus: [], interval: 2 });
const gpuData = ref({});
const availableGpus = ref([]);
const hasSriovDriver = ref(false);
let pollInterval = null;

const isConfigured = computed(() => {
  return settings.value.gpus && settings.value.gpus.length > 0 && settings.value.gpus.some((g) => g.pci);
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const getGpuName = (pci) => {
  const gpu = availableGpus.value.find((g) => g.pci === pci);
  return gpu ? gpu.name : `GPU ${pci}`;
};

const getClientCount = (pci) => {
  const clients = gpuData.value[pci]?.clients;
  if (!clients) return 0;
  return Object.keys(clients).length;
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
  } catch { }
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
  } catch { }
};

const checkSriovDriver = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins', {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.results && Array.isArray(data.results)) {
        hasSriovDriver.value = data.results.some((plugin) => plugin.name === 'sriov-driver');
      }
    }
  } catch { }
};

const fetchGpuData = async (gpu) => {
  try {
    const args = hasSriovDriver.value
      ? ['-J', '-n', '1', '-d', 'sriov']
      : ['-J', '-n', '1', `sys:/sys/devices/pci0000:00/0000:${gpu.pci}`];

    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'intel_gpu_top',
        args: args,
        timeout: 2,
        parse_json: true,
      }),
    });

    if (res.ok) {
      const data = await res.json();
      gpuData.value[gpu.pci] = Array.isArray(data.output) ? data.output[0] : data.output;
    }
  } catch { }
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
  const interval = Math.max(2, settings.value.interval) * 1000;
  pollInterval = setInterval(fetchAllGpuData, interval);
};

const stopPolling = () => {
  if (pollInterval) {
    clearInterval(pollInterval);
    pollInterval = null;
  }
};

onMounted(async () => {
  try {
    await checkSriovDriver();
    await Promise.all([fetchSettings(), fetchAvailableGpus()]);
    if (isConfigured.value) {
      startPolling();
    }
  } catch { }
  finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  stopPolling();
});
</script>
