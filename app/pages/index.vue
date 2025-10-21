<script setup lang="ts">
type ScaleConfig = "greaterOrEqual_50_6" | "greaterOrEqual_40_15";
const scaleConfigOptions: { label: string; value: ScaleConfig }[] = [
  { label: "bis 10. Klasse", value: "greaterOrEqual_50_6" },
  {
    label: "ab 11. Klasse",
    value: "greaterOrEqual_40_15",
  },
];

const halfPointOptions = [
  { label: "Ja", value: true },
  { label: "Nein", value: false },
];

interface State {
  scaleConfig: ScaleConfig;
  minPercentToPass: number;
  output: ScaleOutput;
  maxPoints: number;
  halfPoints: boolean;
  numberOfSamples: number;
  pointsToGradeInput: string[];
}

const state: State = reactive({
  scaleConfig: "greaterOrEqual_50_6",
  minPercentToPass: 50,
  output: "grade",
  maxPoints: 20,
  halfPoints: true,
  numberOfSamples: 12,
  pointsToGradeInput: new Array(12).fill(""),
});

const pointsList = computed(() => {
  return state.pointsToGradeInput.map((value) =>
    parseFloat(value.replace(/,/g, "."))
  );
});

const gradesList = computed(() => {
  return pointsList.value.map((points) => {
    if (isNaN(points)) {
      return { value: NaN, label: "" };
    }

    const scaleItem = gradeScale.value.find((item) => {
      return points >= item.points;
    });

    const value =
      state.output === "grade" ? scaleItem!.grade : scaleItem!.gradePoints;
    let label = value + "";
    if (state.output === "grade") {
      if (scaleItem!.level === "max") {
        label = label + "+";
      } else if (scaleItem!.level === "min") {
        label = label + "-";
      }
    }

    return {
      value,
      label,
    };
  });
});

const filteredPointsList = computed(() => {
  return pointsList.value.filter((value) => !isNaN(value));
});

const progress = computed(() => {
  return (filteredPointsList.value.length / pointsList.value.length) * 100;
});

watch(
  () => state.scaleConfig,
  () => {
    switch (state.scaleConfig) {
      case "greaterOrEqual_50_6":
        state.minPercentToPass = 50;
        state.output = "grade";
        break;
      case "greaterOrEqual_40_15":
        state.minPercentToPass = 40;
        state.output = "gradePoints";
        break;
    }
  }
);

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
        ...new Array(newValue - oldValue).fill(""),
      ];
    }
  }
);

type ScaleOutput = "grade" | "gradePoints";

type GradeLevel = "max" | "mid" | "min";

const round = (
  value: number,
  lsb: number,
  fn: "round" | "ceil" | "floor" = "round"
) => {
  return Math[fn](value / lsb) * lsb;
};

const gradeScale = computed(() => {
  const { minPercentToPass } = state;
  const passGradesStep = (100 - minPercentToPass) / 4 / 3;
  const failGradesStep = minPercentToPass / 2 / 3;
  const levels: GradeLevel[] = ["max", "mid", "min"];
  const passGrades = [1, 2, 3, 4];
  const failGrades = [5, 6];

  const passLimits = new Array(4 * 3).fill(0).map((_, index) => {
    const percent = 100 - (index + 1) * passGradesStep;
    const grade = passGrades[Math.floor(index / 3)];
    const gradePoints = 15 - index;
    const level = levels[index % 3];
    const points = (percent / 100) * state.maxPoints;

    return {
      percent: round(percent, 0.001),
      grade,
      gradePoints,
      level,
      points: round(points, 0.001),
    };
  });

  const failLimits = new Array(2 * 3).fill(0).map((_, index) => {
    const percent = minPercentToPass - (index + 1) * failGradesStep;
    const grade = failGrades[Math.floor(index / 3)] || -1;
    const gradePoints = Math.max(0, 3 - index);
    const level = levels[index % 3];
    const points = (percent / 100) * state.maxPoints;

    return {
      percent: round(percent, 0.001),
      grade,
      gradePoints,
      level,
      points: round(points, 0.001),
    };
  });

  return [...passLimits, ...failLimits];
});

