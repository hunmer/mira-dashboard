<script setup lang="ts">
import type { PluginRouteDefinition } from '#/api/core/plugin-routes';
import type { Plugin } from '#/types/mira';

import { computed, onMounted, reactive, ref } from 'vue';
import { useRouter } from 'vue-router';

import { useVbenDrawer, useVbenModal } from '@vben/common-ui';

import { Modal, notification } from 'ant-design-vue';

import { getPluginRoutesByLibrary } from '#/api/core/plugin-routes';
import miraApiClient from '#/api/mira/client';

defineOptions({ name: 'MiraPlugin' });

const router = useRouter();

// 定义接口
interface LibraryWithPlugins {
  id: string;
  name: string;
  description: string;
  plugins: Plugin[];
}

// 响应式数据
const loading = ref(false);
const showConfigDialog = ref(false);
// 使用 VbenDrawer 替代自定义 drawer
const [PluginDetailDrawer, pluginDetailDrawerApi] = useVbenDrawer({
  title: '插件详细信息',
  class: 'w-[500px]',
  footer: false,
});
const installTab = ref('local');
const configuringPlugin = ref<null | Plugin>(null);
const selectedPlugin = ref<null | Plugin>(null);
const pluginConfig = ref('');
const selectedFile = ref<File | null>(null);
const librariesWithPlugins = ref<LibraryWithPlugins[]>([]);
const activeLibraryTab = ref('');
const activeDropdown = ref<null | string>(null);
const pluginRoutes = reactive<{ [key: string]: any[] }>({});
const dropdownPosition = reactive({ x: 0, y: 0 });
const selectedPluginForAction = ref<null | Plugin>(null);

// 使用 VbenModal 创建安装插件对话框
const [VbenModal, modalApi] = useVbenModal({
  title: '安装插件',
  class: 'w-[500px]',
});

// 每个素材库的搜索、排序、分页状态
const searchKeywords = reactive<{ [key: string]: string }>({});
const sortOptions = reactive<{ [key: string]: string }>({});
const categoryFilters = reactive<{ [key: string]: string }>({});
const currentInstallLibraryId = ref<string>('');

const installForm = ref({
  name: '',
  version: 'latest',
});

// 计算属性
const totalPluginsCount = computed(() => {
  return librariesWithPlugins.value.reduce(
    (total, library) => total + library.plugins.length,
    0,
  );
});

const activePluginsCount = computed(() => {
  return librariesWithPlugins.value.reduce(
    (total, library) =>
      total + library.plugins.filter((p) => p.status === 'active').length,
    0,
  );
});

const inactivePluginsCount = computed(() => {
  return librariesWithPlugins.value.reduce(
    (total, library) =>
      total + library.plugins.filter((p) => p.status === 'inactive').length,
    0,
  );
});

const dropdownStyle = computed(() => ({
  left: `${dropdownPosition.x}px`,
  top: `${dropdownPosition.y}px`,
  position: 'fixed' as const,
}));

const getPluginRoutesForLibrary = (libraryId: string, pluginName: string) => {
  const routes = pluginRoutes[libraryId] || [];
  return routes.filter((route) => route.pluginName === pluginName);
};

// 方法
const getCategoryDisplayName = (category?: string) => {
  const categoryMap: { [key: string]: string } = {
    general: '通用',
    security: '安全',
    storage: '存储',
    ui: '界面',
    utility: '工具',
    integration: '集成',
    development: '开发',
  };
  return categoryMap[category || 'general'] || category || '通用';
};

const getAvailableCategories = (plugins: Plugin[]) => {
  const categories = new Set(plugins.map((p) => p.category || 'general'));
  return [...categories].sort();
};

const getActiveCount = (plugins: Plugin[]) => {
  return plugins.filter((p) => p.status === 'active').length;
};

const getInactiveCount = (plugins: Plugin[]) => {
  return plugins.filter((p) => p.status === 'inactive').length;
};

const getFilteredPlugins = (library: LibraryWithPlugins) => {
  let result = library.plugins;

  // 搜索过滤
  const searchKeyword = searchKeywords[library.id] || '';
  if (searchKeyword) {
    const keyword = searchKeyword.toLowerCase();
    result = result.filter(
      (plugin) =>
        plugin.name.toLowerCase().includes(keyword) ||
        plugin.author.toLowerCase().includes(keyword) ||
        (plugin.description &&
          plugin.description.toLowerCase().includes(keyword)),
    );
  }

  // 分类过滤
  const categoryFilter = categoryFilters[library.id] || '';
  if (categoryFilter) {
    result = result.filter((plugin) => plugin.category === categoryFilter);
  }

  // 排序
  const sortBy = sortOptions[library.id] || 'status';
  result.sort((a, b) => {
    switch (sortBy) {
      case 'author': {
        return a.author.localeCompare(b.author);
      }
      case 'category': {
        return (a.category || '').localeCompare(b.category || '');
      }
      case 'createdAt': {
        return (
          new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
        );
      }
      case 'name': {
        return a.name.localeCompare(b.name);
      }
      case 'status': {
        // 已启用排在前面
        if (a.status !== b.status) {
          return a.status === 'active' ? -1 : 1;
        }
        return a.name.localeCompare(b.name);
      }
      default: {
        return 0;
      }
    }
  });

  return result;
};

