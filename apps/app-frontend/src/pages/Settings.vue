<script setup>
import { ref, watch } from 'vue'
import { LogOutIcon, LogInIcon, BoxIcon, FolderSearchIcon, UpdatedIcon } from '@modrinth/assets'
import { Card, Slider, DropdownSelect, Toggle, Modal, Button } from '@modrinth/ui'
import { handleError, useTheming } from '@/store/state'
import { is_dir_writeable, change_config_dir, get, set } from '@/helpers/settings'
import { get_java_versions, get_max_memory, set_java_version } from '@/helpers/jre'
import { get as getCreds, logout } from '@/helpers/mr_auth.js'
import JavaSelector from '@/components/ui/JavaSelector.vue'
import ModrinthLoginScreen from '@/components/ui/tutorial/ModrinthLoginScreen.vue'
import { mixpanel_opt_out_tracking, mixpanel_opt_in_tracking } from '@/helpers/mixpanel'
import { open } from '@tauri-apps/api/dialog'
import { getOS } from '@/helpers/utils.js'
import { getVersion } from '@tauri-apps/api/app'
import { get_user } from '@/helpers/cache.js'

const pageOptions = ['Home', 'Library']

const themeStore = useTheming()

const version = await getVersion()

const accessSettings = async () => {
  const settings = await get()

  settings.launchArgs = settings.extra_launch_args.join(' ')
  settings.envVars = settings.custom_env_vars.map((x) => x.join('=')).join(' ')

  return settings
}

const fetchSettings = await accessSettings().catch(handleError)

const settings = ref(fetchSettings)
// const settingsDir = ref(settings.value.loaded_config_dir)

const maxMemory = ref(Math.floor((await get_max_memory().catch(handleError)) / 1024))

watch(
  settings,
  async (oldSettings, newSettings) => {
    if (oldSettings.loaded_config_dir !== newSettings.loaded_config_dir) {
      return
    }

    const setSettings = JSON.parse(JSON.stringify(newSettings))

    if (setSettings.telemetry) {
      mixpanel_opt_out_tracking()
    } else {
      mixpanel_opt_in_tracking()
    }

    setSettings.extra_launch_args = setSettings.launchArgs.trim().split(/\s+/).filter(Boolean)
    setSettings.custom_env_vars = setSettings.envVars
      .trim()
      .split(/\s+/)
      .filter(Boolean)
      .map((x) => x.split('=').filter(Boolean))

    if (!setSettings.hooks.pre_launch) {
      setSettings.hooks.pre_launch = null
    }
    if (!setSettings.hooks.wrapper) {
      setSettings.hooks.wrapper = null
    }
    if (!setSettings.hooks.post_exit) {
      setSettings.hooks.post_exit = null
    }

    if (!setSettings.custom_dir) {
      setSettings.custom_dir = null
    }

    await set(setSettings)
  },
  { deep: true },
)

const javaVersions = ref(await get_java_versions().catch(handleError))
async function updateJavaVersion(version) {
  if (version?.path === '') {
    version.path = undefined
  }

  if (version?.path) {
    version.path = version.path.replace('java.exe', 'javaw.exe')
  }

  await set_java_version(version).catch(handleError)
}

async function fetchCredentials() {
  const creds = await getCreds().catch(handleError)
  console.log(creds)
  if (creds && creds.user_id) {
    creds.user = await get_user(creds.user_id).catch(handleError)
  }
  credentials.value = creds
}

const credentials = ref()
await fetchCredentials()

const loginScreenModal = ref()

async function logOut() {
  await logout().catch(handleError)
  await fetchCredentials()
}

async function signInAfter() {
  await fetchCredentials()
}

async function findLauncherDir() {
  const newDir = await open({
    multiple: false,
    directory: true,
    title: 'Select a new app directory',
  })

  if (newDir) {
    settings.value.custom_dir = newDir
  }
}
</script>

