<template>
  <div id="app">
    <MemoryManagement />
  </div>
</template>

<script>
import MemoryManagement from './components/MemoryManagement.vue'

export default {
  name: 'App',
  components: {
    MemoryManagement
  }
}
</script>

<style>
#app {
  font-family: 'Arial', sans-serif;
  background-color: #f7f7f7;
  padding: 20px;
}

h1 {
  font-size: 24px;
  margin-bottom: 20px;
  color: #333;
}

button {
  font-size: 16px;
  padding: 10px 20px;
  margin: 10px;
  background-color: #4caf50;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

button:hover {
  background-color: #45a049;
}

p {
  font-size: 18px;
  color: #555;
  margin-bottom: 10px;
}

.statistics {
  margin-top: 40px;
}

.statistics p {
  margin-bottom: 5px;
}

</style>
