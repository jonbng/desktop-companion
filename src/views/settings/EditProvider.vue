<template>
  <section>
    <v-card-text>
      <!-- header -->
      <div v-if="config && api.providerManifests[config.domain]">
        <v-card-title>
          {{
            $t("settings.setup_provider", [
              api.providerManifests[config.domain].name,
            ])
          }}
        </v-card-title>
        <v-card-subtitle style="white-space: normal">
          {{
            api.providerManifests[config.domain].description
          }} </v-card-subtitle
        ><br />
        <v-card-subtitle
          v-if="api.providerManifests[config.domain].codeowners.length"
          v-html="
            markdownToHtml(
              getAuthorsMarkdown(
                api.providerManifests[config.domain].codeowners,
              ),
            )
          "
        />

        <v-card-subtitle
          v-if="api.providerManifests[config.domain].documentation"
        >
          <b>{{ $t("settings.need_help_setup_provider") }} </b>&nbsp;
          <a
            @click="
              openLinkInNewTab(
                api.providerManifests[config.domain].documentation!,
              )
            "
            >{{ $t("settings.check_docs") }}</a
          >
        </v-card-subtitle>
        <br />
        <v-divider />
        <br />
        <br />

        <!-- name field -->
        <v-text-field
          v-if="'name' in config"
          v-model="config.name"
          :placeholder="api.getProvider(config.instance_id)?.name"
          :label="$t('settings.provider_name')"
          variant="outlined"
          clearable
        />
        <!-- enable field -->
        <v-switch
          v-if="'enabled' in config"
          v-model="config.enabled"
          :label="$t('settings.enable_provider')"
          color="primary"
          :disabled="!api.providerManifests[config.domain]?.allow_disable"
        />
      </div>
      <v-divider />
      <edit-config
        v-if="config"
        :config-entries="Object.values(config.values)"
        :disabled="!config.enabled"
        @submit="onSubmit"
        @action="onAction"
      />
    </v-card-text>
    <v-overlay
      v-model="loading"
      scrim="true"
      persistent
      style="display: flex; align-items: center; justify-content: center"
    >
      <v-card v-if="showAuthLink" style="background-color: white">
        <v-card-title>Authenticating...</v-card-title>
        <v-card-subtitle
          >A new tab/popup should be opened where you can
          authenticate</v-card-subtitle
        >
        <v-card-actions>
          <a id="auth" href="" target="_blank"
            ><v-btn>Click here if the popup did not open</v-btn></a
          >
        </v-card-actions>
      </v-card>
      <v-progress-circular v-else indeterminate size="64" color="primary" />
    </v-overlay>
  </section>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import { useRouter } from "vue-router";
import { api } from "@/plugins/api";
import {
  ProviderConfig,
  ConfigValueType,
  EventMessage,
  EventType,
} from "@/plugins/api/interfaces";
import EditConfig from "./EditConfig.vue";
import { nanoid } from "nanoid";
import { openLinkInNewTab, markdownToHtml } from "@/helpers/utils";
import { useI18n } from "vue-i18n";
import { openUrl } from "@tauri-apps/plugin-opener";

// global refs
const router = useRouter();
const config = ref<ProviderConfig>();
const sessionId = nanoid(11);
const loading = ref(false);
const showAuthLink = ref(false);

// props
const props = defineProps<{
  instanceId?: string;
}>();

onMounted(() => {
  //reload if/when item updates
  const unsub = api.subscribe(EventType.AUTH_SESSION, (evt: EventMessage) => {
    // handle AUTH_SESSION event (used for auth flows to open the auth url)
    // ignore any events that not match our session id.
    if (evt.object_id !== sessionId) return;
    const url = evt.data as string;
    openUrl(url);
  });
  onBeforeUnmount(unsub);
});

// watchers
watch(
  () => props.instanceId,
  async (val) => {
    if (val) {
      config.value = await api.getProviderConfig(val);
    }
  },
  { immediate: true },
);

// methods
const onSubmit = async function (values: Record<string, ConfigValueType>) {
  // save new provider config
  loading.value = true;
  values["enabled"] = config.value!.enabled;
  values["name"] = config.value!.name || null;
  api
    .saveProviderConfig(config.value!.domain, values, config.value!.instance_id)
    .then(() => {
      router.push({ name: "providersettings" });
    })
    .catch((err) => {
      // TODO: make this a bit more fancy someday
      alert(err);
    })
    .finally(() => {
      loading.value = false;
      showAuthLink.value = false;
    });
};

const onAction = async function (
  action: string,
  values: Record<string, ConfigValueType>,
) {
  loading.value = true;
  // append existing ConfigEntry values to allow
  // values be passed between flow steps
  for (const entry of Object.values(config.value!.values)) {
    if (entry.value !== undefined && values[entry.key] == undefined) {
      values[entry.key] = entry.value;
    }
  }
  // ensure the session id is passed along (for auth actions)
  values["session_id"] = sessionId;
  api
    .getProviderConfigEntries(
      config.value!.domain,
      config.value!.instance_id,
      action,
      values,
    )
    .then((entries) => {
      config.value!.values = {};
      for (const entry of entries) {
        config.value!.values[entry.key] = entry;
      }
    })
    .catch((err) => {
      // TODO: make this a bit more fancy someday
      alert(err);
    })
    .finally(() => {
      loading.value = false;
      showAuthLink.value = false;
    });
};

const getAuthorsMarkdown = function (authors: string[]) {
  const allAuthors: string[] = [];
  const { t } = useI18n();
  for (const author of authors) {
    if (author.includes("@")) {
      allAuthors.push(
        `[${author.replace("@", "")}](https://github.com/${author.replace("@", "")})`,
      );
    } else {
      allAuthors.push(author);
    }
  }
  return `**${t("settings.codeowners")}**: ` + allAuthors.join(" / ");
};
</script>
