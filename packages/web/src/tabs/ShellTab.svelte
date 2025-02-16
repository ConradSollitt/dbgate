<script lang="ts" context="module">
  const getCurrentEditor = () => getActiveComponent('ShellTab');

  registerFileCommands({
    idPrefix: 'shell',
    category: 'Shell',
    getCurrentEditor,
    folder: 'shell',
    format: 'text',
    fileExtension: 'js',

    execute: true,
    toggleComment: true,
    findReplace: true,
  });

  const configRegex = /\s*\/\/\s*@ImportExportConfigurator\s*\n\s*\/\/\s*(\{[^\n]+\})\n/;
  const requireRegex = /\s*(\/\/\s*@require\s+[^\n]+)\n/g;
  const initRegex = /([^\n]+\/\/\s*@init)/g;
</script>

<script lang="ts">
  import { getContext  } from 'svelte';

  import invalidateCommands from '../commands/invalidateCommands';
  import { registerFileCommands } from '../commands/stdCommands';

  import VerticalSplitter from '../elements/VerticalSplitter.svelte';
  import AceEditor from '../query/AceEditor.svelte';
  import RunnerOutputPane from '../query/RunnerOutputPane.svelte';
  import useEditorData from '../query/useEditorData';
  import axiosInstance from '../utility/axiosInstance';
  import { changeTab } from '../utility/common';
  import createActivator, { getActiveComponent } from '../utility/createActivator';
  import socket from '../utility/socket';
  import useEffect from '../utility/useEffect';
  import useTimerLabel from '../utility/useTimerLabel';

  export let tabid;

  const tabVisible: any = getContext('tabVisible');
  const timerLabel = useTimerLabel();

  let runnerId;

  export const activator = createActivator('ShellTab', false);

  let busy = false;
  let executeNumber = 0;

  let domEditor;

  // const status = writable({
  //   busy,
  //   canKill: false,
  // });

  // $: {
  //   status.set({
  //     busy,
  //     canKill: busy,
  //   });
  // }

  $: {
    busy;
    invalidateCommands();
  }

  $: if ($tabVisible && domEditor) {
    domEditor?.getEditor()?.focus();
  }

  $: {
    changeTab(tabid, tab => ({ ...tab, busy }));
  }

  $: effect = useEffect(() => registerRunnerDone(runnerId));
  $: $effect;

  function registerRunnerDone(rid) {
    if (rid) {
      socket.on(`runner-done-${rid}`, handleRunnerDone);
      return () => {
        socket.off(`runner-done-${rid}`, handleRunnerDone);
      };
    } else {
      return () => {};
    }
  }

  function handleRunnerDone() {
    busy = false;
    timerLabel.stop();
  }

  // export function getStatus() {
  //   return status;
  // }

  export function getData() {
    return $editorState.value || '';
  }

  export function toggleComment() {
    domEditor.getEditor().execCommand('togglecomment');
  }

  export function find() {
    domEditor.getEditor().execCommand('find');
  }

  export function replace() {
    domEditor.getEditor().execCommand('replace');
  }

  export function isBusy() {
    return busy;
  }

  export async function execute() {
    if (busy) return;
    executeNumber += 1;
    const selectedText = domEditor.getEditor().getSelectedText();
    const editorText = $editorValue;

    let runid = runnerId;
    const resp = await axiosInstance.post('runners/start', {
      script: selectedText
        ? [...(editorText || '').matchAll(requireRegex)].map(x => `${x[1]}\n`).join('') +
          [...(editorText || '').matchAll(initRegex)].map(x => `${x[1]}\n`).join('') +
          selectedText
        : editorText,
    });
    runid = resp.data.runid;
    runnerId = runid;
    busy = true;
    timerLabel.start();
  }

  export function kill() {
    axiosInstance.post('runners/cancel', {
      runid: runnerId,
    });
    timerLabel.stop();
  }

  const { editorState, editorValue, setEditorData } = useEditorData({ tabid });

  function createMenu() {
    return [
      { command: 'shell.execute' },
      { command: 'shell.kill' },
      { divider: true },
      { command: 'shell.toggleComment' },
      { divider: true },
      { command: 'shell.save' },
      { command: 'shell.saveAs' },
      { divider: true },
      { command: 'shell.find' },
      { command: 'shell.replace' },
    ];
  }
</script>

<VerticalSplitter>
  <svelte:fragment slot="1">
    <AceEditor
      value={$editorState.value || ''}
      menu={createMenu()}
      on:input={e => setEditorData(e.detail)}
      on:focus={() => {
        activator.activate();
        invalidateCommands();
      }}
      bind:this={domEditor}
      mode="javascript"
    />
  </svelte:fragment>
  <svelte:fragment slot="2">
    <RunnerOutputPane {runnerId} {executeNumber} />
  </svelte:fragment>
</VerticalSplitter>
