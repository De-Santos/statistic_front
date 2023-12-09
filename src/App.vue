<template>
  <div class="container-fluid mt-5">
    <div class="card">
      <div class="card-header">
        <h1 class="card-title">Кластеризація розповсюдження коронавірусної інфекції</h1>
      </div>
      <div class="row">
        <div class="card-body" :class="{'col-6': showResult, 'col-12': !showResult}">
          <div class="form-group">
            <label for="dataset">Оберіть файл датасету (csv):</label>
            <div class="custom-file">
              <input type="file" class="custom-file-input" id="dataset" accept=".csv" @change="onFileInputChanged">
            </div>
          </div>

          <form @submit.prevent="submitForm" v-if="fileName">
            <div class="form-group mt-3">
              <label>Оберіть колонки для кластеризації:</label>
              <table class="table">
                <thead>
                <tr>
                  <th></th>
                  <th><b>Назва</b></th>
                  <!--<th><b>Min</b></th>
                  <th><b>Max</b></th>-->
                </tr>
                </thead>
                <tbody>
                <tr v-for="column in numericHeaders">
                  <td>
                    <div class="form-check">
                      <input :id="`checkbox_${column}`" class="form-check-input" type="checkbox"
                             v-model="selectedColumns" @change="handleCheckbox($event)" :value="column"
                             :disabled="!numericHeaders.length">
                    </div>
                  </td>
                  <td>
                    <label :for="`checkbox_${column}`" class="form-check-label">{{ column }}</label>
                  </td>
                  <!--<td>
                    <input
                      type="number"
                      class="input-width-100"
                      @change="changeColumnSettings($event, column, 'min')"
                      :value="columnsSettings[column] && columnsSettings[column].min"
                      :disabled="!columnsSettings[column]"
                    />
                  </td>
                  <td>
                    <input
                      type="number"
                      class="input-width-100"
                      @change="changeColumnSettings($event, column, 'max')"
                      :value="columnsSettings[column] && columnsSettings[column].max"
                      :disabled="!columnsSettings[column]"
                    />
                  </td>-->
                </tr>
                </tbody>
              </table>
            </div>

            <div class="form-group mt-3">
              <label for="k">Введіть число K:</label>
              <input type="number" class="form-control" id="k" v-model="k" :disabled="!numericHeaders.length" required>
            </div>

            <div class="form-group mt-3">
              <label for="distance">Оберіть міру відстані:</label>
              <select class="form-control" id="distance" v-model="selectedDistance" @change="onDistanceChange"
                      :disabled="!numericHeaders.length" required>
                <option v-for="(value, key) in distances" :value="key">{{ value }}</option>
              </select>
            </div>

            <div v-if="requireP" class="form-group mt-3">
              <label for="p">Введіть число P:</label>
              <input type="number" class="form-control" id="p" v-model="p" :disabled="!numericHeaders.length" required>
            </div>

            <div v-if="requireR" class="form-group mt-3">
              <label for="r">Введіть число r:</label>
              <input type="number" class="form-control" id="r" v-model="r" :disabled="!numericHeaders.length" required>
            </div>

            <button class="btn btn-primary mt-3 submit" :disabled="!numericHeaders.length">Кластеризувати</button>
          </form>
        </div>
        <div v-if="showResult" class="col-6 table-wrapper">
          <div id="svgMap" ref="map"></div>
          <table class="table table-primary">
            <thead>
            <tr>
              <th v-for="column in resultColumns"><b>{{ column }}</b></th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="row in resultData">
              <td v-for="index in resultColumns.length">{{
                  (!!parseFloat(row[index - 1]) && parseFloat(row[index - 1]) % 1 !== 0) ? parseFloat(row[index - 1]).toFixed(3) : row[index - 1]
                }}
              </td>
            </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import svgMap from "svgmap";
import isoMapper from 'iso-3166-1';

const API_ROUTE = "http://127.0.0.1:5000";
const WRONG_NAMES_MAPPER = {
  "Brunei": "BN",
  "Burma": "BU",
  "Congo (Brazzaville)": "CG",
  "Congo (Kinshasa)": "CD",
  "Cote d'Ivoire": "CI",
  "Czechia": "CZ",
  "Eswatini": "SZ",
  "Iran": "IR",
  "Kosovo": "XK",
  "Laos": "LA",
  "Moldova": "MD",
  "North Macedonia": "MK",
  "Russia": "RU",
  "South Korea": "KR",
  "Syria": "SY",
  "Taiwan*": "TW",
  "Tanzania": "TZ",
  "US": "US",
  "United Kingdom": "GB",
  "Venezuela": "VE",
  "Vietnam": "VN",
  "West Bank and Gaza": "PS",
};

