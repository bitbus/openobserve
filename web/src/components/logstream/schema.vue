<!-- Copyright 2023 Zinc Labs Inc.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<template>
  <q-card class="column full-height no-wrap" v-if="indexData.schema">
    <q-card-section class="q-ma-none">
      <div class="row items-center no-wrap">
        <div class="col">
          <div class="text-body1 text-bold" data-test="schema-title-text">
            {{ t("logStream.schemaHeader") }}
          </div>
        </div>
        <div class="col-auto">
          <q-btn v-close-popup="true" round flat icon="close" />
        </div>
      </div>
    </q-card-section>
    <q-separator />
    <q-card-section class="q-ma-none q-pa-none">
      <q-form ref="updateSettingsForm" @submit.prevent="onSubmit">
        <div
          v-if="loadingState"
          class="q-pt-md text-center q-w-md q-mx-lg"
          style="max-width: 450px"
        >
          <q-spinner-hourglass color="primary" size="lg" />
        </div>
        <div
          v-else-if="indexData.schema.length == 0"
          class="q-pt-md text-center q-w-md q-mx-lg"
          style="max-width: 450px"
        >
          No data available.
        </div>
        <div v-else
class="indexDetailsContainer" style="height: 100vh">
          <div class="title" data-test="schema-stream-title-text">
            {{ indexData.name }}
          </div>
          <div class="q-table__container q-table--cell-separator">
            <table class="q-table" data-test="schema-stream-meta-data-table">
              <thead>
                <tr>
                  <th v-if="store.state.zoConfig.show_stream_stats_doc_num">
                    {{ t("logStream.docsCount") }}
                  </th>
                  <th>{{ t("logStream.storageSize") }}</th>
                  <th v-if="isCloud !== 'true'">
                    {{ t("logStream.compressedSize") }}
                  </th>
                  <th v-if="store.state.zoConfig.show_stream_stats_doc_num">
                    {{ t("logStream.time") }}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr>
                  <td v-if="store.state.zoConfig.show_stream_stats_doc_num">
                    {{
                      parseInt(indexData.stats.doc_num).toLocaleString("en-US")
                    }}
                  </td>
                  <td>
                    {{ formatSizeFromMB(indexData.stats.storage_size) }}
                  </td>
                  <td v-if="isCloud !== 'true'">
                    {{ formatSizeFromMB(indexData.stats.compressed_size) }}
                  </td>
                  <td
                    v-if="store.state.zoConfig.show_stream_stats_doc_num"
                    class="text-center"
                  >
                    {{ indexData.stats.doc_time_min }}
                    <br />
                    to
                    <br />
                    {{ indexData.stats.doc_time_max }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <q-separator v-if="showDataRetention" class="q-mt-lg q-mb-lg" />

          <template v-if="showDataRetention">
            <div class="row flex items-center q-pb-xs q-mt-lg">
              <label class="q-pr-sm text-bold">Data Retention (in days)</label>
              <q-input
                data-test="stream-details-data-retention-input"
                v-model="dataRetentionDays"
                type="number"
                dense
                filled
                min="0"
                round
                class="q-mr-sm data-retention-input"
                :rules="[(val: any) => (!!val && val > 0) || 'Retention period must be at least 1 day']"
                @change="formDirtyFlag = true"
              ></q-input>
              <div>
                <span class="text-bold">Note:</span> Global data retention
                period is {{ store.state.zoConfig.data_retention_days }} days
              </div>
            </div>
          </template>
          <q-separator class="q-mt-lg q-mb-lg" />

          <div class="title" data-test="schema-log-stream-mapping-title-text">
            {{ t("logStream.mapping") }}
            <label
              v-show="indexData.defaultFts"
              class="warning-msg"
              style="font-weight: normal"
              >- Using default fts keys, as no fts keys are set for
              stream.</label
            >
          </div>

          <!-- Note: Drawer max-height to be dynamically calculated with JS -->
          <div
            class="q-table__container q-table--cell-separator"
            style="margin-bottom: 30px"
          >
            <q-table
              data-test="schema-log-stream-field-mapping-table"
              :rows="indexData.schema"
              :columns="columns"
              :row-key="(row) => 'tr_' + row.name"
              :filter="filterField"
              :filter-method="filterFieldFn"
              :pagination="{ rowsPerPage }"
              class="q-table"
              id="schemaFieldList"
              :rows-per-page-options="[]"
              :hidePagination="indexData.schema.length <= rowsPerPage"
              dense
            >
              <template #top-right>
                <q-input
                  data-test="schema-field-search-input"
                  v-model="filterField"
                  data-cy="schema-index-field-search-input"
                  filled
                  borderless
                  dense
                  clearable
                  debounce="1"
                  :placeholder="t('search.searchField')"
                >
                  <template #prepend>
                    <q-icon name="search" />
                  </template>
                </q-input>
              </template>
              <template v-slot:body-cell-delete="props">
                <q-td class="text-center">
                  <q-checkbox
                    v-if="
                      props.row.name !== store.state.zoConfig.timestamp_column
                    "
                    :data-test="`schema-stream-delete-${props.row.name}-field-fts-key-checkbox`"
                    v-model="props.row.delete"
                    size="sm"
                    @click="addDeleteField(props.row)"
                  />
                </q-td>
              </template>
              <template v-slot:body-cell-name="props">
                <q-td>{{ props.row.name }}</q-td>
              </template>
              <template v-slot:body-cell-type="props">
                <q-td>{{ props.row.type }}</q-td>
              </template>
              <template v-slot:body-cell-index_type="props">
                <q-td
                  data-test="schema-stream-index-select"
                  ><q-select
                    v-if="
                      props.row.name !== store.state.zoConfig.timestamp_column
                    "
                    v-model="props.row.index_type"
                    :options="streamIndexType"
                    :popup-content-style="{ textTransform: 'capitalize' }"
                    color="input-border"
                    bg-color="input-bg"
                    class="stream-schema-index-select q-py-xs fit q-pa-xs"
                    size="xs"
                    :option-disable="
                      (_option) => disableOptions(props.row, _option)
                    "
                    multiple
                    :max-values="2"
                    map-options
                    emit-value
                    autoclose
                    clearable
                    stack-label
                    outlined
                    filled
                    dense
                    style="width: 300px;"
                    @update:model-value="markFormDirty(props.row.name, 'fts')"
                  />
                </q-td>
              </template>
              <template v-slot:bottom-row>
                <q-tr
                  ><q-td colspan="100%">
                    <div v-if="indexData.schema.length > 0" class="q-mt-sm">
                      <q-btn
                        v-bind:disable="deleteFieldList.length == 0"
                        data-test="schema-delete-button"
                        class="q-my-sm text-bold btn-delete"
                        color="warning"
                        :label="t('logStream.delete')"
                        text-color="light-text"
                        padding="sm md"
                        no-caps
                        @click="confirmQueryModeChangeDialog = true"
                      />
                      <q-btn
                        v-bind:disable="!formDirtyFlag"
                        data-test="schema-update-settings-button"
                        :label="t('logStream.updateSettings')"
                        class="q-my-sm text-bold no-border q-ml-md float-right"
                        color="secondary"
                        padding="sm xl"
                        type="submit"
                        no-caps
                      />
                      <q-btn
                        v-close-popup="true"
                        data-test="schema-cancel-button"
                        class="q-my-sm text-bold float-right q-ml-md"
                        :label="t('logStream.cancel')"
                        text-color="light-text"
                        padding="sm md"
                        no-caps
                      /></div></q-td
                ></q-tr>
              </template>
            </q-table>
          </div>
        </div>
      </q-form>
      <br /><br /><br />
    </q-card-section>
  </q-card>
  <q-card v-else class="column q-pa-md full-height no-wrap">
    <h5>Wait while loading...</h5>
  </q-card>
  <ConfirmDialog
    title="Delete Action"
    :message="t('logStream.deleteActionMessage')"
    @update:ok="deleteFields()"
    @update:cancel="confirmQueryModeChangeDialog = false"
    v-model="confirmQueryModeChangeDialog"
  />
</template>

<script lang="ts">
// @ts-nocheck
import { computed, defineComponent, onBeforeMount, ref } from "vue";
import { useI18n } from "vue-i18n";
import { useStore } from "vuex";
import { useQuasar, date, format } from "quasar";
import streamService from "../../services/stream";
import segment from "../../services/segment_analytics";
import { formatSizeFromMB, getImageURL } from "@/utils/zincutils";
import config from "@/aws-exports";
import ConfirmDialog from "@/components/ConfirmDialog.vue";
import useStreams from "@/composables/useStreams";

const defaultValue: any = () => {
  return {
    name: "",
    schema: [],
    stats: {},
    defaultFts: false,
  };
};

export default defineComponent({
  name: "SchemaIndex",
  props: {
    // eslint-disable-next-line vue/require-default-prop
    modelValue: {
      type: Object,
      default: () => defaultValue(),
    },
  },
  components: {
    ConfirmDialog,
  },
  setup({ modelValue }) {
    const { t } = useI18n();
    const store = useStore();
    const q = useQuasar();
    const indexData: any = ref(defaultValue());
    const updateSettingsForm: any = ref(null);
    const isCloud = config.isCloud;
    const dataRetentionDays = ref(0);
    const deleteFieldList = ref([]);
    const confirmQueryModeChangeDialog = ref(false);
    const formDirtyFlag = ref(false);
    const loadingState = ref(true);
    const rowsPerPage = ref(250);
    const filterField = ref("");

    const streamIndexType = [
      { label: "Inverted Index", value: "fullTextSearchKey" },
      { label: "Bloom filter", value: "bloomFilterKey" },
      { label: "KeyValue partition", value: "keyPartition" },
      { label: "Hash partition (8 Buckets)", value: "hashPartition_8" },
      { label: "Hash partition (16 Buckets)", value: "hashPartition_16" },
      { label: "Hash partition (32 Buckets)", value: "hashPartition_32" },
      { label: "Hash partition (64 Buckets)", value: "hashPartition_64" },
      { label: "Hash partition (128 Buckets)", value: "hashPartition_128" },
    ];
    const { getStream } = useStreams();

    onBeforeMount(() => {
      dataRetentionDays.value = store.state.zoConfig.data_retention_days || 0;
    });

    const addDeleteField = (schema: any) => {
      if (schema.delete) {
        deleteFieldList.value.push(schema.name);
      } else {
        deleteFieldList.value = deleteFieldList.value.filter(
          (item) => item !== schema.name
        );
      }
    };

    const markFormDirty = () => {
      formDirtyFlag.value = true;
    };

    const deleteFields = async () => {
      loadingState.value = true;
      await streamService
        .deleteFields(
          store.state.selectedOrganization.identifier,
          indexData.value.name,
          deleteFieldList.value
        )
        .then(async (res) => {
          loadingState.value = false;
          if (res.data.code == 200) {
            q.notify({
              color: "positive",
              message: "Field(s) deleted successfully.",
              timeout: 2000,
            });
            confirmQueryModeChangeDialog.value = false;
            deleteFieldList.value = [];
            await getStream(
              indexData.value.name,
              indexData.value.stream_type,
              true,
              true
            );
            getSchema();
          } else {
            q.notify({
              color: "negative",
              message: res.data.message,
              timeout: 2000,
            });
          }
        })
        .catch((err: any) => {
          loadingState.value = false;
          console.log(err);
          q.notify({
            color: "negative",
            message: err.message,
            timeout: 2000,
          });
        });
    };

    const getSchema = async () => {
      const dismiss = q.notify({
        spinner: true,
        message: "Please wait while loading stats...",
      });

      await getStream(indexData.value.name, indexData.value.stream_type, true)
        .then((streamResponse) => {
          if (
            streamResponse.settings.full_text_search_keys.length == 0 &&
            (showFullTextSearchColumn.value || showPartitionColumn.value)
          ) {
            indexData.value.defaultFts = true;
          } else {
            indexData.value.defaultFts = false;
          }

          indexData.value.schema = streamResponse.schema || [];
          indexData.value.stats = JSON.parse(
            JSON.stringify(streamResponse.stats)
          );

          indexData.value.stats.doc_time_max = date.formatDate(
            parseInt(streamResponse.stats.doc_time_max) / 1000,
            "YYYY-MM-DDTHH:mm:ss:SSZ"
          );
          indexData.value.stats.doc_time_min = date.formatDate(
            parseInt(streamResponse.stats.doc_time_min) / 1000,
            "YYYY-MM-DDTHH:mm:ss:SSZ"
          );

          if (showDataRetention.value)
            dataRetentionDays.value =
              streamResponse.settings.data_retention ||
              store.state.zoConfig.data_retention_days;

          if (!streamResponse.schema) {
            loadingState.value = false;
            dismiss();
            return;
          }

          const fieldIndices = [];
          for (var property of streamResponse.schema) {
            if (
              (streamResponse.settings.full_text_search_keys.length > 0 &&
                streamResponse.settings.full_text_search_keys.includes(
                  property.name
                )) ||
              (streamResponse.settings.full_text_search_keys.length == 0 &&
                store.state.zoConfig.default_fts_keys.includes(property.name))
            ) {
              fieldIndices.push("fullTextSearchKey");
            }

            if (
              streamResponse.settings.bloom_filter_fields.length > 0 &&
              streamResponse.settings.bloom_filter_fields.includes(
                property.name
              )
            ) {
              fieldIndices.push("bloomFilterKey");
            }

            property["delete"] = false;

            if (
              streamResponse.settings.partition_keys &&
              Object.values(streamResponse.settings.partition_keys).some(
                (v) => !v.disabled && v.field === property.name
              )
            ) {
              const [level, partition] = Object.entries(
                streamResponse.settings.partition_keys
              ).find(([, partition]) => partition["field"] === property.name);

              property.level = level;

              if (partition.types === "value")
                fieldIndices.push("keyPartition");

              if (partition.types?.hash)
                fieldIndices.push(`hashPartition_${partition.types.hash}`);
            }

            property.index_type = [...fieldIndices];

            fieldIndices.length = 0;
          }

          loadingState.value = false;
          dismiss();
        })
        .catch((err) => {
          loadingState.value = false;
          console.log(err);
        });
    };

    const onSubmit = async () => {
      let settings = {
        partition_keys: [],
        full_text_search_keys: [],
        bloom_filter_fields: [],
      };

      if (showDataRetention.value && dataRetentionDays.value < 1) {
        q.notify({
          color: "negative",
          message:
            "Invalid Data Retention Period: Retention period must be at least 1 day.",
          timeout: 4000,
        });
        return;
      }

      if (showDataRetention.value) {
        settings["data_retention"] = Number(dataRetentionDays.value);
      }

      let added_part_keys = [];
      for (var property of indexData.value.schema) {
        property.index_type?.forEach((index: string) => {
          if (index === "fullTextSearchKey") {
            settings.full_text_search_keys.push(property.name);
          }

          if (property.level && index === "keyPartition") {
            settings.partition_keys.push({
              field: property.name,
              types: "value",
            });
          } else if (index === "keyPartition") {
            added_part_keys.push({
              field: property.name,
              types: "value",
            });
          }

          if (index?.includes("hashPartition")) {
            const [, buckets] = index.split("_");

            if (property.level) {
              settings.partition_keys.push({
                field: property.name,
                types: {
                  hash: Number(buckets),
                },
              });
            } else {
              added_part_keys.push({
                field: property.name,
                types: {
                  hash: Number(buckets),
                },
              });
            }
          }

          if (index === "bloomFilterKey") {
            settings.bloom_filter_fields.push(property.name);
          }
        });

        if (property.delete) {
          deleteFieldList.value.push(property.name);
        }
      }
      if (added_part_keys.length > 0) {
        settings.partition_keys =
          settings.partition_keys.concat(added_part_keys);
      }

      loadingState.value = true;
      await streamService
        .updateSettings(
          store.state.selectedOrganization.identifier,
          indexData.value.name,
          indexData.value.stream_type,
          settings
        )
        .then(async (res) => {
          loadingState.value = false;
          q.notify({
            color: "positive",
            message: "Stream settings updated successfully.",
            timeout: 2000,
          });

          await getStream(
            indexData.value.name,
            indexData.value.stream_type,
            true,
            true
          );

          segment.track("Button Click", {
            button: "Update Settings",
            user_org: store.state.selectedOrganization.identifier,
            user_id: store.state.userInfo.email,
            stream_name: indexData.value.name,
            page: "Schema Details",
          });
        })
        .catch((err: any) => {
          loadingState.value = false;
          q.notify({
            color: "negative",
            message: err.response.data.message,
            timeout: 2000,
          });
        });
    };

    const showPartitionColumn = computed(() => {
      return (
        isCloud != "true" && modelValue.stream_type !== "enrichment_tables"
      );
    });

    const showFullTextSearchColumn = computed(
      () => modelValue.stream_type !== "enrichment_tables"
    );

    const showDataRetention = computed(
      () =>
        !!(store.state.zoConfig.data_retention_days || false) &&
        modelValue.stream_type !== "enrichment_tables"
    );

    const disableOptions = (schema, option) => {
      let selectedHashPartition = "";

      let selectedIndices = "";

      for (let i = 0; i < (schema?.index_type || []).length; i++) {
        if (schema.index_type[i].includes("hashPartition")) {
          selectedHashPartition = schema.index_type[i];
        }
        selectedIndices += schema.index_type[i];
      }

      if (
        selectedIndices.includes("hashPartition") &&
        selectedHashPartition !== option.value &&
        (option.value.includes("hashPartition") ||
          option.value.includes("keyPartition"))
      )
        return true;

      if (
        selectedIndices.includes("keyPartition") &&
        option.value.includes("hashPartition")
      )
        return true;

      return false;
    };

    const filterFieldFn = (rows: any, terms: any) => {
      var filtered = [];
      if (terms != "") {
        terms = terms.toLowerCase();
        for (var i = 0; i < rows.length; i++) {
          if (rows[i]["name"].toLowerCase().includes(terms)) {
            filtered.push(rows[i]);
          }
        }
      }
      return filtered;
    };

    const columns = [
      {
        name: "delete",
        label: t("logStream.deleteActionLabel"),
        align: "center",
        sortable: false,
      },
      {
        name: "name",
        label: t("logStream.propertyName"),
        align: "center",
        sortable: true,
      },
      {
        name: "type",
        label: t("logStream.propertyType"),
        align: "center",
        sortable: true,
      },
      {
        name: "index_type",
        label: t("logStream.indexType"),
        align: "center",
        sortable: false,
      },
    ];

    return {
      t,
      q,
      store,
      isCloud,
      indexData,
      getSchema,
      onSubmit,
      updateSettingsForm,
      format,
      showPartitionColumn,
      showFullTextSearchColumn,
      getImageURL,
      dataRetentionDays,
      showDataRetention,
      formatSizeFromMB,
      addDeleteField,
      deleteFieldList,
      confirmQueryModeChangeDialog,
      deleteFields,
      markFormDirty,
      formDirtyFlag,
      streamIndexType,
      disableOptions,
      loadingState,
      filterFieldFn,
      rowsPerPage,
      filterField,
      columns,
    };
  },
  created() {
    if (this.modelValue && this.modelValue.name) {
      this.indexData.name = this.modelValue.name;
      this.indexData.schema = this.modelValue.schema;
      this.indexData.stream_type = this.modelValue.stream_type;

      this.getSchema();
    } else {
      this.loadingState.value = false;
    }
  },
});
</script>

<style lang="scss" scoped>
.indexDetailsContainer {
  padding: 1.25rem;
  width: 100%;

  .title {
    margin-bottom: 1rem;
    font-weight: 700;
  }

  .q-table {
    border: 1px solid $input-field-border-color;
  }

  .q-table {
    border-radius: 0.5rem;

    thead tr {
      height: 2.5rem;

      th {
        font-size: 0.875rem;
        font-weight: 700;
        height: 35px;
      }
    }

    tbody tr {
      height: 15px;

      td {
        font-size: 0.875rem;
        font-weight: 600;
        height: 25px;
        padding: 0px 5px;
      }
    }
  }

  .q-list {
    border-radius: 0 0 0.5rem 0.5rem;

    .q-item {
      height: 2.5rem;
      padding: 0;

      &__section {
        padding: 0.5rem 1rem;
        font-size: 0.875rem;

        &:not(:first-child) {
          border-left: 1px solid $input-field-border-color;
          align-items: flex-start;
          min-width: 29%;
        }
      }

      &.list-head {
        border: 1px solid $input-field-border-color;
        border-radius: 0.5rem 0.5rem 0 0;
        border-bottom: none;
      }

      &.list-item {
        border-right: 1px solid $input-field-border-color;
        border-left: 1px solid $input-field-border-color;

        &,
        &--side {
          font-weight: 600;
        }

        &:last-of-type {
          border-bottom: 1px solid $input-field-border-color;
          border-radius: 0 0 0.5rem 0.5rem;
        }
      }
    }
  }

  .data-retention-input {
    &.q-field {
      padding-bottom: 0 !important;
    }
    .q-field__bottom {
      padding: 8px 0;
    }
  }
}

// .sticky-buttons {
//   position: sticky;
//   bottom: 0px;
//   margin: 0 auto;
//   background-color: var(--q-accent);
//   box-shadow: 6px 6px 18px var(--q-accent);
//   justify-content: right;
//   width: 100%;
//   padding-right: 20px;
// }

// .btn-delete {
//   left: 20px;
//   position: absolute;
// }

// .sticky-table-header {
//   position: sticky;
//   top: 0px;
//   background: var(--q-accent);
//   z-index: 1;
// }

// .body--dark {
//   .sticky-table-header {
//     background: var(--q-dark);
//   }

//   .sticky-buttons {
//     background-color: var(--q-dark);
//     box-shadow: 6px 6px 18px var(--q-dark);
//   }
// }
</style>

<style lang="scss">
.stream-schema-index-select {
  .q-field__control {
    .q-field__control-container {
      span {
        overflow: hidden;
        text-overflow: ellipsis;
        text-wrap: nowrap;
        display: inline-block;
      }
    }
  }
}
</style>
