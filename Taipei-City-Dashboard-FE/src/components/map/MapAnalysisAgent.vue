<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from "vue";
import http from "../../router/axios";
import { useContentStore } from "../../store/contentStore";
import { useMapStore } from "../../store/mapStore";

const contentStore = useContentStore();
const mapStore = useMapStore();

const isOpen = ref(false);
const isDragging = ref(false);
const isResizing = ref(false);
const didDrag = ref(false);
const isLoading = ref(false);
const hasDataChanged = ref(false);
const analysis = ref("");
const statusText = ref("尚未讀取交叉比較資料");
const sessionId = ref(`map-analysis-${Date.now()}`);
const position = ref({ right: 104, bottom: 24 });
const dragStart = ref({ x: 0, y: 0, right: 104, bottom: 24 });
const panelSize = ref({ width: 330, height: 430 });
const resizeStart = ref({
	x: 0,
	y: 0,
	width: 330,
	height: 430,
	right: 104,
	bottom: 24,
	direction: "",
});
const agentRef = ref(null);
const dataSignature = ref("");
const dataGroupCount = ref(0);
let pollTimer = null;
const PROXIMITY_RADIUS_METERS = 1000;
const MAX_CENTER_FEATURES = 20;
const MAX_MATCHES_PER_COMPONENT = 5;
const VIEWPORT_MARGIN = 8;
const MIN_PANEL_WIDTH = 300;
const MIN_PANEL_HEIGHT = 300;

const canGenerate = computed(
	() => dataGroupCount.value >= 2 && hasDataChanged.value && !isLoading.value,
);
const generateButtonText = computed(() => {
	if (isLoading.value) return "分析中...";
	if (dataGroupCount.value < 2 || !hasDataChanged.value)
		return "等待資料更新";
	return analysis.value ? "重新生成分析" : "生成目前分析";
});
const renderedAnalysis = computed(() => renderBoldText(analysis.value));

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

