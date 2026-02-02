<template>
  <router-view/>
</template>

<style>
@import url("https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600;700&display=swap");
@import url("https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css");

:root {
  --bg-color: #fdfaf7;
  --card-bg: rgba(255, 255, 255, 0.7);
  --primary-color: #88c3a9; /* Sage Green */
  --accent-color: #f3b664; /* Warm Amber */
  --text-main: #3d4449;
  --text-muted: #7a8288;
}

body {
  margin: 0;
  background-color: var(--bg-color);
  background-image: 
    radial-gradient(at 0% 0%, rgba(136, 195, 169, 0.05) 0px, transparent 50%),
    radial-gradient(at 100% 100%, rgba(243, 182, 100, 0.05) 0px, transparent 50%);
  min-height: 100vh;
}

#app {
  font-family: 'Outfit', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: var(--text-main);
}

/* Global button refinements */
.el-button {
  font-family: 'Outfit', sans-serif;
  border-radius: 12px;
}
</style>
