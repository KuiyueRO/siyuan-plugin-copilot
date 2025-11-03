<script lang="ts">
    import { createEventDispatcher } from 'svelte';
    import { fetchModels } from '../ai-chat';
    import { pushMsg, pushErrMsg } from '../api';
    import type { ProviderConfig, ModelConfig } from '../defaultSettings';

    export let providerId: string;
    export let providerName: string;
    export let defaultApiUrl: string = ''; // 默认 API 地址
    export let config: ProviderConfig;

    const dispatch = createEventDispatcher();

    let isLoadingModels = false;
    let searchQuery = '';
    let availableModels: { id: string; name: string }[] = [];
    let showModelSearchModal = false;
    let showAddModelModal = false;
    let manualModelId = '';
    let manualModelName = '';

    // 获取模型列表
    async function loadModels() {
        if (!config.apiKey) {
            pushErrMsg('请先设置 API Key');
            return;
        }

        isLoadingModels = true;
        try {
            const models = await fetchModels(providerId, config.apiKey, config.customApiUrl);
            // 按模型ID升序排序
            availableModels = models
                .map(m => ({ id: m.id, name: m.name }))
                .sort((a, b) => a.id.localeCompare(b.id));
            showModelSearchModal = true;
            searchQuery = '';
            pushMsg(`成功获取 ${models.length} 个模型`);
        } catch (error) {
            pushErrMsg(`获取模型失败: ${error.message}`);
            console.error('Load models error:', error);
        } finally {
            isLoadingModels = false;
        }
    }

    // 打开模型搜索弹窗
    function openModelSearchModal() {
        if (!config.apiKey) {
            pushErrMsg('请先设置 API Key');
            return;
        }
        loadModels();
    }

    // 关闭模型搜索弹窗
    function closeModelSearchModal() {
        showModelSearchModal = false;
        searchQuery = '';
        availableModels = [];
    }

    // 添加模型
    function addModel(modelId: string, modelName: string) {
        // 检查是否已存在
        if (config.models.some(m => m.id === modelId)) {
            pushErrMsg('该模型已添加');
            return;
        }

        const newModel: ModelConfig = {
            id: modelId,
            name: modelName,
            temperature: 0.7,
            maxTokens: -1,
        };

        config.models = [...config.models, newModel];
        dispatch('change');
        pushMsg(`已添加模型: ${modelName}`);
        searchQuery = '';
    }

    // 手动添加模型
    function addManualModel() {
        if (!manualModelId.trim()) {
            pushErrMsg('请输入模型ID');
            return;
        }

        const modelName = manualModelName.trim() || manualModelId.trim();
        addModel(manualModelId.trim(), modelName);

        manualModelId = '';
        manualModelName = '';
        showAddModelModal = false;
    }

    // 打开添加模型弹窗
    function openAddModelModal() {
        showAddModelModal = true;
    }

    // 关闭添加模型弹窗
    function closeAddModelModal() {
        showAddModelModal = false;
        manualModelId = '';
        manualModelName = '';
    }

    // 删除模型
    function removeModel(modelId: string) {
        config.models = config.models.filter(m => m.id !== modelId);
        dispatch('change');
        pushMsg('已删除模型');
    }

    // 更新模型配置
    function updateModel(modelId: string, field: keyof ModelConfig, value: any) {
        const model = config.models.find(m => m.id === modelId);
        if (model) {
            (model as any)[field] = value;
            config.models = [...config.models];
            dispatch('change');
        }
    }

    // 过滤并排序模型
    $: filteredModels = availableModels
        .filter(
            m =>
                m.id.toLowerCase().includes(searchQuery.toLowerCase()) ||
                m.name.toLowerCase().includes(searchQuery.toLowerCase())
        )
        .sort((a, b) => a.id.localeCompare(b.id));
</script>