function escapeHtml(value) {
	return String(value)
		.replace(/&/g, "&amp;")
		.replace(/</g, "&lt;")
		.replace(/>/g, "&gt;")
		.replace(/"/g, "&quot;")
		.replace(/'/g, "&#039;");
}

function renderBoldText(value) {
	if (!value) return "";
	return escapeHtml(value)
		.replace(/\*\*([^*]+)\*\*/g, "<strong>$1</strong>")
		.replace(/\n/g, "<br>");
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
- matchGroup.matches 是該 targetComponent 中距離最近的樣本細項，可用於具體判讀或提出可能問題。
- 判讀時請優先查看 matchGroup.matches[] 的 matches[].featureName 與 matches[].feature，將其整理成類型或趨勢，例如「多筆鳥類紀錄」、「水域生物紀錄」、「特定樹種分布」、「鄰近監測點」等。
- 不要在判讀句中逐字重複列出 matches[].featureName 清單；featureName 只作為判讀依據，不作為獨立列舉內容。
- matches 只是最近樣本，不代表全部；總數必須使用 totalMatches，不要用 matches.length 當總數。

		輸出格式必須固定為以下兩段。不要使用 markdown 標題、清單、編號清單、表格或 inline code；唯一允許的 markdown 是 **粗體字**。

		**交叉結果問題分析**
		請逐一輸出 spatialIntersections.centers[] 裡的每一筆 center；只要該 center 出現在 centers[]，就必須產生一個中心點分組，不要因為中心點顯示名稱相同或 targetComponent 相同而省略任何 center。
		中心點顯示名稱必須從該筆 center 的 center.centerFeature 裡取值，並依照上方「中心點名稱優先從 center.centerFeature 取值」的順序決定，例如 center.centerFeature["測點名稱"]、center.centerFeature["監測站名稱"]、center.centerFeature["事業名稱"] 等。不要把 centerPointName 當成實際資料欄位。
		每一筆 center 的分組標題必須直接套用下列格式（用列點方式呈現）：
		1. **{centerComponent} -- {從該筆 center.centerFeature 取出的中心點顯示名稱}：**

		在每一筆 center 的標題底下，請整理該筆 center.matchesByComponent 裡所有 targetComponent 的交集結果。每個 targetComponent 直接用獨立段落呈現，不要使用清單符號或編號。
		每一段必須直接套用下列句型：
		與 {targetComponent} 有交集，1 公里內共有 {totalMatches} 筆交集資料，最近距離約 {nearestDistanceMeters} 公尺。判讀：{根據 centerFeature、targetComponent 與 matchGroup.matches 的 featureName/feature 整理出 1 句具體判讀或可能問題。}

		如果同一筆 center 底下有多個 targetComponent，請全部放在該筆 center 的標題底下，每個 targetComponent 各自成為一個獨立段落，不要合併成一句。
		不要跨 center 合併資料；即使兩筆 center 有相同的中心點顯示名稱，也要分開輸出兩個中心點分組。只有同一筆 center.matchesByComponent 內的資料可以整理在同一個中心點標題底下。
		如果 spatialIntersections.centers 是空陣列，請只寫：「目前沒有 1 公里內的空間交集。」

	**建議解法**
	請根據交集結果，提出可能問題與解決方法。不要使用清單符號或編號。每一個建議用空行分隔，格式如下：
	**可能問題：**...
	**建議作法：**...
	**優先觀察指標：**...

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
		nextTick(keepAgentInViewport);
	}
}

function handleMiniClick() {
	if (didDrag.value) {
		didDrag.value = false;
		return;
	}
	togglePanel();
}

function startDrag(event) {
	const button = event.target.closest("button");
	if (button && button !== event.currentTarget) return;
	if (event.button !== undefined && event.button !== 0) return;
	event.preventDefault();
	isDragging.value = true;
	didDrag.value = false;
	dragStart.value = {
		x: event.clientX,
		y: event.clientY,
		right: position.value.right,
		bottom: position.value.bottom,
	};
	window.addEventListener("pointermove", drag);
	window.addEventListener("pointerup", stopDrag);
}

function clampSize(width, height) {
	return {
		width: Math.min(
			Math.max(MIN_PANEL_WIDTH, width),
			window.innerWidth - VIEWPORT_MARGIN * 2,
		),
		height: Math.min(
			Math.max(MIN_PANEL_HEIGHT, height),
			window.innerHeight - VIEWPORT_MARGIN * 2,
		),
	};
}

function clampPosition(right, bottom, width, height) {
	return {
		right: Math.min(
			Math.max(VIEWPORT_MARGIN, right),
			Math.max(
				VIEWPORT_MARGIN,
				window.innerWidth - width - VIEWPORT_MARGIN,
			),
		),
		bottom: Math.min(
			Math.max(VIEWPORT_MARGIN, bottom),
			Math.max(
				VIEWPORT_MARGIN,
				window.innerHeight - height - VIEWPORT_MARGIN,
			),
		),
	};
}

function keepAgentInViewport() {
	const rect = agentRef.value?.getBoundingClientRect();
	if (!rect) return;
	const size = isOpen.value
		? clampSize(panelSize.value.width, panelSize.value.height)
		: { width: rect.width, height: rect.height };
	if (isOpen.value) {
		panelSize.value = size;
	}
	position.value = clampPosition(
		position.value.right,
		position.value.bottom,
		size.width,
		size.height,
	);
}

function drag(event) {
	if (!isDragging.value) return;
	const deltaX = event.clientX - dragStart.value.x;
	const deltaY = event.clientY - dragStart.value.y;
	if (Math.abs(deltaX) > 4 || Math.abs(deltaY) > 4) {
		didDrag.value = true;
	}
	const rect = agentRef.value?.getBoundingClientRect();
	const size = rect
		? { width: rect.width, height: rect.height }
		: { width: panelSize.value.width, height: panelSize.value.height };

	position.value = clampPosition(
		dragStart.value.right - deltaX,
		dragStart.value.bottom - deltaY,
		size.width,
		size.height,
	);
}

function stopDrag() {
	isDragging.value = false;
	window.removeEventListener("pointermove", drag);
	window.removeEventListener("pointerup", stopDrag);
	window.setTimeout(() => {
		didDrag.value = false;
	}, 0);
}

function startResize(event, direction) {
	if (event.button !== undefined && event.button !== 0) return;
	event.preventDefault();
	event.stopPropagation();
	isResizing.value = true;
	resizeStart.value = {
		x: event.clientX,
		y: event.clientY,
		width: panelSize.value.width,
		height: panelSize.value.height,
		right: position.value.right,
		bottom: position.value.bottom,
		direction,
	};
	window.addEventListener("pointermove", resize);
	window.addEventListener("pointerup", stopResize);
}

function resize(event) {
	if (!isResizing.value) return;
	const deltaX = event.clientX - resizeStart.value.x;
	const deltaY = event.clientY - resizeStart.value.y;
	const direction = resizeStart.value.direction;
	let nextWidth = resizeStart.value.width;
	let nextHeight = resizeStart.value.height;
	let nextRight = resizeStart.value.right;
	let nextBottom = resizeStart.value.bottom;

	if (direction.includes("e")) {
		nextWidth = resizeStart.value.width + deltaX;
		nextRight = resizeStart.value.right - deltaX;
	}
	if (direction.includes("w")) {
		nextWidth = resizeStart.value.width - deltaX;
	}
	if (direction.includes("s")) {
		nextHeight = resizeStart.value.height + deltaY;
		nextBottom = resizeStart.value.bottom - deltaY;
	}
	if (direction.includes("n")) {
		nextHeight = resizeStart.value.height - deltaY;
	}

	const size = clampSize(nextWidth, nextHeight);
	panelSize.value = size;
	position.value = clampPosition(
		nextRight,
		nextBottom,
		size.width,
		size.height,
	);
}

function stopResize() {
	isResizing.value = false;
	window.removeEventListener("pointermove", resize);
	window.removeEventListener("pointerup", stopResize);
	keepAgentInViewport();
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
	window.addEventListener("resize", keepAgentInViewport);
});

