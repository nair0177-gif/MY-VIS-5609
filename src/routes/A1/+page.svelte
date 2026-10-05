<script lang="ts">
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import type { TMovie } from "../../types";
  import { Bar } from "$lib";
  import BumpChart from "$lib/BumpChart.svelte";
  import GenreCooccurrence from "../../lib/GenreCooccurrence.svelte";
  

  let movies = $state<TMovie[]>([]);

  onMount(() => {
    d3.csv("summer_movies.csv").then((data) => {
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
  <p>Here are {movies.length} movies.</p>

  <Bar movies={movies} />

  <hr />

  <h2>Q1: How do the top three movie genres (by number of movies) change over time?</h2>
  <p>
    This visualization shows how the ranking of the top three genres changes
    across years.
  </p>

  <BumpChart movies={movies} />

  <hr />

  <h2>Q2: Are there any correlations between different genres?</h2>
  <p>
    This visualization shows which genres most frequently co-occur with a
    selected genre.
  </p>

  <GenreCooccurrence movies={movies} />

{:else}
  <p>Loading movie data...</p>
{/if}
