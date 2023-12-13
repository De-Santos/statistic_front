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
          <li v-for="(result) in results" :key="result.drug_name" @click="selectSuggestion(result)">
            {{ result.drug_name }}
          </li>
        </ul>
      </div>
      <div class="selected-drug">
        <span class="label">Назва обраного препарату:</span>
        <span class="value" v-if="selectedDrug">{{ selectedDrug.drug_name }}</span>
      </div>
      <form @submit.prevent="submitForm" class="form">
        <div class="form-group">
          <label class="from-label">
            <span class="form-label-text">Оберіть колонки для кластеризації:</span>
          </label>
          <table class="table">
            <thead>
            <tr>
              <th></th>
              <th><b>Назва</b></th>
              <th><b>Тип</b></th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="column_info in columns_info" :key="column_info">
              <td>
                <input type="checkbox" v-model="selectedColumns" :value="column_info[0]" :disabled="!column_info[2]">
              </td>
              <td>{{ column_info[0] }}</td>
              <td>{{ column_info[1] }}</td>
            </tr>
            </tbody>
          </table>
        </div>
        <div class="main-container" :style="{ backgroundColor: selectedColumns.length ? '#dccccc' : '#dadada' }">
          <div class="form-group" :style="{ backgroundColor: selectedColumns.length ? '#ceeaf3' : disable_color }">
            <label for="distance">Виберіть функцію вимірювання відстані</label>
            <select class="form-control" id="distance" v-model="distance_method" @change="updateDistanceRequirements"
                    :disabled="!selectedColumns.length" required>
              <option v-for="(value, key) in distances" :value="key">{{ value }}</option>
            </select>
          </div>

          <div class="form-group" :style="{ backgroundColor: selectedColumns.length ? '#fce4ec' : disable_color }">
            <label for="k_count">Введіть число K:</label>
            <input type="number" id="k_count" style="margin-left: 10px; width: 10%; border-radius: 10px"
                   v-model="k_count"
                   :disabled="!selectedColumns.length" required min="2">
          </div>

          <div class="form-group" v-if="requireP"
               :style="{ backgroundColor: selectedColumns.length ? '#d0efb1' : disable_color }">
            <label for="p_num">Введіть число P:</label>
            <input type="number" id="p_num" style="margin-left: 10px; width: 10%; border-radius: 10px" v-model="p_num"
                   :disabled="!selectedColumns.length" required>
          </div>

          <div class="form-group" v-if="requireR"
               :style="{ backgroundColor: selectedColumns.length ? '#fff9c4' : disable_color }">
            <label for="r_num">Введіть число R:</label>
            <input type="number" id="r_num" style="margin-left: 10px; width: 10%; border-radius: 10px" v-model="r_num"
                   :disabled="!selectedColumns.length" required>
          </div>

          <div class="form-group"
               :style="{ backgroundColor: selectedColumns.length ? '#bdc4ab' : disable_color, marginBottom: 0 }">
            <label for="limit">Введіть обмеження (опціонально):</label>
            <input type="number" id="limit" style="margin-left: 10px; width: 10%; border-radius: 10px" v-model="limit"
                   :disabled="!selectedColumns.length" value="-1" required min="-1">
          </div>
        </div>
        <button type="submit" :disabled="!selectedColumns.length || !selectedDrug">Calculate!</button>
      </form>
    </div>
    <div v-if="show_calculated_table">
      <h2 class="table-title">Calculated table</h2>
      <table class="table">
        <thead>
        <tr>
          <th v-for="column in columns_info">{{ column[0] }}</th>
          <th>cluster</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="item in calculated_response" :key="item">
          <td v-for="column in columns_info">
            {{ item[column[0]] }}
          </td>
          <td>{{ item[cluster_id_key] }}</td>
        </tr>
        </tbody>
      </table>
    </div>
  </div>
  <div v-if="loading" class="loading-overlay">
    <div class="loading-spinner"></div>
  </div>
</template>


<script>
import axios from 'axios';

