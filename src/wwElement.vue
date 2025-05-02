<template>
  <div class="ww-datagrid" :class="{ editing: isEditing }" :style="cssVars">
    <ag-grid-vue :rowData="computedRowData" :columnDefs="columnDefs" :defaultColDef="defaultColDef"
      :domLayout="content.layout === 'auto' ? 'autoHeight' : 'normal'" :style="style" :rowSelection="rowSelection"
      :selection-column-def="{ pinned: true }" :theme="theme" :getRowId="getRowId" :pagination="content.pagination"
      :paginationPageSize="content.paginationPageSize || 10" :paginationPageSizeSelector="false"
      :suppressMovableColumns="!content.movableColumns" :columnHoverHighlight="content.columnHoverHighlight"
      :pinnedBottomRowData="totalRowData" :localeText="AG_GRID_LOCALE_BR" @grid-ready="onGridReady"
      @row-selected="onRowSelected" @selection-changed="onSelectionChanged" @cell-value-changed="onCellValueChanged"
      @row-double-clicked="onRowDoubleClicked">
    </ag-grid-vue>
  </div>
</template>

<script>
import { shallowRef } from "vue";
import { AgGridVue } from "ag-grid-vue3";
import {
  AllCommunityModule,
  ModuleRegistry,
  themeQuartz,
} from "ag-grid-community";
import { AG_GRID_LOCALE_BR } from "./pt-BR.ts";
import ActionCellRenderer from "./components/ActionCellRenderer.vue";
import ImageCellRenderer from "./components/ImageCellRenderer.vue";
import WewebCellRenderer from "./components/WewebCellRenderer.vue";
import ComparativeCellRenderer from "./components/ComparativeCellRenderer.vue";
import SkeletonCellRenderer from "./components/SkeletonCellRenderer.vue";

// TODO: maybe register less modules
// TODO: maybe register modules per grid instead of globally
ModuleRegistry.registerModules([AllCommunityModule]);

