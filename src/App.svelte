<script lang="ts">
  import * as Plot from "@observablehq/plot";
  import APlot from "./lib/Plot.svelte";

  import initialPop from "./data/initial_pop.json";
  import fertilityRates from "./data/fertility.json";
  import mortality from "./data/mortality_rates.json";
  import migration from "./data/net_migration.json";

  type PopRecord = {
    male: number;
    female: number;
  };

  let tfr = $state(1.45);
  let mortalityImprovement = $state(0.011);
  let netMigrationRate = $state(300000);
  let simulationSteps = $state(100);

  let fertility = $derived.by(() => {
    const currentTFR = fertilityRates.reduce((t, n) => t + n, 0);
    const factor = tfr / currentTFR;

    return fertilityRates.map((f) => f * factor);
  });

  let netMigration = $derived.by(() => {
    const total = migration
      .map((r) => r.male + r.female)
      .reduce((t, n) => t + n, 0);
    const factor = netMigrationRate / total;

    return migration.map((r) => {
      return { male: r.male * factor, female: r.female * factor };
    });
  });

  type SimulationSummary = {
    year: number;
    pop: PopRecord[];
    births: number;
    deaths: number;
    netMigration: number;
  };

  let populationHistory: SimulationSummary[] = $derived.by(() => {
    const history = [
      {
        year: 2022,
        pop: initialPop,
        births: 0,
        deaths: 0,
        netMigration: netMigrationRate,
      },
    ];

    for (let t = 0; t < simulationSteps; t++) {
      const current = history[history.length - 1];
      const mort = mortality.map((r) => {
        return {
          male: r.male * Math.exp(-mortalityImprovement * t),
          female: r.female * Math.exp(-mortalityImprovement * t),
        };
      });

      const nextPop = [];

      const births = calculateBirths(current.pop.map((r) => r.female));
      nextPop.push({ male: births / 2, female: births / 2 });

      let totalDeaths = 0;
      for (let age = 0; age < current.pop.length - 1; age++) {
        const maleDeath = current.pop[age].male * mort[age].male;
        const femaleDeath = current.pop[age].female * mort[age].female;

        totalDeaths += maleDeath + femaleDeath;

        nextPop.push({
          male: current.pop[age].male - maleDeath,
          female: current.pop[age].female - femaleDeath,
        });
      }

      for (let age = 0; age < netMigration.length; age++) {
        nextPop[age].male += netMigration[age].male;
        nextPop[age].female += netMigration[age].female;
      }

      history.push({
        year: current.year + 1,
        pop: nextPop,
        births: births,
        deaths: totalDeaths,
        netMigration: netMigrationRate,
      });
    }

    return history;
  });

  let totalPopulation = $derived.by(() =>
    populationHistory.map((r, t) => {
      const total = r.pop.reduce((t, n) => t + n.male + n.female, 0);
      return {
        year: r.year,
        pop: total,
      };
    }),
  );

  let populationByAge = $derived(
    populationHistory.flatMap((r) => {
      const children = r.pop
        .slice(0, 15)
        .reduce((t, n) => t + n.male + n.female, 0);

      const workingAge = r.pop
        .slice(15, 65)
        .reduce((t, n) => t + n.male + n.female, 0);
      const elderly = r.pop
        .slice(65)
        .reduce((t, n) => t + n.male + n.female, 0);

      return [
        { year: r.year, ageGroup: "Children (0-14)", pop: children },
        { year: r.year, ageGroup: "Working Age (15-64)", pop: workingAge },
        { year: r.year, ageGroup: "Elderly (65+)", pop: elderly },
      ];
    }),
  );

  let oldAgeDependency = $derived(
    populationHistory.flatMap((r) => {
      const children = r.pop
        .slice(0, 15)
        .reduce((t, n) => t + n.male + n.female, 0);
      const workingAge = r.pop
        .slice(15, 65)
        .reduce((t, n) => t + n.male + n.female, 0);
      const elderly = r.pop
        .slice(65)
        .reduce((t, n) => t + n.male + n.female, 0);

      return [
        { year: r.year, group: "Old age (65+)", ratio: elderly / workingAge },
        { year: r.year, group: "Youth (<16)", ratio: children / workingAge },
      ];
    }),
  );

  let populationChange = $derived(
    populationHistory.slice(1).flatMap((r) => {
      let naturalChange =
        r.births > r.deaths
          ? {
              year: r.year,
              category: "Natural Change (births > deaths)",
              change: r.births - r.deaths,
            }
          : {
              year: r.year,
              category: "Natural Change (births < deaths)",
              change: r.births - r.deaths,
            };

      return [
        naturalChange,
        { year: r.year, category: "Net migration", change: r.netMigration },
      ];
    }),
  );

  function calculateBirths(femalePop: number[]): number {
    let births = 0;
    for (let i = 0; i < 30; i++) {
      births += femalePop[i + 15] * fertility[i];
    }
    return births;
  }

  const plots = [
    "Population by age group",
    "Dependency ratios",
    "Population change",
  ];
  let selected = $state("Population by age group");

  let populationChangePlotOptions = $derived({
    marginLeft: 40,
    marginBottom: 40,
    width: 800,
    height: 500,
    color: { legend: true },
    x: {
      label: "Year",
      tickFormat: (d: number) => d.toString(),
      domain: [2023, 2023 + simulationSteps - 1],
      ticks: Math.floor(simulationSteps / 5),
    },
    y: {
      label: "Thousands of people",
      transform: (d: number) => d / 1_000,
      domain: [
        -1_000,
        (populationHistory
          .map((r) => r.births + r.netMigration)
          .reduce((t, n) => Math.max(t, n), 0) /
          1_000) *
          1.1,
      ],
    },
    marks: [
      Plot.rectY(populationChange, {
        x: "year",
        interval: 1,
        fill: "category",
        y: "change",
      }),
      Plot.lineY(
        populationHistory.slice(1).map((r) => {
          return { total: r.births + r.netMigration - r.deaths, ...r };
        }),
        {
          x: "year",
          y: "total",
        },
      ),
    ],
  });

  let oldAgePlotOptions = $derived({
    marginLeft: 40,
    marginBottom: 40,
    width: 800,
    height: 500,
    color: { legend: true },
    x: {
      label: "Year",
      tickFormat: (d: number) => d.toString(),
      domain: [2022, 2022 + simulationSteps],
      ticks: Math.floor(simulationSteps / 5),
    },
    y: {
      label: "Ratio",
      domain: [0, 1],
      tickFormat: (d: number) => (d * 100).toFixed(0) + "%",
    },
    marks: [
      Plot.lineY(oldAgeDependency, {
        x: "year",
        stroke: "group",
        y: "ratio",
      }),
    ],
  });

  let popPlotOptions = $derived({
    color: {
      legend: true,
      type: "categorical",
    },
    marginLeft: 40,
    marginBottom: 40,
    width: 800,
    height: 500,
    x: {
      label: "Year",
      tickFormat: (d: number) => d.toString(),
      domain: [2022, 2022 + simulationSteps],
      ticks: Math.floor(simulationSteps / 5),
    },
    y: {
      label: "Population (millions)",
      domain: [
        0,
        (totalPopulation.reduce((t, n) => Math.max(t, n.pop), 0) / 1_000_000) *
          1.1,
      ],
      transform: (d: number) => d / 1_000_000,
      ticks: 10,
    },
    marks: [
      Plot.areaY(populationByAge, {
        x: "year",
        fill: "ageGroup",
        y: "pop",
      }),
    ],
  });