export default {
  data() {
    return {
      back_host: 'http://127.0.0.1:8000',
      search: '',
      results: [],
      selectedColumns: [], // Added selectedColumns
      columns_info: [],
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
      k_count: 2,
      p_num: 1,
      r_num: 1,
      requireP: false,
      requireR: false,
      distance_method: "euclidean",
      limit: -1,
      disable_color: 'rgba(224,247,250,0.42)',
      calculated_response: [],
      show_calculated_table: false,
      cluster_id_key: "cluster_id",
      loading: false,
    };
  },
  methods: {
    async submitForm() {
      try {
        this.loading = true;
        const distance_params = {};
        if (this.requireP) {
          distance_params.p_num = this.p_num;
        }
        if (this.requireR) {
          distance_params.r_num = this.r_num;
        }

        const request_data = {
          drug_id: this.selectedDrug.drug_id,
          columns: this.selectedColumns,
          centroids_count: this.k_count,
          distance: this.distance_method,
          distance_params,
          limit: this.limit === -1 ? null : this.limit,
        };

        const response = await axios.post(`${this.back_host}/calculate`, request_data)
        this.calculated_response = response.data
        this.show_calculated_table = true
      } catch (error) {
        console.error('Error:', error);
      } finally {
        this.loading = false;
      }
    },

    updateDistanceRequirements() {
      if (this.distance_method === 'minkowski') {
        this.requireP = true;
        this.requireR = false;
      } else if (this.distance_method === 'power') {
        this.requireP = true;
        this.requireR = true;
      } else {
        this.requireP = false;
        this.requireR = false;
      }
    },

    getSearchResults() {
      const response = axios.get(`${this.back_host}/search?text_to_search=${this.search}`);
      response.then((response) => {
        this.results = response.data;
      });
    },

    getNumColumns() {
      return axios.get(`${this.back_host}/columns`)
          .then((response) => this.columns_info = response.data)
    },


    selectSuggestion(result) {
      // Set the selected suggestion in the search bar
      this.search = result.drug_name;
      // Store the full information of the selected drug
      this.selectedDrug = result;
      // Clear the results and hide the dropdown
      this.results = [];
      this.isSearchFocused = false;
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
    selectedColumns(newVal) {
      if (newVal.length > 3) {
        this.selectedColumns.pop(); // Remove the last element if more than 3 are selected
      }
    },
    search: function () {
      this.getSearchResults();
    },
  },
  created() {
    this.getNumColumns();
  }
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
  box-shadow: 0 4px 8px rgba(250, 93, 93, 0.2);
}

.card {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: 900px;
  margin: 0 auto;
  margin-top: 20px;
  margin-bottom: 40px;
  padding: 10px; /* Add padding to create space inside the card */
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

.from-label {
  padding-bottom: 5px;
}

.form-label-text {
  font-weight: bold;
  font-size: 20px; /* Adjust the font size as needed */
}

.main-container {
  box-sizing: border-box; /* Include padding and border in the element's total width and height */
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: calc(100% - 20px); /* Adjust the width to fit within the card */
  margin: 0 auto;
  margin-top: 20px;
  margin-bottom: 40px;
}

.table-title {
  text-align: center;
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 10px;
}

.table {
  width: 100%;
  border-collapse: collapse;
}

.table th,
.table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.table th {
  background-color: #f2f2f2;
}

.value {
  font-style: italic;
  color: #333;
}

.form {
  margin-top: 20px;
}

.form-group {
  margin-bottom: 15px;
  box-sizing: border-box;
  border-radius: 10px;
  padding: 10px;
  width: 100%;
}

.table {
  width: 100%;
  border-collapse: collapse;
}

.table th,
.table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.table th {
  background-color: #f2f2f2;
}

button {
  padding: 10px 20px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #ddd;
  cursor: not-allowed;
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000; /* Adjust z-index as needed */
}

.loading-spinner {
  border: 8px solid #f3f3f3;
  border-top: 8px solid #3498db;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
