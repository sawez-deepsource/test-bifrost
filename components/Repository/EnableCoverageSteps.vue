<template>
  <z-accordion class="p-4 pb-0">
    <z-accordion-item
      :span-custom-height="true"
      custom-max-height="max-h-full"
      @isOpen="isOpenHandler"
    >
      <template #title="{ toggleAccordion }">
        <div class="flex cursor-pointer items-center gap-1 pb-4" @click="toggleAccordion">
          <z-icon
            icon="chevron-right"
            size="x-small"
            class="transform"
            :class="chevronIconAnimateClass"
          />
          <span class="text-xs leading-6 text-vanilla-400">
            How to send coverage reports to DeepSource?
          </span>
        </div>
      </template>

      <z-stepper :show-numbers="true" align="vertical">
        <z-step>
          <template #title>
            <p class="text-xs text-vanilla-400">
              Set up the
              <code class="bifrost-inline-code uppercase text-vanilla-100">DEEPSOURCE_DSN</code>
              environment variable.
            </p>
          </template>
          <template #description>
            <div
              class="copy-content-container h-8 max-w-2xs items-center gap-x-4 py-2 sm:max-w-none"
            >
              <code class="truncate font-mono text-xs font-normal leading-4 text-vanilla-100">
                {{ dsn }}
              </code>

              <unstyled-copy-button
                v-tooltip="'Copy DeepSource DSN'"
                :value="dsn"
                icon-size="x-small"
                class="auxillary-button text-xs text-vanilla-100"
              />
            </div>
          </template>
        </z-step>

        <z-step>
          <template #title>
            <p class="mb-2 text-xs text-vanilla-400">
              Install the
              <a
                href="https://deepsource.com/cli"
                target="blank"
                rel="noreferrer noopener"
                class="inline-flex items-center gap-x-1 text-juniper"
              >
                DeepSource CLI
                <z-icon icon="arrow-up-right" size="x-small" color="current" />
              </a>
            </p>
          </template>
          <template #description>
            <div
              class="copy-content-container h-8 max-w-2xs items-center gap-x-2 py-2 sm:max-w-none"
            >
              <code class="truncate font-mono text-xs leading-3 text-vanilla-100">
                <span class="select-none text-honey">$</span>
                curl -fsSL
                <span class="text-aqua-500">https://cli.deepsource.com/install</span>
                <span class="text-honey"> | BINDIR=./bin sh</span>
              </code>

              <unstyled-copy-button
                :value="installationCommand"
                label="Copy"
                icon-size="x-small"
                class="auxillary-button text-xs text-vanilla-100"
              />
            </div>
          </template>
        </z-step>

        <z-step>
          <template #title>
            <p class="mb-2 text-xs leading-6 text-vanilla-400">
              Report your coverage artifact to DeepSource using the CLI.
            </p>
          </template>
          <template #description>
            <div class="copy-content-container max-w-2xs items-start sm:max-w-none">
              <code
                class="line-clamp-3 max-w-md py-2 font-mono text-xs font-normal leading-4 text-vanilla-100"
              >
                {{ cliCommand }}
              </code>

              <unstyled-copy-button
                v-tooltip="'Copy CLI command'"
                :value="cliCommand"
                icon-size="x-small"
                class="auxillary-button mt-1 text-xs text-vanilla-100"
              />
            </div>

            <a
              href="https://docs.deepsource.com/docs/analyzers-test-coverage"
              target="_blank"
              rel="noopener noreferrer"
              class="auxillary-link mt-3 h-7 w-max text-xs font-normal text-vanilla-400"
            >
              Read full documentation
              <z-icon icon="arrow-up-right" size="x-small" color="current" />
            </a>
          </template>
        </z-step>
      </z-stepper>
    </z-accordion-item>
  </z-accordion>
</template>
<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'EnableCoverageSteps',
  props: {
    dsn: {
      type: String,
      required: true
    }
  },

  data() {
    return {
      chevronIconAnimateClass: ''
    }
  },
  computed: {
    cliCommand(): string {
      const base =
        './bin/deepsource report --analyzer test-coverage --key <key> --value-file <path-to-coverage-report>'
      return this.$config.onPrem ? `${base} --host https://${this.$config.domain}` : base
    },
    installationCommand(): string {
      return 'curl -fsSL https://cli.deepsource.com/install | BINDIR=./bin sh'
    }
  },
  methods: {
    isOpenHandler(open: boolean): void {
      this.chevronIconAnimateClass = open
        ? 'animate-first-quarter-spin rotate-90'
        : 'animate-reverse-quarter-spin rotate-0'
    }
  }
})
</script>

<style lang="postcss" scoped>
.auxillary-link {
  display: flex;
  cursor: pointer;
  align-items: center;
  gap: 0 0.375rem;
  border-radius: 0.125rem;
  border-width: 1px;
  border-color: #303540;
  background-color: #23262e;
  padding-left: 0.5rem;
  padding-right: 0.5rem;
}

.auxillary-link:hover {
  background-color: #2a2e37;
}

.copy-content-container {
  display: flex;
  justify-content: space-between;
  border-radius: 0.125rem;
  border-width: 1px;
  border-color: #23262e;
  background-color: #121317;
  padding-left: 0.625rem;
  padding-right: 0.25rem;
}
</style>
