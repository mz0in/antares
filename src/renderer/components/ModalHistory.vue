<template>
   <Teleport to="#window-content">
      <div class="modal active">
         <a class="modal-overlay" @click.stop="closeModal" />
         <div ref="trapRef" class="modal-container p-0 pb-4">
            <div class="modal-header pl-2">
               <div class="modal-title h6">
                  <div class="d-flex">
                     <i class="mdi mdi-24px mdi-history mr-1" />
                     <span class="cut-text">{{ t('word.history') }}: {{ connectionName }}</span>
                  </div>
               </div>
               <a class="btn btn-clear c-hand" @click.stop="closeModal" />
            </div>
            <div class="modal-body p-0 workspace-query-results">
               <div
                  v-if="history.length"
                  ref="searchForm"
                  class="form-group has-icon-right p-2 m-0"
               >
                  <input
                     v-model="searchTerm"
                     class="form-input"
                     type="text"
                     :placeholder="t('message.searchForQueries')"
                  >
                  <i v-if="!searchTerm" class="form-icon mdi mdi-magnify mdi-18px pr-4" />
                  <i
                     v-else
                     class="form-icon c-hand mdi mdi-backspace mdi-18px pr-4"
                     @click="searchTerm = ''"
                  />
               </div>
               <div
                  v-if="history.length"
                  ref="tableWrapper"
                  class="vscroll px-1 "
                  :style="{'height': resultsSize+'px'}"
               >
                  <div ref="table">
                     <BaseVirtualScroll
                        ref="resultTable"
                        :items="filteredHistory"
                        :item-height="66"
                        :visible-height="resultsSize"
                        :scroll-element="scrollElement"
                     >
                        <template #default="{ items }">
                           <div
                              v-for="query in items"
                              :key="query.uid"
                              class="tile my-2"
                              tabindex="0"
                           >
                              <div class="tile-icon">
                                 <i class="mdi mdi-code-tags pr-1" />
                              </div>
                              <div class="tile-content">
                                 <div class="tile-title">
                                    <code
                                       class="cut-text"
                                       :title="query.sql"
                                       v-html="highlightWord(query.sql)"
                                    />
                                 </div>
                                 <div class="tile-bottom-content">
                                    <small class="tile-subtitle">{{ query.schema }} · {{ formatDate(query.date) }}</small>
                                    <div class="tile-history-buttons">
                                       <button class="btn btn-link pl-1" @click.stop="$emit('select-query', query.sql)">
                                          <i class="mdi mdi-open-in-app pr-1" /> {{ t('word.select') }}
                                       </button>
                                       <button class="btn btn-link pl-1" @click="copyQuery(query.sql)">
                                          <i class="mdi mdi-content-copy pr-1" /> {{ t('word.copy') }}
                                       </button>
                                       <button class="btn btn-link pl-1" @click="deleteQuery(query)">
                                          <i class="mdi mdi-delete-forever pr-1" /> {{ t('word.delete') }}
                                       </button>
                                    </div>
                                 </div>
                              </div>
                           </div>
                        </template>
                     </BaseVirtualScroll>
                  </div>
               </div>
               <div v-else class="empty">
                  <div class="empty-icon">
                     <i class="mdi mdi-history mdi-48px" />
                  </div>
                  <p class="empty-title h5">
                     {{ t('message.thereIsNoQueriesYet') }}
                  </p>
               </div>
            </div>
         </div>
      </div>
   </Teleport>
</template>

