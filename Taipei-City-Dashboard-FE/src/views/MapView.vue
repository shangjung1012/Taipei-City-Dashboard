<!-- Developed By Taipei Urban Intelligence Center 2023-2024 -->
<!-- 
Lead Developer:  Igor Ho (Full Stack Engineer)
Data Pipelines:  Iima Yu (Data Scientist)
Design and UX: Roy Lin (Fmr. Consultant), Chu Chen (Researcher)
Systems: Ann Shih (Systems Engineer)
Testing: Jack Huang (Data Scientist), Ian Huang (Data Analysis Intern) 
-->
<!-- Department of Information Technology, Taipei City Government -->

<!-- Map charts will be hidden in mobile mode and be replaced with the mobileLayers dialog -->

<script setup>
/* global gtag */
import { computed, ref, watch } from "vue";
import { useRoute } from "vue-router";
import DashboardComponent from "../dashboardComponent/DashboardComponent.vue";
import { useContentStore } from "../store/contentStore";
import { useDialogStore } from "../store/dialogStore";
import { useMapStore } from "../store/mapStore";
import MapAnalysisAgent from "../components/map/MapAnalysisAgent.vue";
import MapContainer from "../components/map/MapContainer.vue";
import MoreInfo from "../components/dialogs/MoreInfo.vue";
import ReportIssue from "../components/dialogs/ReportIssue.vue";

const contentStore = useContentStore();
const dialogStore = useDialogStore();
const mapStore = useMapStore();
const route = useRoute();

const toggleOn = ref({
	hasMap: [],
	noMap: [],
	mapLayer: [],
	basicLayer: [],
});

// Separate components with maps from those without
const parseMapLayers = computed(() => {
	const hasMap = contentStore.currentDashboard.components?.filter(
		(item) => item.map_config[0],
	);
	const noMap = contentStore.currentDashboard.components?.filter(
		(item) => !item.map_config[0],
	);

	return { hasMap: hasMap, noMap: noMap };
});

watch(
	() => [route.query.index, route.query.city],
	async ([newIndex, newCity], [oldIndex, oldCity]) => {
		if (newIndex !== oldIndex || newCity !== oldCity) {
			await mapStore.clearIndexedDB();
			toggleOn.value = {
				hasMap: new Array(parseMapLayers.value.hasMap?.length).fill(
					false,
				),
				noMap: new Array(parseMapLayers.value.noMap?.length).fill(
					false,
				),
				mapLayer: new Array(
					contentStore.currentDashboard.components?.length,
				).fill(false),
				basicLayer: new Array(contentStore.mapLayers?.length).fill(
					false,
				),
			};
		}
	},
);

function handleOpenSettings() {
	contentStore.editDashboard = JSON.parse(
		JSON.stringify(contentStore.currentDashboard),
	);
	dialogStore.addEdit = "edit";
	dialogStore.showDialog("addEditDashboards");
}

async function handleDebugIndexedDB() {
	const data = await mapStore.getExtractedFeaturesFromIndexedDB();
	if (!data) {
		console.warn("IndexedDB 中沒有可顯示的資料");
		return;
	}

	console.log("IndexedDB debug data:", data);
}

// Open and closes the component as well as communicates to the mapStore to turn on and off map layers
function handleToggle(value, map_config, componentConfig) {
	if (!map_config[0]) {
		if (value) {
			dialogStore.showNotification(
				"info",
				"本組件沒有空間資料，不會渲染地圖",
			);
		}
		return;
	}
	if (value) {
		mapStore.addToMapLayerList(map_config);
		mapStore.addComponentToIndexedDB(componentConfig, map_config);
	} else {
		mapStore.clearByParamFilter(componentConfig, map_config);
		mapStore.turnOffMapLayerVisibility(map_config);
	}
}

function toggleSwitchBtn(value, Btn, BtnIndex) {
	toggleOn.value[Btn][BtnIndex] = value;
}

function shouldDisable(map_config) {
	const allMapLayerIds = map_config.map(
		(el) => `${el.index}-${el.type}-${el.city}`,
	);
	if (mapStore.isPreloading === true) {
		return true;
	} else {
		return (
			mapStore.loadingLayers.filter((el) => allMapLayerIds.includes(el))
				.length > 0
		);
	}
}

