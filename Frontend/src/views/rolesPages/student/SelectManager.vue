<template
>
    <div class="app-container">
        <div class="filter-container">
            <el-input
                    @keyup.enter.native="handleFilter"
                    class="filter-item"
                    clearable
                    placeholder="搜索内容"
                    style="width: 200px;"
                    v-model="testQuery.content"
            />
            <el-select
                    class="filter-item"
                    clearable
                    placeholder="课程名/授课老师"
                    style="width: 180px"
                    v-model="testQuery.type"
            >
                <el-option
                        :key="item"
                        :label="item"
                        :value="index"
                        v-for="(item,index) in searchTypeOptions"
                />
            </el-select>

            <el-select
                    class="filter-item"
                    clearable
                    placeholder="选课状态"
                    style="width: 120px"
                    @change="handleFilter"
                    v-model="testQuery.status"
            >
                <el-option
                        :key="item"
                        :label="item"
                        :value="item"
                        v-for="item in chooseStatus"
                />
            </el-select>
            <el-button
                    @click="handleFilter"
                    class="filter-item"
                    icon="el-icon-search"
                    type="primary"
                    v-waves
            >
                搜索
            </el-button>
            <el-popconfirm
                    :title="selectTipText"
                    @onConfirm="handleSelected"
            >
                <el-button
                        :type="btnStatus"
                        class="filter-item"
                        icon="el-icon-check"
                        slot="reference"
                        style="margin-left: 10px; float: right"
                        v-waves
                >
                    确认已选择项
                </el-button>
            </el-popconfirm>

        </div>
        <!-- 表格-->
        <el-table
                :data="list"
                :height="tableHeight"
                :key="tableKey"
                :row-key="Math.round(Math.random()*10000).toString()"
                @selection-change="handleSelectionChange"
                @sort-change="sortChange"
                border
                fit
                highlight-current-row
                ref="refTable"
                style="width: 100%"
                v-loading="listLoading"
        >
            <el-table-column align="center" type="selection"/>
            <el-table-column
                    align="center"
                    label="ID"
                    prop="id"
                    sortable="custom"
                    width="70px"
            >
                <template slot-scope="{row}">
                    <span>{{ row.id }}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="课程名"
                    width="150px"
            >
                <template slot-scope="{row}">
                    <span>{{ row.cname }}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="学分"
                    width="110px"
            >
                <template slot-scope="{row}">
                    <span>{{ row.mark }}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="授课老师"
                    min-width="100px"
            >
                <template slot-scope="{row}">
                    <span>{{matchTeacher(row.tId)}}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="授课地点"
                    min-width="90px"
            >
                <template slot-scope="{row}">
                    <span>{{matchClassRoom(row.roomId)}}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="开课时间"
                    min-width="180px"
            >
                <template slot-scope="{row}">
                    <span>{{ row.startTime}}-{{row.endTime}}</span>
                </template>
            </el-table-column>
            <el-table-column
                    align="center"
                    label="选课情况"
                    min-width="60px"
            >
                <template slot-scope="{row}">
                    <span @click="handleFetchStudents(row.id)" class="link-type">{{ row.selectedNum }}</span>
                    <span>/{{row.maxChooseNum}}</span>
                </template>
            </el-table-column>
        </el-table>
        <!-- 分页组件-->
        <pagination
                :limit.sync="testQuery.limit"
                :page.sync="testQuery.page"
                :total="total"
                @pagination="getList"
                v-show="total>0"
        />
        <el-dialog
                :visible.sync="dialogSelectedVisible"
                @dragDialog="handleDrag()"
                title="已选课学生信息汇总"
                v-el-drag-dialog
        >
            <el-table
                    :data="studentTableData"
                    border highlight-current-row ref="rankTable" stripe style="width: 100%;margin-top:20px;">
                <el-table-column :key="item.key" :label="item.value" :prop="item.key"
                                 v-for="item of studentTableHeader"/>
            </el-table>
            <span
                    class="dialog-footer"
                    slot="footer"
            >

                <el-button
                        @click="dialogSelectedVisible = false"
                        type="primary"
                >确认</el-button>
         </span>
        </el-dialog>
    </div>
</template>

