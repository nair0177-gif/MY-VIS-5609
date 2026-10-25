<script lang="ts">
  import type { TMovie } from "../types";

  type Props = {
    movies: TMovie[];
  };

  let { movies }: Props = $props();

  // ------------------------------------------------------------
  // AVAILABLE YEARS
  // ------------------------------------------------------------

  const years = $derived(
    Array.from(
      new Set(
        movies
          .filter((movie) => movie.year instanceof Date)
          .map((movie) => movie.year.getFullYear())
      )
    ).sort((a, b) => a - b)
  );

  // Selected year
  let selectedYear = $state(
    years.length > 0
      ? years[years.length - 1]
      : 2000
  );

  // ------------------------------------------------------------
  // FIND TOP 3 GENRES FOR SELECTED YEAR
  // ------------------------------------------------------------

  type GenreCount = {
    genre: string;
    count: number;
  };

  function getTopGenres(
    movieData: TMovie[],
    year: number
  ): GenreCount[] {
    const counts: Record<string, number> = {};

    movieData
      .filter(
        (movie) =>
          movie.year instanceof Date &&
          movie.year.getFullYear() === year
      )
      .forEach((movie) => {
        movie.genres.forEach((genre) => {
          if (!genre || genre === "NA") return;

          counts[genre] =
            (counts[genre] || 0) + 1;
        });
      });

    return Object.entries(counts)
      .map(([genre, count]) => ({
        genre,
        count
      }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 3);
  }

  const topGenres = $derived(
    getTopGenres(movies, selectedYear)
  );

  // ------------------------------------------------------------
  // MAX VALUE FOR BAR SCALE
  // ------------------------------------------------------------

  const maxCount = $derived(
    Math.max(
      ...topGenres.map((d) => d.count),
      1
    )
  );

  // ------------------------------------------------------------
  // YEAR NAVIGATION
  // ------------------------------------------------------------

  function previousYear() {
    const index = years.indexOf(selectedYear);

    if (index > 0) {
      selectedYear = years[index - 1];
    }
  }

  function nextYear() {
    const index = years.indexOf(selectedYear);

    if (
      index >= 0 &&
      index < years.length - 1
    ) {
      selectedYear = years[index + 1];
    }
  }
</script>

<section class="container">

  <h2>Annual Top 3 Movie Genres</h2>

  <p class="description">
    Explore how the three most common movie genres
    change from year to year.
  </p>

  <!-- -------------------------------------------------------- -->
  <!-- YEAR CONTROLS -->
  <!-- -------------------------------------------------------- -->

  <div class="controls">

    <button
      onclick={previousYear}
      disabled={
        selectedYear === years[0]
      }
      aria-label="Previous year"
    >
      ←
    </button>

    <div class="year-display">
      {selectedYear}
    </div>

    <button
      onclick={nextYear}
      disabled={
        selectedYear ===
        years[years.length - 1]
      }
      aria-label="Next year"
    >
      →
    </button>

  </div>

  <!-- -------------------------------------------------------- -->
  <!-- SLIDER -->
  <!-- -------------------------------------------------------- -->

  <div class="slider-container">

    <span>
      {years[0]}
    </span>

    <input
      type="range"
      min={0}
      max={Math.max(years.length - 1, 0)}
      value={Math.max(
        years.indexOf(selectedYear),
        0
      )}
      oninput={(event) => {
        const index = Number(
          (event.currentTarget as HTMLInputElement).value
        );

        selectedYear = years[index];
      }}
    />

    <span>
      {years[years.length - 1]}
    </span>

  </div>

  <!-- -------------------------------------------------------- -->
  <!-- TOP 3 -->
  <!-- -------------------------------------------------------- -->

  <div class="chart">

    {#if topGenres.length > 0}

      {#each topGenres as item, index}

        <div class="row">

          <div class="rank">
            #{index + 1}
          </div>

          <div class="genre">
            {item.genre}
          </div>

          <div class="bar-area">

            <div
              class="bar"
              style={`width: ${
                (item.count / maxCount) * 100
              }%`}
            ></div>

          </div>

          <div class="count">
            {item.count}
          </div>

        </div>

      {/each}

    {:else}

      <p class="no-data">
        No movie data available for this year.
      </p>

    {/if}

  </div>

  <!-- -------------------------------------------------------- -->
  <!-- EXPLANATION -->
  <!-- -------------------------------------------------------- -->

  <p class="hint">
    Use the slider or the arrow buttons to explore
    the top three genres for each year.
  </p>

</section>

<style>

  .container {
    max-width: 850px;
    margin-top: 40px;
  }

  h2 {
    margin-bottom: 6px;
  }

  .description {
    margin-top: 0;
    margin-bottom: 25px;
    font-size: 16px;
  }

  /* ----------------------------------------------------------
     YEAR CONTROLS
     ---------------------------------------------------------- */

  .controls {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 25px;
    margin-bottom: 20px;
  }

  .controls button {
    width: 42px;
    height: 42px;

    border: 1px solid #ccc;
    border-radius: 6px;

    background: white;

    font-size: 20px;
    cursor: pointer;
  }

  .controls button:hover:not(:disabled) {
    background: #f0f0f0;
  }

  .controls button:disabled {
    opacity: 0.3;
    cursor: default;
  }

  .year-display {
    min-width: 100px;

    text-align: center;

    font-size: 32px;
    font-weight: 700;
  }

  /* ----------------------------------------------------------
     SLIDER
     ---------------------------------------------------------- */

  .slider-container {
    display: flex;
    align-items: center;
    gap: 12px;

    margin-bottom: 45px;

    font-size: 13px;
  }

  .slider-container input {
    flex: 1;
    cursor: pointer;
  }

  /* ----------------------------------------------------------
     CHART
     ---------------------------------------------------------- */

  .chart {
    display: flex;
    flex-direction: column;
    gap: 22px;

    padding: 25px;

    border: 1px solid #ddd;
    border-radius: 8px;

    background: #fafafa;
  }

  .row {
    display: grid;

    grid-template-columns:
      45px
      110px
      1fr
      50px;

    align-items: center;

    gap: 12px;
  }

  .rank {
    font-size: 15px;
    font-weight: 700;
  }

  .genre {
    font-size: 16px;
    font-weight: 600;
  }

  .bar-area {
    height: 30px;

    background: #e8e8e8;

    border-radius: 4px;

    overflow: hidden;
  }

  .bar {
    height: 100%;

    background: #4c78a8;

    border-radius: 4px;

    transition: width 0.25s ease;
  }

  .count {
    font-size: 15px;
    font-weight: 600;
    text-align: right;
  }

  /* ----------------------------------------------------------
     FOOTER
     ---------------------------------------------------------- */

  .hint {
    margin-top: 15px;

    font-size: 13px;

    color: #666;

    text-align: center;
  }

  .no-data {
    text-align: center;
    color: #666;
  }

</style>