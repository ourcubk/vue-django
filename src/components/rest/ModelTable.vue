<template>
    <div>
        <div v-if="showTopBar">
        <span>
            <el-input
                    :placeholder="`搜索${modelTableSearchFieldNames}`"
                    v-model="tableQueries.search"
                    suffix-icon="el-icon-search"
                    @change="tableOnSearch"
                    clearable
                    :style="`width:${modelTableSearchFieldNames.length+5}rem;min-width:10rem;`"
                    v-if="modelTableSearchFields.length>0">
            </el-input>
            <template v-for="f in modelTableFilterFields" v-if="! (f.name in tableBaseQueries)">
                <el-select v-model="tableQueries[f.name]" clearable :placeholder="`请选择${f.label}`"
                           v-if="f.type=='boolean'" @change="tableOnSearch">
                    <el-option :label="f.label" :value="true"></el-option>
                    <el-option :label="getBoolFieldFalseLabel(f.label)" :value="false"></el-option>
                </el-select>
                <related-select :field="f" v-model="tableQueries[f.name]" @input="tableOnSearch"
                                :showCreate="false" :appModelName="f.model"
                                v-if="f.model" :modelListSubUrl="modelListSubUrl"
                                :tablePageSize="100"></related-select>
                &nbsp;
            </template>
        </span>
            <p v-if="batchActionItems && batchActionItems.length>0"/>
            <batch-actions :items="batchActionItems" :count="selectionCount" @done="modelTableRefresh"
                           v-if="batchActionItems"></batch-actions>
            <slot name="actions">
                <actions :items="top_actions" style="float: right"></actions>
            </slot>
        </div>
        <el-table :data="tableData" @row-dblclick="tableOnRowSelect" @sort-change="onSortChange" fixed
                  :max-height="maxHeight"
                  @selection-change="onModelTableSelectionChange"
                  @filter-change="onFilterChanged" v-loading="loading" :element-loading-text="loadingText" ref="table">
            <slot name="left">
            </slot>
            <el-table-column :prop="f.name" :column-key="f.name" :label="f.label || f.name"
                             :min-width="f.min_width" :width="f.width" :fixed="f.fixed"
                             :align="['number','integer'].includes(f.type)?'right':'left'"
                             :sortable="modelTableOrderingFields.includes(f.name) && 'custom'" :class-name="f.type"
                             :type="f.columnType || undefined" :filters="modelTableFilters[f.name]"
                             v-for="f in modelTableItems"
                             :filter-method="f.name in modelTableFilters?filterHandler:undefined"
                             :key="f.columnKey || f.name">
                <template slot-scope="{row}">
                    <form-widget v-if="f.useFormWidget" v-model="row" :field="f" :context="row"
                                 @change="onCellValueChange"></form-widget>
                    <component :is="f.widget" v-model="row[f.name]" :field="f.field" :context="row"
                               v-else-if="f.widget && typeof f.widget == 'object'"></component>
                    <span v-else-if="f.widget && typeof f.widget == 'function'" v-html="f.widget(row)"></span>
                    <template v-else>{{row[f.name]}}</template>
                </template>
            </el-table-column>
            <el-table-column label="" :min-width="`${60*rowActionList.length}px`" align="right">
                <template slot="header" slot-scope="scope">

                </template>
                <template slot-scope="scope">
                    <actions :items="row_actions" :context="scope" class="hover-show" trigger="hover"></actions>
                </template>
            </el-table-column>

            <slot name="right">
            </slot>
        </el-table>
        <div class="model-table-pager-container">
            <el-pagination v-if="tableCount>tablePageSize || showPagger"
                           background
                           layout="total, sizes, prev, pager, next, jumper"
                           :page-size="tablePageSize"
                           :current-page.sync="tablePage"
                           @size-change="tableOnPageSizeChanged"
                           :total="tableCount">
            </el-pagination>
        </div>
    </div>
</template>
<style>
    .filterbox .related-select {
        width: 10rem;
    }
