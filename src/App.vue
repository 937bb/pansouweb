<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, computed } from "vue";
import { search, checkAuthStatus, logout, type SearchParams } from "@/api";
import type { SearchResponse, MergedResults } from "@/types";
import SearchForm from "@/components/SearchForm.vue";
import ResultTabs from "@/components/ResultTabs.vue";
import SearchStats from "@/components/SearchStats.vue";
import ApiStatus from "@/components/ApiStatus.vue";
import ApiDocs from "@/components/ApiDocs.vue";
import LoginDialog from "@/components/LoginDialog.vue";

// 搜索状态
const loading = ref(false);
const searchResults = reactive<{
	total: number;
	mergedResults: MergedResults;
}>({
	total: 0,
	mergedResults: {},
});

// 搜索时间
const searchTime = ref<number | undefined>(undefined);

// 后台更新状态
const isUpdating = ref(false);
const updateCount = ref(0);
const updateTimer = ref<number | null>(null);
const secondSearchTimeout = ref<number | null>(null);
const thirdSearchTimeout = ref<number | null>(null);
const lastSearchParams = ref<SearchParams | null>(null);

// 是否已经执行过搜索
const hasSearched = ref(false);
// 是否正在进行后台搜索（包括初始搜索和后续更新）
const isActivelySearching = ref(false);

// 当前页面状态
const currentPage = ref<"search" | "status" | "docs">("search");

// 登录状态
const showLogin = ref(false);
const isAuthenticated = ref(false);
const currentUsername = ref("");

// 页面切换
const switchToStatus = () => {
	currentPage.value = "status";
};

const switchToDocs = () => {
	currentPage.value = "docs";
};

// 处理搜索
const handleSearch = async (params: SearchParams) => {
	// 停止之前的更新
	stopUpdate();
	keyword.value = params.kw;
	// 标记已执行搜索和正在搜索
	hasSearched.value = true;
	isActivelySearching.value = true;

	// 重置状态
	loading.value = true;

	// 清空之前的搜索结果
	searchResults.total = 0;
	searchResults.mergedResults = {};
	searchTime.value = undefined;

	// 保存搜索参数
	lastSearchParams.value = { ...params };

	const startTime = Date.now();

	try {
		// 创建TG源搜索参数
		const tgParams: SearchParams = {
			...params,
			src: "tg",
		};

		// 创建ALL源搜索参数
		const allParams: SearchParams = {
			...params,
			src: "all",
		};

		// 先发起TG源搜索请求
		search(tgParams)
			.then((tgResponse) => {
				if (tgResponse && tgResponse.total !== undefined) {
					// 使用TG的搜索结果进行显示
					updateSearchResults(tgResponse);
					searchTime.value = Date.now() - startTime;
					// TG搜索完成后，关闭加载状态
					loading.value = false;

					// TG搜索完成后，再发起第一次ALL源搜索
					search(allParams)
						.then((allResponse) => {
							// 记录第一次ALL搜索完成时间
							const firstAllSearchCompleteTime = Date.now();

							// 如果ALL源结果比当前结果更多，则更新显示
							if (allResponse && allResponse.total >= searchResults.total) {
								updateSearchResults(allResponse);
							}

							// 开始第二次ALL源搜索
							startSecondAllSearch(firstAllSearchCompleteTime);
						})
						.catch((error) => {
							console.error("第一次ALL搜索出错:", error);

							// 即使第一次ALL搜索失败，也继续进行第二次搜索
							startSecondAllSearch(Date.now());
						});
				} else {
					console.error("TG搜索结果格式不正确:", tgResponse);
					loading.value = false;

					// 即使TG搜索失败，也尝试ALL源搜索
					search(allParams)
						.then((allResponse) => {
							if (allResponse && allResponse.total !== undefined) {
								updateSearchResults(allResponse);
								const firstAllSearchCompleteTime = Date.now();
								startSecondAllSearch(firstAllSearchCompleteTime);
							}
						})
						.catch((error) => {
							console.error("第一次ALL搜索出错:", error);
							isActivelySearching.value = false;
						});
				}
			})
			.catch((error) => {
				console.error("TG搜索出错:", error);
				loading.value = false;

				// TG搜索出错时，尝试ALL源搜索
				search(allParams)
					.then((allResponse) => {
						if (allResponse && allResponse.total !== undefined) {
							updateSearchResults(allResponse);
							const firstAllSearchCompleteTime = Date.now();
							startSecondAllSearch(firstAllSearchCompleteTime);
						}
					})
					.catch((error) => {
						console.error("第一次ALL搜索出错:", error);
						isActivelySearching.value = false;
					});
			});

		// 设置一个超时，确保即使搜索很慢，UI也不会一直处于加载状态
		setTimeout(() => {
			if (loading.value) {
				loading.value = false;
			}
		}, 5000); // 5秒后如果还在加载，则关闭加载状态
	} catch (error) {
		console.error("搜索初始化出错:", error);
		loading.value = false;
		isActivelySearching.value = false;
	}
};

