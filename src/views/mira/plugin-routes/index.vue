<script setup lang="ts">
import type {
  LibraryInfo,
  PluginRouteDefinition,
} from '#/api/core/plugin-routes';

import { onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';

import { message } from 'ant-design-vue';

import { getPluginRoutesByLibrary } from '#/api/core/plugin-routes';
import LibraryPluginSelector from '#/components/Library/LibraryPluginSelector.vue';

defineOptions({ name: 'MiraPluginRoutes' });

const router = useRouter();

const loading = ref(false);
const selectedLibrary = ref<LibraryInfo | null>(null);
const pluginRoutes = ref<PluginRouteDefinition[]>([]);

const handleLibrarySelected = async (library: LibraryInfo) => {
  selectedLibrary.value = library;
  await loadPluginRoutes(library.id);
};

const loadPluginRoutes = async (libraryId: string) => {
  try {
    loading.value = true;
    const routes = await getPluginRoutesByLibrary(libraryId);
    pluginRoutes.value = routes;
  } catch (error) {
    void message.error('加载插件路由失败');
    console.error('Failed to load plugin routes:', error);
  } finally {
    loading.value = false;
  }
};

const refreshRoutes = async () => {
  if (selectedLibrary.value) {
    await loadPluginRoutes(selectedLibrary.value.id);
    void message.success('路由已刷新');
  }
};

const getRouteStatus = (route: PluginRouteDefinition) => {
  if (route.meta?.roles && route.meta.roles.length > 0) {
    return '受限';
  }
  return '可用';
};

const getRouteStatusClass = (route: PluginRouteDefinition) => {
  const status = getRouteStatus(route);
  switch (status) {
    case '受限': {
      return 'bg-warning/10 text-warning';
    }
    case '可用': {
      return 'bg-success/10 text-success';
    }
    default: {
      return 'bg-muted text-muted-foreground';
    }
  }
};

const previewRoute = (route: PluginRouteDefinition) => {
  void router.push(route.path);
};

const editRoute = (route: PluginRouteDefinition) => {
  void message.info(`编辑路由: ${route.name}`);
};

const openRouteInNewTab = (route: PluginRouteDefinition) => {
  // 在当前应用的标签页系统中打开新标签页
  // 使用唯一的pageKey来确保打开新的标签页
  const pageKey = `${route.name}_${Date.now()}`;
  router.push({
    path: route.path,
    query: {
      pageKey, // 使用pageKey参数来打开新的标签页
    },
  });
};

const createRoute = () => {
  void message.info('创建新路由功能开发中...');
};

const getIconClass = (iconName?: string) => {
  if (!iconName) return '';
  // 确保图标名称格式正确
  if (iconName.startsWith('lucide--')) {
    return `icon-[${iconName}]`;
  }
  // 如果不是 lucide 图标，添加前缀
  return `icon-[lucide--${iconName}]`;
};

onMounted(() => {
  // 页面加载时预加载数据
});
</script>

<template>
  <div class="plugin-routes-management">
    <!-- 页面头部 -->
    <div class="mb-6 flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-bold">插件路由管理</h1>
        <p class="text-muted-foreground mt-1">
          管理各个素材库的插件路由，查看和配置动态加载的插件页面
        </p>
      </div>

      <!-- 素材库选择器 -->
      <LibraryPluginSelector @library-selected="handleLibrarySelected" />
    </div>

    <!-- 选中的素材库信息 -->
    <div v-if="selectedLibrary" class="bg-muted/50 mb-6 rounded-lg border p-4">
      <div class="flex items-center justify-between">
        <div>
          <h3 class="text-lg font-semibold">{{ selectedLibrary.name }}</h3>
          <p class="text-muted-foreground text-sm">
            {{ selectedLibrary.routeCount }} 个可用的插件路由
          </p>
        </div>
        <button
          class="bg-primary text-primary-foreground hover:bg-primary/90 rounded-md px-4 py-2 transition-colors"
          @click="refreshRoutes"
        >
          <span
            class="icon-[lucide--refresh-cw] mr-2"
            :class="{ 'animate-spin': loading }"
          ></span>
          刷新路由
        </button>
      </div>
    </div>

    <!-- 插件路由列表 -->
    <div v-if="selectedLibrary" class="space-y-4">
      <!-- 加载状态 -->
      <div
        v-if="loading"
        class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3"
      >
        <div v-for="i in 6" :key="i" class="animate-pulse">
          <div class="bg-muted/30 space-y-3 rounded-lg p-4">
            <div class="flex items-center space-x-3">
              <div class="bg-muted h-8 w-8 rounded"></div>
              <div class="flex-1 space-y-2">
                <div class="bg-muted h-4 w-3/4 rounded"></div>
                <div class="bg-muted h-3 w-1/2 rounded"></div>
              </div>
            </div>
            <div class="space-y-2">
              <div class="bg-muted h-3 rounded"></div>
              <div class="bg-muted h-3 w-5/6 rounded"></div>
            </div>
          </div>
        </div>
      </div>

      <!-- 路由卡片 -->
      <div
        v-else-if="pluginRoutes.length > 0"
        class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3"
      >
        <div
          v-for="route in pluginRoutes"
          :key="route.path"
          class="bg-card cursor-pointer rounded-lg border p-4 transition-shadow hover:shadow-md"
          @click="previewRoute(route)"
        >
          <div class="mb-3 flex items-start justify-between">
            <div class="flex items-center space-x-3">
              <div
                class="bg-primary/10 flex h-8 w-8 items-center justify-center rounded"
              >
                <span
                  v-if="route.meta?.icon"
                  :class="getIconClass(route.meta.icon)"
                  class="text-primary"
                ></span>
                <span v-else class="icon-[lucide--puzzle] text-primary"></span>
              </div>
              <div>
                <h4 class="text-sm font-semibold">
                  {{ route.meta?.title || route.name }}
                </h4>
                <p class="text-muted-foreground text-xs">{{ route.name }}</p>
              </div>
            </div>
            <span
              class="inline-flex items-center rounded-full px-2 py-1 text-xs font-medium"
              :class="getRouteStatusClass(route)"
            >
              {{ getRouteStatus(route) }}
            </span>
          </div>

          <div class="space-y-2 text-sm">
            <div class="text-muted-foreground flex items-center">
              <span class="icon-[lucide--route] mr-2 text-xs"></span>
              <span class="font-mono text-xs">{{ route.path }}</span>
            </div>

            <div
              v-if="route.meta?.roles"
              class="text-muted-foreground flex items-center"
            >
              <span class="icon-[lucide--shield] mr-2 text-xs"></span>
              <span class="text-xs">
                需要角色: {{ route.meta.roles.join(', ') }}
              </span>
            </div>

            <div
              v-if="route.meta?.order"
              class="text-muted-foreground flex items-center"
            >
              <span class="icon-[lucide--list-ordered] mr-2 text-xs"></span>
              <span class="text-xs">排序: {{ route.meta.order }}</span>
            </div>
          </div>

          <div class="mt-3 flex items-center justify-between border-t pt-3">
            <button
              class="text-primary hover:text-primary/80 text-xs transition-colors"
              @click.stop="editRoute(route)"
            >
              <span class="icon-[lucide--edit] mr-1"></span>
              编辑
            </button>
            <button
              class="text-muted-foreground hover:text-foreground text-xs transition-colors"
              @click.stop="openRouteInNewTab(route)"
            >
              <span class="icon-[lucide--external-link] mr-1"></span>
              打开
            </button>
          </div>
        </div>
      </div>

      <!-- 无数据状态 -->
      <div v-else class="py-12 text-center">
        <div
          class="bg-muted/30 mx-auto mb-4 flex h-16 w-16 items-center justify-center rounded-full"
        >
          <span
            class="icon-[lucide--route] text-muted-foreground text-2xl"
          ></span>
        </div>
        <h3 class="mb-2 text-lg font-semibold">暂无插件路由</h3>
        <p class="text-muted-foreground mb-4">
          {{ selectedLibrary.name }} 素材库中暂未配置任何插件路由
        </p>
        <button
          class="bg-primary text-primary-foreground hover:bg-primary/90 rounded-md px-4 py-2 transition-colors"
          @click="createRoute"
        >
          <span class="icon-[lucide--plus] mr-2"></span>
          创建路由
        </button>
      </div>
    </div>

    <!-- 未选择素材库 -->
    <div v-else class="py-16 text-center">
      <div
        class="bg-muted/30 mx-auto mb-6 flex h-20 w-20 items-center justify-center rounded-full"
      >
        <span
          class="icon-[lucide--layers] text-muted-foreground text-3xl"
        ></span>
      </div>
      <h3 class="mb-2 text-xl font-semibold">选择一个素材库</h3>
      <p class="text-muted-foreground mb-6">
        请使用右上角的素材库选择器选择要管理插件路由的素材库
      </p>
      <LibraryPluginSelector @library-selected="handleLibrarySelected" />
    </div>
  </div>
</template>

<style scoped>
.plugin-routes-management {
  max-width: 80rem;
  margin: 0 auto;
  padding: 1.5rem;
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.5;
  }
}
</style>
