<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import { ref, computed } from 'vue'

import { useMessage } from '@/hooks'
import { deepClone, sampleID } from '@/utils'
import { type ProfileType, useSubscribesStore } from '@/stores'
import { GroupsTypeOptions, StrategyOptions, ProxyGroup, DraggableOptions } from '@/constant'

type GroupsType = ProfileType['proxyGroupsConfig']

const groups = defineModel<GroupsType>({ default: [] })

let updateGroupId = 0
const showModal = ref(false)
const showSortModal = ref(false)
const expandedSet = ref<Set<string>>(new Set(['Built-In', 'Subscribes']))
const SubscribesNameMap = ref<Record<string, string>>({})

const proxyGroup = ref([
  {
    id: 'Built-In',
    name: 'kernel.proxyGroups.builtIn',
    proxies: [
      { id: 'DIRECT', name: 'DIRECT', type: 'Built-In' },
      { id: 'REJECT', name: 'REJECT', type: 'Built-In' },
      ...groups.value.map(({ id, name, type }) => ({ id, name, type }))
    ]
  },
  {
    id: 'Subscribes',
    name: 'kernel.proxyGroups.subscriptions',
    proxies: []
  }
])

const fields = ref<GroupsType[0]>({
  id: sampleID(),
  name: '',
  type: ProxyGroup.Select,
  proxies: [],
  url: '',
  interval: 0,
  strategy: '',
  use: [],
  tolerance: 0,
  lazy: true,
  'disable-udp': false,
  filter: ''
})

const { t } = useI18n()
const { message } = useMessage()
const subscribesStore = useSubscribesStore()

const handleAdd = () => {
  updateGroupId = -1
  fields.value = {
    id: sampleID(),
    name: '',
    type: ProxyGroup.Select,
    proxies: [],
    url: 'https://www.gstatic.com/generate_204',
    interval: 300,
    strategy: 'consistent-hashing',
    use: [],
    tolerance: 150,
    lazy: true,
    'disable-udp': false,
    filter: ''
  }
  showModal.value = true
}

defineExpose({ handleAdd })

const handleDeleteGroup = (index: number) => {
  const name = groups.value[index].name
  groups.value.splice(index, 1)
  proxyGroup.value = proxyGroup.value.map((v) => ({
    ...v,
    proxies: v.proxies.filter((v) => v.name !== name)
  }))
}

const handleClearGroup = async (g: GroupsType[0]) => {
  g.proxies = g.proxies.filter(({ type, name, id }) => {
    if (type === 'Built-In') {
      if (['DIRECT', 'REJECT'].includes(name)) {
        return true
      }
      return groups.value.some((v) => v.id === id)
    }
    const sub = subscribesStore.getSubscribeById(type)
    if (!sub) return false
    return sub.proxies.some((v) => v.id === id)
  })

  g.use = g.use.filter((v) => subscribesStore.subscribes.some(({ id }) => id === v))
}

const handleEditGroup = (index: number) => {
  updateGroupId = index
  fields.value = deepClone(groups.value[index])
  showModal.value = true
}

const handleAddProxy = (groupID: string, proxyID: string, proxyName: string) => {
  // self
  if (groupID === 'Built-In' && proxyID === fields.value.id) return

  // subscribes
  if (groupID === 'Subscribes') {
    const idx = fields.value.use.findIndex((v) => v === proxyID)
    if (idx !== -1) {
      fields.value.use.splice(idx, 1)
    } else {
      fields.value.use.push(proxyID)
    }
    return
  }

  // proxy
  const idx = fields.value.proxies.findIndex((v) => v.id === proxyID)
  if (idx !== -1) {
    fields.value.proxies.splice(idx, 1)
  } else {
    fields.value.proxies.push({ id: proxyID, type: groupID, name: proxyName })
  }
}

const handleAddEnd = () => {
  const { id, name, type } = fields.value
  // Add
  if (updateGroupId === -1) {
    groups.value.unshift(fields.value)
    proxyGroup.value[0].proxies.unshift({ id, name, type })
    return
  }
  // Update
  groups.value[updateGroupId] = fields.value
  const idx = proxyGroup.value[0].proxies.findIndex((v) => v.id === id)
  if (idx !== -1) {
    proxyGroup.value[0].proxies.splice(idx, 1, { id, name, type })
  }
}

