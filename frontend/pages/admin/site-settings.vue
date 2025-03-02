<template>
  <v-container fluid class="narrow-container">
    <BasePageTitle divider>
      <template #header>
        <v-img max-height="200" max-width="150" :src="require('~/static/svgs/admin-site-settings.svg')"></v-img>
      </template>
      <template #title> {{ $t("settings.site-settings") }} </template>
    </BasePageTitle>

    <BaseDialog v-model="bugReportDialog" :title="$t('settings.bug-report')" :width="800" :icon="$globals.icons.github">
      <v-card-text>
        <div class="pb-4">
          {{ $t('settings.bug-report-information') }}
        </div>
        <v-textarea v-model="bugReportText" outlined rows="18" readonly> </v-textarea>
        <div class="d-flex justify-end" style="gap: 5px">
          <BaseButton color="gray" secondary target="_blank" href="https://github.com/hay-kot/mealie/issues/new/choose">
            <template #icon> {{ $globals.icons.github }}</template>
            {{ $t('settings.tracker') }}
          </BaseButton>
          <AppButtonCopy :copy-text="bugReportText" color="info" :icon="false" />
        </div>
      </v-card-text>
    </BaseDialog>

    <div class="d-flex justify-end">
      <BaseButton
        color="info"
        @click="
          dockerValidate();
          bugReportDialog = true;
        "
      >
        <template #icon> {{ $globals.icons.github }}</template>
        {{ $t('settings.bug-report') }}
      </BaseButton>
    </div>

    <section>
      <BaseCardSectionTitle class="pb-0" :icon="$globals.icons.cog" :title="$t('settings.configuration')"> </BaseCardSectionTitle>
      <v-card class="mb-4">
        <template v-for="(check, idx) in simpleChecks">
          <v-list-item :key="`list-item-${idx}`">
            <v-list-item-icon>
              <v-icon :color="check.color">
                {{ check.icon }}
              </v-icon>
            </v-list-item-icon>
            <v-list-item-content>
              <v-list-item-title>
                {{ check.text }}
              </v-list-item-title>
              <v-list-item-subtitle class="wrap-word">
                {{ check.status ? check.successText : check.errorText }}
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
          <v-divider :key="`divider-${idx}`"></v-divider>
        </template>
      </v-card>
    </section>

    <section>
      <BaseCardSectionTitle class="pt-2" :icon="$globals.icons.docker" :title="$t('settings.docker-volume')" />
      <v-alert
        border="left"
        colored-border
        :type="docker.state === DockerVolumeState.Error ? 'error' : 'info'"
        :icon="$globals.icons.docker"
        elevation="2"
        :loading="docker.loading"
      >
        <div class="d-flex align-center font-weight-medium">
          {{ $t('settings.docker-volume') }}
          <HelpIcon small class="my-n3">
            {{ $t('settings.docker-volume-help') }}
          </HelpIcon>
        </div>
        <div>
          <template v-if="docker.state === DockerVolumeState.Error"> {{ $t('settings.volumes-are-misconfigured') }}. </template>
          <template v-else-if="docker.state === DockerVolumeState.Success">
            {{ $t('settings.volumes-are-configured-correctly') }}
          </template>
          <template v-else-if="docker.state === DockerVolumeState.Unknown">
            {{ $t('settings.status-unknown-try-running-a-validation') }}
          </template>
        </div>
        <div class="mt-4">
          <BaseButton color="info" :loading="docker.loading" @click="dockerValidate">
            <template #icon> {{ $globals.icons.checkboxMarkedCircle }} </template>
            {{ $t('settings.validate') }}
          </BaseButton>
        </div>
      </v-alert>
    </section>

    <section>
      <BaseCardSectionTitle class="pt-2" :icon="$globals.icons.email" :title="$tc('user.email')" />
      <v-alert border="left" colored-border :type="appConfig.emailReady ? 'success' : 'error'" elevation="2">
        <div class="font-weight-medium">{{ $t('settings.email-configuration-status') }}</div>
        <div>
          {{ appConfig.emailReady ? $t('settings.ready') : $t('settings.not-ready')  }}
        </div>
        <div>
          <v-text-field v-model="address" class="mr-4" :label="$t('user.email')" :rules="[validators.email]">
          </v-text-field>
          <BaseButton
            color="info"
            :disabled="!appConfig.emailReady || !validEmail"
            :loading="loading"
            @click="testEmail"
          >
            <template #icon> {{ $globals.icons.email }} </template>
            {{ $t("general.test") }}
          </BaseButton>
          <template v-if="tested">
            <v-divider class="my-x mt-6"></v-divider>
            <v-card-text class="px-0">
              <h4>Email Test Results</h4>
              <span class="pl-4">
                {{ success ? $t('settings.succeeded') : $t('settings.failed') }}
              </span>
            </v-card-text>
          </template>
        </div>
      </v-alert>
    </section>

    <!-- General App Info -->
    <section class="mt-4">
      <BaseCardSectionTitle class="pb-0" :icon="$globals.icons.cog" :title="$t('settings.general-about')"> </BaseCardSectionTitle>
      <v-card class="mb-4">
        <template v-for="(property, idx) in appInfo">
          <v-list-item :key="property.name">
            <v-list-item-icon>
              <v-icon> {{ property.icon || $globals.icons.user }} </v-icon>
            </v-list-item-icon>
            <v-list-item-content>
              <v-list-item-title>
                <div>{{ property.name }}</div>
              </v-list-item-title>
              <template v-if="property.slot === 'recipe-scraper'">
                <v-list-item-subtitle>
                  <a
                    target="_blank"
                    :href="`https://github.com/hhursev/recipe-scrapers/releases/tag/${property.value}`"
                  >
                    {{ property.value }}
                  </a>
                </v-list-item-subtitle>
              </template>
              <template v-else-if="property.slot === 'build'">
                <v-list-item-subtitle>
                  <a target="_blank" :href="`https://github.com/hay-kot/mealie/commit/${property.value}`">
                    {{ property.value }}
                  </a>
                </v-list-item-subtitle>
              </template>
              <template v-else>
                <v-list-item-subtitle>
                  {{ property.value }}
                </v-list-item-subtitle>
              </template>
            </v-list-item-content>
          </v-list-item>
          <v-divider v-if="appInfo && idx !== appInfo.length - 1" :key="`divider-${property.name}`"></v-divider>
        </template>
      </v-card>
    </section>
  </v-container>
