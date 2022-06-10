<template>
  <div>
    <!-- 面包屑导航区 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>商品管理</el-breadcrumb-item>
      <el-breadcrumb-item>商品分类</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片试图区域 -->
    <el-card>
      <el-row>
        <el-col>
          <el-button type="primary" @click="showAddCateDialog">添加分类</el-button>
        </el-col>
      </el-row>
      <!-- 表格 -->
      <tree-table class="treeTable" :data="cateList" :columns="columns" :selection-type="false" :expand-type="false" show-index index-text="#" border :show-row-hover="false">
        <!-- 是否有效 -->
        <template slot="isok" slot-scope="scope">
          <i class="el-icon-success" v-if="scope.row.cat_deleted === false" style="color:green"></i>
          <i class="el-icon-error" v-else style="color:red"></i>
        </template>
        <!-- 排序 -->
        <template slot="order" slot-scope="scope">
          <el-tag size="mini" v-if="scope.row.cat_level === 0">一级</el-tag>
          <el-tag size="mini" type="success" v-else-if="scope.row.cat_level === 1">二级</el-tag>
          <el-tag size="mini" type="warning" v-else>三级</el-tag>
        </template>
        <!-- 操作 -->
        <template slot="option" slot-scope="scope">
          <el-button type="primary" size="mini" @click="exitCateDialog(scope.row.cat_id)">编辑</el-button>
          <el-button type="danger" size="mini" @click="removeCateById(scope.row.cat_id)">删除</el-button>
        </template>
      </tree-table>
      <!-- 分页区域 -->
      <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange" :current-page="queryInfo.pagenum" :page-sizes="[3, 5, 10, 15]" :page-size="queryInfo.pagesize" layout="total, sizes, prev, pager, next, jumper" :total="total"> </el-pagination>
    </el-card>
    <!-- 添加分类对话框 -->
    <el-dialog title="添加分类" :visible.sync="addCatedialogVisible" width="50%" @close="addCateDialogClosed">
      <!-- 添加分类表单 -->
      <el-form ref="addCateFormRef" :rules="addCateFormRules" :model="addCateForm" label-width="100px">
        <el-form-item label="分类名称" prop="cat_name">
          <el-input v-model="addCateForm.cat_name"></el-input>
        </el-form-item>
        <el-form-item label="父级分类">
          <el-cascader v-model="selectedKeys" :options="parentCateList" :props="cascaderProps" @change="parentCateChange" clearable></el-cascader>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="addCatedialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCate">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 修改分类名称 -->
    <el-dialog title="修改分类" :visible.sync="exitCatedialogVisible" width="50%">
      <el-form ref="exitCateRef" :rules="addCateFormRules" :model="addCateForm" label-width="100px">
        <el-form-item label="分类名称">
          <el-input v-model="addCateForm.cat_name"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="exitCatedialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="exitCateName">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 查询条件
      queryInfo: {
        type: 3,
        pagenum: 1,
        pagesize: 5
      },
      // 分类列表
      cateList: [],
      // 数据
      total: [],
      columns: [
        {
          label: '分类名称',
          prop: 'cat_name'
        },
        {
          label: '是否有效',
          type: 'template',
          template: 'isok'
        },
        {
          label: '排序',
          type: 'template',
          template: 'order'
        },
        {
          label: '操作',
          type: 'template',
          template: 'option'
        }
      ],
      // 控制添加分类对话框的显示与隐藏
      addCatedialogVisible: false,
      // 添加分类的表单数据对象
      addCateForm: {
        cat_pid: 0,
        cat_name: '',
        // 分类等级，默认添加一级分类
        cat_level: 0
      },
      // 添加分类表单的校验规则
      addCateFormRules: {
        cat_name: [{ required: true, message: '请输入分类名称', trigger: 'blur' }]
      },
      // 父级分类列表
      parentCateList: [],
      // 指定级联选择器的配置对象
      cascaderProps: {
        expandTrigger: 'hover',
        checkStrictly: true,
        value: 'cat_id',
        label: 'cat_name',
        children: 'children'
      },
      // 选中的父级分类数组
      selectedKeys: [],
      // 修改分类对话框的关闭与隐藏
      exitCatedialogVisible: false
    }
  },
  created() {
    this.getCateList()
  },
  methods: {
    async getCateList() {
      const { data: res } = await this.$http.get('categories', { params: this.queryInfo })
      if (res.meta.status !== 200) {
        return this.$message.error('获取商品分类失败')
      }
      this.cateList = res.data.result
      this.total = res.data.total
    },
    // 监听页面变化事件
    handleSizeChange(newSize) {
      this.queryInfo.pagesize = newSize
      this.getCateList()
    },
    handleCurrentChange(newPage) {
      this.queryInfo.pagenum = newPage
      this.getCateList()
    },
    // 点击按钮展示添加分类对话框
    showAddCateDialog() {
      // 先获取父级分类数据列表
      this.getParentCateList()
      this.addCatedialogVisible = true
    },
    // 获取父级分类权限列表
    async getParentCateList() {
      const { data: res } = await this.$http.get(`categories`, { params: { type: 2 } })
      if (res.meta.status !== 200) {
        return this.$message.error('获取父级分类数据失败')
      }
      this.parentCateList = res.data
    },
    // 选择项发生变化触发函数
    parentCateChange() {
      // 如果selectedKeys数组大于0，证明选中了父级分类
      if (this.selectedKeys.length > 0) {
        this.addCateForm.cat_pid = this.selectedKeys[this.selectedKeys.length - 1]
        this.addCateForm.cat_level = this.selectedKeys.length
        return 0
      } else {
        this.addCateForm.cat_pid = 0
        this.addCateForm.cat_level = 0
      }
    },
    addCate() {
      this.$refs.addCateFormRef.validate(async valid => {
        if (!valid) return
        const { data: res } = await this.$http.post('categories', this.addCateForm)
        if (res.meta.status !== 201) {
          return this.$message.error('添加分类失败')
        }

        this.$message.success('添加分类成功！')
        this.getCateList()
        this.addCatedialogVisible = false
      })
    },
    // 监听对话框的关闭事件，重置表单
    addCateDialogClosed() {
      this.$refs.addCateFormRef.resetFields()
      this.selectedKeys = []
      this.addCateForm.cat_pid = 0
      this.addCateForm.cat_level = 0
    },
    // 根据ID查询分类名称
    async exitCateDialog(id) {
      const { data: res } = await this.$http.get('categories/' + id)
      if (res.meta.status !== 200) {
        return this.$message.error('获取分类失败')
      }
      this.addCateForm.cat_pid = id
      this.addCateForm.cat_name = res.data.cat_name
      this.exitCatedialogVisible = true
    },
    // 编辑提交分类
    async exitCateName() {
      const { data: res } = await this.$http.put('categories/' + this.addCateForm.cat_pid, { cat_name: this.addCateForm.cat_name })
      if (res.meta.status !== 200) {
        return this.$message.error('修改分类失败')
      }
      this.$message.success('修改分类成功！')
      this.getCateList()
      this.exitCatedialogVisible = false
    },
    // 根据ID删除分类
    async removeCateById(id) {
      // 弹框提示
      const confirmRes = await this.$confirm('此操作将永久删除该分类, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).catch(err => err)
      // 用户点击确定返回字符串confirm
      if (confirmRes !== 'confirm') {
        return this.$message.info('已取消删除！')
      }
      const { data: res } = await this.$http.delete('categories/' + id)
      if (res.meta.status !== 200) {
        return this.$message.error('删除分类失败！')
      }
      this.getCateList()
      this.$message.success('删除分类成功！')
    }
  }
}
</script>

<style lang="less" scoped>
.treeTable {
  margin-top: 10px;
}
.el-cascader {
  width: 100%;
}
</style>
