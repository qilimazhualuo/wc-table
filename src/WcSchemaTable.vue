<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import { message } from 'antdv-next'
import type {
    DateSearchPrecision,
    PageTableFilter,
    PageTableLoadData,
    PageTableRow,
} from 'wc-core'
import type { PageField } from 'wc-core'
import { isShowInTable } from 'wc-core'
import {
    buildTableFilters,
    getDatePickerConfig,
    isSearchEnabled,
    resolveDatePrecision,
} from 'wc-core'

const props = withDefaults(defineProps<{
    tableName?: string | null
    loadData: PageTableLoadData
    fill?: boolean
}>(), {
    tableName: null,
    fill: true,
})

const emit = defineEmits<{
    edit: [featureId: string, record: PageTableRow]
    delete: [featureId: string, record: PageTableRow]
    create: []
}>()

const tableContentRef = ref<HTMLElement | null>(null)
const loading = ref(false)
const page = ref(1)
const pageSize = ref(20)
const total = ref(0)
const fields = ref<PageField[]>([])
const rows = ref<PageTableRow[]>([])
const bodyScrollHeight = ref<number>()
const searchValues = reactive<Record<string, unknown>>({})
const appliedFilters = ref<PageTableFilter[]>([])
let resizeObserver: ResizeObserver | undefined

const searchFields = computed(() => fields.value.filter((fieldItem) => isSearchEnabled(fieldItem)))

const booleanSearchOptions = [
    { label: '是', value: true },
    { label: '否', value: false },
]

const tableColumns = computed(() => {
    const visibleFields = fields.value.filter((fieldItem) => isShowInTable(fieldItem))
    const sourceFields = visibleFields.length
        ? visibleFields
        : fields.value.filter((fieldItem) => fieldItem.lockReason !== 'geometry' && fieldItem.inputType !== 'geom')
    const dataColumns = sourceFields.map((fieldItem) => ({
        key: fieldItem.field,
        title: fieldItem.title || fieldItem.field,
        ellipsis: true,
        width: 160,
    }))
    if (!dataColumns.length && rows.value[0]) {
        Object.keys(rows.value[0].attributes || {}).forEach((fieldName) => {
            dataColumns.push({
                key: fieldName,
                title: fieldName,
                ellipsis: true,
                width: 160,
            })
        })
    }
    return [
        ...dataColumns,
        {
            key: 'action',
            title: '操作',
            width: 140,
            align: 'center' as const,
            fixed: 'right' as const,
        },
    ]
})

const tableScroll = computed(() => {
    if (!bodyScrollHeight.value || Number.isNaN(bodyScrollHeight.value) || bodyScrollHeight.value <= 0) {
        return { x: 'max-content' }
    }
    return { x: 'max-content', y: bodyScrollHeight.value }
})

const formatCellValue = (rawValue: unknown) => {
    if (rawValue == null || rawValue === '') {
        return '-'
    }
    if (typeof rawValue === 'object') {
        return JSON.stringify(rawValue)
    }
    return String(rawValue)
}

const fieldPrecision = (fieldItem: PageField): DateSearchPrecision =>
    resolveDatePrecision(fieldItem)

const datePickerProps = (fieldItem: PageField) => {
    const pickerConfig = getDatePickerConfig(fieldPrecision(fieldItem))
    return {
        picker: pickerConfig.picker,
        showTime: pickerConfig.showTime,
        format: pickerConfig.format,
        valueFormat: pickerConfig.format,
        allowClear: true,
        style: { width: '100%' },
    }
}

const clearSearchValues = () => {
    Object.keys(searchValues).forEach((fieldName) => {
        delete searchValues[fieldName]
    })
}

const setTableHeight = () => {
    const tableContent = tableContentRef.value
    if (!tableContent) {
        return
    }
    const headerElement = tableContent.getElementsByClassName('ant-table-thead')[0] as HTMLElement | undefined
    const paginationElement = tableContent.getElementsByClassName('ant-pagination')[0] as HTMLElement | undefined
    let extraHeight = headerElement?.clientHeight ?? 0
    extraHeight += paginationElement?.clientHeight ?? 0
    extraHeight += 12
    const calculatedHeight = tableContent.offsetHeight - extraHeight
    bodyScrollHeight.value = calculatedHeight > 0 ? calculatedHeight : undefined
}