<script>
  import waves from '@/directive/waves' // waves directive
  import Pagination from '@/components/Pagination/index'
  import elDragDialog from '@/directive/el-drag-dialog'
  import { getClassRooms, getTeachers } from '@/api/common'
  import { addMySelect, fetchMyClass, fetchMySelect } from '@/api/student/course'

  export default {
    name: 'SelectManager',
    components: { Pagination },
    directives: { waves, elDragDialog },
    computed: {
      //表格自适应高度
      tableHeight() {
        return window.innerHeight * 0.75
      },
      selectTipText() {
        return `确认选择${this.multipleSelection.length}项课程?`
      }
    },

    data() {
      return {
        btnStatus: '',
        multipleSelection: [],
        searchTypeOptions: ['课程名', '授课老师'],
        studentTableData: [],
        studentTableHeader: [
          {
            key: 'sno',
            value: '学号'
          },
          {
            key: 'name',
            value: '姓名'
          },
          {
            key: 'sex',
            value: '性别'
          },
          {
            key: 'fId',
            value: '所在院系'
          }
        ],
        facultyOptions: this.$store.getters.faculties,
        teacherOptions: [],
        classRoomOptions: [],
        chooseStatus: ['已满', '未满'],
        sexOptions: ['男', '女'],
        // 查询串
        testQuery: {
          // 搜索的内容
          content: '',
          // 搜索的类别
          type: '',
          //状态 伪!@
          status: '',
          // 排序方式
          sort: 1,
          // 第几页
          page: 1,
          // 每页限制多少
          limit: 20
        },
        tableKey: 0,
        list: [],
        //总页数
        total: 0,

        listLoading: true,

        dialogSelectedVisible: false

      }
    },

    created() {
      // 初始化获取数据
      this.getList()
    },
    methods: {
      //根据选择状态切换按钮
      handleSelectionChange(val) {
        this.multipleSelection = val
        this.btnStatus = (val.length > 0) ? 'success' : ''
      },
      //根据ID匹配院系,该匹配数据在页面刷新且已登录时获取
      matchFaculty(id) {
        return this.facultyOptions.find(item => item.id === id)?.name
      },
      matchTeacher(t_id) {
        return this.teacherOptions.find(item => item.id === t_id)?.name
      },
      matchClassRoom(room_id) {
        return this.classRoomOptions.find(item => item.id === room_id)?.name
      },
      /**
       * 提醒功能
       * @param how 提醒的方式 默认上浮
       * @param type 提醒的类型
       * @param message 消息正文
       */
      showToast(type, message, how = 1) {
        if (how === 1) {
          this.$notify({
            title: '提示',
            message: message,
            type: type,
            duration: 2000
          })
        } else {

          this.$message({
            message: message,
            type: type
          })
        }

      },
      /**
       * 选择成功后 课程将从面板消失
       */
      handleSelected() {
        if (this.multipleSelection.length > 0) {
          let selection = []
          for (const argument of this.multipleSelection) {
            selection.push(argument.id)
          }

          addMySelect(selection).then(resp => {
            if (resp.code === 20000) {
              for (const selected of this.multipleSelection) {

                let indexOf = this.list.indexOf(selected)
                if (indexOf !== -1) {
                  this.list.splice(indexOf, 1)
                }
              }
              // console.log(this.multipleSelection)
              this.$refs.refTable.clearSelection()

              this.showToast('success', resp.message + ',请前往已选课程面板查看当前已选')
            } else {
              this.showToast('error', response.message)
            }
          })

        }
      },
      // 显示学生信息
      handleFetchStudents(id) {
        this.dialogSelectedVisible = true

        fetchMyClass(id).then(response => {
          if (response.code === 20000) {
            const data = response.data.items

            //数据匹配清洗
            this.studentTableData = data.map(item => {
              item.fId = this.matchFaculty(item.fId)
              return item
            })
          }
        })
      },

      // 表格数据
      async getList() {
        this.listLoading = true
        //教室集
        const classrooms = await getClassRooms()
        this.classRoomOptions = classrooms.data
        //老师集
        const teacherRes = await getTeachers()
        this.teacherOptions = teacherRes.data

        fetchMySelect(this.testQuery).then(response => {
          if (response.code !== 20000) {
            this.showToast('error', response.message)
            this.listLoading = false
            return
          }

          this.list = response.data.items
          this.total = response.data.total
          this.listLoading = false
          // Just to simulate the time of the request
          setTimeout(() => {
            this.listLoading = false
          }, 2 * 100)
        })
      },
      // 搜索        debugger

      handleFilter() {
        this.testQuery.page = 1
        this.getList()
      },

      //响应排序
      sortChange(data) {
        const { prop, order } = data
        if (prop === 'id') {
          this.sortByID(order)
        }
      },
      sortByID(order) {
        if (order === 'ascending') {
          this.testQuery.sort = '-1'
        } else {
          this.testQuery.sort = '1'
        }
        this.handleFilter()
      },
      // v-el-drag-dialog onDrag callback function
      handleDrag() {
        this.$refs.select.blur()
      }

    }
  }
</script>
