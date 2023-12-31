<script context="module">
    import { writable } from 'svelte/store';
    export const PathFrames = writable([]);
</script>

<script>
    import { UserInputFeedback } from '../../stores/user-input-feedback';
    import { fillTracks } from "../../modules/slider";
    import { onMount, onDestroy } from 'svelte';

    import { wallNodes } from './stores/walls';
    import { obstacles } from './stores/obstacle';
    import { pathNodes } from './stores/path';
    import { visitedNodes } from './stores/visited';

    import { PathFinding } from './stores/path-finding';
    import { gridStore } from './stores/grid';
    import { animate, makeBorderWalls, pause, resume } from './animation-logic';
    import { algorithms, algorithmsAsObject } from './algorithms/algorithms';

    /* Maze and Patterns algorithms */
    import { recursive_division } from './maze-algorithms/recursive-division';
    import { randomDFS } from './maze-algorithms/random-dfs';

    onMount(() => {
        fillTracks();
        document.addEventListener('click', handleDropdownClickout);
    });

    onDestroy(() => {
        document.removeEventListener('click', handleDropdownClickout);
    });

    $: xsize = $PathFinding.xsize;
    $: ysize = $PathFinding.ysize;

    let paused = true;
    let playing = false;
    let disableAll = false;
    let showDropdown = false;
    let algoDropdown;

    const handleDropdownClickout = event => {
        if(!algoDropdown.contains(event.target)) {
            toggleAlgorithmsDropdown(false);
        }
    };

    const toggleAlgorithmsDropdown = (show = null) => {
        show == null ? showDropdown = !showDropdown : showDropdown = show;
    };

    const selectAlgorithm = (name) => {
        toggleAlgorithmsDropdown();
        PathFinding.update(prev => ({...prev, currentAlgo: algorithmsAsObject[name]}));
    };

    const populateFrames = () => {
        if(!$PathFinding.currentAlgo || !$PathFinding.currentAlgo.algo) return;
        const { pathAnimationFrames } = $PathFinding.currentAlgo.algo(
            xsize,
            ysize, 
            $gridStore.startIndex, 
            $gridStore.destinationIndex,
            $wallNodes, 
            $obstacles,
        )
        PathFrames.set(pathAnimationFrames);
    }

    const clear = () => {
        wallNodes.clear();
        obstacles.clear();
        pathNodes.clear();
        visitedNodes.clear();
    }

    const hideUserInputFeedback = () => setTimeout(UserInputFeedback.hide, 500);

    const clearAll = () => {
        PathFinding.update(prev => ({...prev, currentAlgo: 'none'}));
        PathFrames.set([]);
        obstacles.clear();
        wallNodes.clear();
        visitedNodes.clear();
        pathNodes.clear();
        makeBorderWalls(xsize, ysize);
    }

    const togglePause = () => {
        if(paused) pause();
        else resume();
        paused = !paused;
    }

    const stop = () => {
        paused = true;
        playing = false;
        pause();
        toggleDisableGrid();
    }

    const toggleDisableGrid = () => {
        document.getElementById('pfd-disable-grid').click();
    }

    const search = () => {
        if($PathFinding.currentAlgo == 'none') {
            alert('Select an algorithm first.');
            return;
        }
        playing = true;
        pathNodes.clear();
        visitedNodes.clear();
        const { searchAnimationFrames, pathAnimationFrames } = $PathFinding.currentAlgo.algo(
            xsize,
            ysize, 
            $gridStore.startIndex, 
            $gridStore.destinationIndex,
            $wallNodes, 
            $obstacles,
        )
        toggleDisableGrid();
        animate(searchAnimationFrames, pathAnimationFrames, stop);
    }
    const toggleDisableAll = () => disableAll = !disableAll;
</script>

