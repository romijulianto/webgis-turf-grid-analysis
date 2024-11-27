<template>
  <div id="app">
    <!-- Header atau Navigation (opsional) -->
    <header>
      <h1>WebGIS: Filter dan Grid Analysis</h1>
    </header>

    <!-- Konten utama -->
    <main>
      <NuxtPage />
    </main>

    <!-- Footer (opsional) -->
    <footer>
      <p>© 2024 WebGIS Application</p>
    </footer>
  </div>
</template>

<style>
/* Gaya dasar untuk aplikasi */
#app {
  font-family: Arial, sans-serif;
  margin: 0;
  padding: 0;
}

header {
  background-color: #007cbf;
  color: white;
  padding: 1rem;
  text-align: center;
}

main {
  padding: 1rem;
}

footer {
  background-color: #f8f8f8;
  text-align: center;
  padding: 1rem;
  border-top: 1px solid #ddd;
}
</style>
