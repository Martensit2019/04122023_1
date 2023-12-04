<template>
  <div v-if="periodsData" class="periods">
    <div class="periods__controls d-flex">
      <PeriodsControls
        :selected-period="selectedPeriod"
        :is-filter-open="isFilterOpen"
        @add-period="onOpenAddModal"
        @open-period="onOpenModalOpen"
        @close-period="onOpenCloseModal"
        @search-period="searchPeriod"
        @toggle-filter="onToggleFilter"
      />
    </div>
    <PeriodsFilter v-if="isFilterOpen" :filter-data="filterData" />
    <div class="periods__table mt-20">
      <PeriodTableSettings
        :key="keyPeriodTableSettings"
        :is-open="isOpen"
        :headers="settingHeaders" 
        @submit="onSubmit" 
        @reset="onReset" />
      <PeriodsTable
        table-id="references_periods_table"
        :table-headers="tableHeaders"
        :table-rows="periodsData"
        :edit-period-id="editPeriodId"
        :sortable-columns="[
          'name',
          'open_at',
          'close_at',
          'status_name',
          'period_ais_name',
          'parent_period_name',
        ]"
        @edit-period="onEditPeriod"
        @delete-period="onOpenDeleteModal"
        @select-period="onSelectPeriod"
      />
    </div>
  </div>
  <PeriodsModalOpen
    id="references_periods_modal_open"
    :is-visible="isOpenModalVisible"
    :is-opened-period-exist="isOpenedPeriodExist"
    @close-modal="onCloseModalOpen"
    @submit="onModalOpen"
  />
  <PeriodsModalAddEdit
    v-if="isAddEditModalVisible"
    id="references_periods_modal_add_edit"
    :is-visible="isAddEditModalVisible"
    :mode="modalAddEditMode"
    :row="editPeriod"
    :periods-for-ui-data="periodsForUiData"
    :normativ-periods-data="normativPeriodsData"
    @submit-add-modal="onModalAdd"
    @submit-edit-modal="onModalEdit"
    @close-modal="onAddEditModalClose"
  />
  <PeriodsModalCloseDelete
    v-if="isCloseDeleteModalVisible"
    id="references_periods_modal_close_delete"
    :is-visible="isCloseDeleteModalVisible"
    :mode="modalCloseDeleteMode"
    @submit-close-modal="onModalClose"
    @submit-delete-modal="onModalDelete"
    @close-modal="onCloseDeleteModalClose"
  />
</template>