<script setup lang="ts">
import { Component, computed, ComputedRef, onBeforeUnmount, onMounted, onUpdated, Prop, Ref, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { ConnectionParams } from 'common/interfaces/antares';
import { HistoryRecord, useHistoryStore } from '@/stores/history';
import { useConnectionsStore } from '@/stores/connections';
import { useFocusTrap } from '@/composables/useFocusTrap';
import { useFilters } from '@/composables/useFilters';
import BaseVirtualScroll from '@/components/BaseVirtualScroll.vue';

const { t } = useI18n();
const { formatDate } = useFilters();

const { getHistoryByWorkspace, deleteQueryFromHistory } = useHistoryStore();
const { getConnectionName } = useConnectionsStore();

const { trapRef } = useFocusTrap();

const props = defineProps({
   connection: Object as Prop<ConnectionParams>
});

const emit = defineEmits(['select-query', 'close']);

const table: Ref<HTMLDivElement> = ref(null);
const resultTable: Ref<Component & { updateWindow: () => void }> = ref(null);
const tableWrapper: Ref<HTMLDivElement> = ref(null);
const searchForm: Ref<HTMLInputElement> = ref(null);
const resultsSize = ref(1000);
const scrollElement: Ref<HTMLDivElement> = ref(null);
const searchTermInterval: Ref<NodeJS.Timeout> = ref(null);
const searchTerm = ref('');
const localSearchTerm = ref('');

const connectionName = computed(() => getConnectionName(props.connection.uid));
const history: ComputedRef<HistoryRecord[]> = computed(() => (getHistoryByWorkspace(props.connection.uid) || []));
const filteredHistory = computed(() => history.value.filter(q => q.sql.toLowerCase().search(searchTerm.value.toLowerCase()) >= 0));

watch(searchTerm, () => {
   clearTimeout(searchTermInterval.value);

   searchTermInterval.value = setTimeout(() => {
      localSearchTerm.value = searchTerm.value;
   }, 200);
});

const copyQuery = (sql: string) => {
   navigator.clipboard.writeText(sql);
};

const deleteQuery = (query: HistoryRecord[]) => {
   deleteQueryFromHistory({
      workspace: props.connection.uid,
      ...query
   });
};

const resizeResults = () => {
   if (resultTable.value) {
      const el = tableWrapper.value.parentElement;

      if (el)
         resultsSize.value = el.offsetHeight - searchForm.value.offsetHeight;

      resultTable.value.updateWindow();
   }
};

const refreshScroller = () => resizeResults();
const closeModal = () => emit('close');

const highlightWord = (string: string) => {
   string = string.replaceAll('<', '&lt;').replaceAll('>', '&gt;');

   if (searchTerm.value) {
      const regexp = new RegExp(`(${searchTerm.value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
      return string.replace(regexp, '<span class="text-primary text-bold">$1</span>');
   }
   else
      return string;
};

const onKey = (e: KeyboardEvent) => {
   e.stopPropagation();
   if (e.key === 'Escape')
      closeModal();
};

window.addEventListener('keydown', onKey, { capture: true });

onUpdated(() => {
   if (table.value)
      refreshScroller();

   if (tableWrapper.value)
      scrollElement.value = tableWrapper.value;
});

onMounted(() => {
   resizeResults();
   window.addEventListener('resize', resizeResults);
});

onBeforeUnmount(() => {
   window.removeEventListener('keydown', onKey, { capture: true });
   window.removeEventListener('resize', resizeResults);
   clearInterval(searchTermInterval.value);
});
</script>

<style lang="scss" scoped>
.vscroll {
  height: 1000px;
  overflow: auto;
  overflow-anchor: none;
}

.tile {
  border-radius: $border-radius;
  display: flex;
  align-items: center;

  &:hover,
  &:focus {
    .tile-content {
      .tile-bottom-content {
        .tile-history-buttons {
          opacity: 1;
        }
      }
    }
  }

  .tile-icon {
    font-size: 1.2rem;
    margin-left: 0.3rem;
    width: 28px;
  }

  .tile-content {
    padding: 0.3rem;
    padding-left: 0.1rem;
    max-width: calc(100% - 30px);

    code {
      max-width: 100%;
      display: inline-block;
      font-size: 100%;
      // color: $primary-color;
      opacity: 0.8;
      font-weight: 600;
    }

    .tile-subtitle {
      opacity: 0.8;
    }

    .tile-bottom-content {
      display: flex;
      justify-content: space-between;

      .tile-history-buttons {
        opacity: 0;
        transition: opacity 0.2s;

        button {
          font-size: 0.7rem;
          height: 1rem;
          line-height: 1rem;
          display: inline-flex;
          align-items: center;
          justify-content: center;
        }
      }
    }
  }
}
</style>
