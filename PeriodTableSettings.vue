<template>
  <div class="period-table-settings">
    <V2Button
      :id="`${pageId}_button_settings`"
      style-type="clean"
      size="md"
      @click="openSettings"
    >
      <IconSprite :width="20" :height="20" name="settings-01" />
    </V2Button>
    <div v-if="isOpenSettings" class="settings">
      <div>
        <div v-for="(header, idx) in headersList" :key="header.code">
          <div
            v-if="header.name"
            draggable="true"
            class="settings__line"
            @dragstart="dragStart($event, idx)"
            @dragover.prevent
            @dragenter.prevent
            @drop="dragDrop($event, idx)"
          >
            <div class="settings__line-icon">
              <IconSprite
                :width="20"
                :height="20"
                name="ic_round-drag-indicator"
              />
              {{ header.name }}
            </div>

            <div>
              <Toggle
                :id="`${pageId}_toggle_${header.code}`"
                size="small"
                :module-checked="header.isVisibility"
                @change="changeValue(header.code)"
              />
            </div>
          </div>
        </div>
      </div>
      <div class="settings__btns">
        <Button
          :id="`${pageId}_button_submit`"
          class="settings__btn"
          text="Применить"
          size="xs"
          color="primary"
          view="fill"
          @click="onSubmit"
        />
        <Button
          :id="`${pageId}_button_reset`"
          class="settings__btn"
          text="Сбросить"
          size="xs"
          color="primary"
          view="smoke"
          @click="onReset"
        />
        
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { IHeaders } from '@/screens/References/Periods/interfacePeriods' 

  interface IProps {
    headers: IHeaders[]
    isOpen: boolean
  }

  const props = defineProps<IProps>()

  const emits = defineEmits<IEmits>()

  interface IEmits {
    (e: 'submit', headers: IHeaders[]): void
    (e: 'reset'): void
  }
  const pageId = 'period_table_settings'
  const isOpenSettings = ref<boolean>(props.isOpen)

  const headersList = ref([...props.headers])

  

  const openSettings = () => {
    isOpenSettings.value = !isOpenSettings.value
  }
  const dragStart = (e: DragEvent, idx: number) => {
    if (!e.dataTransfer) return
    e.dataTransfer.dropEffect = 'move'
    e.dataTransfer.effectAllowed = 'move'
    e.dataTransfer.setData('itemID', idx.toString())
  }

  const dragDrop = (e: DragEvent, idx: number): void => {
    if (!e.dataTransfer) return
    e.stopPropagation()
    e.preventDefault()
    const itemIndex = Number(e.dataTransfer.getData('itemID'))
    const droppedIndex = idx
    changeTable(itemIndex, droppedIndex)
  }
  const changeTable = (itemIndex: number, droppedIndex: number) => {
    const movedItem = headersList.value[itemIndex]
    headersList.value.splice(itemIndex, 1)
    headersList.value.splice(droppedIndex, 0, movedItem)
  }
  const changeValue = (code: string) => {
    const curIdx = headersList.value.findIndex(
      (header: IHeaders) => header.code === code,
    )
    headersList.value[curIdx].isVisibility =
      !headersList.value[curIdx].isVisibility
  }
  const onSubmit = () => {
    emits('submit', [...headersList.value])
  }
  const onReset = () => {
    headersList.value = [...props.headers]
    emits('reset')
  }
</script>

<style lang="sass" scoped>
  @import 'periodTableSettings'
</style>
