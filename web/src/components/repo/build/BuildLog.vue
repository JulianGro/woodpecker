<template>
  <div v-if="build" class="flex flex-col pt-10 md:pt-0">
    <div
      class="fixed top-0 left-0 w-full md:hidden flex px-4 py-2 bg-gray-600 dark:bg-dark-gray-800 text-gray-50"
      @click="$emit('update:proc-id', null)"
    >
      <span>{{ proc?.name }}</span>
      <Icon name="close" class="ml-auto" />
    </div>

    <div
      class="flex flex-grow flex-col bg-gray-300 dark:bg-dark-gray-700 md:m-2 md:mt-0 md:rounded-md overflow-hidden"
      @mouseover="showActions = true"
      @mouseleave="showActions = false"
    >
      <div v-show="showActions" class="absolute top-0 right-0 z-50 mt-2 mr-4 hidden md:flex">
        <Button
          v-if="proc?.end_time !== undefined"
          :is-loading="downloadInProgress"
          :title="$t('repo.build.actions.log_download')"
          start-icon="download"
          @click="download"
        />
      </div>

      <div v-show="loadedLogs" class="w-full flex-grow p-2">
        <div id="terminal" class="w-full h-full" />
      </div>

      <div class="m-auto text-xl text-color">
        <span v-if="proc?.error" class="text-red-400">{{ proc.error }}</span>
        <span v-else-if="proc?.state === 'skipped'" class="text-red-400">{{ $t('repo.build.actions.canceled') }}</span>
        <span v-else-if="!proc?.start_time">{{ $t('repo.build.step_not_started') }}</span>
        <div v-else-if="!loadedLogs">{{ $t('repo.build.loading') }}</div>
      </div>

      <div
        v-if="proc?.end_time !== undefined"
        :class="proc.exit_code == 0 ? 'dark:text-lime-400 text-lime-600' : 'dark:text-red-400 text-red-600'"
        class="w-full bg-gray-400 dark:bg-dark-gray-800 text-md p-4"
      >
        {{ $t('repo.build.exit_code', { exitCode: proc.exit_code }) }}
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import 'xterm/css/xterm.css';

import {
  computed,
  defineComponent,
  inject,
  nextTick,
  onBeforeUnmount,
  onMounted,
  PropType,
  Ref,
  ref,
  toRef,
  watch,
} from 'vue';
import { useI18n } from 'vue-i18n';
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import { WebLinksAddon } from 'xterm-addon-web-links';

import Button from '~/components/atomic/Button.vue';
import Icon from '~/components/atomic/Icon.vue';
import useApiClient from '~/compositions/useApiClient';
import { useDarkMode } from '~/compositions/useDarkMode';
import useNotifications from '~/compositions/useNotifications';
import { Build, Repo } from '~/lib/api/types';
import { findProc, isProcFinished, isProcRunning } from '~/utils/helpers';

