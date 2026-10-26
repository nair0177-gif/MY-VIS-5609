<script lang="ts">
    import type { TMovie } from "../types";
    import * as d3 from "d3";

    type TProps = {
        movies: TMovie[];
        x: keyof TMovie;
        y: keyof TMovie;
        size?: keyof TMovie;
        width?: number;
        height?: number;
    };

    const {
        movies,
        x,
        y,
        size,
        height = 500,
        width = 600,
    }: TProps = $props();

    let selectedMovie: TMovie | undefined = $state();

    const margin = {
        top: 15,
        bottom: 50,
        left: 30,
        right: 10,
    };

    const usableArea = $derived({
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left,
    });

    const sizeRange = [3, 15];

    function getPosition(
    scale: any,
    value: any,
    fallback: number,
): number {
    if (!scale) return fallback;

    const position = scale(value);

    if (position === undefined || !Number.isFinite(position)) {
        return fallback;
    }

    if (typeof scale.bandwidth === "function") {
        return position + scale.bandwidth() / 2;
    }

    return position;
}

function getScale(
    attrName: keyof TMovie,
    axis: "x" | "y" | "color" | "size",
    movies: TMovie[],
) {
    if (movies.length == 0) {
        return null;
    }

    let range: number[] = [0, 0];
    if (axis == "x") {
        range = [usableArea.left, usableArea.right];
    } else if (axis == "y") {
        range = [usableArea.bottom, usableArea.top];
    } else if (axis == "size") {
        range = sizeRange;
    }

    if (typeof movies[0][attrName] == "string") {
        return d3
            .scaleBand()
            .domain(movies.map((d) => d[attrName] as string))
            .range(range);
    } else if (typeof movies[0][attrName] == "number") {
        return d3
            .scaleLinear()
            .domain(
                d3.extent(movies, (d) => d[attrName] as number) as [
                    number,
                    number,
                ],
            )
            .range(range);
    } else if (movies[0][attrName] instanceof Date) {
        return d3
            .scaleTime()
            .domain(
                d3.extent(movies, (d) => d[attrName] as Date) as [
                    Date,
                    Date,
                ],
            )
            .range(range);
    } else if (typeof movies[0][attrName] == "object") {
        let allValues = movies
            .map((d) => d[attrName] as string[])
            .reduce((acc, val) => acc.concat(val), []);

        return d3
            .scaleBand()
            .domain([...new Set(allValues)])
            .range(range);
    } else {
        return null;
    }
}

    const xScale = $derived(getScale(x, "x", movies));
    const yScale = $derived(getScale(y, "y", movies));
    const sizeScale = $derived(
        size ? getScale(size, "size", movies) : null,
    );

    let xAxis: SVGGElement;
    let yAxis: SVGGElement;

    function updateAxis() {
        if (!xScale || !yScale || !xAxis || !yAxis) {
            return;
        }

        const xAxisGenerator = d3.axisBottom(xScale as d3.AxisScale<d3.AxisDomain>);
        const yAxisGenerator = d3.axisLeft(yScale as d3.AxisScale<d3.AxisDomain>);

        d3.select(xAxis)
            .call(xAxisGenerator as any)
            .selectAll("text")
            .attr("transform", "rotate(45)")
            .style("text-anchor", "start");

        d3.select(yAxis).call(yAxisGenerator as any);
    }

    function handleMovieClick(movie: TMovie) {
        // Tip 1: clicking a selected movie again deselects it.
        selectedMovie =
            selectedMovie === movie ? undefined : movie;
    }

    $effect(() => {
        // Re-run axis rendering when the data or selected attributes change.
        movies;
        x;
        y;
        width;
        height;
        updateAxis();
    });
</script>

<svg {width} {height}>
    <g class="points">
        {#each movies as movie, i (`${movie.tconst}-${i}`)}
{#if x == "genres"}
    {#each movie["genres"] as genre}
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <circle
            cx={xScale ? (xScale as any)(genre) : usableArea.left}
            cy={yScale ? (yScale as any)(movie[y]) : usableArea.bottom}
            r={sizeScale
                ? (sizeScale as any)(movie[size!])
                : sizeRange[0]}
            fill={movie == selectedMovie ? "steelblue" : "transparent"}
            stroke="steelblue"
            stroke-width="2"
            opacity={movie == selectedMovie ? 1 : 0.5}
            onclick={() => handleMovieClick(movie)}
        />
    {/each}
{:else}
    <!-- svelte-ignore a11y_click_events_have_key_events -->
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <circle
        cx={xScale ? (xScale as any)(movie[x]) : usableArea.left}
        cy={yScale ? (yScale as any)(movie[y]) : usableArea.bottom}
        r={sizeScale
            ? (sizeScale as any)(movie[size!])
            : sizeRange[0]}
        fill={movie == selectedMovie ? "steelblue" : "transparent"}
        stroke="steelblue"
        stroke-width="2"
        opacity={movie == selectedMovie ? 1 : 0.5}
        onclick={() => handleMovieClick(movie)}
    />
{/if}
        {/each}
    </g>

    <g
        transform="translate(0, {usableArea.bottom})"
        bind:this={xAxis}
    />

    <g
        transform="translate({usableArea.left}, 0)"
        bind:this={yAxis}
    />
</svg>

<div class="selectedInfo">
    {selectedMovie
        ? JSON.stringify(selectedMovie, null, 2)
        : "Click on a point to see details"}
</div>

<style>
    .points circle {
        transition:
            r 0.5s,
            cx 0.5s,
            cy 0.5s;
        cursor: pointer;
    }

    svg {
        display: inline-block;
        vertical-align: top;
    }

    .selectedInfo {
        display: inline-block;
        vertical-align: top;
        width: 250px;
        color: #555;
        font-family: monospace;
        white-space: pre-wrap;
        background-color: #f9f9f9;
        padding: 10px;
        border: 1px solid #ccc;
        border-radius: 4px;
        overflow: auto;
    }
</style>
