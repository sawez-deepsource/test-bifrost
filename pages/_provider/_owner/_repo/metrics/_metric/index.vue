<template>
  <div>
    <template v-if="!$fetchState.pending && isMissingAggregate">
      <lazy-empty-trend
        :namespaces-trend="mockAggregateData"
        :metric-meta="trendMetricData"
        :date-range="dateRange"
        @update-filter="updateFilter"
      />
    </template>
    <template v-if="metric && metric.namespacesTrends?.length">
      <trend-section
        v-for="namespacesTrend in orderedMetrics"
        :key="namespacesTrend?.key"
        :namespaces-trend="namespacesTrend"
        :metric-meta="trendMetricData"
        :date-range="dateRange"
        :clip="true"
        :can-modify-threshold="canModifyThreshold"
        :are-settings-overridden="areSettingsOverridden"
        :repo-group-name="repository.repositoryGroup?.name ?? ''"
        :data-loading="metricDataLoading"
        @addThreshold="openUpdateThresholdModal"
        @openUpdateThresholdModal="openUpdateThresholdModal"
        @deleteThreshold="deleteThreshold"
        @update-filter="updateFilter"
      />
      <portal to="modal">
        <edit-threshold-modal
          v-if="showUpdateThresholdModal"
          v-bind="editModalProps"
          @editThreshold="editThreshold"
          @close="showUpdateThresholdModal = false"
          @refetch="refetch"
        />
      </portal>
    </template>
    <template v-else-if="$fetchState.pending">
      <div class="h-23 animate-pulse bg-ink-300"></div>
      <div v-for="i in 2" :key="i" class="space-y-6 border-b border-slate-400 p-6">
        <div class="h-8 w-32 animate-pulse bg-ink-300"></div>
        <div class="space-y-4 rounded-md">
          <div class="flex gap-x-23">
            <div v-for="j in 2" :key="`s${j}`" class="h-13 w-32 animate-pulse bg-ink-300"></div>
          </div>
          <div class="h-72 animate-pulse bg-ink-300"></div>
        </div>
      </div>
    </template>
    <div v-else class="p-4">
      <lazy-empty-state
        v-if="isMetricDisabled"
        use-v2
        :show-border="true"
        title="Metric capture is disabled"
      >
        <template #subtitle>
          Capturing this metric has been turned off for this repository. To change this, please go
          to
          <nuxt-link to="../settings/reporting" class="text-juniper hover:underline focus:underline"
            >Settings <span class="text-vanilla-400"> › </span> Reporting</nuxt-link
          >.
        </template>
        <template #action>
          <nuxt-link-button to="../settings/reporting" class="space-x-2">
            <z-icon icon="wrench" color="ink-400" class="flex-shrink-0" />
            <span class="text-sm font-medium">Open repository settings</span>
          </nuxt-link-button>
        </template>
      </lazy-empty-state>
      <lazy-empty-state
        v-else-if="checkTestCoverageSetupStatus && !repository.hasTestCoverage"
        use-v2
        :webp-image-path="
          require('~/assets/images/ui-states/metrics/set-up-code-coverage-136px.webp')
        "
        :png-image-path="
          require('~/assets/images/ui-states/metrics/set-up-code-coverage-136px.png')
        "
        :show-border="true"
        title="Set up code coverage"
        subtitle="Enable the Test Coverage analyzer on this repository to start collecting code coverage metrics."
      >
        <template #action>
          <z-button
            icon="scroll"
            size="small"
            to="https://docs.deepsource.com/docs/analyzers-test-coverage"
            target="_blank"
            rel="noopener noreferrer"
            label="Read the docs"
          />
        </template>
      </lazy-empty-state>
      <lazy-empty-state
        v-else
        :webp-image-path="require('~/assets/images/ui-states/metrics/no-data-found-136px.webp')"
        :png-image-path="require('~/assets/images/ui-states/metrics/no-data-found-136px.png')"
        :show-border="false"
        title="Not enough data"
      >
        <template #subtitle>
          We do not have enough data for the selected time range. Please select another time range
          or come back later.
        </template>
      </lazy-empty-state>
    </div>
  </div>
