<script context="module" lang="ts">
  const getCurrentEditor = () => getActiveComponent('JslDataGridCore');

  registerCommand({
    id: 'jslTableGrid.export',
    category: 'Data grid',
    name: 'Export',
    keyText: 'Ctrl+E',
    testEnabled: () => getCurrentEditor() != null,
    onClick: () => getCurrentEditor().exportGrid(),
  });

  async function loadDataPage(props, offset, limit) {
    const { jslid, display } = props;

    const response = await axiosInstance.post('jsldata/get-rows', {
      jslid,
      offset,
      limit,
      filters: display ? display.compileFilters() : null,
    });

    return response.data;
  }

  function dataPageAvailable(props) {
    return true;
  }

  async function loadRowCount(props) {
    const { jslid } = props;

    const response = await axiosInstance.request({
      url: 'jsldata/get-stats',
      method: 'get',
      params: {
        jslid,
      },
    });
    return response.data.rowCount;
  }
</script>

<script lang="ts">
  import _ from 'lodash';
  import registerCommand from '../commands/registerCommand';
  import ImportExportModal from '../modals/ImportExportModal.svelte';
  import { showModal } from '../modals/modalTools';

  import axiosInstance from '../utility/axiosInstance';
  import { registerMenu } from '../utility/contextMenu';
  import createActivator, { getActiveComponent } from '../utility/createActivator';
  import socket from '../utility/socket';
  import useEffect from '../utility/useEffect';

  import LoadingDataGridCore from './LoadingDataGridCore.svelte';
  import RowsArrayGrider from './RowsArrayGrider';

  export let jslid;

  export const activator = createActivator('JslDataGridCore', false);

  export let loadedRows = [];
  let domGrid;

  let changeIndex = 0;
  let rowCountLoaded = null;

  const throttleLoadNext = _.throttle(() => domGrid.resetLoadedAll(), 500);

  const handleJslDataStats = stats => {
    if (stats.changeIndex < changeIndex) return;
    changeIndex = stats.changeIndex;
    rowCountLoaded = stats.rowCount;
    throttleLoadNext();
  };

  $: effect = useEffect(() => onJslId(jslid));
  function onJslId(jslidVal) {
    if (jslidVal) {
      socket.on(`jsldata-stats-${jslidVal}`, handleJslDataStats);
      return () => {
        socket.off(`jsldata-stats-${jslidVal}`, handleJslDataStats);
      };
    }
  }
  $: $effect;

  $: grider = new RowsArrayGrider(loadedRows);

  export function exportGrid() {
    const initialValues = {} as any;
    const archiveMatch = jslid.match(/^archive:\/\/([^/]+)\/(.*)$/);
    if (archiveMatch) {
      initialValues.sourceStorageType = 'archive';
      initialValues.sourceArchiveFolder = archiveMatch[1];
      initialValues.sourceList = [archiveMatch[2]];
    } else {
      initialValues.sourceStorageType = 'jsldata';
      initialValues.sourceJslId = jslid;
      initialValues.sourceList = ['query-data'];
    }
    showModal(ImportExportModal, { initialValues });
  }

  registerMenu({ command: 'jslTableGrid.export', tag: 'export' });
</script>

<LoadingDataGridCore
  bind:this={domGrid}
  {...$$props}
  bind:loadedRows
  {loadDataPage}
  {dataPageAvailable}
  {loadRowCount}
  {grider}
  {rowCountLoaded}
/>