<template>
  <div class="settings-page">
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">常规设置</span>
        </h3>
      </div>
      <ModrinthLoginScreen ref="loginScreenModal" :callback="signInAfter" />
      <div class="adjacent-input">
        <label for="theme">
          <span class="label__title">管理账号</span>
          <span v-if="credentials" class="label__description">
            您当前以以下身份登录 {{ credentials.user.username }}.
          </span>
          <span v-else> 登录您的Modrinth帐户。 </span>
        </label>
        <button v-if="credentials" class="btn" @click="logOut">
          <LogOutIcon />
          Sign out
        </button>
        <button v-else class="btn" @click="$refs.loginScreenModal.show()">
          <LogInIcon />
          Sign in
        </button>
      </div>
      <label for="theme">
        <span class="label__title">App 目录</span>
        <span class="label__description">
          启动器存储其所有文件的目录。
        </span>
      </label>
      <div class="app-directory">
        <div class="iconified-input">
          <BoxIcon />
          <input id="appDir" v-model="settings.custom_dir" type="text" class="input" />
          <Button class="r-btn" @click="findLauncherDir">
            <FolderSearchIcon />
          </Button>
        </div>
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">显示</span>
        </h3>
      </div>
      <div class="adjacent-input">
        <label for="theme">
          <span class="label__title">颜色主题</span>
          <span class="label__description">更改全局启动器颜色主题。</span>
        </label>
        <DropdownSelect
          id="theme"
          name="Theme dropdown"
          :options="themeStore.themeOptions"
          :default-value="settings.theme"
          :model-value="settings.theme"
          class="theme-dropdown"
          @change="
            (e) => {
              themeStore.setThemeState(e.option.toLowerCase())
              settings.theme = themeStore.selectedTheme
            }
          "
        />
      </div>
      <div class="adjacent-input">
        <label for="advanced-rendering">
          <span class="label__title">高级渲染</span>
          <span class="label__description">
            启用高级渲染，例如在没有硬件加速渲染的情况下可能会导致性能问题的模糊效果。
          </span>
        </label>
        <Toggle
          id="advanced-rendering"
          :model-value="themeStore.advancedRendering"
          :checked="themeStore.advancedRendering"
          @update:model-value="
            (e) => {
              themeStore.advancedRendering = e
              settings.advanced_rendering = themeStore.advancedRendering
            }
          "
        />
      </div>
      <div class="adjacent-input">
        <label for="minimize-launcher">
          <span class="label__title">最小化启动器</span>
          <span class="label__description"
            >当Minecraft进程启动时，最小化启动器。</span
          >
        </label>
        <Toggle
          id="minimize-launcher"
          :model-value="settings.hide_on_process_start"
          :checked="settings.hide_on_process_start"
          @update:model-value="
            (e) => {
              settings.hide_on_process_start = e
            }
          "
        />
      </div>
      <div v-if="getOS() != 'MacOS'" class="adjacent-input">
        <label for="native-decorations">
          <span class="label__title">Native decorations</span>
          <span class="label__description">Use system window frame (app restart required).</span>
        </label>
        <Toggle
          id="native-decorations"
          :model-value="settings.native_decorations"
          :checked="settings.native_decorations"
          @update:model-value="
            (e) => {
              settings.native_decorations = e
            }
          "
        />
      </div>
      <div class="adjacent-input">
        <label for="opening-page">
          <span class="label__title">Default landing page</span>
          <span class="label__description">Change the page to which the launcher opens on.</span>
        </label>
        <DropdownSelect
          id="opening-page"
          name="Opening page dropdown"
          :options="pageOptions"
          :default-value="settings.default_page"
          :model-value="settings.default_page"
          class="opening-page"
          @change="
            (e) => {
              settings.default_page = e.option
            }
          "
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">Resource management</span>
        </h3>
      </div>

      <div class="adjacent-input">
        <label for="max-downloads">
          <span class="label__title">最大并发下载量</span>
          <span class="label__description">
            启动器同时可以下载的最大文件量。如果您的互联网连接较差，请将其设置为较低的值。（需要重新启动应用程序才能生效）
          </span>
        </label>
        <Slider
          id="max-downloads"
          v-model="settings.max_concurrent_downloads"
          :min="1"
          :max="10"
          :step="1"
        />
      </div>

      <div class="adjacent-input">
        <label for="max-writes">
          <span class="label__title">最大并发写入数</span>
          <span class="label__description">
            启动器一次可以写入磁盘的最大文件量。如果您经常遇到I/O错误，请将其设置为较低的值。（需要重新启动应用程序才能生效）
          </span>
        </label>
        <Slider
          id="max-writes"
          v-model="settings.max_concurrent_writes"
          :min="1"
          :max="50"
          :step="1"
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">Privacy</span>
        </h3>
      </div>
      <div class="adjacent-input">
        <label for="opt-out-analytics">
          <span class="label__title">Telemetry</span>
          <span class="label__description">
            Modrinth收集匿名分析和使用数据，以改善我们的用户体验并定制您的体验。禁用此选项后，您将选择退出，您的数据将不再被收集。
          </span>
        </label>
        <Toggle
          id="opt-out-analytics"
          :model-value="settings.telemetry"
          :checked="settings.telemetry"
          @update:model-value="
            (e) => {
              settings.telemetry = e
            }
          "
        />
      </div>
      <div class="adjacent-input">
        <label for="disable-discord-rpc">
          <span class="label__title">Discord RPC</span>
          <span class="label__description">
            管理Discord Rich Presence集成。禁用此选项将导致“Modrinth”不再显示为您在Discord个人资料中使用的游戏或应用程序。这不会禁用任何特定于实例的Discord Rich Presence集成，例如由mods添加的集成。（需要重新启动应用程序才能生效）
          </span>
        </label>
        <Toggle
          id="disable-discord-rpc"
          v-model="settings.discord_rpc"
          :checked="settings.discord_rpc"
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">Java设置</span>
        </h3>
      </div>
      <template v-for="version in [21, 17, 8]">
        <label :for="'java-' + version">
          <span class="label__title">Java {{ version }} 位置</span>
        </label>
        <JavaSelector
          :id="'java-selector-' + version"
          v-model="javaVersions[version]"
          :version="version"
          @update:model-value="updateJavaVersion"
        />
      </template>
      <hr class="card-divider" />
      <label for="java-args">
        <span class="label__title">Java 参数</span>
      </label>
      <input
        id="java-args"
        v-model="settings.launchArgs"
        autocomplete="off"
        type="text"
        class="installation-input"
        placeholder="输入Java参数..."
      />
      <label for="env-vars">
        <span class="label__title">环境变量</span>
      </label>
      <input
        id="env-vars"
        v-model="settings.envVars"
        autocomplete="off"
        type="text"
        class="installation-input"
        placeholder="输入环境变量..."
      />
      <hr class="card-divider" />
      <div class="adjacent-input">
        <label for="max-memory">
          <span class="label__title">Java 内存</span>
          <span class="label__description">
            运行时分配给每个实例的内存。
          </span>
        </label>
        <Slider
          id="max-memory"
          v-model="settings.memory.maximum"
          :min="8"
          :max="maxMemory"
          :step="64"
          unit="mb"
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">Hooks</span>
        </h3>
      </div>
      <div class="adjacent-input">
        <label for="pre-launch">
          <span class="label__title">运行前</span>
          <span class="label__description"> 在实例启动之前运行。 </span>
        </label>
        <input
          id="pre-launch"
          v-model="settings.hooks.pre_launch"
          autocomplete="off"
          type="text"
          placeholder="Enter pre-launch command..."
        />
      </div>
      <div class="adjacent-input">
        <label for="wrapper">
          <span class="label__title">Wrapper</span>
          <span class="label__description"> 用于启动Minecraft的Wrapper命令。 </span>
        </label>
        <input
          id="wrapper"
          v-model="settings.hooks.wrapper"
          autocomplete="off"
          type="text"
          placeholder="Enter wrapper command..."
        />
      </div>
      <div class="adjacent-input">
        <label for="post-exit">
          <span class="label__title">退出后</span>
          <span class="label__description"> 关闭游戏后执行。 </span>
        </label>
        <input
          id="post-exit"
          v-model="settings.hooks.post_exit"
          autocomplete="off"
          type="text"
          placeholder="Enter post-exit command..."
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">窗口大小</span>
        </h3>
      </div>
      <div class="adjacent-input">
        <label for="fullscreen">
          <span class="label__title">全屏</span>
          <span class="label__description">
            覆盖options.txt文件，以便在启动时以全屏模式启动。
          </span>
        </label>
        <Toggle
          id="fullscreen"
          :model-value="settings.force_fullscreen"
          :checked="settings.force_fullscreen"
          @update:model-value="
            (e) => {
              settings.force_fullscreen = e
            }
          "
        />
      </div>
      <div class="adjacent-input">
        <label for="width">
          <span class="label__title">宽</span>
          <span class="label__description"> 启动时游戏窗口的宽度。 </span>
        </label>
        <input
          id="width"
          v-model="settings.game_resolution[0]"
          :disabled="settings.force_fullscreen"
          autocomplete="off"
          type="number"
          placeholder="Enter width..."
        />
      </div>
      <div class="adjacent-input">
        <label for="height">
          <span class="label__title">高</span>
          <span class="label__description"> 启动时游戏窗口的高度。 </span>
        </label>
        <input
          id="height"
          v-model="settings.game_resolution[1]"
          :disabled="settings.force_fullscreen"
          autocomplete="off"
          type="number"
          class="input"
          placeholder="Enter height..."
        />
      </div>
    </Card>
    <Card>
      <div class="label">
        <h3>
          <span class="label__title size-card-header">关于</span>
        </h3>
      </div>
      <div>
        <label>
          <span class="label__title">App 版本</span>
          <span class="label__description">Modrinth App v{{ version }} </span>
        </label>
      </div>
    </Card>
  </div>
</template>

<style lang="scss" scoped>
.settings-page {
  margin: 1rem;
}

.installation-input {
  width: 100% !important;
  flex-grow: 1;
}

.theme-dropdown {
  text-transform: capitalize;
}

.card-divider {
  margin: 1rem 0;
}

.app-directory {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: var(--gap-sm);

  .iconified-input {
    flex-grow: 1;

    input {
      flex-basis: auto;
    }
  }
}
</style>
