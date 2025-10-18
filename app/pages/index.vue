<script setup lang="ts">
const scaleModes = [
  { label: "bis 10. Klasse", value: 50 },
  { label: "ab 11. Klasse", value: 40 },
];

const state = reactive({
  totalPoints: 20,
  minPercentageToPass: 50,
  halfPoints: true,
});

const grades = ref([1, 2, 3, 4, 5, 6]);

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
      to = limits.value.absolute[4]!;
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

async function onSubmit() {
  console.log("submit");
}
</script>

<template>
  <div>
    <UForm :state="state" class="space-y-4 mt-8" @submit="onSubmit">
      <div class="flex gap-x-4">
        <UFormField label="Bewertungsskala" name="minPercentageToPass">
          <USelect v-model="state.minPercentageToPass" :items="scaleModes" />
        </UFormField>
        <UFormField label="Bewertungseinheiten (BE)" name="totalPoints">
          <UInput v-model.number="state.totalPoints" />
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
    </div>
  </div>
</template>
