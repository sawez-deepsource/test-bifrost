<template>
  <div class="max-w-2xl space-y-5 p-4">
    <page-title
      class="max-w-2xl"
      title="Code Coverage"
      description-width-class="max-w-2xl"
      description="Manage how you track the code coverage of this repository."
    />

    <main class="space-y-7">
      <div class="space-y-5">
        <z-alert type="info" class="border border-robin border-opacity-10 bg-opacity-5 pb-3 pt-2">
          <div class="space-y-2 text-xs font-medium text-robin-400">
            <p class="text-sm leading-8">How does it work?</p>
            <p class="font-normal leading-6 text-robin-150">
              Code Coverage is automatically enabled when we receive a coverage report for your
              repository. Send a coverage report to DeepSource directly using the CLI or integrate
              coverage reporting in your CI pipeline.
            </p>
            <p>
              <a
                href="https://docs.deepsource.com/docs/analyzers-test-coverage"
                target="_blank"
                rel="noopener noreferrer"
                class="inline-flex items-center gap-x-0.5 hover:underline focus:underline"
              >
                Learn more
                <z-icon icon="arrow-up-right" color="robin-400" size="x-small" />
              </a>
            </p>
          </div>
        </z-alert>

        <coverage-enabled-card
          :has-test-coverage="hasTestCoverage"
          :dsn="dsn"
          :loading="$fetchState.pending"
        />
      </div>

      <div class="space-y-3">
        <h3 class="text-sm leading-5">Preferences</h3>

        <group-managed-alert v-if="areSettingsOverridden" :repository-group="repositoryGroup">
          <template #description>
            <p class="text-xs text-robin-150">
              Settings in this section are managed by the repository group that this repository
              belongs to.
              <nuxt-link
                v-if="canManageGroupSettings"
                :to="{
                  name: 'orgs-provider-owner-groups-groupSlug-settings-repo-settings',
                  params: {
                    ...$route.params,
                    groupSlug: repositoryGroup?.slug
                  }
                }"
                class="text-vanilla-400 hover:underline focus:underline"
              >
                View group settings ->
              </nuxt-link>
            </p>
          </template>
        </group-managed-alert>

        <div
          v-if="$fetchState.pending"
          class="h-infer-toggle animate-pulse rounded-md bg-ink-300"
        ></div>

        <div v-else class="rounded-md border border-ink-200 bg-ink-300 p-4">
          <toggle-input
            :value="inferDefaultBranchCoverage"
            :remove-y-padding="true"
            :disabled="areSettingsOverridden"
            input-width="xx-small"
            label="Enable report inference on the default branch"
            description="Ask DeepSource to re-use coverage reports from pull-requests after a merge. Turn this on if you don't run tests on your default branch commits."
            input-id="enable-infer-default-branch-coverage"
            @input="toggleInferDefaultBranchCoverage"
          />
        </div>
      </div>
    </main>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

import updateInferDefaultBranchCoverage from '~/apollo/mutations/repository/settings/updateInferDefaultBranchCoverage.gql'
import repositoryId from '~/apollo/queries/repository/id.gql'
import codeCoverage from '~/apollo/queries/repository/settings/codeCoverage.gql'

import activeUserMixin from '~/mixins/activeUserMixin'
import { RepoPerms, TeamPerms } from '~/types/permTypes'
import {
  CodeCoverageQueryVariables,
  Repository,
  RepositoryGroup,
  RepositoryIdQueryVariables,
  RepositorySetting,
  UpdateRepositorySettingsInput
} from '~/types/types'

export default defineComponent({
  name: 'SettingsCodeCoverage',
  layout: 'repository',
  middleware: ['perm'],
  meta: {
    auth: {
      strict: true,
      repoPerms: [RepoPerms.TOGGLE_REPORT_INFERENCE]
    }
  },
  mixins: [activeUserMixin],
  data() {
    return {
      inferDefaultBranchCoverage: false,
      hasTestCoverage: false,
      repoId: '',
      dsn: '',
      repositoryGroup: null as RepositoryGroup | null
    }
  },
  computed: {
    areSettingsOverridden(): boolean {
      return this.repositoryGroup?.overrideRepoSetting ?? false
    },
    canManageGroupSettings(): boolean {
      return this.$gateKeeper.team(TeamPerms.MANAGE_GROUPS, this.teamPerms.permission)
    }
  },
  /**
   * Fetch lifecycle hook for the page.
   */
  async fetch() {
    const { provider, owner, repo: name } = this.$route.params

    try {
      const args: CodeCoverageQueryVariables = {
        provider: this.$providerMetaMap[provider].value,
        owner,
        name
      }
      const res = await this.$fetchGraphqlData(codeCoverage, args)

      const repoDetails = res.data?.repository as Repository

      if (repoDetails) {
        this.hasTestCoverage = repoDetails?.hasTestCoverage ?? false

        this.inferDefaultBranchCoverage =
          repoDetails?.repositorySetting?.inferDefaultBranchCoverage ?? false

        this.repositoryGroup = repoDetails?.repositoryGroup ?? null

        this.repoId = repoDetails?.id ?? ''

        this.dsn = repoDetails?.dsn ?? ''
      } else {
        this.$toast.danger('Unable to fetch repository data. Please contact support.')
      }
    } catch (e) {
      this.$logErrorAndToast(e as Error, 'Unable to fetch repository data. Please contact support.')
    }
  },
  methods: {
    /**
     * Toggles the inferDefaultBranchCoverage value.
     * @param newVal - The new value for inferDefaultBranchCoverage.
     */
    async toggleInferDefaultBranchCoverage(newVal: boolean) {
      try {
        if (!this.repoId) {
          const { provider, owner, repo: name } = this.$route.params
          const args: RepositoryIdQueryVariables = {
            provider: this.$providerMetaMap[provider].value,
            owner,
            name
          }
          const res = await this.$fetchGraphqlData(repositoryId, args)
          this.repoId = res.data?.repository?.id
        }

        const args: UpdateRepositorySettingsInput = {
          id: this.repoId,
          inferDefaultBranchCoverage: newVal
        }
        const res = await this.$applyGraphqlMutation(updateInferDefaultBranchCoverage, {
          input: args
        })

        const updatedRepoSettings = res?.data?.updateRepositorySettings?.repository
          ?.repositorySetting as RepositorySetting

        if (
          updatedRepoSettings &&
          Object.prototype.hasOwnProperty.call(updatedRepoSettings, 'inferDefaultBranchCoverage')
        ) {
          this.inferDefaultBranchCoverage = updatedRepoSettings.inferDefaultBranchCoverage ?? newVal
        } else {
          this.$toast.danger('Unable to update code coverage preferences. Please contact support.')
        }
      } catch (e) {
        this.$logErrorAndToast(
          e as Error,
          'Unable to update code coverage preferences. Please contact support.'
        )
      }
    }
  }
})
</script>

<style scoped>
.h-infer-toggle {
  height: 95px;
}
</style>
