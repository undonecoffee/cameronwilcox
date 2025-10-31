<script>
  // For now, we’ll just list a few projects manually.
  const projects = [
    { name: "Tracker", link: "/tracker", desc: "View progress ofjdskalfjdsaklfjdsljfdsalkjfdkan ongoing projects." },
    { name: "RenderLib", link: "/renderlib", desc: "My custom graphics library for Minecraft." }
  ];
</script>

<style>
  main {
    max-width: 700px;
    margin: 3rem auto;
    padding: 2rem;
    font-family: 'Fira Code', monospace;
    color: #eee;
    background-color: #1b1b1b;
    border-radius: 10px;
  }

  a {
    color: #9acd32;
    text-decoration: none;
  }

  a:hover {
    text-decoration: underline;
  }

  h1 {
    font-size: 2rem;
    margin-bottom: 1.5rem;
  }

  ul {
    list-style: none;
    padding: 0;
  }

  li {
    margin: 1rem 0;
  }
</style>

<main>
  <h1>Welcome to cameronwilcox.dev</h1>
  <p>Here’s where I post my projects, experiments, and random dev stuff.</p>

  <ul>
    {#each projects as project}
      <li>
        <a href={project.link}>{project.name}</a> — {project.desc}
      </li>
    {/each}
  </ul>
</main>
