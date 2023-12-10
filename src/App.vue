<template>
  <div id="app">
    <div id="lab-name-container">
      <h1 class="lab-name" style="background-color: #eeb8b8; padding: 10px; text-align: center; border-radius: 8px;">
        Система аналізу відгуків пацієнтів на лікарські препарати</h1>
    </div>
    <div class="card">
      <input type="text" v-model="search" placeholder="Пошук" class="card-input"
             style="width: 80%; margin-top: 10px; margin-bottom: 10px; text-align: center;">
      <ul class="card-list">
        <li v-for="result in results" :key="result.Drug">
          <a @click="scrollTo(result.Drug)" class="card-link" data-query="{{ result.Drug }}"
             style="border-radius: 4px;">{{
              result.Drug
            }}</a>
        </li>
      </ul>
      <div v-show="results" class="card-results">
        <ul>
          <li v-for="result in results" :key="result">
            {{ result }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
#lab-name-container {
  width: 900px;
  margin: 0 auto;
  margin-top: 40px;
}

.lab-name {
  font-size: 24px;
  font-weight: bold;
  border-radius: 8px;
}

.card {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: 900px;
  margin: 0 auto;
  margin-top: 20px;
}

.card-title {
  font-size: 24px;
  font-weight: bold;
  border-radius: 8px;
}

.card-input {
  width: 80%;
  padding: 10px;
  text-align: center;
  border-radius: 8px;
  margin: 0 auto
}

.card-list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.card-link {
  font-size: 16px;
  padding: 10px 15px;
  border-radius: 4px;
  color: #000;
  border-radius: 4px;
}

.card-results {
  display: none;
}
</style>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      back_host: 'http://127.0.0.1:8000',
      search: '',
      frequentQueries: ['Задание 1', 'Задание 2', 'Задание 3'],
      results: [],
    };
  },
  methods: {
    scrollTo(query) {
      const element = document.querySelector(`[data-query="${query}"`);
      element.scrollIntoView({behavior: 'smooth'});
    },

    getResults() {
      const response = axios.get(`${this.back_host}/search?text_to_search=${this.search}`);
      console.log(response);
      response.then((response) => {
        this.results = response.data;
      });
    },
  },
  watch: {
    search: function () {
      this.getResults();
    },
  },
};
</script>
