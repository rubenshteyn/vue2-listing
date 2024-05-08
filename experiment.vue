<template>
  <v-container
    v-if="!$apollo.queries.group_operation_by_pk.loading"
    fluid
    class="pt-0 pr-0 pr-sm-3 pl-0 pl-sm-3 o-hidden"
  >
    <div class="d-md-flex align-center justify-space-between mb-4 w-100">
      <!-- Title container -->
      <div class="d-flex flex-row">
        <h2
          class="heading heading--h2 grey--text text--darken-4"
        >
          {{ experiment_title }}
        </h2>
        <span
          class="badge ml-2"
          :class="experiment_status"
        >
          {{ experiment_status }}
        </span>
      </div>
      <div class="mr-3 text-body-1">
        Author: <span class="text-bold">{{ experiment_author_text }}</span>
      </div>
      <!-- Container with inputs for linking experiments -->
      <div class="linked-operation-container">
        <!-- protocol  -->
        <v-autocomplete
          v-model="experiment_protocol_id"
          label="Protocol"
          outlined
          dense
          :items="protocol_options"
          no-data-text="No compatible protocols found"
          item-text="name"
          item-value="id"
          :search-input.sync="protocol_search_input"
          @change="on_protocol_change"
        >
          <template #selection="{ item: protocol }">
            <span class="mr-1">{{ protocol.name }}</span>
            <span :class="protocol.approved ? 'approved' : 'unapproved'">
              {{ protocol.approved == null ? "unreviewed" : protocol.approved ? "approved" : "unapproved ver." }}
            </span>
          </template>
          <template #item="{ item: protocol }">
            <span class="mr-1">{{ protocol.name }}</span>
            <span v-if="protocol.approved === null">unreviewed</span>
            <span
              v-else
              :class="protocol.approved ? 'approved' : 'unapproved'"
            >
              {{ protocol.approved ? "approved" : "unapproved ver." }}
            </span>
          </template>
        </v-autocomplete>
      </div>
      <!-- Assignee and reviewer badges -->
      <div
        v-if="assignee_options.length > 0 && reviewer_options.length > 0"
        class="badge-container"
      >
        <EditableBadge
          :search_input.sync="input.assignee_search"
          :value="task?.assignee || -1"
          :items="assignee_options"
          :badge="false"
          size="sm"
          label="Assignee"
          :loading="$apollo.queries.assignee_options.loading"
          :fields="['first_name', 'last_name']"
          :access="badge_edit_access"
          :item_text="'first_name,last_name'"
          fallback="no assignee"
          @input="on_task_update('assignee', $event)"
        />
        <EditableBadge
          :search_input.sync="input.reviewer_search"
          :value="task?.reviewer || -1"
          :items="reviewer_options"
          :badge="false"
          size="sm"
          :loading="$apollo.queries.reviewer_options.loading"
          :fields="['first_name', 'last_name']"
          :access="badge_edit_access"
          label="Reviewer"
          :item_text="'first_name,last_name'"
          fallback="no reviewer"
          @input="on_task_update('reviewer', $event)"
        />
      </div>
      <v-menu
        bottom
        :close-on-click="true"
      >
        <template #activator="{ on, attrs }">
          <v-hover v-slot="{ hover }">
            <v-btn
              v-if="$user.can('insert_dataset_one')"
              class="btn-create blue white--text"
              :class="{ 'darken-2': hover}"
              depressed
              :ripple="false"
              v-bind="attrs"
              v-on="on"
            >
              <v-icon small>
                mdi-plus
              </v-icon>
              <span>Add dataset</span>
            </v-btn>
          </v-hover>
        </template>

        <v-list>
          <v-list-item
            v-for="(dataset_type, index) in dataset_type_items"
            :key="index"
            @click="create_dataset(dataset_type.value)"
          >
            <v-list-item-title>
              {{ dataset_type.text }}
            </v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
    </div>

    <!-- Exp. status controls -->
    <v-row>
      <v-col cols="auto">
        <v-hover v-slot="{ hover }">
          <v-btn
            v-show="is_experiment_planned"
            :disabled="!is_checkin_completed"
            :color="hover ? 'blue darken-2' : 'blue'"
            class="white--text mr-1"
            :ripple="false"
            depressed
            @click="on_start_click"
          >
            Start
          </v-btn>
        </v-hover>
        <v-hover v-slot="{ hover }">
          <v-btn
            v-show="is_experiment_in_progress"
            :color="hover ? 'green darken-2' : 'green'"
            class="white--text mr-1"
            :ripple="false"
            depressed
            @click="on_complete_click"
          >
            Complete
          </v-btn>
        </v-hover>
        <v-hover v-slot="{ hover }">
          <v-btn
            v-show="is_experiment_in_progress"
            :color="hover ? 'red darken-2' : 'red'"
            class="white--text"
            :ripple="false"
            depressed
            @click="on_abort_click"
          >
            Abort
          </v-btn>
        </v-hover>
        <v-hover v-slot="{ hover }">
          <v-btn
            v-show="is_experiment_completed"
            :disabled="is_experiment_rollback_disabled"
            :color="hover ? 'red darken-2' : 'red'"
            class="white--text"
            :ripple="false"
            depressed
            @click="on_rollback_click"
          >
            Rollback
          </v-btn>
        </v-hover>
      </v-col>
    </v-row>

    <!-- Exp. status messages -->
    <v-row no-gutters>
      <v-col
        cols="auto"
        :style="{ 'font-size': '13px' }"
      >
        <div v-show="is_experiment_planned && is_checkin_completed">
          The experiment can be started now.
        </div>
        <div v-show="is_experiment_in_progress">
          The experiment is in progress. Click <b>Complete</b> when ready, or <b>Abort</b> to restart.
        </div>
        <div v-show="is_experiment_completed && !is_experiment_rollback_disabled">
          The experiment is completed. You can click <b>Rollback</b> to return to the <b>in progress</b> state.
        </div>
      </v-col>
    </v-row>

    <!-- Exp. tabs -->
    <v-row>
      <v-col class="pt-0 pb-0">
        <v-tabs
          v-model="tab"
          show-arrows
          hide-slider
          height="45"
          @change="on_tab_change"
        >
          <v-tab
            href="#planning"
            :ripple="false"
          >
            <span>Planning</span>
          </v-tab>
          <v-tab
            href="#check-in"
            :ripple="false"
          >
            <span>Check in samples</span>
          </v-tab>
          <v-tab
            :ripple="false"
            href="#method"
          >
            <span>Method description</span>
          </v-tab>
          <v-tab
            :ripple="false"
            href="#access"
          >
            <span>Access</span>
          </v-tab>
          <v-tab
            :ripple="false"
            href="#datasets"
          >
            <span>Datasets</span>
          </v-tab>
        </v-tabs>
      </v-col>
      <v-progress-linear
        v-show="$apollo.queries.group_operation_by_pk.loading"
        indeterminate
        class="progress-linear-in-tabs"
      />
    </v-row>

    <!-- Tab windows w/ widgets -->
    <v-row v-if="current_experiment">
      <v-col>
        <v-tabs-items
          v-model="tab"
          touchless
          class="fill-height"
        >
          <v-tab-item value="planning">
            <WidgetWrapper
              class="widget"
              :widget_title="widgets.synthesis_settings.title"
              :widget_type="widgets.synthesis_settings.widget_type"
              :edit_scene_mode="layout_editing_on"
              :editable="!disabled_widgets_map[widgets.synthesis_settings.widget_type]"
              :user_can="user_can"
              @delete="on_delete_widget(i)"
              @save="on_save"
              @reset="on_reset"
            />
          </v-tab-item>
          <v-tab-item value="check-in">
            <WidgetWrapper
              class="widget"
              :widget_title="widgets.check_in_editor.title"
              :widget_type="widgets.check_in_editor.widget_type"
              :edit_scene_mode="layout_editing_on"
              :editable="!disabled_widgets_map[widgets.check_in_editor.widget_type]"
              :user_can="user_can"
              @delete="on_delete_widget(i)"
              @save="on_save"
              @reset="on_reset"
            />
          </v-tab-item>
          <v-tab-item value="method">
            <WidgetWrapper
              class="widget"
              :widget_title="widgets.method_editor.title"
              :widget_type="widgets.method_editor.widget_type"
              :edit_scene_mode="layout_editing_on"
              :editable="!disabled_widgets_map[widgets.method_editor.widget_type]"
              :user_can="user_can"
              @delete="on_delete_widget(i)"
              @save="on_save"
              @reset="on_reset"
            />
          </v-tab-item>
          <v-tab-item value="datasets">
            <v-menu
              bottom
              :close-on-click="true"
            >
              <template #activator="{ on, attrs }">
                <v-hover v-slot="{ hover }">
                  <v-btn
                    v-if="$user.can('insert_dataset_one')"
                    class="btn-create blue white--text"
                    :class="{ 'darken-2': hover}"
                    depressed
                    :ripple="false"
                    v-bind="attrs"
                    v-on="on"
                  >
                    <v-icon small>
                      mdi-plus
                    </v-icon>
                    <span>Add dataset</span>
                  </v-btn>
                </v-hover>
              </template>

              <v-list>
                <v-list-item
                  v-for="(dataset_type, index) in dataset_type_items"
                  :key="index"
                  @click="create_dataset(dataset_type.value)"
                >
                  <v-list-item-title>
                    {{ dataset_type.text }}
                  </v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>
            <div
              v-if="group_operation_by_pk.datasets.length > 0"
            >
              <WidgetWrapper
                v-for="dataset in group_operation_by_pk.datasets"
                :key="dataset.id"
                class="widget"
                :widget_title="dataset_type_items.find(({ value }) => value === dataset.type_key).text"
                :widget_type="widgets.datasets_editor.widget_type"
                :edit_scene_mode="false"
                :editable="true"
                :user_can="user_can"
                :widget_item_id="dataset.id"
              />
            </div>
            <div v-else>
              This experiment does not have any datasets.
            </div>
          </v-tab-item>
          <v-tab-item value="access">
            This is acces-tab stub
          </v-tab-item>
        </v-tabs-items>
      </v-col>
    </v-row>

    <CTXDialog
      :value="confirmation_dialog"
      :config="confirmation_dialog_config"
    />

    <v-dialog
      v-if="original_experiment"
      v-model="show_new_protocol_modal"
    >
      <v-card>
        <v-card-title>
          Cannot edit a protocol with completed experiments!
        </v-card-title>
        <v-card-text>
          To introduce changes to protocol "{{ original_experiment.protocol.name }}", you have to post a new one.
        </v-card-text>
        <v-card-text>
          <ul>
            <li>
              A new protocol will be related to the protocol "{{ original_experiment.protocol.name }}".
            </li>
            <li>
              All changes will be saved in the new protocol.
            </li>
            <li>
              The experiment will be linked to the new protocol.
            </li>
          </ul>
        </v-card-text>
        <v-card-text class="mt-6">
          <v-text-field
            v-model="new_protocol_name"
            label="Protocol name"
            outlined
            dense
          />
        </v-card-text>
        <v-expand-transition>
          <v-card-text
            v-show="new_protocol_actions_visible"
          >
            Please, provide a name for the new protocol
          </v-card-text>
        </v-expand-transition>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="success"
            depressed
            :disabled="!new_protocol_name"
            @click="on_confirm_breaking_changes"
          >
            Confirm
          </v-btn>
          <v-btn
            text
            @click="show_new_protocol_modal = false"
          >
            Cancel
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <CTXDialog
      :value="show_protocol_switched_modal"
      persistent
      title="Are you sure you want to change the protocol?"
      confirm_text="Confirm"
      tone="primary"
      @update="on_protocol_switched_modal_close"
    >
      <template #simple-body-slot>
        All operation overrides will be discarded!
      </template>
    </CTXDialog>

    <CTXDialog
      v-if="show_protocol_compound_editors_modal"
      persistent
      :value="show_protocol_compound_editors_modal"
      title="Fill in all the resulting properties of the substance"
      confirm_text="Save"
      :is_confirm_button_disabled="product_editors_have_errors"
      @update="on_protocol_compound_editors_modal_close"
    >
      <template #simple-body-slot>
        <v-container>
          <v-row
            justify="center"
            dense
          >
            <v-col
              v-for="(_, index) in protocol_compound_products"
              :key="index"
              cols="auto"
            >
              <ProtocolCompoundEditor
                show_only_compound_editor
                :main_protocol="is_protocol_published"
                :operation_id="operation_0_getter.id"
                :protocol_compound="protocol_compound_products[index]"
                :compound_ind="protocol_compound_products[index].original_index"
                :disabled="false"
                @error="on_protocol_compound_editors_set_error(index)"
                @clear-error="on_protocol_compound_editors_clear_error(index)"
              >
                <!-- Longer content can be truncated with a text ellipsis using the .text-truncate utility class: -->
                <div class="text-truncate mb-3">
                  {{ protocol_compound_products[index].compound.name }}
                </div>
              </ProtocolCompoundEditor>
            </v-col>
          </v-row>
        </v-container>
      </template>
    </CTXDialog>
  </v-container>
