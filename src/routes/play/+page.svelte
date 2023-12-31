<script lang="ts">
  import { goto } from "$app/navigation";
  import { onMount } from "svelte";
  import { scale, slide } from "svelte/transition";

  import IconSpinner from "~icons/gg/spinner-alt";
  import IconMore from "~icons/gg/trending";
  import IconLess from "~icons/gg/trending-down";
  import IconCorrect from "~icons/gg/check";
  import IconWrong from "~icons/gg/close";

  const delay = 2000;
  const numberFormatter = new Intl.NumberFormat("en-US");
  let highScore: number;

  let video1: any = null;
  let video2: any = null;

  let game: "loading" | "playing" | "over" = "loading";
  let state: "choosing" | "correct" | "incorrect" = "choosing";
  let score = 0;

  async function setVideos() {
    video1 = null;
    video2 = null;
    game = "loading"

    video1 = await fetchVideo();
    video2 = await fetchVideo();

    game = "playing";
  }

  async function fetchVideo() {
    const response = await fetch("/api/random-video", {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
      }
    });

    const result = await response.json();
    return result;
  }

  async function setHighScore() {
    localStorage.setItem("highScore", score.toString());

    const username = localStorage.getItem("username");

    if (!username) return;

    await fetch("/api/update-score", {
      method: "POST",
      body: JSON.stringify({ username, score }),
      headers: {
        "Content-Type": "application/json",
      },
    });
  }

  function wrong() {
    state = "incorrect";

    setTimeout(async () => {
      game = "over";

      if (score > (highScore ?? 0)) {
        await setHighScore();
      }
    }, delay);
  }

  async function correct() {
    state = "correct";
    score++;

    setTimeout(async () => {
      game = "loading"

      video1 = video2;
      video2 = await fetchVideo();

      state = "choosing";
      game = "playing";
    }, delay);
  }

  async function choose(choice: -1 | 1) {
    if (choice === 1) {
      if (Number(video2.views) > Number(video1.views)) {
        correct();
      } else {
        wrong();
      }
    } else {
      if (Number(video2.views) < Number(video1.views)) {
        correct();
      } else {
        wrong();
      }
    }
  }

  onMount(() => {
    setVideos();

    highScore = Number(localStorage.getItem("highScore"));
  });
</script>

<!-- Loading Videos -->
{#if game === "loading"}
  <main class="flex flex-col gap-4 justify-center items-center h-full bg-youtube text-white" transition:slide>
    <div class="flex gap-4 items-center">
      <h1 class="text-4xl">Loading videos</h1>
      <IconSpinner class="animate-spin text-4xl" />
    </div>
    <p class="block text-lg">Hang tight...</p>
  </main>

<!-- Game Over -->
{:else if game === "over"}
  <main class="flex flex-col gap-8 justify-center items-center h-full bg-neutral text-white" transition:slide>
    <h1 class="text-4xl">Game Over!</h1>

    <div class="text-center">
      <h2 class="text-2xl">Your score: <span class="text-youtube">{score}</span></h2>
      <h2 class="text-2xl">Your high score: <span class="text-youtube">{highScore}</span></h2>
    </div>

    {#if score > highScore}
      <h2 class="text-2xl"><span class="text-youtube">
        Nice! You beat your high score!
      </span></h2>
    {/if}

    <div class="space-x-2">
      <a href="/" class="bg-white text-neutral rounded-full px-8 p-2 uppercase text-xl btn">
        Go to menu
      </a>

      <button
        class="bg-youtube text-white rounded-full px-8 p-2 uppercase text-xl btn"
        on:click={() => location.reload()}>
        Play again?
      </button>
    </div>
  </main>

<!-- Game -->
{:else}
  <main class="h-full flex lg:flex-row flex-col relative" transition:slide>
    {#if video1}
      <div class="bg-cover bg-center lg:h-full lg:w-1/2 h-1/2 inline-flex justify-center items-center text-white text-center bg-gray-500 bg-blend-multiply"
        style:background-image="url('{video1.thumbnails?.maxres?.url ?? video1.thumbnails?.high?.url ?? video1.thumbnails?.standard?.url}')">
        <h1 class="text-xl lg:w-2/3 w-4/5">
          The YouTube video
          <a href="https://youtube.com/watch?v={video1.videoId}"
            class="lg:text-4xl text-2xl font-semibold block hover:underline"
            target="_blank">
            "{video1.title}"
          </a>
          has
          <span class="text-5xl block text-youtube">{numberFormatter.format(video1.views)}</span>
          total views
        </h1>
      </div>
    {/if}

    {#if state === "choosing"}
      <div class="bg-youtube text-white lg:p-8 p-4 lg:text-4xl text-2xl rounded-full absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-center" transition:scale>
        VS
      </div>
    {:else if state === "correct"}
      <div class="bg-green-500 text-white lg:p-8 p-4 lg:text-4xl text-4xl rounded-full absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-center" transition:scale>
        <IconCorrect />
      </div>
    {:else}
      <div class="bg-neutral text-white lg:p-8 p-4 lg:text-4xl text-4xl rounded-full absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-center" transition:scale>
        <IconWrong />
      </div>
    {/if}

    {#if video2}
      <div class="bg-cover bg-center lg:h-full lg:w-1/2 h-1/2 inline-flex justify-center items-center text-white text-center bg-gray-500 bg-blend-multiply"
        style:background-image="url('{video2.thumbnails?.maxres?.url ?? video2.thumbnails?.high?.url ?? video2.thumbnails?.standard?.url}')">
        <h1 class="text-xl lg:w-2/3 w-4/5">
          The YouTube video
          <a href="https://youtube.com/watch?v={video2.videoId}"
            class="lg:text-4xl text-2xl font-semibold block hover:underline"
            target="_blank">
            "{video2.title}"
          </a>
          has
          <span class="text-xl text-youtube flex lg:flex-col my-4 gap-1">
            {#if state === "choosing"}
              <button class="uppercase bg-youtube text-white rounded-full py-2 btn w-full" on:click={() => choose(1)}>
                More
                <IconMore class="inline ml-2" />
              </button>
              <button class="uppercase bg-neutral text-white rounded-full py-2 btn w-full" on:click={() => choose(-1)}>
                Less
                <IconLess class="inline ml-2" />
              </button>
            {/if}

            {#if state !== "choosing"}
              <h1 id="countUp" class="text-5xl text-youtube text-center w-full">
                {numberFormatter.format(video2.views)}
              </h1>
            {/if}
          </span>
          <span class="hidden lg:block">total views</span>
        </h1>
      </div>
    {/if}

    <p class="absolute lg:left-8 left-0 lg:bottom-4 lg:top-auto top-1/2 -translate-y-1/2 text-white text-xl text-center lg:w-auto w-[45vw]">Score: {score}</p>
    <p class="absolute lg:right-8 right-0 lg:bottom-4 lg:top-auto top-1/2 -translate-y-1/2 text-white text-xl text-center lg:w-auto w-[45vw]">High Score: {highScore}</p>
  </main>
{/if}