</script>

<main>
  <div id="main">
    <div id="params">
      <h3>Simulation parameters</h3>
      <label for="tfr">Total fertility rate: {tfr.toFixed(2)}</label>
      <input
        id="tfr"
        type="range"
        min="0"
        max="5"
        step="0.01"
        bind:value={tfr}
      />

      <label for="mort"
        >Rate of improvement in mortality: {(
          mortalityImprovement * 100
        ).toFixed(1)}%</label
      >
      <input
        id="mort"
        type="range"
        min="0"
        max="0.05"
        step="0.001"
        bind:value={mortalityImprovement}
      />

      <label for="migration"
        >Net migration rate: {netMigrationRate.toLocaleString("en-gb")}</label
      >
      <input
        type="range"
        min="0"
        max="1000000"
        step="10000"
        bind:value={netMigrationRate}
      />
      <button
        onclick={() => {
          netMigrationRate = 300000;
          tfr = 1.45;
          mortalityImprovement = 0.011;
        }}>Reset</button
      >
    </div>

    <div id="plot">
      <label for="plot-type">Select a plot from the dropdown:</label>
      <select id="plot-type" bind:value={selected}>
        {#each plots as plot}
          <option value={plot}>{plot}</option>
        {/each}
      </select>

      {#if selected == "Population by age group"}
        <h3>{selected}</h3>
        <APlot options={popPlotOptions} />
      {:else if selected == "Dependency ratios"}
        <h3>{selected}</h3>
        <APlot options={oldAgePlotOptions} />
      {:else if selected == "Population change"}
        <h3>{selected}</h3>
        <APlot options={populationChangePlotOptions} />
      {/if}
    </div>
  </div>
</main>

<style>
  #params {
    display: flex;
    flex-direction: column;
  }

  #main {
    display: flex;
  }
</style>
