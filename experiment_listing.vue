<template>
  <div>
    <v-container
      fluid
      class="pr-0 pt-0 pr-sm-3 pl-0 pl-sm-3"
    >
      <CTXDialog
        :value="dialog"
        :config="dialog_config"
      >
        <template #creation>
          <ExperimentCreationForm
            v-model="new_gp"
            @confirm="create_gp"
          />
        </template>
      </CTXDialog>
      <v-row class="mt-0">
        <v-col class="pt-0">
          <div class="d-flex justify-space-between mt-1 align-center">
            <h1 class="heading heading--h1">
              Experiments
            </h1>
            <v-hover v-slot="{ hover }">
              <v-btn
                v-if="$user.can('insert_group_operation')"
                depressed
                class="btn-create blue white--text"
                :class="{ 'darken-2': hover}"
                :ripple="false"
                @click="(dialog = true), (creation = true)"
              >
                <v-icon small>
                  mdi-plus
                </v-icon>
                <span>Add experiment</span>
              </v-btn>
            </v-hover>
          </div>
        </v-col>
      </v-row>
      <v-row>
        <v-col class="pt-0 pb-0 experiment-smart-filter">
          <SmartFilter
            type="backend"
            :cells="cells"
            :counters="filter_counters"
            view_count="all"
            @filtered="on_filtered"
            @aggregation_counts_updated="on_aggregation_counts_updated"
          >
            <template #reagents="{ update_explicit_filter }">
              <SubstancePicker
                class="picker"
                :width="300"
                :height="200"
                :search_only="true"
                @picked="update_explicit_filter"
              />
            </template>
            <template #solvents="{ update_explicit_filter }">
              <SubstancePicker
                class="picker"
                :width="300"
                :height="200"
                :search_only="true"
                @picked="update_explicit_filter"
              />
            </template>
            <template #products="{ update_explicit_filter }">
              <SubstancePicker
                class="picker"
                :width="300"
                :height="200"
                :search_only="true"
                @picked="update_explicit_filter"
              />
            </template>
          </SmartFilter>
        </v-col>
      </v-row>
      <v-row>
        <v-col class="pr-0 pr-sm-3 pl-0 pl-sm-3 pt-0 pt-sm-3">
          <v-data-table
            :headers="headers"
            :items="experiment_list"
            class="mb-1 transparent c-table"
            :footer-props="table_footer_props"
            :server-items-length="item_count_total"
            :options.sync="pagination"
            :sort-by.sync="sort_by"
            :sort-desc.sync="sort_desc"
            :loading="$apollo.queries.experiment_list.loading"
            @pagination="on_pagination"
          >
            <template #item.name="{ item }">
              <div class="experiment-name-container">
                <v-btn
                  text
                  small
                  class="name-link mr-2"
                >
                  <router-link :to="{ name: 'experiment', params: { id: item.id} }">
                    {{ item.meta.name }}
                  </router-link>
                </v-btn>
              </div>
            </template>
            <template #item.status="{ item }">
              <p
                class="badge mw-100 mb-0 mr-2"
                :class="item.status"
              >
                {{ item.status }}
              </p>
            </template>
            <template #item.assignee="{ item }">
              <span
                v-if="item.tasks.length > 0"
                class="assignee"
              >
                {{ item.tasks[0].assignee_user?.last_name ?
                  item.tasks[0].assignee_user.last_name : "" }}
                {{ item.tasks[0].assignee_user?.first_name ?
                  item.tasks[0].assignee_user.first_name : "" }}
              </span>
              <span
                v-else
              >
                -
              </span>
            </template>
            <template #item.operations="{ item }">
              <div class="mw-100">
                <span
                  class="operations__amount"
                >
                  {{ item.operations.length === 0 ? "-" : item.operations.length }}
                </span>
              </div>
            </template>
            <template #item.reagents="{ item }">
              <div class="related-compound-container">
                <div
                  v-for="reagent_p_cpd in item.reagents"
                  :key="reagent_p_cpd.compound.id"
                  class="related-compound"
                >
                  <div class="related-compound-content">
                    <v-tooltip
                      top
                      :disabled="reagent_p_cpd.compound.name.length < 25"
                    >
                      <template #activator="{ on }">
                        <v-btn
                          text
                          small
                          class="pl-0 pr-0 name-link"
                          v-on="on"
                        >
                          <router-link :to="{ name: 'compound', params: { id: reagent_p_cpd.compound.id} }">
                            {{ reagent_p_cpd.compound.name }}
                          </router-link>
                        </v-btn>
                      </template>
                      {{ reagent_p_cpd.compound.name }}
                    </v-tooltip>
                  </div>
                  <div
                    v-if="reagent_p_cpd.ss"
                    class="d-flex align-center"
                  >
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-btn
                          icon
                          small
                          v-on="on"
                        >
                          <v-icon
                            :color="reagent_p_cpd.checked_in_ss?.operation_in?.status === 'completed' ? '#219653' : '#555'"
                            class="location-icon"
                            size="16px"
                          >
                            mdi-location-exit
                          </v-icon>
                        </v-btn>
                      </template>
                      <!-- TODO: CTX-1337 replace with list of checked-in samples -->
                      Samples selected
                    </v-tooltip>
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-icon
                          v-if="reagent_p_cpd.checked_in_ss?.operation_in?.status === 'completed'"
                          color="#219653"
                          size="16px"
                          v-on="on"
                        >
                          mdi-check
                        </v-icon>
                      </template>
                      Check in completed
                    </v-tooltip>
                  </div>
                </div>
              </div>
            </template>
            <template #item.solvents="{ item }">
              <div class="related-compound-container">
                <div
                  v-for="solvent_p_cpd in item.solvents"
                  :key="solvent_p_cpd.compound.id"
                  class="related-compound"
                >
                  <div class="related-compound-content">
                    <v-tooltip
                      top
                      :disabled="solvent_p_cpd.compound.name.length < 25"
                    >
                      <template #activator="{ on }">
                        <v-btn
                          text
                          small
                          class="pl-0 pr-0 name-link"
                          v-on="on"
                        >
                          <router-link :to="{ name: 'compound', params: { id: solvent_p_cpd.compound.id} }">
                            {{ solvent_p_cpd.compound.name }}
                          </router-link>
                        </v-btn>
                      </template>
                      {{ solvent_p_cpd.compound.name }}
                    </v-tooltip>
                  </div>
                  <div
                    v-if="solvent_p_cpd.checked_in_ss?.operation_in?.status"
                    class="d-flex align-center"
                  >
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-btn
                          icon
                          small
                          v-on="on"
                        >
                          <v-icon
                            :color="solvent_p_cpd.checked_in_ss?.operation_in?.status === 'completed' ? '#219653' : '#555'"
                            class="location-icon"
                            size="16px"
                          >
                            mdi-location-exit
                          </v-icon>
                        </v-btn>
                      </template>
                      <!-- TODO: CTX-1337 replace with list of checked-in samples -->
                      Samples selected
                    </v-tooltip>
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-icon
                          v-if="solvent_p_cpd.checked_in_ss?.operation_in?.status === 'completed'"
                          color="#219653"
                          size="16px"
                          v-on="on"
                        >
                          mdi-check
                        </v-icon>
                      </template>
                      Check in completed
                    </v-tooltip>
                  </div>
                </div>
              </div>
            </template>
            <template #item.products="{ item }">
              <div class="related-compound-container">
                <div
                  v-for="product_p_cpd in item.products"
                  :key="product_p_cpd.compound.id"
                  class="related-compound"
                >
                  <div class="related-compound-content">
                    <v-tooltip
                      top
                      :disabled="product_p_cpd.compound.name.length < 28"
                    >
                      <template #activator="{ on }">
                        <v-btn
                          text
                          small
                          class="pl-0 pr-0 name-link"
                          v-on="on"
                        >
                          <router-link :to="{ name: 'compound', params: { id: product_p_cpd.compound.id} }">
                            {{ product_p_cpd.compound.name }}
                          </router-link>
                        </v-btn>
                      </template>
                      {{ product_p_cpd.compound.name }}
                    </v-tooltip>
                  </div>
                  <div
                    v-if="product_p_cpd.checked_in_ss?.operation_in?.status"
                    class="d-flex align-center"
                  >
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-btn
                          icon
                          small
                          v-on="on"
                        >
                          <v-icon
                            :color="product_p_cpd.checked_in_ss?.operation_in?.status === 'completed' ? '#219653' : '#555'"
                            class="location-icon"
                            size="16px"
                          >
                            mdi-location-exit
                          </v-icon>
                        </v-btn>
                      </template>
                      <!-- TODO: CTX-1337 replace with list of checked-in samples -->
                      Samples selected
                    </v-tooltip>
                    <v-tooltip top>
                      <template #activator="{ on }">
                        <v-icon
                          v-if="product_p_cpd.checked_in_ss?.operation_in?.status === 'completed'"
                          color="#219653"
                          size="16px"
                          v-on="on"
                        >
                          mdi-check
                        </v-icon>
                      </template>
                      Check in completedeted
                    </v-tooltip>
                  </div>
                </div>
              </div>
            </template>
            <template #item.samples="{ item }">
              <div
                :class="sample_list_exp_id === item.id ? 'sample-list all' : 'sample-list'"
              >
                <div
                  v-for="(sample, i) in item.samples_in_experiment"
                  :key="i"
                  class="mr-1"
                >
                  <div
                    v-if="i < 4 || sample_list_exp_id"
                  >
                    <v-btn
                      text
                      small
                      class="pl-0 pr-0 name-link"
                    >
                      <router-link :to="{ name: 'sample', params: { id: sample.id} }">
                        {{ sample.name }}
                      </router-link>
                    </v-btn>
                    <span
                      v-if="sample_list_exp_id !== item.id"
                      class="ellipses"
                      @click="sample_list_exp_id = item.id"
                    >
                      {{ i === 3 && item.samples_in_experiment.length > 3 ? '...' : ' ' }}
                    </span>
                    <span
                      v-if="sample_list_exp_id === item.id"
                      class="ellipses"
                      @click="sample_list_exp_id = null"
                    >
                      {{ item.samples_in_experiment.length - 1 === i ? '<' : ' ' }}
                    </span>
                  </div>
                </div>
              </div>
            </template>
            <template #item.actions="{ item }">
              <TableActions
                :item="item"
                :user_can="user_can"
                :crud_routes="{
                  view: `/experiment/${item.id}`,
                }"
                :crud_callbacks="{
                  update: () => on_edit_task_experiment(item),
                  clone: () => on_clone_experiment(item),
                  delete: () => on_delete_experiment(item),
                }"
                :disabled="{
                  delete:
                    (['in-progress', 'completed'].includes(item.tasks[0]?.status) || some_ops_w_sss(item))
                      ? 'Live/completed experiment cannot be deleted'
                      : false,
                }"
                :hide="{ edit: true }"
              />
            </template>

            <template #item.parent_experiment_go="{ item }">
              <v-btn
                v-if="item.meta?.parent_experiment_go?.length > 0"
                text
                small
                class="pl-0 pr-0 name-link"
                @click="handle_parent_experiment_go_click(item)"
              >
                <router-link :to="{ name: 'experiment', params: { id: item.meta?.parent_experiment_go[0] } }">
                  {{ item.meta?.parent_experiment_go[0] }}
                </router-link>
              </v-btn>
              <span
                v-else
                class="task__absent"
              >
                -
              </span>
            </template>
          </v-data-table>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import {
  uniqBy, pick, cloneDeep, set,
  isEqual, isArray,
} from "lodash";
import SmartFilter from "@lumiprobejs/smart-filter";
import { FALSE_EXPRESSION } from "@lumiprobejs/smart-filter/constants";
import TableActions from "@/components/table_actions.vue";
import CTXDialog from "@/components/dialog";
import SubstancePicker from "../substance/picker.vue";
import ExperimentCreationForm from "./experiment_creation_form.vue";

