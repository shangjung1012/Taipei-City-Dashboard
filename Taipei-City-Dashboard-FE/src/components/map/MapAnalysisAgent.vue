<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from "vue";
import http from "../../router/axios";
import { useContentStore } from "../../store/contentStore";
import { useMapStore } from "../../store/mapStore";

const contentStore = useContentStore();
const mapStore = useMapStore();

const isOpen = ref(false);
const isDragging = ref(false);
const isLoading = ref(false);
const hasDataChanged = ref(false);
const analysis = ref("");
const statusText = ref("尚未讀取交叉比較資料");
const sessionId = ref(`map-analysis-${Date.now()}`);
const position = ref({ right: 104, bottom: 24 });
const dragStart = ref({ x: 0, y: 0, right: 104, bottom: 24 });
const dataSignature = ref("");
const dataGroupCount = ref(0);
let pollTimer = null;
const PROXIMITY_RADIUS_METERS = 1000;
const MAX_CENTER_FEATURES = 20;
const MAX_MATCHES_PER_COMPONENT = 5;

const canGenerate = computed(
	() => dataGroupCount.value >= 2 && hasDataChanged.value && !isLoading.value,
);
const generateButtonText = computed(() => {
	if (isLoading.value) return "分析中...";
	if (dataGroupCount.value < 2 || !hasDataChanged.value)
		return "等待資料更新";
	return analysis.value ? "重新生成分析" : "生成目前分析";
});

const activeLayerTitles = computed(() =>
	mapStore.currentVisibleLayers
		.map((layerName) => mapStore.mapConfigs[layerName]?.title || layerName)
		.filter(Boolean),
);

const currentComponents = computed(() => [
	...(contentStore.currentDashboard.components || []),
	...(contentStore.mapLayers || []),
]);

function getComponentSummary() {
	return currentComponents.value.map((component) => ({
		name: component.name,
		city: component.city,
		description: component.long_desc,
		useCase: component.use_case,
		hasMap: Boolean(component.map_config?.[0]),
	}));
}

function getDataSignature(records) {
	if (!records?.length) return "";
	return records
		.map(
			(record) =>
				`${record.componentKey}:${record.timestamp}:${record.featuresCount}`,
		)
		.sort()
		.join("|");
}

function compactFeature(feature) {
	const entries = Object.entries(feature || {})
		.filter(([, value]) =>
			["string", "number", "boolean"].includes(typeof value),
		)
		.slice(0, 12);
	return Object.fromEntries(entries);
}

function toNumber(value) {
	if (value === null || value === undefined || value === "") return null;
	const number = Number(value);
	return Number.isFinite(number) ? number : null;
}

// 計算經緯度
function getFeatureCoordinates(feature) {
	const lng = toNumber(
		feature?.經度 ?? feature?.longitude ?? feature?.lng ?? feature?.lon,
	);
	const lat = toNumber(feature?.緯度 ?? feature?.latitude ?? feature?.lat);

	if (lng === null || lat === null) return null;
	return { lng, lat };
}

// 計算兩點距離
function distanceMeters(a, b) {
	const earthRadius = 6371000;
	const toRadians = (degree) => (degree * Math.PI) / 180;
	const dLat = toRadians(b.lat - a.lat);
	const dLng = toRadians(b.lng - a.lng);
	const lat1 = toRadians(a.lat);
	const lat2 = toRadians(b.lat);
	const h =
		Math.sin(dLat / 2) ** 2 +
		Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLng / 2) ** 2;
	return 2 * earthRadius * Math.atan2(Math.sqrt(h), Math.sqrt(1 - h));
}

function featureName(feature) {
	return (
		feature?.名稱 ||
		feature?.測點名稱 ||
		feature?.事業名稱 ||
		feature?.中文名 ||
		feature?.樹種 ||
		feature?.河流 ||
		feature?.使用分區 ||
		feature?.行政區 ||
		"未命名資料"
	);
}

