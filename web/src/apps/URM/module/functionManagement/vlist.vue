<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one or more
  ~ contributor license agreements.  See the NOTICE file distributed with
  ~ this work for additional information regarding copyright ownership.
  ~ The ASF licenses this file to You under the Apache License, Version 2.0
  ~ (the "License"); you may not use this file except in compliance with
  ~ the License.  You may obtain a copy of the License at
  ~ 
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~ 
  ~ Unless required by applicable law or agreed to in writing, software
  ~ distributed under the License is distributed on an "AS IS" BASIS,
  ~ WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  ~ See the License for the specific language governing permissions and
  ~ limitations under the License.
  -->

<template>
  <div>
    <Table class="table-content" border :columns="cols" :data="pageList" :loading="loading"></Table>
    <Page
      style="text-align:center;margin-top:10px"
      :total="pageSetting.total"
      :page-size="pageSetting.pageSize"
      :current="pageSetting.current"
      size="small"
      show-total
      show-elevator
      @on-change="changePage"
    />
    <Modal v-model="showContent" footer-hide width="80%" title="查看源码" class="view-code">
      <Spin v-if="loadingContent" size="large" fix />
      <Input
        v-show="!loadingContent"
        type="textarea"
        v-model="content"
        :disabled="true"
        :autosize="{ minRows: 5, maxRows: 20 }"
      />
    </Modal>
  </div>
</template>
<script>
import moment from 'moment'
import api from '@/common/service/api'
import axios from 'axios'
export default {
  props: {
    row: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      cols: [
        {
          title: '版本',
          key: 'bmlResourceVersion',
          align: 'center',
          width: 95
        },
        {
          title: '发布',
          key: 'publish',
          align: 'center',
          width: 75
        },
        {
          title: '状态',
          key: 'status',
          align: 'center',
          width: 80
        },
        {
          title: '注释',
          key: 'description',
          align: 'center',
          width: 170
        },
        {
          title: '修改时间',
          key: 'createTimeFormat',
          align: 'center',
          width: 160
        },
        {
          title: '创建者',
          key: 'createUser',
          align: 'center',
          width: 120
        }, {
          title: '操作',
          align: 'center',
          width: 285,
          render: (h, params) => {
            return h('div', [
              h('Button', {
                props: {
                  size: 'small',
                },
                style: {
                  marginRight: '5px',
                },
                on: {
                  click: () => {
                    this.back(params.row)
                  }
                }
              }, '创建新版本'),
              h('Button', {
                props: {
                  size: 'small',
                },
                style: {
                  marginRight: '5px',
                  display: this.row && this.row.operationStatus && this.row.operationStatus.canPublish ? 'inline-block' : 'none'
                },
                on: {
                  click: () => {
                    this.publish(params.row)
                  }
                }
              }, '发布'),
              h('Button', {
                props: {
                  size: 'small',
                },
                style: {
                  marginRight: '5px',
                },
                on: {
                  click: () => {
                    this.download(params.row)
                  }
                }
              }, '下载'),
              h('Button', {
                props: {
                  size: 'small',
                },
                style: {
                  marginRight: '5px',
                  display: this.row.udfType === 0 ? 'none' : 'inline-block'
                },
                on: {
                  click: () => {
                    this.viewCode(params.row)
                  }
                }
              }, '查看源码')
            ]);
          }
        }],
      data: [],
      loading: false,
      loadingContent: false,
      content: '',
      showContent: false,
      pageSetting: {
        total: 0,
        pageSize: 10,
        current: 1
      }
    }
  },
  watch: {
    "row": function () {
      this.fetchList()
    }
  },
  computed: {
    pageList() {
      return this.data.slice((this.pageSetting.current - 1) * this.pageSetting.pageSize, this.pageSetting.current * this.pageSetting.pageSize)
    }
  },
  mounted() {
  },
  methods: {
    fetchList() {
      if (this.row && this.row.id) {
        this.loading = true
        api
          .fetch('/udf/versionList', {
            udfId: this.row.id
          }, 'get')
          .then((res) => {
            this.data = (res.versionList || []).map(it => {
              it.status = it.expire ? '过期' : '正常'
              it.publish = it.published ? '是' : '否'
              it.createTimeFormat = moment(it.createTime).format('YYYY-MM-DD HH:mm:ss')
              return it
            })
            this.pageSetting.total = this.data.length
            this.loading = false
          })
          .catch(() => {
            this.loading = false
          })
      }
    },
    viewCode(row) {
      this.showContent = true
      this.loadingContent = true
      api
        .fetch('/udf/downloadUdf', {
          udfId: row.udfId,
          version: row.bmlResourceVersion
        }, 'post')
        .then((data) => {
          this.content = (data && data.content) || ''
          this.loadingContent = false
        })
        .catch(() => {
          this.showContent = false
          this.loadingContent = false
        })
    },
    back(row) {
      this.loading = true
      api
        .fetch('/udf/rollback', {
          udfId: row.udfId,
          version: row.bmlResourceVersion
        }, 'post')
        .then(() => {
          this.fetchList()
          this.$Message.success('操作成功')
          this.$emit('refresh-list')
        })
        .catch(() => {
          this.loading = false
        })
    },
    publish(row) {
      this.loading = true
      api
        .fetch('/udf/publish', {
          udfId: row.udfId,
          version: row.bmlResourceVersion
        }, 'post')
        .then(() => {
          this.$Message.success('操作成功')
          this.fetchList()
        })
        .catch(() => {
          this.loading = false
        })
    },
    download(row) {
      axios({
        url: process.env.VUE_APP_MN_CONFIG_PREFIX || `http://${window.location.host}/api/rest_j/v1/udf/downloadToLocal`,
        method: 'post',
        responseType: 'blob',
        data: {
          udfId: row.udfId,
          version: row.bmlResourceVersion
        }
      })
        .then((res) => {
          let blob = res.data
          let url = window.URL.createObjectURL(blob);
          let l = document.createElement('a')
          l.href = url;
          let name = row.path.split('/').pop()
          l.download = name
          document.body.appendChild(l);
          l.click()
          window.URL.revokeObjectURL(url)
          this.$Message.success('操作成功')
        })
        .catch(() => {
          this.loading = false
        })
    },
    changePage(page) {
      this.pageSetting.current = page
    }
  }
};
</script>
