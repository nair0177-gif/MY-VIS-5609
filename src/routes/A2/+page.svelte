
<script lang="ts">
    import * as d3 from "d3";
    import { Scatter, Line } from "$lib";
    import { onMount } from "svelte";
    import type { TMovie } from "../../types";
    import { base } from "$app/paths";

    // ---------------------------------------------------------
    // SHARED DATA
    // ---------------------------------------------------------

    let movies: TMovie[] = $state([]);
    let yearRange: [Date, Date] | undefined = $state();

    async function loadCsv() {
        try {
            const response = await fetch(`${base}/summer_movies.csv`);

            if (!response.ok) {
                throw new Error(`CSV request failed: HTTP ${response.status}`);
            }

            const csvText = await response.text();

            movies = d3.csvParse(csvText, (row) => ({
                ...row,
                num_votes: Number(row.num_votes),
                runtime_minutes: Number(row.runtime_minutes),
                genres: (row.genres ?? "")
                    .split(",")
                    .map((genre) => genre.trim())
                    .filter(Boolean),
                year: new Date(`${row.year ?? ""}-01-01`),
                average_rating: Number(row.average_rating),
            }) as TMovie);

            console.log("Loaded CSV Data:", movies);
        } catch (error) {
            console.error("Error loading CSV:", error);
        }
    }

    onMount(loadCsv);

    // ---------------------------------------------------------
    // PART 3: ORIGINAL SCATTERPLOT AND YEAR BRUSH
    // ---------------------------------------------------------

    type TAxisSelection = {
        x: keyof TMovie;
        y: keyof TMovie;
        size: keyof TMovie;
    };

    let axisSelection: TAxisSelection = $state({
        x: "year",
        y: "average_rating",
        size: "num_votes",
    });

    const attrOptionsX = $derived(
        movies[0] ? Object.keys(movies[0]) : [],
    );

    const attrOptionsY = $derived(
        movies[0]
            ? Object.keys(movies[0]).filter((key) => key !== "genres")
            : [],
    );

    const attrOptionsS = $derived(
        movies[0]
            ? Object.keys(movies[0]).filter((key) => key !== "genres")
            : [],
    );

    function getYearCountArray(data: TMovie[]) {
        const yearCount: Record<number, number> = {};

        for (const movie of data) {
            if (
                movie.year instanceof Date &&
                !Number.isNaN(movie.year.getTime())
            ) {
                const year = movie.year.getFullYear();
                yearCount[year] = (yearCount[year] ?? 0) + 1;
            }
        }

        return Object.entries(yearCount)
            .map(([year, count]) => ({
                x: new Date(Number(year), 0, 1),
                y: count,
            }))
            .sort((a, b) => a.x.getTime() - b.x.getTime());
    }

    const yearCountArray = $derived(getYearCountArray(movies));

    const filteredMovies = $derived.by(() => {
        if (!yearRange) return movies;

        const [startDate, endDate] = yearRange;

        return movies.filter(
            (movie) =>
                movie.year instanceof Date &&
                !Number.isNaN(movie.year.getTime()) &&
                movie.year >= startDate &&
                movie.year <= endDate,
        );
    });

    // ---------------------------------------------------------
    // PART 4: RUNTIME & GENRE EXPLORER
    // Research question:
    // Do runtime and genre relate to popularity, measured by votes?
    // ---------------------------------------------------------

    type PopularityMovie = TMovie & {
        runtime_minutes: number;
        num_votes: number;
        genres: string[];
    };

    const validPopularityMovies = $derived(
        movies.filter((movie) => {
            const runtime = Number(movie.runtime_minutes);
            const votes = Number(movie.num_votes);

            return (
                Number.isFinite(runtime) &&
                runtime > 0 &&
                Number.isFinite(votes) &&
                votes > 0 &&
                Array.isArray(movie.genres) &&
                movie.genres.length > 0
            );
        }) as PopularityMovie[],
    );

    const allGenres = $derived(
        [...new Set(validPopularityMovies.flatMap((movie) => movie.genres))]
            .sort((a, b) => a.localeCompare(b)),
    );

    let selectedGenre: string | null = $state(null);

    const part4Movies = $derived(
        selectedGenre
            ? validPopularityMovies.filter((movie) =>
                  movie.genres.includes(selectedGenre),
              )
            : validPopularityMovies,
    );

    // View 1: Runtime vs. number of votes
    const chartWidth = 680;
    const chartHeight = 390;

    const chartMargin = {
        top: 20,
        right: 25,
        bottom: 65,
        left: 80,
    };

    const plotLeft = chartMargin.left;
    const plotRight = chartWidth - chartMargin.right;
    const plotTop = chartMargin.top;
    const plotBottom = chartHeight - chartMargin.bottom;

    const runtimeExtent = $derived(
    d3.extent(
        validPopularityMovies,
        (movie) => movie.runtime_minutes,
    ) as [number, number],
);

    const runtimeDomain = $derived.by(() => {
        const [min, max] = runtimeExtent;

        if (!Number.isFinite(min) || !Number.isFinite(max)) {
            return [0, 200] as [number, number];
        }

        if (min === max) {
            return [Math.max(0, min - 10), max + 10] as [number, number];
        }

        return [Math.max(0, min), max] as [number, number];
    });

    const minVotes = $derived(
        Math.max(
            1,
            d3.min(validPopularityMovies, (movie) => movie.num_votes) ?? 1,
        ),
    );

    const maxVotes = $derived(
        Math.max(
            minVotes * 10,
            d3.max(validPopularityMovies, (movie) => movie.num_votes) ?? 10,
        ),
    );

    const runtimeScale = $derived(
        d3.scaleLinear()
            .domain(runtimeDomain)
            .range([plotLeft, plotRight])
            .nice(),
    );

    const voteScale = $derived(
        d3.scaleLog()
            .domain([minVotes, maxVotes])
            .range([plotBottom, plotTop])
            .clamp(true),
    );

    const runtimeTicks = $derived(runtimeScale.ticks(6));
    
