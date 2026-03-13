<template>
  <div class="issues-page pb-16 lg:pb-0">
    <!-- Analyzer Tab -->
    <div
      class="analyzer-tab z-20 flex flex-col justify-between space-y-2 border-b border-slate-400 bg-ink-400 px-4 py-2 pr-4 lg:sticky lg:flex-row lg:space-y-0"
    >
      <div class="mx-auto flex w-full max-w-6xl justify-between px-4">
        <div class="flex flex-wrap gap-2">
          <issue-analyzer-selector
            :selected-analyzer="parsedParams.analyzer"
            @updateAnalyzer="updateAnalyzer"
          />
          <autofix-available
            v-if="showAutofixAvailableFilter"
            :autofix-filter-applied="parsedParams.autofixAvailable"
            @toggle-autofix-filter="(value) => addFilters({ autofixAvailable: value })"
          />

          <severity-filter
            :selected-severity-filter="parsedParams.severity"
            @update-severity-filter="(value) => addFilters({ severity: value })"
            @menu-toggle="showCategorySelectorMobile = !showCategorySelectorMobile"
          />
        </div>
        <div class="flex w-auto items-center justify-end gap-x-2">
          <issue-sort
            :selected-sort-filter="parsedParams.sort"
            @updateSortFilter="(value) => addFilters({ sort: value })"
            @reset="removeFilter('sort')"
          />
          <issue-search
            :search-candidate="parsedParams.q"
            placeholder="Search for an issue..."
            @updateSearch="(value) => addFilters({ q: value })"
          />
        </div>
      </div>
    </div>
    <!-- Main Content -->

    <div class="mx-auto grid w-full max-w-6xl grid-cols-1 lg:grid-cols-16-fr">
      <!-- Group by Filter Section -->
      <div>
        <category-selector-mobile
          v-show="showCategorySelectorMobile"
          :issue-categories="issueCategories"
          :occurrence-counts="issueOccurrenceDistributionCounts"
          :active-sidebar-item="activeSidebarItem"
          @updateCategory="updateCategory"
        />
        <issue-category-selector
          :issue-categories="issueCategories"
          :occurrence-counts="issueOccurrenceDistributionCounts"
          :active-sidebar-item="activeSidebarItem"
          class="category-sidebar sticky"
          @update-category="updateCategory"
        >
          <template #footer>
            <z-radio-group
              :model-value="issueDistributionType"
              class="grid h-8 w-full min-w-52 flex-shrink-0 grid-cols-2 font-medium text-vanilla-100 shadow-blur-lg"
              @change="handleDistributionTypeChange"
            >
              <z-radio-button
                :value="IssueOccurrenceDistributionType.PRODUCT"
                spacing="w-full h-full pt-1"
                class="text-center"
              >
                <z-icon icon="nested-list" class="inline" />
              </z-radio-button>
              <z-radio-button
                :value="IssueOccurrenceDistributionType.ISSUE_TYPE"
                spacing="w-full h-full pt-1"
                class="text-center"
              >
                <z-icon icon="flat-list" class="inline" />
              </z-radio-button>
            </z-radio-group>
          </template>
        </issue-category-selector>
      </div>
      <!-- List of issues -->
      <div v-if="issuesLoading" class="flex min-h-screen flex-grow flex-col gap-y-4 p-4">
        <div
          v-for="idx in 7"
          :key="idx"
          class="relative z-0 h-26 animate-pulse rounded-md bg-ink-300"
        ></div>
      </div>

      <div
        v-else
        class="flex flex-1 flex-grow flex-col gap-y-4 p-4 pb-10"
        :class="{
          'lg:h-64': totalIssueCount === 0,
          'max-h-auto': totalIssueCount > 0
        }"
      >
        <template v-if="totalIssueCount">
          <issue-list-item
            v-for="issue in issueList"
            v-bind="getRepositoryIssueProps(issue)"
            :key="issue.id"
            :show-autofix-button="allowAutofix && hasRepoReadAccess"
            :disable-autofix-button="!canCreateAutofix"
            :issue-link="$generateRoute(['issue', issue.shortcode || '', 'occurrences'])"
            @autofix="openAutofixModal"
          />
        </template>

        <div v-else class="mx-auto sm:mt-24">
          <lazy-empty-state-card
            v-if="issueCategoryDisabled"
            :webp-image-path="disabledStateImagePath.webp"
            :png-image-path="disabledStateImagePath.png"
            :title="`${issueCategoryName} issues are not being reported`"
            :subtitle="disabledStateSubtitle"
            class="px-4 py-8 sm:pl-8"
          >
            <template #action>
              <nuxt-link
                v-if="canModifyReportingSettings"
                :to="$generateRoute(['settings', 'quality-gates'])"
                class="auxillary-button py-1.5 text-xs"
              >
                <span>Open repository settings</span>
                <z-icon icon="arrow-right" color="current" size="x-small" />
              </nuxt-link>
              <div
                v-else
                class="flex items-center gap-x-2 rounded-md border-robin-500 border-opacity-10 bg-robin-500 bg-opacity-5 p-2 text-sm text-robin-300"
              >
                <z-icon icon="solid-alert-circle" color="robin-500" />

                {{
                  issueCategoryIsCoverage
                    ? 'Contact your admin to enable Code coverage.'
                    : `Contact your admin to enable ${issueCategoryName} issues.`
                }}
              </div>
            </template>
          </lazy-empty-state-card>

          <lazy-empty-state-card
            v-else-if="issueCategoryIsCoverage && !hasTestCoverage"
            :png-image-path="require('~/assets/images/ui-states/timeout/timeout-state-136px.gif')"
            :webp-image-path="require('~/assets/images/ui-states/timeout/timeout-state-136px.webp')"
            title="No Code Coverage issues yet"
            subtitle="Send a coverage report to DeepSource to start seeing coverage issues here."
            class="px-4 py-8 sm:pl-8"
          >
            <template #action>
              <a
                href="https://docs.deepsource.com/docs/analyzers-test-coverage"
                target="_blank"
                rel="noopener noreferrer"
                class="text-sm leading-6 text-vanilla-100 duration-300 ease-out hover:text-juniper-300 focus:text-juniper-300"
              >
                How to send coverage reports to DeepSource?
                <z-icon icon="arrow-up-right" color="current" class="inline" />
              </a>
            </template>
          </lazy-empty-state-card>

          <lazy-empty-state
            v-else-if="parsedParams.q"
            :png-image-path="require('~/assets/images/ui-states/directory/empty-search.gif')"
            :webp-image-path="require('~/assets/images/ui-states/directory/empty-search.webp')"
            :show-border="true"
            :title="`No results found for '${parsedParams.q}'`"
            :alt-text="`No results found for '${parsedParams.q}'`"
            subtitle="Please try changing the search query or clearing the filters."
          />

          <lazy-empty-state
            v-else
            :webp-image-path="
              require('~/assets/images/ui-states/issues/no-issues-found-static-140px.webp')
            "
            :png-image-path="
              require('~/assets/images/ui-states/issues/no-issues-found-static-140px.png')
            "
            title="No issues found"
            subtitle="That's a good thing, right?!"
            alt-text="No issues found"
          />
        </div>

        <z-pagination
          v-if="pageCount > 1"
          :key="$route.fullPath"
          :page="queryParams.page"
          :total-pages="pageCount"
          :total-visible="5"
          class="flex justify-center"
          @selected="(page) => addFilters({ page })"
        />
      </div>
    </div>

    <!-- Autofix Modal -->
    <template v-if="isAutofixOpen">
      <autofix-file-chooser
        v-bind="autofixIssue"
        :is-open="isAutofixOpen"
        @close="isAutofixOpen = false"
        @run-quota-exhausted="openUpgradeAccountModal"
      />
    </template>

    <upgrade-account-modal
      v-if="isUpgradeAccountModalOpen"
      :login="$route.params.owner"
      :provider="$route.params.provider"
      @close="isUpgradeAccountModalOpen = false"
    />
  </div>