// 開啟主題圖層時觸發GA自訂事件
function popularThematicLayerGA(map_config) {
	if (map_config[0].city && map_config[0].title) {
		gtag("event", "popular_thematic_layer", {
			dashboard_city: map_config[0].city,
			layer_name: map_config[0].title,
			city_layer: `${map_config[0].city}-${map_config[0].title}`,
			time: Date.now(),
		});
	}
}

// 開啟基本圖層時觸發GA自訂事件
function popularBasicLayerGA(map_config) {
	if (map_config[0].city && map_config[0].title) {
		gtag("event", "popular_basic_layer", {
			dashboard_city: map_config[0].city,
			layer_name: map_config[0].title,
			city_layer: `${map_config[0].city}-${map_config[0].title}`,
			time: Date.now(),
		});
	}
}

/* District Filter Panel */
// 行政區資料
const taipeiDistricts = [
	"松山區",
	"信義區",
	"大安區",
	"中山區",
	"中正區",
	"大同區",
	"萬華區",
	"文山區",
	"南港區",
	"內湖區",
	"士林區",
	"北投區",
];

const newtaipeiDistricts = [
	"新莊區",
	"淡水區",
	"汐止區",
	"板橋區",
	"三重區",
	"樹林區",
	"土城區",
	"蘆洲區",
	"中和區",
	"永和區",
	"新店區",
	"鶯歌區",
	"三峽區",
	"瑞芳區",
	"五股區",
	"泰山區",
	"林口區",
	"深坑區",
	"石碇區",
	"坪林區",
	"三芝區",
	"石門區",
	"八里區",
	"平溪區",
	"雙溪區",
	"貢寮區",
	"金山區",
	"萬里區",
	"烏來區",
];

// 篩選面板狀態
const selectedCity = ref("");
const selectedDistrict = ref("");

// 根據選擇的城市顯示對應的行政區
const districtOptions = computed(() => {
	if (selectedCity.value === "taipei") {
		return taipeiDistricts;
	} else if (selectedCity.value === "newtaipei") {
		return newtaipeiDistricts;
	}
	return [];
});

// 當城市改變時，清空行政區選擇
watch(
	() => selectedCity.value,
	() => {
		selectedDistrict.value = "";
	},
);

function getToggleOnMapComponents(requireMapFilter = false) {
	return (
		contentStore.currentDashboard.components?.filter((component) => {
			if (!component.map_config?.[0]) return false;

			const hasMapIdx = parseMapLayers.value.hasMap?.indexOf(component);
			const isToggleOn =
				hasMapIdx !== undefined &&
				hasMapIdx !== -1 &&
				toggleOn.value.hasMap?.[hasMapIdx];

			return isToggleOn && (!requireMapFilter || component.map_filter);
		}) || []
	);
}

// 對所有打開的 component 執行 filterByParam
function applyDistrictFilter() {
	if (!selectedDistrict.value) return;

	const componentsToFilter = getToggleOnMapComponents(true);

	// 對每個符合條件的 component 執行 filterByParam
	componentsToFilter.forEach((component) => {
		// 确保使用正確的 map_config
		const map_config = component.map_config;
		if (!map_config) return;

		mapStore.filterByParam(
			component,
			component.map_filter,
			map_config,
			selectedDistrict.value, // xParam: 行政區名稱
			null, // yParam: 不使用
		);
	});
}

// 清除篩選
async function clearDistrictFilter() {
	const toggleOnComponents = getToggleOnMapComponents(false);
	selectedCity.value = "";
	selectedDistrict.value = "";

	// 清除目前打開 component 的篩選，回到 all
	await Promise.all(
		toggleOnComponents.map((component) => {
			if (component.map_config && component.map_filter) {
				return mapStore.clearByParamFilter(
					component,
					component.map_config,
				);
			}
			return Promise.resolve();
		}),
	);

	// 重新把目前打開 component 的完整資料寫回 IndexedDB
	await Promise.all(
		toggleOnComponents.map((component) =>
			mapStore.addComponentToIndexedDB(component, component.map_config),
		),
	);
}
</script>

