<script lang="ts">
  import * as d3 from "d3";
  import type { TMovie } from "../types";

  type Props = {
    movies: TMovie[];
    width?: number;
    height?: number;
  };

  let {
    movies,
    width = 1100,
    height = 600
  }: Props = $props();

  // ------------------------------------------------------------
  // MARGINS
  // ------------------------------------------------------------

  const margin = {
    top: 60,
    right: 180,
    bottom: 75,
    left: 70
  };

  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  // ------------------------------------------------------------
  // ALL AVAILABLE YEARS
  // ------------------------------------------------------------

  const allYears = $derived(
    Array.from(
      new Set(
        movies.map((movie) =>
          movie.year.getFullYear()
        )
      )
    ).sort((a, b) => a - b)
  );

  const minYear = $derived(
    allYears.length > 0 ? allYears[0] : 1900
  );

  const maxYear = $derived(
    allYears.length > 0
      ? allYears[allYears.length - 1]
      : 2023
  );

  // ------------------------------------------------------------
  // SELECTED YEAR RANGE
  // ------------------------------------------------------------

  let startYear = $state(minYear);
  let endYear = $state(maxYear);

  // ------------------------------------------------------------
  // TOP 3 GENRES FOR EACH YEAR
  // ------------------------------------------------------------

  const rankedData = $derived.by(() => {
    const yearly = d3.rollup(
      movies,
      (yearMovies) => {
        const counts = new Map<string, number>();

        yearMovies.forEach((movie) => {
          movie.genres.forEach((genre) => {
            if (!genre || genre === "NA") return;

            counts.set(
              genre,
              (counts.get(genre) ?? 0) + 1
            );
          });
        });

        return Array.from(counts.entries())
          .sort((a, b) => b[1] - a[1])
          .slice(0, 3)
          .map(([genre, count], index) => ({
            genre,
            count,
            rank: index + 1
          }));
      },
      (movie) => movie.year.getFullYear()
    );

    return Array.from(yearly.entries())
      .sort((a, b) => a[0] - b[0])
      .flatMap(([year, genres]) =>
        genres.map((item) => ({
          year,
          ...item
        }))
      );
  });

  // ------------------------------------------------------------
  // DATA WITHIN SELECTED RANGE
  // ------------------------------------------------------------

  const visibleData = $derived(
    rankedData.filter(
      (d) =>
        d.year >= startYear &&
        d.year <= endYear
    )
  );

  // ------------------------------------------------------------
  // VISIBLE YEARS
  // ------------------------------------------------------------

  const years = $derived(
    Array.from(
      new Set(
        visibleData.map((d) => d.year)
      )
    ).sort((a, b) => a - b)
  );

  // ------------------------------------------------------------
  // GENRES IN VISIBLE RANGE
  // ------------------------------------------------------------

  const genres = $derived(
    Array.from(
      new Set(
        visibleData.map((d) => d.genre)
      )
    ).sort()
  );

  // ------------------------------------------------------------
  // SELECTED GENRE
  // ------------------------------------------------------------

  let selectedGenre = $state<string | undefined>(
    undefined
  );

  // ------------------------------------------------------------
  // X SCALE
  // ------------------------------------------------------------

  const xScale = $derived(
    d3
      .scalePoint<number>()
      .domain(years)
      .range([0, innerWidth])
      .padding(0.5)
  );

  // ------------------------------------------------------------
  // Y SCALE
  // ------------------------------------------------------------

  const yScale = $derived(
    d3
      .scalePoint<number>()
      .domain([1, 2, 3])
      .range([0, innerHeight])
  );

  // ------------------------------------------------------------
  // COLOR SCALE
  // ------------------------------------------------------------

  const colorScale = $derived(
    d3
      .scaleOrdinal<string>()
      .domain(genres)
      .range(d3.schemeTableau10)
  );

  // ------------------------------------------------------------
  // YEAR LABELS
  // ------------------------------------------------------------

  const labeledYears = $derived.by(() => {
    if (years.length <= 12) {
      return years;
    }

    const interval =
      years.length > 40 ? 5 : 2;

    return years.filter(
      (year, index) =>
        index === 0 ||
        index === years.length - 1 ||
        year % interval === 0
    );
  });

  // ------------------------------------------------------------
  // GENRE LINE SEGMENTS
  // ------------------------------------------------------------

  const genreSegments = $derived.by(() => {
    return genres.flatMap((genre) => {
      const points = visibleData
        .filter((d) => d.genre === genre)
        .sort((a, b) => a.year - b.year);

      const segments: {
        genre: string;
        points: typeof points;
      }[] = [];

      let current: typeof points = [];

      points.forEach((point, index) => {
        if (index === 0) {
          current = [point];
          return;
        }

        const previous = points[index - 1];

        if (point.year === previous.year + 1) {
          current.push(point);
        } else {
          if (current.length >= 2) {
            segments.push({
              genre,
              points: current
            });
          }

          current = [point];
        }
      });

      if (current.length >= 2) {
        segments.push({
          genre,
          points: current
        });
      }

      return segments;
    });
  });

  // ------------------------------------------------------------
  // LINE PATH
  // ------------------------------------------------------------

  function linePath(
    points: {
      year: number;
      rank: number;
    }[]
  ) {
    return d3
      .line<{
        year: number;
        rank: number;
      }>()
      .x((d) => xScale(d.year) ?? 0)
      .y((d) => yScale(d.rank) ?? 0)
      .curve(d3.curveLinear)(points);
  }

  // ------------------------------------------------------------
  // INTERACTION HELPERS
  // ------------------------------------------------------------

  function genreOpacity(genre: string) {
    if (!selectedGenre) return 0.8;

    return selectedGenre === genre
      ? 1
      : 0.12;
  }

  function pointRadius(genre: string) {
    return selectedGenre === genre ? 7 : 4;
  }

  function updateStartYear(value: number) {
    startYear = Math.min(
      value,
      endYear - 1
    );
  }

  function updateEndYear(value: number) {
    endYear = Math.max(
      value,
      startYear + 1
    );
  }