const handleIconError = (event: Event) => {
  // 当图标加载失败时，隐藏图片元素
  const img = event.target as HTMLImageElement;
  img.style.display = 'none';
};

const toggleDropdown = (
  pluginName: string,
  plugin: Plugin,
  event: MouseEvent,
) => {
  if (activeDropdown.value === pluginName) {
    activeDropdown.value = null;
    selectedPluginForAction.value = null;
  } else {
    activeDropdown.value = pluginName;
    selectedPluginForAction.value = plugin;

    // Calculate position for the dropdown
    const rect = (event.target as HTMLElement).getBoundingClientRect();
    dropdownPosition.x = rect.right - 128; // 128px is the width of dropdown
    dropdownPosition.y = rect.bottom + 4;
  }
};

const openInstallDialog = (libraryId: string) => {
  currentInstallLibraryId.value = libraryId;
  const library = librariesWithPlugins.value.find(
    (lib) => lib.id === libraryId,
  );
  const libraryName = library ? library.name || library.id : '插件';
  modalApi.setState({ title: `为 ${libraryName} 安装插件` }).open();
};

const handleSearch = (_libraryId: string) => {
  // 搜索逻辑已在getFilteredPlugins中实现
};

const handleSort = (_libraryId: string) => {
  // 排序逻辑已在getFilteredPlugins中实现
};

const handleFilter = (_libraryId: string) => {
  // 过滤逻辑已在getFilteredPlugins中实现
};

const showPluginDetail = (plugin: Plugin) => {
  selectedPlugin.value = plugin;
  pluginDetailDrawerApi.open();
  activeDropdown.value = null;
};

const formatDate = (dateString: string) => {
  return new Date(dateString).toLocaleString('zh-CN');
};

const loadPluginRoutes = async (libraryId: string) => {
  try {
    const routes = await getPluginRoutesByLibrary(libraryId);
    pluginRoutes[libraryId] = routes || [];
  } catch (error) {
    console.error('Failed to load plugin routes:', error);
    pluginRoutes[libraryId] = [];
  }
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

const loadLibrariesWithPlugins = async () => {
  loading.value = true;
  try {
    const response = await miraApiClient.get('/plugins/by-library');
    librariesWithPlugins.value = (response.data as LibraryWithPlugins[]) || [];

    // 初始化各库的状态
    librariesWithPlugins.value.forEach((library) => {
      if (!searchKeywords[library.id]) searchKeywords[library.id] = '';
      if (!sortOptions[library.id]) sortOptions[library.id] = 'status';
      if (!categoryFilters[library.id]) categoryFilters[library.id] = '';
    });

    // 设置默认活动标签
    if (librariesWithPlugins.value.length > 0 && !activeLibraryTab.value) {
      activeLibraryTab.value = librariesWithPlugins.value[0]!.id;
    }

    // 加载每个库的插件路由
    for (const library of librariesWithPlugins.value) {
      await loadPluginRoutes(library.id);
    }
  } catch (error) {
    notification.error({
      message: '加载失败',
      description: '加载插件列表失败，请稍后重试',
    });
    console.error('Failed to load plugins:', error);
    librariesWithPlugins.value = [];
  } finally {
    loading.value = false;
  }
};

const togglePlugin = async (plugin: Plugin, checked?: boolean) => {
  try {
    const newStatus =
      checked === undefined
        ? plugin.status === 'active'
          ? 'inactive'
          : 'active'
        : checked
          ? 'active'
          : 'inactive';

    // 使用POST接口避免URL字符冲突
    await miraApiClient.post('/plugins/toggle-status', {
      pluginName: plugin.name,
      libraryId: plugin.libraryId,
      status: newStatus,
    });

    plugin.status = newStatus;

    // 如果在详情面板中，也要更新选中的插件状态
    if (selectedPlugin.value && selectedPlugin.value.name === plugin.name) {
      selectedPlugin.value.status = newStatus;
    }

    notification.success({
      message: '状态更新',
      description: `插件已${newStatus === 'active' ? '启用' : '禁用'}`,
    });
  } catch (error: any) {
    console.error('Toggle plugin error:', error);
    notification.error({
      message: '操作失败',
      description: error.response?.data?.error || error.message || '未知错误',
    });
  }
};

const configurePlugin = async (plugin: Plugin) => {
  try {
    const response = await miraApiClient.get(`/plugins/${plugin.name}/config`);
    pluginConfig.value = JSON.stringify(response.data, null, 2);
    configuringPlugin.value = plugin;
    showConfigDialog.value = true;
    pluginDetailDrawerApi.close(); // 关闭详情面板
  } catch {
    notification.error({
      message: '加载失败',
      description: '加载插件配置失败',
    });
  }
};

const savePluginConfig = async () => {
  if (!configuringPlugin.value) return;

  try {
    const config = JSON.parse(pluginConfig.value);
    await miraApiClient.put(
      `/plugins/${configuringPlugin.value.name}/config`,
      config,
    );
    notification.success({
      message: '保存成功',
      description: '配置保存成功',
    });
    showConfigDialog.value = false;
  } catch (error: any) {
    if (error instanceof SyntaxError) {
      notification.error({
        message: '格式错误',
        description: 'JSON 格式错误',
      });
    } else {
      notification.error({
        message: '保存失败',
        description: '保存失败',
      });
    }
  }
};

const handlePluginAction = async (command: string, plugin: Plugin) => {
  activeDropdown.value = null;

  switch (command) {
    case 'uninstall': {
      try {
        Modal.confirm({
          title: '确认卸载',
          content: `确定要卸载插件 "${plugin.name}" 吗？此操作不可撤销。`,
          onOk: async () => {
            await miraApiClient.delete(`/plugins/${plugin.name}`);
            notification.success({
              message: '卸载成功',
              description: '插件卸载成功',
            });

            // 如果卸载的是当前选中的插件，关闭详情面板
            if (
              selectedPlugin.value &&
              selectedPlugin.value.name === plugin.name
            ) {
              pluginDetailDrawerApi.close();
              selectedPlugin.value = null;
            }

            loadLibrariesWithPlugins();
          },
        });
      } catch {
        notification.error({
          message: '卸载失败',
          description: '卸载失败',
        });
      }
      break;
    }

    case 'update': {
      try {
        await miraApiClient.post(`/plugins/${plugin.name}/update`);
        notification.success({
          message: '更新成功',
          description: '插件更新成功',
        });
        loadLibrariesWithPlugins();
      } catch {
        notification.error({
          message: '更新失败',
          description: '更新失败',
        });
      }
      break;
    }
  }
};

const handleFileSelect = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    selectedFile.value = target.files[0] || null;
  }
};