</template>

<script lang="ts">
import {
  AutofixAvailable,
  IssueAnalyzerSelector,
  IssueCategorySelector,
  IssueSearch,
  IssueSort
} from '@/components/Issue'
import IssueListItem from '@/components/IssueListItem.vue'
import { AutofixFileChooser } from '@/components/RepoIssues'
import { defineComponent } from 'vue'

import RepoDetailMixin from '~/mixins/repoDetailMixin'
import RoleAccessMixin from '~/mixins/roleAccessMixin'
import RouteQueryMixin, { RouteQueryParamsT } from '~/mixins/routeQueryMixin'

import RepositoryIssuesListGQLQuery from '~/apollo/queries/repository/issue/list.gql'

import { RepoPerms } from '~/types/permTypes'
import { Maybe, RepositoryIssue } from '~/types/types'

import { resolveNodes } from '~/utils/array'
import { formatRepoName } from '~/utils/string'

import { GraphqlQueryResponse } from '~/types/apollo-graphql-types'
import {
  IssueCategoryTypeFilterMap,
  IssueFilterChoice,
  IssueOccurrenceDistributionType,
  IssueSeverityFilter,
  issueTypeDistributionList,
  IssueTypeOptions,
  productDistributionList,
  ProductTypeFilterMap,
  ProductTypeOptions
} from '~/types/issues'
import { getRepositoryIssueProps } from '~/utils/issue'