function buildProximityIntersections(records) {
	// indexedDB 的資料 -> 結構: feature 資料 + 經緯度座標
	const recordsWithCoordinates = (records || [])
		.map((record) => ({
			...record,
			coordinateFeatures: (record.features || [])
				.map((feature) => ({
					feature,
					coordinates: getFeatureCoordinates(feature),
				}))
				.filter((item) => item.coordinates),
		}))
		.filter((record) => record.coordinateFeatures.length > 0);

	if (recordsWithCoordinates.length < 2) {
		return {
			radiusMeters: PROXIMITY_RADIUS_METERS,
			message: "可比較的座標資料少於兩組，未計算鄰近交集。",
			relations: [],
			centers: [],
		};
	}

	const centerRecord = [...recordsWithCoordinates].sort(
		(a, b) => a.coordinateFeatures.length - b.coordinateFeatures.length,
	)[0];
	const targetRecords = recordsWithCoordinates.filter(
		(record) => record.componentKey !== centerRecord.componentKey,
	);

	// 用中心點與其他資料集算距離
	const centers = centerRecord.coordinateFeatures
		.slice(0, MAX_CENTER_FEATURES)
		.map((centerItem) => {
			const matchesByComponent = targetRecords
				.map((targetRecord) => {
					const matches = targetRecord.coordinateFeatures
						.map((targetItem) => ({
							distanceMeters: Math.round(
								distanceMeters(
									centerItem.coordinates,
									targetItem.coordinates,
								),
							),
							featureName: featureName(targetItem.feature),
							feature: compactFeature(targetItem.feature),
							coordinates: targetItem.coordinates,
						}))
						.filter(
							(match) =>
								match.distanceMeters <= PROXIMITY_RADIUS_METERS,
						)
						.sort((a, b) => a.distanceMeters - b.distanceMeters);

					if (matches.length === 0) return null;
					return {
						componentName: targetRecord.componentName,
						totalMatches: matches.length,
						nearestDistanceMeters: matches[0].distanceMeters,
						matches: matches.slice(0, MAX_MATCHES_PER_COMPONENT),
					};
				})
				.filter(Boolean);

			return {
				centerName:
					centerRecord.componentDescription ||
					centerRecord.componentInfo?.long_desc ||
					featureName(centerItem.feature),
				centerFeature: compactFeature(centerItem.feature),
				coordinates: centerItem.coordinates,
				matchesByComponent,
			};
		})
		.filter((center) => center.matchesByComponent.length > 0);

	const relationMap = new Map();
	centers.forEach((center) => {
		center.matchesByComponent.forEach((matchGroup) => {
			const key = `${centerRecord.componentName}->${matchGroup.componentName}`;
			const current = relationMap.get(key) || {
				from: centerRecord.componentName,
				to: matchGroup.componentName,
				matchedCenters: 0,
				totalMatches: 0,
				nearestDistanceMeters: matchGroup.nearestDistanceMeters,
			};
			current.matchedCenters += 1;
			current.totalMatches += matchGroup.totalMatches;
			current.nearestDistanceMeters = Math.min(
				current.nearestDistanceMeters,
				matchGroup.nearestDistanceMeters,
			);
			relationMap.set(key, current);
		});
	});

	return {
		radiusMeters: PROXIMITY_RADIUS_METERS,
		centerComponent: {
			componentKey: centerRecord.componentKey,
			componentName: centerRecord.componentName,
			componentInfo: {
				name:
					centerRecord.componentName ||
					centerRecord.componentInfo?.name,
				long_desc:
					centerRecord.componentDescription ||
					centerRecord.componentInfo?.long_desc,
			},
			totalCoordinateFeatures: centerRecord.coordinateFeatures.length,
			usedCenterFeatures: Math.min(
				centerRecord.coordinateFeatures.length,
				MAX_CENTER_FEATURES,
			),
		},
		relations: Array.from(relationMap.values()),
		centers,
	};
}

function compactIndexedDBRecords(records) {
	return records.map((record) => ({
		componentName: record.componentName,
		componentDescription: record.componentDescription,
		useCase: record.useCase,
		totalRecords: record.featuresCount,
		sample: (record.features || []).slice(0, 3).map(compactFeature),
		updatedAt: record.timestamp
			? new Date(record.timestamp).toLocaleString("zh-TW")
			: null,
	}));
}

