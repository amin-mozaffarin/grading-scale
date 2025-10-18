<script setup lang="ts">
const scaleModes = [
  { label: "bis 10. Klasse", value: 50 },
  { label: "ab 11. Klasse", value: 40 },
];

const state = reactive({
  totalPoints: 20,
  minPercentageToPass: 50,
  halfPoints: true,
  numberOfSamples: 12,
  pointsToGradeInput: new Array(12).fill(null),
});

watch(
  () => state.numberOfSamples,
  (newValue, oldValue) => {
    if (newValue < oldValue) {
      state.pointsToGradeInput = state.pointsToGradeInput.slice(
        0,
        newValue - oldValue
      );
    } else {
      state.pointsToGradeInput = [
        ...state.pointsToGradeInput,
        ...new Array(newValue - oldValue).fill(null),
      ];
    }
  }
);

const grades = ref([1, 2, 3, 4, 5, 6]);

const bins = computed(() => {
  const data = new Array(grades.value.length).fill(0);
  let sum = 0;
  let n = 0;

  gradesList.value.forEach((grade) => {
    if (grade === null) {
      return;
    }

    n += 1;

    data[grade - 1] = data[grade - 1] + 1;
    sum += grade;
  });

  const avg = sum / n;

  return {
    data,
    avg,
  };
});

const resolution = computed(() => {
  return state.halfPoints ? 0.5 : 1.0;
});

const limits = computed(() => {
  const delta = (100 - state.minPercentageToPass) / 4;

  let relative = [
    100 - 1 * delta,
    100 - 2 * delta,
    100 - 3 * delta,
    state.minPercentageToPass,
    state.minPercentageToPass / 2,
  ];

  relative = relative.map((value) => Math.round(value * 100) / 100); // 2 digits

  const absolute = relative.map((r) => {
    const from = Math.round((r / 100) * state.totalPoints * 100) / 100; // 2 digits
    const discreteFrom = Math.ceil(from / resolution.value) * resolution.value;
    return discreteFrom;
  });

  return {
    relative,
    absolute,
  };
});

const pointIntervals = computed(() => {
  return new Array(6).fill(null).map((_, index) => {
    let from = 0;
    let to = 0;

    if (index === 0) {
      from = limits.value.absolute[0]!;
      to = state.totalPoints;
    } else if (index === 5) {
      from = 0;
      to = limits.value.absolute[4]! - resolution.value;
    } else {
      from = limits.value.absolute[index]!;
      to = limits.value.absolute[index - 1]! - resolution.value;
    }

    return {
      from,
      to,
      range: to - from,
    };
  });
});

const data = computed(() => {
  const _data = new Array(6).fill(null).map((_, index) => {
    const grade = index + 1;
    const relativeLimit =
      index < 5
        ? `≥ ${limits.value.relative[index]!.toLocaleString()}%`
        : `< ${limits.value.relative[index - 1]!.toLocaleString()}%`;

    return {
      grade,
      relativeLimit,
    };
  });

  return _data;
});

const pointsList = computed(() => {
  return state.pointsToGradeInput.map((_input) => {
    return _input === null ? null : parseFloat(_input.replace(/,/g, "."));
  });
});

const gradesList = computed(() => {
  const grades = pointsList.value.map((_points) => {
    console.debug(pointsList.value);
    if (_points === null || isNaN(_points)) {
      return null;
    }

    const index = limits.value.absolute.findIndex((limit) => {
      return limit <= _points;
    });
    return index === -1 ? 6 : index + 1;
  });

  return grades;
});

async function onSubmit() {
  console.log("submit");
}

function onInput(event: Event) {
  const el = event.target as HTMLInputElement;
  el.value = el.value.replace(/[^\d.,\r\n]/g, "");
  el.value = el.value.replace(/(\r\n|\r|\n){2,}/g, "\n");
  const _value = parseFloat(el.value.replace(/,/g, "."));
  if (_value >= state.totalPoints) {
    el.value = state.totalPoints.toLocaleString();
  }
}
</script>

