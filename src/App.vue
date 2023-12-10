<template>
  <div id="app">
    <div id="lab-name-container">
      <h1 class="lab-name" style="background-color: #eeb8b8; padding: 10px; text-align: center; border-radius: 8px;">
        Система аналізу відгуків пацієнтів на лікарські препарати</h1>
    </div>
    <div class="card">
      <input
          type="text"
          v-model="search"
          placeholder="Пошук"
          class="card-input"
          style="width: 80%; margin-top: 10px; margin-bottom: 10px; text-align: center;"
          @focus="handleSearchFocus"
          @blur="handleSearchBlur"
      >
      <div class="dropdown" v-show="results.length > 0 && isSearchFocused">
        <ul class="dropdown-list" @mousedown="handleDropdownClick">
          <li v-for="(result) in results" :key="result.Drug" @click="selectSuggestion(result)">
            {{ result.Drug }}
          </li>
        </ul>
      </div>
      <div class="selected-drug">
        <span class="label">Назва обраного продукта:</span>
        <span class="value" v-if="selectedDrug">{{ selectedDrug.Drug }}</span>
      </div>
    </div>
  </div>
</template>


<script>
import axios from 'axios';

export default {
  data() {
    return {
      back_host: 'http://127.0.0.1:8000',
      search: '',
      frequentQueries: ['Задание 1', 'Задание 2', 'Задание 3'],
      results: [],
      distances: {
        euclidean: "Евклідова",
        squared_euclidean: "Квадрат відстані Евкліда",
        manhattan: "Манхетенська відстань",
        chebyshev: "Відстань Чебишева",
        minkowski: "Відстань Мінковського порядку p",
        power: "Степенева відстань",
      },
      isSearchFocused: false, // Track whether the search input is focused
      isDropdownClicked: false, // Track whether the dropdown was clicked
      selectedDrug: null, // Store the selected drug information
    };
  },
  methods: {
    scrollTo(query) {
      const element = document.querySelector(`[data-query="${query}"`);
      element.scrollIntoView({ behavior: 'smooth' });
    },

    getResults() {
      const response = axios.get(`${this.back_host}/search?text_to_search=${this.search}`);
      response.then((response) => {
        this.results = response.data;
      });
    },

    selectSuggestion(result) {
      // Set the selected suggestion in the search bar
      this.search = result.Drug;
      // Store the full information of the selected drug
      this.selectedDrug = result;
      // Clear the results and hide the dropdown
      this.results = [];
      this.isSearchFocused = false;
      console.log(this.selectedDrug)
      console.log(this.results)
    },

    handleSearchFocus() {
      this.isSearchFocused = true;
    },

    handleSearchBlur() {
      // Close the dropdown only if it was not clicked
      if (!this.isDropdownClicked) {
        this.isSearchFocused = false;
      }
      // Reset the flag
      this.isDropdownClicked = false;
    },

    handleDropdownClick() {
      // Set the flag when the dropdown is clicked
      this.isDropdownClicked = true;
    },
  },
  watch: {
    search: function () {
      this.getResults();
    },
  },
};
</script>


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

.card-input {
  width: 80%;
  padding: 10px;
  text-align: center;
  border-radius: 8px;
  margin: 0 auto;
  margin-bottom: 10px; /* Adjusted margin-bottom */
}

.selected-drug {
  margin-top: 5px; /* Adjusted margin-top */
  text-align: center;
  margin-bottom: 5px;
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

.dropdown {
  position: relative;
  width: 80%;
  margin: 0 auto;
}

.dropdown-list {
  list-style: none;
  padding: 0;
  margin: 0;
  width: 100%;
  position: absolute;
  background-color: #ded1d1;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  border-radius: 4px;
  z-index: 1;
}

.dropdown-list li {
  padding: 10px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
  transition: background-color 0.3s ease;
}

.dropdown-list li:last-child {
  border-bottom: none;
}

.dropdown-list li:hover {
  background-color: #f0f0f0;
}

.label {
  font-weight: bold;
  margin-right: 10px;
}

.value {
  font-style: italic;
  color: #333;
}
</style>