const PAGE_SIZE = 25
const VISIBLE_PAGES = 5

export interface IssueListFilters extends RouteQueryParamsT {
  page?: number
  category?: string
  product?: string
  analyzer?: string
  q?: string
  sort?: string
  autofixAvailable?: boolean
  all?: boolean
  recommended?: boolean
  auditRequired?: boolean
  severity?: IssueSeverityFilter
}

interface IssueListQueryParams {
  provider: string
  owner: string
  name: string
  limit: number
  currentPageNumber?: number
  issueType?: string
  product?: string
  analyzer?: string
  sort?: string
  q?: string
  autofixAvailable?: boolean
  all?: boolean
  recommended?: boolean
  auditRequired?: boolean
  severity?: IssueSeverityFilter
  refetch?: boolean
}

/**
 * Class for `Issues` page. Lists the issues reported for the repository under various categories.
 */
export default defineComponent({
  name: 'Issues',
  components: {
    IssueAnalyzerSelector,
    IssueListItem,
    IssueCategorySelector,
    IssueSort,
    IssueSearch,
    AutofixAvailable,
    AutofixFileChooser
  },
  layout: 'repository',
  mixins: [RepoDetailMixin, RoleAccessMixin, RouteQueryMixin],
  /**
   * The head method to set the HTML Head tags for issues page
   *
   * @returns {Record<string, string>}
   */
  head() {
    const { owner, repo } = this.$route.params
    return {
      title: `Issues • ${owner}/${formatRepoName(repo)}`
    }
  },
  data() {
    return {
      issueList: [] as RepositoryIssue[],
      totalIssueCount: 0,
      isAutofixOpen: false,
      isActivationModalOpen: false,
      isUpgradeAccountModalOpen: false,
      currentPage: null as Maybe<number>,
      urlFilterState: {} as Record<string, string | (string | null)[]>,
      autofixIssue: {} as { shortcode: string; raisedInFiles: string[]; issueId: string },
      issuesLoading: false,
      issueDistributionType: IssueOccurrenceDistributionType.ISSUE_TYPE,
      showCategorySelectorMobile: true
    }
  },
  /**
   * The fetch hook
   *
   * @returns {Promise<void>}
   */
  async fetch(): Promise<void> {
    await this.getIssuesData()
  },

  /**
   * Mounted hook for the page
   *
   * @returns {void}
   */
  mounted(): void {
    this.$root.$on('refetchCheck', this.refetchIssues)
    this.$root.$on('repo-activation-triggered', this.refetchRepoDetails)
    this.$socket.$on('repo-analysis-updated', this.refetchIssues)
    this.$socket.$on('autofixrun-fixes-ready', this.refetchIssues)
  },

  /**
   * BeforeDestroy hook for the page
   *
   * @returns {void}
   */
  beforeDestroy(): void {
    this.$root.$off('refetchCheck', this.refetchIssues)
    this.$root.$off('repo-activation-triggered', this.refetchRepoDetails)
    this.$socket.$off('repo-analysis-updated', this.refetchIssues)
    this.$socket.$off('autofixrun-fixes-ready', this.refetchIssues)
  },
  computed: {
    IssueOccurrenceDistributionType() {
      return IssueOccurrenceDistributionType
    },

    parsedParams(): IssueListFilters {
      const parsed = {} as IssueListFilters
      const { q, page, sort, analyzer, category, product, autofixAvailable, severity } =
        this.queryParams

      if (q) parsed['q'] = String(q)
      if (page) parsed['page'] = page as number
      if (sort) parsed['sort'] = sort as string
      if (analyzer && analyzer !== 'all') {
        parsed['analyzer'] = analyzer as string
      }
      if (autofixAvailable) {
        parsed['autofixAvailable'] = true
      }

      if (category && IssueCategoryTypeFilterMap[category as string]) {
        const categoryFilter = IssueCategoryTypeFilterMap[category as string]
        Object.assign(parsed, categoryFilter)
      }

      if (product && ProductTypeFilterMap[product as string]) {
        const productFilter = ProductTypeFilterMap[product as string]
        Object.assign(parsed, productFilter)
      }

      // ? Check if severity derived from query params is valid
      if (
        severity &&
        Object.values(IssueSeverityFilter).includes(severity as IssueSeverityFilter)
      ) {
        parsed.severity = severity as IssueSeverityFilter
      }

      return parsed
    },
    issueCategories(): Array<IssueFilterChoice> {
      if (this.issueDistributionType === IssueOccurrenceDistributionType.ISSUE_TYPE) {
        return issueTypeDistributionList
      }
      if (this.issueDistributionType === IssueOccurrenceDistributionType.PRODUCT) {
        return productDistributionList
      }

      return []
    },

    issueOccurrenceDistributionCounts(): Record<string, number> {
      const distributionMap: Record<string, number> = {}
      if (this.issueDistributionType === IssueOccurrenceDistributionType.ISSUE_TYPE) {
        const issueOccurrenceDistributionByIssueType =
          this.repository?.issueOccurrenceDistributionByIssueType ?? []

        issueOccurrenceDistributionByIssueType.forEach((el) => (distributionMap[el.key] = el.count))
      } else if (this.issueDistributionType === IssueOccurrenceDistributionType.PRODUCT) {
        const issueOccurrenceDistributionByProduct =
          this.repository?.issueOccurrenceDistributionByProduct ?? []

        issueOccurrenceDistributionByProduct.forEach((el) => (distributionMap[el.key] = el.count))
      }

      return distributionMap
    },

    canModifyReportingSettings(): boolean {
      return this.$gateKeeper.repo(
        RepoPerms.CHANGE_ISSUE_TYPES_TO_REPORT,
        this.repoPerms.permission
      )
    },

    hasTestCoverage(): boolean {
      return Boolean(this.repository?.hasTestCoverage)
    },

    canCreateAutofix(): boolean {
      if (!this.allowAutofix) {
        return false
      }
      return this.$gateKeeper.repo(RepoPerms.CREATE_AUTOFIXES, this.repoPerms.permission)
    },

    pageCount(): number {
      return Math.ceil(this.totalIssueCount / PAGE_SIZE)
    },

    totalVisible(): number {
      return this.pageCount >= VISIBLE_PAGES ? VISIBLE_PAGES : this.pageCount
    },

    issueCategoryDisabled(): boolean {
      // Return early if the category is falsy or anything among `all` or `recommended`
      if (!this.isDefinedCategory) {
        return false
      }

      const status = this.repository.issueTypeSettings?.find(
        (issueType) => issueType?.slug === this.queryParams.category
      )

      return status?.isIgnoredToDisplay as boolean
    },

    hasInvalidQueryParam(): boolean {
      const { product, category } = this.queryParams

      return (
        Boolean(product && !ProductTypeFilterMap[product as string]) ||
        Boolean(category && !IssueCategoryTypeFilterMap[category as string])
      )
    },

    issueCategoryName(): string {
      // Return early if the category is falsy or anything among `all` or `recommended`
      if (!this.isDefinedCategory) {
        return ''
      }

      const match = issueTypeDistributionList.find(
        (category) => category.shortcode === this.queryParams.category
      )

      return match?.name || ''
    },

    isDefinedCategory(): boolean {
      // Returns `true` if `category` is truthy and it is not set to `all` or `recommended`.
      const { category } = this.queryParams
      return Boolean(category && category !== 'all' && category !== 'recommended')
    },

    issueCategoryIsCoverage(): boolean {
      const { category, product } = this.queryParams
      return category === IssueTypeOptions.COVERAGE || product === ProductTypeOptions.COVERAGE
    },

    disabledStateImagePath(): Record<string, string> {
      if (this.issueCategoryIsCoverage) {
        return {
          webp: require('~/assets/images/ui-states/SSH-Key-136px.webp'),
          png: require('~/assets/images/ui-states/SSH-Key-136px.png')
        }
      }

      return {
        webp: require('~/assets/images/ui-states/runs/no-recent-analyses.webp'),
        png: require('~/assets/images/ui-states/runs/no-recent-analyses.png')
      }
    },

    disabledStateSubtitle(): string {
      return this.issueCategoryIsCoverage
        ? 'Enable Code Coverage tracking from your repository settings and send a coverage report to DeepSource to start seeing coverage issues here.'
        : `You can change this in the repository's settings. Once enabled, ${this.issueCategoryName.toLowerCase()} issues will start getting reported from the next analysis`
    },

    activeSidebarItem(): string {
      const { category, product } = this.queryParams

      if (product && category) {
        // If category type is "all", we just return the product type alone
        if (category === 'all') {
          return product as string
        }

        // But if the category is anything else, we return the combo
        return `${product}-${category}`
      }

      return (category || product || '') as string
    },

    showAutofixAvailableFilter(): boolean {
      const autofixableIssueCount =
        this.repository.autofixableIssuesMetadata?.autofixableIssueCount ?? 0

      return this.allowAutofix && this.hasRepoReadAccess && autofixableIssueCount > 0
    }
  },
  methods: {
    getRepositoryIssueProps,
    /**
     * Function to update category query param based on the new value received
     *
     * @param {string} newVal
     * @returns {void}
     */
    updateCategory(newVal: string, newSubCategory?: string): void {
      let filtersToAdd: Record<string, string | number | null> = {}

      if (
        this.issueDistributionType === IssueOccurrenceDistributionType.ISSUE_TYPE ||
        newVal === 'all' ||
        newVal === 'recommended'
      ) {
        filtersToAdd = { category: newVal, page: 1, product: null }
      } else if (this.issueDistributionType === IssueOccurrenceDistributionType.PRODUCT) {
        filtersToAdd = {
          product: newVal,
          page: 1,
          category: newSubCategory ?? 'all'
        }
      }

      this.addFilters(filtersToAdd)
    },
    /**
     * Function to update analyzer query param based on the new value received
     *
     * @param {string} newVal
     * @returns {void}
     */
    updateAnalyzer(newVal: string): void {
      this.addFilters({ analyzer: newVal, page: 1 })
    },
    /**
     * Callback for route replace
     *
     * @return {Promise<void>}
     */
    async refetchAfterRouteChange(): Promise<void> {
      // don't use fetch, because it cannot be executed concurrently
      await this.getIssuesData()
    },
    /**
     * Methods to fetch all the data required for the page
     * Also used as a callback for refetch after route replace
     *
     * @return {Promise<void>}
     */
    async getIssuesData(): Promise<void> {
      // Ensure updates to query params happen before accessing
      await this.fetchIssuesForOwner()

      try {
        await Promise.all([
          this.fetchRepoDetails(this.baseRouteParams),
          this.fetchRepoPerms(this.baseRouteParams),
          this.fetchRepoAutofixStats(this.baseRouteParams)
        ])
      } catch (err) {
        this.$logErrorAndToast(
          err as Error,
          'Unable to fetch details for the repository. Please try again later or contact support.'
        )
      }
    },
    /**
     * Fetch occurrence counts, issue list and type distribution info
     *
     * @param {boolean} [refetch=false]
     * @returns {Promise<void>}
     */
    async fetchIssuesForOwner(refetch = false): Promise<void> {
      this.issuesLoading = true

      const {
        q,
        page,
        sort,
        analyzer,
        autofixAvailable,
        product,
        all,
        category,
        auditRequired,
        recommended,
        severity
      } = this.parsedParams

      if (product && this.issueDistributionType !== IssueOccurrenceDistributionType.PRODUCT) {
        this.issueDistributionType = IssueOccurrenceDistributionType.PRODUCT
      }

      try {
        await this.fetchIssueOccurrenceDistributionCounts({
          ...this.baseRouteParams,
          distributionType: this.issueDistributionType,
          q,
          analyzer,
          autofixAvailable,
          severity,
          refetch
        })
      } catch (err) {
        this.$logErrorAndToast(
          err as Error,
          'Unable to fetch issue occurrence counts. Please try again later or contact support.'
        )
      }

      // only apply default category when no category is specified in query params or invalid params
      if (!this.queryParams.category || this.hasInvalidQueryParam) {
        if (this.issueOccurrenceDistributionCounts['recommended']) {
          this.updateCategory('recommended')
          return
        }
        this.updateCategory('all')
        return
      }

      try {
        await Promise.all([
          this.fetchIssueList({
            ...this.baseRouteParams,
            currentPageNumber: page,
            limit: PAGE_SIZE,
            sort,
            q,
            analyzer,
            product,
            issueType: category,
            autofixAvailable,
            auditRequired,
            recommended,
            all,
            severity,
            refetch
          }),
          this.fetchIssueTypeSettings({ ...this.baseRouteParams, refetch })
        ])
      } catch (err) {
        this.$logErrorAndToast(
          err as Error,
          'Unable to fetch issues. Please try again later or contact support.'
        )
      }

      this.issuesLoading = false
    },

    /**
     * Refetch issues on websocket event
     *
     * @returns {Promise<void>}
     */
    refetchIssues(): void {
      this.fetchIssuesForOwner(true)
    },

    /**
     * Fetch list of repository issues
     *
     * @param {IssueListQueryParams} args — arguments for fetching repository issues
     * @returns {Promise<void>}
     */
    async fetchIssueList({
      provider,
      owner,
      name,
      limit,
      currentPageNumber,
      issueType,
      product,
      analyzer,
      sort,
      q,
      autofixAvailable,
      all,
      recommended,
      auditRequired,
      severity,
      refetch
    }: IssueListQueryParams): Promise<void> {
      const response: GraphqlQueryResponse = await this.$fetchGraphqlData(
        RepositoryIssuesListGQLQuery,
        {
          provider: this.$providerMetaMap[provider].value,
          owner,
          name,
          after: this.$getGQLAfter(currentPageNumber || 1, limit),
          limit,
          issueType,
          product,
          analyzer,
          sort,
          q,
          autofixAvailable,
          all,
          recommended,
          auditRequired,
          severity
        },
        refetch
      )

      this.issueList = resolveNodes(response.data.repository?.issues)
      this.totalIssueCount = response.data.repository?.issues?.totalCount ?? 0
    },
    /**
     * Refetch repo details on websocket event
     *
     * @returns {Promise<void>}
     */
    async refetchRepoDetails(): Promise<void> {
      await this.fetchBasicRepoDetails({ ...this.baseRouteParams, refetch: true })
    },

    async handleDistributionTypeChange(
      distributionType: IssueOccurrenceDistributionType,
      refetch = false
    ) {
      this.issueDistributionType = distributionType
      const { q, analyzer, autofixAvailable, severity } = this.parsedParams

      await this.fetchIssueOccurrenceDistributionCounts({
        ...this.baseRouteParams,
        distributionType,
        q,
        analyzer,
        autofixAvailable,
        severity,
        refetch
      })

      if (this.issueOccurrenceDistributionCounts['recommended']) {
        this.updateCategory('recommended')
      } else {
        this.updateCategory('all')
      }
    },

    /**
     * Function to open Autofix modal on the autofix event
     *
     * @param {Record<string, string | Array<string>>} issue
     * @returns {void}
     */
    openAutofixModal(issue: { shortcode: string; raisedInFiles: string[]; issueId: string }): void {
      this.autofixIssue = { ...issue }
      this.isAutofixOpen = true
    },

    /**
     * Function to open Upgrade account modal on the runQuotaExhausted event
     *
     * @returns {void}
     */
    openUpgradeAccountModal() {
      this.isAutofixOpen = false
      this.isUpgradeAccountModalOpen = true
    }
  }
})
</script>
<style scoped>
.issues-page {
  --repo-header-height: 99px;
  --analyzers-filter-header-offset: 51px;

  --category-sidebar-top-offset: calc(
    var(--repo-header-height) + var(--analyzers-filter-header-offset)
  );
}

@media screen and (min-width: 1024px) {
  .analyzer-tab {
    top: var(--repo-header-height);
  }
}

.category-sidebar {
  height: calc(100vh - var(--category-sidebar-top-offset));
  top: var(--category-sidebar-top-offset);
}
</style>