function logSpatialIntersections(spatialIntersections) {
	if (!spatialIntersections?.centerComponent) {
		console.log(
			"AI 空間交集計算：目前沒有可比較的中心資料",
			spatialIntersections,
		);
		return;
	}

	console.log("AI 空間交集計算：中心資料摘要", {
		centerComponent: spatialIntersections.centerComponent.componentName,
		totalCenterPoints:
			spatialIntersections.centerComponent.totalCoordinateFeatures,
		usedCenterPoints:
			spatialIntersections.centerComponent.usedCenterFeatures,
		radiusMeters: spatialIntersections.radiusMeters,
	});

	console.table(
		spatialIntersections.centers.map((center) => ({
			centerName: center.centerName,
			lng: center.coordinates.lng,
			lat: center.coordinates.lat,
			targetComponents: center.matchesByComponent.length,
			targetMatches: center.matchesByComponent
				.map((group) => `${group.componentName}: ${group.totalMatches}`)
				.join(", "),
			nearestDistanceMeters: Math.min(
				...center.matchesByComponent.map(
					(group) => group.nearestDistanceMeters,
				),
			),
		})),
	);

	console.log("AI 空間交集計算：完整結果", spatialIntersections);
}

async function readAgentData() {
	const result = await mapStore.getFilteredFeaturesForAgent();
	if (!result.success || !result.data?.length) {
		dataGroupCount.value = 0;
		hasDataChanged.value = false;
		statusText.value = "目前沒有交叉比較資料";
		return [];
	}

	dataGroupCount.value = result.data.length;
	const nextSignature = getDataSignature(result.data);
	if (nextSignature !== dataSignature.value) {
		hasDataChanged.value = true;
	}

	dataSignature.value = nextSignature;
	statusText.value = `已讀取 ${result.data.length} 組資料`;
	return result.data;
}

// function buildSystemPrompt() {
// 	return `你是一位城市環境與空間資料分析顧問，負責判讀排放、水質與植被資料之間的空間訊號。

// 	請用繁體中文回答，對象是非技術決策者。請保留專業判斷，但避免技術名詞與資料格式說明。

// 	分析時請優先檢查：
// 	1. spatialIntersections.relations 是否指出兩個資料表在 1 公里內有交集。
// 	2. spatialIntersections.centers 中每個中心點周邊出現了哪些其他資料。
// 	3. 排放點、水質異常、噪音、動物、植被等資料是否在同一區域形成鄰近訊號。
// 	4. 若資料包含河川或河岸位置，請注意可能的上下游或沿岸關係。

// 	判讀規則：
// 	- 優先根據 spatialIntersections 的 1 公里鄰近計算結果，不要只憑 sample 資料猜測。
// 	- 若 spatialIntersections.relations 為空，請明確說目前沒有 1 公里內的鄰近交集。
// 	- 空間接近只能視為風險線索，不等於因果。
// 	- 不要寫「排放會污染」這類常識句。
// 	- 沒有資料支持時，請明確說不能判斷。
// 	- 每個觀察都要附可信度。

// 	請用以下格式回答：

// 	## 核心判讀
// 	2 到 3 句話。

// 	## 空間訊號
// 	列出 2 到 4 點：
// 	- 訊號：
// 	- 判讀：
// 	- 決策意義：
// 	- 可信度：高／中／低

// 	## 風險與限制
// 	列出目前最容易誤判的地方。

// 	## 建議優先行動
// 	列出 2 到 3 點：
// 	- 優先檢查：
// 	- 行動：
// 	- 指標：`;
// }

function buildSystemPrompt() {
	return `你是一位城市環境與空間資料分析顧問。請只根據使用者提供的 JSON 資料回答，不要自行補不存在的資料。

資料讀取規則：
- centerComponent 名稱請取 spatialIntersections.centerComponent.componentName。
- targetComponent 名稱請取 spatialIntersections.centers[].matchesByComponent[].componentName。
- 每一個有交集的中心點，請逐一讀取 spatialIntersections.centers[]。
- 中心點名稱優先從 center.centerFeature 取值，依序使用：
  1. center.centerFeature["測點名稱"]
  2. center.centerFeature["監測站名稱"]
  3. center.centerFeature["事業名稱"]
  4. center.centerFeature["中文名"]
  5. center.centerFeature["樹種"]
  6. center.centerFeature["河流"]
  7. center.centerFeature["行政區"]
  若以上都沒有，才使用 center.centerName。
- 中心點所屬資料集名稱請使用 spatialIntersections.centerComponent.componentName。
- 目標資料集名稱請使用 matchGroup.componentName。
- 該中心點與該目標資料集的交集總數請使用 matchGroup.totalMatches。
- 最近距離請使用 matchGroup.nearestDistanceMeters。
- matches 只是最近樣本，不代表全部；總數必須使用 totalMatches。

輸出格式必須固定為以下兩段：

## 1. 交叉結果問題分析
請用條列式列出每一個有交集的中心點。每一列必須直接套用下列句型：
- {centerComponent} 中的「{centerPointName}」與 {targetComponent} 有交集，1 公里內共有 {totalMatches} 筆交集資料，最近距離約 {nearestDistanceMeters} 公尺。判讀：{根據 centerFeature 與 targetComponent 寫 1 句具體判讀}

如果同一個中心點同時和多個 targetComponent 有交集，請分開列出，不要合併成一句。
如果 spatialIntersections.centers 是空陣列，請只寫：「目前沒有 1 公里內的空間交集。」

## 2. 建議解法
請根據第 1 段的交集結果，提出 2 到 4 點可能問題與解決方法。每點使用以下格式：
- 可能問題：
- 建議作法：
- 優先觀察指標：

判讀限制：
- 空間接近只能視為風險線索，不等於因果。
- 不要說「噪音會影響植物生長」這類不合理判斷。
- 噪音和鳥類、動物可描述為可能干擾棲息或活動。
- 水質異常和動植物可描述為可能反映水域棲地壓力。
- 焚化廠或空污資料和動植物可描述為可能需要觀察空氣品質與周邊生態狀態。
- 沒有資料支持時，請明確說不能判斷。
- 請用繁體中文回答。`;
}