export default defineComponent({
  name: 'BuildLog',

  components: { Icon, Button },

  props: {
    build: {
      type: Object as PropType<Build>,
      required: true,
    },

    // used by toRef
    // eslint-disable-next-line vue/no-unused-properties
    procId: {
      type: Number,
      required: true,
    },
  },

  emits: {
    // eslint-disable-next-line @typescript-eslint/no-unused-vars
    'update:proc-id': (procId: number | null) => true,
  },

  setup(props) {
    const notifications = useNotifications();
    const i18n = useI18n();
    const build = toRef(props, 'build');
    const procId = toRef(props, 'procId');
    const repo = inject<Ref<Repo>>('repo');
    const apiClient = useApiClient();

    const loadedProcSlug = ref<string>();
    const procSlug = computed(() => `${repo?.value.owner} - ${repo?.value.name} - ${build.value.id} - ${procId.value}`);
    const proc = computed(() => build.value && findProc(build.value.procs || [], procId.value));
    const stream = ref<EventSource>();
    const term = ref(
      new Terminal({
        convertEol: true,
        disableStdin: true,
        theme: {
          cursor: 'transparent',
        },
      }),
    );
    const fitAddon = ref(new FitAddon());
    const loadedLogs = ref(true);
    const autoScroll = ref(true); // TODO
    const showActions = ref(false);
    const downloadInProgress = ref(false);

    async function download() {
      if (!repo?.value || !build.value || !proc.value) {
        throw new Error('The reposiotry, build or proc was undefined');
      }
      let logs;
      try {
        downloadInProgress.value = true;
        logs = await apiClient.getLogs(repo.value.owner, repo.value.name, build.value.number, proc.value.pid);
      } catch (e) {
        notifications.notifyError(e, i18n.t('repo.build.log_download_error'));
        return;
      } finally {
        downloadInProgress.value = false;
      }
      const fileURL = window.URL.createObjectURL(
        new Blob([logs.map((line) => line.out).join('')], {
          type: 'text/plain',
        }),
      );
      const fileLink = document.createElement('a');

      fileLink.href = fileURL;
      fileLink.setAttribute(
        'download',
        `${repo.value.owner}-${repo.value.name}-${build.value.number}-${proc.value.name}.log`,
      );
      document.body.appendChild(fileLink);

      fileLink.click();
      document.body.removeChild(fileLink);
      window.URL.revokeObjectURL(fileURL);
    }

    async function loadLogs() {
      if (loadedProcSlug.value === procSlug.value) {
        return;
      }
      loadedProcSlug.value = procSlug.value;
      loadedLogs.value = false;
      term.value.reset();
      term.value.write('\x1b[?25l');

      if (!repo) {
        throw new Error('Unexpected: "repo" should be provided at this place');
      }

      if (stream.value) {
        stream.value.close();
      }

      // we do not have logs for skipped jobs
      if (
        !repo.value ||
        !build.value ||
        !proc.value ||
        proc.value.state === 'skipped' ||
        proc.value.state === 'killed'
      ) {
        return;
      }

      if (isProcFinished(proc.value)) {
        const logs = await apiClient.getLogs(repo.value.owner, repo.value.name, build.value.number, proc.value.pid);
        term.value.write(
          logs
            .slice(Math.max(logs.length, 0) - 300, logs.length) // TODO: think about way to support lazy-loading more than last 300 logs (#776)
            .map((line) => `${(line.pos || 0).toString().padEnd(logs.length.toString().length)}  ${line.out}`)
            .join(''),
        );
        loadedLogs.value = true;
      }

      if (isProcRunning(proc.value)) {
        // load stream of parent process (which receives all child processes logs)
        // TODO: change stream to only send data of single child process
        stream.value = apiClient.streamLogs(
          repo.value.owner,
          repo.value.name,
          build.value.number,
          proc.value.ppid,
          (l) => {
            loadedLogs.value = true;
            term.value.write(l.out, () => {
              if (autoScroll.value) {
                term.value.scrollToBottom();
              }
            });
          },
        );
      }
    }

    function resize() {
      fitAddon.value.fit();
    }

    const unmounted = ref(false);
    onMounted(async () => {
      term.value.loadAddon(fitAddon.value);
      term.value.loadAddon(new WebLinksAddon());

      await nextTick(() => {
        if (unmounted.value) {
          // need to check if unmounted already because we are async here
          return;
        }
        const element = document.getElementById('terminal');
        if (element === null) {
          throw new Error('Unexpected: "terminal" should be provided at this place');
        }
        term.value.open(element);
        fitAddon.value.fit();

        window.addEventListener('resize', resize);
      });

      loadLogs();
    });

    watch(procSlug, () => {
      loadLogs();
    });

    const { darkMode } = useDarkMode();
    watch(
      darkMode,
      () => {
        if (darkMode.value) {
          term.value.options = {
            theme: {
              background: '#303440', // dark-gray-700
              foreground: '#d3d3d3', // gray-...
            },
          };
        } else {
          term.value.options = {
            theme: {
              background: 'rgb(209,213,219)', // gray-300
              foreground: '#000',
              selection: '#000',
            },
          };
        }
      },
      { immediate: true },
    );

    onBeforeUnmount(() => {
      unmounted.value = true;
      if (stream.value) {
        stream.value.close();
      }
      const element = document.getElementById('terminal');
      if (element !== null) {
        // Clean up any custom DOM added in onMounted above
        element.innerHTML = '';
      }
      window.removeEventListener('resize', resize);
    });

    return { proc, loadedLogs, showActions, download, downloadInProgress };
  },
});
</script>