</template>

<script type="text/javascript">
import {
  cloneDeep, setWith, get, isEqual, pick, omit,
} from "lodash";
import { get_displayable_variables_map, extract_protocol_variables } from "@/lib/experiment_state.js";

import { DATASET_TYPE_ITEMS } from "@/views/general/dataset/types/dataset_types.js";
import { shift_and_insert } from "@/lib/util.js";
import CTXDialog from "@/components/dialog.vue";
import EditableBadge from "@/components/editables/editable_badge.vue";
import WidgetWrapper from "@/components/widget_wrapper.vue";
import ProtocolCompoundEditor from "@/components/widgets/synthesis_settings/protocol_compound_editor.vue";

const WIDGET_TYPES_CONFIG = {
  check_in_editor: {
    title: "Сheck in editor",
    meta_field: "check_in",
    widget_type: "check_in_editor",
  },
  synthesis_settings: {
    title: "Synthesis settings",
    widget_type: "synthesis_settings",
  },
  method_editor: {
    title: "Method editor",
    meta_field: "method",
    widget_type: "method_editor",
  },
  datasets_editor: {
    title: "Datasets editor",
    meta_field: "datasets",
    widget_type: "datasets_editor",
  },
};

const DEFAULT_COMPOUND_CONFIG = {
  eq: { value: null, units: "unit" },
  amount: { value: 0, units: "mmol" },
  mw: { value: 0, units: "g/mol" },
  yield: { value: null, units: "%" },

  mass: { value: null, units: "mg" },
  density: { value: null, units: "g/L" },
  mass_fraction: { value: null, units: "%" },

  c: { value: null, units: "M" },
  volume: { value: null, units: "L" },

  solvent: null,
  sol_c: { value: null, units: "M" },
  sol_v: { value: null, units: "mL" },

  modes: {
    mass: true,
    volume_mod_c: false,
  },
};

