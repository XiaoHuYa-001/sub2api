<template>
  <BaseDialog
    :show="show"
    :title="t('admin.accounts.importTokensTitle')"
    width="normal"
    close-on-click-outside
    @close="handleClose"
  >
    <form id="import-tokens-form" class="space-y-4" @submit.prevent="handleImport">
      <div class="text-sm text-gray-600 dark:text-dark-300">
        {{ t('admin.accounts.importTokensHint') }}
      </div>

      <!-- Input mode toggle -->
      <div class="flex rounded-lg bg-gray-100 p-1 dark:bg-dark-700">
        <button
          type="button"
          @click="inputMode = 'paste'"
          :class="[
            'flex flex-1 items-center justify-center rounded-md px-3 py-2 text-sm font-medium transition-all',
            inputMode === 'paste'
              ? 'bg-white text-gray-900 shadow-sm dark:bg-dark-600 dark:text-white'
              : 'text-gray-600 hover:text-gray-900 dark:text-gray-400 dark:hover:text-gray-200'
          ]"
        >
          {{ t('admin.accounts.importTokensPaste') }}
        </button>
        <button
          type="button"
          @click="inputMode = 'file'"
          :class="[
            'flex flex-1 items-center justify-center rounded-md px-3 py-2 text-sm font-medium transition-all',
            inputMode === 'file'
              ? 'bg-white text-gray-900 shadow-sm dark:bg-dark-600 dark:text-white'
              : 'text-gray-600 hover:text-gray-900 dark:text-gray-400 dark:hover:text-gray-200'
          ]"
        >
          {{ t('admin.accounts.importTokensFile') }}
        </button>
      </div>

      <!-- Paste mode -->
      <div v-if="inputMode === 'paste'">
        <label class="input-label">{{ t('admin.accounts.importTokensInput') }}</label>
        <textarea
          v-model="tokenText"
          rows="6"
          class="input font-mono text-xs"
          :placeholder="t('admin.accounts.importTokensPlaceholder')"
          :disabled="importing"
        />
      </div>

      <!-- File mode -->
      <div v-if="inputMode === 'file'">
        <label class="input-label">{{ t('admin.accounts.importTokensFileLabel') }}</label>
        <div
          class="flex items-center justify-between gap-3 rounded-lg border border-dashed border-gray-300 bg-gray-50 px-4 py-3 dark:border-dark-600 dark:bg-dark-800"
        >
          <div class="min-w-0">
            <div class="truncate text-sm text-gray-700 dark:text-dark-200">
              {{ fileName || t('admin.accounts.importTokensSelectFile') }}
            </div>
            <div class="text-xs text-gray-500 dark:text-dark-400">.txt / .json</div>
          </div>
          <button type="button" class="btn btn-secondary shrink-0" @click="openFilePicker">
            {{ t('common.chooseFile') }}
          </button>
        </div>
        <input
          ref="fileInput"
          type="file"
          class="hidden"
          accept=".txt,.json,text/plain,application/json"
          @change="handleFileChange"
        />
      </div>

      <!-- Account name prefix -->
      <div>
        <label class="input-label">{{ t('admin.accounts.importTokensNamePrefix') }}</label>
        <input
          v-model="namePrefix"
          type="text"
          class="input"
          :placeholder="t('admin.accounts.importTokensNamePrefixPlaceholder')"
          :disabled="importing"
        />
      </div>

      <!-- Concurrency -->
      <div>
        <label class="input-label">{{ t('admin.accounts.concurrency') }}</label>
        <input
          v-model.number="concurrency"
          type="number"
          min="1"
          max="100"
          class="input"
          :disabled="importing"
        />
      </div>

      <!-- Progress -->
      <div v-if="importing" class="space-y-2">
        <div class="flex items-center justify-between text-sm">
          <span class="text-gray-600 dark:text-dark-300">
            {{ t('admin.accounts.importTokensProgress', { current: processedCount, total: totalCount }) }}
          </span>
          <span class="text-gray-500 dark:text-dark-400">
            {{ Math.round((processedCount / Math.max(totalCount, 1)) * 100) }}%
          </span>
        </div>
        <div class="h-2 overflow-hidden rounded-full bg-gray-200 dark:bg-dark-700">
          <div
            class="h-full rounded-full bg-primary-500 transition-all duration-300"
            :style="{ width: `${(processedCount / Math.max(totalCount, 1)) * 100}%` }"
          />
        </div>
      </div>

      <!-- Result -->
      <div
        v-if="result"
        class="space-y-2 rounded-xl border border-gray-200 p-4 dark:border-dark-700"
      >
        <div class="text-sm font-medium text-gray-900 dark:text-white">
          {{ t('admin.accounts.importTokensResult') }}
        </div>
        <div class="text-sm text-gray-700 dark:text-dark-300">
          {{ t('admin.accounts.importTokensResultSummary', { success: result.success, failed: result.failed, total: result.total }) }}
        </div>

        <div v-if="result.errors.length" class="mt-2">
          <div class="text-sm font-medium text-red-600 dark:text-red-400">
            {{ t('admin.accounts.importTokensErrors') }}
          </div>
          <div
            class="mt-2 max-h-48 overflow-auto rounded-lg bg-gray-50 p-3 font-mono text-xs dark:bg-dark-800"
          >
            <div v-for="(item, idx) in result.errors" :key="idx" class="whitespace-pre-wrap">
              #{{ item.index }} — {{ item.message }}
            </div>
          </div>
        </div>
      </div>
    </form>

    <template #footer>
      <div class="flex justify-end gap-3">
        <button class="btn btn-secondary" type="button" :disabled="importing" @click="handleClose">
          {{ t('common.cancel') }}
        </button>
        <button
          class="btn btn-primary"
          type="submit"
          form="import-tokens-form"
          :disabled="importing"
        >
          {{ importing ? t('admin.accounts.importTokensImporting') : t('admin.accounts.importTokensButton') }}
        </button>
      </div>
    </template>
  </BaseDialog>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import BaseDialog from '@/components/common/BaseDialog.vue'
