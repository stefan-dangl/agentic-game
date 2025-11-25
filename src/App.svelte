<script>
  let name = 'Svelte';
  let count = 0;

  const greetings = [
    'Hello',
    'Hola',
    'Bonjour',
    'Ciao',
    'こんにちは'
  ];

  const nextGreeting = () => greetings[count % greetings.length];
</script>

<main class="app">
  <header>
    <h1>Basic Svelte Example</h1>
    <p>This page shows a tiny, executable Svelte component.</p>
  </header>

  <section class="card">
    <h2>{nextGreeting()}, {name}!</h2>
    <p>Update the name and click the button to see reactivity in action.</p>

    <label class="field">
      <span>Display name</span>
      <input
        bind:value={name}
        placeholder="Enter your name"
        aria-label="Display name"
      />
    </label>

    <button class="cta" on:click={() => count += 1}>
      Clicked {count} {count === 1 ? 'time' : 'times'}
    </button>
  </section>
</main>