<template>
  <div>
    <h1 class="text-2xl font-bold mt-8">Bewertungsschlüssel</h1>
    <UForm :state="state" class="space-y-4 mt-8" @submit="onSubmit">
      <div class="flex gap-x-4">
        <UFormField label="Bewertungsskala" name="minPercentageToPass">
          <USelect v-model="state.minPercentageToPass" :items="scaleModes" />
        </UFormField>
        <UFormField label="Bewertungseinheiten (BE)" name="totalPoints">
          <UInputNumber v-model="state.totalPoints" />
        </UFormField>
        <UFormField label="&nbsp;">
          <USwitch v-model="state.halfPoints" label="Halbe BE" class="pt-1.5" />
        </UFormField>
      </div>

      <UButton type="submit" class="sr-only"> Submit </UButton>
    </UForm>

    <div class="mt-8">
      <table class="relative min-w-full divide-y divide-gray-300 table-fixed">
        <tbody class="divide-y divide-gray-200">
          <!-- Note -->
          <tr>
            <th
              scope="row"
              class="w-1/7 text-left py-4 pr-3 pl-4 text-sm font-medium whitespace-nowrap text-gray-900 sm:pl-0"
            >
              Note
            </th>
            <td
              v-for="grade in grades"
              :key="grade"
              class="w-1/7 px-3 py-4 text-sm whitespace-nowrap text-gray-500"
            >
              {{ grade }}
            </td>
          </tr>
          <!-- In % -->
          <tr>
            <th
              scope="row"
              class="text-left py-4 pr-3 pl-4 text-sm font-medium whitespace-nowrap text-gray-900 sm:pl-0"
            >
              Bereich in %
            </th>
            <td
              v-for="(item, index) in data"
              :key="index"
              class="px-3 py-4 text-sm whitespace-nowrap text-gray-500"
            >
              {{ item.relativeLimit }}
            </td>
          </tr>
          <!-- In BE -->
          <tr>
            <th
              scope="row"
              class="text-left py-4 pr-3 pl-4 text-sm font-medium whitespace-nowrap text-gray-900 sm:pl-0"
            >
              Bereich in BE
            </th>
            <td
              v-for="(interval, index) in pointIntervals"
              :key="index"
              class="px-3 py-4 text-sm whitespace-nowrap text-gray-500"
            >
              {{ interval.to.toLocaleString() }} ..
              {{ interval.from.toLocaleString() }}
            </td>
          </tr>
          <!-- In BE -->
          <tr>
            <th
              scope="row"
              class="text-left py-4 pr-3 pl-4 text-sm font-medium whitespace-nowrap text-gray-900 sm:pl-0"
            >
              Spanne in BE
            </th>
            <td
              v-for="(interval, index) in pointIntervals"
              :key="index"
              class="px-3 py-4 text-sm whitespace-nowrap text-gray-500"
            >
              {{ interval.range.toLocaleString() }}
            </td>
          </tr>
        </tbody>
      </table>
      <!-- Notenberechnung -->
      <h2 class="mt-8">Notenberechnung und Notenspiegel</h2>
      <UFormField label="Anzahl Prüflinge" name="numberOfSamples">
        <UInputNumber v-model="state.numberOfSamples" />
      </UFormField>
      <!--
      <div>
        <UTextarea v-model="state.pointsToGradeInput" @input="onInput" />
        <UTextarea v-model="pointsToGradeOutput" disabled />
      </div>
      -->
      <div class="mt-8">
        <table class="relative max-w-1/4 divide-y divide-gray-300 table-fixed">
          <thead>
            <tr>
              <th
                scope="col"
                class="py-3.5 pr-3 pl-4 text-left text-sm font-semibold whitespace-nowrap text-gray-900 sm:pl-0"
              >
                #
              </th>
              <th
                scope="col"
                class="px-2 py-3.5 text-left text-sm font-semibold whitespace-nowrap text-gray-900"
              >
                Erreichte BE
              </th>
              <th
                scope="col"
                class="px-2 py-3.5 text-left text-sm font-semibold whitespace-nowrap text-gray-900"
              >
                Note
              </th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200">
            <tr v-for="s in state.numberOfSamples" :key="s">
              <td
                class="py-2 pr-3 pl-4 text-sm whitespace-nowrap text-gray-500 sm:pl-0"
              >
                <div
                  class="rounded-md bg-gray-100 px-2 py-1 text-xs text-right font-medium text-gray-600 outline-1 outline-gray-200"
                >
                  {{ s }}
                </div>
              </td>
              <td class="px-2 py-2 text-sm whitespace-nowrap text-gray-500">
                <UInput
                  v-model="state.pointsToGradeInput[s - 1]"
                  variant="soft"
                  @input="onInput"
                />
              </td>
              <td class="px-2 py-2 text-sm whitespace-nowrap text-gray-500">
                {{ gradesList[s - 1] }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <p>
        {{ bins }}
      </p>
      <p>{{ pointsList }}</p>
      <p>{{ gradesList }}</p>
    </div>
  </div>
</template>