onBeforeUnmount(() => {
	clearInterval(pollTimer);
	mapStore.clearAIProximityRadius();
	mapStore.clearAIMatchedComponentHighlight();
	mapStore.clearAIMatchedFeatures();
	stopDrag();
	stopResize();
	window.removeEventListener("resize", keepAgentInViewport);
});
</script>

<template>
	<div
		ref="agentRef"
		class="map-analysis-agent"
		:style="{
			right: `${position.right}px`,
			bottom: `${position.bottom}px`,
		}"
	>
		<section
			v-if="isOpen"
			class="map-analysis-agent__panel"
			:style="{
				width: `${panelSize.width}px`,
				height: `${panelSize.height}px`,
			}"
		>
			<span
				v-for="direction in [
					'n',
					'e',
					's',
					'w',
					'ne',
					'nw',
					'se',
					'sw',
				]"
				:key="direction"
				:class="[
					'map-analysis-agent__resize-handle',
					`map-analysis-agent__resize-handle--${direction}`,
				]"
				@pointerdown="startResize($event, direction)"
			/>
			<header class="map-analysis-agent__header" @pointerdown="startDrag">
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
				<article
					v-if="analysis"
					class="map-analysis-agent__result"
					v-html="renderedAnalysis"
				/>
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
			@pointerdown="startDrag"
			@click="handleMiniClick"
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
			font-size: 2.04rem;
		}

		em {
			font-style: normal;
			font-size: 1.2rem;
			font-weight: 700;
			line-height: 1;
		}
	}

	&__panel {
		max-width: calc(100vw - 16px);
		max-height: calc(100vh - 16px);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		border: 1px solid var(--color-border);
		border-radius: 8px;
		background: var(--color-component-background);
		box-shadow: 0 14px 42px rgb(0 0 0 / 42%);
		position: relative;
	}

	&__resize-handle {
		position: absolute;
		z-index: 2;
		background: transparent;
		touch-action: none;
	}

	&__resize-handle--n,
	&__resize-handle--s {
		left: 12px;
		width: calc(100% - 24px);
		height: 10px;
		cursor: ns-resize;
	}

	&__resize-handle--n {
		top: 0;
	}

	&__resize-handle--s {
		bottom: 0;
	}

	&__resize-handle--e,
	&__resize-handle--w {
		top: 12px;
		width: 10px;
		height: calc(100% - 24px);
		cursor: ew-resize;
	}

	&__resize-handle--e {
		right: 0;
	}

	&__resize-handle--w {
		left: 0;
	}

	&__resize-handle--ne,
	&__resize-handle--nw,
	&__resize-handle--se,
	&__resize-handle--sw {
		width: 18px;
		height: 18px;
	}

	&__resize-handle--ne {
		top: 0;
		right: 0;
		cursor: nesw-resize;
	}

	&__resize-handle--nw {
		top: 0;
		left: 0;
		cursor: nwse-resize;
	}

	&__resize-handle--se {
		right: 0;
		bottom: 0;
		cursor: nwse-resize;
	}

	&__resize-handle--sw {
		bottom: 0;
		left: 0;
		cursor: nesw-resize;
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
			font-size: 1.2rem;
			line-height: 1.2;
		}

		p {
			color: #c7c7c7;
			margin-top: 0.25rem;
			font-size: 0.936rem;
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
				font-size: 1.68rem;
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
		font-size: 1.2rem;
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
		font-size: 1.056rem;
		line-height: 1.55;
		white-space: pre-line;
	}

	&__result {
		flex: 1;
		color: #ffffff;
		font-size: 1.267rem;
		line-height: 1.55;
		overflow-y: auto;
		padding-right: 0.25rem;
		white-space: normal;

		:deep(strong) {
			font-size: inherit;
			font-weight: 800;
		}
	}

	&__hint {
		color: #c7c7c7;
	}
}
</style>
