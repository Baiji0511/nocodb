<script setup lang="ts">
import type { ColumnReqType, ColumnType, FormulaType, LinkToAnotherRecordType, LookupType, RollupType } from 'nocodb-sdk'
import { UITypes, UITypesName, substituteColumnIdWithAliasInFormula } from 'nocodb-sdk'
import {
  ColumnInj,
  IsFormInj,
  MetaInj,
  computed,
  inject,
  isBt,
  isFormula,
  isHm,
  isLookup,
  isMm,
  isRollup,
  isVirtualColRequired,
  provide,
  ref,
  toRef,
  useI18n,
  useMetas,
  useRoles,
} from '#imports'

const props = defineProps<{ column: ColumnType; hideMenu?: boolean; required?: boolean | number; hideIcon?: boolean }>()

const { t } = useI18n()

const column = toRef(props, 'column')

const hideMenu = toRef(props, 'hideMenu')

const { isMobileMode } = useGlobal()

const editColumnDropdown = ref(false)

const isDropDownOpen = ref(false)

const isLocked = inject(IsLockedInj, ref(false))

provide(ColumnInj, column)

const { metas } = useMetas()

const { isUIAllowed } = useRoles()

const meta = inject(MetaInj, ref())

const isGrid = inject(IsGridInj, ref(false))

const isForm = inject(IsFormInj, ref(false))

const isExpandedForm = inject(IsExpandedFormOpenInj, ref(false))

const colOptions = computed(() => column.value?.colOptions)

const tableTile = computed(() => meta?.value?.title)

const relationColumnOptions = computed<LinkToAnotherRecordType | null>(() => {
  if (isMm(column.value) || isHm(column.value) || isBt(column.value)) {
    return column.value?.colOptions as LinkToAnotherRecordType
  } else if ((column?.value?.colOptions as LookupType | RollupType)?.fk_relation_column_id) {
    return meta?.value?.columns?.find(
      (c) => c.id === (column?.value?.colOptions as LookupType | RollupType)?.fk_relation_column_id,
    )?.colOptions as LinkToAnotherRecordType
  }
  return null
})

const relatedTableMeta = computed(
  () =>
    relationColumnOptions.value?.fk_related_model_id && metas.value?.[relationColumnOptions.value?.fk_related_model_id as string],
)

const relatedTableTitle = computed(() => relatedTableMeta.value?.title)

const childColumn = computed(() => {
  if (relatedTableMeta.value?.columns) {
    if (isRollup(column.value)) {
      return relatedTableMeta.value?.columns.find(
        (c: ColumnType) => c.id === (colOptions.value as RollupType).fk_rollup_column_id,
      )
    }
    if (isLookup(column.value)) {
      return relatedTableMeta.value?.columns.find(
        (c: ColumnType) => c.id === (colOptions.value as LookupType).fk_lookup_column_id,
      )
    }
  }
  return ''
})

const tooltipMsg = computed(() => {
  if (!column.value) {
    return ''
  }

  if (isHm(column.value)) {
    return `'${tableTile.value}' ${t('labels.hasMany')} '${relatedTableTitle.value}'`
  } else if (isMm(column.value)) {
    return `'${tableTile.value}' & '${relatedTableTitle.value}' ${t('labels.manyToMany')}`
  } else if (isBt(column.value)) {
    return `'${column?.value?.title}' ${t('labels.belongsTo')} '${relatedTableTitle.value}'`
  } else if (isLookup(column.value)) {
    return `'${childColumn.value.title}' from '${relatedTableTitle.value}' (${childColumn.value.uidt})`
  } else if (isFormula(column.value)) {
    const formula = substituteColumnIdWithAliasInFormula(
      (column.value?.colOptions as FormulaType)?.formula,
      meta?.value?.columns as ColumnType[],
      (column.value?.colOptions as any)?.formula_raw,
    )
    return `Formula - ${formula}`
  } else if (isRollup(column.value)) {
    return `'${childColumn.value.title}' of '${relatedTableTitle.value}' (${childColumn.value.uidt})`
  }
  return column?.value?.title || ''
})

const columnOrder = ref<Pick<ColumnReqType, 'column_order'> | null>(null)

const columnTypeName = computed(() => {
  if (column.value.uidt === UITypes.LongText && parseProp(column?.value?.meta)?.richMode) {
    return UITypesName.RichText
  }
  return column.value.uidt ? UITypesName[column.value.uidt] : ''
})

const addField = async (payload: any) => {
  columnOrder.value = payload
  editColumnDropdown.value = true
}

const closeAddColumnDropdown = () => {
  columnOrder.value = null
  editColumnDropdown.value = false
}

const openHeaderMenu = () => {
  if (isLocked.value) return

  if (!isForm.value && !isExpandedForm.value && isUIAllowed('fieldEdit') && !isMobileMode.value) {
    editColumnDropdown.value = true
  }
}

const openDropDown = (e: Event) => {
  if (isLocked.value) return

  if (isForm.value || isExpandedForm.value || (!isUIAllowed('fieldEdit') && !isMobileMode.value)) return

  e.preventDefault()
  e.stopPropagation()

  isDropDownOpen.value = !isDropDownOpen.value
}
</script>

<template>
  <div
    class="flex items-center w-full h-full text-xs text-gray-500 font-weight-medium"
    @dblclick="openHeaderMenu"
    @click.right="openDropDown"
  >
    <template v-if="column && !props.hideIcon">
      <NcTooltip v-if="isGrid && !isExpandedForm" class="flex items-center" placement="bottom">
        <template #title> {{ columnTypeName }} </template>
        <LazySmartsheetHeaderVirtualCellIcon />
      </NcTooltip>
      <LazySmartsheetHeaderVirtualCellIcon v-else />
    </template>
    <NcTooltip placement="bottom" class="truncate name pl-1" show-on-truncate-only>
      <template #title>
        {{ tooltipMsg }}
      </template>
      <span :data-test-id="column.title">
        {{ column.title }}
      </span>
    </NcTooltip>

    <span v-if="isVirtualColRequired(column, meta?.columns || []) || required" class="text-red-500">&nbsp;*</span>

    <template v-if="!hideMenu">
      <div class="flex-1" />

      <LazySmartsheetHeaderMenu
        v-if="!isForm && isUIAllowed('fieldEdit') && !isExpandedForm"
        v-model:is-open="isDropDownOpen"
        :virtual="true"
        @add-column="addField"
        @edit="editColumnDropdown = true"
      />
    </template>

    <a-dropdown
      v-model:visible="editColumnDropdown"
      class="h-full"
      :trigger="['click']"
      placement="bottomRight"
      overlay-class-name="nc-dropdown-edit-column"
    >
      <div />
      <template #overlay>
        <SmartsheetColumnEditOrAddProvider
          v-if="editColumnDropdown"
          :column="columnOrder ? null : column"
          :column-position="columnOrder"
          class="w-full"
          @submit="closeAddColumnDropdown"
          @cancel="closeAddColumnDropdown"
          @click.stop
          @keydown.stop
        />
      </template>
    </a-dropdown>
  </div>
</template>

<style scoped>
.name {
  max-width: calc(100% - 40px);
  word-break: break-all;
}
</style>