// 搜索完成处理
const handleSearchComplete = () => {
	// 只处理UI相关的状态，不影响搜索流程
};

// 更新搜索结果
const diskSortOrder = ["aliyun", "baidu", "quark", "115", "123", "xunlei", "mobile", "tianyi", "uc", "pikpak", "ed2k", "magnet", "other"];

const updateSearchResults = (response: SearchResponse) => {
	if (!response) return;

	searchResults.total = response.total || 0;

	if (!response.merged_by_type) {
		console.warn("搜索结果中没有 merged_by_type 字段");
		searchResults.mergedResults = {};
		return;
	}

	const merged = response.merged_by_type;

	// 临时给整数键加前缀
	const normalizeKey = (key: string) => (/^\d+$/.test(key) ? `disk_${key}` : key);
	const denormalizeKey = (key: string) => key.replace(/^disk_/, "");

	// 1️⃣ 转数组并加前缀
	const entries: [string, any][] = Object.entries(merged).map(([k, v]) => [normalizeKey(k), v]);

	// 2️⃣ 按自定义顺序排序
	entries.sort(([aKey], [bKey]) => {
		const aIdx = diskSortOrder.indexOf(denormalizeKey(aKey));
		const bIdx = diskSortOrder.indexOf(denormalizeKey(bKey));
		const aPos = aIdx === -1 ? 999 : aIdx;
		const bPos = bIdx === -1 ? 999 : bIdx;
		return aPos - bPos;
	});

	// 3️⃣ 转回对象，并保留前缀顺序
	const sortedMerged = entries.reduce((acc, [k, v]) => {
		acc[k] = v;
		return acc;
	}, {} as Record<string, any>);
	// 4️⃣ 赋值给响应式对象
	searchResults.mergedResults = sortedMerged;
};

// 开始第二次ALL源搜索
const startSecondAllSearch = (firstAllSearchCompleteTime: number) => {
	if (!lastSearchParams.value) return;

	isUpdating.value = true;
	isActivelySearching.value = true;
	updateCount.value = 1;

	// 创建ALL源搜索参数
	const allParams: SearchParams = {
		...lastSearchParams.value,
		src: "all",
	};

	// 计算需要等待的时间，确保与第一次ALL搜索至少间隔2秒
	const currentTime = Date.now();
	const timeElapsedSinceFirstAllSearch = currentTime - firstAllSearchCompleteTime;
	const delayForSecondSearch = Math.max(0, 2000 - timeElapsedSinceFirstAllSearch);

	// 执行第二次ALL搜索
	const executeSecondAllSearch = async () => {
		if (!lastSearchParams.value) {
			stopUpdate();
			return;
		}

		try {
			const secondAllSearchStartTime = Date.now();
			const response = await search(allParams);

			// 更新结果
			if (response && response.total >= searchResults.total) {
				updateSearchResults(response);
			}

			// 记录第二次ALL搜索完成时间
			const secondAllSearchCompleteTime = Date.now();

			// 开始第三次ALL源搜索
			startThirdAllSearch(secondAllSearchCompleteTime);
		} catch (error) {
			console.error("第二次ALL搜索出错:", error);
			stopUpdate();
		}
	};

	// 设置定时器，在适当的时间执行第二次ALL搜索
	secondSearchTimeout.value = window.setTimeout(executeSecondAllSearch, delayForSecondSearch);
};

// 开始第三次ALL源搜索
const startThirdAllSearch = (secondAllSearchCompleteTime: number) => {
	if (!lastSearchParams.value) return;

	updateCount.value = 2;

	// 创建ALL源搜索参数
	const allParams: SearchParams = {
		...lastSearchParams.value,
		src: "all",
	};

	// 计算需要等待的时间，确保与第二次ALL搜索至少间隔3秒
	const currentTime = Date.now();
	const timeElapsedSinceSecondAllSearch = currentTime - secondAllSearchCompleteTime;
	const delayForThirdSearch = Math.max(0, 3000 - timeElapsedSinceSecondAllSearch);

	// 执行第三次ALL搜索
	const executeThirdAllSearch = async () => {
		if (!lastSearchParams.value) {
			stopUpdate();
			return;
		}

		try {
			const response = await search(allParams);

			// 更新结果
			if (response && response.total >= searchResults.total) {
				updateSearchResults(response);
			}
		} catch (error) {
			console.error("第三次ALL搜索出错:", error);
		} finally {
			// 完成所有搜索，停止更新
			stopUpdate();
		}
	};

	// 设置定时器，在适当的时间执行第三次ALL搜索
	thirdSearchTimeout.value = window.setTimeout(executeThirdAllSearch, delayForThirdSearch);
};