<template>
	<div class="map">
		<div class="hide-if-mobile">
			<!-- District Filter Panel -->
			<div class="district-filter-panel">
				<div class="district-filter-content">
					<span class="filter-icon">tune</span>
					<div class="filter-selects">
						<select
							v-model="selectedCity"
							class="district-select"
							@change="selectedDistrict = ''"
						>
							<option value="">全部</option>
							<option value="taipei">台北市</option>
							<option value="newtaipei">新北市</option>
						</select>
						<select
							v-if="selectedCity && districtOptions.length > 0"
							v-model="selectedDistrict"
							class="district-select"
						>
							<option value="">選擇行政區</option>
							<option
								v-for="district in districtOptions"
								:key="district"
								:value="district"
							>
								{{ district }}
							</option>
						</select>
					</div>
					<button
						v-if="selectedCity"
						class="search-filter-btn"
						:disabled="!selectedDistrict"
						@click="applyDistrictFilter"
						title="搜尋行政區"
					>
						search
					</button>
					<button
						v-if="selectedCity || selectedDistrict"
						class="clear-filter-btn"
						@click="clearDistrictFilter"
						title="清除行政區篩選"
					>
						close
					</button>
				</div>
			</div>

			<!-- <div class="map-debug-actions">
        <button
          type="button"
          class="map-debug-button"
          @click="handleDebugIndexedDB"
        >
          Debug IndexedDB
        </button>
      </div> -->
			<!-- 1. If the dashboard is map-layers -->
			<div
				v-if="
					contentStore.currentDashboard.index?.includes('map-layers')
				"
				class="map-charts"
			>
				<DashboardComponent
					v-for="(item, arrayIdx) in contentStore.currentDashboard
						.components"
					:key="`map-layer-${item.index}-${item.city}`"
					:config="item"
					mode="halfmap"
					:info-btn="true"
					:active-city="item.city"
					:select-btn="true"
					:select-btn-disabled="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						).length === 1
					"
					:select-btn-list="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						)
					"
					:city-tag="
						contentStore.cityManager.getTagList(
							contentStore.currentDashboard?.city,
						)
					"
					:toggle-disable="shouldDisable(item.map_config)"
					:toggle-on="toggleOn.mapLayer[arrayIdx]"
					@info="
						(item) => {
							dialogStore.showMoreInfo(item);
						}
					"
					@toggle="
						(value, map_config) => {
							handleToggle(value, map_config, item);
							toggleSwitchBtn(value, 'mapLayer', arrayIdx);
							popularThematicLayerGA(map_config);
						}
					"
					@filter-by-param="
						(map_filter, map_config, x, y) => {
							mapStore.filterByParam(
								item,
								map_filter,
								map_config,
								x,
								y,
							);
						}
					"
					@filter-by-layer="
						(map_config, layer) => {
							mapStore.filterByLayer(map_config, layer);
						}
					"
					@clear-by-param-filter="
						(map_config, componentConfig) => {
							mapStore.clearByParamFilter(
								componentConfig || item,
								map_config,
							);
						}
					"
					@clear-by-layer-filter="
						(map_config) => {
							mapStore.clearByLayerFilter(map_config);
						}
					"
					@change-city="
						(city) => {
							const selectedData =
								contentStore.cityDashboard.components.find(
									(data) => {
										if (
											data.index === item.index &&
											data.city === city
										) {
											return data;
										}
									},
								);

							const componentIndex =
								contentStore.currentDashboard.components.findIndex(
									(item) => item.id === selectedData.id,
								);

							if (selectedData) {
								mapStore.clearByParamFilter(
									item,
									item.map_config,
								);
								mapStore.turnOffMapLayerVisibility(
									item.map_config,
								);
								mapStore.addToMapLayerList(
									selectedData.map_config,
								);

								contentStore.setComponentData(
									componentIndex,
									selectedData,
								);
							}
						}
					"
				/>
			</div>
			<!-- 2. Dashboards that have components -->
			<div
				v-else-if="
					contentStore.currentDashboard.components?.length !== 0
				"
				class="map-charts"
			>
				<DashboardComponent
					v-for="(item, arrayIdx) in parseMapLayers.hasMap"
					:key="`map-layer-${item.index}-${item.city}`"
					:config="item"
					mode="map"
					:info-btn="true"
					:active-city="item.city"
					:select-btn="true"
					:select-btn-disabled="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						).length === 1 ||
						contentStore.currentDashboardExcluded.components.filter(
							(data) => data.index === item.index,
						).length === 0
					"
					:select-btn-list="
						contentStore.currentDashboard?.city
							? contentStore.cityManager.getSelectList(
									contentStore.currentDashboard?.city,
								)
							: contentStore.cityManager.getCities(
									contentStore.cityManager.activeCities,
								)
					"
					:city-tag="
						contentStore.currentDashboard?.city
							? contentStore.cityManager.getTagList(
									contentStore.currentDashboard?.city,
								)
							: contentStore.cityManager.getTagList(item.city)
					"
					:toggle-disable="shouldDisable(item.map_config)"
					:toggle-on="toggleOn.hasMap[arrayIdx]"
					@info="
						(item) => {
							dialogStore.showMoreInfo(item);
						}
					"
					@toggle="
						(value, map_config) => {
							handleToggle(value, map_config, item);
							toggleSwitchBtn(value, 'hasMap', arrayIdx);
							popularThematicLayerGA(map_config);
						}
					"
					@filter-by-param="
						(map_filter, map_config, x, y) => {
							mapStore.filterByParam(
								item,
								map_filter,
								map_config,
								x,
								y,
							);
						}
					"
					@filter-by-layer="
						(map_config, layer) => {
							mapStore.filterByLayer(map_config, layer);
						}
					"
					@clear-by-param-filter="
						(map_config, componentConfig) => {
							mapStore.clearByParamFilter(
								componentConfig || item,
								map_config,
							);
						}
					"
					@clear-by-layer-filter="
						(map_config) => {
							mapStore.clearByLayerFilter(map_config);
						}
					"
					@fly="
						(location) => {
							mapStore.flyToLocation(location);
						}
					"
					@change-city="
						(city) => {
							const selectedData =
								contentStore.cityDashboard.components.find(
									(data) => {
										if (
											data.index === item.index &&
											data.city === city
										) {
											return data;
										}
									},
								);

							const componentIndex =
								contentStore.currentDashboard.components.findIndex(
									(item) => item.id === selectedData.id,
								);

							if (selectedData) {
								mapStore.clearByParamFilter(
									item,
									item.map_config,
								);
								mapStore.turnOffMapLayerVisibility(
									item.map_config,
								);
								mapStore.addToMapLayerList(
									selectedData.map_config,
								);

								contentStore.setComponentData(
									componentIndex,
									selectedData,
								);
							}
						}
					"
				/>
				<h2 v-if="contentStore.mapLayers.length > 0">基本圖層</h2>
				<DashboardComponent
					v-for="(item, arrayIdx) in contentStore.mapLayers"
					:key="`map-layer-${item.index}-${item.city}`"
					:config="item"
					mode="halfmap"
					:info-btn="true"
					:active-city="item.city"
					:select-btn="true"
					:select-btn-disabled="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						).length === 1
					"
					:select-btn-list="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						)
					"
					:city-tag="
						contentStore.cityManager.getTagList(
							contentStore.currentDashboard?.city,
						)
					"
					:toggle-disable="shouldDisable(item.map_config)"
					:toggle-on="toggleOn.basicLayer[arrayIdx]"
					@info="
						(item) => {
							dialogStore.showMoreInfo(item);
						}
					"
					@toggle="
						(value, map_config) => {
							handleToggle(value, map_config, item);
							toggleSwitchBtn(value, 'basicLayer', arrayIdx);
							popularBasicLayerGA(map_config);
						}
					"
					@filter-by-param="
						(map_filter, map_config, x, y) => {
							mapStore.filterByParam(
								item,
								map_filter,
								map_config,
								x,
								y,
							);
						}
					"
					@filter-by-layer="
						(map_config, layer) => {
							mapStore.filterByLayer(map_config, layer);
						}
					"
					@clear-by-param-filter="
						(map_config, componentConfig) => {
							mapStore.clearByParamFilter(
								componentConfig || item,
								map_config,
							);
						}
					"
					@clear-by-layer-filter="
						(map_config) => {
							mapStore.clearByLayerFilter(map_config);
						}
					"
					@change-city="
						(city) => {
							const selectedData = contentStore.allMapLayers.find(
								(data) => {
									if (
										data.index === item.index &&
										data.city === city
									) {
										return data;
									}
								},
							);

							if (selectedData) {
								mapStore.clearByParamFilter(
									item,
									item.map_config,
								);
								mapStore.turnOffMapLayerVisibility(
									item.map_config,
								);
								mapStore.addToMapLayerList(
									selectedData.map_config,
								);

								contentStore.setMapLayerData(
									arrayIdx,
									selectedData,
								);
							}
						}
					"
				/>
				<h2 v-if="parseMapLayers.noMap?.length > 0">無空間資料組件</h2>
				<DashboardComponent
					v-for="(item, arrayIdx) in parseMapLayers.noMap"
					:key="`map-layer-${item.index}-${item.city}`"
					:config="item"
					mode="map"
					:info-btn="true"
					:active-city="item.city"
					:select-btn="true"
					:select-btn-disabled="
						contentStore.cityManager.getSelectList(
							contentStore.currentDashboard?.city,
						).length === 1 ||
						contentStore.currentDashboardExcluded.components.filter(
							(data) => data.index === item.index,
						).length === 0
					"
					:select-btn-list="
						contentStore.currentDashboard?.city
							? contentStore.cityManager.getSelectList(
									contentStore.currentDashboard?.city,
								)
							: contentStore.cityManager.getCities(
									contentStore.cityManager.activeCities,
								)
					"
					:city-tag="
						contentStore.currentDashboard?.city
							? contentStore.cityManager.getTagList(
									contentStore.currentDashboard?.city,
								)
							: contentStore.cityManager.getTagList(item.city)
					"
					:toggle-on="toggleOn.noMap[arrayIdx]"
					@info="
						(item) => {
							dialogStore.showMoreInfo(item);
						}
					"
					@toggle="
						(value, map_config) => {
							handleToggle(value, map_config, item);
							toggleSwitchBtn(value, 'noMap', arrayIdx);
						}
					"
					@change-city="
						(city) => {
							const selectedData =
								contentStore.cityDashboard.components.find(
									(data) => {
										if (
											data.index === item.index &&
											data.city === city
										) {
											return data;
										}
									},
								);
							const componentIndex =
								contentStore.currentDashboard.components.findIndex(
									(data) =>
										data.index === item.index &&
										data.city === item.city,
								);
							if (selectedData && componentIndex !== -1) {
								contentStore.setComponentData(
									componentIndex,
									selectedData,
								);
							}
						}
					"
				/>
			</div>
			<!-- 3. If dashboard is still loading -->
			<div
				v-else-if="contentStore.loading"
				class="map-charts-nodashboard"
			>
				<div />
			</div>
			<!-- 4. If dashboard failed to load -->
			<div v-else-if="contentStore.error" class="map-charts-nodashboard">
				<span>sentiment_very_dissatisfied</span>
				<h2>發生錯誤，無法載入儀表板</h2>
			</div>
			<!-- 5. Dashboards that don't have components -->
			<div v-else class="map-charts-nodashboard">
				<span>addchart</span>
				<h2>尚未加入組件</h2>
				<button
					v-if="contentStore.currentDashboard.icon !== 'favorite'"
					class="hide-if-mobile"
					@click="handleOpenSettings"
				>
					加入您的第一個組件
				</button>
				<p v-else>點擊其他儀表板組件之愛心以新增至收藏組件</p>
			</div>
		</div>
		<MapContainer />
		<MapAnalysisAgent />
		<MoreInfo />
		<ReportIssue />
	</div>
