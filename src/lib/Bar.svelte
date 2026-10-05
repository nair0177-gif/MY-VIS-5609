<script lang="ts">
  import * as d3 from "d3";
  import type { TMovie } from "../types";

  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };

  let {
    movies,
    progress = 100,
    width = 800,
    height = 650
  }: Props = $props();

  let selectedGenre = $state<string | undefined>(undefined);

  // ------------------------------------------------------------
  // YEAR RANGE
  // ------------------------------------------------------------

  const yearRange = $derived(
    d3.extent(movies.map((d) => d.year))
  );

  function getUpYear(
    range: [Date | undefined, Date | undefined]
  ) {
    if (!range[0] || !range[1]) {
      return new Date();
    }

    const timeScale = d3
      .scaleTime()
      .domain(range as [Date, Date])
      .range([0, 100]);

    return timeScale.invert(progress);
  }

  const upYear = $derived(
    getUpYear(yearRange)
  );

  // ------------------------------------------------------------
  // COUNT GENRES
  // ------------------------------------------------------------

  function getGenreNums(
    movieData: TMovie[],
    cutoffYear: Date
  ) {
    const result: Record<string, number> = {};

    movieData
      .filter((movie) => movie.year <= cutoffYear)
      .forEach((movie) => {
        movie.genres.forEach((genre) => {
          // Ignore missing genre values
          if (!genre || genre === "NA") return;

          result[genre] =
            (result[genre] || 0) + 1;
        });
      });

    return result;
  }

  const genreNums = $derived(
    getGenreNums(movies, upYear)
  );

  // ------------------------------------------------------------
  // SORT GENRES
  // ------------------------------------------------------------

  const sortedGenres = $derived(
    Object.entries(genreNums)
      .sort((a, b) => b[1] - a[1])
  );

  // ------------------------------------------------------------
  // LIMIT DISPLAY
  //
  // Showing every genre makes the chart unnecessarily long.
  // The full dataset is still used for counting.
  // ------------------------------------------------------------

  const displayedGenres = $derived(
    sortedGenres.slice(0, 20)
  );

  // ------------------------------------------------------------
  // MARGINS
  // ------------------------------------------------------------

  const margin = {
    top: 20,
    right: 70,
    bottom: 45,
    left: 120
  };

  const usableWidth =
    width - margin.left - margin.right;

  const usableHeight =
    height - margin.top - margin.bottom;

  // ------------------------------------------------------------
  // SCALES
  // ------------------------------------------------------------

  const xScale = $derived(
    d3
      .scaleLinear()
      .domain([
        0,
        d3.max(
          displayedGenres,
          ([, count]) => count
        ) ?? 1
      ])
      .nice()
      .range([0, usableWidth])
  );

  const yScale = $derived(
    d3
      .scaleBand<string>()
      .domain(
        displayedGenres.map(([genre]) => genre)
      )
      .range([0, usableHeight])
      .padding(0.25)
  );

  // ------------------------------------------------------------
  // AXIS
  // ------------------------------------------------------------

  let xAxis: SVGGElement;

  function updateAxis() {
    if (!xAxis) return;

    d3.select(xAxis).call(
      d3
        .axisBottom(xScale)
        .ticks(6)
        .tickFormat(d3.format("d"))
    );
  }

  $effect(() => {
    updateAxis();
  });
</script>

<h2>
  Genre Distribution
</h2>

<p class="subtitle">
  Number of movies containing each genre
  {#if yearRange[0] && yearRange[1]}
    ({yearRange[0].getFullYear()}–{upYear.getFullYear()})
  {/if}
</p>

{#if displayedGenres.length > 0}

  <div class="chart-wrapper">

    <svg {width} {height}>

      <!-- -------------------------------------------------- -->
      <!-- CHART AREA -->
      <!-- -------------------------------------------------- -->

      <g
        transform={`translate(${margin.left}, ${margin.top})`}
      >

        <!-- X-axis -->

        <g
          transform={`translate(0, ${usableHeight})`}
          bind:this={xAxis}
        />

        <!-- Bars -->

        {#each displayedGenres as [genre, count]}

          <g
            class="bar-group"
            onmouseenter={() => {
              selectedGenre = genre;
            }}
            onmouseleave={() => {
              selectedGenre = undefined;
            }}
          >

            <!-- Genre label -->

            <text
              x={-12}
              y={
                (yScale(genre) ?? 0) +
                yScale.bandwidth() / 2
              }
              text-anchor="end"
              dominant-baseline="middle"
              class="genre-label"
              opacity={
                selectedGenre === undefined ||
                selectedGenre === genre
                  ? 1
                  : 0.35
              }
            >
              {genre}
            </text>

            <!-- Bar -->

            <rect
              x={0}
              y={yScale(genre)}
              width={xScale(count)}
              height={yScale.bandwidth()}
              class="bar"
              class:selected={
                selectedGenre === genre
              }
              opacity={
                selectedGenre === undefined ||
                selectedGenre === genre
                  ? 1
                  : 0.3
              }
            />

            <!-- Value -->

            <text
              x={xScale(count) + 8}
              y={
                (yScale(genre) ?? 0) +
                yScale.bandwidth() / 2
              }
              dominant-baseline="middle"
              class="value"
              opacity={
                selectedGenre === undefined ||
                selectedGenre === genre
                  ? 1
                  : 0.3
              }
            >
              {count}
            </text>

          </g>

        {/each}

      </g>

      <!-- X-axis title -->

      <text
        x={margin.left + usableWidth / 2}
        y={height - 5}
        text-anchor="middle"
        class="axis-title"
      >
        Number of Movies
      </text>

    </svg>

  </div>

  <!-- Hover information -->

  {#if selectedGenre}

    {@const selectedCount =
      genreNums[selectedGenre] ?? 0}

    <div class="tooltip">
      <strong>{selectedGenre}</strong>
      <span>{selectedCount} movies</span>
    </div>

  {/if}

{/if}

<style>
  h2 {
    margin-bottom: 4px;
  }

  .subtitle {
    margin-top: 0;
    margin-bottom: 20px;
    font-size: 16px;
  }

  .chart-wrapper {
    overflow-x: auto;
  }

  svg {
    display: block;
    overflow: visible;
  }

  .bar {
    fill: #4c78a8;
    transition:
      opacity 0.15s ease,
      width 0.2s ease;
  }

  .bar:hover,
  .bar.selected {
    fill: #2f5597;
  }

  .bar-group {
    cursor: pointer;
  }

  .genre-label {
    font-size: 14px;
    transition: opacity 0.15s ease;
  }

  .value {
    font-size: 13px;
    font-weight: 600;
    transition: opacity 0.15s ease;
  }

  .axis-title {
    font-size: 14px;
    font-weight: 600;
  }

  :global(.tick text) {
    font-size: 12px;
  }

  .tooltip {
    display: flex;
    gap: 12px;
    align-items: center;
    margin-top: 8px;
    padding: 8px 12px;
    width: fit-content;
    border: 1px solid #ddd;
    border-radius: 5px;
    background: #f8f8f8;
    font-size: 14px;
  }

  .tooltip span {
    color: #555;
  }
</style>