</template>

<script lang="ts">
import {
  computed,
  onMounted,
  reactive,
  toRefs,
  ref,
  defineComponent,
  useAsync,
  useContext,
} from "@nuxtjs/composition-api";
import { TranslateResult } from "vue-i18n";
import { useAdminApi, useUserApi } from "~/composables/api";
import { validators } from "~/composables/use-validators";
import { useAsyncKey } from "~/composables/use-utils";
import { CheckAppConfig } from "~/lib/api/types/admin";

enum DockerVolumeState {
  Unknown = "unknown",
  Success = "success",
  Error = "error",
}

interface SimpleCheck {
  id: string;
  text: TranslateResult;
  status: boolean | undefined;
  successText: TranslateResult;
  errorText: TranslateResult;
  color: string;
  icon: string;
}

interface CheckApp extends CheckAppConfig {
  isSiteSecure?: boolean;
}

export default defineComponent({
  layout: "admin",
  setup() {
    // ==========================================================
    // Docker Volume Validation
    const docker = reactive({
      loading: false,
      state: DockerVolumeState.Unknown,
    });

    async function dockerValidate() {
      docker.loading = true;

      // Do API Check
      const { data } = await adminApi.about.checkDocker();
      if (data == null) {
        docker.state = DockerVolumeState.Error;
        return;
      }

      // Get File Contents
      const { data: fileContents } = await adminApi.about.getDockerValidateFileContents();

      if (data.text === fileContents) {
        docker.state = DockerVolumeState.Success;
      } else {
        docker.state = DockerVolumeState.Error;
      }

      docker.loading = false;
    }

    const state = reactive({
      loading: false,
      address: "",
      success: false,
      error: "",
      tested: false,
    });

    const appConfig = ref<CheckApp>({
      emailReady: true,
      baseUrlSet: true,
      isSiteSecure: true,
      isUpToDate: false,
      ldapReady: false,
    });

    function isLocalHostOrHttps() {
      return window.location.hostname === "localhost" || window.location.protocol === "https:";
    }

    const api = useUserApi();
    const adminApi = useAdminApi();

    onMounted(async () => {
      const { data } = await adminApi.about.checkApp();

      if (data) {
        appConfig.value = { ...data, isSiteSecure: false };
      }

      appConfig.value.isSiteSecure = isLocalHostOrHttps();
    });

    const simpleChecks = computed<SimpleCheck[]>(() => {
      const goodIcon = $globals.icons.checkboxMarkedCircle;
      const badIcon = $globals.icons.alert;
      const warningIcon = $globals.icons.alertCircle;

      const goodColor = "success";
      const badColor = "error";
      const warningColor = "warning";


      const data: SimpleCheck[] = [
        {
          id: "application-version",
          text: i18n.t("settings.application-version"),
          status: appConfig.value.isUpToDate,
          errorText: i18n.t("settings.application-version-error-text", [rawAppInfo.value.version, rawAppInfo.value.versionLatest]),
          successText: i18n.t("settings.mealie-is-up-to-date"),
          color: appConfig.value.isUpToDate ? goodColor : warningColor,
          icon: appConfig.value.isUpToDate ? goodIcon : warningIcon,
        },
        {
          id: "secure-site",
          text: i18n.t("settings.secure-site"),
          status: appConfig.value.isSiteSecure,
          errorText: i18n.t("settings.secure-site-error-text"),
          successText: i18n.t("settings.secure-site-success-text"),
          color: appConfig.value.isSiteSecure ? goodColor : badColor,
          icon: appConfig.value.isSiteSecure ? goodIcon : badIcon,
        },
        {
          id: "server-side-base-url",
          text: i18n.t("settings.server-side-base-url"),
          status: appConfig.value.baseUrlSet,
          errorText:
            i18n.t("settings.server-side-base-url-error-text"),
          successText: i18n.t("settings.server-side-base-url-success-text"),
          color: appConfig.value.baseUrlSet ? goodColor : badColor,
          icon: appConfig.value.baseUrlSet ? goodIcon : badIcon,
        },
        {
          id: "ldap-ready",
          text: i18n.t("settings.ldap-ready"),
          status: appConfig.value.ldapReady,
          errorText:
            i18n.t("settings.ldap-ready-error-text"),
          successText: i18n.t("settings.ldap-ready-success-text"),
          color: appConfig.value.ldapReady ? goodColor : warningColor,
          icon: appConfig.value.ldapReady ? goodIcon : warningIcon,
        },
      ];

      return data;
    });

    async function testEmail() {
      state.loading = true;
      state.tested = false;
      const { data } = await api.email.test({ email: state.address });

      if (data) {
        if (data.success) {
          state.success = true;
        } else {
          state.error = data.error ?? "";
          state.success = false;
        }
      }
      state.loading = false;
      state.tested = true;
    }

    const validEmail = computed(() => {
      if (state.address === "") {
        return false;
      }
      const valid = validators.email(state.address);

      // Explicit bool check because validators.email sometimes returns a string
      if (valid === true) {
        return true;
      }
      return false;
    });

    // ============================================================
    // General About Info

    const { $globals, i18n } = useContext();

    const rawAppInfo = ref({
      version: "null",
      versionLatest: "null",
    });

    function getAppInfo() {
      const statistics = useAsync(async () => {
        const { data } = await adminApi.about.about();

        if (data) {
          rawAppInfo.value.version = data.version;
          rawAppInfo.value.versionLatest = data.versionLatest;

          const prettyInfo = [
            {
              name: i18n.t("about.version"),
              icon: $globals.icons.information,
              value: data.version,
            },
            {
              slot: "build",
              name: i18n.t("settings.build"),
              icon: $globals.icons.information,
              value: data.buildId,
            },
            {
              name: i18n.t("about.application-mode"),
              icon: $globals.icons.devTo,
              value: data.production ? i18n.t("about.production") : i18n.t("about.development"),
            },
            {
              name: i18n.t("about.demo-status"),
              icon: $globals.icons.testTube,
              value: data.demoStatus ? i18n.t("about.demo") : i18n.t("about.not-demo"),
            },
            {
              name: i18n.t("about.api-port"),
              icon: $globals.icons.api,
              value: data.apiPort,
            },
            {
              name: i18n.t("about.api-docs"),
              icon: $globals.icons.file,
              value: data.apiDocs ? i18n.t("general.enabled") : i18n.t("general.disabled"),
            },
            {
              name: i18n.t("about.database-type"),
              icon: $globals.icons.database,
              value: data.dbType,
            },
            {
              name: i18n.t("about.database-url"),
              icon: $globals.icons.database,
              value: data.dbUrl,
            },
            {
              name: i18n.t("about.default-group"),
              icon: $globals.icons.group,
              value: data.defaultGroup,
            },
            {
              slot: "recipe-scraper",
              name: i18n.t("settings.recipe-scraper-version"),
              icon: $globals.icons.primary,
              value: data.recipeScraperVersion,
            },
          ];

          return prettyInfo;
        }

        return data;
      }, useAsyncKey());

      return statistics;
    }

    const appInfo = getAppInfo();

    const bugReportDialog = ref(false);

    const bugReportText = computed(() => {
      const ignore = {
        [i18n.tc("about.database-url")]: true,
        [i18n.tc("about.default-group")]: true,
      };
      let text = "**Details**\n";

      appInfo.value?.forEach((item) => {
        if (ignore[item.name as string]) {
          return;
        }
        text += `${item.name as string}: ${item.value as string}\n`;
      });

      const ignoreChecks: { [key: string]: boolean } = {
        "application-version": true,
      };

      text += "\n**Checks**\n";

      simpleChecks.value.forEach((item) => {
        if (ignoreChecks[item.id]) {
          return;
        }
        const status = item.status ? "Yes" : "No";
        text += `${item.text.toString()}: ${status}\n`;
      });

      text += `Email Configured: ${appConfig.value.emailReady ? "Yes" : "No"}\n`;
      text += `Docker Volumes: ${docker.state}`;

      return text;
    });

    return {
      bugReportDialog,
      bugReportText,
      DockerVolumeState,
      docker,
      dockerValidate,
      simpleChecks,
      appConfig,
      validEmail,
      validators,
      ...toRefs(state),
      testEmail,
      appInfo,
    };
  },
  head() {
    return {
      title: this.$t("settings.site-settings") as string,
    };
  },
});
</script>

<style scoped>
.wrap-word {
  white-space: normal;
  word-wrap: break-word;
}
</style>
