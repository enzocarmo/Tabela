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
    SkeletonCellRenderer,
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
      // Armazenamos a referência para a API da grade
      gridApi.value = params.api;

      // Adicionar listeners para eventos usando a função direta
      const updateDisplayedDataFn = () => {
        updateDisplayedData();
      };

      params.api.addEventListener('sortChanged', updateDisplayedDataFn);
      params.api.addEventListener('filterChanged', updateDisplayedDataFn);
      params.api.addEventListener('paginationChanged', updateDisplayedDataFn);

      // Garantir que as configurações de grupo funcionem corretamente
      if (props.content.parentColumns && props.content.parentColumns.length > 0) {
        // Habilitamos explicitamente recursos importantes
        params.api.setSuppressRowDrag(false);
        params.columnApi.setColumnsResizable(props.content.resizableColumns);
        params.api.setSuppressMovableColumns(!props.content.movableColumns);

        // Configuração crítica para garantir que os eventos funcionem
        params.api.refreshHeader();
      }

      // Aplica a configuração das colunas e atualiza os dados
      updateColumnsVisibility();
      refreshColumnState();
      updateDisplayedData();

      // Ajustar o tamanho após um pequeno delay para garantir renderização completa
      if (props.content.autoSizeColumns) {
        sizeColumnsToFit();
      }
    };

    const updateDisplayedData = () => {
      if (!gridApi.value) {
        return;
      }

      try {
        let displayedData = [];

        if (props.content.pagination) {
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
          gridApi.value.forEachNodeAfterFilterAndSort((node) => {
            displayedData.push(node.data);
          });
        }

        // Remover linhas de skeleton
        displayedData = displayedData.filter(row => !row?._isSkeletonRow);

        // Atualizar a variável
        setDataTable(displayedData);
      } catch (error) {
      }
    };

    const sizeColumnsToFit = () => {
      if (!gridApi.value) return;

      try {
        // Obtenha todas as colunas visíveis
        const allColumnIds = [];
        gridApi.value.getColumns().forEach((column) => {
          allColumnIds.push(column.getId());
        });

        if (allColumnIds.length > 0) {
          // Aplique autoSize nas colunas visíveis
          gridApi.value.autoSizeColumns(allColumnIds, false);

        }
      } catch (error) {
      }
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
      setDataTable, // Adicionando setDataTable
      updateDisplayedData, // Adicionando updateDisplayedData,
      sizeColumnsToFit,
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
        const skeletonRows = Array(15)
          .fill()
          .map((_, index) => {
            const row = {
              _isSkeletonRow: true,
              _skeletonId: `skeleton-${index}`, // Propriedade para uso no idFormula caso necessário
            };
            // Adicionar um campo para cada coluna
            this.content.columns.forEach((column) => {
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
      this.content.columns.forEach((column) => {
        // Verifica se a coluna tem totalValue definido
        if (column.totalValue) {
          // Verifica se as colunas são bindable (dinâmicas)
          if (this.wwEditorState?.boundProps?.columns) {
            // Se for bindable, tenta usar a propriedade 'total' do objeto
            // Assume que totalValue é um objeto com a propriedade 'total'
            totalRow[column.field] =
              column.totalValue?.total ||
              this.resolveMappingFormula(column.totalValue, null);
          } else {
            // Se não for bindable, usa a fórmula como antes
            totalRow[column.field] = this.resolveMappingFormula(
              column.totalValue,
              null
            );
          }
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
      // Primeiro, processamos as colunas com suas configurações básicas
      const processedColumns = this.content.columns.map((col) => {
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

        // Propriedades básicas da coluna
        const columnDef = {
          field: col.field,
          headerName: col.headerName,
          minWidth,
          maxWidth,
          width,
          flex,
          pinned: col.pinned === "none" ? false : col.pinned,
          hide: col.display === false,
          sortable: col.sortable === false ? false : !!col.sortable,
          filter: col.filter === false ? false : !!col.filter,
          editable: !!col.editable,
          colId: col.field, // Garantir que temos um ID único e estável
          // Armazenamos o parentColumn como metadado
          parentColumn: col.parentColumn || null,
        };

        // Se estiver em modo de carregamento, simplifica para o renderer de skeleton
        if (this.content.loading) {
          return {
            ...columnDef,
            cellRenderer: "SkeletonCellRenderer",
            sortable: false,
            filter: false,
          };
        }

        // Processamento específico por tipo de célula
        switch (col.cellDataType) {
          case "action": {
            return {
              ...columnDef,
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
          case "custom": {
            return {
              ...columnDef,
              cellRenderer: "WewebCellRenderer",
              cellRendererParams: {
                containerId: col.containerId,
              },
            };
          }
          case "image": {
            return {
              ...columnDef,
              cellRenderer: "ImageCellRenderer",
              cellRendererParams: {
                width: col.imageWidth,
                height: col.imageHeight,
              },
            };
          }
          // Tipos de células com formatadores
          case "formatted-number": {
            return {
              ...columnDef,
              valueFormatter: (params) => {
                if (params.value === null || params.value === undefined)
                  return "";
                return this.formatNumber(params.value);
              },
            };
          }
          case "currency": {
            return {
              ...columnDef,
              valueFormatter: (params) => {
                if (params.value === null || params.value === undefined)
                  return "";
                return `R$ ${this.formatNumber(params.value)}`;
              },
            };
          }
          case "percentage": {
            return {
              ...columnDef,
              valueFormatter: (params) => {
                if (params.value === null || params.value === undefined)
                  return "";
                return `${this.formatNumber(params.value)}%`;
              },
            };
          }
          default: {
            // Configurações para outros tipos de células
            if (col.comparative) {
              columnDef.cellRenderer = "ComparativeCellRenderer";
            }

            if (col.useCustomLabel) {
              columnDef.valueFormatter = (params) => {
                return this.resolveMappingFormula(
                  col.displayLabelFormula,
                  params.value
                );
              };
            }

            return columnDef;
          }
        }
      });

      if (this.content.parentColumns && this.content.parentColumns.length > 0) {
        // Cria os grupos de colunas na ordem definida pelos parentColumns
        const columnGroups = [];

        this.content.parentColumns.forEach(parent => {
          if (columnsByParent.has(parent.label)) {
            const children = columnsByParent.get(parent.label);
            // Garantimos que cada coluna filha tenha suas propriedades de ordenação e filtragem
            children.forEach(child => {
              // Preservamos explicitamente estas propriedades que estavam sendo perdidas
              child.sortable = child.sortable !== false;
              child.filter = child.filter !== false;
              child.resizable = this.content.resizableColumns;
            });

            columnGroups.push({
              headerName: parent.label,
              children: children,
              // Propriedades importantes para grupos
              marryChildren: false, // Mudança crítica para permitir operações individuais nas colunas filhas
              groupId: `group_${parent.label}`,
              // Configurações explícitas para o grupo
              sortable: true,
              resizable: this.content.resizableColumns,
              suppressMovable: !this.content.movableColumns,
              // Propriedades adicionais para garantir funcionalidades em colunas agrupadas
              openByDefault: true,
              enableRowGroup: true
            });
          }
        });

        // Retorna as colunas agrupadas e não agrupadas
        return [...columnsWithoutParent, ...columnGroups];
      }

      // Se não houver grupos, apenas retorna as colunas processadas
      // Garantimos que removemos a propriedade parentColumn
      return processedColumns.map((col) => {
        const newCol = { ...col };
        delete newCol.parentColumn;
        return newCol;
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
    refreshColumnState() {
      if (!this.gridApi?.value && !this.gridApi) return;

      const api = this.gridApi?.value || this.gridApi;
      const columnApi = this.columnApi?.value || this.columnApi;

      if (!api || !columnApi) return;

      try {
        // Obtemos o estado atual das colunas
        const columnState = columnApi.getColumnState();

        // Para cada coluna definida no conteúdo
        this.content.columns.forEach(column => {
          // Encontramos a coluna correspondente no estado
          const stateColumn = columnState.find(state => state.colId === column.field);
          if (stateColumn) {
            // Atualizamos propriedades básicas
            stateColumn.hide = column.display === false;
            stateColumn.pinned = column.pinned === "none" ? null : column.pinned;

            // Garantimos que as propriedades essenciais para funcionalidades são mantidas
            stateColumn.sort = stateColumn.sort; // Preserva ordenação

            // Se a coluna tiver um parentColumn
            if (column.parentColumn) {
              // Mantemos propriedades específicas para colunas em grupos
              stateColumn.flex = column.widthAlgo === "flex" ? column.flex ?? 1 : null;
              stateColumn.width = !column.width || column.width === "auto"
                ? null
                : wwLib.wwUtils.getLengthUnit(column.width)?.[0];
            }
          }
        });

        // Aplicamos o estado atualizado das colunas
        columnApi.applyColumnState({
          state: columnState,
          defaultState: {
            sort: null,
            filter: null,
            pinned: null
          }
        });

        // Forçamos um refresh do header para garantir que os eventos funcionem
        api.refreshHeader();
      } catch (error) {
        console.error("Erro ao atualizar estado das colunas:", error);
      }
    },
    updateColumnsVisibility() {
      if (!this.gridApi?.value) return;

      // Para cada coluna na configuração
      this.content.columns.forEach((column) => {
        const columnState = {
          colId: column.field,
          hide: column.display === false,
        };

        // Verifica se a coluna existe antes de tentar atualizar
        const columnExists = this.gridApi.value.getColumn(column.field);
        if (columnExists) {
          // Atualiza o estado da coluna
          this.gridApi.value.columnModel.setColumnState([columnState]);
        }
      });

      // Após atualizar a visibilidade, também atualizamos os dados exibidos
      this.updateDisplayedData();
    },
    formatNumber(value) {
      // Garante que o valor é um número
      const num = Number(value);
      if (isNaN(num)) return value;

      // Formata o número com pontos para milhares e vírgula para decimais
      return num.toLocaleString("pt-BR", {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
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
    'wwEditorState.boundProps.columns': {
      handler(newColumns) {
        // Se as colunas estão sendo alteradas dinamicamente
        this.$nextTick(() => {
          if (this.gridApi?.value || this.gridApi) {
            const api = this.gridApi?.value || this.gridApi;

            // 1. Primeiro destrua todos os listeners
            api.eventService.removeEventListener('sortChanged');
            api.eventService.removeEventListener('filterChanged');
            api.eventService.removeEventListener('paginationChanged');

            // 2. Determine se as colunas têm parentColumns
            const hasParentColumns = newColumns.some(col => col.parentColumn);

            if (hasParentColumns) {
              // 3. Recarregue completamente as definições de coluna
              api.setColumnDefs(this.columnDefs);

              // 4. Reconfigure os listeners
              const updateDisplayedDataFn = () => {
                this.updateDisplayedData();
              };

              api.addEventListener('sortChanged', updateDisplayedDataFn);
              api.addEventListener('filterChanged', updateDisplayedDataFn);
              api.addEventListener('paginationChanged', updateDisplayedDataFn);
            }

            // 5. Atualize o estado das colunas e dados
            this.refreshColumnState();
            this.updateDisplayedData();

            // 6. Force um refresh no header e no grid
            api.refreshHeader();
            api.redrawRows();
          }
        });
      },
      deep: true
    },
    computedRowData: {
      handler() {
        this.$nextTick(() => {
          if (this.gridApi?.value) {
            this.updateDisplayedData();
          }
        });
      },
      deep: true,
    },
    // No watch section do componente
    "content.columns": {
      handler(newColumns, oldColumns) {
        // Verificamos se houve mudança nas propriedades das colunas
        const columnsChanged =
          JSON.stringify(newColumns) !== JSON.stringify(oldColumns);

        // Se as colunas foram alteradas e temos acesso à API da grade
        if (columnsChanged && this.gridApi?.value) {
          this.$nextTick(() => {
            // Primeiro atualizamos a visibilidade das colunas
            this.updateColumnsVisibility();

            // Depois atualizamos o estado completo das colunas
            // (para garantir parent columns, ordenação, etc.)
            this.refreshColumnState();

            // Finalmente, atualizamos os dados exibidos
            this.updateDisplayedData();
          });
        }
      },
      deep: true,
    },
    // Adicionamos um watcher específico para parentColumns
    "content.parentColumns": {
      handler() {
        // Se temos acesso à API da grade
        if (this.gridApi?.value) {
          this.$nextTick(() => {
            // Para mudanças em parentColumns, precisamos forçar uma atualização completa
            // da estrutura de colunas, o que é mais facilmente feito recarregando a grade
            this.gridApi.value.setColumnDefs(this.columnDefs);
            this.refreshColumnState();
          });
        }
      },
      deep: true,
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
  background-color: #a19b9d4d;
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
  background: linear-gradient(90deg,
      transparent,
      rgba(255, 255, 255, 0.2),
      transparent);
  animation: skeleton-wave 1.5s ease-in-out 0.5s infinite;
}
</style>