const voteTicks = $derived(
    [1, 10, 100, 1_000, 10_000, 100_000, 1_000_000].filter(
        (value) => value >= minVotes && value <= maxVotes,
    ),
);


    // View 2: Median votes by genre
    function median(values: number[]) {
        const sorted = [...values].sort((a, b) => a - b);

        if (sorted.length === 0) return 0;

        const middle = Math.floor(sorted.length / 2);

        return sorted.length % 2 === 0
            ? (sorted[middle - 1] + sorted[middle]) / 2
            : sorted[middle];
    }

    const genreSummaries = $derived(
        allGenres
            .map((genre) => {
                const genreMovies = validPopularityMovies.filter((movie) =>
                    movie.genres.includes(genre),
                );

                return {
                    genre,
                    count: genreMovies.length,
                    medianVotes: median(
                        genreMovies.map((movie) => movie.num_votes),
                    ),
                };
            })
            .sort((a, b) => b.medianVotes - a.medianVotes),
    );

    const maxMedianVotes = $derived(
        Math.max(1, d3.max(genreSummaries, (item) => item.medianVotes) ?? 1),
    );

    const barScale = $derived(
        d3.scaleLinear()
            .domain([0, maxMedianVotes])
            .range([0, 300]),
    );

    // Selecting the same genre again clears the selection.
    function toggleGenre(genre: string) {
        selectedGenre = selectedGenre === genre ? null : genre;
    }

    function formatVotes(value: number) {
        return new Intl.NumberFormat("en-US", {
            notation: "compact",
            maximumFractionDigits: 1,
        }).format(value);
    }
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

    <!-- =====================================================
         PART 4
         ===================================================== -->

    <section class="part4">
        <div class="section-heading">
            <p class="eyebrow">PART 4 · FURTHER EDA</p>
            <h2>Runtime &amp; Genre Explorer</h2>

            <p class="research-question">
                Do movie runtime and genre relate to audience popularity,
                measured by the number of votes, and do these patterns
                differ across genres?
            </p>
        </div>

        <div class="summary-strip">
            <div class="summary-card">
                <span>Movies analyzed</span>
                <strong>{part4Movies.length.toLocaleString()}</strong>
            </div>

            <div class="summary-card">
                <span>Genres in dataset</span>
                <strong>{allGenres.length}</strong>
            </div>

            <div class="summary-card">
                <span>Current selection</span>
                <strong>{selectedGenre ?? "All genres"}</strong>
            </div>
        </div>

        <div class="view-card">
            <div class="view-header">
                <div>
                    <h3>1. Runtime vs. audience popularity</h3>
                    <p>
                        Each circle represents one movie. The vertical axis
                        uses a logarithmic scale because vote counts can
                        vary substantially.
                    </p>
                </div>

                {#if selectedGenre}
                    <button
                        class="reset-button"
                        onclick={() => (selectedGenre = null)}
                    >
                        Clear genre
                    </button>
                {/if}
            </div>

            <div class="chart-scroll">
                <svg
                    width={chartWidth}
                    height={chartHeight}
                    role="img"
                    aria-label="Scatterplot of movie runtime against number of votes"
                >
                    {#each runtimeTicks as tick}
                        <g>
                            <line
                                x1={runtimeScale(tick)}
                                x2={runtimeScale(tick)}
                                y1={plotTop}
                                y2={plotBottom}
                                class="grid-line"
                            />
                            <text
                                x={runtimeScale(tick)}
                                y={plotBottom + 22}
                                text-anchor="middle"
                                class="tick-label"
                            >
                                {tick}
                            </text>
                        </g>
                    {/each}

                    {#each voteTicks as tick}
                        <g>
                            <line
                                x1={plotLeft}
                                x2={plotRight}
                                y1={voteScale(tick)}
                                y2={voteScale(tick)}
                                class="grid-line"
                            />
                            <text
                                x={plotLeft - 10}
                                y={voteScale(tick) + 4}
                                text-anchor="end"
                                class="tick-label"
                            >
                                {formatVotes(tick)}
                            </text>
                        </g>
                    {/each}

                    <line
                        x1={plotLeft}
                        x2={plotRight}
                        y1={plotBottom}
                        y2={plotBottom}
                        class="axis-line"
                    />

                    <line
                        x1={plotLeft}
                        x2={plotLeft}
                        y1={plotTop}
                        y2={plotBottom}
                        class="axis-line"
                    />

                    
{#each part4Movies as movie, i (`${movie.tconst}-${i}`)}
    <circle
        cx={runtimeScale(movie.runtime_minutes)}
        cy={Math.max(
            plotTop,
            Math.min(
                plotBottom,
                voteScale(movie.num_votes) +
                    ((i % 5) - 2) * 1.2,
            ),
        )}
        r="2.8"
        class="movie-point"
    >
        <title>
            {movie.primary_title || movie.original_title || movie.tconst}
            — Runtime: {movie.runtime_minutes} min
            — Votes: {movie.num_votes.toLocaleString()}
            — Genres: {movie.genres.join(", ")}
        </title>
    </circle>
{/each}


                    <text
                        x={(plotLeft + plotRight) / 2}
                        y={chartHeight - 15}
                        text-anchor="middle"
                        class="axis-label"
                    >
                        Runtime (minutes)
                    </text>

                    <text
                        transform={`translate(20, ${(plotTop + plotBottom) / 2}) rotate(-90)`}
                        text-anchor="middle"
                        class="axis-label"
                    >
                        Number of votes (log scale)
                    </text>
                </svg>
            </div>

            <p class="chart-note">
                {#if selectedGenre}
                    Showing movies tagged with <strong>{selectedGenre}</strong>.
                    Movies belonging to multiple genres may appear in multiple
                    genre selections.
                {:else}
                    Showing all movies with valid runtime, vote count, and genre
                    data. Select a genre in the chart below to filter this view.
                {/if}
            </p>
        </div>

        <div class="view-card">
            <div class="view-header">
                <div>
                    <h3>2. Typical popularity by genre</h3>
                    <p>
                        Bars represent median vote count for each genre.
                        Select a genre to filter the scatterplot above.
                    </p>
                </div>

                <button
                    class="reset-button"
                    onclick={() => (selectedGenre = null)}
                    disabled={selectedGenre === null}
                >
                    Show all genres
                </button>
            </div>

            <div class="genre-list">
                {#each genreSummaries as item (item.genre)}
                    <button
                        class:genre-selected={selectedGenre === item.genre}
                        class="genre-row"
                        aria-pressed={selectedGenre === item.genre}
                        onclick={() => toggleGenre(item.genre)}
                    >
                        <span class="genre-name">{item.genre}</span>

                        <span class="bar-track">
                            <span
                                class="bar-fill"
                                style={`width: ${(barScale(item.medianVotes) / 300) * 100}%`}
                            ></span>
                        </span>

                        <span class="genre-value">
                            {formatVotes(item.medianVotes)}
                        </span>

                        <span class="genre-count">
                            n={item.count}
                        </span>
                    </button>
                {/each}
            </div>

            <p class="chart-note">
                Median vote count summarizes the middle of the distribution
                and is less affected by extremely popular movies than the mean.
                Bars use a linear scale; vote counts may still be highly skewed.
            </p>
        </div>

    </section>
</div>

<style>
    .container {
        width: 60vw;
        min-width: 600px;
        margin: 10px auto;
        padding: 10px;
        color: #172b4d;
        font-family: Arial, Helvetica, sans-serif;
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

    .part4 {
        margin-top: 64px;
        padding-top: 32px;
        border-top: 2px solid #dbe5f1;
    }

    .section-heading {
        margin-bottom: 24px;
    }

    .eyebrow {
        color: #2563eb;
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 1.4px;
    }

    .section-heading h2 {
        margin: 8px 0 12px;
        color: #163a70;
        font-size: 30px;
    }

    .research-question {
        max-width: 780px;
        color: #53647b;
        line-height: 1.6;
    }

    .summary-strip {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        gap: 12px;
        margin-bottom: 20px;
    }

    .summary-card {
        display: flex;
        flex-direction: column;
        gap: 8px;
        padding: 16px;
        border: 1px solid #dce5f0;
        border-radius: 8px;
        background: #f7f9fc;
    }

    .summary-card span {
        color: #64748b;
        font-size: 12px;
    }

    .summary-card strong {
        overflow-wrap: anywhere;
        color: #163a70;
        font-size: 20px;
    }

    .view-card {
        margin: 20px 0;
        padding: 20px;
        border: 1px solid #dce5f0;
        border-radius: 10px;
        background: white;
    }

    .view-header {
        display: flex;
        justify-content: space-between;
        align-items: flex-start;
        gap: 16px;
        margin-bottom: 16px;
    }

    .view-header h3,
    .interpretation-card h3 {
        margin: 0 0 8px;
        color: #163a70;
        font-size: 18px;
    }

    .view-header p,
    .chart-note {
        color: #64748b;
        font-size: 13px;
        line-height: 1.55;
    }

    .view-header p {
        max-width: 620px;
        margin: 0;
    }

    .chart-scroll {
        overflow-x: auto;
    }

    .grid-line {
        stroke: #e7edf5;
        stroke-width: 1;
    }

    .axis-line {
        stroke: #8b9bb0;
        stroke-width: 1.2;
    }

    .tick-label {
        fill: #64748b;
        font-size: 11px;
    }

    .axis-label {
        fill: #334155;
        font-size: 12px;
        font-weight: 600;
    }

    
.movie-point {
    fill: #2563eb;
    fill-opacity: 0.25;
    stroke: #1d4ed8;
    stroke-opacity: 0.3;
    stroke-width: 0.5;
}

.movie-point:hover {
    fill-opacity: 0.95;
    stroke-opacity: 1;
    stroke-width: 1.5;
}


    .reset-button {
        flex-shrink: 0;
        padding: 8px 12px;
        border: 1px solid #cbd5e1;
        border-radius: 6px;
        background: white;
        color: #163a70;
        font-size: 12px;
        cursor: pointer;
    }

    .reset-button:hover:not(:disabled) {
        background: #eff6ff;
    }

    .reset-button:disabled {
        cursor: default;
        opacity: 0.45;
    }

    .genre-list {
        display: flex;
        flex-direction: column;
        gap: 5px;
    }

    .genre-row {
        display: grid;
        grid-template-columns: 100px minmax(80px, 1fr) 65px 50px;
        align-items: center;
        gap: 10px;
        width: 100%;
        padding: 7px 8px;
        border: 1px solid transparent;
        border-radius: 5px;
        background: white;
        text-align: left;
        cursor: pointer;
    }

    .genre-row:hover {
        background: #f5f8fd;
    }

    .genre-row.genre-selected {
        border-color: #93b4f7;
        background: #eff6ff;
    }

    .genre-name {
        overflow: hidden;
        color: #334155;
        font-size: 12px;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .bar-track {
        display: block;
        height: 15px;
        overflow: hidden;
        border-radius: 3px;
        background: #edf2f8;
    }

    .bar-fill {
        display: block;
        height: 100%;
        border-radius: 3px;
        background: #3974d8;
    }

    .genre-value {
        color: #163a70;
        font-size: 12px;
        font-weight: 700;
        text-align: right;
    }

    .genre-count {
        color: #64748b;
        font-size: 11px;
        text-align: right;
    }

    .chart-note {
        margin: 14px 0 0;
    }

    .interpretation-card {
        margin-top: 20px;
        padding: 20px;
        border: 1px solid #dce5f0;
        border-radius: 10px;
    }

    .interpretation-card ul {
        padding-left: 20px;
        color: #475569;
        line-height: 1.8;
    }

    .interpretation-card p {
        color: #64748b;
        font-size: 13px;
        line-height: 1.7;
    }

    @media (max-width: 900px) {
        .container {
            width: 94%;
            min-width: 0;
            box-sizing: border-box;
        }

        .summary-strip {
            grid-template-columns: 1fr;
        }

        .view-card {
            padding: 14px;
        }

        .view-header {
            flex-wrap: wrap;
        }
    }
</style>
