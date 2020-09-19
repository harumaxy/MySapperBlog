<script>
  import Logo from "./Logo.svelte";
  import Nav from "./Nav.svelte";

  export let segment;
</script>

<style>
  header {
    display: flex;
    margin: 0 auto;
    padding: 2em;
    width: 100%;
  }
</style>

<header class="bg-green-400">
  <Logo />
  <Nav {segment} />
</header>