export default {
  data() {
    return {
      numericHeaders: [],
      selectedColumns: [],
      columnsSettings: {},
      fileName: null,
      showResult: false,
      resultColumns: [],
      resultData: [],
      requireP: false,
      requireR: false,
      countryHeatMap: {},
      k: 1,
      p: 1,
      r: 1,
      selectedDistance: "euclidean",
      API_ROUTE,
      distances: {
        euclidean: "Евклідова",
        squared_euclidean: "Квадрат відстані Евкліда",
        manhattan: "Манхетенська відстань",
        chebyshev: "Відстань Чебишева",
        minkowski: "Відстань Мінковського порядку p",
        power: "Степенева відстань",
      },
      map: {
        defaultCountryFillColor: "#663300",
        highColor: "red",
        lowColor: "white",
        countryStrokeColor: "white",
      },
    };
  },
  methods: {
    async submitForm() {
      const distanceParams = [];
      if (this.requireP) {
        distanceParams.push(this.p);
      }
      if (this.requireR) {
        distanceParams.push(this.r);
      }

      const result = await fetch(`${API_ROUTE}/process_form`, {
        method: 'POST',
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          columns: this.columnsSettings,
          distance: this.selectedDistance,
          kValue: this.k,
          filename: this.fileName,
          distanceParams
        }),
      }).then(response => response.json()).catch(error => {
        console.error('Помилка відправки форми:', error);
      });

      this.showResult = true;
      this.resultColumns = result.columns;
      this.resultData = result.data;
      const countryClusters = this.resultData.map(el => {
        const result = {
          country: el[this.resultColumns.findIndex(el => el === 'Country/Region')],
          cluster: el[this.resultColumns.findIndex(el => el === 'Cluster')],
        }

        for (let i = 0; i < this.selectedColumns.length; i++) {
          const foundIndex = this.resultColumns.findIndex(column => column === this.selectedColumns[i]);
          if (foundIndex !== -1) {
            result[this.selectedColumns[i]] = el[foundIndex];
          }
        }

        return result;
      });
      countryClusters.map(el => {
        let country = isoMapper.whereCountry(el.country) || {alpha2: WRONG_NAMES_MAPPER[el.country]};
        this.countryHeatMap[country.alpha2] = el;
      });
      this.loadMap();
    },
    loadMap() {
      const data = {
        cluster: {
          name: "Cluster",
          format: "{0}",
          thresholdMax: this.k,
          thresholdMin: 0,
        },
      }
      for (let i = 0; i < this.selectedColumns.length; i++) {
        data[this.selectedColumns[i]] = {
          format: "{0}",
          name: this.selectedColumns[i]
        };
      }
      setTimeout(() => {
        new svgMap({
          targetElementID: "svgMap",
          mouseWheelZoomEnabled: false,
          hideFlag: false,
          data: {
            data,
            applyData: "cluster",
            values: this.countryHeatMap,
          },
        });
      }, 0)
    },
    onDistanceChange() {
      if (this.selectedDistance === 'minkowski') {
        this.requireP = true;
        this.requireR = false;
      } else if (this.selectedDistance === 'power') {
        this.requireP = true;
        this.requireR = true;
      } else {
        this.requireP = false;
        this.requireR = false;
      }
    },
    async onFileInputChanged(e) {
      let files = e.target.files || e.dataTransfer.files;
      if (!files.length) {
        return;
      }

      const formData = new FormData();
      formData.append('file', files[0]);

      const resp = await fetch(`${API_ROUTE}/upload_dataset`, {
        method: 'POST',
        body: formData,
      }).then(response => response.json()).catch(error => {
        console.error('Помилка відправки файлу:', error);
      });
      this.showResult = false;
      this.fileName = resp.filename;
      this.numericHeaders = resp.numericHeaders;
    },
    handleCheckbox(event) {
      const status = this.selectedColumns.indexOf(event.target.value) !== -1;
      if (status) {
        this.columnsSettings[event.target.value] = {
          min: 0,
          max: 0
        }
      } else {
        delete this.columnsSettings[event.target.value];
      }
    },
    changeColumnSettings(value, column, key) {
      if (this.columnsSettings[column]) {
        this.columnsSettings[column][key] = +value.target.value;
      }
    }
  },
};
</script>

<style>
#app {
  display: flex;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  color: #2c3e50;
}

.card {
  max-width: 1200px;
  margin: auto;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.card-header {
  background-color: #007bff;
  color: #fff;
  border-radius: 8px 8px 0 0;
}

.card-body {
  padding: 20px;
}

.input-width-100 {
  width: 100px !important;
}

.table-wrapper {
  max-height: 80vh;
  overflow: auto;
  display: inline-block;
}
</style>
