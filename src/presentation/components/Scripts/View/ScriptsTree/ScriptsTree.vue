<template>
  <span id="container">
    <span v-if="nodes != null && nodes.length > 0">
      <SelectableTree
        :initialNodes="nodes"
        :selectedNodeIds="selectedNodeIds"
        :filterPredicate="filterPredicate"
        :filterText="filterText"
        v-on:nodeSelected="toggleNodeSelection($event)"
      />
    </span>
    <span v-else>Nooo 😢</span>
  </span>
</template>

<script lang="ts">
import {
  defineComponent, watch, ref,
} from 'vue';
import { useCollectionState } from '@/presentation/components/Shared/Hooks/UseCollectionState';
import { IScript } from '@/domain/IScript';
import { ICategory } from '@/domain/ICategory';
import { ICategoryCollectionState } from '@/application/Context/State/ICategoryCollectionState';
import { IFilterResult } from '@/application/Context/State/Filter/IFilterResult';
import { SelectedScript } from '@/application/Context/State/Selection/SelectedScript';
import {
  parseAllCategories, parseSingleCategory, getScriptNodeId, getCategoryNodeId, getCategoryId,
  getScriptId,
} from './ScriptNodeParser';
import SelectableTree from './SelectableTree/SelectableTree.vue';
import { INodeContent, NodeType } from './SelectableTree/Node/INodeContent';
import { INodeSelectedEvent } from './SelectableTree/INodeSelectedEvent';

export default defineComponent({
  props: {
    categoryId: {
      type: Number,
      default: undefined,
    },
  },
  components: {
    SelectableTree,
  },
  setup(props) {
    const {
      modifyCurrentState, currentState, onStateChange, events,
    } = useCollectionState();

    const nodes = ref<ReadonlyArray<INodeContent>>([]);
    const selectedNodeIds = ref<ReadonlyArray<string>>([]);
    const filterText = ref<string | undefined>(undefined);

    let filtered: IFilterResult | undefined;

    watch(
      () => props.categoryId,
      async (newCategoryId) => { await setNodes(newCategoryId); },
      { immediate: true },
    );

    onStateChange((state) => {
      setCurrentFilter(state.filter.currentFilter);
      if (!props.categoryId) {
        nodes.value = parseAllCategories(state.collection);
      }
      events.unsubscribeAll();
      modifyCurrentState((mutableState) => {
        registerStateMutators(mutableState);
      });
    }, { immediate: true });

    function toggleNodeSelection(event: INodeSelectedEvent) {
      modifyCurrentState((state) => {
        switch (event.node.type) {
          case NodeType.Category:
            toggleCategoryNodeSelection(event, state);
            break;
          case NodeType.Script:
            toggleScriptNodeSelection(event, state);
            break;
          default:
            throw new Error(`Unknown node type: ${event.node.id}`);
        }
      });
    }

    function filterPredicate(node: INodeContent): boolean {
      return containsScript(node, filtered.scriptMatches)
        || containsCategory(node, filtered.categoryMatches);
    }

    async function setNodes(categoryId?: number) {
      if (categoryId) {
        nodes.value = parseSingleCategory(categoryId, currentState.value.collection);
      } else {
        nodes.value = parseAllCategories(currentState.value.collection);
      }
      selectedNodeIds.value = currentState.value.selection.selectedScripts
        .map((selected) => getScriptNodeId(selected.script));
    }

    function registerStateMutators(state: ICategoryCollectionState) {
      events.register(
        state.selection.changed.on((scripts) => handleSelectionChanged(scripts)),
        state.filter.filterRemoved.on(() => handleFilterRemoved()),
        state.filter.filtered.on((filterResult) => handleFiltered(filterResult)),
      );
    }

    function setCurrentFilter(currentFilter: IFilterResult | undefined) {
      if (!currentFilter) {
        handleFilterRemoved();
      } else {
        handleFiltered(currentFilter);
      }
    }

    function handleSelectionChanged(selectedScripts: ReadonlyArray<SelectedScript>): void {
      selectedNodeIds.value = selectedScripts
        .map((node) => node.id);
    }

    function handleFilterRemoved() {
      filterText.value = '';
    }

    function handleFiltered(result: IFilterResult) {
      filterText.value = result.query;
      filtered = result;
    }

    return {
      nodes,
      selectedNodeIds,
      filterText,
      toggleNodeSelection,
      filterPredicate,
    };
  },
});

function containsScript(expected: INodeContent, scripts: readonly IScript[]) {
  return scripts.some((existing: IScript) => expected.id === getScriptNodeId(existing));
}

function containsCategory(expected: INodeContent, categories: readonly ICategory[]) {
  return categories.some((existing: ICategory) => expected.id === getCategoryNodeId(existing));
}

function toggleCategoryNodeSelection(
  event: INodeSelectedEvent,
  state: ICategoryCollectionState,
): void {
  const categoryId = getCategoryId(event.node.id);
  if (event.isSelected) {
    state.selection.addOrUpdateAllInCategory(categoryId, false);
  } else {
    state.selection.removeAllInCategory(categoryId);
  }
}

function toggleScriptNodeSelection(
  event: INodeSelectedEvent,
  state: ICategoryCollectionState,
): void {
  const scriptId = getScriptId(event.node.id);
  const actualToggleState = state.selection.isSelected(scriptId);
  const targetToggleState = event.isSelected;
  if (targetToggleState && !actualToggleState) {
    state.selection.addSelectedScript(scriptId, false);
  } else if (!targetToggleState && actualToggleState) {
    state.selection.removeSelectedScript(scriptId);
  }
}
</script>