</template>

<style scoped lang="scss">
.map {
	height: calc(100vh - 127px);
	height: calc(var(--vh) * 100 - 127px);
	display: flex;
	margin: var(--font-m) var(--font-m);
	/* side column width for map-charts and district panel */
	width: calc(100% - var(--font-m) * 2);
	max-width: calc(100% - var(--font-m) * 2);

	&-debug-actions {
		width: 360px;
		display: flex;
		justify-content: flex-end;
		margin-right: var(--font-s);
		margin-bottom: var(--font-s);

		@media (min-width: 1000px) {
			--side-width: 370px;
		}

		@media (min-width: 2000px) {
			--side-width: 400px;
		}
	}

	&-debug-button {
		padding: 8px 12px;
		border: 1px solid #c7c7c7;
		border-radius: 6px;
		background: #ffffff;
		color: #333;
		font-size: 0.9rem;
		cursor: pointer;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);

		&:hover {
			background: #f5f5f5;
		}
	}
}

.district-filter-panel {
	/* match inner card content width used by DashboardComponent (data-v-c44d4e75) */
	width: calc(var(--side-width) - (var(--font-m) * 2));
	margin-right: var(--font-s);
	margin-bottom: var(--font-m);
	/* use same background as components */
	background: var(--color-component-background);
	/* match dashboardcomponent border radius visually */
	border-radius: 5px;
	box-shadow: none;
	padding: var(--font-s);
	color: #ffffff;

	@media (min-width: 1000px) {
		width: calc(var(--side-width) - (var(--font-m) * 2));
	}

	@media (min-width: 2000px) {
		width: calc(var(--side-width) - (var(--font-m) * 2));
	}

	&-content {
		display: flex;
		align-items: center;
		gap: var(--font-s);
	}

	.filter-icon {
		font-family: var(--font-icon);
		font-size: 1.3rem;
		flex-shrink: 0;
	}

	.district-filter-content {
		display: flex;
		align-items: center;
		gap: var(--font-s);
		flex-wrap: nowrap;
	}

	.filter-selects {
		display: flex;
		gap: var(--font-xs);
		flex: 1;
		min-width: 0; /* allow children to shrink */
	}

	.district-select {
		/* 讓每個 select 能縮小，預設較窄以避免換行 */
		flex: 0 1 120px;
		min-width: 80px;
		padding: 6px 8px;
		border: 1px solid rgba(255, 255, 255, 0.12);
		border-radius: 4px;
		font-size: 0.9rem;
		background: transparent;
		color: #ffffff;
		cursor: pointer;
		transition: border-color 0.2s;

		&:hover {
			border-color: rgba(255, 255, 255, 0.22);
		}

		&:focus {
			outline: none;
			border-color: rgba(255, 255, 255, 0.32);
			box-shadow: 0 0 0 2px rgba(255, 255, 255, 0.04);
		}
	}

	.search-filter-btn,
	.clear-filter-btn {
		width: 32px;
		height: 32px;
		padding: 0;
		border: 1px solid rgba(255, 255, 255, 0.12);
		border-radius: 4px;
		background: transparent;
		color: #ffffff;
		font-family: var(--font-icon);
		font-size: 1.2rem;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		transition: all 0.2s;
		flex-shrink: 0;

		&:hover {
			background: rgba(255, 255, 255, 0.04);
			border-color: rgba(255, 255, 255, 0.22);
		}

		&:active {
			transform: scale(0.95);
		}
	}

	.search-filter-btn {
		&:disabled {
			opacity: 0.35;
			cursor: not-allowed;
		}

		&:disabled:hover {
			background: transparent;
			border-color: rgba(255, 255, 255, 0.12);
		}
	}
}