<script setup lang="ts">
  import { useAppStore } from '@/stores/app'
  import { PERIOD_STATUS } from '@/enums'
  import {
    getPeriodsRequest,
    postPeriodRequest,
    patchPeriodRequest,
    deletePeriodRequest,
    getNormativPeriodsRequest,
    getPeriodsForUiRequest,
    getPeriodTableSettings,
    postPeriodTableSettings,
    IPeriodsForUi,
    INormativPeriod,
    type IGetPeriodsParams,
    type IPeriod,
    type IPostPeriodPayload,
    type IPatchPeriodPayload,
    type IPostPeriodTableSettingsPayload
  } from '@/api'
  import { IPeriodFilter } from '@/screens/References/Periods/types'
  import { IObject } from '@/components/common/interfaceCommon'
  import { periodHeaders } from './dataReferencesPeriods'
  import { IHeaders } from '@/screens/References/Periods/interfacePeriods' 
  import PeriodsTable from './components/PeriodsTable/PeriodsTable.vue'
  import PeriodsControls from './components/PeriodsControls/PeriodsControls.vue'
  import PeriodsFilter from './components/PeriodsFilter/PeriodsFilter.vue'
  import PeriodsModalOpen from './components/modals/PeriodsModalOpen/PeriodsModalOpen.vue'
  import PeriodsModalAddEdit from './components/modals/PeriodsModalAddEdit/PeriodsModalAddEdit.vue'
  import PeriodsModalCloseDelete from './components/modals/PeriodsModalCloseDelete/PeriodsModalCloseDelete.vue'
  import PeriodTableSettings from './components/PeriodTableSettings/PeriodTableSettings.vue'
  import {
    getPeriodStatusLabel,
    formatVisualDate,
    VISUAL_DATE_REVERSE,
  } from '@/helpers'

  const appStore = useAppStore()

  
  const periodsData = ref<IPeriod[]>([])

  const isOpenModalVisible = ref<boolean>(false)
  const isAddEditModalVisible = ref<boolean>(false)
  const isCloseDeleteModalVisible = ref<boolean>(false)
  const isFilterOpen = ref<boolean>(false)
  const filterData = ref<IPeriodFilter>({
    dateOpenSince: undefined,
    dateOpenUntil: undefined,
    dateCloseSince: undefined,
    dateCloseUntil: undefined,
  })

  const modalAddEditMode = ref<'add' | 'edit'>('add')
  const modalCloseDeleteMode = ref<'close' | 'delete'>('close')

  const selectedPeriod = ref()
  const editPeriod = ref<IPeriod | undefined>()
  const editPeriodId = ref<number | string | undefined>()

  const normativPeriodsData = ref<INormativPeriod[]>([])
  const periodsForUiData = ref<IPeriodsForUi[]>([])

  const isOpenedPeriodExist = computed((): boolean =>
    periodsData.value.some(
      (period: IPeriod) => period.status === PERIOD_STATUS.opened,
    ),
  )

  const onToggleFilter = () => {
    isFilterOpen.value = !isFilterOpen.value
  }

  const search = ref<string | undefined>(undefined)
  const searchPeriod = (value: string): void => {
    search.value = value || undefined
    fetchPeriods()
  }
  async function fetchPeriods() {
    const params: IGetPeriodsParams = {
      search_text: search.value,
      open_at_start: filterData.value.dateOpenSince
        ? formatVisualDate(filterData.value.dateOpenSince, VISUAL_DATE_REVERSE)
        : undefined,
      open_at_finish: filterData.value.dateOpenUntil
        ? formatVisualDate(filterData.value.dateOpenUntil, VISUAL_DATE_REVERSE)
        : undefined,
      close_at_start: filterData.value.dateCloseSince
        ? formatVisualDate(filterData.value.dateCloseSince, VISUAL_DATE_REVERSE)
        : undefined,
      close_at_finish: filterData.value.dateCloseUntil
        ? formatVisualDate(filterData.value.dateCloseUntil, VISUAL_DATE_REVERSE)
        : undefined,
    }
    getPeriodsRequest(params).then((res) => {
      if (res?.data?.results) {
        periodsData.value = res.data.results.map((period: IPeriod) => {
          const periodData = { ...period }
          periodData.status = period.status && period.status?.toLowerCase()
          periodData.status_name =
            period.status && getPeriodStatusLabel(period.status?.toLowerCase())
          periodData.parent_period_name = period.parent
            ? period.parent.name
            : ''
          return periodData
        })
      }
    })
  }

  async function fetchNormativPeriods() {
    getNormativPeriodsRequest().then((res) => {
      if (res?.data) normativPeriodsData.value = res.data.periods
    })
  }

  async function fetchPeriodsForUi() {
    getPeriodsForUiRequest().then((res) => {
      if (res?.data) periodsForUiData.value = res.data
    })
  }

  const onSelectPeriod = (period: IPeriod) => {
    selectedPeriod.value = period
  }

  const onEditPeriod = (periodId: string | number) => {
    editPeriodId.value = periodId
    editPeriod.value = periodsData.value.find(
      (period) => period.id === periodId,
    )
    if (editPeriod.value) onOpenEditModal(editPeriod.value)
  }

  const onOpenAddModal = () => {
    modalAddEditMode.value = 'add'
    isAddEditModalVisible.value = true
  }

  const onOpenEditModal = (row: IPeriod) => {
    editPeriod.value = row
    modalAddEditMode.value = 'edit'
    isAddEditModalVisible.value = true
  }

  const onAddEditModalClose = () => {
    isAddEditModalVisible.value = false
    editPeriod.value = undefined
  }

  const onModalAdd = (period: IObject) => {
    isAddEditModalVisible.value = false
    const payload: IPostPeriodPayload = {
      name: period.name,
    }

    if (period.parentPeriod?.id) {
      payload.parent_id = period.parentPeriod?.id
    }

    if (period.periodNormAis?.id) {
      payload.period_ais_id = period.periodNormAis?.id
    }

    postPeriodRequest(payload).finally(() => fetchPeriods())
  }

  const onModalEdit = (period: IObject) => {
    if (!editPeriod.value) return {}

    isAddEditModalVisible.value = false
    const payload: IPatchPeriodPayload = {
      name: period.name,
    }

    if (period.parentPeriod?.id) {
      payload.parent_id = period.parentPeriod.id
    }
    if (period.periodNormAis?.id) {
      payload.period_ais_id = period.periodNormAis.id
    }

    patchPeriodRequest(period.id, payload).finally(() => fetchPeriods())

    return period
  }

  const periodOpenParams = computed(() => ({
    hasParent: selectedPeriod.value?.parent != null,
    hasAisNorm: selectedPeriod.value?.period_ais_id != null,
  }))

  const onOpenModalOpen = () => {
    const { hasParent, hasAisNorm } = periodOpenParams.value
    if (hasParent && hasAisNorm) isOpenModalVisible.value = true
    else {
      const missingProperty = hasParent
        ? 'Период Норматив ТИМ'
        : 'Родительский период'

      appStore.addAppAlert({
        text: `Период не возможно перевести в статус Открыт. Необходимо заполнить поле ${missingProperty}`,
        color: 'wrong',
      })
    }
  }
  const onCloseModalOpen = () => {
    isOpenModalVisible.value = false
  }
  const onModalOpen = () => {
    if (!selectedPeriod.value) return

    const payload: IPatchPeriodPayload = {
      status: PERIOD_STATUS.opened,
    }

    patchPeriodRequest(selectedPeriod.value.id, payload).finally(() =>
      fetchPeriods(),
    )
    isOpenModalVisible.value = false
  }

  const onOpenCloseModal = () => {
    modalCloseDeleteMode.value = 'close'
    isCloseDeleteModalVisible.value = true
  }
  const onOpenDeleteModal = () => {
    modalCloseDeleteMode.value = 'delete'
    isCloseDeleteModalVisible.value = true
  }
  const onCloseDeleteModalClose = () => {
    isCloseDeleteModalVisible.value = false
    selectedPeriod.value = undefined
  }
  const onModalClose = () => {
    if (!selectedPeriod.value) return

    const payload: IPatchPeriodPayload = {
      status: PERIOD_STATUS.closed,
    }

    patchPeriodRequest(selectedPeriod.value.id, payload).finally(() =>
      fetchPeriods(),
    )
    isCloseDeleteModalVisible.value = false
  }
  const onModalDelete = () => {
    if (!selectedPeriod.value) return

    deletePeriodRequest(selectedPeriod.value.id).finally(() => fetchPeriods())
    isCloseDeleteModalVisible.value = false
  }


  const tableHeaders = ref(periodHeaders)
  const settingHeaders = ref(periodHeaders)
  const isOpen = ref<boolean>(false)

  const keyPeriodTableSettings = ref<number>(0)
  
  const onSubmit = (headers: IHeaders[]) => {

    tableHeaders.value = headers.filter((header: IHeaders) => header.isVisibility)
    const order = headers.map((header: IHeaders, idx: number) => ({...header, order: idx + 1})).reduce((acc: IObject, header: IHeaders) => {
      acc[header.code] = header.order
      return acc
    }, {})
    
    const visibility = headers.reduce((acc: IObject, header: IHeaders) => {
      acc[header.code] = header.isVisibility
      return acc
    }, {})
          const payload: IPostPeriodTableSettingsPayload = {
        user_settings: {
          order,
          visibility
        }
    }
    postPeriodTableSettings('administration-periods', payload)
  }
  const onReset = () => {
    tableHeaders.value = periodHeaders    
    settingHeaders.value = periodHeaders.map((header: IHeaders) => ({...header, isVisibility: true}))
    keyPeriodTableSettings.value++
    isOpen.value = true
  }

  const getSettingsTable = () => {
    getPeriodTableSettings('administration-periods').then((res) => {
      if(res?.data?.user_settings){
         const userSettings = res.data.user_settings
         if(userSettings.order){
          tableHeaders.value = tableHeaders.value.map((header: IHeaders) => ({
              ...header, order: userSettings.order[header.code]
            })).sort((head1: IHeaders, head2: IHeaders) => (head1.order && head2.order) && (head1.order > head2.order )? 1 : -1)
            settingHeaders.value = settingHeaders.value.map((header: IHeaders) => ({
              ...header, order: userSettings.order[header.code]
            })).sort((head1: IHeaders, head2: IHeaders) => (head1.order && head2.order) && (head1.order > head2.order )? 1 : -1)
            keyPeriodTableSettings.value++
         }
         if(userSettings.visibility){
          tableHeaders.value = tableHeaders.value.map((header: IHeaders) => ({
            ...header, isVisibility: userSettings.visibility[header.code]
          })).filter((header: IHeaders) => header.isVisibility)
          settingHeaders.value = settingHeaders.value.map((header: IHeaders) => ({
            ...header, isVisibility: userSettings.visibility[header.code]
          }))
         }
      }
     
      
    })
  }

  watch(
    filterData,
    () => {
      fetchPeriods()
    },
    { deep: true },
  )

  onMounted(() => {
    fetchPeriods()
    getSettingsTable()
    fetchNormativPeriods()
    fetchPeriodsForUi()
  })
</script>

<style lang="sass" scoped>
  @import 'referencesPeriods'
</style>