import { adminAPI } from '@/api/admin'
import { useAppStore } from '@/stores/app'
import { useOpenAIOAuth } from '@/composables/useOpenAIOAuth'

interface Props {
  show: boolean
}

interface Emits {
  (e: 'close'): void
  (e: 'imported'): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

const { t } = useI18n()
const appStore = useAppStore()
const openaiOAuth = useOpenAIOAuth()

const inputMode = ref<'paste' | 'file'>('paste')
const tokenText = ref('')
const file = ref<File | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)
const namePrefix = ref('GPT')
const concurrency = ref(10)

const importing = ref(false)
const processedCount = ref(0)
const totalCount = ref(0)

interface ImportResult {
  total: number
  success: number
  failed: number
  errors: Array<{ index: number; message: string }>
}

const result = ref<ImportResult | null>(null)

const fileName = computed(() => file.value?.name || '')

watch(
  () => props.show,
  (open) => {
    if (open) {
      tokenText.value = ''
      file.value = null
      result.value = null
      processedCount.value = 0
      totalCount.value = 0
      inputMode.value = 'paste'
      if (fileInput.value) {
        fileInput.value.value = ''
      }
    }
  }
)

const openFilePicker = () => {
  fileInput.value?.click()
}

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  file.value = target.files?.[0] || null
}

const handleClose = () => {
  if (importing.value) return
  emit('close')
}

/**
 * Detect whether a token string is an access_token (JWT) or refresh_token.
 * OpenAI access_tokens are JWTs starting with "eyJ".
 */
function isAccessToken(token: string): boolean {
  return token.startsWith('eyJ')
}

const readFileAsText = async (sourceFile: File): Promise<string> => {
  if (typeof sourceFile.text === 'function') {
    return sourceFile.text()
  }
  return await new Promise<string>((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(String(reader.result ?? ''))
    reader.onerror = () => reject(reader.error || new Error('Failed to read file'))
    reader.readAsText(sourceFile)
  })
}