const gradeScaleData = computed(() => {
  const data: {
    grade: number;
    from: number;
    to: number;
    percent: string;
    points: string;
  }[] = [];

  const { halfPoints, maxPoints } = state;
  const lsb = halfPoints ? 0.5 : 1;
  let _prevTo = -1;
  let _prevPercent = -1;
  gradeScale.value.forEach((scaleItem) => {
    if (scaleItem.grade === undefined) {
      return;
    }

    const { grade, points, level, percent } = scaleItem;

    if (grade < 6 && level === "min") {
      let from = grade === 1 ? maxPoints : _prevTo - lsb;
      const to = round(points, lsb, "ceil");
      from = Math.max(from, to);
      _prevTo = to;
      _prevPercent = percent;
      data.push({
        grade,
        from,
        to,
        percent: `≥ ${percent.toFixed()}`,
        points: `≥ ${to}`,
      });
    } else if (grade === 6 && level === "max") {
      let from = _prevTo - lsb;
      const to = 0;
      from = Math.max(from, to);
      data.push({
        grade,
        from,
        to,
        percent: `< ${_prevPercent.toFixed()}`,
        points: `< ${_prevTo}`,
      });
    }
  });

  return data;
});

const gradePointsScaleData = computed(() => {
  const data: {
    gradePoints: number;
    from: number;
    to: number;
    percent: string;
    points: string;
  }[] = [];

  const { halfPoints, maxPoints } = state;
  const lsb = halfPoints ? 0.5 : 1;
  let _prevTo = -1;
  let _prevPercent = -1;
  gradeScale.value.forEach((scaleItem) => {
    if (scaleItem.gradePoints === undefined) {
      return;
    }

    const { gradePoints, points, percent, level } = scaleItem;

    if (gradePoints > 0) {
      let from = gradePoints === 15 ? maxPoints : _prevTo - lsb;
      const to = round(points, lsb, "ceil");
      from = Math.max(from, to);
      _prevTo = to;
      _prevPercent = percent;
      data.push({
        gradePoints,
        from,
        to,
        percent: `≥ ${percent.toFixed()}`,
        points: `≥ ${to}`,
      });
    } else if (level === "max") {
      let from = _prevTo - lsb;
      const to = 0;
      from = Math.max(from, to);
      data.push({
        gradePoints,
        from,
        to,
        percent: `< ${_prevPercent.toFixed()}`,
        points: `< ${_prevTo}`,
      });
    }
  });

  return data;
});

const bins = computed(() => {
  const numberOfBins = state.output === "grade" ? 6 : 16;
  let sum = 0;
  let n = 0;

  const data = new Array(numberOfBins).fill(-1).map((_, index) => {
    const value = state.output === "grade" ? index + 1 : 15 - index;
    const count = gradesList.value.filter(
      (grade) => grade.value === value
    ).length;
    sum += value * count;
    n += count;
    return {
      value,
      count,
    };
  });

  const avg = sum / n;

  return {
    avg,
    data,
  };
});

async function onSubmit() {
  console.log("submit");
}

function parsePointsInput(el: HTMLInputElement) {
  el.value = el.value.replace(/[^\d.,\r\n]/g, "");
  el.value = el.value.replace(/(\r\n|\r|\n){2,}/g, "\n");
  const parsedValue = Math.min(
    state.maxPoints,
    parseFloat(el.value.replace(/,/g, "."))
  );

  if (parsedValue === state.maxPoints) {
    el.value = state.maxPoints.toString();
  }
}

function onPointsInput(event: Event) {
  parsePointsInput(event.target as HTMLInputElement);
}
</script>

