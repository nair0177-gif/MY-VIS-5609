<script lang="ts">
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import type { TMovie } from "../../types";
  import { Bar } from "$lib";
  import BumpChart from "$lib/BumpChart.svelte";
  import GenreCooccurrence from "../../lib/GenreCooccurrence.svelte";
  import RankMatrix from "../../lib/RankMatrix.svelte";

  let movies = $state<TMovie[]>([]);

  onMount(() => {
    d3.csv("/summer_movies.csv").then((data) => {
      movies = data.map((d) => ({
        num_votes: Number(d.num_votes),
        runtime_minutes: Number(d.runtime_minutes),
        genres: d.genres
          ? d.genres.split(",").map((genre) => genre.trim())
          : [],
        year: new Date(`${d.year}-01-01`),
        average_rating: Number(d.average_rating),
        tconst: d.tconst,
        title_type: d.title_type,
        primary_title: d.primary_title,
        original_title: d.original_title
      }));
    });
  });
</script>

<h1>Summer Movies</h1>

{#if movies.length > 0}
  <p>Loaded {movies.length} movies.</p>

  <Bar movies={movies} />

  <hr />

  <BumpChart movies={movies} />

  <hr />

  <GenreCooccurrence movies={movies} />
{:else}
  <p>Loading movie data...</p>
{/if}