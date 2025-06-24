<template>
  <div
    id="app"
    :class="
      typeof weather.main != 'undefined' && weather.main.temp > 16
        ? 'warm'
        : 'cold'
    "
  >
    <main>
      <div class="search-box">
        <input
          type="text"
          class="search-bar"
          placeholder="Type City Name For Weather Detail..."
          v-model="query"
          @keypress="fetchWeather"
        />
      </div>

      <div v-if="loading" class="loader-container">
        <div class="loader"></div>
      </div>

      <div v-if="error && !loading" class="error">{{ error }}</div>

      <div
        class="weather-wrap"
        v-if="!loading && typeof weather.main != 'undefined'"
      >
        <div class="location-box">
          <div class="location">
            {{ weather.name }}, {{ getCountryName(weather.sys.country) }}
          </div>
          <div class="date">{{ dateFormat() }}</div>
        </div>

        <div class="weather-box">
          <div class="temp">{{ Math.round(weather.main.temp) }}°c</div>
          <div class="weather-image">
            <img
              v-bind:src="`https://api.openweathermap.org/img/w/${weather.weather[0].icon}.png`"
              alt="weather-icon"
            />
          </div>
          <div class="weather">{{ weather.weather[0].main }}</div>
          <div class="weather-description">
            {{ weather.weather[0].description }}
          </div>

          <ul class="weather-detail-list">
            <li><strong>Temperature:</strong> {{ weather.main.temp }} °C</li>
            <li>
              <strong>Feels Like:</strong> {{ weather.main.feels_like }} °C
            </li>
            <li><strong>Humidity:</strong> {{ weather.main.humidity }}%</li>
            <li><strong>Pressure:</strong> {{ weather.main.pressure }} hPa</li>
            <li>
              <strong>Sea Level:</strong>
              {{ weather.main.sea_level || "N/A" }} hPa
            </li>
            <li>
              <strong>Ground Level:</strong>
              {{ weather.main.grnd_level || "N/A" }} hPa
            </li>
            <li><strong>Min Temp:</strong> {{ weather.main.temp_min }} °C</li>
            <li><strong>Max Temp:</strong> {{ weather.main.temp_max }} °C</li>
            <li>
              <strong>Timezone:</strong> {{ formatTimezone(weather.timezone) }}
            </li>
            <li>
              <strong>Sunrise:</strong>
              {{ convertToIST(weather.sys.sunrise) }}
              <!-- {{ formatTime(weather.sys.sunrise, weather.timezone) }} -->
            </li>
            <li>
              <strong>Sunset:</strong>
              {{ convertToIST(weather.sys.sunset) }}
              <!-- {{ formatTime(weather.sys.sunset, weather.timezone) }} -->
            </li>
            <li>
              <strong>Visibility:</strong>
              {{ formatVisibility(weather.visibility) }}
            </li>
          </ul>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
export default {
  name: "app",
  data() {
    return {
      api_key: import.meta.env.VITE_API_TOKEN,
      url_base: "https://api.openweathermap.org/data/2.5/",
      query: "",
      weather: {},
      error: false,
      loading: false,
      loadedOnce: false,
    };
  },
  mounted() {
    const lastCity = localStorage.getItem("lastCity");
    if (lastCity) {
      this.query = lastCity;
      this.fetchWeather({ key: "Enter" });
    } else if (!this.loadedOnce && navigator.geolocation) {
      this.loading = true;
      navigator.geolocation.getCurrentPosition(
        async (position) => {
          const lat = position.coords.latitude;
          const lon = position.coords.longitude;
          await this.fetchWeatherByLocation(lat, lon);
          this.loadedOnce = true;
          this.loading = false;
        },
        (err) => {
          console.warn("Geolocation failed or denied:", err);
          this.loading = false;
          this.error = "Unable to fetch location";
        }
      );
    }
  },
  methods: {
    async fetchWeatherByLocation(lat, lon) {
      this.error = false;
      try {
        const res = await fetch(
          `${this.url_base}weather?lat=${lat}&lon=${lon}&units=metric&APPID=${this.api_key}`
        );
        const data = await res.json();

        if (res.ok && data?.main) {
          this.weather = data;
          this.error = false;
          localStorage.setItem("lastCity", data.name); // ✅ Save city from coordinates
        } else {
          this.weather = {};
          this.error = "City not found";
        }
      } catch (err) {
        console.error("Location-based fetch error:", err);
        this.weather = {};
        this.error = "Failed to fetch data";
      }
    },
    async fetchWeather(e) {
      if (e.key === "Enter" && this.query.trim() !== "") {
        this.error = false;
        this.loading = true;

        try {
          const res = await fetch(
            `${this.url_base}weather?q=${this.query}&units=metric&APPID=${this.api_key}`
          );
          const data = await res.json();
          console.log("object", data);

          if (res.ok && data?.main) {
            this.weather = data;
            this.error = false;
            localStorage.setItem("lastCity", this.query); // ✅ Save city
          } else if (data.cod === "404") {
            this.weather = {};
            this.error = "City not found";
          } else {
            this.weather = {};
            this.error = "Something went wrong";
          }
        } catch (err) {
          console.error("Fetch error:", err);
          this.weather = {};
          this.error = "Failed to fetch data";
        } finally {
          this.loading = false;
          this.query = "";
        }
      }
    },
    setResults(results) {
      this.weather = results;
    },
    dateFormat() {
      const d = new Date();
      const months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
      ];
      const days = [
        "Sunday",
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
        "Saturday",
      ];
      return `${days[d.getDay()]} ${d.getDate()} ${
        months[d.getMonth()]
      } ${d.getFullYear()}`;
    },
    formatTime(unixTime, timezoneOffset) {
      const utc = unixTime * 1000;
      const localTime = new Date(utc + timezoneOffset * 1000);

      return localTime.toLocaleTimeString("en-US", {
        hour: "2-digit",
        minute: "2-digit",
        hour12: true,
      });
    },
    convertToIST(unixTime) {
      return new Date(unixTime * 1000).toLocaleTimeString("en-IN", {
        hour: "2-digit",
        minute: "2-digit",
        hour12: true,
        timeZone: "Asia/Kolkata",
      });
    },

    formatVisibility(value) {
      return `${value / 1000} km`;
    },

    formatTimezone(offset) {
      const hours = offset / 3600;
      const sign = hours >= 0 ? "+" : "-";
      return `UTC${sign}${Math.abs(hours)}`;
    },
    getCountryName(code) {
      const regionNames = new Intl.DisplayNames(["en"], { type: "region" });
      return regionNames.of(code); // e.g., "India"
    },
  },
};
</script>