const cancelInstall = () => {
  modalApi.close();
  selectedFile.value = null;
  currentInstallLibraryId.value = '';
  installForm.value = { name: '', version: 'latest' };
};

const handleInstallOk = async () => {
  await (installTab.value === 'repository'
    ? installFromRepository()
    : uploadPlugin());
};

const uploadPlugin = async () => {
  if (!selectedFile.value) {
    notification.error({
      message: '文件错误',
      description: '请选择插件包文件',
    });
    return;
  }

  try {
    const formData = new FormData();
    formData.append('file', selectedFile.value);
    if (currentInstallLibraryId.value) {
      formData.append('libraryId', currentInstallLibraryId.value);
    }

    await miraApiClient.post('/plugins/upload', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
      },
    });

    notification.success({
      message: '安装成功',
      description: '插件上传安装成功，稍后刷新插件列表',
    });

    cancelInstall();

    // 延迟3秒刷新插件列表
    setTimeout(() => {
      loadLibrariesWithPlugins();
    }, 3000);
  } catch {
    notification.error({
      message: '安装失败',
      description: '插件安装失败',
    });
  }
};

const installFromRepository = async () => {
  if (!installForm.value.name) {
    notification.error({
      message: '输入错误',
      description: '请输入插件名称',
    });
    return;
  }

  try {
    const requestData = {
      ...installForm.value,
      libraryId: currentInstallLibraryId.value,
    };
    await miraApiClient.post('/plugins/install', requestData);
    notification.success({
      message: '安装成功',
      description: '插件安装成功，稍后刷新插件列表',
    });

    cancelInstall();

    // 延迟3秒刷新插件列表
    setTimeout(() => {
      loadLibrariesWithPlugins();
    }, 3000);
  } catch (error: any) {
    if (error.response?.data?.error) {
      notification.error({
        message: '安装失败',
        description: error.response.data.error,
      });
    } else {
      notification.error({
        message: '安装失败',
        description: '安装失败',
      });
    }
  }
};

onMounted(() => {
  loadLibrariesWithPlugins();
});
</script>

