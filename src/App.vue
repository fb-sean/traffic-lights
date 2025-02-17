<template>
  <div class="container mx-auto p-4 bg-gray-900/60 min-h-screen min-w-screen">
    <div class="max-w-4xl mx-auto">
      <div class="bg-gray-200 p-4 rounded-lg shadow-md mb-4">
        <h1 class="text-2xl font-bold mb-4 text-center">Ampelsteuerung</h1>
        <div class="grid grid-cols-2 gap-4 mb-4">
          <div class="flex items-center space-x-2">
            <span class="font-medium">Aktuelle Zeit:</span>
            <input
                type="time"
                v-model="currentTime"
                class="border rounded px-2 py-1"
                :disabled="!systemActive"
            >
            <span class="text-sm text-gray-600">
              {{ currentDateTime.toLocaleTimeString() }}
            </span>
          </div>
          <div class="flex items-center justify-end space-x-4">
            <label class="inline-flex items-center">
              <input
                  type="checkbox"
                  v-model="systemActive"
                  class="form-checkbox h-5 w-5 text-blue-600"
              >
              <span class="ml-2">System {{ systemActive ? 'EIN' : 'AUS' }}</span>
            </label>
          </div>
        </div>
        <div class="flex justify-between">
          <button
              @click="toggleMaintenance"
              :class="buttonClass(maintenanceMode, 'yellow')"
          >
            Wartungsmodus {{ maintenanceMode ? 'AUS' : 'EIN' }}
          </button>
          <button
              @click="triggerSystemError"
              :class="buttonClass(systemError, 'red')"
              :disabled="!systemActive || maintenanceMode"
          >
            Systemfehler {{ systemError ? 'Beheben' : 'Simulieren' }}
          </button>
        </div>
      </div>

      <div class="bg-gray-200 p-4 rounded-lg shadow-md">
        <div class="relative w-full" style="height: 500px;">
          <canvas
              ref="canvas"
              width="500"
              height="500"
              class="absolute top-0 left-0"
          ></canvas>

          <div class="absolute top-2 right-2 bg-white/90 p-2 rounded shadow">
            <p class="font-medium">Status: {{ currentStatus }}</p>
            <p class="text-sm">Richtung A: {{ stateA }}</p>
            <p class="text-sm">Richtung B: {{ stateB }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {ref, onMounted, onUnmounted, watch, computed} from 'vue';

const STATES = {
  RED: 'rot',
  RED_YELLOW: 'rotgelb',
  GREEN: 'grün',
  YELLOW: 'gelb',
  BLINK: 'blinken',
  OFF: 'aus'
};

export default {
  setup() {
    const canvas = ref(null);
    const ctx = ref(null);

    const systemActive = ref(false);
    const maintenanceMode = ref(false);
    const systemError = ref(false);

    const stateA = ref(STATES.RED);
    const stateB = ref(STATES.RED);
    const currentStatus = ref('inaktiv');

    let mainTimer = null;
    let blinkTimer = null;
    const isBlinkOn = ref(false);

    const trafficLightsA = [
      {x: 150, y: 100},
      {x: 350, y: 400}
    ];

    const trafficLightsB = [
      {x: 100, y: 150},
      {x: 400, y: 350}
    ];

    const currentDateTime = ref(new Date());
    const timeInterval = ref(null);

    const currentTime = computed({
      get: () => {
        const hours = currentDateTime.value.getHours().toString().padStart(2, '0');
        const minutes = currentDateTime.value.getMinutes().toString().padStart(2, '0');
        return `${hours}:${minutes}`;
      },
      set: (newValue) => {
        const [hours, minutes] = newValue.split(':').map(Number);
        const newDate = new Date(currentDateTime.value);
        newDate.setHours(hours, minutes, 0);
        currentDateTime.value = newDate;
        startClock();
      }
    });

    function startClock() {
      clearInterval(timeInterval.value);
      timeInterval.value = setInterval(() => {
        currentDateTime.value = new Date(currentDateTime.value.getTime() + 1000);
        drawIntersection();
      }, 1000);
    }

    onMounted(() => {
      ctx.value = canvas.value.getContext('2d');
      startClock();
      drawIntersection();
    });

    onUnmounted(() => {
      clearAllTimers();
    });

    function clearAllTimers() {
      clearTimeout(mainTimer);
      clearInterval(blinkTimer);
      clearInterval(timeInterval.value);
    }

    function drawIntersection() {
      if (!ctx.value) return;

      const context = ctx.value;

      context.fillStyle = '#2ba11f';
      context.fillRect(0, 0, 500, 500);

      context.fillStyle = '#666';
      context.fillRect(200, 0, 100, 500);
      context.fillRect(0, 200, 500, 100);

      context.strokeStyle = '#fff';
      context.setLineDash([10, 10]);
      context.beginPath();
      context.moveTo(250, 0);
      context.lineTo(250, 200);
      context.moveTo(250, 300);
      context.lineTo(250, 500);
      context.moveTo(0, 250);
      context.lineTo(200, 250);
      context.moveTo(300, 250);
      context.lineTo(500, 250);
      context.stroke();

      drawTrafficLights();
    }

    function drawTrafficLights() {
      trafficLightsA.forEach(light => drawTrafficLight(light.x, light.y, stateA.value));
      trafficLightsB.forEach(light => drawTrafficLight(light.x, light.y, stateB.value, Math.PI / 2));
    }

    function drawTrafficLight(x, y, state, rotation = 0) {
      const context = ctx.value;
      context.save();
      context.translate(x, y);
      context.rotate(rotation);

      context.fillStyle = '#222';
      context.fillRect(-15, -45, 30, 90);

      const lights = {
        [STATES.RED]: [true, false, false],
        [STATES.RED_YELLOW]: [true, true, false],
        [STATES.GREEN]: [false, false, true],
        [STATES.YELLOW]: [false, true, false],
        [STATES.OFF]: [false, false, false],
        [STATES.BLINK]: [false, isBlinkOn.value, false]
      }[state] || [false, false, false];

      const colors = ['#ff0000', '#ffff00', '#00ff00'];
      lights.forEach((isOn, i) => {
        context.beginPath();
        context.arc(0, -25 + i * 25, 10, 0, Math.PI * 2);
        context.fillStyle = isOn ? colors[i] : '#333';
        context.fill();
      });

      context.restore();
    }

    async function startTrafficCycle() {
      startClock();

      while (systemActive.value && !maintenanceMode.value && !systemError.value) {
        stateA.value = STATES.RED;
        stateB.value = STATES.RED;
        await delay(1000);

        stateA.value = STATES.RED_YELLOW;
        await delay(2000);

        stateA.value = STATES.GREEN;
        await delay(20000);

        stateA.value = STATES.YELLOW;
        await delay(3000);

        stateA.value = STATES.RED;
        await delay(1000);

        stateB.value = STATES.RED_YELLOW;
        await delay(2000);

        stateB.value = STATES.GREEN;
        await delay(20000);

        stateB.value = STATES.YELLOW;
        await delay(3000);
      }
    }

    function delay(ms) {
      return new Promise(resolve => {
        mainTimer = setTimeout(resolve, ms);
      });
    }

    function startMaintenanceMode() {
      clearAllTimers();
      blinkTimer = setInterval(() => {
        isBlinkOn.value = !isBlinkOn.value;
        stateA.value = STATES.BLINK;
        stateB.value = STATES.BLINK;
        drawIntersection();
      }, 500);
    }

    watch(systemActive, (newValue) => {
      clearAllTimers();
      if (newValue) {
        currentStatus.value = 'aktiv';
        maintenanceMode.value = false;
        systemError.value = false;
        startTrafficCycle();
      } else {
        currentStatus.value = 'inaktiv';
        stateA.value = STATES.OFF;
        stateB.value = STATES.OFF;
      }
      drawIntersection();
    });

    watch([maintenanceMode, systemError], ([newMaintenance, newError]) => {
      clearAllTimers();
      startClock();
      if (newMaintenance) {
        currentStatus.value = 'Wartungsmodus';
        startMaintenanceMode();
      } else if (newError) {
        currentStatus.value = 'Systemfehler';
        startMaintenanceMode();
      } else if (systemActive.value) {
        currentStatus.value = 'aktiv';
        startTrafficCycle();
      }
    });

    watch(currentDateTime, (newDateTime) => {
      const hours = newDateTime.getHours();
      if (hours >= 20 || hours < 6) {
        if (systemActive.value && !maintenanceMode.value) {
          maintenanceMode.value = true;
        }
      } else if (maintenanceMode.value && systemActive.value) {
        maintenanceMode.value = false;
      }
    });

    function toggleMaintenance() {
      if (!maintenanceMode.value) {
        systemActive.value = false;
      }

      maintenanceMode.value = !maintenanceMode.value;

      if (!maintenanceMode.value) {
        currentStatus.value = 'inaktiv';
        isBlinkOn.value = false;
        stateA.value = STATES.OFF;
        stateB.value = STATES.OFF;
        drawIntersection();
      }
    }

    function triggerSystemError() {
      systemError.value = !systemError.value;
    }

    function buttonClass(active, color) {
      return `px-4 py-2 rounded-lg font-medium transition-colors ${
          (
              active
                  ? `bg-${color}-600 hover:bg-${color}-700 text-gray-700`
                  : 'bg-gray-400/60 hover:bg-gray-300 text-gray-700'
          )
      }`;
    }

    watch([stateA, stateB], () => {
      drawIntersection();
    });

    return {
      canvas,
      systemActive,
      maintenanceMode,
      systemError,
      currentTime,
      currentDateTime,
      currentStatus,
      stateA,
      stateB,
      toggleMaintenance,
      triggerSystemError,
      buttonClass
    };
  }
};
</script>