<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: "montserrat", sans-serif;
}

#app {
  background-size: cover;
  background-position: bottom;
  transition: 0.4s;
}

#app.warm {
  background-image: url("./assets/hot.jpg");
}

#app.cold {
  background-image: url("./assets/cold.jpg");
}

main {
  min-height: 100vh;
  padding: 25px;
  background-image: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.25),
    rgba(0, 0, 0, 0.75)
  );
}

.search-box {
  margin-top: 5%;
  width: 100%;
  margin-bottom: 30px;
}

.search-box .search-bar {
  display: block;
  width: 100%;
  padding: 15px;
  color: #313131;
  font-size: 20px;
  appearance: none;
  border: none;
  outline: none;
  background: none;
  box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.5);
  border-radius: 10px 10px 10px 10px;
  transition: 0.4s;
}

.search-box .search-bar:focus {
  box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.75);
  border-radius: 6px 0px 16px 0px;
}

.error {
  text-align: center;
  color: #fff;
  font-size: 20px;
}

.location-box .location {
  color: #fff;
  font-size: 32px;
  font-weight: 500;
  text-align: center;
  text-shadow: 1px 3px rgba(0, 0, 0, 0.25);
}

.location-box .date {
  color: #fff;
  font-size: 20px;
  font-weight: 300;
  font-style: italic;
  text-align: center;
}

.weather-box {
  text-align: center;
}

.weather-box .temp {
  display: inline-block;
  padding: 10px 25px;
  color: #fff;
  font-size: 102px;
  font-weight: 900;
  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 16px;
  margin: 30px 0px;
  box-shadow: 3px 6px rgba(0, 0, 0, 0.25);
}

.weather-box .weather {
  color: #ffffff;
  font-size: 35px;
  font-weight: 300;
  font-style: italic;
  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
}

.weather-box .weather-description {
  color: #ffffff;
  font-size: 10px;
  font-weight: 300;
  font-style: italic;
}

.weather-box .weather-image {
  position: center;
}

.weather-detail-list {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 20px;
  padding: 16px;
  background-color: rgba(255, 255, 255, 0.2);
  color: #fff;
  border-radius: 8px;
  font-size: 14px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(6px);
}

.weather-detail-list li {
  list-style-type: none;
}

@media (min-width: 1024px) {
  .search-box {
    margin-top: 5%;
    margin-left: auto;
    margin-right: auto;
    align-items: center;
    width: 75%;
    max-width: 800px;
    margin-bottom: 30px;
  }
  .search-box .search-bar {
    display: block;
    width: 100%;
    padding: 15px;
    color: #313131;
    font-size: 20px;
    appearance: none;
    border: none;
    outline: none;
    background: none;
    box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);
    background-color: rgba(255, 252, 252, 0.773);
    border-radius: 10px 10px 10px 10px;
    transition: 0.4s;
  }
  .weather-detail-list {
    font-size: 16px;
  }
}

@media (max-width: 450px) {
  .weather-detail-list {
    grid-template-columns: 1fr;
  }
}

.loader-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 40px;
}

.loader {
  border: 6px solid #f3f3f3;
  border-top: 6px solid #3498db;
  border-radius: 50%;
  width: 48px;
  height: 48px;
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