export default {
  name: "Experiment",
  components: {
    EditableBadge,
    CTXDialog,
    WidgetWrapper,
    ProtocolCompoundEditor,
  },
  provide() {
    const reactive_vars = {};
    Object.defineProperties(reactive_vars, {
      op_vars_template: {
        enumerable: true,
        get: () => this.op_vars_template,
      },
      original_experiment: {
        enumerable: true,
        get: () => this.original_experiment,
      },
      current_experiment: {
        enumerable: true,
        get: () => this.current_experiment,
      },
      variables_with_metadata_map: {
        enumerable: true,
        get: () => this.variables_w_metadata_map,
      },
    });

    return {
      reactive_vars,
      setup_checkin_operation: this.setup_checkin_operation,
      checkout_sample: this.checkout_sample,
      rollback_checkin: this.rollback_checkin,
      complete_checkin: this.complete_checkin,
      update_protocol_variable: this.update_protocol_variable,
      update_protocol_compound_role: this.update_protocol_compound_role,
      update_operation_variable: this.update_operation_variable,
      update_protocol_method: this.update_protocol_method,
      add_operation: this.add_operation,
      delete_operation: this.delete_operation,
      add_protocol_compound: this.add_protocol_compound,
      delete_protocol_compound: this.delete_protocol_compound,
      add_method_var: this.add_method_var,
      delete_method_var: this.delete_method_var,
      delete_dataset: this.delete_dataset,
    };
  },
  props: {
    id: {
      type: Number,
      default() { return Number(this.$route.params.id); },
    },
  },
  apollo: {
    group_operation_by_pk: {
      query: require("@/graphql/chemistry/experiment/get_by_pk.gql"),
      fetchPolicy: "network-only",
      skip() { return !this.id; },
      variables() {
        return {
          id: this.id,
        };
      },
      result({ data }) {
        if (!data.group_operation_by_pk) {
          this.$router.replace({ name: "404" });
          return;
        }
        const gp = cloneDeep(data.group_operation_by_pk);

        this.initialize_experiment_page(gp);

        // initialize the widgets
        this.initialize_widgets(gp);

        // - initialize variables for the maintained experiment state
        // - reset local accumulators of changes
        // get new protocol state, overrides state
        // get new original agregate state of variables
        // rerun transformers for variabless
        this.initialize_maintained_experiment_state(gp);

        return gp;
      },
    },
    protocol_options: {
      query: require("@/graphql/chemistry/protocol/search.gql"),
      debounce: 300,
      fetchPolicy: "network-only",
      skip() {
        return !this.original_experiment;
      },
      variables() {
        return {
          where: {
            _and: [
              { name: { _ilike: `%${this.protocol_search_input ?? ""}%` } },
              { operation_type_key: { _eq: this.original_experiment.protocol.operation_type_key } },
            ],
          },
        };
      },
      update({ protocol: protocols }) {
        let res = protocols;
        if (!res.some((p) => p.id === this.original_experiment.protocol_id)) {
          res = [this.original_experiment.protocol, ...res];
        }
        return res.map((protocol) => (protocol.name !== null ? protocol : {
          ...protocol,
          name: `Unnamed protocol #${protocol.id}`,
        }));
      },
    },
    assignee_options: {
      query: require("@/graphql/config/user/search.gql"),
      debounce: 300,
      variables() {
        const search_snippets = this.input.assignee_search.split(" ");
        return {
          where: {
            _or: search_snippets.flatMap(
              (snippet) => [
                { first_name: { _ilike: `%${snippet}%` } },
                { last_name: { _ilike: `%${snippet}%` } },
              ],
            ),
          },
          order_by: { last_name: "asc" },
          limit: 5,
        };
      },
      update(data) {
        const users = cloneDeep(data.user);
        if (this.group_operation_by_pk?.tasks.length === 0) return users;

        const assigned_user = this.group_operation_by_pk?.tasks[0].assignee_user;
        if (assigned_user) {
          users.splice(0, 0, assigned_user);
        }
        users.push(this.get_no_user());
        return users;
      },
    },
    reviewer_options: {
      query: require("@/graphql/config/user/search.gql"),
      debounce: 300,
      variables() {
        const search_snippets = this.input.reviewer_search.split(" ");
        return {
          where: {
            _or: search_snippets.flatMap(
              (snippet) => [
                { first_name: { _ilike: `%${snippet}%` } },
                { last_name: { _ilike: `%${snippet}%` } },
              ],
            ),
          },
          order_by: { last_name: "asc" },
          limit: 5,
        };
      },
      update(data) {
        const users = cloneDeep(data.user);
        if (this.group_operation_by_pk?.tasks.length === 0) return users;

        const assigned_reviewer = this.group_operation_by_pk?.tasks[0].reviewer_user;
        if (assigned_reviewer) {
          users.splice(0, 0, assigned_reviewer);
        }
        users.push(this.get_no_user());
        return users;
      },
    },
  },

  data() {
    return {
      dataset_type_items: DATASET_TYPE_ITEMS,
      view_types_datasets: false,
      // general utility
      LOCAL_ONLY_ID: -1,
      loading: false,
      assignee_options: [],
      reviewer_options: [],

      // Editable experiment config
      original_experiment: null,
      current_experiment: null,
      experiment_protocol_id: null,
      protocol_search_input: null,
      task: null,
      experiment_type: "",
      experiment_author_text: "",
      // creating new protocol
      new_protocol_actions_visible: false,
      new_protocol_name: "",
      // to controll widgets
      badge_edit_access: { edit: true, delete: false },
      widget_wrappers: [],
      drag_options: {
        scrollSensitivity: 800,
        animation: 200,
        disabled: false,
        ghostClass: "dragged",
        direction: "horizontal",
        forceFallback: true,
      },
      new_widgets: [],
      layout_editing_on: false,
      // TODO: fixs this when the ABA is on
      user_can: {
        create: true,
        update: true,
        delete: true,
      },
      input: { assignee_search: "", reviewer_search: "" },
      dataset_tab: null,
      experiment_title: "",
      experiment_status: "",
      tab: this.$route.params.tab,
      confirmation_dialog: false,
      action_to_confirm: null,
      task_update_constraints: {
        OligoChemist: () => true,
        superuser: () => true,
      },
      op_overrides_map: {},
      raw_protocol_variables: null,
      op_vars_template: {},
      original_op_vars_template: {},
      // variables for discerning local accumulation of changes
      protocol_changes_buffer: [],
      variables_w_metadata_map: {},
      breaking_changes_ctr: 0,
      protocol_changes_ctr: 0,
      operation_meta_changes_ctr: 0,
      // flags to disable widget_editing
      disabled_widgets_map: {},
      show_new_protocol_modal: false,
      widgets: WIDGET_TYPES_CONFIG,
      protocol_compound_changes: {
        deletions: [],
        ex_products: [],
      },
      created_sss_out: [],
      show_protocol_switched_modal: false,
      // these fields are added to fill in the experiment results:
      show_protocol_compound_editors_modal: false,
      protocol_compound_editors_errors: [],

      protocol_compound_products: [],
    };
  },

  computed: {
    /**
     * All possible widget_types minus those, that have already been added.
     */
    widget_types_to_be_added() {
      return Object.values(WIDGET_TYPES_CONFIG)
        .filter(({ widget_type }) => !this.widget_wrappers.some((wr) => wr.widget_type === widget_type));
    },
    datasets_enabled() {
      return this.group_operation_by_pk?.datasets.length > 0;
    },
    dataset_addition_available() {
      return this.datasets_enabled
        && this.$user.can("insert_dataset_one")
        && this.group_operation_by_pk?.status === "in-progress"
        && this.group_operation_by_pk?.datasets.length === 0;
    },
    confirmation_dialog_config() {
      if (this.action_to_confirm === "reset") {
        return {
          title: "Discard changes",
          body_text: "Are you sure you want to reset and lose all unsaved content?",
          confirm_text: "Reset",
          cancel_text: "Cancel",
          tone: "error",
          on_update: this.on_reset_confirm_dialog_update,
        };
      }
      if (this.action_to_confirm === "close") {
        return {
          title: "Close confirmation",
          body_text: "Are you sure you want to close and lose all unsaved content?",
          confirm_text: "Close",
          cancel_text: "Cancel",
          tone: "error",
          on_update: this.on_close_without_save_confirm_dialog_update,
        };
      }
      return {
        title: "Uhhhh, woops!",
        body_text: "Please, contact the devs and let them now how you got this message.",
        tone: "error",
        on_update: () => { this.confirmation_dialog = false; },
      };
    },
    is_checkin_completed() {
      const { operations } = this.original_experiment ?? {};
      if (!operations || operations.length === 0) return false;
      return operations.every(({ sample_states_in }) => {
        // every acts like the "for all" quantifier in mathematics. In particular, for an empty array, it returns true.
        // (It is vacuously true that all elements of the empty set satisfy any given condition.)
        if (sample_states_in.length === 0) return false;
        return sample_states_in.every((ss_in) => {
          const { status, operation_in: { status: op_in_status } } = ss_in;
          return status === "now" && op_in_status === "completed";
        });
      });
    },
    is_experiment_planned() {
      return this.original_experiment?.status === "planned";
    },
    is_experiment_in_progress() {
      return this.original_experiment?.status === "in-progress";
    },
    is_experiment_completed() {
      return this.original_experiment?.status === "completed";
    },
    is_experiment_rollback_disabled() {
      return this.is_sss_out_occupied;
    },
    is_sss_out_occupied() {
      return this.original_experiment?.operations
        .flatMap((op) => op.sample_states_out)
        .some((ss_out) => ss_out.status === "past" || ss_out.operation_out?.status === "in-progress") ?? false;
    },
    operation_0_getter() { // TODO (CTX-1395): Let it rest here until we go multi-operational within one experiment
      const { operations } = this.current_experiment;
      if (operations.length !== 1) throw new Error(`(operations.length === ${operations.length}) !== 1`);
      return operations[0];
    },
    is_protocol_published() {
      return this.current_experiment.protocol.approved !== null;
    },
    product_editors_have_errors() {
      return this.protocol_compound_editors_errors.some(Boolean);
    },
  },

  methods: {
    get_no_user() {
      return { id: -1, first_name: "no", last_name: "assignee" };
    },
    on_protocol_change() {
      this.show_protocol_switched_modal = true;
    },
    on_protocol_compound_editors_set_error(index) {
      this.$set(this.protocol_compound_editors_errors, index, true);
    },
    on_protocol_compound_editors_clear_error(index) {
      this.$set(this.protocol_compound_editors_errors, index, false);
    },
    on_protocol_compound_editors_modal_close(confirmed) {
      this.show_protocol_compound_editors_modal = false;
      this.protocol_compound_editors_errors = [];
      if (!confirmed) this.on_reset_confirm_dialog_update(true); // reset changes
      else this.complete_experiment();
    },
    on_protocol_switched_modal_close(confirmed) {
      this.show_protocol_switched_modal = false;
      if (!confirmed) {
        this.experiment_protocol_id = this.group_operation_by_pk.protocol_id;
      } else {
        this.update_experiment("protocol", this.experiment_protocol_id);
      }
    },
    prepare_op_metas_for_autocalculator() {
      for (const protocol_compound_product of this.protocol_compound_products) {
        const path = `protocol.protocol_compounds[${protocol_compound_product.original_index}]`;
        for (const operation of this.current_experiment.operations) {
          const { compound_id, compound_config } = this.is_protocol_published ? get(this.current_experiment, path) : get(operation, path);
          for (const ss_out of operation.sample_states_out) {
            if (ss_out.sample.compound_id !== compound_id) continue;

            const mass = { ...compound_config.mass };
            const amount = { ...compound_config.scale }; // now amount = scale
            setWith(operation, `meta.properties.set[${ss_out.id}]`, { mass, amount }, Object);
          }
        }
      }
    },
    complete_experiment() {
      this.prepare_op_metas_for_autocalculator();
      const operation_ids = this.current_experiment.operations.map(({ id }) => id);

      // turn current_experiment into upsert-mutation payload:
      const experiment_upsert_payload = this.gen_experiment_upsert(this.current_experiment, false, true);

      this.$apollo.mutate({
        mutation: require("@/graphql/chemistry/experiment/complete_experiment.gql"),
        variables: {
          object: experiment_upsert_payload,
          operation_ids,
          gp_id: this.id,
        },
        update(cache, { data: { set_gp_status_completed: gp } }) {
          const { status, operations } = gp;
          for (const operation of operations) {
            cache.modify({
              id: cache.identify(operation),
              fields: {
                status() {
                  return status;
                },
              },
            });
          }
        },
      }).then(() => {
        this.$root.$emit("toast", "Experiment completed successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", "Couldn't complete experiment", error);
      });
    },

    zero_protocol_compound_config_values(protocol_compound) {
      for (const { compound_config } of protocol_compound) {
        for (const key in compound_config) {
          const variable_data = compound_config[key];
          if (Object.hasOwn(variable_data, "value")) variable_data.value = 0;
          else compound_config[key] = 0;
        }
      }
    },
    prepare_protocol_compound_products() {
      const path = "protocol.protocol_compounds";
      const protocol_compounds = cloneDeep(get(this.is_protocol_published ? this.current_experiment : this.operation_0_getter, path, []))
        .map((protocol_compound, original_index) => ({ ...protocol_compound, original_index }));
      const protocol_compound_products = protocol_compounds.filter(({ compound_role }) => compound_role === "product");
      this.zero_protocol_compound_config_values(protocol_compound_products);
      this.protocol_compound_products = protocol_compound_products;
    },
    on_complete_click() {
      this.prepare_protocol_compound_products();
      this.show_protocol_compound_editors_modal = true;
    },

    on_start_click() {
      this.update_experiment_status("in-progress", "Experiment started successfully", "Couldn't start experiment");
    },
    on_abort_click() {
      this.update_experiment_status("planned", "Experiment aborted successfully", "Couldn't abort experiment");
    },
    on_rollback_click() {
      this.update_experiment_status("in-progress", "Experiment rolled back successfully", "Couldn't rollback experiment");
    },
    /**
     * Utility func
     * @param {"planned" | "in-progress"} new_status - group_operation.status options
     * @param {string} success_message - message to show when mutation succeeds
     * @param {string} error_message - message to show when mutation fails
     */
    update_experiment_status(new_status, success_message, error_message) {
      const { id: gp_id, $apollo, $root } = this;
      const _set = { status: new_status };

      $apollo.mutate({
        mutation: require("@/graphql/chemistry/experiment/update_experiment_status.gql"),
        variables: { gp_id, _set },
        update(cache, { data: { update_group_operation_by_pk: gp } }) {
          const { status, operations } = gp;
          for (const operation of operations) {
            cache.modify({
              id: cache.identify(operation),
              fields: {
                status() {
                  return status;
                },
              },
            });
          }
        },
      }).then(() => {
        $root.$emit("toast", success_message);
      }).catch((error) => {
        $root.$emit("toast_error", `${error_message}: ${error}`);
      });
    },
    /**
     * Uses data from the loaded group_operation to initialize local variables
     * for the inputs in the header of the page to manage
     * author, protocol, assignee and reviewer.
     *
     * @param {GroupOperation} - group operation, describing the experiment
     *
     * It contains:
     * {Object} meta - group operation meta, holds info abt the parent protocol
     * {"planned" | "in-progress" | "completed"} status - group operation status
     * {Protocl} protocol - protocol used throughout the group operation
     * {Array<Task>} tasks - all the 'task'-table entries, related to the protocol
     * {Array<Dataset>} datasets - all the 'dataset'-table entries, related to the protocol
     */
    initialize_experiment_page({
      meta, status, protocol, tasks, datasets,
    }) {
      this.experiment_title = `${meta.name} #${this.id}`;
      this.experiment_status = status;
      this.experiment_protocol_id = protocol.id;
      const { creator_user } = tasks[0] ?? { creator_user: {} };
      [this.task] = cloneDeep(tasks);
      this.experiment_author_text = `${creator_user.first_name} ${creator_user.last_name}`;

      if (datasets.length > 0 && this.dataset_tab === null) this.dataset_tab = 0;
    },
    /**
     * Scours each operation's meta and returns a map of overrides in a format
     * compatible with that of raw_protocol_variables (refer to the previous function).
     *
     * @param {Array} operations - array of operations from the loaded group_operation.
     */
    extract_protocol_variables_overrides(operations) {
      // operation->protocol->protocol_compound json:
      // [{ id, meta, protocol: { vars, protocol_compounds: [{ id, compound_role, compound_config }] } }]
      const operation_overrides_map = {};
      for (const op of operations) {
        if (!op.meta) continue;
        const res = {};
        const op_id = op.id;
        const variable_overrides = op.meta?.parameters?.variable_overrides ?? {};
        res.method_vars = { ...variable_overrides.method_vars };
        res.protocol_compound_vars = cloneDeep(variable_overrides.protocol_compound_vars ?? {});
        operation_overrides_map[op_id] = res;
      }
      return operation_overrides_map;
    },
    /**
     * Generates a version of the protocol, that is pasted inside each of the current_experiment.operations array.
     * Takes the raw version of the protocol from group_operation and applies meta.parameters.variable_overrides to it.
     *
     * @param {Object} protocol - general protocol for all operations
     * @param {{protocol_compound_vars: Object, method_vars: Object}} operation_overrides - singular operation's overrides of the general protocol
     */
    gen_overriden_operation_protocol(protocol, operation_overrides) {
      const res_protocol = cloneDeep(protocol);
      delete res_protocol.method;
      for (const [p_cpd_ind, override] of Object.entries(operation_overrides.protocol_compound_vars ?? {})) {
        if (res_protocol.protocol_compounds[p_cpd_ind]) {
          res_protocol.protocol_compounds[p_cpd_ind].compound_config = {
            ...protocol.protocol_compounds[p_cpd_ind].compound_config,
            ...override,
          };
        }
      }

      res_protocol.vars = {
        ...protocol.vars,
        ...operation_overrides.method_vars,
      };
      return res_protocol;
    },
    /**
     * Updates this.op_vars_template. Writes new value to 'method_vars' and/or 'protocol_compound_vars'
     * fields of every operation in the map.
     * Both updates can be turned off by passing the arguments:
     * @param {Boolean} update_method_vars - whether to update method_vars
     * @param {Boolean} update_p_cpds - whether to update protocol_compound_vars in each operation
     */
    update_op_vars_template(update_method_vars = true, update_p_cpds = true) {
      if (update_method_vars) this.op_vars_template.method_vars = this.current_experiment.protocol.vars;
      if (update_p_cpds) {
        const p_cpd_entries = this.current_experiment.protocol.protocol_compounds.map((p_cpd, p_cpd_ind) => [
          p_cpd_ind,
          cloneDeep(p_cpd.compound_config),
        ]);
        this.op_vars_template.protocol_compound_vars = Object.fromEntries(p_cpd_entries);
      }
    },
    /**
     * Shortens the code that updates the 'original_experiment' by
     * incapsulating the logic that transforms protocol_variables
     * for different widgets.
     *
     * Updates the op_vars_template first, cuz it is in a convenient format and easy to override.
     *  Thus changes to current_experiment.protocol get translated to the op_vars_template.
     * Then, applies overrides from current_experiment.operations to the current protocol_map state.
     */
    rerun_variable_transformers() {
      this.variables_w_metadata_map = get_displayable_variables_map(this.op_vars_template, this.current_experiment.operations, true);
    },
    /**
     * Generates extra data for maintained experiment state used to buffer updates locally
     *  and generate correct updates for operation.meta and protocol.
     *
     * (Re)initializes 'op_vars_template', 'original_experiment', 'current_experiment', 'op_overrides_map'
     * and change counters.
     *
     * @param {GroupOperation} gp - group_operation. Fields of interest: 'protocol' and 'operations'
     */
    initialize_maintained_experiment_state(gp) {
      const { protocol, operations } = gp;
      const raw_protocol_variables = extract_protocol_variables(protocol);
      const op_overrides_map = this.extract_protocol_variables_overrides(operations);

      this.original_op_vars_template = raw_protocol_variables;
      this.op_vars_template = cloneDeep(raw_protocol_variables);

      this.breaking_changes_ctr = 0;
      this.protocol_changes_ctr = 0;
      this.LOCAL_ONLY_ID = -1;
      const operations_w_applied_overrides = operations.map((op) => ({
        ...op,
        protocol: this.gen_overriden_operation_protocol(protocol, op.meta?.parameters?.variable_overrides ?? {}),
      })).sort((a, b) => a.id - b.id);
      this.original_experiment = {
        ...gp,
        operations: operations_w_applied_overrides.map((op) => ({
          ...op,
          check_in_exists: op.sample_states_in.length > 0,
        }
        )),
      };
      this.current_experiment = cloneDeep(this.original_experiment);
      this.raw_protocol_variables = raw_protocol_variables;
      this.op_overrides_map = op_overrides_map;
      this.rerun_variable_transformers();
    },
    /**
     * Incapsulates this.widget_wrappers init-logic.
     * Reads from gp.meta and maps mentioned widgets onto widget_wrapper-props objects
     * @param {GroupOperation} gp - group_operation. Field of interest: meta.widgets
     */
    initialize_widgets(gp = {}) {
      const { meta } = gp;
      if (!gp || !meta) this.widget_wrappers = [];
      this.widget_wrappers = meta.widgets
        ?.filter((widget) => WIDGET_TYPES_CONFIG[widget?.widget_type])
        .map((widget) => WIDGET_TYPES_CONFIG[widget?.widget_type]) ?? [];

      this.disabled_widgets_map = Object.fromEntries(
        this.widget_wrappers.map(({ widget_type }) => [widget_type, false]),
      );
    },
    disable_all_but(widget_key_to_leave_on) {
      for (const w_type of Object.keys(this.disabled_widgets_map)) {
        if (w_type === widget_key_to_leave_on) continue;
        this.disabled_widgets_map[w_type] = true;
      }
    },
    is_local_only(id) {
      return typeof id === "number" && id < 0;
    },
    mark_as_local_new(entity) {
      return {
        id: this.LOCAL_ONLY_ID--,
        ...entity,
      };
    },
    /**
     * Called every time the page registers a change in one of the widgets.
     * Increments 'breaking_changes_ctr' counter any g_ops has status 'completed'
     */
    update_change_counters() {
      if (!this.current_experiment) return;

      let block = false;
      if (this.current_experiment.protocol.group_operations?.length > 0) {
        for (const gop of this.current_experiment.protocol.group_operations) {
          if (gop.status === "completed") {
            block = true;
            break;
          }
        }
      }
      this.breaking_changes_ctr += +block;
      this.protocol_changes_ctr += +!block;
    },
    /**
     * Takes an update from a component somewhere down the line in the experiment component-tree.
     * Decides if the hoisted change is breaking or not and places it into an appropriate change-buffer.
     * Said buffer is to be applied and sent to DB or discarded depending on the events sent from WidgetWrappers.
     *
     * @param {"synthesis_settings" | "method_editor" | "check_in"} widget_type
     * @param {String} path - path to the variable withing the operation->protocol->protocol_compounds tree
     * @param {Object | String | Null} var_value - new value of the variable at the specified path
     */
    update_protocol_variable(widget_type, path, var_value) {
      // disable the other widgets whilst there are unsaved changes in one of them
      this.disable_all_but(widget_type);
      this.update_change_counters();

      setWith(this.current_experiment, path, var_value, Object);
      this.update_op_vars_template();
      this.rerun_variable_transformers();
    },
    /**
     * Added to API to avoid having to search for '.compound_role' in the path argument
     * of the 'update_protocol_variable' function.
     *
     * @param {"synthesis_settings" | "method_editor" | "check_in"} widget_type
     * @param {Number} p_cpd_ind - index of the updated protocol_compound
     * @param {"reagent" | "solvent" | "product"} role - new role
     */
    update_protocol_compound_role(widget_type, p_cpd_ind, role) {
      this.disable_all_but(widget_type);
      this.update_change_counters();
      const p_cpd = this.current_experiment.protocol.protocol_compounds[p_cpd_ind];
      const p_cpd_already_being_deleted = this.protocol_compound_changes.ex_products.some(({ id }) => id === p_cpd.id);
      if (p_cpd.compound_role === "product" && role !== "product" && !p_cpd_already_being_deleted) {
        this.protocol_compound_changes.ex_products.push(p_cpd);
      }
      p_cpd.compound_role = role;
      for (const operation of this.current_experiment.operations) {
        operation.protocol.protocol_compounds[p_cpd_ind].compound_role = role;
        if (role === "product") {
          this.created_sss_out = [
            ...this.created_sss_out,
            this.gen_ss_w_sample(operation.protocol.protocol_compounds[p_cpd_ind].compound_id, operation.protocol.protocol_compounds[p_cpd_ind].compound.name),
          ];
          operation.sample_states_out = this.created_sss_out;
        } else {
          this.created_sss_out = this.created_sss_out.filter((ss) => ss.sample.compound_id !== p_cpd.compound_id);
        }
      }

      this.update_op_vars_template(false, true);
      this.rerun_variable_transformers();
    },
    /**
     * Works similarly to update_protocol_variable, but works with a specified operation's meta.
     * Every hoisted update is meant to be registered relative to the operation's 'protocol'-field.
     *
     * @param {Number} op_id - id of the operation inside the group_operation
     * @param {"synthesis_settings" | "method_editor" | "check_in"} widget_type
     * @param {String} path - starts with "protocol."; generated manually or automatically when using 'this.variables_w_metadata_map'
     * @param {Array | Object | String | Number} var_value - new value;
     */
    update_operation_variable(op_id, widget_type, path, var_value) {
      this.disable_all_but(widget_type);
      this.protocol_changes_ctr += 1;
      const operation = this.current_experiment.operations.find(({ id }) => id === op_id);
      setWith(operation, path, var_value, Object);
      this.rerun_variable_transformers();
    },
    /**
     * Updates the content of the experiment's protocol's method.
     * Used by the MethodEditor.
     * @param {Record<Lang, { published: boolean, content: string }>} new_method_content
     */
    update_protocol_method(new_method_content) {
      this.disable_all_but("method_editor");
      this.update_change_counters();

      this.current_experiment.protocol.method = new_method_content;
    },
    /**
     * Directly calls a mutation to add another operation (aka experiment variation)
     * to the experiment (w/o updating 'current_experiment')
     */
    add_operation() {
      const { operation_type_key, id } = this.original_experiment;
      const product_compounds = this.current_experiment.protocol.protocol_compounds.filter((protocol_compound) => protocol_compound.compound_role === "product");
      const product_sss = {
        data: product_compounds.map(({ compound_id, compound }) => ({
          sample: {
            data: {
              compound_id,
              status: "planned",
              name: compound.name,
            },
          },
        })),
      };
      this.$apollo.mutate({
        mutation: require("@/graphql/ht/operation/insert.gql"),
        variables: {
          objects: [{
            group_operation_id: id,
            operation_type_key,
            sample_states_out: product_sss,
            meta: {
              properties: {},
            },
          }],
        },
        update(cache, { data: { insert_operation: { returning: inserted_ops } } }) {
          const new_op = inserted_ops[0];
          cache.modify({
            id: cache.identify({ id, __typename: "group_operation" }),
            fields: {
              operations(cached_ops) {
                return [...cached_ops, new_op];
              },
            },
          });
        },
      }).then(() => {
        this.$root.$emit("toast", "Operation created successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", error);
      });
    },
    /**
     * Directly calls a mutation to remove an operation (aka experiment variation)
     * from the experiment (w/o updating 'current_experiment')
     * @param {integer} op_id - id of the operation to be deleted
     */
    delete_operation(op_id) {
      const deleted_operation = this.current_experiment.operations.find(({ id }) => id === op_id);
      const sample_ids_in_ss_out = deleted_operation?.sample_states_out?.map(({ sample_id }) => sample_id) ?? null;
      this.$apollo.mutate({
        mutation: require("@/graphql/chemistry/experiment/delete_exp_op.gql"),
        variables: {
          id: op_id,
          sample_ids: sample_ids_in_ss_out,
        },
        update(cache, { data: { delete_operation_by_pk: deleted_op } }) {
          cache.evict(cache.identify(deleted_op));
        },
      }).then(() => {
        this.$root.$emit("toast", "Operation deleted successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", error);
      });
    },
    /**
     * Provided to children.
     * Introduced to avoid dealing with parsing 'path' to look if any new compounds were introduced to protocol.
     * Takes care of the fact, that in the local buffer there may be more than 1 newely created protocol_compounds
     *
     * @param {ProtocolCompound} protocol_compound - containing 'compound_id' and 'compound_role' fields
     */
    add_protocol_compound(protocol_compound) {
      this.disable_all_but("synthesis_settings");

      this.update_change_counters();
      const trimmed_protocol_compound = {
        ...(!("compound_config" in protocol_compound) && { compound_config: cloneDeep(DEFAULT_COMPOUND_CONFIG) }),
        ...protocol_compound,
      };
      const new_protocol_compound = this.mark_as_local_new(trimmed_protocol_compound);
      if (!("protocol" in this.current_experiment)) {
        this.current_experiment.protocol = this.mark_as_local_new({
          operation_type_key: this.original_experiment.operation_type_key,
          protocol_compounds: [],
        });
      }
      this.current_experiment.protocol.protocol_compounds.push(new_protocol_compound);
      for (const operation of this.current_experiment.operations) {
        operation.protocol.protocol_compounds.push(cloneDeep(new_protocol_compound));
        if (protocol_compound.compound_role === "product") {
          this.created_sss_out = [...this.created_sss_out, this.gen_ss_w_sample(new_protocol_compound.compound_id, new_protocol_compound.compound.name)];
          operation.sample_states_out = this.created_sss_out;
        }
      }
      this.update_op_vars_template(false, true);
      this.rerun_variable_transformers();
    },
    /**
     * Provided to children.
     * Introduced to avoid dealing with parsing 'path' to look if any variables were introduced to protocol.vars.
     * Takes care of the fact, that in the local buffer there may be more than 1 newely created protocol_compounds
     */
    add_method_var(var_name, var_val) {
      this.disable_all_but("method_editor");
      this.update_change_counters();

      this.current_experiment.protocol.vars = {
        ...this.current_experiment.protocol.vars,
        [var_name]: var_val,
      };
      this.update_op_vars_template(true, false);
      this.rerun_variable_transformers();
    },
    /**
     * Convenience function for children to push a delete-change to breaking changes buffer
     * @param {Number} op_id
     * @param {Number} protocol_compound_id - index in the protocol.protocol_compounds array of the current experiment state
     */
    delete_protocol_compound(protocol_compound_id) {
      this.disable_all_but("synthesis_settings");
      this.update_change_counters();

      // if the protocol_compound at specified path is not local
      // then the change should be doubled in the mutations to be dispatched
      const protocol_compound_ind = this.current_experiment.protocol.protocol_compounds.findIndex(({ id }) => id === protocol_compound_id);
      const protocol_compound_to_be_deleted = this.current_experiment.protocol.protocol_compounds[protocol_compound_ind];
      if (!this.is_local_only(protocol_compound_to_be_deleted.id)) {
        this.protocol_compound_changes.deletions.push(protocol_compound_to_be_deleted);
      }
      this.current_experiment.protocol.protocol_compounds.splice(protocol_compound_ind, 1);
      for (const operation of this.current_experiment.operations) {
        operation.sample_states_out = operation.sample_states_out.filter((ss) => !(!ss?.id && protocol_compound_to_be_deleted.compound_id === ss.sample.compound_id));
        operation.protocol.protocol_compounds.splice(protocol_compound_ind, 1);
      }

      this.update_op_vars_template(false, true);
      this.rerun_variable_transformers();
    },
    /**
     * Convenience function for children to push a delete-change to breaking changes buffer.
     * Removes the specified variable from the 'g_op.protocol.vars' and every operation's 'protocol.vars' objects.
     * @param {Number} op_id
     * @param {Number} var_name
     */
    delete_method_var(var_name) {
      this.disable_all_but("method_editor");
      this.update_change_counters();

      const entries_w_o_field = (obj, field_to_remove) => Object.entries(obj).filter(([old_var_name]) => old_var_name !== field_to_remove);

      const new_vars_entries = entries_w_o_field(this.current_experiment.protocol.vars, var_name);

      this.current_experiment.protocol.vars = Object.fromEntries(new_vars_entries);
      for (const operation of this.current_experiment.operations) {
        operation.protocol.vars = Object.fromEntries(entries_w_o_field(operation.protocol.vars, var_name));
      }

      this.update_op_vars_template(true, false);
      this.rerun_variable_transformers();
    },

    /**
     * Compares last saved operation meta with it's current state.
     * Extracts new overrides object.
     *
     * Both original_operation and current_operation hold a 'protocol' object with all the overrides.
     * @param {OperationWProtocol} original_operation - the operatoin as it were saved the last time.
     * @param {OperationWProtocol} current_operation - current state with new protocol object.
     * @returns Operation - operation object w/ updated meta.parameters.variable_overrides
     */
    get_new_meta_w_overrides(original_operation, current_operation) {
      const { protocol: original_protocol, meta: original_meta } = original_operation,
        { protocol: current_protocol, meta: current_meta } = current_operation;

      const new_overrides = {};
      const new_protocol_compound_vars = original_meta?.parameters?.variable_overrides?.protocol_compound_vars ?? {};

      // adding protocol_compound to the protocol_compound_vars map
      for (const [p_cpd_ind, current_p_cpd] of Object.entries(current_protocol.protocol_compounds)) {
        const changes_present = !isEqual(original_protocol.protocol_compounds[p_cpd_ind]?.compound_config, current_p_cpd.compound_config);
        const override_not_empty = !isEqual({}, current_p_cpd.compound_config);
        if (changes_present && override_not_empty) {
          new_protocol_compound_vars[p_cpd_ind] = current_p_cpd.compound_config;
        }
      }
      new_overrides.protocol_compound_vars = new_protocol_compound_vars;

      new_overrides.method_vars = "vars" in current_protocol && !isEqual(original_protocol.vars, current_protocol.vars)
        ? current_protocol.vars
        : original_protocol.vars;

      const new_meta = {
        ...original_meta,
        parameters: {
          ...current_meta?.parameters,
          variable_overrides: new_overrides,
        },
        properties: {
          ...current_meta?.properties,
        },
      };

      return new_meta;
    },

    /**
     * Generates hasura-compatible group_operation-upsert payload.
     * Takes a snapshot of the experiment-json (each operation w/ a local-only conveniece field in 'protocol').
     * Trims it of local-only things and adds utility json-layers, like 'data' and 'on_conflict'.
     *
     * @param {Array} current_experiment
     * @param {Boolean} breaking - true for upsert of an experiment with a new protocol (saving breaking changes).
     */
    gen_experiment_upsert(current_experiment, breaking = false, keep_old_protocol = false) {
      const protocol_compounds_pruned = [];

      for (const p_compound of current_experiment.protocol?.protocol_compounds ?? []) {
        const new_comp = { ...p_compound };
        delete new_comp.compound;
        delete new_comp.__typename;
        if (this.breaking_changes_ctr > 0 && !keep_old_protocol) {
          delete new_comp.protocol_id;
          delete new_comp.id;
        }
        if (this.is_local_only(new_comp.id)) {
          delete new_comp.id;
        }
        protocol_compounds_pruned.push(new_comp);
      }

      const id_defined = "id" in current_experiment.protocol;
      const exists_in_DB = !this.is_local_only(current_experiment.protocol.id);

      const protocol_nested_insert = {
        data: {
          ...(!breaking && id_defined && exists_in_DB && { id: current_experiment.protocol.id }),
          operation_type_key: current_experiment.operation_type.key,
          method: current_experiment.protocol.method,
          vars: current_experiment.protocol.vars ?? {},
          ...(protocol_compounds_pruned.length > 0 && {
            protocol_compounds: {
              data: protocol_compounds_pruned,
              on_conflict: {
                constraint: "protocol_compound_pkey",
                update_columns: ["compound_role", "compound_config"],
              },
            },
          }),
        },
        on_conflict: { constraint: "protocol_pkey", update_columns: ["method", "vars"] },
      };

      return {
        id: current_experiment.id,
        operation_type_key: current_experiment.operation_type.key,
        operations: {
          data: this.prune_operations(current_experiment.operations),
          on_conflict: { constraint: "operation_pkey1", update_columns: ["meta"] },
        },
        protocol: protocol_nested_insert,
        ...(current_experiment.meta && { meta: current_experiment.meta }),
      };
    },
    /**
      * Removes __typename to keep hasura happy
      * removes protocol, cuz it's for frontend purposes only
    */
    prune_operations(ops) {
      return ops.map((op, ind) => {
        const new_op = cloneDeep(op);
        delete new_op.__typename;
        new_op.meta = this.get_new_meta_w_overrides(this.original_experiment.operations[ind], op);
        delete new_op.protocol;
        new_op.sample_states_out = {
          data: new_op.sample_states_out
            .filter((ss) => !ss?.id && !this.protocol_compound_changes.deletions.some(({ compound_id }) => compound_id === ss.sample.compound_id))
            .map((ss) => ({
              status: ss.status,
              sample: {
                data: {
                  compound_id: ss.sample.compound_id, name: ss.sample.name,
                },
              },
            }
            )),
        };
        if (new_op.sample_states_out.data.length === 0) delete new_op.sample_states_out;
        delete new_op.sample_states_in;
        delete new_op.operation_type;
        delete new_op.check_in_exists;
        return new_op;
      });
    },
    /**
     * Called when user clicks the "save"-button in the WidgetWrapper.
     * If breaking_changes object is not empty, the modal should be shown.
     * All current change-accumulators should be morphed into mutation payloads.
     * Mutations - applied.
     * group_operation-query - refetched
     *  - original_op_vars_template - reevaluated
     *  - "disable"-flags - lowered
     *
     * works with this.current_experiment
     */
    on_save() {
      // if there are breaking changes, talk to the user, and return when they've decided
      // whether they want to post a new verion of the protocol, or create a fully new protocol
      if (this.breaking_changes_ctr > 0) {
        this.show_new_protocol_modal = true;
        return;
      }
      // otherwise, turn current_experiment into upsert-mutation payload and dispatch it
      const experiment_upsert_payload = this.gen_experiment_upsert(this.current_experiment);
      this.$apollo.mutate({
        mutation: require("@/graphql/ht/group_operation/upsert.gql"),
        variables: {
          object: experiment_upsert_payload,
        },
      }).then(() => {
        this.$root.$emit("toast", "Protocol changes saved successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", "Couldn't save changes to protocol", error);
      });

      // also check if some of the previously existing protocol_compounds
      // got deleted in the process
      if (this.protocol_compound_changes.deletions.length > 0 || this.protocol_compound_changes.ex_products.length > 0) this.delete_p_cpds_w_samples();
      this.created_sss_out = [];
    },
    /**
     * @param {"id" | "sample_id"} id_field
     * @param {Array} cpd_ids
     */
    arr_to_ids(id_field, cpd_ids) {
      return this.current_experiment.operations.flatMap(({ sample_states_out }) => sample_states_out
        .filter(({ sample }) => cpd_ids.has(sample.compound_id))
        .map((ss) => ss[id_field]));
    },
    /**
     * Called when the "deletions" or "ex_products" fields contain protocol compounds.
     */
    delete_p_cpds_w_samples() {
      const deleted_p_cpds = this.protocol_compound_changes.deletions;
      const cpd_ex_products = this.protocol_compound_changes.ex_products;
      const product_p_cpd_to_delete = deleted_p_cpds.filter(({ compound_role }) => compound_role === "product") ?? [];
      const cpd_ids_to_delete = new Set(deleted_p_cpds.map(({ compound_id }) => compound_id));
      const cpd_ids_to_delete_s_w_ss = new Set([
        ...cpd_ex_products
          .filter(({ compound_id }) => !cpd_ids_to_delete.has(compound_id))
          .map(({ compound_id }) => compound_id),
        ...product_p_cpd_to_delete.map(({ compound_id }) => compound_id),
      ]);

      this.$apollo.mutate({
        mutation: require("@/graphql/chemistry/experiment/delete_p_cpds_w_samples.gql"),
        variables: {
          p_cpd_ids: deleted_p_cpds.map(({ id }) => id),
          ss_ids_to_delete: this.arr_to_ids("id", cpd_ids_to_delete_s_w_ss),
          s_ids_to_delete: this.arr_to_ids("sample_id", cpd_ids_to_delete_s_w_ss),
          delete_p_cpd: deleted_p_cpds.length > 0,
          delete_samples_w_states: product_p_cpd_to_delete.length > 0 || cpd_ex_products.length > 0,
        },
      }).then(() => {
        this.$root.$emit("toast", "Unbound unused connections having a product role from the protocol");
      }).catch((error) => {
        this.$root.$emit("toast_error", "Failed to detach protocol of unused connections with product role.", error);
      }).finally((() => {
        this.protocol_compound_changes.deletions = [];
        this.protocol_compound_changes.ex_products = [];
      }));
    },
    /**
     * Called when one of the widgets emits the "reset"-event.
     * Which means that all current unsaved changes should be discarded.
     * All disabled widgets should no longer be disabled.
     */
    on_reset() {
      this.confirmation_dialog = true;
      this.action_to_confirm = "reset";
    },
    on_close() {
      this.confirmation_dialog = true;
      this.action_to_confirm = "close";
    },
    on_reset_confirm_dialog_update(confirmed) {
      if (!confirmed) {
        this.confirmation_dialog = false;
        return;
      }

      this.op_vars_template = cloneDeep(this.original_op_vars_template);

      // reset local accumulators of changes
      this.protocol_changes_ctr = 0;
      this.breaking_changes_ctr = 0;
      this.current_experiment = cloneDeep(this.original_experiment);
      this.rerun_variable_transformers();
      for (const w_type of Object.keys(this.disabled_widgets_map)) {
        this.disabled_widgets_map[w_type] = false;
      }
      this.confirmation_dialog = false;
    },
    /**
     * Called when the user has confirmed that they want to create a new version of the protocol
     * or a new protocol altogether.
     * All unsaved unbreaking changes are commited to the new protocol.
     * - protocol_changes_buffer applied along with breaking changes
     * - variable_overrides (in operation.meta) are updated by separate 'objects' elements but in the same mutation
     */
    async on_confirm_breaking_changes() {
      // const operations_w_changes = this.apply_protocol_changes_buffer(this.original_experiment.operations, this.protocol_changes_buffer);
      const experiment_upsert_payload = this.gen_experiment_upsert(this.current_experiment, true);

      experiment_upsert_payload.protocol.data.approved = null;
      experiment_upsert_payload.protocol.data.name = this.new_protocol_name;
      experiment_upsert_payload.protocol.data.parent_protocol_id = this.original_experiment.protocol_id;

      await this.$apollo.mutate({
        mutation: require("@/graphql/ht/group_operation/upsert.gql"),
        variables: {
          object: experiment_upsert_payload,
        },
      }).then(() => {
        this.$root.$emit("toast", "New protocol created successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", `Couldn't create a new version of the protocol${error}`);
      }).finally(() => {
        this.new_protocol_name = "";
        this.show_new_protocol_modal = false;
        this.new_protocol_actions_visible = false;
        this.protocol_compound_changes.deletions = [];
        this.protocol_compound_changes.ex_products = [];
        this.created_sss_out = [];
      });
    },

    on_delete_widget(w_index) {
      this.widget_wrappers.splice(w_index, 1);
    },
    on_edit_layout() {
      this.layout_editing_on = true;
    },
    on_cancel_layout_editing() {
      this.new_widgets = [];
      this.layout_editing_on = false;
    },
    /**
     * Called by the draggable component when order of widgets has changed.
     * Only accessible when layout_editing_on
     * @param {Object} dragEventPayload - info w/ new indices { moved: { oldIndex: number, newIndex: number }}
     */
    on_widget_drag({ moved }) {
      shift_and_insert(this.widget_wrappers, moved.oldIndex, moved.newIndex);
    },
    on_add_widget(widget_type_config) {
      this.widget_wrappers.push(widget_type_config);
    },
    on_save_layout_editing() {
      this.loading = true;
      /** ... dispatch mutations here */
      const new_meta = {
        ...this.group_operation_by_pk.meta,
        widgets: this.widget_wrappers,
      };
      this.$apollo.mutate({
        mutation: require("@/graphql/ht/group_operation/update_by_pk.gql"),
        variables: {
          id: this.group_operation_by_pk.id,
          _set: {
            meta: new_meta,
          },
        },
        update(cache, { data }) {
          cache.modify({
            id: cache.identify(data.update_group_operation_by_pk),
            fields: {
              meta() {
                return data.update_group_operation_by_pk.meta;
              },
            },
          });
        },
      }).catch((error) => {
        this.$root.$emit("toast_error", error);
      }).finally(() => {
        this.layout_editing_on = false;
        this.loading = false;
        this.$root.$emit("toast", "Layout updated!");
      });
    },
    on_error(er) {
      this.$root.$emit("toast_error", er);
    },

    /**
     * Updates experiment meta-data. Receives a key-string and its value.
     *
     * @param {"type" | "protocol"} to_update - a field to be updated w/ _set operator
     * @param {String | Number} val - new value of the field
     */
    update_experiment(to_update, val) {
      const _set = {};
      if (to_update === "type") {
        _set.operation_type_key = this.experiment_type;
      }

      const mutation_vars = { id: this.id, is_experiment: true };

      if (to_update === "protocol") {
        if (!val) return;
        _set.protocol_id = val;
        this.protocol_search_input = "";

        const cleaned_ops = this.current_experiment.operations.map((op) => {
          const new_meta = cloneDeep(op.meta);
          if (new_meta?.parameters?.variable_overrides?.protocol_compound_vars) {
            new_meta.parameters.variable_overrides.protocol_compound_vars = {};
          }
          return {
            id: op.id,
            meta: new_meta,
          };
        });
        mutation_vars.do_clean = true;
        mutation_vars.cleaned_ops_vars = cleaned_ops;
      }
      mutation_vars._set = _set;

      this.$apollo.mutate({
        mutation: require("@/graphql/ht/group_operation/update_by_pk.gql"),
        variables: mutation_vars,
        update(cache, { data: { update_group_operation_by_pk } }) {
          const updated_field = Object.keys(_set)[0];
          cache.modify({
            id: cache.identify(update_group_operation_by_pk),
            fields: {
              [updated_field]() {
                return update_group_operation_by_pk[updated_field];
              },
            },
          });

          if (to_update === "protocol") {
            cache.modify({
              id: cache.identify(update_group_operation_by_pk),
              fields: {
                protocol() {
                  return update_group_operation_by_pk.protocol;
                },
              },
            });
          }
        },
      }).then(() => {
        this.$root.$emit("toast", "Experiment configuration updated");
      }).catch((error) => {
        console.error(error);
        this.$root.$emit("toast", `Couldn't update experiment: ${error}`, "error");
      });
    },
    /**
     * Updates experiment's task's fields': assignee, reviewer
     * @param {object} new_meta
     */
    on_task_update(field, val) {
      if (val === this.original_experiment.tasks[0][field]) return;

      const task_object = {
        ...this.task,
        [field]: val === -1 ? null : val,
      };
      delete task_object.__typename;
      delete task_object.creator_user;
      delete task_object.created_by;
      delete task_object.assignee_user;
      delete task_object.reviewer_user;

      this.$apollo.mutate({
        mutation: require("@/graphql/tasks/update_by_pk.gql"),
        variables: { object: task_object },
        update(cache, { data: { insert_task_one } }) {
          cache.modify({
            id: cache.identify(insert_task_one),
            fields: {
              [field]() {
                return insert_task_one[field];
              },
            },
          });
        },
      }).then(({ data: { insert_task_one } }) => {
        this.task[field] = insert_task_one[field];
        this.$root.$emit("toast", "Task updated successfully");
      }).catch((error) => {
        this.$root.$emit("toast_error", `Couldn't update related task:${error}`);
      });
    },
    on_tab_change() {
      this.$router.replace({
        name: "experiment",
        params: { id: this.$route.params.id, tab: this.tab },
      });
    },
    /**
     * @param {Object} synthetic_op - a field to be updated operation by id
     * @param {Object} selected_p_cpd - to update operation variable
     * @param {Object} selected_ss - for this one sample_states_out and meta will be created in a transfer operation
     * @param {Number} actual_delta - check-in result
     * @param {"dV" | "dm"} delta_key - the measured-property delta key
     */
    async setup_checkin_operation(synthetic_op, selected_p_cpd, selected_ss, actual_delta, delta_key) {
      // TODO: call 'update_operation_variable' and save changes to DB
      // if (different_from_protocol) {
      //   await this.update_overrides_before_checkin(synthetic_op_id, selected_p_cpd, actual_amount);
      // }

      // first, fetch the current state of the now-ss.
      // if it is different, use the newer version and its operation_out_id
      // in the ideal world it is the same as selected_ss
      const now_ss = await this.$apollo.query({
        query: require("@/graphql/ht/sample_state/get_ss.gql"),
        variables: {
          where: {
            sample_id: { _eq: selected_ss.sample_id },
            label_id: selected_ss.label_id !== null ? { _eq: selected_ss.label_id } : { _is_null: true },
            status: { _eq: "now" },
            operation_in: { status: { _eq: "completed" } },
          },
          limit: 1,
        },
      }).then((resp) => resp.data.sample_state[0]);

      const checkin_transfer_op_payload = {
        status: "planned",
        operation_type_key: "transfer",
        meta: {},
        sample_states_in: { // this existing ss should get a new 'operation_out_id'
          data: pick(now_ss, ["id", "status", "location_id", "sample_id", "housing_state_id", "label_id"]),
          on_conflict: { constraint: "sample_state_pkey", update_columns: ["operation_out_id"] },
        },
        sample_states_out: {
          data: [
            { // stays on shelf, must be connected to all the previously planned check-ins
              sample_id: now_ss.sample_id,
              location_id: now_ss.location_id,
              label_id: now_ss.label_id,
              status: "future",
              operation_out_id: now_ss.operation_out_id ?? null, // <- previously planned check-in op; this is crucial
              meta: { properties: {} },
            },
            { // goes into the experiment
              sample_id: now_ss.sample_id,
              location_id: now_ss.location_id,
              status: "future",
              meta: { properties: {} },
              operation_out_id: synthetic_op.id,
            },
          ],
        },
      };
      const checkin_transfer_op = await this.$apollo
        .mutate({
          mutation: require("@/graphql/ht/operation/insert.gql"),
          variables: {
            objects: [checkin_transfer_op_payload],
          },
          update(cache, { data: { insert_operation: { returning: ops } } }) {
            const [upserted_transfer_op] = ops;
            const upserted_exp_ss_in = cloneDeep(upserted_transfer_op.sample_states_out.find((ss_out) => ss_out.operation_out_id === synthetic_op.id));
            upserted_exp_ss_in.operation_in = omit(upserted_transfer_op, "sample_states_out");
            // add new ss_in to the 'synthetic_op'
            cache.modify({
              id: cache.identify({ id: synthetic_op.id, __typename: "operation" }),
              fields: {
                sample_states_in(cached_sss_in) {
                  return [...cached_sss_in, upserted_exp_ss_in];
                },
              },
            });
          },
        })
        .then((resp) => resp.data.insert_operation.returning[0])
        .catch((error) => {
          this.$root.$emit("toast_error", `Couldn't create check-in transfer ${error}`);
        });
      const experiment_ss_in = checkin_transfer_op.sample_states_out[1];
      const new_meta_for_received_op = {
        properties: {
          [delta_key]: {
            [experiment_ss_in.id]: actual_delta,
          },
        },
      };

      await this.$apollo // Update meta in received operation
        .mutate({
          mutation: require("@/graphql/ht/operation/update_by_pk.gql"),
          variables: {
            id: checkin_transfer_op.id,
            set: { meta: new_meta_for_received_op },
          },
          update(cache, { data: { update_operation_by_pk: updated_transfer_op } }) {
            cache.modify({
              id: cache.identify(updated_transfer_op),
              fields: {
                meta() { return updated_transfer_op.meta; },
              },
            });
          },
        }).then(() => {
          this.$root.$emit("toast", "Sample check-in planned!");
        }).catch((error) => {
          this.$root.$emit("toast_error", `Couldn't record the consumed amount ${error}`);
        });

      // update preferences map
      const old_compound_sample_map = this.$usher.get_preferences("compound_sample_map") ?? {};
      if (old_compound_sample_map[selected_p_cpd.compound_id] !== selected_ss.sample_id) {
        await this.$usher.write_preference(
          "compound_sample_map",
          { ...old_compound_sample_map, [selected_p_cpd.compound_id]: selected_ss.sample_id },
        );
      }
    },
    async checkout_sample(synth_op_id, sample_id) {
      const synth_op = this.current_experiment.operations.find((op) => op.id === synth_op_id);
      const checkin_transfer_op = synth_op.sample_states_in.find((ss) => ss.sample_id === sample_id).operation_in_id;
      this.$apollo.mutate({
        mutation: require("@/graphql/chemistry/experiment/checkout_sample.gql"),
        variables: {
          operation_ids: [synth_op_id],
          sample_id,
        },
        update(cache) {
          cache.evict(cache.identify({ id: checkin_transfer_op, __typename: "operation" }));
          cache.gc(); // calling gc to get rid of the input_sss for that evicted check-in operation
        },
      }).then(() => {
        this.$root.$emit("toast", "Sample checked out from the experiment");
      }).catch((error) => {
        this.$root.$emit("toast_error", `Checkout failed: ${error}`);
      });
    },
    /**
     * Dispatches a mutation that does 3 things at once:
     * - queues up the check-in transfers in the histories of all the samples in the experiment
     * - calculates meta.properties for the sss_out of all of those checkin-transfsers
     * - sets status="complete" for every checkin-transfer
     */
    complete_checkin() {
      const check_in_transfer_ops = this.current_experiment.operations
        .flatMap((op) => op.sample_states_in.map((ss_in) => ss_in.operation_in));

      const exp_sss_in = this.current_experiment.operations.flatMap((op) => op.sample_states_in);
      const remaining_sss = check_in_transfer_ops.flatMap((op) => op.sample_states_out);

      const transfer_ops_sss_out = [
        ...exp_sss_in,
        ...remaining_sss,
      ];

      const group_operation_id = this.current_experiment.id;

      this.$apollo
        .mutate({
          mutation: require("@/graphql/chemistry/experiment/complete_check_in.gql"),
          variables: {
            group_operation_id,
            operation_ids: check_in_transfer_ops.map((op) => op.id),
          },
          update(cache) {
            for (const op of check_in_transfer_ops) {
              cache.modify({
                id: cache.identify({ id: op.id, __typename: "operation" }),
                fields: {
                  status() { return "completed"; },
                },
              });
            }
            // experiment's input sss must be in status "now"
            for (const ss_out of transfer_ops_sss_out) {
              cache.modify({
                id: cache.identify({ id: ss_out.id, __typename: "sample_state" }),
                fields: {
                  status() { return "now"; },
                },
              });
            }
          },
        }).then(() => {
          this.$root.$emit("toast", "Check-in completed!");
        })
        .catch((error) => {
          this.$root.$emit("toast_error", `Check-in completion failed: ${error}`);
        });
    },
    /**
     * Dispatches a mutation that sets status="planned" for all check-in-transfers
     */
    async rollback_checkin() {
      const check_in_transfer_ops = this.current_experiment.operations
        .flatMap((op) => op.sample_states_in.map((ss_in) => ss_in.operation_in));
      const exp_sss_in = this.current_experiment.operations.flatMap((op) => op.sample_states_in);
      const remaining_sss = check_in_transfer_ops.flatMap((op) => op.sample_states_out);

      const transfer_ops_sss_out = [
        ...exp_sss_in,
        ...remaining_sss,
      ];

      this.$apollo
        .mutate({
          mutation: require("@/graphql/chemistry/experiment/rollback_checkin.gql"),
          variables: {
            operation_ids: check_in_transfer_ops.map((op) => op.id),
          },
          update(cache) {
            for (const op of check_in_transfer_ops) {
              cache.modify({
                id: cache.identify({ id: op.id, __typename: "operation" }),
                fields: {
                  status() { return "planned"; },
                },
              });
            }
            // experiment's input sss must be in status "future" now
            for (const ss_out of transfer_ops_sss_out) {
              cache.modify({
                id: cache.identify({ id: ss_out.id, __typename: "sample_state" }),
                fields: {
                  status() { return "future"; },
                },
              });
            }
          },
        }).then(() => {
          this.$root.$emit("toast", "Check-in rollback successful");
        })
        .catch((error) => {
          this.$root.$emit("toast_error", `Check-in rollback failed: ${error}`);
        });
    },
    gen_ss_w_sample(cpd_id, cpd_name) {
      return {
        status: "future",
        sample: {
          status: "planned",
          compound_id: cpd_id,
          name: cpd_name,
        },
      };
    },
    create_dataset(type) {
      this.$apollo
        .mutate({
          mutation: require("@/graphql/chemistry/dataset/insert_one.gql"),
          variables: {
            object: {
              type_key: type,
              group_operation_id: this.group_operation_by_pk.id,
              files: [],
              meta: {},
              description: "",
            },
          },
          update(cache, { data: { dataset: inserted_dataset } }) {
            cache.modify({
              id: cache.identify({ id: inserted_dataset.group_operation_id, __typename: "group_operation" }),
              fields: {
                datasets(cached_datasets) {
                  return [...cached_datasets, inserted_dataset];
                },
              },
            });
          },
        })
        .then(() => {
          this.$root.$emit("toast", "Dataset created.");
        })
        .catch((error) => {
          this.$root.$emit("toast", `An error occurred: ${error}`, "error");
        });
    },
    delete_dataset(id) {
      this.$apollo.mutate({
        mutation: require("@/graphql/chemistry/dataset/delete_by_pk.gql"),
        variables: {
          id,
        },
        update(cache, { data: { delete_dataset_by_pk: deleted_dataset } }) {
          cache.evict(cache.identify(deleted_dataset));
        },
      })
        .then(() => {
          this.$root.$emit("toast", "Deleted dataset");
        }).catch((error) => {
          this.$root.$emit("toast_error", `Couldn't delete dataset${error}`);
        });
    },
  },
};
</script>
<style lang="sass" scoped>
.approved,
.unapproved
  font-size: 11px
.approved
  color: green

.unapproved
  color: grey

.parent-experiment-type
  font-size: 11px
  color: grey
.custom-tab
  margin-right: -1px
  & .v-tab
    height: 100%
.heading_block
  padding: 5px
  width: calc(100px + 6vw)
.text-bold
  font-weight: bolder
.v-progress-linear.progress-linear-in-tabs
  margin-left: 12px
  margin-right: 12px
  transform: translateY(-100%)

.fixed-width
  width: 300px
.fixed-height
  height: 40px
  align-items: center
  justify-content: left
  display: flex
.follows
  display: flex
  flex-direction: column
.full_width
  width: 100%
.badge-container
  justify-self: end
  display: flex
  flex-direction: row
  justify-items: end
  gap: 16px
  font-size: 14px
  width: 450px
  max-width: 450px

.linked-operation-container
  width: 320px
  flex-grow: 1
  max-width: 420px

.edit-layout-buttons
  display:  flex
  flex-direction: row
  flex-wrap: nowrap
  justify-content: flex-start
  align-items: center
  align-content: center
  gap: 8px

.widget + .widget
  margin-top: 16px
.btn-create
  top: -60px
  position: absolute
  right: 0
</style>
