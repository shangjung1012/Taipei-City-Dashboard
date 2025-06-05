<template>
  <div class="location-selector">
    <div class="category-search-row">
      <div class="form-field">
        <label for="city">縣市：</label>
        <select
          id="city"
          v-model="cityFilter"
          class="search-box"
        >
          <option
            v-for="city in cityOptions"
            :key="city"
            :value="city"
          >
            {{ city }}
          </option>
        </select>
      </div>
      <div class="form-field">
        <label for="category">景點分類：</label>
        <select
          id="category"
          v-model="category"
          class="search-box"
        >
          <option
            v-for="cat in categoryOptions"
            :key="cat"
            :value="cat"
          >
            {{ cat }}
          </option>
        </select>
      </div>

      <div class="form-field">
        <label for="search">輸入關鍵字：</label>
        <input
          id="search"
          v-model="searchQuery"
          type="text"
          placeholder="請輸入地點關鍵字..."
          class="search-box"
        >
      </div>

      <div class="form-field">
        <label for="location">可選地點：</label>
        <select
          id="location"
          v-model="selectedDropdownId"
          class="search-box"
        >
          <option
            disabled
            value="__placeholder__"
          >
            請選擇地點
          </option>
          <option
            v-for="loc in filteredLocations"
            :key="loc.id"
            :value="String(loc.id)"
          >
            {{ loc.label }}
          </option>
        </select>
      </div>
    </div>

    <div class="selected-tags">
      <span
        v-for="id in selectedIds"
        :key="id"
        class="tag"
      >
        {{ locationMapAll[id]?.label || "未知地點" }}
        <button
          class="remove-tag"
          @click="removeTag(id)"
        >×</button>
      </span>
    </div>

    <div class="preferences">
      <h4>額外需求：</h4>
      <label><input
        v-model="preferences"
        type="checkbox"
        value="accessible"
      >
        無障礙環境</label>
      <label><input
        v-model="preferences"
        type="checkbox"
        value="family"
      >
        親子友善</label>
      <label><input
        v-model="preferences"
        type="checkbox"
        value="restroom"
      >
        附近有廁所</label>
      <label><input
        v-model="preferences"
        type="checkbox"
        value="hotel"
      >
        附近有旅館</label>
    </div>

    <button
      class="search-button"
      @click="onSearchClick"
    >
      搜尋推薦地點
    </button>

    <div v-if="showResult">
      <!-- 完全符合的推薦地點 -->
      <div v-if="matchedResults.length">
        <h3>推薦地點</h3>
        <table class="results">
          <thead>
            <tr>
              <th>地點</th>
              <th>縣市</th>
              <th>無障礙環境</th>
              <th>親子友善</th>
              <th>附近有廁所</th>
              <th>附近有旅館</th>
              <th>車位數指標</th>
              <th>治安指標</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(loc, index) in matchedResults"
              :key="loc.id"
              :class="{
                'top-1': index === 0,
                'top-2': index === 1,
                'top-3': index === 2,
              }"
            >
              <td>
                <span v-if="index === 0">🥇</span>
                <span v-else-if="index === 1">🥈</span>
                <span v-else-if="index === 2">🥉</span>
                {{ loc.label }}
              </td>
              <td>{{ loc.city }}</td>
              <td>{{ loc.accessible ? "有" : "無" }}</td>
              <td>{{ loc.family ? "有" : "無" }}</td>
              <td>{{ loc.restroom ? "有" : "無" }}</td>
              <td>{{ loc.hotel ? "有" : "無" }}</td>
              <td class="bar-with-label">
                <div class="bar-container">
                  <div
                    class="bar-fill car"
                    :style="{
                      width: loc.carspace * 10 + '%',
                    }"
                  />
                </div>
                <span class="bar-label">{{
                  loc.carspace
                }}</span>
              </td>
              <td class="bar-with-label">
                <div class="bar-container">
                  <div
                    class="bar-fill safety"
                    :style="{
                      width: loc.security * 10 + '%',
                    }"
                  />
                </div>
                <span class="bar-label">{{
                  loc.security
                }}</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <!-- 部分符合的地點 -->
      <div
        v-if="unmatchedResults.length"
        style="margin-top: 2rem"
      >
        <h3>未完全符合額外條件</h3>
        <table class="results">
          <thead>
            <tr>
              <th>地點</th>
              <th>縣市</th>
              <th>無障礙環境</th>
              <th>親子友善</th>
              <th>附近有廁所</th>
              <th>附近有旅館</th>
              <th>車位數指標</th>
              <th>治安指標</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="loc in unmatchedResults"
              :key="loc.id"
            >
              <td>{{ loc.label }}</td>
              <td>{{ loc.city }}</td>
              <td>{{ loc.accessible ? "有" : "無" }}</td>
              <td>{{ loc.family ? "有" : "無" }}</td>
              <td>{{ loc.restroom ? "有" : "無" }}</td>
              <td>{{ loc.hotel ? "有" : "無" }}</td>
              <td class="bar-with-label">
                <div class="bar-container">
                  <div
                    class="bar-fill car"
                    :style="{
                      width: loc.carspace * 10 + '%',
                    }"
                  />
                </div>
                <span class="bar-label">{{
                  loc.carspace
                }}</span>
              </td>
              <td class="bar-with-label">
                <div class="bar-container">
                  <div
                    class="bar-fill safety"
                    :style="{
                      width: loc.security * 10 + '%',
                    }"
                  />
                </div>
                <span class="bar-label">{{
                  loc.security
                }}</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <p class="indicator-note">
        ※
        有無障礙環境、親子友善、附近有廁所、附近有旅館等指標，為查詢附近
        1 公里內有無相關設施之情形。<br>
        ※「車位數指標」代表該地點在縣市中的停車便利程度，指標 10
        代表是該縣市中最容易停車的 10% 地點，指標 1 則為最難停車的 10%
        地點。<br>
        ※「治安指標」來自114年竊盜案件統計， 指標 10 代表是案件最少的
        10% 地點，指標 1 則為該縣市中竊盜案件較多的 10% 地點。
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, nextTick, onMounted } from "vue";