export default {
  components: {
    AgGridVue,
    ActionCellRenderer,
    ImageCellRenderer,
    WewebCellRenderer,
    ComparativeCellRenderer,
    SkeletonCellRenderer
  },
  props: {
    content: {
      type: Object,
      required: true,
    },
    uid: {
      type: String,
      required: true,
    },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ["trigger-event", "update:content:effect"],
  setup(props, ctx) {
    const { resolveMappingFormula } = wwLib.wwFormula.useFormula();

    const gridApi = shallowRef(null);
    const { value: selectedRows, setValue: setSelectedRows } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "selectedRows",
        type: "array",
        defaultValue: [],
        readonly: true,
      });

    const { value: dataTable, setValue: setDataTable } =
    wwLib.wwVariable.useComponentVariable({
    uid: props.uid,
    name: "dataTable",
    type: "array",
    defaultValue: [],
    readonly: true,
  });

    const onGridReady = (params) => {
  gridApi.value = params.api;
  
  // Adicionar listeners para vários eventos que podem mudar os dados exibidos
  gridApi.value.addEventListener('sortChanged', updateDisplayedData);
  gridApi.value.addEventListener('filterChanged', updateDisplayedData);
  gridApi.value.addEventListener('paginationChanged', updateDisplayedData);

  updateColumnsVisibility();
      
  // Atualizar os dados inicialmente
  updateDisplayedData();
};

    // Adicione esta nova função para atualizar os dados exibidos
const updateDisplayedData = () => {
  if (!gridApi.value) return;
  
  let displayedData = [];
  
  if (props.content.pagination) {
    // Se tiver paginação, pegamos apenas os dados da página atual
    const currentPage = gridApi.value.paginationGetCurrentPage();
    const pageSize = props.content.paginationPageSize || 10;
    const startRow = currentPage * pageSize;
    const endRow = startRow + pageSize;
    
    let rowIndex = 0;
    gridApi.value.forEachNodeAfterFilterAndSort((node) => {
      if (rowIndex >= startRow && rowIndex < endRow) {
        displayedData.push(node.data);
      }
      rowIndex++;
    });
  } else {
    // Se não tiver paginação, pegamos todos os dados filtrados e ordenados
    gridApi.value.forEachNodeAfterFilterAndSort((node) => {
      displayedData.push(node.data);
    });
  }
  
  // Removemos linhas de skeleton
  displayedData = displayedData.filter(row => !row?._isSkeletonRow);
  
  // Atualizamos a variável
  setDataTable(displayedData);
};
    
    const onRowSelected = (event) => {
      const name = event.node.isSelected() ? "rowSelected" : "rowDeselected";
      ctx.emit("trigger-event", {
        name,
        event: { row: event.data },
      });
    };

    const onSelectionChanged = (event) => {
      if (!gridApi.value) return;
      const selected = gridApi.value.getSelectedRows() || [];
      setSelectedRows(selected);
    };

    /* wwEditor:start */
    const { createElement } = wwLib.useCreateElement();
    /* wwEditor:end */

    return {
      resolveMappingFormula,
      onGridReady,
      onRowSelected,
      onSelectionChanged,
      gridApi,
      AG_GRID_LOCALE_BR,
      setDataTable,        // Adicionando setDataTable
      updateDisplayedData, // Adicionando updateDisplayedData
      /* wwEditor:start */
      createElement,
      /* wwEditor:end */
    };
  },
  computed: {
    rowData() {
      const data = wwLib.wwUtils.getDataFromCollection(this.content.rowData);
      return Array.isArray(data) ? data ?? [] : [];
    },
    computedRowData() {
      if (this.content.loading) {
        // Gerar 15 linhas de skeleton
        const skeletonRows = Array(15).fill().map((_, index) => {
          const row = {
            _isSkeletonRow: true,
            _skeletonId: `skeleton-${index}` // Propriedade para uso no idFormula caso necessário
          };
          // Adicionar um campo para cada coluna
          this.content.columns.forEach(column => {
            if (column.field) {
              row[column.field] = null;
            }
          });
          return row;
        });
        return skeletonRows;
      }

      // Retornar os dados originais quando não estiver carregando
      return this.rowData;
    },
    totalRowData() {
      if (!this.content.showTotalRow) return null;

      // Create an empty object for the total row
      const totalRow = {};

      // Add the totals from each column's totalValue field
      this.content.columns.forEach(column => {
        if (column.totalValue) {
          totalRow[column.field] = this.resolveMappingFormula(column.totalValue, null);
        }
      });

      return [totalRow];
    },
    defaultColDef() {
      return {
        editable: false,
        resizable: this.content.resizableColumns,
      };
    },
    columnDefs() {
    return this.content.columns
    .map((col) => {
        const minWidth =
          !col.minWidth || col.minWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.minWidth)?.[0];
        const maxWidth =
          !col.maxWidth || col.maxWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.maxWidth)?.[0];
        const width =
          !col.width || col.width === "auto" || col.widthAlgo === "flex"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.width)?.[0];
        const flex = col.widthAlgo === "flex" ? col.flex ?? 1 : null;
        const commonProperties = {
          minWidth,
          maxWidth,
          pinned: col.pinned === "none" ? false : col.pinned,
          width,
          flex,
          hide: col.display === false, 
        };
        if (this.content.loading) {
          return {
            ...commonProperties,
            headerName: col.headerName,
            field: col.field,
            cellRenderer: "SkeletonCellRenderer",
            sortable: false,
            filter: false,
          };
        }
        switch (col.cellDataType) {
          case "action": {
            return {
              ...commonProperties,
              headerName: col.headerName,
              cellRenderer: "ActionCellRenderer",
              cellRendererParams: {
                name: col.actionName,
                label: col.actionLabel,
                trigger: this.onActionTrigger,
                withFont: !!this.content.actionFont,
              },
              sortable: false,
              filter: false,
            };
          }
          case "custom":
            return {
              ...commonProperties,
              headerName: col.headerName,
              field: col.field,
              cellRenderer: "WewebCellRenderer",
              cellRendererParams: {
                containerId: col.containerId,
              },
              sortable: col.sortable,
              filter: col.filter,
            };
          case "image": {
            return {
              ...commonProperties,
              headerName: col.headerName,
              field: col.field,
              cellRenderer: "ImageCellRenderer",
              cellRendererParams: {
                width: col.imageWidth,
                height: col.imageHeight,
              },
            };
          }
          // No caso default
          default: {
            const result = {
              ...commonProperties,
              headerName: col.headerName,
              field: col.field,
              sortable: col.sortable,
              filter: col.filter,
              editable: col.editable,
            };

            if (col.comparative) {
              result.cellRenderer = "ComparativeCellRenderer";
            }

            // Lógica para formatação de números
            if (col.cellDataType === "formatted-number") {
              result.valueFormatter = (params) => {
                if (params.value === null || params.value === undefined) return '';
                return this.formatNumber(params.value);
              };
            }
            // Lógica para formatação de moeda (R$)
            else if (col.cellDataType === "currency") {
              result.valueFormatter = (params) => {
                if (params.value === null || params.value === undefined) return '';
                return `R$ ${this.formatNumber(params.value)}`;
              };
            }
            // Lógica para formatação de porcentagem (%)
            else if (col.cellDataType === "percentage") {
              result.valueFormatter = (params) => {
                if (params.value === null || params.value === undefined) return '';
                return `${this.formatNumber(params.value)}%`;
              };
            }
            // Lógica existente para customização de label
            else if (col.useCustomLabel) {
              result.valueFormatter = (params) => {
                return this.resolveMappingFormula(
                  col.displayLabelFormula,
                  params.value
                );
              };
            }

            return result;
          }
        }
      });
    },
    rowSelection() {
      if (this.content.rowSelection === "multiple") {
        return { mode: "multiRow", checkboxes: true };
      } else if (this.content.rowSelection === "single") {
        return { mode: "singleRow", checkboxes: true };
      } else {
        return {
          mode: "singleRow",
          checkboxes: false,
          isRowSelectable: () => false,
        };
      }
    },
    style() {
      if (this.content.layout === "auto") return {};
      return {
        height: this.content.height || "400px",
      };
    },
    cssVars() {
      return {
        "--ww-data-grid_action-backgroundColor":
          this.content.actionBackgroundColor,
        "--ww-data-grid_action-color": this.content.actionColor,
        "--ww-data-grid_action-padding": this.content.actionPadding,
        "--ww-data-grid_action-border": this.content.actionBorder,
        "--ww-data-grid_action-borderRadius": this.content.actionBorderRadius,
        ...(this.content.actionFont
          ? { "--ww-data-grid_action-font": this.content.actionFont }
          : {
            "--ww-data-grid_action-fontSize": this.content.actionFontSize,
            "--ww-data-grid_action-fontFamily": this.content.actionFontFamily,
            "--ww-data-grid_action-fontWeight": this.content.actionFontWeight,
            "--ww-data-grid_action-fontStyle": this.content.actionFontStyle,
            "--ww-data-grid_action-lineHeight": this.content.actionLineHeight,
          }),
      };
    },
    theme() {
      return themeQuartz.withParams({
        headerBackgroundColor: this.content.headerBackgroundColor,
        headerTextColor: this.content.headerTextColor,
        headerFontSize: this.content.headerFontSize,
        headerFontFamily: this.content.headerFontFamily,
        headerFontWeight: this.content.headerFontWeight,
        borderColor: this.content.borderColor,
        cellTextColor: this.content.cellColor,
        cellFontFamily: this.content.cellFontFamily,
        dataFontSize: this.content.cellFontSize,
        oddRowBackgroundColor: this.content.rowAlternateColor,
        backgroundColor: this.content.rowBackgroundColor,
        rowHoverColor: this.content.rowHoverColor,
        selectedRowBackgroundColor: this.content.selectedRowBackgroundColor,
        rowVerticalPaddingScale: this.content.rowVerticalPaddingScale || 1,
        menuBackgroundColor: this.content.menuBackgroundColor,
        menuTextColor: this.content.menuTextColor,
        columnHoverColor: this.content.columnHoverColor,
        foregroundColor: this.content.textColor,
        borderRadius: 6,
        wrapperBorderRadius: 6,
      });
    },
    isEditing() {
      /* wwEditor:start */
      return (
        this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION
      );
      /* wwEditor:end */
      // eslint-disable-next-line no-unreachable
      return false;
    },
  },
  methods: {
    getRowId(params) {
      if (params.data?._isSkeletonRow) {
        return params.data._skeletonId;
      }
      return this.resolveMappingFormula(this.content.idFormula, params.data);
    },
    onActionTrigger(event) {
      this.$emit("trigger-event", {
        name: "action",
        event,
      });
    },
    onCellValueChanged(event) {
      this.$emit("trigger-event", {
        name: "cellValueChanged",
        event: {
          oldValue: event.oldValue,
          newValue: event.newValue,
          columnId: event.column.getColId(),
          row: event.data,
        },
      });
    },
    updateColumnsVisibility() {
  if (!this.gridApi?.value) return;
  
  // Para cada coluna na configuração
  this.content.columns.forEach(column => {
    const columnState = {
      colId: column.field,
      hide: column.display === false
    };
    
    // Atualiza o estado da coluna
    this.gridApi.value.columnModel.setColumnState([columnState]);
  });
  
  // Após atualizar a visibilidade, também atualizamos os dados exibidos
  this.updateDisplayedData();
},
    formatNumber(value) {
      // Garante que o valor é um número
      const num = Number(value);
      if (isNaN(num)) return value;

      // Formata o número com pontos para milhares e vírgula para decimais
      return num.toLocaleString('pt-BR', {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2
      });
    },
    // Novo método para tratar o double click na linha
    onRowDoubleClicked(event) {
      this.$emit("trigger-event", {
        name: "rowdoubleclicked",
        event: {
          row: event.data,
        },
      });
    },
    updateDisplayedData() {
    if (!this.gridApi?.value) return;
    
    let displayedData = [];
    
    if (this.content.pagination) {
      const currentPage = this.gridApi.value.paginationGetCurrentPage();
      const pageSize = this.content.paginationPageSize || 10;
      const startRow = currentPage * pageSize;
      const endRow = startRow + pageSize;
      
      let rowIndex = 0;
      this.gridApi.value.forEachNodeAfterFilterAndSort((node) => {
        if (rowIndex >= startRow && rowIndex < endRow) {
          displayedData.push(node.data);
        }
        rowIndex++;
      });
    } else {
      this.gridApi.value.forEachNodeAfterFilterAndSort((node) => {
        displayedData.push(node.data);
      });
    }
    
    // Removemos linhas de skeleton
    displayedData = displayedData.filter(row => !row?._isSkeletonRow);
    
    // Atualizamos a variável
    this.setDataTable(displayedData);
  },
    // Método para ajustar automaticamente o tamanho de todas as colunas com base no conteúdo
  sizeColumnsToFit() {
    const api = this.gridApi?.value || this.gridApi;
    
    if (api) {
      // Obtém todos os IDs de coluna
      const allColumnIds = [];
      api.getColumns().forEach((column) => {
        allColumnIds.push(column.getId());
      });
      
      // Aplica o autoSize em todas as colunas
      // Parâmetro false para não pular o cabeçalho no cálculo de largura
      api.autoSizeColumns(allColumnIds, false);
    }
  },
    /* wwEditor:start */
    generateColumns() {
      this.$emit("update:content", {
        columns: this.rowData?.[0]
          ? Object.keys(this.rowData[0]).map((key) => ({
            field: key,
            sortable: true,
            filter: true,
          }))
          : [],
      });
    },
    getOnActionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        actionName: "actionName",
        row: data[0],
        id: 0,
        index: 0,
        displayIndex: 0,
      };
    },
    getOnCellValueChangedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        oldValue: "oldValue",
        newValue: "newValue",
        columnId: "columnId",
        row: data[0],
      };
    },
    getSelectionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
      };
    },
    /* wwEditor:end */
  },
  /* wwEditor:start */
  watch: {
    columnDefs: {
      // TODO: also do data cleaning
      async handler() {
        if (this.wwEditorState?.boundProps?.columns) return;
        for (const col of this.content.columns) {
          // const column = this.gridApi.getColumn(col.field);
          // if (!column) continue;
          // if (col.pinned && !column.pinned) {
          //   this.gridApi.setColumnsPinned([col.field], col.pinned);
          // } else if ((!col.pinned || col.pinned === 'none') && column.pinned) {
          //   this.gridApi.setColumnsPinned([col.field], null);
          // }
        }
        this.gridApi.resetColumnState();

        if (this.wwEditorState.isACopy) return;

        // We assume there will only be one custom column each time
        const columnIndex = (this.content.columns || []).findIndex(
          (col) => col.cellDataType === "custom" && !col.containerId
        );
        if (columnIndex === -1) return;
        const newColumns = [...this.content.columns];
        let column = { ...newColumns[columnIndex] };
        column.containerId = await this.createElement("ww-flexbox", {
          _state: { name: `Cell ${column.headerName || column.field}` },
        });
        newColumns[columnIndex] = column;
        this.$emit("update:content:effect", { columns: newColumns });
      },
      deep: true,
    },
    computedRowData: {
    handler() {
      this.$nextTick(() => {
        if (this.gridApi?.value) {
          this.updateDisplayedData();
        }
      });
    },
    deep: true
  },
    'content.columns': {
  handler(newColumns, oldColumns) {
    // Verificamos se houve mudança nas propriedades de display
    const displayChanged = newColumns.some((newCol, index) => {
      const oldCol = oldColumns[index];
      return oldCol && newCol.display !== oldCol.display;
    });
    
    // Se mudou a visibilidade de alguma coluna, atualizamos
    if (displayChanged && this.gridApi?.value) {
      this.$nextTick(() => {
        this.updateColumnsVisibility();
      });
    }
  },
  deep: true
},
  },
  /* wwEditor:end */
};
</script>

<style lang="scss">
.ww-datagrid {
  position: relative;

  /* wwEditor:start */
  &.editing {
    &::before {
      content: "";
      position: absolute;
      inset: 0;
      display: block;
      pointer-events: initial;
      z-index: 10;
    }
  }

  /* wwEditor:end */
}

.ag-cell {
  font-weight: 600;
}

.skeleton-cell-container {
  display: flex;
  align-items: center;
  /* Centraliza verticalmente */
  height: 100%;
  /* Ocupa toda a altura da célula */
  width: 100%;
  /* Ocupa toda a largura da célula */
  padding: 8px 0;
  /* Espaçamento opcional para melhor aparência */
}

.skeleton-loader {
  height: 14px;
  background-color: #A19B9D4D;
  border-radius: 2px;
  width: 100%;
  animation: skeleton-wave 1.5s ease-in-out infinite;
  position: relative;
  overflow: hidden;
}

@keyframes skeleton-wave {
  0% {
    background-position: -200px 0;
  }

  100% {
    background-position: calc(200px + 100%) 0;
  }
}

.skeleton-loader::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  animation: skeleton-wave 1.5s ease-in-out 0.5s infinite;
}
</style>
