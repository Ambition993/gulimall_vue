<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
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
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            添加分类
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            编辑分类
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除分类
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
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
      title: "", // dialog title
      dialogType: "", // dialogType  which type you need {add edit}
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: "",
        icon: "",
      },
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [],
      dialogVisible: false,
    };
  },
  created() {
    this.getDataList();
  },
  methods: {
    // 获取数据列表
    getDataList() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.data;
      });
    },
    // before edit this category i need some info about this category and show in the form
    edit(data) {
      console.log("hello edit");
      this.title = "修改分类";
      this.dialogType = "edit";
      this.dialogVisible = true;
      //request for newest data
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //request ok  and show data
        console.log(data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    // prepare data
    // before adding  category to database  i will collect all of the data that i put in and then click submit call addCategory()
    // data will be sent
    append(data) {
      this.title = "添加分类";
      this.dialogType = "append";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.name = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.showStatus = 1;
      this.category.sort = 0;
    },
    //remove category from database
    remove(node, data) {
      let ids = [data.catId];
      this.$confirm(`此操作将删除[${data.name}]菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({}) => {
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            // reflesh
            this.getDataList();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    //edit category
    editCategory(data) {
      // get data from form and create a object  dataWillSend
      let { catId, name, icon, productUnit } = this.category;
      // the data i will send some fields of the category is not needed  you just need to update some of them
      let dataWillSend = {
        catId: catId,
        name: name,
        icon: icon,
        productUnit: productUnit,
      };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(dataWillSend, false),
      }).then(() => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        //close dialog
        this.dialogVisible = false;
        //reflesh data
        this.getDataList();
        //show parent category
        this.expandedKey = [this.category.parentCid];
      });
    },
    //添加三级分类的方法
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(() => {
        this.$message({
          message: "菜单添加成功",
          type: "success",
        });
        // close dialog
        this.dialogVisible = false;
        // reflesh
        this.getDataList();
        // expand
        this.expandedKey = [this.category.parentCid];
      });
    },
    //before submit data you need to switch methods  and then you will collect(apend edit) all of these data and send
    submitData() {
      if (this.dialogType == "edit") {
        this.editCategory();
      }
      if (this.dialogType == "append") {
        this.addCategory();
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      //被拖动的节点以及父节点的总层数不能大于3
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点的 + 父节点的深度不大于3 就可以
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log(deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    //
    countNodeLevel(node) {
      //找到所有子节点 求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", dropNode.label, dropType);
      // 1当前节点最新的父节点ID
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        // 当前拖拽节点的最新顺序
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        // 当前拖拽节点的最新顺序
        siblings = dropNode.childNodes;
      }
      this.pCid.push(pCid);
      // 2当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            // 当前节点的层级发生了变化
            catLevel = siblings[i].level;
            // 当前拖拽的节点的层级如果改变了就要修改当前拖拽节点的子节点的层级
            // 修改子节点的层级
            this.updateChildNoesLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      //
      console.log("updateNoes", this.updateNodes);
      // 当前拖拽节点的最新层级
    },
    // 更新子节点的层级
    updateChildNoesLevel(node) {
      if (node.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var currentNode = node.childNodes[i].data;
          this.updateNodes.push({
            catid: currentNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNoesLevel(node.childNodes[i]);
        }
      }
    },
    //批量删除
    batchDelete() {
      let catIds = [];
      let checkedNoes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的Nodes", checkedNoes);
      for (let i = 0; i < checkedNoes.length; i++) {
        catIds.push(checkedNoes[i].catId);
      }
      // 发送请求
      this.$confirm(`是否批量删除[${catIds}]菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({}) => {
            this.$message({
              message: "菜单批量删除成功",
              type: "success",
            });
            // reflesh
            this.getDataList();
          });
        })
        .catch(() => {});
    },
    // 批量保存拖拽的效果  点击批量保存按钮之后发送请求
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(() => {
        this.$message({
          message: "菜单顺序等成功",
          type: "success",
        });
        // reflesh
        this.getDataList();
        // expand
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
      });
    },
  },
};
</script>
<style>
</style>
