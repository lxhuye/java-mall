<template>
  <el-tree :data="data" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
</template>

<script>
  export default {
    data(){
        return{
            data: [],
            defaultProps: {
                children: "children",
                label: "label"
            }
        };
    },
    methods: {
      handleNodeClick(data) {
        console.log(data);
      },
      getMenus(){
          this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get',
          params: this.$http.adornParams({
            
          })
        }).then(data=>{
            console.log("成功获取后台数据")
        })
      }
    },
    created(){
        this.getMenus();
    }
  };
</script>
<style>
</style>