<!-- choose algrotihm -->
<div class="dropdown" style="width: calc(100% - 1rem); left: .5rem; top: .5rem" bind:this={algoDropdown}>
    <button disabled={playing || disableAll} 
            on:click={toggleAlgorithmsDropdown}
            type="submit" 
            class="dropdown-toggler"> 
            { $PathFinding.currentAlgo == 'none'
              ? 'Choose an algorithm'
              : $PathFinding.currentAlgo.name }
    </button>
    {#if showDropdown}
        <ul class="dropdown-menu">
            <ul role="group" name="option group">
                <label for="option group"> Weighted Algorithm </label>
                {#each algorithms as algo (algo)}
                {#if algo.isWeighted}
                <li role="option" on:click={() => selectAlgorithm(algo.name)} > {algo.name} </li>
                {/if}
                {/each}
            </ul>
            <ul role="group" name="option group">
                <label for="option group"> Unweighted Algorithm </label>
                {#each algorithms as algo (algo)}
                {#if !algo.isWeighted}
                <li role="option" on:click={() => selectAlgorithm(algo.name)} > {algo.name} </li>
                {/if}
                {/each}
            </ul>
        </ul>
    {/if }
</div>

<p style="text-align: center; margin-top: 1rem;"> Maze and Obstacles </p>

<!-- maze and patterns -->
<div class="two-btns" >
    <button 
    class="small"
        disabled={playing || disableAll} 
        title="Recursive Division" 
        on:click={() => {
            clear();
            makeBorderWalls(xsize, ysize);
            recursive_division(xsize, ysize);
        }}> 
        <!-- svelte-ignore a11y-invalid-attribute -->
        <a href="#" style="width: 100%; height: 100%; display: block;"> 
            Recursive Division
        </a>
    </button>
    <button 
        class="small"
        disabled={playing || disableAll} 
        title="Random Depth-first Search" 
        on:click={() => {
            clear();
            makeBorderWalls(xsize, ysize);
            randomDFS(xsize, ysize);
        }}> 
        <!-- svelte-ignore a11y-invalid-attribute -->
        <a href="#" style="width: 100%; height: 100%; display: block;"> 
            Random DFS
        </a>
    </button>
</div>

<button 
    class="small btns"
    disabled={playing || disableAll} 
    title="Random Depth-first Search for Obstacles" 
    on:click={() => {
        clear();
        makeBorderWalls(xsize, ysize);
        if($PathFinding.currentAlgo.isWeighted) randomDFS(xsize, ysize, false);
        else alert('Weighted nodes only works with Weighted Algorithms');
    }}> 
    <!-- svelte-ignore a11y-invalid-attribute -->
    <a href="#" style="width: 100%; height: 100%; display: block;"> 
        Random Weighted Nodes
    </a>
</button>

<!-- Find the Path! -->
<button 
    on:click={search} 
    disabled={playing || disableAll || $PathFinding.currentAlgo == 'none'}
    color="accent" class="btns">
     
    <!-- svelte-ignore a11y-invalid-attribute -->
    <a href="#" style="width: 100%; height: 100%; display: block;"> 
        {$PathFinding.currentAlgo == 'none' ? 'Pick an algorihtm first.':'Find the path!'}
    </a>
</button>

<div class="two-btns">
    <!-- clear walls -->
    <button 
        color="primary" 
        disabled={playing || disableAll}
        on:click={() => {wallNodes.clear(); makeBorderWalls(xsize, ysize)}}> 
        Clear Walls
    </button>
    <!-- clear obstacles -->
    <button color="primary" on:click={obstacles.clear} disabled={playing || disableAll}> 
        Clear Obstacles
    </button>
</div>

<div class="two-btns">
    <!-- Clear All -->
    <button color="primary" on:click={clearAll} disabled={playing || disableAll}> Clear All </button>
    <!-- Clear path -->
    <button color="primary" on:click={pathNodes.clear} disabled={playing || disableAll}> Clear Path </button>
</div>

<div class="two-btns">    
    <button 
        on:click={togglePause} 
        disabled={!playing || disableAll}
        color="{!paused ? 'primary':'accent'}" >
        {!paused ? 'Play':'Pause'}
    </button>
    <button 
        on:click={stop}
        color="accent" 
        disabled={!playing || disableAll}> Stop </button>
</div>

<!-- Speed -->
<div class="not-btn" title="Change animation speed">
    <label for="path-finding-speed"> Speed </label>
    <input 
        on:input={
        () => UserInputFeedback.set(true, `Animation Speed: ${$PathFinding.speed}`)}
        on:change={hideUserInputFeedback}
        bind:value={$PathFinding.speed}
        disabled={playing || disableAll}
        id="path-finding-speed"
        color="primary"
        min="1"
        max="10"
        step="1"
        role="slider"
        type="range">
</div>

<button hidden on:click={toggleDisableAll} id="pfp-disable-all" />
<button hidden on:click={populateFrames} id="populate-frames" />

<style>
    button {
        text-overflow: ellipsis !important;
        white-space: nowrap;
        overflow: hidden;
    }
    *[disabled] {
        opacity: 0.2;
        pointer-events: none;
    }
    button[color="accent"] > a {
        color: white;
    }
    button.small a {
        color: var(--text1) !important;
        font-size: .8rem;
        /* text-overflow: ellipsis;
        white-space: nowrap;
        overflow: hidden; */
    }
    button > a:hover {
        text-decoration: none;
    }
    button > a {
        color: var(--surface1);
    }
    .not-btn {
        display: flex;
        place-items: center;
        justify-content: space-between;
        border-radius: 0.25rem;
        padding: 0.7rem 0.5rem;
    }
    .not-btn:hover {
        background-color: var(--surface3);
    }
    .two-btns {
        padding: 0.7rem 0.5rem;
        display: grid;
        grid-template-columns: calc(50% - .5rem) calc(50% - .5rem);
        gap: 1rem;
    }
    .btns {
        margin: .7rem .5rem;
    }
 </style>