function buildUserPrompt(records) {
	const spatialIntersections = buildProximityIntersections(records);
	logSpatialIntersections(spatialIntersections);
	mapStore.showAIProximityRadius(
		spatialIntersections.centers,
		spatialIntersections.radiusMeters,
	);
	mapStore.clearAIMatchedComponentHighlight();
	mapStore.showAIMatchedFeatures(spatialIntersections.centers);
	const payload = {
		dashboard: {
			name: contentStore.currentDashboard.name,
			index: contentStore.currentDashboard.index,
			city: contentStore.currentDashboard.city,
		},
		activeMapLayers: activeLayerTitles.value,
		visibleComponents: getComponentSummary(),
		filteredMapData: compactIndexedDBRecords(records),
		spatialIntersections,
	};

	return `以下是雙北環境資料，包含資料摘要與前端先用 1 公里半徑算出的鄰近交集。請優先引用 ｃ，再搭配資料摘要判斷值得注意的環境風險訊號。${JSON.stringify(payload, null, 2)}`;
}

async function generateAnalysis() {
	if (!canGenerate.value) return;
	isLoading.value = true;
	analysis.value = "";

	try {
		const records = await readAgentData();
		if (records.length === 0) {
			analysis.value = "請先在地圖交叉比較中開啟或篩選資料，再生成分析。";
			return;
		}

		const response = await http.post("/ai/chat/twai", {
			session: sessionId.value,
			stream: false,
			messages: [
				{ role: "system", content: buildSystemPrompt() },
				{ role: "user", content: buildUserPrompt(records) },
			],
			max_new_tokens: 1200,
			temperature: 0.2,
		});

		sessionId.value = response.data?.data?.session || sessionId.value;
		analysis.value =
			response.data?.data?.content || "目前沒有取得可顯示的分析結果。";
		hasDataChanged.value = false;
		statusText.value = "分析已生成，等待資料更新";
	} catch (error) {
		analysis.value = "目前無法連線到 LLM 分析服務，請稍後再試。";
		console.error("Map analysis agent error:", error);
	} finally {
		isLoading.value = false;
	}
}

async function refreshDataStatus() {
	try {
		await readAgentData();
	} catch {
		statusText.value = "目前無法讀取交叉比較資料";
	}
}

function togglePanel() {
	isOpen.value = !isOpen.value;
	if (isOpen.value) {
		refreshDataStatus();
	}
}

function startDrag(event) {
	if (event.target.closest("button")) return;
	isDragging.value = true;
	dragStart.value = {
		x: event.clientX,
		y: event.clientY,
		right: position.value.right,
		bottom: position.value.bottom,
	};
	window.addEventListener("mousemove", drag);
	window.addEventListener("mouseup", stopDrag);
}

function drag(event) {
	if (!isDragging.value) return;
	position.value = {
		right: Math.max(
			8,
			dragStart.value.right - (event.clientX - dragStart.value.x),
		),
		bottom: Math.max(
			8,
			dragStart.value.bottom - (event.clientY - dragStart.value.y),
		),
	};
}

function stopDrag() {
	isDragging.value = false;
	window.removeEventListener("mousemove", drag);
	window.removeEventListener("mouseup", stopDrag);
}

