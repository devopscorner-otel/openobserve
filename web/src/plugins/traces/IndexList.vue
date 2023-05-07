<!-- Copyright 2022 Zinc Labs Inc. and Contributors

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http:www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License. 
-->

<template>
  <div class="column index-menu">
    <div class="index-table q-mt-xs">
      <q-table
        data-test="log-search-index-list-fields-table"
        v-model="searchObj.data.stream.selectedFields"
        :visible-columns="['name']"
        :rows="searchObj.data.stream.selectedStreamFields"
        row-key="name"
        :filter="searchObj.data.stream.filterField"
        :filter-method="filterFieldFn"
        :pagination="{ rowsPerPage: 10000 }"
        hide-header
        hide-bottom
        :wrap-cells="searchObj.meta.resultGrid.wrapCells"
        class="traces-field-table"
        id="tracesFieldList"
      >
        <template #body-cell-name="props">
          <q-tr :props="props">
            <q-td
              :props="props"
              class="field_list"
              :class="
                searchObj.data.stream.selectedFields.includes(props.row.name)
                  ? 'selected'
                  : ''
              "
            >
              <!-- TODO OK : Repeated code make seperate component to display field  -->
              <div
                v-if="props.row.ftsKey || !props.row.showValues"
                class="field-container flex content-center ellipsis q-pl-lg q-pr-sm"
                :title="props.row.name"
              >
                <div class="field_label ellipsis">
                  {{ props.row.name }}
                </div>
                <div class="field_overlay">
                  <q-icon
                    :name="'img:' + getImageURL('images/common/add_icon.svg')"
                    :data-test="`log-search-index-list-filter-${props.row.name}-field-btn`"
                    style="margin-right: 0.375rem"
                    size="1rem"
                    @click.stop="addToFilter(props.row.name)"
                  />
                </div>
              </div>
              <q-expansion-item
                v-else
                dense
                switch-toggle-side
                :label="props.row.name"
                expand-icon-class="field-expansion-icon"
                :expand-icon="
                  'img:' + getImageURL('images/common/down-solid.svg')
                "
                :expanded-icon="
                  'img:' + getImageURL('images/common/up-solid.svg')
                "
                @before-show="(event: any) => openFilterCreator(event, props.row)"
              >
                <template v-slot:header>
                  <div
                    class="flex content-center ellipsis"
                    :title="props.row.name"
                  >
                    <div class="field_label ellipsis">
                      {{ props.row.name }}
                    </div>
                    <div class="field_overlay">
                      <q-icon
                        :data-test="`log-search-index-list-filter-${props.row.name}-field-btn`"
                        :name="
                          'img:' + getImageURL('images/common/add_icon.svg')
                        "
                        style="margin-right: 0.375rem"
                        size="1rem"
                        @click.stop="addToFilter(props.row.name)"
                      />
                    </div>
                  </div>
                </template>
                <q-card>
                  <q-card-section class="q-pl-md q-pr-xs q-py-xs">
                    <div class="filter-values-container">
                      <div
                        v-show="fieldValues[props.row.name]?.isLoading"
                        class="q-pl-md q-py-xs"
                        style="height: 60px"
                      >
                        <q-inner-loading
                          size="xs"
                          :showing="fieldValues[props.row.name]?.isLoading"
                          label="Fetching values..."
                          label-style="font-size: 1.1em"
                        />
                      </div>
                      <div
                        v-show="
                          !fieldValues[props.row.name]?.values?.length &&
                          !fieldValues[props.row.name]?.isLoading
                        "
                        class="q-pl-md q-py-xs text-subtitle2"
                      >
                        No values found
                      </div>
                      <div
                        v-for="value in fieldValues[props.row.name]?.values ||
                        []"
                        :key="value.key"
                      >
                        <q-list dense>
                          <q-item tag="label" class="q-pr-none">
                            <div
                              class="flex row wrap justify-between"
                              style="width: calc(100% - 36px)"
                            >
                              <div
                                :title="value.key"
                                class="ellipsis q-pr-xs"
                                style="width: calc(100% - 50px)"
                              >
                                {{ value.key }}
                              </div>
                              <div
                                :title="value.count"
                                class="ellipsis text-right q-pr-sm"
                                style="width: 50px"
                              >
                                {{ value.count }}
                              </div>
                            </div>
                            <div class="flex row">
                              <q-icon
                                :name="
                                  'img:' +
                                  getImageURL('images/common/add_icon.svg')
                                "
                                class="q-mr-xs"
                                size="1rem"
                                @click="
                                  addSearchTerm(
                                    `${props.row.name}='${value.key}'`
                                  )
                                "
                              />
                              <q-icon
                                :name="
                                  'img:' +
                                  getImageURL('images/common/remove_icon.svg')
                                "
                                size="1rem"
                                @click="
                                  addSearchTerm(
                                    `${props.row.name}!='${value.key}'`
                                  )
                                "
                              />
                            </div>
                          </q-item>
                        </q-list>
                      </div>
                    </div>
                  </q-card-section>
                </q-card>
              </q-expansion-item>
            </q-td>
          </q-tr>
        </template>
        <template #top-right>
          <q-input
            data-test="log-search-index-list-field-search-input"
            v-model="searchObj.data.stream.filterField"
            data-cy="index-field-search-input"
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
      </q-table>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, type Ref } from "vue";