const loadRows = async () => {
    if (!props.tableName) {
        rows.value = []
        total.value = 0
        fields.value = []
        await nextTick()
        setTableHeight()
        return
    }
    loading.value = true
    try {
        const result = await props.loadData({
            tableName: props.tableName,
            page: page.value,
            pageSize: pageSize.value,
            filters: appliedFilters.value,
        })
        fields.value = result.fields || []
        total.value = result.total || 0
        rows.value = result.items || []
    } catch (error) {
        rows.value = []
        total.value = 0
        message.error(error instanceof Error ? error.message : '加载表格数据失败')
    } finally {
        loading.value = false
        await nextTick()
        setTableHeight()
    }
}

const handleSearch = () => {
    appliedFilters.value = buildTableFilters(fields.value, searchValues)
    page.value = 1
    void loadRows()
}

const handleResetSearch = () => {
    clearSearchValues()
    appliedFilters.value = []
    page.value = 1
    void loadRows()
}

const handlePageChange = (nextPage: number, nextPageSize: number) => {
    page.value = nextPage
    pageSize.value = nextPageSize
    void loadRows()
}

const setRangeValue = (fieldName: string, rangeIndex: 0 | 1, nextValue: unknown) => {
    const currentRange = Array.isArray(searchValues[fieldName])
        ? [...(searchValues[fieldName] as unknown[])]
        : [null, null]
    currentRange[rangeIndex] = nextValue ?? null
    searchValues[fieldName] = currentRange
}

const rangeValueAt = (fieldName: string, rangeIndex: 0 | 1): number | null => {
    const rawValue = searchValues[fieldName]
    if (!Array.isArray(rawValue)) {
        return null
    }
    const item = rawValue[rangeIndex]
    return typeof item === 'number' ? item : null
}

watch(
    () => props.tableName,
    () => {
        page.value = 1
        clearSearchValues()
        appliedFilters.value = []
        void loadRows()
    },
    { immediate: true },
)

onMounted(() => {
    setTableHeight()
    const tableContent = tableContentRef.value
    if (!tableContent) {
        return
    }
    resizeObserver = new ResizeObserver(() => {
        setTableHeight()
    })
    resizeObserver.observe(tableContent)
})

onBeforeUnmount(() => {
    resizeObserver?.disconnect()
})

defineExpose({
    reload: () => {
        void loadRows()
    },
})
</script>

