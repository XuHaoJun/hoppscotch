<template>
  <div class="flex flex-1 flex-col">
    <div
      v-if="showEventField"
      class="sticky z-10 flex flex-shrink-0 items-center justify-center overflow-x-auto border-b border-dividerLight bg-primary"
      :class="eventFieldStyles"
    >
      <icon-lucide-rss class="svg-icons mx-4 text-accentLight" />
      <input
        id="event_name"
        v-model="eventName"
        class="w-full truncate bg-primary py-2 pr-4"
        name="event_name"
        :placeholder="`${t('socketio.event_name')}`"
        type="text"
        autocomplete="off"
      />
    </div>
    <div
      class="sticky z-10 flex flex-shrink-0 items-center justify-between overflow-x-auto border-b border-dividerLight bg-primary pl-4"
      :class="stickyHeaderStyles"
    >
      <span class="flex items-center">
        <label class="truncate font-semibold text-secondaryLight">
          {{ t("websocket.message") }}
        </label>
        <HoppButtonSecondary
          v-if="props.multiple"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.add_arg')"
          :icon="IconPlus"
          @click="addNewArg"
        />
        <tippy
          interactive
          trigger="click"
          theme="popover"
          :on-shown="() => tippyActions.focus()"
        >
          <HoppSmartSelectWrapper>
            <HoppButtonSecondary
              :label="contentType || t('state.none').toLowerCase()"
              class="ml-2 rounded-none pr-8"
            />
          </HoppSmartSelectWrapper>
          <template #content="{ hide }">
            <div
              ref="tippyActions"
              class="flex flex-col focus:outline-none"
              tabindex="0"
              @keyup.escape="hide()"
            >
              <HoppSmartItem
                v-for="(contentTypeItem, index) in validContentTypes"
                :key="`contentTypeItem-${index}`"
                :label="contentTypeItem"
                :info-icon="
                  contentTypeItem === contentType ? IconDone : undefined
                "
                :active-info-icon="contentTypeItem === contentType"
                @click="
                  () => {
                    contentType = contentTypeItem
                    hide()
                  }
                "
              />
            </div>
          </template>
        </tippy>
      </span>
      <div class="flex">
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip', delay: [500, 20], allowHTML: true }"
          :title="`${t(
            'request.run'
          )} <kbd>${getSpecialKey()}</kbd><kbd>↩</kbd>`"
          :label="`${t('action.send')}`"
          :disabled="!communicationBody || !isConnected"
          :icon="IconSend"
          class="!hover:text-accentDark rounded-none !text-accent"
          @click="sendMessage()"
        />
        <HoppSmartCheckbox
          v-tippy="{ theme: 'tooltip' }"
          :on="clearInputOnSend"
          class="px-2"
          :title="`${t('mqtt.clear_input_on_send')}`"
          @change="clearInputOnSend = !clearInputOnSend"
        >
          {{ t("mqtt.clear_input") }}
        </HoppSmartCheckbox>
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          to="https://docs.hoppscotch.io/documentation/features/realtime-api-testing"
          blank
          :title="t('app.wiki')"
          :icon="IconHelpCircle"
        />
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.clear')"
          :icon="IconTrash2"
          @click="clearContent"
        />
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('state.linewrap')"
          :class="{ '!text-accent': linewrapEnabled }"
          :icon="IconWrapText"
          @click.prevent="linewrapEnabled = !linewrapEnabled"
        />
        <HoppButtonSecondary
          v-if="contentType && contentType == 'JSON'"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.prettify')"
          :icon="prettifyIcon"
          @click="prettifyRequestBody"
        />
        <label for="payload">
          <HoppButtonSecondary
            v-tippy="{ theme: 'tooltip' }"
            :title="t('import.title')"
            :icon="IconFilePlus"
            @click="payload!.click()"
          />
        </label>
        <input
          ref="payload"
          class="input"
          name="payload"
          type="file"
          @change="uploadPayload"
        />
      </div>
    </div>
    <div class="flex flex-1">
      <div
        v-if="props.multiple && messagesBodies.length > 1"
        class="flex-shrink-0 w-20 flex flex-col max-h-full overflow-auto"
      >
        <div
          v-for="(_, index) in props.multiple ? messagesBodies : [0]"
          :key="index"
          class="px-2 py-2 cursor-pointer hover:bg-primaryLight transition rounded m-1 flex items-center justify-between group relative"
          :class="{
            'bg-primaryLight font-medium border-l-2 border-accent':
              selectedArgIndex === index,
            'text-secondaryLight': selectedArgIndex !== index,
          }"
          @click="selectArg(index)"
        >
          <span>Arg {{ index + 1 }}</span>
          <div class="flex items-center">
            <HoppButtonSecondary
              v-if="props.multiple && messagesBodies.length > 1"
              v-tippy="{ theme: 'tooltip' }"
              class="!p-0.5 absolute top-0 right-0 opacity-0 group-hover:opacity-100 scale-75"
              :title="t('action.remove')"
              :icon="IconX"
              @click.stop="removeArg(index)"
            />
          </div>
        </div>
      </div>
      <div class="h-full w-px bg-dividerLight"></div>
      <div class="h-full relative overflow-auto flex flex-col flex-1">
        <div ref="wsCommunicationBody" class="absolute inset-0"></div>
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { Component, computed, reactive, ref, watch } from "vue"
import IconSend from "~icons/lucide/send"
import IconHelpCircle from "~icons/lucide/help-circle"
import IconWrapText from "~icons/lucide/wrap-text"
import IconTrash2 from "~icons/lucide/trash-2"
import IconWand2 from "~icons/lucide/wand-2"
import IconCheck from "~icons/lucide/check"
import IconInfo from "~icons/lucide/info"
import IconDone from "~icons/lucide/check"
import IconFilePlus from "~icons/lucide/file-plus"
import IconPlus from "~icons/lucide/plus"
import IconX from "~icons/lucide/x"
import { pipe } from "fp-ts/function"
import * as TO from "fp-ts/TaskOption"
import * as O from "fp-ts/Option"
import { refAutoReset } from "@vueuse/core"
import { useCodemirror } from "@composables/codemirror"
import jsonLinter from "@helpers/editor/linting/json"
import { readFileAsText } from "@functional/files"
import { useI18n } from "@composables/i18n"
import { useToast } from "@composables/toast"
import { isJSONContentType } from "@helpers/utils/contenttypes"
import { defineActionHandler } from "~/helpers/actions"
import { getPlatformSpecialKey as getSpecialKey } from "~/helpers/platformutils"

const props = defineProps({
  showEventField: {
    type: Boolean,
    default: false,
  },
  eventFieldStyles: {
    type: String,
    default: "",
  },
  stickyHeaderStyles: {
    type: String,
    default: "",
  },
  isConnected: {
    type: Boolean,
    default: false,
  },
  multiple: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits<{
  (
    e: "send-message",
    body: {
      eventName: string
      message: string
    }
  ): void
  (
    e: "send-messages",
    body: {
      eventName: string
      messages: string[]
    }
  ): void
}>()

const t = useI18n()
const toast = useToast()

// Template refs
const tippyActions = ref<any | null>(null)
const linewrapEnabled = ref(true)
const wsCommunicationBody = ref<HTMLElement>()
const payload = ref<HTMLInputElement>()

const prettifyIcon = refAutoReset<Component>(IconWand2, 1000)
const clearInputOnSend = ref(false)

const messagesBodies = ref<string[]>([""])
const selectedArgIndex = ref(0)

const knownContentTypes = {
  JSON: "application/ld+json",
  Raw: "text/plain",
} as const

const validContentTypes = Object.keys(knownContentTypes) as ["JSON", "Raw"]

const contentType = ref<keyof typeof knownContentTypes>("JSON")
const eventName = ref("")
const communicationBody = ref("")

const rawInputEditorLang = computed(() => knownContentTypes[contentType.value])
const langLinter = computed(() =>
  isJSONContentType(contentType.value) ? jsonLinter : null
)

useCodemirror(
  wsCommunicationBody,
  communicationBody,
  reactive({
    extendedEditorConfig: {
      lineWrapping: linewrapEnabled,
      mode: rawInputEditorLang,
      placeholder: t("websocket.message").toString(),
    },
    linter: langLinter,
    completer: null,
    environmentHighlights: true,
  })
)

// Watch for changes in communicationBody and update the current message body
watch(communicationBody, (newValue) => {
  if (selectedArgIndex.value < messagesBodies.value.length) {
    messagesBodies.value[selectedArgIndex.value] = newValue
  }
})

// Function to select an argument
const selectArg = (index: number) => {
  selectedArgIndex.value = index
  communicationBody.value = messagesBodies.value[index] || ""
}

const addNewArg = () => {
  messagesBodies.value.push("")
  selectArg(messagesBodies.value.length - 1)
}

const clearContent = () => {
  if (clearInputOnSend.value) {
    if (props.multiple) {
      messagesBodies.value = [""]
      selectedArgIndex.value = 0
    }
    communicationBody.value = ""
    eventName.value = ""
  }
}

const sendMessage = () => {
  if (!communicationBody.value) return

  if (props.multiple) {
    sendMessages()
  } else {
    emit("send-message", {
      eventName: eventName.value,
      message: communicationBody.value,
    })
    clearContent()
  }
}

const sendMessages = () => {
  const nonEmptyMessages = messagesBodies.value.filter(
    (msg) => msg.trim() !== ""
  )

  if (nonEmptyMessages.length === 0) return

  emit("send-messages", {
    eventName: eventName.value,
    messages: nonEmptyMessages,
  })

  clearContent()
}

const uploadPayload = async (e: Event) => {
  const result = await pipe(
    (e.target as HTMLInputElement).files?.[0],
    TO.fromNullable,
    TO.chain(readFileAsText)
  )()

  if (O.isSome(result)) {
    communicationBody.value = result.value
    toast.success(`${t("state.file_imported")}`)
  } else {
    toast.error(`${t("action.choose_file")}`)
  }
}
const prettifyRequestBody = () => {
  try {
    const jsonObj = JSON.parse(communicationBody.value)
    communicationBody.value = JSON.stringify(jsonObj, null, 2)
    prettifyIcon.value = IconCheck
  } catch (e) {
    console.error(e)
    prettifyIcon.value = IconInfo
    toast.error(`${t("error.json_prettify_invalid_body")}`)
  }
}

const removeArg = (index: number) => {
  messagesBodies.value.splice(index, 1)

  if (selectedArgIndex.value === index) {
    selectedArgIndex.value = Math.max(0, index - 1)
    communicationBody.value = messagesBodies.value[selectedArgIndex.value] || ""
  } else if (selectedArgIndex.value > index) {
    selectedArgIndex.value--
  }
}

defineActionHandler("request.send-cancel", sendMessage)
</script>