import { useI18n } from "vue-i18n";
import { useStore } from "vuex";
import { useQuasar } from "quasar";
import { useRouter } from "vue-router";
import useTraces from "../../composables/useTraces";
import { getImageURL } from "../../utils/zincutils";
import streamService from "../../services/stream";
import { getConsumableDateTime } from "@/utils/commons";

export default defineComponent({
  name: "ComponentSearchIndexSelect",
  setup() {
    const store = useStore();
    const router = useRouter();
    const { t } = useI18n();
    const $q = useQuasar();
    const { searchObj, updatedLocalLogFilterField } = useTraces();
    const streamOptions: any = ref(searchObj.data.stream.streamLists);
    const fieldValues: Ref<{
      [key: string | number]: {
        isLoading: boolean;
        values: { key: string; count: string }[];
      };
    }> = ref({});

    const filterStreamFn = (val: string, update: any) => {
      update(() => {
        streamOptions.value = searchObj.data.stream.streamLists;
        const needle = val.toLowerCase();
        streamOptions.value = streamOptions.value.filter(
          (v: any) => v.label.toLowerCase().indexOf(needle) > -1
        );
      });
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

    const addToFilter = (field: any) => {
      searchObj.data.stream.addToFilter = field;
    };

    function clickFieldFn(row: { name: never }, pageIndex: number) {
      if (searchObj.data.stream.selectedFields.includes(row.name)) {
        searchObj.data.stream.selectedFields =
          searchObj.data.stream.selectedFields.filter(
            (v: any) => v !== row.name
          );
      } else {
        searchObj.data.stream.selectedFields.push(row.name);
      }
      searchObj.organizationIdetifier =
        store.state.selectedOrganization.identifier;
      updatedLocalLogFilterField();
    }

    const openFilterCreator = (event: any, { name, ftsKey }: any) => {
      if (ftsKey) {
        event.stopPropagation();
        event.preventDefault();
        return;
      }
      const timestamps = getConsumableDateTime(searchObj.data.datetime);
      const startISOTimestamp: any =
        new Date(timestamps.start_time.toISOString()).getTime() * 1000;
      const endISOTimestamp: any =
        new Date(timestamps.end_time.toISOString()).getTime() * 1000;

      fieldValues.value[name] = {
        isLoading: true,
        values: [],
      };
      streamService
        .fieldValues({
          org_identifier: store.state.selectedOrganization.identifier,
          stream_name: searchObj.data.stream.selectedStream.value,
          start_time: startISOTimestamp,
          end_time: endISOTimestamp,
          fields: [name],
          size: 10,
          type: "traces",
        })
        .then((res: any) => {
          if (res.data.hits.length) {
            fieldValues.value[name]["values"] = res.data.hits
              .find((field: any) => field.field === name)
              .values.map((value: any) => {
                return {
                  key: value.key ? value.key : "null",
                  count: formatNumberWithPrefix(value.num),
                };
              });
          }
        })
        .catch(() => {
          $q.notify({
            type: "negative",
            message: `Error while fetching values for ${name}`,
          });
        })
        .finally(() => {
          fieldValues.value[name]["isLoading"] = false;
        });
    };

    function formatNumberWithPrefix(number: number) {
      if (number >= 1000000000) {
        return (number / 1000000000).toFixed(1) + "B";
      } else if (number >= 1000000) {
        return (number / 1000000).toFixed(1) + "M";
      } else if (number >= 1000) {
        return (number / 1000).toFixed(1) + "K";
      } else {
        return number.toString();
      }
    }

    const addSearchTerm = (term: string) => {
      // searchObj.meta.showDetailTab = false;
      searchObj.data.stream.addToFilter = term;
    };

    return {
      t,
      store,
      router,
      searchObj,
      streamOptions,
      filterFieldFn,
      addToFilter,
      clickFieldFn,
      getImageURL,
      filterStreamFn,
      openFilterCreator,
      addSearchTerm,
      fieldValues,
    };
  },
});
</script>

<style lang="scss" scoped>
.traces-field-table {
  height: calc(100vh - 172px) !important;
}
.q-menu {
  box-shadow: 0px 3px 15px rgba(0, 0, 0, 0.1);
  transform: translateY(0.5rem);
  border-radius: 0px;

  .q-virtual-scroll__content {
    padding: 0.5rem;
  }
}
.index-menu {
  width: 100%;

  .q-field {
    &__control {
      height: 35px;
      padding: 0px 5px;
      min-height: auto !important;

      &-container {
        padding-top: 0px !important;
      }
    }
    &__native :first-of-type {
      padding-top: 0.25rem;
    }
  }

  .index-table {
    width: 100%;
    // border: 1px solid rgba(0, 0, 0, 0.02);

    .q-table {
      display: table;
      table-layout: fixed !important;
    }
    tr {
      margin-bottom: 1px;
    }
    tbody,
    tr,
    td {
      width: 100%;
      display: block;
      height: fit-content;
      overflow: hidden;
    }

    .q-table__control,
    label.q-field {
      width: 100%;
    }
    .q-table thead tr,
    .q-table tbody td {
      height: auto;
    }

    .q-table__top {
      border-bottom: unset;
    }
  }
  .traces-field-table {
    width: 100%;
  }

  .field_list {
    padding: 0px;
    margin-bottom: 0.125rem;
    position: relative;
    overflow: visible;
    cursor: default;

    .field_label {
      pointer-events: none;
      font-size: 0.825rem;
      position: relative;
      display: inline;
      z-index: 2;
      left: 0;
      // text-transform: capitalize;
    }

    .field-container {
      height: 25px;
    }

    .field_overlay {
      position: absolute;
      height: 100%;
      right: 0;
      top: 0;
      z-index: 5;
      background-color: #e8e8e8;
      padding: 0 6px;
      visibility: hidden;
      display: flex;
      align-items: center;

      .q-icon {
        cursor: pointer;
        opacity: 0;
        margin: 0 1px;
      }
    }

    &.selected {
      .field_overlay {
        background-color: rgba(89, 96, 178, 0.3);

        .field_icons {
          opacity: 0;
        }
      }
    }
    &:hover {
      .field-container {
        background-color: #e8e8e8;
      }
    }
  }
}

.q-item {
  color: $dark-page;
  min-height: 1.3rem;
  padding: 5px 10px;

  &__label {
    font-size: 0.75rem;
  }

  &.q-manual-focusable--focused > .q-focus-helper {
    background: none !important;
    opacity: 0.3 !important;
  }

  &.q-manual-focusable--focused > .q-focus-helper,
  &--active {
    background-color: $selected-list-bg !important;
  }

  &.q-manual-focusable--focused > .q-focus-helper,
  &:hover,
  &--active {
    color: $primary;
  }
}
.q-field--dense .q-field__before,
.q-field--dense .q-field__prepend {
  padding: 0px 0px 0px 0px;
  height: auto;
  line-height: auto;
}
.q-field__native,
.q-field__input {
  padding: 0px 0px 0px 0px;
}

.q-field--dense .q-field__label {
  top: 5px;
}
.q-field--dense .q-field__control,
.q-field--dense .q-field__marginal {
  height: 34px;
}
</style>

<style lang="scss">
.index-table {
  .q-table {
    width: 100%;
    table-layout: fixed;

    .q-expansion-item {
      .q-item {
        display: flex;
        align-items: center;
        padding: 0;
        height: 25px !important;
        min-height: 25px !important;
      }
      .q-item__section--avatar {
        min-width: 12px;
        max-width: 12px;
        margin-right: 8px;
      }

      .filter-values-container {
        .q-item {
          padding-left: 4px;

          .q-focus-helper {
            background: none !important;
          }
        }
      }
      .q-item-type {
        &:hover {
          .field_overlay {
            visibility: visible;

            .q-icon {
              opacity: 1;
            }
          }
        }
      }
      .field-expansion-icon {
        img {
          width: 12px;
          height: 12px;
        }
      }
    }

    .field-container {
      &:hover {
        .field_overlay {
          visibility: visible;

          .q-icon {
            opacity: 1;
          }
        }
      }
    }

    .field_list {
      &.selected {
        .q-expansion-item {
          background-color: rgba(89, 96, 178, 0.3);
        }

        .field_overlay {
          background-color: #ffffff;
        }
      }
    }
  }
}
</style>