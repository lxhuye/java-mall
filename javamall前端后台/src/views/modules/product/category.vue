<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button @click="batchDelete" type="danger">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <!--设置可拖动节点 -->
      <!-- 设置append 和 remove -->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            v-if="node.level <= 2"
            @click="() => append(data)"
          >Append</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length==0"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>
    <!-- 添加弹窗 -->
    <el-dialog :title="title" :visible.sync="dialogVisible" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: "",
      dialogType: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },

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
    //设置可拖动功能
    allowDrop(draggingNode, dropNode, type) {
      //被拖动的节点和要去的节点父节点总层数不大于3
      console.log("allowdrop", draggingNode, dropNode, type);
      //统计总层数
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+父节点！>3 就可
      console.log("最大深度", this.maxLevel);
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      if (type == "inner") {
        return deep + dropNode.catLevel <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //找到最深的一个节点
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    //节点拖拽完成后的回调函数
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handledrop: ", draggingNode, dropNode, dropType);
      //1.得到拖拽节点当前的父节点ID
      let pCid = 0;
      let siblings = [];
      let level = 0;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.data.parentCid == undefined ? 0 : dropNode.data.parentCid;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.chlidNodes;
      }
      this.pCid.push(pCid);
      //2.得到当前拖拽节点的排序
      for (let i = 0; i < siblings.length; i++) {
        //如果遍历的是当前的拖拽节点
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前层级发生变化
            catLevel = siblings[i].level;
            //修改子节点的层级
            this.updateChildLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log("updateNodes", this.updateNodes);
    },
    //更新拖拽后子节点的层级
    updateChildLevel(node) {
      if (ode.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildLevel(node.childNodes[i]);
        }
      }
    },

    //批量保存拖拽
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序修改成功",
          type: "success",
        });

        //关闭对话框
        this.dialogVisible = false;
        //重新刷新菜单
        this.getMenus();
        //保留刷新前的层级
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
        // this.parentCid = 0;
      });
    },
    //设置提交数据方法 判断是添加还是修改
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //点击更新按钮的效果
    edit(data) {
      console.log("要修改的数据", data);
      this.title = "修改";
      this.dialogType = "edit";
      this.dialogVisible = true;
      //获取节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    //点击添加按钮的效果
    append(data) {
      this.title = "添加";
      this.dialogVisible = true;
      this.dialogType = "add";
      this.category.parentCid = data.catId;
      //确定层级
      this.category.catLevel = data.catLevel * 1 + 1;
      //重新将数据设置为默认数据
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
    },
    //发送添加信息到后台
    addCategory() {
      console.log("提交三级分类数据", this.category);
      //发送post请求将数据写入数据库
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单添加成功",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //重新刷新菜单
        this.getMenus();
        //保留刷新前的层级
        this.expandedKey = [this.category.parentCid];
      });
    },
    //修改三级分类数据
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = { catId, name, icon, productUnit };
      var ids = [];
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        console.log("修改成功成功");
        //成功提示
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //重新刷新菜单
        this.getMenus();
        //保留刷新前的层级
        this.expandedKey = [this.category.parentCid];
      });
    },
    //添加删除功能
    remove(node, data) {
      var ids = [data.catId];
      //请求前的弹框提示
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            console.log("删除成功");
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            this.getMenus();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    //添加批量删除功能
    batchDelete(){
      let catIds = [];
      //找到我们当前选中的节点
      let checkNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("显示被选中的元素",checkNodes);
      for(let i =0;i<checkNodes.length;i++){
        catIds.push(checkNodes[i].catId);
      }
        //删除请求前的弹框提示
      this.$confirm(`是否批量删除菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            console.log("删除成功");
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            this.getMenus();
          });
        })
        .catch(() => {});

    },
  },
  created() {
    var ids = [];
    this.getMenus();
  },
};
</script>
<style>
</style>