/**
 * Parse tokens from raw text input.
 * Supports:
 *  - one token per line (plain text)
 *  - JSON array of strings: ["token1", "token2"]
 *  - JSON array of objects: [{"access_token": "..."}, {"refresh_token": "..."}]
 */
function parseTokens(raw: string): string[] {
  const trimmed = raw.trim()

  // Try JSON array
  if (trimmed.startsWith('[')) {
    try {
      const parsed = JSON.parse(trimmed)
      if (Array.isArray(parsed)) {
        return parsed.flatMap((item) => {
          if (typeof item === 'string') return [item.trim()]
          if (typeof item === 'object' && item !== null) {
            // Support {"access_token": "..."} or {"refresh_token": "..."} or {"token": "..."}
            const token = item.access_token || item.refresh_token || item.token || ''
            return token ? [String(token).trim()] : []
          }
          return []
        }).filter(Boolean)
      }
    } catch {
      // Not valid JSON, fall through to line-by-line
    }
  }

  // Line-by-line
  return trimmed
    .split('\n')
    .map((line) => line.trim())
    .filter((line) => line && !line.startsWith('#') && !line.startsWith('//'))
}

const handleImport = async () => {
  let rawText = ''

  if (inputMode.value === 'paste') {
    rawText = tokenText.value
  } else {
    if (!file.value) {
      appStore.showError(t('admin.accounts.importTokensSelectFile'))
      return
    }
    try {
      rawText = await readFileAsText(file.value)
    } catch {
      appStore.showError(t('admin.accounts.importTokensParseFailed'))
      return
    }
  }

  const tokens = parseTokens(rawText)
  if (tokens.length === 0) {
    appStore.showError(t('admin.accounts.importTokensEmpty'))
    return
  }

  importing.value = true
  result.value = null
  processedCount.value = 0
  totalCount.value = tokens.length

  const importResult: ImportResult = { total: tokens.length, success: 0, failed: 0, errors: [] }

  try {
    for (let i = 0; i < tokens.length; i++) {
      const token = tokens[i]
      const index = i + 1

      try {
        let credentials: Record<string, unknown>
        let extra: Record<string, unknown> | undefined

        if (isAccessToken(token)) {
          // Access token: use directly as credentials
          credentials = { access_token: token }
        } else {
          // Refresh token: call backend to exchange for full token info
          const tokenInfo = await openaiOAuth.validateRefreshToken(token)
          if (!tokenInfo) {
            importResult.failed++
            importResult.errors.push({
              index,
              message: openaiOAuth.error.value || 'Refresh token validation failed'
            })
            openaiOAuth.error.value = ''
            processedCount.value = index
            continue
          }
          credentials = openaiOAuth.buildCredentials(tokenInfo)
          const extraInfo = openaiOAuth.buildExtraInfo(tokenInfo)
          if (extraInfo) {
            extra = extraInfo
          }
        }

        const accountName = tokens.length > 1
          ? `${namePrefix.value || 'GPT'} #${index}`
          : (namePrefix.value || 'GPT')

        await adminAPI.accounts.create({
          name: accountName,
          platform: 'openai',
          type: 'oauth',
          credentials,
          extra,
          concurrency: concurrency.value
        })

        importResult.success++
      } catch (error: any) {
        importResult.failed++
        const errMsg = error.message || error.response?.data?.message || 'Unknown error'
        importResult.errors.push({ index, message: errMsg })
      }

      processedCount.value = index
    }

    result.value = importResult

    if (importResult.success > 0 && importResult.failed === 0) {
      appStore.showSuccess(t('admin.accounts.importTokensSuccess', { count: importResult.success }))
      emit('imported')
    } else if (importResult.success > 0 && importResult.failed > 0) {
      appStore.showWarning(
        t('admin.accounts.importTokensPartialSuccess', {
          success: importResult.success,
          failed: importResult.failed
        })
      )
      emit('imported')
    } else {
      appStore.showError(t('admin.accounts.importTokensAllFailed'))
    }
  } finally {
    importing.value = false
  }
}
</script>
