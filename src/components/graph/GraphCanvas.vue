<template>
  <teleport to=".graph-canvas-container">
    <LiteGraphCanvasSplitterOverlay v-if="betaMenuEnabled">
      <template #side-bar-panel>
        <SideToolbar />
      </template>
    </LiteGraphCanvasSplitterOverlay>
    <NodeTitleEditor />
    <canvas ref="canvasRef" id="graph-canvas" tabindex="1" />
  </teleport>
  <NodeSearchboxPopover v-if="nodeSearchEnabled" />
  <NodeTooltip />
</template>

<script setup lang="ts">
import NodeTitleEditor from '@/components/graph/NodeTitleEditor.vue'
import SideToolbar from '@/components/sidebar/SideToolbar.vue'
import LiteGraphCanvasSplitterOverlay from '@/components/LiteGraphCanvasSplitterOverlay.vue'
import NodeSearchboxPopover from '@/components/searchbox/NodeSearchBoxPopover.vue'
import NodeTooltip from '@/components/graph/NodeTooltip.vue'
import { ref, computed, onUnmounted, watch, onMounted, watchEffect } from 'vue'
import { app as comfyApp } from '@/scripts/app'
import { useSettingStore } from '@/stores/settingStore'
import { dropTargetForElements } from '@atlaskit/pragmatic-drag-and-drop/element/adapter'
import { ComfyNodeDefImpl, useNodeDefStore } from '@/stores/nodeDefStore'
import { useWorkspaceStore } from '@/stores/workspaceStateStore'
import {
  LiteGraph,
  LGraph,
  LLink,
  LGraphNode,
  LGraphGroup,
  DragAndScale,
  LGraphCanvas,
  ContextMenu
} from '@comfyorg/litegraph'
import type { RenderedTreeExplorerNode } from '@/types/treeExplorerTypes'
import { useNodeBookmarkStore } from '@/stores/nodeBookmarkStore'

const emit = defineEmits(['ready'])
const canvasRef = ref<HTMLCanvasElement | null>(null)
const settingStore = useSettingStore()
const nodeDefStore = useNodeDefStore()
const workspaceStore = useWorkspaceStore()

const betaMenuEnabled = computed(
  () => settingStore.get('Comfy.UseNewMenu') !== 'Disabled'
)
const nodeSearchEnabled = computed<boolean>(
  () => settingStore.get('Comfy.NodeSearchBoxImpl') === 'default'
)
watch(
  nodeSearchEnabled,
  (newVal) => {
    LiteGraph.release_link_on_empty_shows_menu = !newVal
    if (comfyApp.canvas) comfyApp.canvas.allow_searchbox = !newVal
  },
  { immediate: true }
)
const canvasInfoEnabled = computed<boolean>(() =>
  settingStore.get('Comfy.Graph.CanvasInfo')
)
watch(
  canvasInfoEnabled,
  (newVal) => {
    if (comfyApp.canvas) comfyApp.canvas.show_info = newVal
  },
  { immediate: true }
)

const zoomSpeed = computed(() => settingStore.get('Comfy.Graph.ZoomSpeed'))
watch(
  zoomSpeed,
  (newVal: number) => {
    if (comfyApp.canvas) comfyApp.canvas['zoom_speed'] = newVal
  },
  { immediate: true }
)

watchEffect(() => {
  nodeDefStore.showDeprecated = settingStore.get('Comfy.Node.ShowDeprecated')
})

watchEffect(() => {
  nodeDefStore.showExperimental = settingStore.get(
    'Comfy.Node.ShowExperimental'
  )
})

watchEffect(() => {
  const spellcheckEnabled = settingStore.get('Comfy.TextareaWidget.Spellcheck')
  const textareas = document.querySelectorAll('textarea.comfy-multiline-input')

  textareas.forEach((textarea: HTMLTextAreaElement) => {
    textarea.spellcheck = spellcheckEnabled
    // Force recheck to ensure visual update
    textarea.focus()
    textarea.blur()
  })
})

let dropTargetCleanup = () => {}

onMounted(async () => {
  // Backward compatible
  // Assign all properties of lg to window
  window['LiteGraph'] = LiteGraph
  window['LGraph'] = LGraph
  window['LLink'] = LLink
  window['LGraphNode'] = LGraphNode
  window['LGraphGroup'] = LGraphGroup
  window['DragAndScale'] = DragAndScale
  window['LGraphCanvas'] = LGraphCanvas
  window['ContextMenu'] = ContextMenu

  comfyApp.vueAppReady = true

  workspaceStore.spinner = true
  await comfyApp.setup(canvasRef.value)
  comfyApp.canvas.allow_searchbox = !nodeSearchEnabled.value
  comfyApp.canvas.show_info = canvasInfoEnabled.value
  comfyApp.canvas['zoom_speed'] = zoomSpeed.value
  workspaceStore.spinner = false

  window['app'] = comfyApp
  window['graph'] = comfyApp.graph

  dropTargetCleanup = dropTargetForElements({
    element: canvasRef.value,
    onDrop: (event) => {
      const loc = event.location.current.input
      const dndData = event.source.data

      if (dndData.type === 'tree-explorer-node') {
        const node = dndData.data as RenderedTreeExplorerNode
        if (node.data instanceof ComfyNodeDefImpl) {
          const nodeDef = node.data
          // Add an offset on x to make sure after adding the node, the cursor
          // is on the node (top left corner)
          const pos = comfyApp.clientPosToCanvasPos([
            loc.clientX - 20,
            loc.clientY
          ])
          comfyApp.addNodeOnGraph(nodeDef, { pos })
        }
      }
    }
  })

  // Migrate legacy bookmarks
  useNodeBookmarkStore().migrateLegacyBookmarks()
  emit('ready')
})

onUnmounted(() => {
  dropTargetCleanup()
})
</script>