<template>
  <div class="plugin-manager">
    <!-- 总体统计卡片 -->
    <div class="mb-8 grid grid-cols-1 gap-6 md:grid-cols-3">
      <div class="stats-card total-plugins">
        <div class="stats-content">
          <div class="stats-icon">🔧</div>
          <div class="stats-info">
            <h3>总插件数</h3>
            <p class="stats-number">{{ totalPluginsCount }}</p>
          </div>
        </div>
      </div>

      <div class="stats-card active-plugins">
        <div class="stats-content">
          <div class="stats-icon">✅</div>
          <div class="stats-info">
            <h3>已启用</h3>
            <p class="stats-number">{{ activePluginsCount }}</p>
          </div>
        </div>
      </div>

      <div class="stats-card inactive-plugins">
        <div class="stats-content">
          <div class="stats-icon">❌</div>
          <div class="stats-info">
            <h3>已禁用</h3>
            <p class="stats-number">{{ inactivePluginsCount }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- 资源库标签页 -->
    <div class="library-tabs">
      <div class="tab-nav">
        <button
          v-for="library in librariesWithPlugins"
          :key="library.id"
          class="tab-button"
          :class="[{ active: activeLibraryTab === library.id }]"
          @click="activeLibraryTab = library.id"
        >
          {{ library.name || library.id }}
          <span class="tab-count">{{ library.plugins.length }}</span>
        </button>
      </div>

      <!-- 当前库的插件内容 -->
      <div
        v-for="library in librariesWithPlugins"
        :key="library.id"
        class="tab-content"
      >
        <div v-if="activeLibraryTab === library.id">
          <!-- 控制栏背景 -->
          <div class="mb-6 flex flex-wrap gap-4 rounded-lg p-4">
            <div class="relative min-w-64 flex-1">
              <div
                class="pointer-events-none absolute inset-y-0 left-0 flex items-center pl-3"
              >
                <svg
                  class="h-5 w-5"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
                  />
                </svg>
              </div>
              <input
                v-model="searchKeywords[library.id]"
                type="text"
                placeholder="搜索插件名称、作者或描述"
                class="block w-full rounded-md border border-gray-300 py-2 pl-10 pr-3 leading-5 placeholder-gray-500 focus:border-blue-500 focus:placeholder-gray-400 focus:outline-none focus:ring-1 focus:ring-blue-500"
                @input="handleSearch(library.id)"
              />
            </div>

            <select
              v-model="sortOptions[library.id]"
              class="block rounded-md border border-gray-300 px-3 py-2 leading-5 focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              @change="handleSort(library.id)"
            >
              <option value="status">启用状态</option>
              <option value="name">名称</option>
              <option value="author">作者</option>
              <option value="createdAt">安装时间</option>
              <option value="category">分类</option>
            </select>

            <select
              v-model="categoryFilters[library.id]"
              class="block rounded-md border border-gray-300 px-3 py-2 leading-5 focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              @change="handleFilter(library.id)"
            >
              <option value="">全部分类</option>
              <option
                v-for="category in getAvailableCategories(library.plugins)"
                :key="category"
                :value="category"
              >
                {{ getCategoryDisplayName(category) }}
              </option>
            </select>

            <button
              type="button"
              @click="openInstallDialog(library.id)"
              class="inline-flex items-center rounded-md border border-transparent bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
            >
              <svg
                class="-ml-1 mr-2 h-4 w-4"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M12 4v16m8-8H4"
                />
              </svg>
              安装插件
            </button>
          </div>

          <!-- 当前库插件统计 -->
          <div class="mb-6 flex gap-4">
            <div class="library-stat">
              <div class="stat-title">插件数量</div>
              <div class="stat-value">{{ library.plugins.length }}</div>
            </div>
            <div class="library-stat">
              <div class="stat-title">已启用</div>
              <div class="stat-value">
                {{ getActiveCount(library.plugins) }}
              </div>
            </div>
            <div class="library-stat">
              <div class="stat-title">已禁用</div>
              <div class="stat-value">
                {{ getInactiveCount(library.plugins) }}
              </div>
            </div>
          </div>

          <!-- 插件网格视图 -->
          <div
            v-if="loading && library.plugins.length === 0"
            class="grid grid-cols-1 gap-6 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4"
          >
            <div v-for="i in 8" :key="i" class="plugin-skeleton">
              <div class="animate-pulse">
                <div class="mb-4 flex items-center space-x-4">
                  <div class="h-10 w-10 rounded-lg"></div>
                  <div class="flex-1">
                    <div class="mb-2 h-4 w-3/4 rounded"></div>
                    <div class="h-3 w-1/2 rounded"></div>
                  </div>
                  <div class="h-6 w-12 rounded"></div>
                </div>
                <div class="space-y-2">
                  <div class="h-3 rounded"></div>
                  <div class="h-3 w-5/6 rounded"></div>
                </div>
                <div class="mt-4 flex justify-between">
                  <div class="h-6 w-12 rounded"></div>
                  <div class="h-6 w-12 rounded"></div>
                </div>
              </div>
            </div>
          </div>

          <div
            v-else
            class="grid grid-cols-1 gap-6 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4"
          >
            <div
              v-for="plugin in getFilteredPlugins(library)"
              :key="plugin.name"
              class="plugin-card cursor-pointer transition-all duration-200"
              :class="[
                {
                  'border-green-200': plugin.status === 'active',
                  'border-gray-200': plugin.status === 'inactive',
                },
              ]"
            >
              <div class="plugin-header mb-4 flex items-center justify-between">
                <div class="flex items-center">
                  <div
                    class="mr-3 flex h-10 w-10 items-center justify-center rounded-lg"
                  >
                    <img
                      v-if="plugin.icon"
                      :src="plugin.icon"
                      :alt="plugin.name"
                      class="h-8 w-8 object-contain"
                      @error="handleIconError"
                    />
                    <span v-else class="text-xl">🔧</span>
                  </div>
                  <div>
                    <h3 class="truncate text-lg font-semibold">
                      {{ plugin.name }}
                    </h3>
                    <p class="text-sm">v{{ plugin.version }}</p>
                  </div>
                </div>
                <label class="switch">
                  <input
                    type="checkbox"
                    :checked="plugin.status === 'active'"
                    @change="
                      (e) =>
                        togglePlugin(
                          plugin,
                          (e.target as HTMLInputElement).checked,
                        )
                    "
                  />
                  <span class="slider"></span>
                </label>
              </div>

              <p class="mb-4 line-clamp-2 text-sm">
                {{ plugin.description || '暂无描述' }}
              </p>

              <div class="plugin-info mb-4 space-y-2">
                <div class="flex justify-between text-sm">
                  <span>作者:</span>
                  <span class="ml-2 truncate">{{ plugin.author }}</span>
                </div>
                <div class="flex justify-between text-sm">
                  <span>分类:</span>
                  <span class="rounded px-2 py-1 text-xs">{{
                    getCategoryDisplayName(plugin.category)
                  }}</span>
                </div>
              </div>

              <div class="plugin-actions mt-auto flex gap-2">
                <button
                  type="button"
                  @click="showPluginDetail(plugin)"
                  class="flex-1 rounded bg-blue-600 px-3 py-2 text-sm text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
                >
                  详情
                </button>

                <button
                  v-if="plugin.configurable"
                  type="button"
                  @click="configurePlugin(plugin)"
                  class="rounded bg-gray-600 px-3 py-2 text-sm text-white hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-gray-500"
                >
                  配置
                </button>

                <button
                  type="button"
                  @click="toggleDropdown(plugin.name, plugin, $event)"
                  class="rounded bg-gray-500 px-3 py-2 text-sm text-white hover:bg-gray-600 focus:outline-none focus:ring-2"
                >
                  更多
                  <svg
                    class="ml-1 inline h-3 w-3"
                    fill="none"
                    stroke="currentColor"
                    viewBox="0 0 24 24"
                  >
                    <path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      stroke-width="2"
                      d="M19 9l-7 7-7-7"
                    />
                  </svg>
                </button>
              </div>

              <!-- 插件入口按钮 -->
              <div
                v-if="
                  getPluginRoutesForLibrary(library.id, plugin.name).length > 0
                "
                class="mt-3 border-t pt-3"
              >
                <h4 class="mb-2 text-xs font-semibold text-gray-600">
                  插件入口
                </h4>
                <div class="flex flex-wrap gap-1">
                  <button
                    v-for="route in getPluginRoutesForLibrary(
                      library.id,
                      plugin.name,
                    )"
                    :key="route.path"
                    @click="openRouteInNewTab(route)"
                    class="rounded bg-indigo-100 px-2 py-1 text-xs text-indigo-700 hover:bg-indigo-200 focus:outline-none focus:ring-1 focus:ring-indigo-500"
                  >
                    {{ route.meta?.title || route.name }}
                  </button>
                </div>
              </div>
            </div>
          </div>

          <!-- 空状态 -->
          <div
            v-if="!loading && getFilteredPlugins(library).length === 0"
            class="py-12 text-center"
          >
            <div class="mb-4 text-4xl">🔧</div>
            <p class="mb-2 text-lg font-medium">
              {{
                searchKeywords[library.id] || categoryFilters[library.id]
                  ? '没有找到匹配的插件'
                  : '暂无插件'
              }}
            </p>
            <p class="text-sm">
              {{
                searchKeywords[library.id] || categoryFilters[library.id]
                  ? '请尝试调整搜索条件'
                  : '点击"安装插件"开始添加'
              }}
            </p>
          </div>
        </div>
      </div>
    </div>

    <!-- 使用 VbenDrawer 替代自定义抽屉 -->
    <PluginDetailDrawer>
      <div v-if="selectedPlugin" class="plugin-detail">
        <div class="mb-6 text-center">
          <div
            class="mx-auto mb-4 flex h-16 w-16 items-center justify-center rounded-lg"
          >
            <img
              v-if="selectedPlugin.icon"
              :src="selectedPlugin.icon"
              :alt="selectedPlugin.name"
              class="h-12 w-12 object-contain"
              @error="handleIconError"
            />
            <span v-else class="text-3xl">🔧</span>
          </div>
          <h2 class="text-xl font-bold">{{ selectedPlugin.name }}</h2>
          <p>v{{ selectedPlugin.version }}</p>
          <span
            class="mt-2 inline-block rounded-full px-3 py-1 text-sm font-medium"
            :class="[
              selectedPlugin.status === 'active'
                ? 'bg-green-100 text-green-800'
                : 'bg-gray-100 text-gray-800',
            ]"
          >
            {{ selectedPlugin.status === 'active' ? '已启用' : '已禁用' }}
          </span>
        </div>

        <div class="space-y-4">
          <div class="detail-item">
            <label class="detail-label">描述</label>
            <div class="detail-value">
              {{ selectedPlugin.description || '暂无描述' }}
            </div>
          </div>
          <div class="detail-item">
            <label class="detail-label">作者</label>
            <div class="detail-value">{{ selectedPlugin.author }}</div>
          </div>
          <div class="detail-item">
            <label class="detail-label">分类</label>
            <div class="detail-value">
              {{ getCategoryDisplayName(selectedPlugin.category) }}
            </div>
          </div>
          <div class="detail-item">
            <label class="detail-label">所属库</label>
            <div class="detail-value">
              {{
                selectedPlugin.libraryName || selectedPlugin.libraryId || '未知'
              }}
            </div>
          </div>
          <div class="detail-item">
            <label class="detail-label">依赖数量</label>
            <div class="detail-value">
              {{ selectedPlugin.dependencies.length }} 个
            </div>
          </div>
          <div class="detail-item">
            <label class="detail-label">入口文件</label>
            <div class="detail-value">{{ selectedPlugin.main }}</div>
          </div>
          <div class="detail-item">
            <label class="detail-label">安装时间</label>
            <div class="detail-value">
              {{ formatDate(selectedPlugin.createdAt) }}
            </div>
          </div>
          <div class="detail-item">
            <label class="detail-label">更新时间</label>
            <div class="detail-value">
              {{ formatDate(selectedPlugin.updatedAt) }}
            </div>
          </div>
        </div>

        <div
          v-if="selectedPlugin.tags && selectedPlugin.tags.length > 0"
          class="mt-6"
        >
          <h4 class="mb-2 font-semibold">标签</h4>
          <div class="flex flex-wrap gap-2">
            <span
              v-for="tag in selectedPlugin.tags"
              :key="tag"
              class="rounded bg-blue-100 px-2 py-1 text-xs text-blue-800"
            >
              {{ tag }}
            </span>
          </div>
        </div>

        <div v-if="selectedPlugin.dependencies.length > 0" class="mt-6">
          <h4 class="mb-2 font-semibold">依赖项</h4>
          <div class="space-y-1">
            <span
              v-for="dep in selectedPlugin.dependencies"
              :key="dep"
              class="block rounded bg-gray-100 px-2 py-1 text-xs text-gray-700"
            >
              {{ dep }}
            </span>
          </div>
        </div>

        <div class="mt-6 flex gap-2">
          <button
            type="button"
            :disabled="!selectedPlugin.configurable"
            @click="configurePlugin(selectedPlugin)"
            class="flex-1 rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:cursor-not-allowed disabled:opacity-50"
          >
            配置插件
          </button>
          <button
            type="button"
            @click="
              togglePlugin(selectedPlugin, selectedPlugin.status !== 'active')
            "
            class="rounded bg-gray-600 px-4 py-2 text-white hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-gray-500"
          >
            {{ selectedPlugin.status === 'active' ? '禁用' : '启用' }}
          </button>
        </div>
      </div>
    </PluginDetailDrawer>

    <!-- 插件配置对话框 -->
    <div
      v-if="showConfigDialog"
      class="fixed inset-0 z-50 h-full w-full overflow-y-auto bg-gray-600 bg-opacity-50"
    >
      <div
        class="relative top-10 mx-auto w-2/3 max-w-4xl rounded-md border p-5 shadow-lg"
      >
        <div class="mt-3">
          <h3 class="mb-4 text-lg font-medium">
            配置 {{ configuringPlugin?.name }}
          </h3>

          <div v-if="configuringPlugin" class="config-editor">
            <!-- 使用 JSON 查看器替代 Monaco 编辑器 -->
            <div class="mb-4">
              <h4 class="mb-2 text-sm font-semibold">配置预览</h4>
              <div class="max-h-96 overflow-auto rounded border p-4">
                <pre class="whitespace-pre-wrap text-sm">{{
                  pluginConfig
                }}</pre>
              </div>
            </div>
          </div>

          <div class="flex justify-end space-x-3 pt-4">
            <button
              type="button"
              @click="showConfigDialog = false"
              class="rounded-md border px-4 py-2 text-sm font-medium focus:outline-none focus:ring-2 focus:ring-offset-2"
            >
              取消
            </button>
            <button
              type="button"
              @click="savePluginConfig"
              :disabled="loading"
              class="rounded-md border border-transparent px-4 py-2 text-sm font-medium text-white shadow-sm hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50"
            >
              {{ loading ? '保存中...' : '保存' }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- VbenModal 安装插件对话框 -->
    <VbenModal
      :loading="loading"
      :confirm-loading="loading"
      @confirm="handleInstallOk"
      @cancel="cancelInstall"
    >
      <!-- 安装方式选择 -->
      <div class="mb-4">
        <div class="flex border-b">
          <button
            class="px-4 py-2 text-sm font-medium"
            :class="[
              installTab === 'local' ? 'border-b-2 border-blue-500' : '',
            ]"
            @click="installTab = 'local'"
          >
            从本地安装
          </button>
          <button
            class="px-4 py-2 text-sm font-medium"
            :class="[
              installTab === 'repository' ? 'border-b-2 border-blue-500' : '',
            ]"
            @click="installTab = 'repository'"
          >
            从仓库安装
          </button>
        </div>
      </div>

      <!-- 本地安装 -->
      <div v-if="installTab === 'local'" class="space-y-4">
        <div>
          <label class="mb-2 block text-sm font-medium">选择插件包</label>
          <input
            type="file"
            accept=".zip,.tar.gz"
            @change="handleFileSelect"
            class="block w-full text-sm file:mr-4 file:rounded-full file:border-0 file:bg-blue-50 file:px-4 file:py-2 file:text-sm file:font-semibold hover:file:bg-blue-100"
          />
          <p class="mt-1 text-xs">支持 .zip 和 .tar.gz 格式的插件包</p>
        </div>
        <div v-if="selectedFile" class="text-sm">
          已选择: {{ selectedFile.name }}
        </div>
      </div>

      <!-- 仓库安装 -->
      <div v-if="installTab === 'repository'" class="space-y-4">
        <div>
          <label class="mb-1 block text-sm font-medium">插件名称</label>
          <input
            v-model="installForm.name"
            type="text"
            placeholder="请输入npm包名称，如：mira-plugin-example"
            class="w-full rounded-md border border-gray-300 px-3 py-2 focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
          />
        </div>
        <div>
          <label class="mb-1 block text-sm font-medium">版本</label>
          <input
            v-model="installForm.version"
            type="text"
            placeholder="latest"
            class="w-full rounded-md border border-gray-300 px-3 py-2 focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
          />
        </div>
      </div>
    </VbenModal>

    <!-- 全局插件操作菜单 -->
    <div
      v-if="activeDropdown"
      class="fixed z-[9999] mt-1 w-32 rounded-md border border-gray-200 bg-white shadow-lg"
      :style="dropdownStyle"
    >
      <button
        @click="
          selectedPluginForAction &&
          handlePluginAction('update', selectedPluginForAction)
        "
        class="block w-full px-4 py-2 text-left text-sm text-gray-700 hover:bg-gray-100"
      >
        更新
      </button>
      <hr class="border-gray-100" />
      <button
        @click="
          selectedPluginForAction &&
          handlePluginAction('uninstall', selectedPluginForAction)
        "
        class="block w-full px-4 py-2 text-left text-sm text-red-600 hover:bg-red-50"
      >
        卸载
      </button>
    </div>

    <!-- 点击遮罩关闭下拉菜单 -->
    <div
      v-if="activeDropdown"
      @click="activeDropdown = null"
      class="fixed inset-0 z-[9998]"
    ></div>
  </div>