const DEFAULT_QUERY_CONDITION = Object.freeze({
  operation_type_key: { _eq: "synthetic" },
});

export default {
  name: "ExperimentListing",
  components: {
    SmartFilter,
    TableActions,
    CTXDialog,
    SubstancePicker,
    ExperimentCreationForm,
  },
  props: {},
  apollo: {
    experiment_list: {
      query: require("@/graphql/chemistry/experiment/list_ssp_with_filters.gql"),
      fetchPolicy: "no-cache",
      variables() {
        return {
          limit: this.pagination.itemsPerPage,
          offset: (this.pagination.page - 1) * this.pagination.itemsPerPage,
          order_by: this.order_by,
          ...this.filter_conditions,
        };
      },
      update(data) {
        this.data_from_apollo = data;
        this.item_count_total = data.total.aggregate.count;
        const length = data.group_operation?.length ?? 0;
        if (!length) return [];
        /**
         * Формирую эксперимент для вывода в UI, добавив к нему по новому полю с компаундами по compound_role
         * Также добавляю в эксперимент поле samples_in_experiment, в котором получаю все образцы, что задействованы в эксперименте
         * */
        return data.group_operation.map((g_op) => {
          const sss_in = g_op.operations.flatMap((op) => op.sample_states_in);
          const sss_out = g_op.operations.flatMap((op) => op.sample_states_out);
          const all_sss = [...sss_in, ...sss_out];
          const p_cpds_w_ss = g_op.protocol?.protocol_compounds
            .map((p_cpd) => ({
              ...p_cpd,
              ...(["reagent", "solvent"].includes(p_cpd.compound_role) && {
                checked_in_ss: sss_in.find((ss) => ss.sample.compound_id === p_cpd.compound_id),
              }),
            })) ?? [];
          const samples_in_experiment = uniqBy(all_sss.map((ss) => ({ id: ss.sample.id, name: ss.sample.name })), (sample) => sample.id);
          return {
            ...g_op,
            reagents: p_cpds_w_ss.filter(({ compound_role }) => compound_role === "reagent"),
            solvents: p_cpds_w_ss.filter(({ compound_role }) => compound_role === "solvent"),
            products: p_cpds_w_ss.filter(({ compound_role }) => compound_role === "product"),
            samples_in_experiment,
          };
        });
      },
    },
  },
  data() {
    return {
      sample_list_exp_id: null,
      default_gp_generator: () => ({
        operation_type_key: "synthetic",
        meta: {
          name: "",
        },
        protocol: null,
        protocol_id: null,
        operations: [],
        tasks: [
          {
            title: "New experiment",
            description: "",
            deadline: null,
            assignee: null,
          },
        ],
      }),
      new_gp: {
        operation_type_key: "synthetic",
        meta: {
          name: "",
        },
        protocol: null,
        protocol_id: null,
        operations: [],
        tasks: [
          {
            title: "New experiment",
            description: "",
            deadline: null,
            assignee: null,
          },
        ],
      },
      creation: false,
      cloning: false,
      deletion: false,
      pagination: {},
      sort_by: "id",
      sort_desc: true,
      headers: [
        {
          text: "Name",
          value: "name",
          sortable: false,
          align: "center",
        },
        {
          text: "Status",
          value: "status",
          align: "center",
        },
        {
          text: "assignee",
          value: "assignee",
          sortable: false,
          align: "center",
        },
        {
          text: "Operations",
          value: "operations",
          sortable: false,
          align: "center",
        },
        {
          text: "Reagents",
          value: "reagents",
          sortable: false,
          align: "center",
        },
        {
          text: "Solvents",
          value: "solvents",
          sortable: false,
          align: "center",
        },
        {
          text: "Products",
          value: "products",
          sortable: false,
          align: "center",
        },
        {
          text: "Samples",
          value: "samples",
          sortable: false,
          align: "center",
        },
        {
          text: "",
          value: "actions",
          sortable: false,
        },
      ],
      cells: [
        {
          text: "Name",
          value: "name", // meta.name
          type: "input",
          json_value_type: "string",
          value_type: "json",
          callback: (strs) => ({
            _or: strs.map((str) => ({ meta: { _cast: { String: { _iregex: `\\w*"name":\\s*"\\w*${str}\\w*` } } } })),
          }),
        },
        {
          text: "Status",
          value: "status",
          type: "select",
          value_type: "string",
          options: [
            { label: "Planned", value: "planned" },
            { label: "In progress", value: "in-progress" },
            { label: "Completed", value: "completed" },
          ],
        },
        {
          text: "Reagents",
          value: "reagents", // protocol.protocol_compounds.protocol_compound.compound.id
          type: "custom",
          value_type: "number",
          slot_name: "reagents",
          callback: (query) => {
            if (!query) return null;
            if (isArray(query)) {
              if (!query.every(Number)) {
                return set({}, "protocol.protocol_compounds._and", [
                  { compound_role: { _eq: "reagent" } },
                  { compound: { _or: query.map((q) => ({ name: { _ilike: `%${q}%` } })) } },
                ]);
              }
              return set({}, "protocol.protocol_compounds._and", [
                { compound_role: { _eq: "reagent" } },
                {
                  compound: {
                    _or: query.map((q) => ({ id: { _eq: q } })),
                  },
                },
              ]);
            }
            if (Number.isNaN(Number(query))) return null;
            return set({}, "protocol.protocol_compounds._and", [
              { compound_role: { _eq: "reagent" } },
              {
                compound: {
                  _or: { id: { _eq: Number(query) } },
                },
              },
            ]);
          },
        },
        {
          text: "Solvents",
          value: "solvents", // protocol.protocol_compounds.protocol_compound.compound.id
          type: "custom",
          value_type: "number",
          slot_name: "solvents",
          callback: (query) => {
            if (!query) return null;
            if (isArray(query)) {
              if (!query.every(Number)) {
                return set({}, "protocol.protocol_compounds._and", [
                  { compound_role: { _eq: "solvent" } },
                  { compound: { _or: query.map((q) => ({ name: { _ilike: `%${q}%` } })) } },
                ]);
              }
              return set({}, "protocol.protocol_compounds._and", [
                { compound_role: { _eq: "solvent" } },
                {
                  compound: {
                    _or: query.map((q) => ({ id: { _eq: q } })),
                  },
                },
              ]);
            }
            if (Number.isNaN(Number(query))) return null;
            return set({}, "protocol.protocol_compounds._and", [
              { compound_role: { _eq: "solvent" } },
              {
                compound: {
                  _or: { id: { _eq: Number(query) } },
                },
              },
            ]);
          },
        },
        {
          text: "Products",
          value: "products", // protocol.protocol_compounds.protocol_compound.compound.id
          type: "custom",
          value_type: "number",
          slot_name: "products",
          callback: (query) => {
            if (!query) return null;
            if (isArray(query)) {
              if (!query.every(Number)) {
                return set({}, "protocol.protocol_compounds._and", [
                  { compound_role: { _eq: "product" } },
                  { compound: { _or: query.map((q) => ({ name: { _ilike: `%${q}%` } })) } },
                ]);
              }
              return set({}, "protocol.protocol_compounds._and", [
                { compound_role: { _eq: "product" } },
                {
                  compound: {
                    _or: query.map((q) => ({ id: { _eq: q } })),
                  },
                },
              ]);
            }
            if (Number.isNaN(Number(query))) return null;
            return set({}, "protocol.protocol_compounds._and", [
              { compound_role: { _eq: "product" } },
              {
                compound: {
                  _or: { id: { _eq: Number(query) } },
                },
              },
            ]);
          },
        },
        {
          text: "Task assignee",
          value: "tasks.assignee_user",
          type: "input",
          value_type: "string",
          callback: (query) => {
            if (!query) return null;
            return this.get_user_condition(query, "assignee_user");
          },
        },
      ],
      item_to_delete: null,
      dialog: false,
      user_can: {
        view: true,
        update: this.$user.can("update_group_operation"),
        clone: this.$user.can("insert_group_operation"),
        delete: this.$user.can("delete_group_operation"),
      },
      filter_conditions: {
        where: DEFAULT_QUERY_CONDITION,
        total: DEFAULT_QUERY_CONDITION,
        name: FALSE_EXPRESSION,
        reagents: FALSE_EXPRESSION,
        solvents: FALSE_EXPRESSION,
        products: FALSE_EXPRESSION,
        samples: FALSE_EXPRESSION,
        status: FALSE_EXPRESSION,
        operation_type_key: FALSE_EXPRESSION,
        tasks_assignee_user: FALSE_EXPRESSION,
      },
      filter_counters_template: {
        total: null,
        name: null,
        status: null,
        reagents: null,
        solvents: null,
        products: null,
        samples: null,
        operation_type_key: null,
        tasks_assignee_user: null,
      },
      data_from_apollo: null,
      item_count_total: 0,
    };
  },
  computed: {
    dialog_config() {
      if (this.creation) {
        return {
          title: "Creating an experiment",
          body_slot: "creation",
          on_update: this.create_gp,
        };
      }

      if (this.cloning) {
        return {
          title: "Cloning experiment",
          body_slot: "creation",
          on_update: this.create_gp,
        };
      }

      if (this.deletion) {
        return {
          title: "Deleting experiment",
          body_text: `Are you sure you want to delete experiment #${this.item_to_delete.id}?`,
          confirm_text: "Delete",
          tone: "error",
          on_update: this.delete,
        };
      }
      return {};
    },
    order_by() {
      const desc_mapper = {
        true: "desc",
        false: "asc",
        undefined: "desc",
      };

      return { [this.sort_by ?? "id"]: desc_mapper[this.sort_desc] };
    },
    table_footer_props() {
      return {
        "items-per-page-options": [10, 25, 50, 100],
        "show-first-last-page": true,
        options: this.pagination,
      };
    },
    filter_counters() {
      const counters = { ...this.filter_counters_template };
      if (!this.experiment_list || this.experiment_list.length === 0) return counters;
      for (const counters_data_key in counters) {
        counters[counters_data_key] = this.data_from_apollo[counters_data_key].aggregate.count;
      }
      return counters;
    },
  },
  methods: {
    handle_parent_experiment_go_click(item) {
      this.$router.push({ name: "experiment", params: { id: item.meta?.parent_experiment_go[0] } });
    },
    get_user_condition(query, user_task_relation) {
      return {
        tasks: {
          [user_task_relation]: {
            _or: query.flatMap((q) => [
              { username: { _ilike: `%${q}%` } },
              { first_name: { _ilike: `%${q}%` } },
              { last_name: { _ilike: `%${q}%` } },
            ]),
          },
        },
      };
    },
    some_ops_w_sss(item) {
      return item.operations.some((op) => (op.sample_states_in.length > 0)); // only sample_states_in is checked
    },
    async create_gp(confirmed) {
      if (confirmed) {
        const {
          operations, meta = {}, protocol, operation_type_key,
        } = this.new_gp;
        const group_operation = {
          meta: { name: meta.name ?? "New experiment" },
          ...pick(this.new_gp, ["operation_type_key", "protocol_id"]),
        };
        const tasks = [{
          ...pick(this.new_gp.tasks[0], ["deadline", "description", "meta", "status", "title", "assignee"]),
        }];

        if (tasks[0].assignee === -1) {
          tasks[0].assignee = null;
        }

        const new_operations = operations?.length > 0
          ? operations.map((op) => ({ operation_type_key: op.operation_type_key }))
          : [{ operation_type_key }];

        // add sample_states_out if experiment has configured protocol_compounds w/ role "product"
        if (protocol?.protocol_compounds?.length > 0) {
          for (const op of new_operations) {
            op.sample_states_out = {
              data: protocol.protocol_compounds
                .filter((p_cpd) => p_cpd.compound_role === "product")
                .map((p_cpd) => ({
                  status: "future",
                  sample: {
                    data: {
                      status: "planned",
                      compound_id: p_cpd.compound_id,
                    },
                  },
                })),
            };
          }
        }
        try {
          const new_experiment = {
            ...group_operation,
            tasks: {
              data: tasks,
            },
            operations: {
              data: new_operations,
            },
          };

          delete new_experiment.assigned_to;
          delete new_experiment.__typename;
          delete new_experiment.operation_type;
          delete new_experiment.id;

          if (protocol?.id) {
            new_experiment.protocol_id = protocol.id;
          } else {
            delete new_experiment.protocol_id;
            new_experiment.protocol = { data: protocol, on_conflict: { constraint: "protocol_pkey", update_columns: ["id"] } };
          }
          const new_g_op_id = await this.$apollo
            .mutate({
              mutation: require("@/graphql/ht/group_operation/insert.gql"),
              variables: {
                objects: [new_experiment],
              },
            })
            .then(({ data: { r: { returning: new_gps } } }) => new_gps[0].id);

          this.$root.$emit("toast", "Experiment created", "success");
          this.$router.push({ name: "experiment", params: { id: new_g_op_id } });
        } catch (error) {
          this.$root.$emit("toast_error", error, "error");
        }
      }

      this.dialog = false;
      this.creation = false;
      this.cloning = false;
      this.new_gp = this.default_gp_generator();
    },
    delete(confirmed) {
      if (confirmed) {
        const sample_ids_in_sss_out = this.item_to_delete.operations
          .map(({ sample_states_out }) => sample_states_out)
          .reduce((prev_sss_out, current_sss_out) => [...prev_sss_out, ...current_sss_out])
          .map(({ sample }) => sample.id);
        this.$apollo
          .mutate({
            mutation: require("@/graphql/chemistry/experiment/delete_experiment.gql"),
            variables: {
              task_id: this.item_to_delete.tasks[0].id,
              gp_id: this.item_to_delete.id,
              operation_ids: this.item_to_delete.operations.map(({ id }) => id),
              sample_ids: sample_ids_in_sss_out,
            },
          })
          .then(() => {
            this.$apollo.queries.experiment_list.refetch();
            this.$root.$emit("toast", "Experiment deleted", "success");
          })
          .catch((error) => {
            this.$root.$emit("toast_error", error);
          });
      }

      this.item_to_delete = null;
      this.deletion = false;
      this.dialog = false;
    },
    on_delete_experiment(task) {
      this.item_to_delete = task;
      this.deletion = true;
      this.dialog = true;
    },
    on_clone_experiment(gp) {
      this.$apollo
        .query({
          query: require("@/graphql/chemistry/experiment/get_by_pk.gql"),
          variables: { id: gp.id, exp_in_listing: true },
          fetchPolicy: "network-only",
        })
        .then(({ data: { group_operation_by_pk } }) => {
          this.$apollo.queries.experiment_list.refetch();
          const assignee = group_operation_by_pk.tasks[0].assignee ?? -1;
          const description = `Clone of the task ${group_operation_by_pk.tasks[0].title}, #${group_operation_by_pk.tasks[0].id} \n ${group_operation_by_pk.tasks[0].description}`;
          const operations = cloneDeep(group_operation_by_pk).operations.map((op) => ({
            operation_type_key: "synthetic",
            meta: op.meta,
          }));
          this.new_gp = { ...group_operation_by_pk, operations, tasks: [{ ...group_operation_by_pk.tasks[0], description, assignee }] };
          this.cloning = true;
          this.dialog = true;
        });
    },
    get_sample(gp, sample_state) {
      if (gp.operations.length === 0) return [];
      return gp.operations
        .flatMap((op) => op[sample_state])
        .map((ss) => ss?.sample);
    },
    on_pagination(payload) {
      for (const key of Object.keys(payload)) {
        this.pagination[key] = payload[key];
      }
      const { page, pageCount } = payload;
      if (pageCount < page && pageCount > 0) {
        this.pagination.page = pageCount;
        this.pagination.pageStart = pageCount;
      }
    },
    on_filtered(where_condition) {
      this.filter_conditions.where = {
        _and: [
          DEFAULT_QUERY_CONDITION,
          ...where_condition._and,
        ],
      };
    },
    on_aggregation_counts_updated(payload) {
      for (const condition_key in payload) {
        if (isEqual(payload[condition_key], FALSE_EXPRESSION)) {
          this.filter_conditions[condition_key] = FALSE_EXPRESSION;
          continue;
        }
        this.filter_conditions[condition_key] = {
          _and: [
            DEFAULT_QUERY_CONDITION,
            ...payload[condition_key]._and,
          ],
        };
      }
    },
  },
};
</script>

