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
let pollTimer = null;

const canGenerate = computed(() => hasDataChanged.value && !isLoading.value);
const generateButtonText = computed(() => {
	if (isLoading.value) return "分析中...";
	if (!hasDataChanged.value) return "等待資料更新";
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
		.map((record) => `${record.componentKey}:${record.timestamp}:${record.featuresCount}`)
		.sort()
		.join("|");
}

function compactFeature(feature) {
	const entries = Object.entries(feature || {})
		.filter(([, value]) => ["string", "number", "boolean"].includes(typeof value))
		.slice(0, 12);
	return Object.fromEntries(entries);
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

async function readAgentData() {
	const result = await mapStore.getFilteredFeaturesForAgent();
	if (!result.success || !result.data?.length) {
		hasDataChanged.value = false;
		statusText.value = "目前沒有交叉比較資料";
		return [];
	}

	const nextSignature = getDataSignature(result.data);
	if (nextSignature !== dataSignature.value) {
		hasDataChanged.value = true;
	}

	dataSignature.value = nextSignature;
	statusText.value = `已讀取 ${result.data.length} 組資料`;
	return result.data;
}

function buildSystemPrompt() {
	return `你是一位擅長把城市開放資料轉成一般人看得懂結論的資料分析助理。

你的回答會直接顯示給沒有工程或資料分析背景的一般使用者，因此請避免解釋資料格式、欄位結構、GeoJSON、Feature、properties、geometry、API、JSON 等技術細節。不要逐一說明欄位名稱，除非該名稱本身對理解結果很重要；若必須提到，請用白話解釋它代表什麼。

回答規則：
1. 使用繁體中文。
2. 語氣自然、清楚，像是在向市民或非技術決策者說明。
3. 不要輸出程式碼、JSON、欄位定義或資料表結構。
4. 如果資料只足以做初步判斷，請明確說「目前只能做初步觀察，還不能代表因果關係」。
5. 不要過度推論；沒有資料支持的地方請說明需要更多資料驗證。
6. 請把重點放在：目前顯示的地圖資料可能呈現什麼現象、哪些區域值得關注、對城市治理或環境改善有什麼意義。

請用以下格式回答：
## 一起看數據分析問題
針對目前儀表板中的經緯度資料，計算目前組件地理位置比較重疊或有大量資料交集的區域，並列點說明這些地方可能發生的現象及需要注意的問題。
例如：如果有一個區域同時有很多「娛樂噪音點」和「兩棲類生態出沒」，可以針對其點出可能會造成的問題。
請用1. 2. 3. ...的方式列點說明，並且每一點都要說明「為什麼這件事重要」。

## 解決方法
對應上述分析出的問題中的每一點來提出解決方法，例如：針對「娛樂噪音點」和「兩棲類生態出沒」重疊的區域，可能需要限制夜間娛樂活動等。
針對每一個問題提出可以讓環境更好、城市發展更永續的建議。
請用1. 2. 3. ...的方式列點說明，並且每一點都要說明「為什麼這個建議可以解決問題」。`;
}

function buildUserPrompt(records) {
	const payload = {
		dashboard: {
			name: contentStore.currentDashboard.name,
			index: contentStore.currentDashboard.index,
			city: contentStore.currentDashboard.city,
		},
		activeMapLayers: activeLayerTitles.value,
		visibleComponents: getComponentSummary(),
		filteredMapData: compactIndexedDBRecords(records),
	};

	return `以下是使用者目前在地圖交叉比較介面中開啟、篩選後的資料摘要：

${JSON.stringify(payload, null, 2)}

使用者想知道：
這些目前顯示在地圖上的資料，在空間分布上是否可能有關聯？哪些地方或現象值得注意？請產出一份可以直接給一般使用者閱讀的分析結果。`;
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
		right: Math.max(8, dragStart.value.right - (event.clientX - dragStart.value.x)),
		bottom: Math.max(8, dragStart.value.bottom - (event.clientY - dragStart.value.y)),
	};
}

function stopDrag() {
	isDragging.value = false;
	window.removeEventListener("mousemove", drag);
	window.removeEventListener("mouseup", stopDrag);
}

onMounted(() => {
	refreshDataStatus();
	pollTimer = setInterval(refreshDataStatus, 4000);
});

onBeforeUnmount(() => {
	clearInterval(pollTimer);
	stopDrag();
});
</script>

<template>
  <div
    class="map-analysis-agent"
    :style="{ right: `${position.right}px`, bottom: `${position.bottom}px` }"
  >
    <section
      v-if="isOpen"
      class="map-analysis-agent__panel"
    >
      <header
        class="map-analysis-agent__header"
        @mousedown="startDrag"
      >
        <div>
          <h3>AI 交叉分析</h3>
          <p>{{ statusText }}</p>
        </div>
        <button
          type="button"
          title="收合"
          @click="togglePanel"
        >
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
        >
          {{ analysis }}
        </article>
        <p
          v-else
          class="map-analysis-agent__hint"
        >
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
      <strong>AI</strong>
      <em>交叉分析</em>
    </button>
  </div>
</template>

<style scoped lang="scss">
.map-analysis-agent {
	position: fixed;
	z-index: 30;

	&__mini {
		width: 108px;
		height: 42px;
		display: flex;
		align-items: center;
		gap: 0.3rem;
		justify-content: center;
		border-radius: 999px;
		background: var(--color-highlight);
		color: var(--color-complement-text);
		border: 2px solid var(--color-complement-text);
		box-shadow: 0 8px 24px rgb(0 0 0 / 35%);
		position: relative;
		pointer-events: auto;

		span {
			font-family: var(--font-icon);
			font-size: 1.35rem;
		}

		strong {
			min-width: 22px;
			height: 22px;
			display: flex;
			align-items: center;
			justify-content: center;
			border-radius: 50%;
			background: var(--color-complement-text);
			color: var(--color-component-background);
			font-size: 0.65rem;
			font-weight: 800;
			line-height: 1;
		}

		em {
			font-style: normal;
			font-size: 0.78rem;
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
			color: var(--color-complement-text);
		}

		h3 {
			font-size: 1rem;
			line-height: 1.2;
		}

		p {
			margin-top: 0.25rem;
			font-size: 0.78rem;
			opacity: 0.7;
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
		color: var(--color-complement-text);
		font-size: 0.88rem;
		line-height: 1.55;
		white-space: pre-line;
	}

	&__result {
		flex: 1;
		overflow-y: auto;
		padding-right: 0.25rem;
	}

	&__hint {
		opacity: 0.72;
	}
}
</style>