</script>

<h2>Annual Top 3 Movie Genres</h2>

<p class="subtitle">
  Ranking of the three most common genres in each year.
  Adjust the range below to explore different periods.
</p>

<!-- ========================================================== -->
<!-- YEAR RANGE CONTROLS -->
<!-- ========================================================== -->

<div class="range-container">

  <div class="range-header">

    <strong>Year Range</strong>

    <span>
      {startYear} – {endYear}
    </span>

  </div>

  <div class="range-values">

    <label>
      Start year
      <input
        type="number"
        min={minYear}
        max={endYear - 1}
        bind:value={startYear}
      />
    </label>

    <label>
      End year
      <input
        type="number"
        min={startYear + 1}
        max={maxYear}
        bind:value={endYear}
      />
    </label>

  </div>

  <div class="slider-wrapper">

    <input
      class="slider"
      type="range"
      min={minYear}
      max={maxYear}
      step="1"
      value={startYear}
      oninput={(event) =>
        updateStartYear(
          Number(
            (event.currentTarget as HTMLInputElement).value
          )
        )
      }
    />

    <input
      class="slider"
      type="range"
      min={minYear}
      max={maxYear}
      step="1"
      value={endYear}
      oninput={(event) =>
        updateEndYear(
          Number(
            (event.currentTarget as HTMLInputElement).value
          )
        )
      }
    />

  </div>

  <div class="slider-labels">
    <span>{minYear}</span>
    <span>{maxYear}</span>
  </div>

</div>

<!-- ========================================================== -->
<!-- CHART -->
<!-- ========================================================== -->

