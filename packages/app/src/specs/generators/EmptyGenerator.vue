<template>
  <div class="flex flex-col flex-grow justify-between">
    <template v-if="!result">
      <div class="p-24px w-720px">
        <Input
          v-model="specFile"
          :placeholder="t('createSpec.e2e.importEmptySpec.inputPlaceholder')"
          :aria-label="t('createSpec.e2e.importEmptySpec.inputPlaceholder')"
          :has-error="hasError"
        >
          <template #prefix>
            <i-cy-document-blank_x16
              class="icon-light-gray-50 icon-dark-gray-300"
              :class="{
                'icon-light-error-50 icon-dark-error-400': hasError,
              }"
            />
          </template>
        </Input>

        <div
          v-if="props.gql"
        >
          <div
            v-if="hasError"
            class="rounded flex font-medium bg-error-100 mt-16px p-14px ring-2 ring-error-100 text-error-600 gap-8px items-center"
          >
            <i-cy-errored-outline_x16 class="icon-dark-error-600" />
            <span>{{ invalidSpecWarning }}<em class="font-medium">specPattern</em>:</span>
          </div>
          <div
            v-else-if="showExtensionWarning && props.type === 'e2e'"
            class="rounded flex font-medium bg-warning-100 mt-16px p-16px text-warning-600 gap-8px items-center"
          >
            <i-cy-errored-outline_x16 class="icon-dark-warning-600" />
            {{ t('createSpec.e2e.importEmptySpec.specExtensionWarning') }}<span class="rounded bg-warning-200 py-2px px-8px text-warning-700">{{ recommendedFileName }}</span>
          </div>

          <div
            v-if="hasError"
            class="mt-16px"
          >
            <SpecPatterns
              :gql="props.gql"
              variant="info"
            />
          </div>
        </div>
      </div>
      <StandardModalFooter
        v-if="!result"
        class="flex gap-16px"
      >
        <Button
          size="lg"
          :disabled="!isValidSpecFile"
          @click="createSpec"
        >
          {{ t('createSpec.createSpec') }}
        </Button>

        <Button
          v-if="props.otherGenerators"
          size="lg"
          variant="outline"
          @click="emits('restart')"
        >
          {{ t('components.button.back') }}
        </Button>
        <Button
          v-else
          size="lg"
          variant="outline"
          @click="emits('close')"
        >
          {{ t('components.button.cancel') }}
        </Button>
      </StandardModalFooter>
    </template>

    <template v-else>
      <GeneratorSuccess
        :file="result.file"
      />
      <StandardModalFooter
        class="flex gap-16px items-center"
      >
        <router-link
          class="outline-none"
          :to="{
            name: 'SpecRunner',
            query: {
              file: result.file.relative?.replace(/\\/g, '/')
            },
            params: props.type === 'component'
              ? {
                shouldShowTroubleRenderingAlert: true
              }
              : undefined
          }"
        >
          <Button
            size="lg"
            :prefix-icon="TestResultsIcon"
            prefix-icon-class="w-16px h-16px icon-dark-white"
            @click="emits('close')"
          >
            {{ t('createSpec.successPage.runSpecButton') }}
          </Button>
        </router-link>
        <Button
          size="lg"
          :prefix-icon="PlusButtonIcon"
          prefix-icon-class="w-16px h-16px icon-dark-gray-500"
          variant="outline"
          @click="emits('restart')"
        >
          {{ t('createSpec.successPage.createAnotherSpecButton') }}
        </Button>
      </StandardModalFooter>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import { useI18n } from '@packages/frontend-shared/src/locales/i18n'
import Input from '@packages/frontend-shared/src/components/Input.vue'
import Button from '@packages/frontend-shared/src/components/Button.vue'
import { useVModels, whenever } from '@vueuse/core'
import { gql, useMutation } from '@urql/vue'
import SpecPatterns from '../../components/SpecPatterns.vue'
import type { EmptyGeneratorFragment, GeneratorSuccessFileFragment } from '../../generated/graphql'
import { EmptyGenerator_MatchSpecFileDocument, EmptyGenerator_GenerateSpecDocument } from '../../generated/graphql'
import StandardModalFooter from '@packages/frontend-shared/src/components/StandardModalFooter.vue'
import GeneratorSuccess from './GeneratorSuccess.vue'
import TestResultsIcon from '~icons/cy/test-results_x24.svg'
import PlusButtonIcon from '~icons/cy/add-large_x16.svg'

const props = defineProps<{
  title: string
  gql: EmptyGeneratorFragment
  type: 'e2e' | 'component'
  specFileName: string
  erroredCodegenCandidate?: string
  /** is there any other generator available when clicking "Back" */
  otherGenerators: boolean
}>()

const { t } = useI18n()

gql`
fragment EmptyGenerator on CurrentProject {
  id
  config
  ...SpecPatterns
}
`

gql`
mutation EmptyGenerator_MatchSpecFile($specFile: String!) {
  matchesSpecPattern (specFile: $specFile) 
}
`

gql`
mutation EmptyGenerator_generateSpec($codeGenCandidate: String!, $type: CodeGenType!, $erroredCodegenCandidate: String) {
  generateSpecFromSource(codeGenCandidate: $codeGenCandidate, type: $type, erroredCodegenCandidate: $erroredCodegenCandidate) {
    ...GeneratorSuccess
  }
}`

const emits = defineEmits<{
  (event: 'update:title', value: string): void
  (event: 'update:description', value: string): void
  (event: 'restart'): void
  (event: 'close'): void
  (event: 'updateTitle', value: string): void
}>()

const { title } = useVModels(props, emits)

const specFile = ref(props.specFileName)

const matches = useMutation(EmptyGenerator_MatchSpecFileDocument)
const writeFile = useMutation(EmptyGenerator_GenerateSpecDocument)

const isValidSpecFile = ref(true)
const hasError = computed(() => !isValidSpecFile.value && !!specFile.value)

const result = ref<GeneratorSuccessFileFragment | null>(null)

whenever(result, () => {
  title.value = t('createSpec.successPage.header')
  emits('updateTitle', t('createSpec.successPage.header'))
})

const createSpec = async () => {
  const { data } = await writeFile.executeMutation({ codeGenCandidate: specFile.value, type: props.type, erroredCodegenCandidate: props.erroredCodegenCandidate ?? null })

  result.value = data?.generateSpecFromSource?.generatedSpecResult?.__typename === 'ScaffoldedFile' ? data?.generateSpecFromSource?.generatedSpecResult : null
}

watch(specFile, async (value) => {
  const result = await matches.executeMutation({ specFile: value })

  isValidSpecFile.value = result.data?.matchesSpecPattern ?? false
}, { immediate: true })

title.value = t('createSpec.e2e.importEmptySpec.chooseFilenameHeader')

const showExtensionWarning = computed(() => isValidSpecFile.value && !specFile.value.includes('.cy'))
const recommendedFileName = computed(() => {
  const split = specFile.value.split('.')

  return `{filename}.cy.${split[split.length - 1]}`
})

const invalidSpecWarning = computed(() => props.type === 'e2e' ? t('createSpec.e2e.importEmptySpec.invalidSpecWarning') : t('createSpec.component.importEmptySpec.invalidComponentWarning'))

</script>
