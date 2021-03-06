<template>
    <el-row>
        <el-col :sm="c.sm || 24" :md="c.md || 12" :xl="c.xl || 8" v-for="c in chartItems" :key="c.name" v-loading="loading"
                :element-loading-text="loading">
            <template v-if="c.data">
                <x-table v-if="c.type === 'table'" :title="c.title" :group="c.group" :value="genTableData(c)"
                            :items="c.fields"
                            :options="Object.assign({maxHeight:500},c.options)"></x-table>
                <chart v-else :options="c.options" :auto-resize="true"></chart>
            </template>
        </el-col>
    </el-row>
</template>
<script>
    import Qs from 'qs'
    import ServerResponse from 'vue-django/src/mixins/server_response'
    import XTable from 'vue-django/src/components/table/Table.vue'
    import {zipObject} from 'lodash'
    import arrayNormalize from 'vue-django/src/utils/array_normalize'
    let OPTIONS_TOOLBOX = {
        show: true,
        right: '5%',
        feature: {
            dataView: {show: true, title: '数据视图', readOnly: true},
        }
    }
    let COMMON_OPTIONS = {
        tooltip: {
            trigger: 'axis',
            axisPointer: {
                type: 'cross'
            }
        },
        toolbox: OPTIONS_TOOLBOX,
    }

    export default{
        mixins: [ServerResponse],
        props: {
            period: Array,
            url: String,
            items: Array,
            base: String
        },
        data () {
            return {
                chartItems: []
            }
        },
        components: {XTable},
        created () {
            this.normalizeItems()
        },
        mounted() {
            this.loadTimeData(this.period)
        },
        methods: {
            genTableData(c){
                let fns = c.fields.map(f => f.name)
                let data = this.chartData[c.name].map((d) => zipObject(fns, d))
//                console.log(data)
                return data
            },
            genDailyOption(item, data){
                return {
                    visualMap: item.visualMap,
                    xAxis: {
                        type: 'category',
//                        data: data.map((a) => a[0])
                    },
                    yAxis: {
                        type: 'value',
                    },
                    series: [{
                        type: 'line',
                        smooth: true,
                        name: item.title,
//                        data: data.map((a) => a[1])
                    }]
                }
            },
            normalizeItems () {
                let res = {}
                this.chartItems = arrayNormalize(this.items, {}, (a) => {
                    let optionFunc = a.type == 'daily' ? this.genDailyOption : (a.type === 'funnel' ? this.genFunnelOption : this.genBarOption)
                    a.options = {
                        ...COMMON_OPTIONS,
                        title: {
                            text: a.title
                        },
                        ...optionFunc(a),
                        ...a.options
                    }
                    return a
                })
            },
            genTreeMapOption(item, data){

            },
            genFunnelOption (item, data) {
                return {
                    tooltip: {
                        trigger: 'item',
                        formatter: "{a} <br/>{b} : {c}%"
                    },
                    legend: {
//                        data: data.map(a => a[0])
                    },
                    calculable: true,
                    series: [
                        {
                            name: '漏斗图',
                            type: 'funnel',
                            left: '10%',
                            top: 60,
                            bottom: 60,
                            width: '80%',
                            min: 0,
                            max: 100,
                            minSize: '0%',
                            maxSize: '100%',
                            sort: 'descending',
                            gap: 2,
                            label: {
                                show: true,
                                position: 'inside'
                            },
                            labelLine: {
                                length: 10,
                                lineStyle: {
                                    width: 1,
                                    type: 'solid'
                                }
                            },
                            itemStyle: {
                                borderColor: '#fff',
                                borderWidth: 1
                            },
                            emphasis: {
                                label: {
                                    fontSize: 20
                                }
                            },
//                            data: data.map(a => {
//                                return {value: (a[1] / data[0][1] * 100).toFixed(0), name: a[0]}
//                            })
                        }
                    ]
                }
            },
            genBarOption (item, data){
                let dataZoom = []
//                if (data.length >= 16) {
//                    dataZoom.push(
//                        {
//                            id: 'dataZoomY',
//                            type: 'slider',
//                            yAxisIndex: [0],
//                            filterMode: 'filter',
//                            show: true,
//                            start: 0,
//                            end: 900 / data.length
//                        })
//                }
                return  {
                    dataZoom,
                    yAxis: {
                        type: 'category',
//                        data: data.map((a) => a[0])
                    },
                    grid: {
                        left: '33%',
                    },
                    xAxis: {
                        position: 'top',
                        type: 'value',
                    },
                    series: [{
                        type: 'bar',
                        smooth: true,
                        name: item.title,
//                        data: data.map((a) => a[1])
                    }]
                }
            },
            loadTimeData(period){
                let context = {measures: this.chatItems.map((a) => a.name), period: `${period[0]}至${period[1]}`}
                let qs = Qs.stringify(context, {indices: false})
                this.loading = "加载中"
                this.$http.get(`${this.url}?${qs}`).then(({data}) => {
                    this.loading = false
                    let ds = data
                    if (this.base) {
                        ds = ds[this.base]
                    }
                    this.chartData = Object.assign({}, ds)
                }).catch(this.onServerResponseError)
            },
        },
        computed: {
            chartOptions(){
                let res = {}
                this.items.forEach((a) => {
                    let optionFunc = a.type == 'daily' ? this.genDailyOption : (a.type === 'funnel' ? this.genFunnelOption : this.genBarOption)
                    res[a.name] = {
                        ...COMMON_OPTIONS,
                        title: {
                            text: a.title
                        },
                        ...optionFunc(a, this.chartData[a.name]),
                        ...a.options
                    }
                })
                return res
            }
        },
        watch: {
            period(val){
                this.loadTimeData(val)
            }
        }
    }
</script>