const isInuse = (groupID: string, proxyID: string) => {
  if (groupID === 'Subscribes') {
    return fields.value.use.includes(proxyID)
  }
  return fields.value.proxies.find((v) => v.id === proxyID)
}

const isSupportInverval = computed(() =>
  [ProxyGroup.UrlTest, ProxyGroup.Fallback, ProxyGroup.LoadBalance].includes(fields.value.type)
)

const hasLost = (g: GroupsType[0]) => {
  const isProxiesLost = g.proxies.some(({ type, id }) => {
    if (type === 'Built-In') {
      if (['DIRECT', 'REJECT'].includes(id)) {
        return false
      }
      return groups.value.every((v) => v.id !== id)
    }

    const sub = subscribesStore.getSubscribeById(type)
    if (!sub) return true
    return sub.proxies.every((v) => v.id !== id)
  })

  const isUseLost = g.use.some((v) => {
    return subscribesStore.subscribes.every(({ id }) => id !== v)
  })

  return isProxiesLost || isUseLost
}

const handleSortGroup = (index: number) => {
  updateGroupId = index
  fields.value = deepClone(groups.value[index])
  showSortModal.value = true
}

const handleSortGroupEnd = () => {
  groups.value[updateGroupId] = fields.value
}

const needToAdd = (g: GroupsType[0]) => g.use.length === 0 && g.proxies.length === 0

const toggleExpanded = (key: string) => {
  if (expandedSet.value.has(key)) {
    expandedSet.value.delete(key)
  } else {
    expandedSet.value.add(key)
  }
}

const isExpanded = (key: string) => expandedSet.value.has(key)

const showLost = () => message.warn('kernel.proxyGroups.notFound')

const showNeedToAdd = () => message.warn('kernel.proxyGroups.needToAdd')

subscribesStore.subscribes.forEach(async ({ id, name, proxies }) => {
  proxyGroup.value[1].proxies.push({ id, name, type: 'use' })
  proxyGroup.value.push({ id, name, proxies })
  SubscribesNameMap.value[id] = name
})
</script>