<template>
  <div class="py-8">
    <h1 class="text-2xl font-bold">Mein Notenrechner</h1>
    <UForm :state="state" class="space-y-4 mt-8" @submit="onSubmit">
      <div class="flex gap-x-4">
        <UFormField label="Bewertungsskala" name="minPercentageToPass">
          <USelect v-model="state.scaleConfig" :items="scaleConfigOptions" />
        </UFormField>
        <UFormField label="Maximalpunktzahl" name="totalPoints">
          <UInputNumber v-model="state.maxPoints" />
        </UFormField>
        <UFormField label="Halbe Punkte" name="halfPoints">
          <USelect v-model="state.halfPoints" :items="halfPointOptions" />
        </UFormField>
      </div>

      <UButton type="submit" class="sr-only"> Submit </UButton>
    </UForm>

    <div class="mt-12">
      <table class="w-full divide-y divide-gray-300 table-fixed text-center">
        <tbody class="divide-y divide-gray-200">
          <!-- Note -->
          <tr
            :class="{
              'divide-x divide-gray-200': state.output === 'gradePoints',
            }"
          >
            <RowHeaderCell class="pt-0 w-24"> Note </RowHeaderCell>
            <RowDataCell
              v-for="grade in 6"
              :key="grade"
              :colspan="state.output === 'gradePoints' && grade < 6 ? 3 : 1"
              class="pt-0"
            >
              {{ grade }}
            </RowDataCell>
          </tr>
          <!-- Notenpunkte -->
          <tr v-if="state.output === 'gradePoints'">
            <RowHeaderCell> Notenpunkte </RowHeaderCell>
            <RowDataCell v-for="count in 16" :key="count">
              {{ 16 - count }}
            </RowDataCell>
          </tr>
          <!-- Bereich in % -->
          <tr>
            <RowHeaderCell> In % </RowHeaderCell>
            <RowDataCell
              v-for="(item, index) in state.output === 'grade'
                ? gradeScaleData
                : gradePointsScaleData"
              :key="index"
            >
              {{ item.percent }}
            </RowDataCell>
          </tr>
          <!-- Bereich in Punkten -->
          <tr>
            <RowHeaderCell> In Punkten </RowHeaderCell>
            <RowDataCell
              v-for="(item, index) in state.output === 'grade'
                ? gradeScaleData
                : gradePointsScaleData"
              :key="index"
            >
              {{ item.points }}
            </RowDataCell>
          </tr>
        </tbody>
      </table>
    </div>
    <div class="flex mt-10 gap-x-36">
      <!-- Notenberechnung -->
      <div class="print:hidden">
        <h2 class="text-xl font-bold">Notenberechnung</h2>
        <UFormField
          label="Anzahl Prüflinge"
          name="numberOfSamples"
          class="mt-4"
        >
          <UInputNumber v-model="state.numberOfSamples" />
        </UFormField>
        <div class="mt-8">
          <UProgress v-model="progress" size="sm" />
          <p class="text-right text-sm whitespace-nowrap text-gray-500 mt-1">
            {{ filteredPointsList.length }} von {{ pointsList.length }}
          </p>
          <table class="divide-y divide-gray-300 table-fixed">
            <thead>
              <tr>
                <ColHeaderCell> # </ColHeaderCell>
                <ColHeaderCell class="text-center">
                  Erreichte Punktzahl
                </ColHeaderCell>
                <ColHeaderCell class="text-center"> Note </ColHeaderCell>
              </tr>
            </thead>
            <tbody class="divide-y divide-gray-200">
              <tr v-for="(s, index) in state.numberOfSamples" :key="s">
                <ColDataCell class="py-0!">
                  <div
                    class="rounded-md bg-gray-100 px-2 py-1 text-xs text-right font-medium text-gray-600 outline-1 outline-gray-200"
                  >
                    {{ s }}
                  </div>
                </ColDataCell>
                <ColDataCell class="py-0!">
                  <UInput
                    v-model="state.pointsToGradeInput[index]"
                    style="text-align: center"
                    variant="soft"
                    @input="(event: Event) => onPointsInput(event)"
                  />
                </ColDataCell>
                <ColDataCell class="py-0! text-center!">
                  {{ gradesList[index]!.label }}
                </ColDataCell>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
      <!-- Notenverteilung -->
      <div class="min-w-md grow">
        <h2 class="text-xl font-bold">Notenverteilung</h2>
        <div class="mt-4">
          <table class="w-full divide-y divide-gray-300 table-fixed">
            <tbody class="divide-y divide-gray-200">
              <tr>
                <RowHeaderCell scope="row"> Note </RowHeaderCell>
                <RowDataCell
                  v-for="item in bins.data"
                  :key="item.value"
                  class="text-center"
                >
                  {{ item.value }}
                </RowDataCell>
              </tr>
              <tr>
                <RowHeaderCell scope="row"> Anzahl </RowHeaderCell>
                <RowDataCell
                  v-for="item in bins.data"
                  :key="item.value"
                  class="text-center"
                >
                  {{ item.count || "--" }}
                </RowDataCell>
              </tr>
            </tbody>
            <tfoot>
              <tr>
                <RowDataCell
                  scope="row"
                  :colspan="bins.data.length + 1"
                  class="text-right pr-0 pb-0"
                >
                  Durchschnitt
                </RowDataCell>
              </tr>
              <tr>
                <td
                  :colspan="bins.data.length + 1"
                  class="pt-4 pr-4 pl-3 text-right text-xl font-semibold tracking-tight text-gray-900 sm:pr-0"
                >
                  {{
                    isNaN(bins.avg)
                      ? "--"
                      : round(bins.avg, 0.1).toFixed(1).replace(/\./g, ",")
                  }}
                </td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>