onMounted(async () => {
	await mapStore.clearIndexedDB();
	mapStore.clearAIProximityRadius();
	mapStore.clearAIMatchedComponentHighlight();
	mapStore.clearAIMatchedFeatures();
	dataSignature.value = "";
	hasDataChanged.value = false;
	analysis.value = "";
	refreshDataStatus();
	pollTimer = setInterval(refreshDataStatus, 4000);
});

onBeforeUnmount(() => {
	clearInterval(pollTimer);
	mapStore.clearAIProximityRadius();
	mapStore.clearAIMatchedComponentHighlight();
	mapStore.clearAIMatchedFeatures();
	stopDrag();
});
</script>

<template>
	<div
		class="map-analysis-agent"
		:style="{
			right: `${position.right}px`,
			bottom: `${position.bottom}px`,
		}"
	>
		<section v-if="isOpen" class="map-analysis-agent__panel">
			<header class="map-analysis-agent__header" @mousedown="startDrag">
				<div>
					<h3>AI 交叉分析</h3>
					<p>{{ statusText }}</p>
				</div>
				<button type="button" title="收合" @click="togglePanel">
					<span>keyboard_arrow_down</span>
				</button>
			</header>

			<div class="map-analysis-agent__body">
				<button
					type="button"
					class="map-analysis-agent__primary"
					:disabled="!canGenerate"
					@click="generateAnalysis"
				>
					{{ generateButtonText }}
				</button>
				<article v-if="analysis" class="map-analysis-agent__result">
					{{ analysis }}
				</article>
				<p v-else class="map-analysis-agent__hint">
					開啟或篩選地圖交叉比較資料後，按鈕會亮起來讓你生成目前分析。
				</p>
			</div>
		</section>

		<button
			v-else
			type="button"
			class="map-analysis-agent__mini"
			title="AI 交叉分析"
			@click="togglePanel"
		>
			<span>insights</span>
			<em>AI 交叉分析</em>
		</button>
	</div>
</template>

<style scoped lang="scss">
.map-analysis-agent {
	position: fixed;
	z-index: 30;
	&__mini {
		width: 168px;
		height: 64px;
		display: flex;
		align-items: center;
		gap: 0.5rem;
		justify-content: center;
		border-radius: 20px;
		background: var(--color-highlight);
		color: var(--color-complement-text);
		border: 2px solid var(--color-complement-text);
		box-shadow: 0 8px 24px rgb(0 0 0 / 35%);
		position: relative;
		pointer-events: auto;

		span {
			font-family: var(--font-icon);
			font-size: 1.7rem;
		}

		em {
			font-style: normal;
			font-size: 1rem;
			font-weight: 700;
			line-height: 1;
		}
	}

	&__panel {
		width: 330px;
		max-width: calc(100vw - 32px);
		height: 430px;
		max-height: calc(100vh - 120px);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		border: 1px solid var(--color-border);
		border-radius: 8px;
		background: var(--color-component-background);
		box-shadow: 0 14px 42px rgb(0 0 0 / 42%);
	}

	&__header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.75rem;
		padding: 0.75rem;
		border-bottom: 1px solid var(--color-border);
		cursor: move;
		user-select: none;

		h3,
		p {
			margin: 0;
		}

		h3 {
			color: #ffffff;
			font-size: 1rem;
			line-height: 1.2;
		}

		p {
			color: #c7c7c7;
			margin-top: 0.25rem;
			font-size: 0.78rem;
		}

		button {
			width: 32px;
			height: 32px;
			display: flex;
			align-items: center;
			justify-content: center;
			border-radius: 50%;
			color: var(--color-complement-text);

			span {
				font-family: var(--font-icon);
				font-size: 1.4rem;
			}
		}
	}

	&__body {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 0.65rem;
		padding: 0.75rem;
		overflow: hidden;
	}

	&__primary {
		height: 36px;
		flex-shrink: 0;
		border-radius: 6px;
		background: var(--color-highlight);
		color: #ffffff;
		font-weight: 700;

		&:disabled {
			cursor: not-allowed;
			filter: grayscale(1);
			opacity: 0.45;
		}
	}

	&__result,
	&__hint {
		margin: 0;
		font-size: 0.88rem;
		line-height: 1.55;
		white-space: pre-line;
	}

	&__result {
		flex: 1;
		color: #ffffff;
		overflow-y: auto;
		padding-right: 0.25rem;
	}

	&__hint {
		color: #c7c7c7;
	}
}
</style>