<div class="provider-config">
    <div class="provider-config__header">
        <h4>{providerName}</h4>
    </div>

    <div class="provider-config__section">
        <div class="b3-label">
            <div class="b3-label__text">API Key</div>
            <input
                class="b3-text-field fn__flex-1"
                type="password"
                bind:value={config.apiKey}
                on:change={() => dispatch('change')}
                placeholder="输入 API Key"
            />
        </div>

        <div class="b3-label">
            <div class="b3-label__text">
                API 地址
                {#if defaultApiUrl}
                    <span class="label-hint">（默认: {defaultApiUrl}）</span>
                {/if}
            </div>
            <input
                class="b3-text-field fn__flex-1"
                type="text"
                bind:value={config.customApiUrl}
                on:change={() => dispatch('change')}
                placeholder={defaultApiUrl || '输入自定义 API 地址'}
            />
            <div class="b3-label__text label-description">
                留空使用默认地址。以 / 结尾忽略 v1 版本，以 # 结尾强制使用输入地址
            </div>
        </div>

        <div class="b3-label">
            <div class="b3-label__text">模型管理</div>
            <div class="provider-config__model-buttons">
                <button
                    class="b3-button b3-button--outline"
                    on:click={openModelSearchModal}
                    disabled={isLoadingModels || !config.apiKey}
                >
                    {isLoadingModels ? '获取中...' : '搜索并添加模型'}
                </button>
                <button class="b3-button b3-button--outline" on:click={openAddModelModal}>
                    + 手动添加模型
                </button>
            </div>
        </div>
    </div>

    <!-- 模型搜索弹窗 -->
    {#if showModelSearchModal}
        <div class="modal-overlay" on:click={closeModelSearchModal}>
            <div class="modal-content modal-content--large" on:click|stopPropagation>
                <div class="modal-header">
                    <h4>搜索并添加模型</h4>
                    <button class="modal-close" on:click={closeModelSearchModal}>
                        <svg class="b3-button__icon" style="width: 13px;height: 13px"><use xlink:href="#iconClose"></use></svg>
                    </button>
                </div>
                <div class="modal-body">
                    {#if availableModels.length > 0}
                        <div class="b3-label">
                            <div class="b3-label__text">搜索模型</div>
                            <input
                                class="b3-text-field fn__flex-1"
                                type="text"
                                bind:value={searchQuery}
                                placeholder="搜索模型..."
                            />
                        </div>

                        <div class="model-search-results">
                            {#each filteredModels.slice(0, 20) as model}
                                <div class="model-search-item">
                                    <div class="model-search-item__info">
                                        <span class="model-search-item__name">{model.name}</span>
                                        <span class="model-search-item__id">{model.id}</span>
                                    </div>
                                    <button
                                        class="b3-button b3-button--text"
                                        on:click={() => addModel(model.id, model.name)}
                                        disabled={config.models.some(m => m.id === model.id)}
                                    >
                                        {config.models.some(m => m.id === model.id)
                                            ? '已添加'
                                            : '添加'}
                                    </button>
                                </div>
                            {/each}
                            {#if filteredModels.length === 0}
                                <div class="model-search-empty">没有找到匹配的模型</div>
                            {/if}
                        </div>
                    {:else}
                        <div class="loading-models">
                            <div class="loading-spinner"></div>
                            <span>正在获取模型列表...</span>
                        </div>
                    {/if}
                </div>
                <div class="modal-footer">
                    <button class="b3-button b3-button--text" on:click={closeModelSearchModal}>
                        关闭
                    </button>
                </div>
            </div>
        </div>
    {/if}

    {#if config.models.length > 0}
        <div class="provider-config__models">
            <h5>已添加的模型</h5>
            {#each config.models as model}
                <div class="model-item">
                    <div class="model-item__header">
                        <span class="model-item__name">{model.name}</span>
                        <button
                            class="b3-button b3-button--text b3-button--error"
                            on:click={() => removeModel(model.id)}
                            title="删除"
                        >
                            <svg class="b3-button__icon">
                                <use xlink:href="#iconTrashcan"></use>
                            </svg>
                        </button>
                    </div>
                    <div class="model-item__config">
                        <div class="model-config-item">
                            <span>温度 (Temperature): {model.temperature}</span>
                            <input
                                type="range"
                                min="0"
                                max="2"
                                step="0.1"
                                bind:value={model.temperature}
                                on:change={() =>
                                    updateModel(model.id, 'temperature', model.temperature)}
                            />
                        </div>
                        <div class="model-config-item">
                            <span>最大 Tokens (-1表示不限制)</span>
                            <input
                                class="b3-text-field"
                                type="number"
                                min="-1"
                                max="128000"
                                bind:value={model.maxTokens}
                                on:change={() =>
                                    updateModel(model.id, 'maxTokens', model.maxTokens)}
                            />
                        </div>
                        <div class="model-config-item">
                            <span>模型能力</span>
                            <div class="model-capabilities">
                                <label class="b3-label b3-label--noborder">
                                    <input
                                        type="checkbox"
                                        class="b3-switch"
                                        checked={model.capabilities?.thinking || false}
                                        on:change={(e) => {
                                            if (!model.capabilities) model.capabilities = {};
                                            model.capabilities.thinking = e.currentTarget.checked;
                                            updateModel(model.id, 'capabilities', model.capabilities);
                                        }}
                                    />
                                    <span class="capability-label">思考模式 (Thinking)</span>
                                </label>
                                <label class="b3-label b3-label--noborder">
                                    <input
                                        type="checkbox"
                                        class="b3-switch"
                                        checked={model.capabilities?.vision || false}
                                        on:change={(e) => {
                                            if (!model.capabilities) model.capabilities = {};
                                            model.capabilities.vision = e.currentTarget.checked;
                                            updateModel(model.id, 'capabilities', model.capabilities);
                                        }}
                                    />
                                    <span class="capability-label">视觉 (Vision)</span>
                                </label>
                            </div>
                        </div>
                    </div>
                </div>
            {/each}
        </div>
    {/if}

    <!-- 手动添加模型弹窗 -->
    {#if showAddModelModal}
        <div class="modal-overlay" on:click={closeAddModelModal}>
            <div class="modal-content" on:click|stopPropagation>
                <div class="modal-header">
                    <h4>手动添加模型</h4>
                    <button class="modal-close" on:click={closeAddModelModal}>
                        <svg class="b3-button__icon" style="width: 13px;height: 13px"><use xlink:href="#iconClose"></use></svg>
                    </button>
                </div>
                <div class="modal-body">
                    <div class="b3-label">
                        <div class="b3-label__text">模型ID *</div>
                        <input
                            class="b3-text-field fn__flex-1"
                            type="text"
                            bind:value={manualModelId}
                            placeholder="例如: gpt-4, gemini-pro"
                            on:keydown={e => e.key === 'Enter' && addManualModel()}
                        />
                    </div>
                    <div class="b3-label">
                        <div class="b3-label__text">模型名称（可选）</div>
                        <input
                            class="b3-text-field fn__flex-1"
                            type="text"
                            bind:value={manualModelName}
                            placeholder="不填则使用模型ID"
                            on:keydown={e => e.key === 'Enter' && addManualModel()}
                        />
                    </div>
                </div>
                <div class="modal-footer">
                    <button class="b3-button b3-button--text" on:click={closeAddModelModal}>
                        取消
                    </button>
                    <button
                        class="b3-button b3-button--outline"
                        on:click={addManualModel}
                        disabled={!manualModelId.trim()}
                    >
                        添加模型
                    </button>
                </div>
            </div>
        </div>
    {/if}
</div>

<style lang="scss">
    .provider-config {
        padding: 16px;
        background: var(--b3-theme-surface);
        border-radius: 6px;
        margin-bottom: 16px;
    }

    .provider-config__header {
        margin-bottom: 16px;

        h4 {
            margin: 0;
            font-size: 16px;
            font-weight: 600;
            color: var(--b3-theme-on-background);
        }
    }

    .provider-config__section {
        display: flex;
        flex-direction: column;
        gap: 12px;
    }

    .provider-config__model-buttons {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;

        button {
            flex: 1;
            min-width: 0;
        }
    }

    .label-hint {
        font-size: 11px;
        color: var(--b3-theme-on-surface-light);
        font-weight: normal;
        margin-left: 4px;
    }

    .label-description {
        font-size: 11px;
        color: var(--b3-theme-on-surface-light);
        margin-top: 4px;
        line-height: 1.4;
    }

    .model-search-results {
        max-height: 300px;
        overflow-y: auto;
        margin-top: 8px;
        border: 1px solid var(--b3-border-color);
        border-radius: 4px;
    }

    .model-search-item {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 8px 12px;
        border-bottom: 1px solid var(--b3-border-color);

        &:last-child {
            border-bottom: none;
        }

        &:hover {
            background: var(--b3-theme-background);
        }
    }

    .model-search-item__name {
        font-size: 13px;
        color: var(--b3-theme-on-surface);
    }

    .provider-config__models {
        margin-top: 16px;
        padding-top: 16px;
        border-top: 1px solid var(--b3-border-color);

        h5 {
            margin: 0 0 12px 0;
            font-size: 14px;
            font-weight: 600;
            color: var(--b3-theme-on-surface);
        }
    }

    .model-item {
        background: var(--b3-theme-background);
        border: 1px solid var(--b3-border-color);
        border-radius: 6px;
        padding: 12px;
        margin-bottom: 12px;
    }

    .model-item__header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        margin-bottom: 12px;
    }

    .model-item__name {
        font-size: 14px;
        font-weight: 600;
        color: var(--b3-theme-on-background);
    }

    .model-item__config {
        display: flex;
        flex-direction: column;
        gap: 12px;
    }

    .model-config-item {
        display: flex;
        flex-direction: column;
        gap: 4px;

        span {
            font-size: 12px;
            color: var(--b3-theme-on-surface);
        }

        input[type='range'] {
            width: 100%;
        }

        input[type='number'] {
            width: 100%;
        }
    }

    .model-capabilities {
        display: flex;
        flex-direction: column;
        gap: 8px;
        margin-top: 4px;

        .b3-label {
            display: flex;
            align-items: center;
            gap: 8px;
            margin: 0;
        }

        .capability-label {
            font-size: 12px;
            color: var(--b3-theme-on-surface);
        }
    }

    // 弹窗样式
    .modal-overlay {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: rgba(0, 0, 0, 0.5);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 1000;
    }

    .modal-content {
        background: var(--b3-theme-background);
        border-radius: 8px;
        box-shadow: var(--b3-dialog-shadow);
        min-width: 400px;
        max-width: 600px;
        animation: modalShow 0.2s ease-out;
    }

    .modal-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 16px 20px;
        border-bottom: 1px solid var(--b3-border-color);

        h4 {
            margin: 0;
            font-size: 16px;
            font-weight: 600;
            color: var(--b3-theme-on-background);
        }
    }

    .modal-close {
        background: none;
        border: none;
        padding: 4px;
        cursor: pointer;
        color: var(--b3-theme-on-surface-light);
        border-radius: 4px;

        &:hover {
            background: var(--b3-theme-surface);
            color: var(--b3-theme-on-background);
        }
    }

    .modal-body {
        padding: 20px;
        display: flex;
        flex-direction: column;
        gap: 16px;
    }

    .modal-footer {
        display: flex;
        align-items: center;
        justify-content: flex-end;
        gap: 8px;
        padding: 16px 20px;
        border-top: 1px solid var(--b3-border-color);
    }

    @keyframes modalShow {
        from {
            opacity: 0;
            transform: scale(0.95);
        }
        to {
            opacity: 1;
            transform: scale(1);
        }
    }

    // 模型搜索弹窗专用样式
    .modal-content--large {
        min-width: 600px;
        max-width: 800px;
        max-height: 70vh;
    }

    .model-search-results {
        max-height: 400px;
        overflow-y: auto;
        border: 1px solid var(--b3-border-color);
        border-radius: 4px;
        background: var(--b3-theme-background);
    }

    .model-search-item {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 12px;
        border-bottom: 1px solid var(--b3-border-color);

        &:last-child {
            border-bottom: none;
        }

        &:hover {
            background: var(--b3-theme-surface);
        }
    }

    .model-search-item__info {
        display: flex;
        flex-direction: column;
        gap: 2px;
        flex: 1;
    }

    .model-search-item__name {
        font-size: 14px;
        font-weight: 500;
        color: var(--b3-theme-on-background);
    }

    .model-search-item__id {
        font-size: 12px;
        color: var(--b3-theme-on-surface-light);
    }

    .model-search-empty {
        padding: 40px;
        text-align: center;
        color: var(--b3-theme-on-surface-light);
        font-size: 14px;
    }

    .loading-models {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 12px;
        padding: 40px;
        color: var(--b3-theme-on-surface);
    }

    .loading-spinner {
        width: 20px;
        height: 20px;
        border: 2px solid var(--b3-theme-border);
        border-top-color: var(--b3-theme-primary);
        border-radius: 50%;
        animation: spin 1s linear infinite;
    }

    @keyframes spin {
        to {
            transform: rotate(360deg);
        }
    }
</style>
