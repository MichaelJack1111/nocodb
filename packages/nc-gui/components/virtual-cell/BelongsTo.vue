<script setup lang="ts">
import type { ColumnType } from 'nocodb-sdk'
import type { Ref } from 'vue'
import {
  ActiveCellInj,
  CellValueInj,
  ColumnInj,
  IsFormInj,
  IsLockedInj,
  IsUnderLookupInj,
  ReadonlyInj,
  ReloadRowDataHookInj,
  RowInj,
  computed,
  createEventHook,
  inject,
  ref,
  useProvideLTARStore,
  useSelectedCellKeyupListener,
  useSmartsheetRowStoreOrThrow,
  useUIPermission,
} from '#imports'

const column = inject(ColumnInj)!

const reloadRowTrigger = inject(ReloadRowDataHookInj, createEventHook())

const cellValue = inject(CellValueInj, ref<any>(null))

const row = inject(RowInj)!

const active = inject(ActiveCellInj)!

const readOnly = inject(ReadonlyInj, ref(false))

const isForm = inject(IsFormInj, ref(false))

const isLocked = inject(IsLockedInj, ref(false))

const isUnderLookup = inject(IsUnderLookupInj, ref(false))

const { isUIAllowed } = useUIPermission()

const listItemsDlg = ref(false)

const { state, isNew, removeLTARRef } = useSmartsheetRowStoreOrThrow()

const { loadRelatedTableMeta, relatedTableDisplayValueProp, unlink } = useProvideLTARStore(
  column as Ref<Required<ColumnType>>,
  row,
  isNew,
  reloadRowTrigger.trigger,
)

await loadRelatedTableMeta()

const addIcon = computed(() => (cellValue?.value ? 'expand' : 'plus'))

const value = computed(() => {
  if (cellValue?.value) {
    return cellValue?.value
  } else if (isNew.value) {
    return state?.value?.[column?.value.title as string]
  }
  return null
})

const unlinkRef = async (rec: Record<string, any>) => {
  if (isNew.value) {
    await removeLTARRef(rec, column?.value as ColumnType)
  } else {
    await unlink(rec)
  }
}

useSelectedCellKeyupListener(active, (e: KeyboardEvent) => {
  switch (e.key) {
    case 'Enter':
      listItemsDlg.value = true
      e.stopPropagation()
      break
  }
})
</script>

<template>
  <div class="flex w-full chips-wrapper items-center" :class="{ active }">
    <div class="chips flex items-center flex-1">
      <template v-if="value && relatedTableDisplayValueProp">
        <VirtualCellComponentsItemChip :item="value" :value="value[relatedTableDisplayValueProp]" @unlink="unlinkRef(value)" />
      </template>
    </div>

    <div
      v-if="!readOnly && !isLocked && (isUIAllowed('xcDatatableEditable') || isForm) && !isUnderLookup"
      class="flex justify-end gap-1 min-h-[30px] items-center"
    >
      <GeneralIcon
        :icon="addIcon"
        class="text-sm nc-action-icon text-gray-500/50 hover:text-gray-500 select-none group-hover:(text-gray-500) nc-plus"
        @click.stop="listItemsDlg = true"
      />
    </div>

    <LazyVirtualCellComponentsListItems v-model="listItemsDlg" @attach-record="listItemsDlg = true" />
  </div>
</template>

<style scoped lang="scss">
.nc-action-icon {
  @apply hidden cursor-pointer;
}

.chips-wrapper:hover,
.chips-wrapper.active {
  .nc-action-icon {
    @apply inline-block;
  }
}
</style>