// 停止后台更新
const stopUpdate = () => {
	// 清除所有定时器
	if (updateTimer.value) {
		clearInterval(updateTimer.value);
		updateTimer.value = null;
	}

	if (secondSearchTimeout.value) {
		clearTimeout(secondSearchTimeout.value);
		secondSearchTimeout.value = null;
	}

	if (thirdSearchTimeout.value) {
		clearTimeout(thirdSearchTimeout.value);
		thirdSearchTimeout.value = null;
	}

	// 标记搜索已结束
	isUpdating.value = false;
	isActivelySearching.value = false;
};

// 重置到初始页面
const resetToInitial = () => {
	// 停止之前的更新
	stopUpdate();

	// 切换到搜索页面
	currentPage.value = "search";

	// 重置所有状态
	hasSearched.value = false;
	isActivelySearching.value = false;
	loading.value = false;
	searchResults.total = 0;
	searchResults.mergedResults = {};
	searchTime.value = undefined;
	isUpdating.value = false;
	updateCount.value = 0;
};

// 检查认证状态
const checkAuth = async () => {
	const status = await checkAuthStatus();
	if (status.enabled && !status.authenticated) {
		showLogin.value = true;
		isAuthenticated.value = false;
	} else if (status.enabled && status.authenticated) {
		isAuthenticated.value = true;
		currentUsername.value = localStorage.getItem("auth_username") || "";
	}
};

// 监听401事件
const handleAuthRequired = () => {
	showLogin.value = true;
};

// 登录成功处理
const handleLoginSuccess = () => {
	window.location.reload();
};

// 退出登录
const handleLogout = async () => {
	if (confirm("确定要退出登录吗？")) {
		await logout();
		window.location.reload();
	}
};

// 组件卸载时清除定时器
const getRouteParams = () => {
	const params = {};
	const searchParams = new URLSearchParams(window.location.search);
	for (const [key, value] of searchParams) {
		params[key] = value;
	}
	return params;
};
const searchForm = ref(null);
const keyword = ref("");
onMounted(() => {
	checkAuth();
	window.addEventListener("auth:required", handleAuthRequired);
	// 获取路由上的参数
	const params = getRouteParams();
	if (params.keyword) {
		searchForm.value.setKeyword(params.keyword);
		searchForm.value.handleSearch();
		// 搜索成功后清空路由上的参数 不然刷新还是会搜索
		window.history.replaceState({}, document.title, window.location.pathname);
	}
});

onUnmounted(() => {
	// 确保在组件卸载时清理所有定时器
	stopUpdate();
	window.removeEventListener("auth:required", handleAuthRequired);
});
</script>