{#if years.length > 0}

  <div class="chart-wrapper">

    <svg {width} {height}>

      <g
        transform={`translate(${margin.left}, ${margin.top})`}
      >

        <!-- -------------------------------------------------- -->
        <!-- RANK GUIDES -->
        <!-- -------------------------------------------------- -->

        {#each [1, 2, 3] as rank}

          <line
            x1={0}
            x2={innerWidth}
            y1={yScale(rank)}
            y2={yScale(rank)}
            class="rank-line"
          />

          <text
            x={-15}
            y={yScale(rank)}
            text-anchor="end"
            dominant-baseline="middle"
            class="rank-label"
          >
            #{rank}
          </text>

        {/each}

        <!-- -------------------------------------------------- -->
        <!-- YEAR GUIDES -->
        <!-- -------------------------------------------------- -->

        {#each labeledYears as year}

          <line
            x1={xScale(year)}
            x2={xScale(year)}
            y1={0}
            y2={innerHeight}
            class="year-guide"
          />

          <text
            x={xScale(year)}
            y={innerHeight + 30}
            text-anchor="middle"
            class="year-label"
          >
            {year}
          </text>

        {/each}

        <!-- -------------------------------------------------- -->
        <!-- GENRE LINES -->
        <!-- -------------------------------------------------- -->

        {#each genreSegments as segment}

          {@const path =
            linePath(segment.points)}

          {#if path}

            <path
              d={path}
              fill="none"
              stroke={colorScale(segment.genre)}
              stroke-width={
                selectedGenre === segment.genre
                  ? 5
                  : 2.5
              }
              opacity={genreOpacity(
                segment.genre
              )}
              class="genre-line"
            />

          {/if}

        {/each}

        <!-- -------------------------------------------------- -->
        <!-- POINTS -->
        <!-- -------------------------------------------------- -->

        {#each visibleData as point}

          <circle
            cx={xScale(point.year)}
            cy={yScale(point.rank)}
            r={pointRadius(point.genre)}
            fill={colorScale(point.genre)}
            opacity={genreOpacity(point.genre)}
            class="genre-point"
            tabindex="0"
            role="button"
            aria-label={`${point.genre}, ${point.year}, rank ${point.rank}, ${point.count} movies`}
            onmouseenter={() => {
              selectedGenre = point.genre;
            }}
            onmouseleave={() => {
              selectedGenre = undefined;
            }}
            onfocus={() => {
              selectedGenre = point.genre;
            }}
            onblur={() => {
              selectedGenre = undefined;
            }}
          />

        {/each}

        <!-- -------------------------------------------------- -->
        <!-- END LABELS -->
        <!-- -------------------------------------------------- -->

        {#each visibleData.filter(
          (d) => d.year === years[years.length - 1]
        ) as point}

          <text
            x={(xScale(point.year) ?? 0) + 12}
            y={yScale(point.rank)}
            dominant-baseline="middle"
            fill={colorScale(point.genre)}
            opacity={genreOpacity(point.genre)}
            class="genre-label"
          >
            {point.genre}
          </text>

        {/each}

        <!-- -------------------------------------------------- -->
        <!-- AXIS TITLES -->
        <!-- -------------------------------------------------- -->

        <text
          x={innerWidth / 2}
          y={innerHeight + 60}
          text-anchor="middle"
          class="axis-title"
        >
          Release Year
        </text>

        <text
          transform={`translate(-55, ${
            innerHeight / 2
          }) rotate(-90)`}
          text-anchor="middle"
          class="axis-title"
        >
          Genre Rank
        </text>

      </g>

    </svg>

  </div>

{:else}

  <p>
    No Top 3 data is available for this year range.
  </p>

{/if}

<!-- ========================================================== -->
<!-- HOVER INFORMATION -->
<!-- ========================================================== -->

{#if selectedGenre}

  {@const selectedPoints =
    visibleData.filter(
      (d) => d.genre === selectedGenre
    )}

  <div class="tooltip">

    <strong>{selectedGenre}</strong>

    <span>
      Top 3 in {selectedPoints.length} year{selectedPoints.length === 1 ? "" : "s"}
    </span>

    <span>
      Hover over a point for its rank and movie count.
    </span>

  </div>

{/if}

<style>
  h2 {
    margin-bottom: 4px;
  }

  .subtitle {
    margin-top: 0;
    margin-bottom: 18px;
    font-size: 16px;
  }

  /* ----------------------------------------------------------
     RANGE CONTROLS
     ---------------------------------------------------------- */

  .range-container {
    width: min(900px, 100%);
    margin-bottom: 20px;
    padding: 14px 16px;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: #fafafa;
  }

  .range-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
  }

  .range-header span {
    font-weight: 600;
  }

  .range-values {
    display: flex;
    gap: 20px;
    margin-bottom: 10px;
  }

  .range-values label {
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 13px;
  }

  .range-values input {
    width: 90px;
    padding: 5px 7px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .slider-wrapper {
    position: relative;
    height: 28px;
  }

  .slider {
    position: absolute;
    left: 0;
    width: 100%;
    height: 6px;
    margin: 0;
    appearance: none;
    background: #ddd;
    pointer-events: none;
  }

  .slider::-webkit-slider-thumb {
    width: 17px;
    height: 17px;
    appearance: none;
    border-radius: 50%;
    background: #4c78a8;
    cursor: pointer;
    pointer-events: auto;
  }

  .slider::-moz-range-thumb {
    width: 17px;
    height: 17px;
    border: none;
    border-radius: 50%;
    background: #4c78a8;
    cursor: pointer;
    pointer-events: auto;
  }

  .slider-labels {
    display: flex;
    justify-content: space-between;
    font-size: 12px;
    color: #666;
  }

  /* ----------------------------------------------------------
     CHART
     ---------------------------------------------------------- */

  .chart-wrapper {
    width: 100%;
    overflow-x: auto;
  }

  svg {
    display: block;
    overflow: visible;
  }

  .rank-line {
    stroke: #d9d9d9;
    stroke-width: 1;
  }

  .year-guide {
    stroke: #eeeeee;
    stroke-width: 1;
    stroke-dasharray: 3 4;
  }

  .rank-label {
    font-size: 13px;
    font-weight: 600;
  }

  .year-label {
    font-size: 12px;
  }

  .genre-line {
    transition:
      opacity 0.2s ease,
      stroke-width 0.2s ease;
  }

  .genre-point {
    cursor: pointer;
    stroke: white;
    stroke-width: 1.5;
    transition:
      opacity 0.2s ease,
      r 0.15s ease;
  }

  .genre-label {
    font-size: 13px;
    font-weight: 600;
  }

  .axis-title {
    font-size: 14px;
    font-weight: 600;
  }

  /* ----------------------------------------------------------
     TOOLTIP
     ---------------------------------------------------------- */

  .tooltip {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
    margin-top: 10px;
    padding: 9px 12px;
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