.map {
	height: calc(100vh - 127px);
	height: calc(var(--vh) * 100 - 127px);
	display: flex;
	margin: var(--font-m) var(--font-m);

	&-debug-actions {
		width: 360px;
		display: flex;
		justify-content: flex-end;
		margin-right: var(--font-s);
		margin-bottom: var(--font-s);

		@media (min-width: 1000px) {
			width: 370px;
		}

		@media (min-width: 2000px) {
			width: 400px;
		}
	}

	&-debug-button {
		padding: 8px 12px;
		border: 1px solid #c7c7c7;
		border-radius: 6px;
		background: #ffffff;
		color: #333;
		font-size: 0.9rem;
		cursor: pointer;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);

		&:hover {
			background: #f5f5f5;
		}
	}

	&-charts {
		width: var(--side-width);
		max-height: 100%;
		height: fit-content;
		display: grid;
		row-gap: var(--font-m);
		margin-right: var(--font-s);
		border-radius: 5px;
		overflow-y: scroll;

		@media (min-width: 1000px) {
			width: 370px;
		}

		@media (min-width: 2000px) {
			width: 400px;
		}

		&-nodashboard {
			width: 360px;
			height: calc(100vh - 127px);
			height: calc(var(--vh) * 100 - 127px);
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;
			margin-right: var(--font-s);

			@media (min-width: 1000px) {
				width: 370px;
			}

			@media (min-width: 2000px) {
				width: 400px;
			}

			span {
				margin-bottom: var(--font-ms);
				font-family: var(--font-icon);
				font-size: 2rem;
			}

			button {
				color: var(--color-highlight);
			}

			div {
				width: 2rem;
				height: 2rem;
				border-radius: 50%;
				border: solid 4px var(--color-border);
				border-top: solid 4px var(--color-highlight);
				animation: spin 0.7s ease-in-out infinite;
			}
		}
	}
}

@keyframes spin {
	to {
		transform: rotate(360deg);
	}
}
</style>