<template>
	<div class="min-h-screen bg-background text-foreground transition-colors duration-300 flex flex-col">
		<!-- 登录对话框 -->
		<LoginDialog v-model:visible="showLogin" @success="handleLoginSuccess" />

		<!-- 背景装饰 -->
		<div class="bg-decorative"></div>

		<!-- 导航栏 -->
		<nav class="nav-header backdrop-blur-md bg-background/80 border-b border-border">
			<div class="container mx-auto px-4 h-16 flex items-center justify-between">
				<div class="flex items-center gap-3 cursor-pointer" @click="resetToInitial">
					<div class="w-8 h-8 bg-primary rounded-lg flex items-center justify-center">
						<svg class="w-5 h-5 text-primary-foreground" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"></path>
						</svg>
					</div>
					<div>
						<h1 class="text-xl font-bold">937影视 - PanSou</h1>
					</div>
				</div>

				<!-- 导航菜单 -->
				<nav class="flex items-center gap-2" v-if="currentPage === 'search'">
					<!-- <button @click="switchToStatus" class="nav-button">
						<span class="nav-icon">📊</span>
						状态
					</button> -->
					<!-- <button @click="switchToDocs" class="nav-button">
						<span class="nav-icon">📖</span>
						API文档
					</button> -->
					<button v-if="isAuthenticated" @click="handleLogout" class="nav-button logout-button" :title="'当前用户: ' + currentUsername">
						<span class="nav-icon">🚪</span>
						退出
					</button>
				</nav>
			</div>
		</nav>

		<!-- 主要内容区域 -->
		<main class="container mx-auto px-4 py-8 flex-1">
			<!-- 搜索页面 -->
			<div v-if="currentPage === 'search'" class="search-page">
				<!-- 搜索表单 -->
				<div class="mb-6">
					<SearchForm ref="searchForm" @search="handleSearch" @search-complete="handleSearchComplete" />
				</div>

				<!-- 搜索统计 -->
				<div v-if="hasSearched || loading" class="mb-6">
					<SearchStats
						:total="searchResults.total || 0"
						:mergedResults="searchResults.mergedResults || {}"
						:loading="loading"
						:searchTime="searchTime"
						:isUpdating="isUpdating"
						:updateCount="updateCount"
					/>
				</div>

				<!-- 加载状态 -->
				<div v-if="loading" class="card p-6">
					<div class="space-y-3">
						<div class="h-4 bg-muted rounded animate-pulse"></div>
						<div class="h-4 bg-muted rounded animate-pulse w-3/4"></div>
						<div class="h-4 bg-muted rounded animate-pulse w-1/2"></div>
						<div class="h-4 bg-muted rounded animate-pulse w-2/3"></div>
						<div class="h-4 bg-muted rounded animate-pulse"></div>
					</div>
				</div>

				<!-- 搜索结果 -->
				<div v-else>
					<ResultTabs
						:mergedResults="searchResults.mergedResults || {}"
						:loading="loading"
						:hasSearched="hasSearched"
						:isActivelySearching="isActivelySearching"
						:keyword="keyword"
					/>
				</div>
			</div>

			<!-- 状态页面 -->
			<div v-else-if="currentPage === 'status'" class="status-page">
				<ApiStatus />
			</div>

			<!-- API文档页面 -->
			<div v-else-if="currentPage === 'docs'" class="docs-page">
				<ApiDocs />
			</div>
		</main>

		<!-- 页脚 -->
		<footer class="border-t border-border bg-background/50 backdrop-blur-sm mt-auto">
			<div class="container mx-auto px-4 py-4">
				<div class="flex items-center justify-center gap-4 text-sm text-muted-foreground">
					<span>© {{ new Date().getFullYear() }}-{{ new Date().getFullYear() + 10 }}</span>
					<span>Powered by PanSou</span>
				</div>
				<br />
				<div class="flex items-center justify-center gap-4 text-sm text-muted-foreground">
					<span><a href="https://www.937tv.vip" target="_blank">937影视 - 全网视频免VIP</a></span>
				</div>
			</div>
		</footer>
	</div>
</template>

<style scoped>
.bg-decorative {
	position: fixed;
	inset: 0;
	z-index: -10;
	background-image: radial-gradient(circle at 1px 1px, hsl(var(--muted-foreground)) 1px, transparent 0);
	background-size: 20px 20px;
	opacity: 0.1;
}

.nav-header {
	position: sticky;
	top: 0;
	z-index: 50;
}

/* 导航按钮样式 */
.nav-button {
	display: flex;
	align-items: center;
	gap: 0.5rem;
	padding: 0.5rem 1rem;
	background: transparent;
	color: hsl(var(--muted-foreground));
	border: 1px solid hsl(var(--border));
	border-radius: 0.375rem;
	font-size: 0.875rem;
	font-weight: 500;
	cursor: pointer;
	transition: all 0.2s ease;
}

.nav-button:hover {
	background: hsl(var(--accent));
	color: hsl(var(--accent-foreground));
	border-color: hsl(var(--accent));
}

.logout-button {
	border-color: hsl(0, 84%, 60%);
	color: hsl(0, 84%, 60%);
}

.logout-button:hover {
	background: hsl(0, 84%, 95%);
	border-color: hsl(0, 84%, 60%);
	color: hsl(0, 84%, 50%);
}

@media (prefers-color-scheme: dark) {
	.logout-button:hover {
		background: hsl(0, 84%, 20%);
		color: hsl(0, 84%, 90%);
	}
}

.nav-icon {
	font-size: 1rem;
}

/* 页面切换动画 */
.search-page,
.status-page {
	animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
	from {
		opacity: 0;
		transform: translateY(10px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

@media (max-width: 768px) {
	.container {
		padding-left: 1rem;
		padding-right: 1rem;
	}

	.nav-button {
		padding: 0.375rem 0.75rem;
		font-size: 0.8rem;
	}

	.nav-icon {
		font-size: 0.875rem;
	}
}

/* 页脚按钮样式 */
footer button {
	background: transparent;
	border: none;
	padding: 0;
	font-size: inherit;
	color: inherit;
	cursor: pointer;
}
</style>