</style>
<script>
    import model_table from '../../mixins/model_table'
    import Actions from '../layout/Actions.vue'
    import FormWidget from '../widgets/FormWidget.vue'
    import BatchActions from '../layout/BatchActions.vue'
    import RelatedSelect from './RelatedSelect.vue'
    //    import DownloadExcel from 'vue-json-excel'
    import Qs from 'qs'
    export default{
        mixins: [model_table],

        components: {
            RelatedSelect, Actions, BatchActions, FormWidget//, DownloadExcel
        },
        created () {
        },
        mounted (){
            this.modelTableInit()
        },
        data(){
            return {
                selectionCount: 0,
                maxHeight: window.screen.availHeight - 100,
                changedData: {}
            }
        },
        methods: {
            getGridData(data){
                let ds = data.map((d) => {
                    return this.modelTableItems.map((a) => {
                        let v = d[a.name]
                        if (a.choices) {
                            return a.choices.find(a => a.value === v).display_name
                        } else if (a.model) {
                            return d[`${a.name}_name`]
                        }
//                        if(typeof v === 'object'){
//                            return JSON.stringify(v)
//                        }
                        return v

                    })
                })
                return [this.modelTableItems.map((a) => a.label)].concat(ds)
            },
            dumpExcelData(){
//                console.log('dumpExcelData')
                let d = Object.assign({}, this.tableQueries, {page: 1, page_size: this.tableCount})
                this.loading = true
                this.loadingText = '正在导出'
                this.$http.get(`${this.tableUrl}?${Qs.stringify(d)}`).then(({data}) => {
                    import('xlsx').then(XLSX => {
                        let wb = XLSX.utils.book_new()
                        let ws = XLSX.utils.aoa_to_sheet(this.getGridData(data.results))
                        XLSX.utils.book_append_sheet(wb, ws, 'Sheet 1')
                        XLSX.writeFile(wb, `${this.modelTableTitle}.xlsx`)
                        this.loading = false
                    })
                }).catch(this.onServerResponseError)
            },
            onModelTableSelectionChange(selection){
                this.selectionCount = selection.length
                this.$emit("selection-change", selection)
            },
            onCommand(name){
                let rc = this.selection.length
                if (rc == 0) {
                    this.$message('请先勾选至少一条记录')
                    return
                }
                let action = this.batchActionItems.find((a) => a.name == name)
                if (action && action.do) {
                    this.$confirm(`确定要对勾选中的${rc}条记录执行"${action.label}"操作吗?`, action.notice, {type: 'warning'}).then(() => {
                        action.do(name).then(({data}) => {
                            this.$message(`操作成功 ${data.rows}`)
                            this.modelTableRefresh()
                        })
                    }).catch(this.onServerResponseError)
                }
                this.$emit("command", name)
            },
            toggleRowExpansion(row, expanded){
                this.$refs.table.toggleRowExpansion(row, expanded)
            },
            filterHandler(value, row, column) {
                return true
//                console.log(column)
//                const property = column['property']
//                console.log(property)
//                return row[property] === value
            },
            onFilterChanged(filters){
                let d = {}
                Object.keys(filters).forEach((a) => {
                    let v = filters[a]
                    if (v instanceof Array) {
                        d[`${a}__in`] = v.join(",")
                    } else {
                        d[a] = v
                    }
                })
//                console.log(d)
                this.tableUpdateQueries(d)
            },
            onSortChange(payLoad){
                this.tableUpdateQueries({ordering: (payLoad.order == 'descending' ? '-' : '') + payLoad.prop})
            },
            getBoolFieldFalseLabel(trueLabel){
                let l = trueLabel
                return l.startsWith('已') ? `未${l.substr(1)}` : (
                    l.startsWith('有') ? `无${l.substr(1)}` : (
                        l.startsWith('是') ? `非${l.substr(1)}` : `非${l}`
                    )
                )
            },
            onCellValueChange({form, field, value}){
                let id = form.id
                let fn = field.name
                let d = this.changedData[id] || {}
                d[fn] = value
                this.changedData[id] = d
                let pd = {}
                this.patchFields.forEach(a => {
                    pd[a.name] = form[a.name]
                })
                this.$http.patch(this.modelGetDetailUrl(id), pd).then(({data}) => {
                    this.$message({message: '保存成功', type: 'success'})
                    this.$emit('change', {data: this.tableData})
                }).catch(error => {
                    this.onServerResponseError(error)
                    this.$message({message: '数据检验未通过，请修正.', type: 'error'})
                })
            }
        },
        computed: {
            selection(){
                return this.$refs.table.selection
            },
            selectionIds(){
                return this.selection.map((a) => a.id)
            },
            patchFields () {
                return this.modelTableItems.filter(a => a.useFormWidget)
            }
        },
        watch: {
            tableData(val){
                this.$emit("input", val)

            }
        }
    }
</script>
<style>
    .model-table-pager-container {
        background-color: white;
        right: 0px;
    }

    .el-table__row td.decimal {
        text-align: right;
    }
</style>