<template>
  <div v-draggable="[groups, DraggableOptions]">
    <Card v-for="(g, index) in groups" :key="g.id" class="groups-item">
      <div class="name">
        <span v-if="hasLost(g)" @click="showLost" class="warn"> [ ! ] </span>
        <span v-if="needToAdd(g)" @click="showNeedToAdd" class="warn"> [ ! ] </span>
        {{ g.name }}
      </div>
      <div class="count">
        <Button @click="handleSortGroup(index)" type="link" size="small">
          (
          {{ t('profile.use') }}: {{ g.use.length }}
          /
          {{ t('profile.proxies') }}: {{ g.proxies.length }}
          )
        </Button>
      </div>
      <div class="action">
        <Button @click="handleClearGroup(g)" v-if="hasLost(g)" type="text">
          {{ t('common.clear') }}
        </Button>
        <Button @click="handleEditGroup(index)" type="text" size="small">
          {{ t('common.edit') }}
        </Button>
        <Button @click="handleDeleteGroup(index)" type="text" size="small">
          {{ t('common.delete') }}
        </Button>
      </div>
    </Card>
  </div>

  <Modal
    v-model:open="showSortModal"
    :title="t('kernel.proxyGroups.sort')"
    @ok="handleSortGroupEnd"
    max-width="80"
    max-height="80"
  >
    <div class="group">
      <Divider>{{ t('profile.use') }}</Divider>
      <Empty v-if="fields.use.length === 0"></Empty>
      <div v-draggable="[fields.use, DraggableOptions]" class="group-proxies">
        <Button v-for="use in fields.use" :key="use" type="link" class="group-item">
          {{ SubscribesNameMap[use] || use }}
        </Button>
      </div>
      <Divider>{{ t('profile.proxies') }}</Divider>
      <Empty v-if="fields.proxies.length === 0"></Empty>
      <div v-draggable="[fields.proxies, DraggableOptions]" class="group-proxies">
        <Button v-for="proxy in fields.proxies" :key="proxy.id" type="link" class="group-item">
          {{ proxy.name }}
        </Button>
      </div>
    </div>
  </Modal>

  <Modal
    v-model:open="showModal"
    @ok="handleAddEnd"
    title="profile.group"
    max-width="80"
    max-height="80"
  >
    <div class="form-item">
      {{ t('kernel.proxyGroups.name') }}
      <Input v-model="fields.name" autofocus />
    </div>
    <div class="form-item">
      {{ t('kernel.proxyGroups.filter') }}
      <Input v-model="fields.filter" placeholder="keyword1|keyword2" />
    </div>
    <div class="form-item">
      {{ t('kernel.proxyGroups.type.name') }}
      <Radio v-model="fields.type" :options="GroupsTypeOptions" />
    </div>
    <div class="form-item">
      {{ t('kernel.proxyGroups.disable-udp') }}
      <Switch v-model="fields['disable-udp']" />
    </div>
    <template v-if="isSupportInverval">
      <div class="form-item">
        {{ t('kernel.proxyGroups.lazy') }}
        <Switch v-model="fields.lazy" />
      </div>
      <div class="form-item">
        {{ t('kernel.proxyGroups.interval') }}
        <Input v-model="fields.interval" type="number" />
      </div>
      <div class="form-item">
        {{ t('kernel.proxyGroups.url') }}
        <Input v-model="fields.url" />
      </div>
    </template>
    <div v-show="fields.type === ProxyGroup.UrlTest" class="form-item">
      {{ t('kernel.proxyGroups.tolerance') }}
      <Input v-model="fields.tolerance" type="number" />
    </div>
    <div v-show="fields.type === ProxyGroup.LoadBalance" class="form-item">
      {{ t('kernel.proxyGroups.strategy.name') }}
      <Radio v-model="fields.strategy" :options="StrategyOptions" />
    </div>

    <Divider> {{ t('profile.use') }} & {{ t('profile.proxies') }} </Divider>

    <div v-for="group in proxyGroup" :key="group.id" class="group">
      <Button
        :type="isExpanded(group.id) ? 'link' : 'text'"
        @click="toggleExpanded(group.id)"
        class="group-title"
      >
        {{ t(group.name) }}
        <div style="margin-left: auto; margin-right: 8px">{{ group.proxies.length }}</div>
        <Icon
          :class="{ 'rotate-z': isExpanded(group.id) }"
          icon="arrowRight"
          class="action-expand"
        />
      </Button>
      <div v-show="isExpanded(group.id)" class="group-proxies">
        <Empty v-if="group.proxies.length === 0" :description="t('kernel.proxyGroups.empty')" />
        <template v-else>
          <div v-for="proxy in group.proxies" :key="proxy.id" class="group-item">
            <Button
              @click="handleAddProxy(group.id, proxy.id, proxy.name)"
              :type="isInuse(group.id, proxy.id) ? 'link' : 'text'"
            >
              {{ proxy.name }}
              <br />
              {{ proxy.type }}
            </Button>
          </div>
        </template>
      </div>
    </div>
  </Modal>
</template>

<style lang="less" scoped>
.groups-item {
  display: flex;
  align-items: center;
  padding: 0 8px;
  margin-bottom: 2px;
  .name {
    font-weight: bold;
    min-width: 90px;
    .warn {
      color: rgb(200, 193, 11);
      cursor: pointer;
    }
  }
  .count {
    margin-left: 4px;
  }
  .action {
    margin-left: auto;
  }
}

.group {
  .group-title {
    position: sticky;
    top: 0;
    width: 100%;
    display: flex;
    align-items: center;
    backdrop-filter: blur(4px);
  }
  .group-proxies {
    display: flex;
    flex-wrap: wrap;
    background: var(--card-bg);
    border-radius: 8px;
  }
  .group-item {
    display: flex;
    justify-content: center;
    width: calc(25% - 8px);
    margin: 4px;
  }
  .action-expand {
    transition: all 0.2s;
  }
  .rotate-z {
    transform: rotateZ(90deg);
  }
}
</style>
