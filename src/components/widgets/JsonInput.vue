<template>
    <el-input v-model="valueStr" :autosize="field.autosize || { minRows: 4, maxRows: 10}" type="textarea" @change="onChange"></el-input>
</template>
<script>
    export default{
        props: {
            value: Object,
            field:Object,
            defaultValue : [String, Object, Array]
        },
        data () {
            return {
                valueStr: ''
            }
        },
        created(){
            this.setValue()
        },
        components: {},
        methods: {
            setValue(){
                this.valueStr = JSON.stringify(this.value || this.defaultValue, null, 4)

            },
            onChange(v){
                this.$emit("input", JSON.parse(v))
            }
        },
        watch: {
            value(val){
                this.setValue()
            }
        }
    }
</script>