</template>

<style scoped>
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 响应式设计 */
@media (max-width: 768px) {
  .plugin-manager {
    padding: 16px;
  }

  .flex.flex-wrap.gap-4 {
    flex-direction: column;
    gap: 16px;
  }

  .grid {
    grid-template-columns: 1fr;
  }

  .tab-nav {
    flex-direction: column;
  }
}

@media (max-width: 1024px) {
  .grid.grid-cols-1.md\:grid-cols-2.lg\:grid-cols-3.xl\:grid-cols-4 {
    grid-template-columns: repeat(2, 1fr);
  }
}

.plugin-manager {
  min-height: 100vh;
  padding: 24px;
}

/* 统计卡片样式 */
.stats-card {
  padding: 20px;
  color: white;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  box-shadow: 0 4px 15px rgb(0 0 0 / 10%);
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease;
}

.stats-card:hover {
  box-shadow: 0 8px 25px rgb(0 0 0 / 15%);
  transform: translateY(-2px);
}

.stats-card.total-plugins {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.stats-card.active-plugins {
  background: linear-gradient(135deg, #56ab2f 0%, #a8e6cf 100%);
}

.stats-card.inactive-plugins {
  background: linear-gradient(135deg, #ff6b6b 0%, #ffa8a8 100%);
}

.stats-content {
  display: flex;
  gap: 16px;
  align-items: center;
}

.stats-icon {
  font-size: 24px;
  opacity: 0.9;
}

.stats-info h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 500;
  opacity: 0.9;
}

.stats-number {
  margin: 4px 0 0;
  font-size: 28px;
  font-weight: 700;
}

/* 标签页样式 */
.library-tabs {
  margin-top: 24px;
}

.tab-nav {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding-bottom: 12px;
  margin-bottom: 24px;
  border-bottom: 1px solid #e5e7eb;
}

.tab-button {
  display: flex;
  gap: 8px;
  align-items: center;
  padding: 8px 16px;
  font-weight: 500;
  cursor: pointer;
  border: 1px solid;
  border-radius: 6px;
  transition: all 0.2s ease;
}

.tab-button:hover {
  background-color: rgb(0 0 0 / 5%);
}

.tab-button.active {
  background-color: rgb(0 0 0 / 10%);
}

.tab-count {
  padding: 2px 6px;
  font-size: 12px;
  font-weight: 600;
  background: rgb(255 255 255 / 20%);
  border-radius: 10px;
}

.tab-button.active .tab-count {
  background: rgb(255 255 255 / 30%);
}

/* 库统计样式 */
.library-stat {
  min-width: 100px;
  padding: 16px;
  text-align: center;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

.stat-title {
  margin-bottom: 4px;
  font-size: 12px;
  font-weight: 500;
}

.stat-value {
  font-size: 18px;
  font-weight: 600;
}

/* 插件卡片样式 */
.plugin-card {
  position: relative;
  display: flex;
  flex-direction: column;
  height: 320px;
  padding: 20px;
  overflow: hidden;
  background-color: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  transition: all 0.3s ease;
}

.plugin-card:hover {
  background-color: #f8fafc;
  box-shadow: 0 8px 30px rgb(0 0 0 / 12%);
  transform: translateY(-2px);
}

.plugin-card.border-green-200 {
  background-color: #f0f9ff;
  border-color: #d9f7be;
}

.plugin-card.border-gray-200 {
  background-color: #fafafa;
  border-color: #f0f0f0;
}

.plugin-header {
  padding-bottom: 12px;
  margin-bottom: 12px;
  border-bottom: 1px solid #f0f0f0;
}

.plugin-info {
  padding: 12px;
  background-color: rgb(248 250 252 / 0.5);
  border-radius: 6px;
}

.plugin-actions {
  margin-top: auto;
}

.line-clamp-2 {
  display: -webkit-box;
  overflow: hidden;
  text-overflow: ellipsis;
  -webkit-line-clamp: 2;
  line-clamp: 2;
  -webkit-box-orient: vertical;
}

/* 状态指示器 */
.plugin-card.border-green-200::before {
  position: absolute;
  top: 0;
  left: 0;
  width: 4px;
  height: 100%;
  content: '';
  background: #52c41a;
  border-radius: 0 4px 4px 0;
}

.plugin-card.border-gray-200::before {
  position: absolute;
  top: 0;
  left: 0;
  width: 4px;
  height: 100%;
  content: '';
  background: #d9d9d9;
  border-radius: 0 4px 4px 0;
}

/* Switch 样式 */
.switch {
  position: relative;
  display: inline-block;
  width: 50px;
  height: 24px;
}

.switch input {
  width: 0;
  height: 0;
  opacity: 0;
}

.slider {
  position: absolute;
  inset: 0;
  cursor: pointer;
  border-radius: 24px;
  transition: 0.4s;
}

.slider::before {
  position: absolute;
  bottom: 3px;
  left: 3px;
  width: 18px;
  height: 18px;
  content: '';
  background-color: var(--ant-color-bg-base, white);
  border-radius: 50%;
  transition: 0.4s;
}

input:checked + .slider {
  background-color: #52c41a;
}

input:checked + .slider::before {
  transform: translateX(26px);
}

/* 详情面板样式 */
.plugin-detail {
  padding: 16px 0;
}

.detail-item {
  padding: 12px 0;
  border-bottom: 1px solid #f0f0f0;
}

.detail-item:last-child {
  border-bottom: none;
}

.detail-label {
  display: block;
  margin-bottom: 4px;
  font-size: 14px;
  font-weight: 600;
}

.detail-value {
  font-size: 14px;
  word-break: break-word;
}

/* 骨架屏样式 */
.plugin-skeleton {
  height: 320px;
  padding: 20px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
}

/* 动画效果 */
.plugin-card {
  animation: fadeInUp 0.3s ease-out;
}

/* 下拉菜单样式 */
.relative {
  position: relative;
}
</style>
