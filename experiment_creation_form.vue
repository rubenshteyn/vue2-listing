<template>
  <v-container v-if="edited_value">
    <v-row>
      <v-col>
        <v-text-field
          v-model="edited_value.meta.name"
          :rules="[
            (v) => !!v || 'Name required'
          ]"
          dense
          outlined
          label="Experiment name"
          :disabled="disabled"
        />
      </v-col>
    </v-row>
    <v-row>
      <v-col>
        <v-autocomplete
          v-if="!empty_prot"
          v-model="edited_value.protocol"
          class="protocol_search"
          :rules="[
            (v) => !!v || 'Protocol required'
          ]"
          label="Protocol"
          outlined
          dense
          :items="protocol_options"
          no-data-text="No protocols found"
          :search-input.sync="protocol_search_input"
          :disabled="!edited_value.operation_type_key || empty_prot"
          return-object
          @change="edited_value.protocol = $event"
        >
          <template #selection="{ item: { name } }">
            {{ name }}
          </template>
          <template #item="{ item: { name }}">
            {{ name }}
          </template>
        </v-autocomplete>
        <v-text-field
          v-else
          v-model="edited_value.protocol.name"
          outlined
          dense
          label="Protocol name"
          :rules="[
            (v) => !!v || 'Protocol name required'
          ]"
        />
      </v-col>
      <v-col>
        <v-checkbox
          v-model="empty_prot"
          class="create_empty_prot h-50 mt-0"
          label="New empty protocol"
        />
      </v-col>
    </v-row>
    <v-row class="d-flex">
      <v-col class="d-flex">
        <v-autocomplete
          v-model="edited_value.tasks[0].assignee"
          :rules="[
            (v) => !!v || 'Assignee must be set'
          ]"
          :search-input.sync="user_search_query"
          :items="users"
          dense
          outlined
          item-value="id"
          item-text="first_name,last_name"
          label="Assignee"
          :filter="(item, query) => (`${item.first_name} ${item.last_name}`).toLowerCase().includes(query.toLowerCase())"
          :disabled="
            disabled ||
              !$user.field_accessible('update', 'task', 'assignee') ||
              ['in-progress', 'completed'].includes(edited_value.tasks[0].status)
          "
        >
          <template #item="{ item }">
            {{ get_user_label(item) }}
          </template>
          <template #selection="{ item }">
            {{ get_user_label(item) }}
          </template>
        </v-autocomplete>
      </v-col>
      <v-col>
        <v-menu v-model="deadline_picker">
          <template #activator="{ on, attrs }">
            <v-text-field
              v-bind="attrs"
              outlined
              dense
              :value="computed_deadline"
              hide-details
              label="Task deadline (optional)"
              :disabled="disabled"
              v-on="on"
            />
          </template>
          <v-date-picker
            v-model="edited_value.tasks[0].deadline"
            no-title
            scrollable
            :disabled="disabled"
          />
        </v-menu>
      </v-col>
    </v-row>
    <v-row class="mt-10 mb-0">
      <v-col class="d-flex">
        <v-hover v-slot="{ hover }">
          <v-btn
            depressed
            class="btn-create blue white--text"
            :class="{ 'darken-2': hover}"
            :ripple="false"
            :disabled="create_disabled"
            @click="on_save"
          >
            {{ dialog_mode ? "Create" : "Save" }}
          </v-btn>
        </v-hover>
        <v-spacer />
        <v-hover v-slot="{ hover }">
          <v-btn
            depressed
            :ripple="false"
            :color="hover ? 'info' : 'grey darken-2'"
            outlined
            @click="on_cancel"
          >
            {{ dialog_mode ? "Cancel" : "Go to experiments" }}
          </v-btn>
        </v-hover>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import cloneDeep from "lodash/cloneDeep";
import { format_date } from "@/lib/date.js";

export default {
  name: "ExperimentCreationForm",
  props: {
    value: {
      type: Object,
      required: true,
    },
    /**
      * Used to disable the form (based on custom-checks)
      */
    disabled: {
      type: Boolean,
      default: false,
    },
  },
  apollo: {
    users: {
      query: require("@/graphql/config/user/search.gql"),
      debounce: 500,
      skip() {
        return !this.edited_value;
      },
      variables() {
        const snippets = (this.user_search_query || "").split(" ");
        return {
          where: {
            _and: [
              {
                _or: snippets.flatMap(
                  (snippet) => [
                    { username: { _ilike: `%${snippet}` } },
                    { last_name: { _ilike: `%${snippet}%` } },
                    { first_name: { _ilike: `%${snippet}%` } },
                  ],
                ),
              },
              ...this.chemist_role_bool,
            ],
          },
          order_by: { last_name: "asc" },
          limit: 5,
        };
      },
      update({ user }) {
        return [
          ...user,
          this.value.tasks[0].assignee_user,
          {
            first_name: "no",
            id: -1,
            last_name: "assignee",
          },
        ];
      },
    },
    protocol_options: {
      query: require("@/graphql/chemistry/protocol/search.gql"),
      debounce: 300,
      skip() {
        return !this.edited_value.operation_type_key;
      },
      variables() {
        const _and = [
          { operation_type_key: { _eq: this.edited_value.operation_type_key } },
          { approved: { _eq: true } },
        ];
        if (this.protocol_search_input) {
          _and.push({ name: { _ilike: `%${this.protocol_search_input}%` } });
        }
        return {
          where: {
            _and,
          },
        };
      },
      update({ protocol }) { return protocol; },
    },
  },
  data() {
    return {
      protocol_search_input: null,
      chemist_role_bool: [
        {
          user_hasura_roles: {
            hasura_role: {
              key: { _in: ["SyntheticChemist", "superuser"] },
            },
          },
        },
      ],
      statuses: ["wait", "new", "in-progress", "completed"],
      user_search_query: "",
      edited_value: null,
      deadline_picker: false,
      id: 0,
      empty_prot: false,
    };
  },
  computed: {
    dialog_mode() {
      return this.$route.name === "experiment_listing";
    },
    computed_deadline() {
      return this.edited_value.tasks[0].deadline ? format_date(new Date(this.edited_value.tasks[0].deadline)) : null;
    },
    create_disabled() {
      return !(this.edited_value.tasks[0].title
        && this.edited_value.tasks[0].assignee
        && this.edited_value.meta.name
        && this.edited_value.protocol
        && this.edited_value.protocol.name !== "");
    },
  },
  watch: {
    value: {
      deep: true,
      immediate: true,
      handler(v) {
        this.edited_value = cloneDeep(v);
        this.edited_value.tasks[0].status = "new";
      },
    },
    edited_value: {
      deep: true,
      handler(v) {
        if (JSON.stringify(v) !== JSON.stringify(this.value)) {
          this.$emit("input", v);
        }
      },
    },
    empty_prot(new_val) {
      this.edited_value.protocol = new_val === true ? {
        operation_type_key: "synthetic",
        name: "",
        meta: {},
        method: {},
      } : null;
    },
  },
  methods: {
    get_user_label(user) {
      const user_fields = ["first_name", "last_name"];
      return user_fields.reduce((label, field) => `${label} ${user[field]}`, "");
    },
    on_save() {
      this.edited_value.tasks[0].description = `The task accompanying synthetic experiment "${this.edited_value.name}"`;
      this.$emit("confirm", true);
    },
    on_cancel() {
      if (this.dialog_mode) {
        this.$emit("confirm", false);
      }
    },
  },
};
</script>