const searchQuery = ref("");
const selectedIds = ref([]);
const selectedDropdownId = ref("__placeholder__");
const preferences = ref([]);
const showResult = ref(false);
const committedIds = ref([]);
const committedPreferences = ref([]);
const cityFilter = ref("全部縣市");
const cityOptions = ["全部縣市", "臺北市", "新北市"];
const category = ref("全部景點");
const categoryOptions = [
	"全部景點",
	"熱門景點",
	"藝文古蹟",
	"自然生態",
	"夜市",
];

watch(selectedDropdownId, (newId) => {
	if (newId !== "__placeholder__") {
		selectedIds.value.push(Number(newId));
		nextTick(() => {
			selectedDropdownId.value = "__placeholder__";
		});
	}
});

const rawData = ref([]);

onMounted(async () => {
	const files = [
		"/mapData/attraction_tpe.geojson",
		"/mapData/attraction_new_tpe.geojson",
	];
	const allData = [];

	for (const path of files) {
		const res = await fetch(path);
		const geo = await res.json();

		geo.features.forEach((feature, idx) => {
			const props = feature.properties;
			const address = props["地址"] || "";
			const city = address.startsWith("新北市") ? "新北市" : "臺北市";
			allData.push({
				id: allData.length + 1,
				label: props["館所名稱"] || "",
				category: props["建物屬性"] || "",
				city,
				accessible: props["無障礙"] || false,
				family: props["family"] || false,
				restroom: props["toilet"] || false,
				hotel: props["hotel"] || false,
				carspace: props["parking_score"] || 0,
				security: props["steal_score"] || 0,
			});
		});
	}

	rawData.value = allData;
});

const locationOptions = computed(() => {
	return rawData.value.filter(
		(loc) =>
			(cityFilter.value === "全部縣市" ||
				loc.city === cityFilter.value) &&
			(category.value === "全部景點" || loc.category === category.value)
	);
});

const locationMap = computed(() =>
	Object.fromEntries(locationOptions.value.map((l) => [l.id, l]))
);
const locationMapAll = computed(() =>
	Object.fromEntries(rawData.value.map((l) => [l.id, l]))
);

function onSearchClick() {
	showResult.value = true;
	committedIds.value = [...selectedIds.value];
	committedPreferences.value = [...preferences.value];
	selectedIds.value = [];
	preferences.value = [];
	searchQuery.value = "";
}

const filteredLocations = computed(() => {
	const query = searchQuery.value.toLowerCase();
	return rawData.value
		.filter(
			(loc) =>
				cityFilter.value === "全部縣市" || loc.city === cityFilter.value
		)
		.filter(
			(loc) =>
				(category.value === "全部景點" ||
					loc.category === category.value) &&
				loc.label.toLowerCase().includes(query) &&
				!selectedIds.value.includes(loc.id)
		);
});

const matchedResults = computed(() => {
	return committedIds.value
		.map((id) => locationMapAll.value[id])
		.filter((loc) => committedPreferences.value.every((pref) => loc[pref]))
		.sort((a, b) => {
			if (b.carspace !== a.carspace) return b.carspace - a.carspace;
			return a.security - b.security;
		});
});

