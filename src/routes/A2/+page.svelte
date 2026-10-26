
<script lang="ts">
    import * as d3 from "d3";
    import { Scatter, Line } from "$lib";
    import { onMount } from "svelte";
    import type { TMovie } from "../../types";
    import { base } from "$app/paths";

    // Reactive variable for storing the data
    let movies: TMovie[] = $state([]);

    let yearRange: [Date, Date] | undefined = $state();

    function getYearCountArray(movies: TMovie[]) {
        let yearCount: { [year: number]: number } = {};

        const allYears = [
            ...new Set(
                movies
                    .filter(
                        (d) =>
                            d.year instanceof Date &&
                            !Number.isNaN(d.year.getTime()),
                    )
                    .map((d) => d.year.getFullYear()),
            ),
        ];

        for (let year of allYears) {
            yearCount[year] = movies.filter(
                (d) =>
                    d.year instanceof Date &&
                    !Number.isNaN(d.year.getTime()) &&
                    d.year.getFullYear() === year,
            ).length;
        }

        // Convert the map to an array of { year, count } objects
        const yearCountArray = Object.entries(yearCount).map(
            ([year, count]) => ({
                x: new Date(Number(year), 0, 1),
                y: count as number,
            }),
        );

        // Sort the array by year in ascending order
        yearCountArray.sort((a, b) => a.x.getTime() - b.x.getTime());

        return yearCountArray;
    }

    let yearCountArray = $derived(getYearCountArray(movies));

    type TAxisSelection = {
        x: keyof TMovie;
        y: keyof TMovie;
        size: keyof TMovie;
    };

    // Default chart: Year vs. Average Rating, sized by Number of Votes
    let axisSelection: TAxisSelection = $state({
        x: "year",
        y: "average_rating",
        size: "num_votes",
    });

    // Load the CSV using the GitHub Pages base path
    async function loadCsv() {
        try {
            const csvUrl = `${base}/summer_movies.csv`;

            const response = await fetch(csvUrl);

            if (!response.ok) {
                throw new Error(`CSV request failed: HTTP ${response.status}`);
            }

            const csvText = await response.text();

            movies = d3.csvParse(csvText, (row) => ({
                ...row,
                num_votes: Number(row.num_votes),
                runtime_minutes: Number(row.runtime_minutes),
                genres: (row.genres ?? "").split(","),
                year: new Date(`${row.year ?? ""}-01-01`),
                average_rating: Number(row.average_rating),
            }) as TMovie);

            console.log("Loaded CSV Data:", movies);
            console.log("Movie fields:", Object.keys(movies[0] ?? {}));
        } catch (error) {
            console.error("Error loading CSV:", error);
        }
    }

    // Load data when the component mounts
    onMount(loadCsv);

    // X Axis: all fields, matching the professor's original code
    const attrOptionsX = $derived(
        movies[0] ? Object.keys(movies[0]) : [],
    );

    // Y Axis: exclude genres because it is an array
    const attrOptionsY = $derived(
        movies[0]
            ? Object.keys(movies[0]).filter((key) => key !== "genres")
            : [],
    );

    // Size: exclude genres because it is an array
    const attrOptionsS = $derived(
        movies[0]
            ? Object.keys(movies[0]).filter((key) => key !== "genres")
            : [],
    );

    // Filter movies based on the selected year range
    const filteredMovies = $derived.by(() => {
        const range = yearRange;

        if (!range) {
            return movies;
        }

        const [startDate, endDate] = range;

        return movies.filter(
            (movie) =>
                movie.year instanceof Date &&
                !Number.isNaN(movie.year.getTime()) &&
                movie.year >= startDate &&
                movie.year <= endDate,
        );
    });
</script>

<div class="container">
    <h1>Summer Movies</h1>

    <p>
        Here are {movies.length === 0 ? "..." : `${movies.length} movies`}
    </p>

    {#if movies.length > 0}
        <div class="selectors">
            <label>
                X Axis:
                <select bind:value={axisSelection.x}>
                    {#each attrOptionsX as key}
                        <option value={key}>{key}</option>
                    {/each}
                </select>
            </label>

            <label>
                Y Axis:
                <select bind:value={axisSelection.y}>
                    {#each attrOptionsY as key}
                        <option value={key}>{key}</option>
                    {/each}
                </select>
            </label>

            <label>
                Size:
                <select bind:value={axisSelection.size}>
                    {#each attrOptionsS as key}
                        <option value={key}>{key}</option>
                    {/each}
                </select>
            </label>
        </div>

        <Scatter
            movies={filteredMovies}
            x={axisSelection.x}
            y={axisSelection.y}
            size={axisSelection.size}
        />

        <br />

        <Line data={yearCountArray} bind:yearRange />
    {/if}
</div>

<style>
    .container {
        width: 60vw;
        margin: 10px auto;
        padding: 10px;
    }

    .selectors {
        display: flex;
        flex-wrap: wrap;
        gap: 16px;
        align-items: center;
        margin: 16px 0;
    }

    .selectors label {
        display: flex;
        flex-direction: column;
        gap: 6px;
        font-size: 14px;
        font-weight: 600;
    }

    .selectors select {
        padding: 7px 10px;
        border: 1px solid #ccc;
        border-radius: 5px;
        background: white;
        font-size: 14px;
    }
</style>