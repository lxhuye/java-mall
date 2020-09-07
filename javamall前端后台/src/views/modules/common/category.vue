// 这是一个三级分类组件

<template>
  <el-tree
    :data="menus"
    :props="defaultProps"
    node-key="catId"
    ref="menuTree"
    @node-click="nodeclick"
  ></el-tree>
</template>

<script>
export default {
  components: {},
  props: {},
  data() {
    return {
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  computed: {},
  watch: {},
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        console.log("成功获取菜单数据", data.data);
        this.menus = data.data;
      });
    },
    nodeclick(data, node, component) {
      this.$emit("check-node-click", data, node, component);
      console.log("子节点捕捉到点击事件",data, node, component);
    },
  },

  created() {
    this.getMenus();
  },
};
</script>
<style>
</style>