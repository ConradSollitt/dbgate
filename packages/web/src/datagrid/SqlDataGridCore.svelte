<script context="module" lang="ts">
  const getCurrentEditor = () => getActiveComponent('SqlDataGridCore');

  registerCommand({
    id: 'sqlDataGrid.openActiveChart',
    category: 'Data grid',
    name: 'Open active chart',
    testEnabled: () => getCurrentEditor() != null,
    onClick: () => getCurrentEditor().openActiveChart(),
  });

  registerCommand({
    id: 'sqlDataGrid.openQuery',
    category: 'Data grid',
    name: 'Open query',
    testEnabled: () => getCurrentEditor() != null,
    onClick: () => getCurrentEditor().openQuery(),
  });

  registerCommand({
    id: 'sqlDataGrid.export',
    category: 'Data grid',
    name: 'Export',
    keyText: 'Ctrl+E',
    testEnabled: () => getCurrentEditor() != null,
    onClick: () => getCurrentEditor().exportGrid(),
  });

  async function loadDataPage(props, offset, limit) {
    const { display, conid, database } = props;

    const sql = display.getPageQuery(offset, limit);

    const response = await axiosInstance.request({
      url: 'database-connections/query-data',
      method: 'post',
      params: {
        conid,
        database,
      },
      data: { sql },
    });

    if (response.data.errorMessage) return response.data;
    return response.data.rows;
  }

  function dataPageAvailable(props) {
    const { display } = props;
    const sql = display.getPageQuery(0, 1);
    return !!sql;
  }

  async function loadRowCount(props) {
    const { display, conid, database } = props;

    const sql = display.getCountQuery();

    const response = await axiosInstance.request({
      url: 'database-connections/query-data',
      method: 'post',
      params: {
        conid,
        database,
      },
      data: { sql },
    });

    return parseInt(response.data.rows[0].count);
  }
</script>

<script lang="ts">
  import { changeSetToSql, createChangeSet } from 'dbgate-datalib';
  import { scriptToSql } from 'dbgate-sqltree';
  import registerCommand from '../commands/registerCommand';
  import ConfirmSqlModal from '../modals/ConfirmSqlModal.svelte';
  import ErrorMessageModal from '../modals/ErrorMessageModal.svelte';
  import ImportExportModal from '../modals/ImportExportModal.svelte';
  import { showModal } from '../modals/modalTools';

  import axiosInstance from '../utility/axiosInstance';
  import { registerMenu } from '../utility/contextMenu';
  import createActivator, { getActiveComponent } from '../utility/createActivator';
  import openNewTab from '../utility/openNewTab';
  import ChangeSetGrider from './ChangeSetGrider';

  import LoadingDataGridCore from './LoadingDataGridCore.svelte';

  export let conid;
  export let display;
  export let database;
  export let schemaName;
  export let pureName;
  export let config;
  export let changeSetState;
  export let dispatchChangeSet;

  export let macroPreview;
  export let macroValues;
  export let selectedCellsPublished = () => [];

  // export let onChangeGrider = undefined;

  export const activator = createActivator('SqlDataGridCore', false);

  let loadedRows = [];

  // $: console.log('loadedRows BIND', loadedRows);
  $: grider = new ChangeSetGrider(
    loadedRows,
    changeSetState,
    dispatchChangeSet,
    display,
    macroPreview,
    macroValues,
    selectedCellsPublished()
  );
  // $: console.log('GRIDER', grider);
  // $: if (onChangeGrider) onChangeGrider(grider);

  export function exportGrid() {
    const initialValues: any = {};
    initialValues.sourceStorageType = 'query';
    initialValues.sourceConnectionId = conid;
    initialValues.sourceDatabaseName = database;
    initialValues.sourceSql = display.getExportQuery();
    initialValues.sourceList = display.baseTable ? [display.baseTable.pureName] : [];
    showModal(ImportExportModal, { initialValues });
  }

  export function openQuery() {
    openNewTab(
      {
        title: 'Query #',
        icon: 'img sql-file',
        tabComponent: 'QueryTab',
        props: {
          schemaName: display.baseTable.schemaName,
          pureName: display.baseTable.pureName,
          conid,
          database,
        },
      },
      {
        editor: display.getExportQuery(),
      }
    );
  }

  export function openActiveChart() {
    openNewTab(
      {
        title: 'Chart #',
        icon: 'img chart',
        tabComponent: 'ChartTab',
        props: {
          conid,
          database,
        },
      },
      {
        editor: {
          config: { chartType: 'bar' },
          sql: display.getExportQuery(select => {
            select.orderBy = null;
          }),
        },
      }
    );
  }

  registerMenu(
    { command: 'sqlDataGrid.openActiveChart', tag: 'chart' },
    { command: 'sqlDataGrid.openQuery', tag: 'export' },
    { command: 'sqlDataGrid.export', tag: 'export' }
  );
</script>

<LoadingDataGridCore
  {...$$props}
  {loadDataPage}
  {dataPageAvailable}
  {loadRowCount}
  bind:loadedRows
  bind:selectedCellsPublished
  frameSelection={!!macroPreview}
  {grider}
/>
