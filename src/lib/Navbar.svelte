<script lang="ts">
  import logo from "../assets/logo.svg";
</script>

<nav class="navbar" aria-label="main navigation">
  <div class="navbar-brand">
    <a class="navbar-item" href={"#"} on:click|preventDefault={() => (window.location = window.location)}>
      <img src={logo} alt="ChatGPT-web" width="28" height="28" />
      <p class="ml-2 is-size-4 has-text-weight-bold">Transplant Chatbot</p>
    </a>
  </div>
</nav>
