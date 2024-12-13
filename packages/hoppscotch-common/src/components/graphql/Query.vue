<template>
  <div class="flex items-center space-x-2">
    <label>{{ t("request.max_nesting_depth") }}: </label>
    <!-- provides limits on the minimum and maximum for the max_nesting_depth setting -->
    <input
      type="range"
      min="1"
      max="10"
      v-model="max_nesting_depth"
      @input="generateQueryWithMaxDepth"
    />
    <span>{{ max_nesting_depth }}</span>
  </div>
  <div
    class="sticky top-sidebarPrimaryStickyFold z-10 flex items-center justify-between border-y border-dividerLight bg-primary pl-4"
  >
    <label class="font-semibold text-secondaryLight">
      {{ t("request.query") }}
    </label>
    <div class="flex">
      <HoppButtonSecondary
        v-if="subscriptionState === 'SUBSCRIBED'"
        v-tippy="{
          theme: 'tooltip',
          delay: [500, 20],
          allowHTML: true,
        }"
        :title="`${t('request.stop')}`"
        :label="`${t('request.stop')}`"
        :icon="IconStop"
        class="!hover:text-accentDark rounded-none !text-accent"
        @click="unsubscribe()"
      />

      <HoppButtonSecondary
        v-if="selectedOperation && subscriptionState !== 'SUBSCRIBED'"
        v-tippy="{
          theme: 'tooltip',
          delay: [500, 20],
          allowHTML: true,
        }"
        :title="`${t('request.run')} <kbd>${getSpecialKey()}</kbd><kbd>G</kbd>`"
        :label="`${selectedOperation.name?.value ?? t('request.run')}`"
        :icon="IconPlay"
        :disabled="!selectedOperation"
        class="!hover:text-accentDark rounded-none !text-accent"
        @click="runQuery(selectedOperation)"
      />

      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip', delay: [500, 20], allowHTML: true }"
        :title="`${t(
          'request.save'
        )} <kbd>${getSpecialKey()}</kbd><kbd>S</kbd>`"
        :label="`${t('request.save')}`"
        :icon="IconSave"
        class="rounded-none"
        @click="saveRequest"
      />
      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip' }"
        to="https://docs.hoppscotch.io/documentation/features/graphql-api-testing"
        blank
        :title="t('app.wiki')"
        :icon="IconHelpCircle"
      />
      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip' }"
        :title="t('action.clear_all')"
        :icon="IconTrash2"
        @click="clearGQLQuery()"
      />
      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip' }"
        :title="t('state.linewrap')"
        :class="{ '!text-accent': WRAP_LINES }"
        :icon="IconWrapText"
        @click.prevent="toggleNestedSetting('WRAP_LINES', 'graphqlQuery')"
      />
      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip' }"
        :title="t('action.prettify')"
        :icon="prettifyQueryIcon"
        @click="prettifyQuery"
      />
      <HoppButtonSecondary
        v-tippy="{ theme: 'tooltip' }"
        :title="t('action.copy')"
        :icon="copyQueryIcon"
        @click="copyQuery"
      />
    </div>
  </div>
  <div ref="queryEditor" class="flex flex-1 flex-col"></div>
</template>

<script setup lang="ts">
import IconPlay from "~icons/lucide/play"
import IconStop from "~icons/lucide/stop-circle"
import IconSave from "~icons/lucide/save"
import IconHelpCircle from "~icons/lucide/help-circle"
import IconTrash2 from "~icons/lucide/trash-2"
import IconCopy from "~icons/lucide/copy"
import IconCheck from "~icons/lucide/check"
import IconInfo from "~icons/lucide/info"
import IconWand from "~icons/lucide/wand"
import IconWrapText from "~icons/lucide/wrap-text"
import { onMounted, reactive, ref, markRaw, watch } from "vue"
import { copyToClipboard } from "@helpers/utils/clipboard"
import { useCodemirror } from "@composables/codemirror"
import { useI18n } from "@composables/i18n"
import { refAutoReset, useVModel } from "@vueuse/core"
import { useToast } from "~/composables/toast"
import { getPlatformSpecialKey as getSpecialKey } from "~/helpers/platformutils"
import * as gql from "graphql"
import { createGQLQueryLinter } from "~/helpers/editor/linting/gqlQuery"
import queryCompleter from "~/helpers/editor/completion/gqlQuery"
import { selectedGQLOpHighlight } from "~/helpers/editor/gql/operation"
import { debounce } from "lodash-es"
import { ViewUpdate } from "@codemirror/view"
import { defineActionHandler } from "~/helpers/actions"
import {
  schema,
  socketDisconnect,
  subscriptionState,
} from "~/helpers/graphql/connection"
import { useNestedSetting } from "~/composables/settings"
import { toggleNestedSetting } from "~/newstore/settings"
import { generateQuery } from "@helpers/graphql/queryBuilder"
import { GraphQLSchema } from "graphql"

const generatedQuery = ref<string | null>(null)

// Template refs
const max_nesting_depth = ref(3)
const queryEditor = ref<any | null>(null)

const t = useI18n()
const toast = useToast()

const props = defineProps<{
  modelValue: string
}>()

const emit = defineEmits<{
  (e: "save-request"): void
  (e: "update:modelValue", val: string): void
  (e: "run-query", definition: gql.OperationDefinitionNode | null): void
}>()

const copyQueryIcon = refAutoReset<typeof IconCopy | typeof IconCheck>(
  IconCopy,
  1000
)
const prettifyQueryIcon = refAutoReset<
  typeof IconWand | typeof IconCheck | typeof IconInfo
>(IconWand, 1000)

const WRAP_LINES = useNestedSetting("WRAP_LINES", "graphqlQuery")

const selectedOperation = ref<gql.OperationDefinitionNode | null>(null)

const gqlQueryString = useVModel(props, "modelValue", emit)

const debouncedOnUpdateQueryState = debounce((update: ViewUpdate) => {
  const selectedPos = update.state.selection.main.head
  const queryString = update.state.doc.toJSON().join(update.state.lineBreak)

  try {
    const operations = gql.parse(queryString)
    if (operations.definitions.length === 1) {
      selectedOperation.value = operations
        .definitions[0] as gql.OperationDefinitionNode
      return
    }

    selectedOperation.value =
      (operations.definitions.find((def) => {
        if (def.kind !== "OperationDefinition") return false
        const { start, end } = def.loc!
        return selectedPos >= start && selectedPos <= end
      }) as gql.OperationDefinitionNode) ?? null
  } catch (error) {
    // console.error(error)
  }
}, 300)

onMounted(() => {
  try {
    const operations = gql.parse(gqlQueryString.value)
    if (operations.definitions.length) {
      selectedOperation.value = operations
        .definitions[0] as gql.OperationDefinitionNode
      return
    }
  } catch (error) {}
})

useCodemirror(
  queryEditor,
  gqlQueryString,
  reactive({
    extendedEditorConfig: {
      mode: "graphql",
      placeholder: `${t("request.query")}`,
      lineWrapping: WRAP_LINES,
    },
    linter: createGQLQueryLinter(schema),
    completer: queryCompleter(schema),
    environmentHighlights: false,
    additionalExts: [markRaw(selectedGQLOpHighlight)],
    onUpdate: debouncedOnUpdateQueryState,
  })
)

const clearGQLQuery = () => {
  gqlQueryString.value = ""
}

// defines the schema and generatedQuery variables for use below
const docSchema: GraphQLSchema | null = null

// generate query with max_nesting_depth
const generateQueryWithMaxDepth = () => {
  if (docSchema) {
    generatedQuery.value = generateQuery(docSchema, max_nesting_depth.value)
    gqlQueryString.value = generatedQuery.value || ""
  } else {
    toast.error(t("error.schema_not_loaded"))
  }
}

// add watcher to allow the user to update max_nesting_depth without needing to refresh the page
watch(max_nesting_depth, () => generateQueryWithMaxDepth(), { immediate: true })

// copy the query to the clipboard
const copyQuery = () => {
  copyToClipboard(gqlQueryString.value)
  copyQueryIcon.value = IconCheck
  toast.success(t("state.copied_to_clipboard"))
}

// prettify the graphql query
const prettifyQuery = () => {
  try {
    gqlQueryString.value = gql.print(gql.parse(gqlQueryString.value))
    prettifyQueryIcon.value = IconCheck
  } catch {
    toast.error(t("error.gql_prettify_invalid_query"))
  }
}

const runQuery = (operation: any) => {
  console.log("Running query:", operation)
}

const unsubscribe = () => {
  socketDisconnect()
}
const saveRequest = () => {
  emit("save-request")
}

defineActionHandler("editor.format", prettifyQuery)
</script>