</template>
<script lang="ts">
// Internals
import { defineComponent, PropType } from 'vue'
import { mapActions } from 'vuex'

// Store imports
import { RepositoryDetailActions } from '~/store/repository/detail'
import { GraphqlMutationResponse } from '~/types/apollo-graphql-types'
import { DEFAULT_BR_COVERAGE_METRICS, MetricType } from '~/types/metric'
import {
  Maybe,
  Metric,
  MetricNamespaceTrend,
  MetricSetting,
  MetricTypeChoices,
  Repository
} from '~/types/types'
import { dateRangeOptions, DurationTypeT, getUnitFromDate } from '~/utils/date'

import { formatRepoName } from '~/utils/string'

/**
 * Metric detail page for a Repository metric.
 */
export default defineComponent({
  name: 'MetricPage',
  layout: 'repository',
  props: {
    repoMetricSettings: {
      type: Array as PropType<MetricSetting[]>,
      required: true
    }
  },
  /**
   * Head hook that sets meta information for a page.
   *
   * @returns {Record<string,string>}
   */
  head(): Record<string, string> {
    const { owner, repo } = this.$route.params
    return {
      title: `${this.metric?.name ?? ''} Metric • ${owner}/${formatRepoName(repo)}`
    }
  },
  data() {
    return {
      dateRange: '7d' as keyof typeof dateRangeOptions,
      showUpdateThresholdModal: false,
      repository: {} as Repository,
      editModalProps: {},
      mockAggregateData: { key: '' } as MetricNamespaceTrend,
      metricDataLoading: false
    }
  },
  /**
   * Fetch hook for the metric detail page.
   *
   * @returns {Promise<void>}
   */
  async fetch(): Promise<void> {
    await this.fetchMetric(true)
  },
  computed: {
    lastDays() {
      return getUnitFromDate(
        dateRangeOptions[this.dateRange].count,
        dateRangeOptions[this.dateRange].durationType,
        DurationTypeT.days
      )
    },
    isMissingAggregate(): boolean {
      const isMissing = !this.metric?.namespacesTrends?.some(
        (namespaceTrend) => namespaceTrend?.key === MetricType.aggregate
      )
      if (isMissing) {
        this.setMockAggregateData()
      }
      return isMissing
    },
    orderedMetrics() {
      const metric = this.metric
      return (
        metric?.namespacesTrends?.sort(
          (prevTrend, nextTrend) =>
            prevTrend?.key?.localeCompare(nextTrend?.key ?? '', 'en', {
              sensitivity: 'base',
              ignorePunctuation: true
            }) ?? 0
        ) ?? []
      )
    },
    metric(): Maybe<Metric> | undefined {
      const metric = this.repository?.metricsCaptured?.find(
        (metric) => metric?.shortcode === this.$route.params.metric
      )
      return metric
    },
    trendMetricData():
      | Pick<
          Metric,
          | 'shortcode'
          | 'name'
          | 'description'
          | 'supportsAggregateThreshold'
          | 'unit'
          | 'newCodeMetricShortcode'
        >
      | Record<string, never> {
      if (this.metric) {
        const {
          shortcode,
          name,
          description,
          supportsAggregateThreshold,
          unit,
          newCodeMetricShortcode
        } = this.metric as Metric

        return {
          shortcode,
          name,
          description,
          supportsAggregateThreshold,
          unit,
          newCodeMetricShortcode
        }
      }
      return {}
    },

    canModifyThreshold(): boolean {
      return (
        Boolean(this.repository.userPermissionMeta?.can_modify_metric_thresholds) &&
        !this.areSettingsOverridden
      )
    },

    areSettingsOverridden(): boolean {
      return this.repository?.repositoryGroup?.overrideRepoSetting ?? false
    },

    checkTestCoverageSetupStatus(): boolean {
      return DEFAULT_BR_COVERAGE_METRICS.includes(this.$route.params.metric)
    },

    isMetricDisabled(): boolean {
      return Boolean(
        this.repoMetricSettings?.find(
          (metricSetting) => metricSetting.shortcode === this.$route.params.metric
        )?.isIgnoredToDisplay
      )
    }
  },
  methods: {
    ...mapActions('repository/detail', {
      fetchMetricData: RepositoryDetailActions.FETCH_METRIC,
      setMetricThreshold: RepositoryDetailActions.SET_METRIC_THRESHOLD
    }),
    setMockAggregateData(): void {
      this.mockAggregateData = {
        key: MetricType.aggregate
      }
    },
    /**
     * ? Sort metrics based on key, while ignoring the case they are in
     * Not a computed due to side effects.
     */

    /**
     * Fetch a metric depending on page route.
     *
     * @param {boolean} [refetch] - Optional parameter that performs a `network-only` fetch of metric data.
     *
     * @returns {Promise<void>}
     */
    async fetchMetric(refetch?: boolean): Promise<void> {
      this.metricDataLoading = true
      const { provider, owner, repo, metric } = this.$route.params
      try {
        this.repository = await this.fetchMetricData({
          provider,
          owner,
          name: repo,
          shortcode: metric,
          metricType: MetricTypeChoices.DefaultBranchOnly,
          lastDays: this.lastDays,
          refetch
        })
      } catch (e) {
        this.$router.replace(`/${provider}/${owner}/${repo}/metrics`)
      } finally {
        this.metricDataLoading = false
      }
    },
    /**
     * Refetch hook for the metric detail page.
     *
     * @returns {Promise<void>}
     */
    async refetch(): Promise<void> {
      await this.fetchMetric(true)
    },
    /**
     * Opens the update threshold modal after setting the required paramters.
     *
     * @param {MetricNamespaceTrend} namespacesTrend - Object of the namespace whose threshold needs to be updated
     *
     * @returns {void}
     */
    openUpdateThresholdModal(
      shortcode: string,
      threshold: MetricNamespaceTrend['threshold'] | MetricNamespaceTrend['newCodeThreshold'],
      key: MetricNamespaceTrend['key'],
      thresholdName: string
    ): void {
      this.editModalProps = {
        thresholdValue: threshold,
        metricName: this.metric?.name,
        analyzerKey: key,
        metricShortcode: shortcode,
        unit: this.metric?.unit,
        thresholdName
      }
      this.showUpdateThresholdModal = true
    },

    /**
     * Updates the threshold of a namespace.
     *
     * @param {number} newThresholdValue - Value to update the threshold to
     * @param {string} analyzerKey - `key` of the namespace whose threshold is being updated
     * @param {Function} close - Function to close the modal.
     *
     * @returns {Promise<void>}
     */
    async editThreshold(
      newThresholdValue: number | null,
      analyzerKey: string,
      metricShortcode: string,
      close?: () => void
    ): Promise<void> {
      if (newThresholdValue !== undefined) {
        try {
          const response = (await this.setMetricThreshold({
            metricShortcode,
            repositoryId: this.repository.id,
            thresholdValue: newThresholdValue,
            key: analyzerKey
          })) as GraphqlMutationResponse

          if (response.data.updateRepoMetricThreshold?.ok) {
            this.$toast.success('Successfully updated threshold.')
            close?.()
          }
        } catch (e) {
          this.$logErrorAndToast(e as Error, 'An error occured while updating repository metrics.')
        } finally {
          this.refetch()
        }
      } else {
        this.$toast.danger('An error occured while updating repository metrics.')
      }
    },

    /**
     * Deletes the threshold of a metric.
     *
     * @param {string} shortcode - Shortcode of the metric
     * @param {string} key - `key` of the metric
     *
     * @returns {Promise<void>}
     */
    async deleteThreshold(shortcode: string, key: string): Promise<void> {
      await this.editThreshold(null, key, shortcode)
    },

    /**
     * Refetch the metric when `$route.params.metric` changes.
     *
     * @returns {Promise<void>}
     */
    async updateFilter(newValue: string): Promise<void> {
      const { provider, owner, repo } = this.$route.params

      this.dateRange = newValue
      this.$localStore.set(`${provider}-${owner}-${repo}`, 'metrics-last-duration-filter', newValue)
      await this.refetch()
    }
  },
  watch: {
    '$route.params.metric': {
      handler() {
        this.$fetch()
      }
    }
  }
})
</script>