<style scoped>
.sample-list {
  display: flex;
  flex-wrap: wrap;
  overflow: hidden;
}
.all {
  max-width: 180px;
}
.ellipses {
  font-size: 15px;
  position: relative;
  top: 2px;
  color: #5282ff;
  cursor: pointer;
}
.name-link {
  min-width: unset !important;
}
.location-icon:hover {
  opacity: 0.7;
  transition: 0.1 all ease-in;
}
.assignee {
  white-space: nowrap;
  color: #757575;
}
.operations__amount {
  font-family: monospace;
}
.disabled {
  color: #757575;
}
.undisabled {
  color: #2160EA;
}
.task__username {
  display: flex;
  align-items: center;
  height: 28px;
  font-size: 13px;
  color: #555;
}
.experiment-name-container {
  width: 110px;
  max-width: 200px;
  height: 80px;
  display: flex;
  align-items: start;
  margin: auto;
  flex-direction: column;
  padding: 10px 0px;
  justify-content: center;
}

.related-compound-container {
  display: flex;
  align-items: start;
  flex-direction: column;
  margin: 10px 0px;
  width: max-content;
  justify-content: space-between;
}

.related-compound {
  width: 100%;
  display: flex;
  justify-content: space-between;
}
.related-compound-content a {
  max-width: 180px;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  text-decoration: unset;
}
.compound-type {
  display: block
}

.experiment-smart-filter .explicit-filters-container {
  align-items: unset;
}
.picker {
  text-align: left;
  z-index: 1;
}
#app .seafil-block .search-field .picker .v-input__control {
  height: 34px !important;
  }
#app .seafil-block .picker .v-input__control .v-input__slot {
  width: 100% !important;
}
#app .seafil-block .picker .v-input input {
  padding-left: 5px !important;
}
.picker .v-input {
  margin-bottom: 20px;
}
</style>
