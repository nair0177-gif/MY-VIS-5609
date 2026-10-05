<script lang="ts">
  import type { TMovie } from "../types";

  type Props = {
    movies: TMovie[];
  };

  let { movies }: Props = $props();

  // ------------------------------------------------------------
  // AVAILABLE GENRES
  // ------------------------------------------------------------

  const genres = $derived(
    Array.from(
      new Set(
        movies.flatMap((movie) =>
          movie.genres.filter(
            (genre) => genre && genre !== "NA"
          )
        )
      )
    ).sort()
  );

  let selectedGenre = $state("");


  // Set Comedy as the initial selection once genres are available.
  $effect(() => {
    if (!selectedGenre && genres.length > 0) {
      selectedGenre = genres.includes("Comedy")
        ? "Comedy"
        : genres[0];
    }
  });


  // ------------------------------------------------------------
  // CO-OCCURRENCE CALCULATION
  // ------------------------------------------------------------

  type Cooccurrence = {
    genre: string;
    count: number;
  };

  function getCooccurrence(
    movieData: TMovie[],
    selected: string
  ): Cooccurrence[] {
    if (!selected) {
      return [];
    }

    const counts: Record<string, number> = {};

    movieData.forEach((movie) => {
      const movieGenres = movie.genres.filter(
        (genre) =>
          genre &&
          genre !== "NA"
      );

      // Only consider movies containing
      // the selected genre.
      if (!movieGenres.includes(selected)) {
        return;
      }

      // Count every other genre in the same movie.
      movieGenres.forEach((genre) => {
        if (genre === selected) {
          return;
        }

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
      .slice(0, 10);
  }


  const cooccurringGenres = $derived(
    getCooccurrence(
      movies,
      selectedGenre
    )
  );


  // ------------------------------------------------------------
  // SCALE
  // ------------------------------------------------------------

  const maxCount = $derived(
    Math.max(
      ...cooccurringGenres.map(
        (item) => item.count
      ),
      1
    )
  );


  // ------------------------------------------------------------
  // SELECTED GENRE SUMMARY
  // ------------------------------------------------------------

  const selectedMovieCount = $derived(
    selectedGenre
      ? movies.filter((movie) =>
          movie.genres.includes(selectedGenre)
        ).length
      : 0
  );


  // ------------------------------------------------------------
  // HOVER STATE
  // ------------------------------------------------------------

  let hoveredGenre = $state<string | null>(null);

  const hoveredItem = $derived(
    cooccurringGenres.find(
      (item) => item.genre === hoveredGenre
    )
  );
</script>


<section class="container">

  <h2>Genre Co-occurrence</h2>

  <p class="description">
    Explore which genres most often appear in the
    same movies as the selected genre.
  </p>


  <!-- -------------------------------------------------------- -->
  <!-- GENRE SELECTOR -->
  <!-- -------------------------------------------------------- -->

  <div class="selector">

    <label for="genre">
      Select a genre:
    </label>

    <select
      id="genre"
      bind:value={selectedGenre}
    >
      {#each genres as genre}
        <option value={genre}>
          {genre}
        </option>
      {/each}
    </select>

  </div>


  <!-- -------------------------------------------------------- -->
  <!-- SELECTED GENRE SUMMARY -->
  <!-- -------------------------------------------------------- -->

  {#if selectedGenre}

    <div class="summary">

      <strong>{selectedGenre}</strong>

      appears in

      <strong>{selectedMovieCount}</strong>

      movies.

      <span>
        The bars below show how often another genre
        appears in those same movies.
      </span>

    </div>

  {/if}


  <!-- -------------------------------------------------------- -->
  <!-- CHART -->
  <!-- -------------------------------------------------------- -->

  {#if cooccurringGenres.length > 0}

    <div class="chart-header">

      <h3>
        Genres co-occurring with {selectedGenre}
      </h3>

      <span>
        Top 10 genre pairings
      </span>

    </div>


    <div class="chart">

      {#each cooccurringGenres as item, index}

        <div
          class="row"
          class:highlighted={hoveredGenre === item.genre}
          class:dimmed={hoveredGenre !== null &&
                        hoveredGenre !== item.genre}
          onmouseenter={() => {
            hoveredGenre = item.genre;
          }}
          onmouseleave={() => {
            hoveredGenre = null;
          }}
        >

          <!-- Rank -->

          <div class="rank">
            {index + 1}
          </div>


          <!-- Genre -->

          <div class="genre">
            {item.genre}
          </div>


          <!-- Bar -->

          <div class="bar-area">

            <div
              class="bar"
              style={`width: ${
                (item.count / maxCount) * 100
              }%`}
            ></div>

          </div>


          <!-- Count -->

          <div class="count">
            {item.count}
          </div>

        </div>

      {/each}

    </div>


    <!-- ------------------------------------------------------ -->
    <!-- HOVER DETAIL -->
    <!-- ------------------------------------------------------ -->

    <div class="detail">

      {#if hoveredItem}

        <strong>
          {selectedGenre} + {hoveredItem.genre}
        </strong>

        <span>
          {hoveredItem.count} movies
        </span>

      {:else}

        <span>
          Hover over a bar to inspect a genre pairing.
        </span>

      {/if}

    </div>


  {:else}

    <p class="no-data">
      No co-occurring genres found.
    </p>

  {/if}


  <p class="hint">
    Bar length represents the number of movies containing
    both genres. Select another genre to compare its
    co-occurrence pattern.
  </p>

</section>


<style>

  .container {
    max-width: 900px;
    margin-top: 50px;
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
     SELECTOR
     ---------------------------------------------------------- */

  .selector {
    display: flex;
    align-items: center;
    gap: 12px;

    margin-bottom: 20px;
  }


  .selector label {
    font-weight: 600;
  }


  select {
    min-width: 180px;

    padding: 8px 12px;

    border: 1px solid #bbb;
    border-radius: 5px;

    background: white;

    font-size: 15px;

    cursor: pointer;
  }


  select:focus {
    outline: 2px solid #4c78a8;
    outline-offset: 1px;
  }


  /* ----------------------------------------------------------
     SUMMARY
     ---------------------------------------------------------- */

  .summary {
    margin-bottom: 25px;

    padding: 13px 16px;

    border-left: 4px solid #4c78a8;

    background: #f5f5f5;

    font-size: 15px;
  }


  .summary span {
    display: block;

    margin-top: 5px;

    color: #555;

    font-size: 14px;
  }


  /* ----------------------------------------------------------
     CHART HEADER
     ---------------------------------------------------------- */

  .chart-header {
    display: flex;
    align-items: baseline;
    justify-content: space-between;

    margin-bottom: 10px;
  }


  .chart-header h3 {
    margin: 0;

    font-size: 18px;
  }


  .chart-header span {
    color: #666;

    font-size: 13px;
  }


  /* ----------------------------------------------------------
     CHART
     ---------------------------------------------------------- */

  .chart {
    display: flex;
    flex-direction: column;

    gap: 14px;

    padding: 25px;

    border: 1px solid #ddd;
    border-radius: 8px;

    background: #fafafa;
  }


  .row {
    display: grid;

    grid-template-columns:
      35px
      120px
      1fr
      60px;

    align-items: center;

    gap: 12px;

    padding: 4px 6px;

    border-radius: 5px;

    transition:
      opacity 0.15s ease,
      background 0.15s ease;
  }


  .row.highlighted {
    background: #eeeeee;
  }


  .row.dimmed {
    opacity: 0.35;
  }


  .rank {
    font-size: 14px;
    font-weight: 700;

    color: #777;
  }


  .genre {
    font-size: 15px;
    font-weight: 600;
  }


  .bar-area {
    height: 28px;

    background: #e8e8e8;

    border-radius: 4px;

    overflow: hidden;
  }


  .bar {
    height: 100%;

    background: #4c78a8;

    border-radius: 4px;

    transition:
      width 0.25s ease,
      background 0.15s ease;
  }


  .row.highlighted .bar {
    background: #2f5597;
  }


  .count {
    font-size: 14px;
    font-weight: 600;

    text-align: right;
  }


  /* ----------------------------------------------------------
     HOVER DETAIL
     ---------------------------------------------------------- */

  .detail {
    display: flex;
    align-items: center;
    gap: 12px;

    min-height: 42px;

    margin-top: 10px;

    padding: 8px 12px;

    border: 1px solid #ddd;
    border-radius: 5px;

    background: white;

    font-size: 14px;
  }


  .detail span {
    color: #555;
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
    padding: 30px;

    text-align: center;

    color: #666;
  }

</style>