<template>
    <div
        class="wc-page-schema-table"
        :class="{ 'wc-page-schema-table--fill': fill }"
    >
        <div
            v-if="searchFields.length"
            class="wc-page-schema-table__search"
        >
            <div class="wc-page-schema-table__search-fields">
                <div
                    v-for="fieldItem in searchFields"
                    :key="fieldItem.field"
                    class="wc-page-schema-table__search-item"
                >
                    <span class="wc-page-schema-table__search-label">
                        {{ fieldItem.title || fieldItem.field }}
                    </span>
                    <a-input
                        v-if="fieldItem.searchMode === 'text'"
                        :value="(searchValues[fieldItem.field] as string) || ''"
                        allow-clear
                        placeholder="请输入"
                        @update:value="(value: string | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <a-input-number
                        v-else-if="fieldItem.searchMode === 'number'"
                        :value="searchValues[fieldItem.field] as number | null"
                        allow-clear
                        placeholder="请输入"
                        style="width: 100%"
                        @update:value="(value: number | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <div
                        v-else-if="fieldItem.searchMode === 'numberRange'"
                        class="wc-page-schema-table__search-range"
                    >
                        <a-input-number
                            :value="rangeValueAt(fieldItem.field, 0)"
                            allow-clear
                            placeholder="最小值"
                            style="width: 100%"
                            @update:value="(value: number | null) => setRangeValue(fieldItem.field, 0, value)"
                        />
                        <span class="wc-page-schema-table__search-sep">~</span>
                        <a-input-number
                            :value="rangeValueAt(fieldItem.field, 1)"
                            allow-clear
                            placeholder="最大值"
                            style="width: 100%"
                            @update:value="(value: number | null) => setRangeValue(fieldItem.field, 1, value)"
                        />
                    </div>
                    <a-date-picker
                        v-else-if="fieldItem.searchMode === 'date' || fieldItem.searchMode === 'datetime'"
                        :value="(searchValues[fieldItem.field] as string) || undefined"
                        v-bind="datePickerProps(fieldItem)"
                        @update:value="(value: string | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <a-range-picker
                        v-else-if="fieldItem.searchMode === 'dateRange' || fieldItem.searchMode === 'datetimeRange'"
                        :value="(searchValues[fieldItem.field] as [string, string] | undefined)"
                        v-bind="datePickerProps(fieldItem)"
                        @update:value="(value: [string, string] | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <a-select
                        v-else-if="fieldItem.searchMode === 'select'"
                        :value="searchValues[fieldItem.field]"
                        :options="fieldItem.options || []"
                        allow-clear
                        show-search
                        placeholder="请选择"
                        style="width: 100%"
                        @update:value="(value: string | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <a-select
                        v-else-if="fieldItem.searchMode === 'selectMultiple'"
                        :value="(searchValues[fieldItem.field] as unknown[]) || []"
                        :options="fieldItem.options || []"
                        allow-clear
                        show-search
                        mode="multiple"
                        placeholder="请选择"
                        style="width: 100%"
                        @update:value="(value: unknown[] | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                    <a-select
                        v-else-if="fieldItem.searchMode === 'boolean'"
                        :value="searchValues[fieldItem.field]"
                        :options="booleanSearchOptions"
                        allow-clear
                        placeholder="请选择"
                        style="width: 100%"
                        @update:value="(value: boolean | null) => { searchValues[fieldItem.field] = value || null }"
                    />
                </div>
            </div>
            <div class="wc-page-schema-table__search-actions">
                <a-button type="primary" @click="handleSearch">查询</a-button>
                <a-button @click="handleResetSearch">重置</a-button>
                <a-button type="primary" @click="emit('create')">新增</a-button>
            </div>
        </div>
        <div ref="tableContentRef" class="wc-page-schema-table__body">
            <a-table
                :columns="tableColumns"
                :data-source="rows"
                :loading="loading"
                :pagination="false"
                :scroll="tableScroll"
                row-key="id"
                size="middle"
            >
                <template #bodyCell="{ column, record }">
                    <template v-if="column.key === 'action'">
                        <a-space>
                            <a-button
                                type="link"
                                size="small"
                                @click="emit('edit', record.id, record)"
                            >
                                编辑
                            </a-button>
                            <a-popconfirm
                                title="确定删除这条数据？"
                                ok-text="确定"
                                cancel-text="取消"
                                @confirm="emit('delete', record.id, record)"
                            >
                                <a-button
                                    type="link"
                                    size="small"
                                    danger
                                >
                                    删除
                                </a-button>
                            </a-popconfirm>
                        </a-space>
                    </template>
                    <template v-else>
                        {{ formatCellValue(record.attributes?.[column.key as string]) }}
                    </template>
                </template>
                <template #emptyText>
                    <a-empty :description="tableName ? '暂无数据' : '请选择左侧数据表'" />
                </template>
            </a-table>
            <a-pagination
                v-model:current="page"
                v-model:page-size="pageSize"
                :total="total"
                show-size-changer
                show-quick-jumper
                :show-total="(count: number) => `共 ${count} 条`"
                @change="handlePageChange"
            />
        </div>
    </div>
</template>

<style scoped lang="less">
.wc-page-schema-table {
    display: flex;
    flex-direction: column;
    height: 100%;
    min-height: 0;
    overflow: hidden;
    box-sizing: border-box;
    padding: 16px;
    background: var(--theme-color-bg-container, #fff);

    &--fill {
        position: absolute;
        inset: 0;
    }

    &__search {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 12px;
        margin-bottom: 12px;
        flex-shrink: 0;
    }

    &__search-fields {
        display: flex;
        flex-wrap: wrap;
        gap: 12px 16px;
        flex: 1;
        min-width: 0;
        align-items: center;
    }

    &__search-item {
        display: flex;
        flex-direction: row;
        align-items: center;
        gap: 8px;
        width: auto;
        min-width: 240px;
        max-width: 360px;
        flex: 1 1 240px;

        :deep(.ant-input),
        :deep(.ant-input-number),
        :deep(.ant-picker),
        :deep(.ant-select) {
            flex: 1;
            min-width: 0;
        }
    }

    &__search-label {
        font-size: 13px;
        color: var(--theme-color-text, rgba(0, 0, 0, 0.88));
        line-height: 32px;
        flex-shrink: 0;
        white-space: nowrap;
    }

    &__search-range {
        display: flex;
        align-items: center;
        gap: 6px;
        flex: 1;
        min-width: 0;
    }

    &__search-sep {
        color: var(--theme-color-text-secondary, rgba(0, 0, 0, 0.45));
        flex-shrink: 0;
    }

    &__search-actions {
        display: flex;
        gap: 8px;
        flex-shrink: 0;
        align-items: center;
    }

    &__body {
        height: 2px;
        flex: 1 1 auto;
        min-height: 0;
        display: flex;
        flex-direction: column;
        overflow: hidden;

        :deep(.ant-pagination) {
            flex-shrink: 0;
            margin-top: 12px;
            align-self: flex-end;
        }
    }
}
</style>