const unmatchedResults = computed(() => {
	return committedIds.value
		.map((id) => locationMapAll.value[id])
		.filter((loc) => !committedPreferences.value.every((pref) => loc[pref]))
		.map((loc) => ({
			...loc,
			missing: committedPreferences.value.filter((pref) => !loc[pref]),
		}))
		.sort((a, b) => {
			if (b.carspace !== a.carspace) return b.carspace - a.carspace;
			return a.security - b.security;
		});
});

function removeTag(id) {
	const index = selectedIds.value.indexOf(id);
	if (index >= 0) selectedIds.value.splice(index, 1);
}
</script>

<style>
h3 {
	font-size: 1.4rem;
	margin-top: 1.5rem;
	margin-bottom: 1rem;
	color: #fff;
}

.dropdown-wrapper {
	display: inline-block;
	width: 68%;
}
.category-search-row {
	display: flex;
	gap: 1rem;
	align-items: flex-start;
	margin-bottom: 1rem;
}
.form-field {
	flex: 1;
	display: flex;
	flex-direction: column;
}

.location-selector {
	max-height: 90vh;
	overflow-y: auto;
	width: 1200px;
	width: 1200px;
	margin: auto;
	padding: 1.5rem;
	background-color: #1f1f1f;
	color: #f0f0f0;
	border-radius: 8px;
	box-shadow: 0 0 10px rgba(0, 0, 0, 0.4);
	font-family: "Noto Sans TC", sans-serif;
}
.search-box {
	width: 100%;
	height: 2.5rem;
	padding: 0 0.75rem;
	font-size: 1rem;
	border-radius: 4px;
	background-color: #2a2a2a;
	color: #fff;
	border: 1px solid #444;
	box-sizing: border-box;
}

.preferences {
	margin: 1rem 0;
	padding: 0.5rem;
	background-color: #2e2e2e;
	border-radius: 4px;
	border: 1px solid #444;
}
.preferences label {
	margin-right: 1rem;
}
.search-button {
	background-color: #2979ff;
	color: white;
	border: none;
	padding: 0.6rem 1.2rem;
	border-radius: 5px;
	cursor: pointer;
	transition: background-color 0.2s;
	font-weight: bold;
	margin-top: 1rem;
}
.search-button:hover {
	background-color: #5393ff;
}
/* === .results 區塊：表格 === */
.results table {
	margin: 0 auto;
	border-collapse: collapse;
	width: auto;
}

.results th,
.results td {
	border: 1px solid #555;
	text-align: center;
	white-space: nowrap;
	padding: 0.6rem 1.2rem;
	padding-inline: clamp(0.8rem, calc(0.5rem + 1ch), 2rem);
}

.results th {
	background: #3a3a3a;
	color: #fff;
	font-weight: bold;
}

.results tr:nth-child(even) {
	background: #262626;
}
.tag {
	background-color: #444;
	color: #fff;
	border-radius: 12px;
	padding: 0.2rem 0.6rem;
	font-size: 0.85rem;
	display: inline-flex;
	align-items: center;
	margin-right: 0.4rem;
	margin-bottom: 0.2rem;
	white-space: nowrap;
}

.remove-tag {
	background: none;
	border: none;
	color: #ccc;
	font-size: 1rem;
	margin-left: 0.4rem;
	cursor: pointer;
}
.remove-tag:hover {
	color: red;
}
.indicator-note {
	font-size: 0.9rem;
	color: #ccc;
	margin-top: 1.5rem;
	line-height: 1.6;
	padding: 0.5rem 1rem;
	background-color: #2a2a2a;
	border-left: 4px solid #2979ff;
	border-radius: 4px;
}
.bar-container {
	width: 100px;
	height: 12px;
	background-color: #444;
	border-radius: 6px;
	overflow: hidden;
	margin: auto;
	box-shadow: inset 0 0 3px #000;
}

.bar-fill {
	height: 100%;
	transition: width 0.3s ease;
}

.bar-fill.car {
	background-color: #4caf50;
}

.bar-fill.safety {
	background-color: #2196f3;
}
table.results tr.top-1 {
	background-color: #2e3f1f !important;
	font-weight: bold;
	color: #ffd700;
}
table.results tr.top-2 {
	background-color: #2f2f3f !important;
	font-weight: bold;
	color: #c0c0c0;
}
table.results tr.top-3 {
	background-color: #3f2f2f !important;
	font-weight: bold;
	color: #cd7f32;
}
table.results tr.top-1 td,
table.results tr.top-2 td,
table.results tr.top-3 td {
	background-color: inherit !important;
}
</style>
