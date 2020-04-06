<template>
  <div>
    <el-row style="margin-top: 10px">
      <el-col>
        <el-card>
          <div slot="header">
            Subscription Converter
            <svg-icon icon-class="github" style="margin-left: 20px" @click="goToProject" />

            <div style="display: inline-block; position:absolute; right: 20px">{{ backendVersion }}</div>
          </div>
          <el-container>
            <el-form :model="form" label-width="120px" label-position="left" style="width: 100%">
              <el-form-item label="模式设置:">
                <el-radio v-model="advanced" label="1">基础模式</el-radio>
                <el-radio v-model="advanced" label="2">进阶模式</el-radio>
              </el-form-item>
              <el-form-item label="订阅链接:">
                <el-input
                  v-model="form.sourceSubUrl"
                  type="textarea"
                  rows="3"
                  placeholder="支持订阅或ss/ssr/vmess单链接。多个链接请每行一个或用 | 分隔"
                />
              </el-form-item>
              <el-form-item label="客户端:">
                <el-select v-model="form.clientType" style="width: 100%">
                  <el-option v-for="(v, k) in options.clientTypes" :key="k" :label="k" :value="v" />
                </el-select>
              </el-form-item>

              <div v-if="advanced === '2'">
                <el-form-item label="后端地址:">
                  <el-autocomplete
                    v-model="form.customBackend"
                    style="width: 100%"
                    :fetch-suggestions="backendSearch"
                    placeholder="动动小手，（建议）自行搭建后端服务。例：http://127.0.0.1:25500/sub?"
                  >
                    <el-button slot="append" icon="el-icon-link" @click="gotoGayhub">前往项目仓库</el-button>
                  </el-autocomplete>
                </el-form-item>
                <el-form-item label="远程配置:">
                  <el-select
                    v-model="form.remoteConfig"
                    allow-create
                    filterable
                    placeholder="请选择"
                    style="width: 100%"
                  >
                    <el-option-group
                      v-for="group in options.remoteConfig"
                      :key="group.label"
                      :label="group.label"
                    >
                      <el-option
                        v-for="item in group.options"
                        :key="item.value"
                        :label="item.label"
                        :value="item.value"
                      />
                    </el-option-group>
                    <el-button slot="append" icon="el-icon-link" @click="gotoRemoteConfig">配置示例</el-button>
                  </el-select>
                </el-form-item>
                <el-form-item label="IncludeRemarks:">
                  <el-input v-model="form.includeRemarks" placeholder="节点名包含的关键字，支持正则" />
                </el-form-item>
                <el-form-item label="ExcludeRemarks:">
                  <el-input v-model="form.excludeRemarks" placeholder="节点名不包含的关键字，支持正则" />
                </el-form-item>
                <el-form-item label="FileName:">
                  <el-input v-model="form.filename" placeholder="返回的订阅文件名" />
                </el-form-item>
                <el-form-item label-width="0px">
                  <el-row type="flex">
                    <el-col>
                      <el-checkbox v-model="form.nodeList" label="输出为 Node List" border />
                      <el-checkbox v-model="form.emoji" label="Emoji" border />
                    </el-col>
                    <el-popover v-model="form.extraset">
                      <el-row>
                        <el-checkbox v-model="form.sort" label="排序节点" />
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.appendType" label="节点类型" />
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.udp" label="启用 UDP" />
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.tfo" label="启用 TFO" />
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.scv" label="跳过证书验证" />
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.fdn" label="过滤非法节点" />
                      </el-row>
                      <el-button slot="reference">更多选项</el-button>
                    </el-popover>
                  </el-row>
                </el-form-item>
              </div>

              <div style="margin-top: 50px" />

              <el-divider content-position="center">
                <i class="el-icon-magic-stick" />
              </el-divider>

              <el-form-item label="定制订阅:">
                <el-input v-model="customSubUrl" class="copy-content" disabled>
                  <el-button
                    slot="append"
                    ref="copy-btn"
                    v-clipboard:copy="customSubUrl"
                    v-clipboard:success="onCopy"
                    icon="el-icon-document-copy"
                  >复制</el-button>
                </el-input>
              </el-form-item>
              <el-form-item label="订阅短链接:">
                <el-input v-model="curtomShortSubUrl" class="copy-content" disabled>
                  <el-button
                    slot="append"
                    ref="copy-btn"
                    v-clipboard:copy="curtomShortSubUrl"
                    v-clipboard:success="onCopy"
                    icon="el-icon-document-copy"
                  >复制</el-button>
                </el-input>
              </el-form-item>

              <el-form-item label-width="0px" style="margin-top: 40px; text-align: center">
                <el-button
                  style="width: 120px"
                  type="danger"
                  :disabled="form.sourceSubUrl.length === 0"
                  @click="makeUrl"
                >生成订阅链接</el-button>
                <el-button
                  style="width: 120px"
                  type="danger"
                  :loading="loading"
                  :disabled="customSubUrl.length === 0"
                  @click="makeShortUrl"
                >生成短链接</el-button>
                <!-- <el-button style="width: 120px" type="primary" @click="surgeInstall" icon="el-icon-connection">一键导入Surge</el-button> -->
              </el-form-item>

              <el-form-item label-width="0px" style="text-align: center">
                <el-button
                  style="width: 120px"
                  type="primary"
                  icon="el-icon-upload"
                  :loading="loading"
                  @click="dialogUploadConfigVisible = true"
                >上传配置</el-button>
                <el-button
                  style="width: 120px"
                  type="primary"
                  icon="el-icon-connection"
                  :disabled="customSubUrl.length === 0"
                  @click="clashInstall"
                >一键导入Clash</el-button>
              </el-form-item>
            </el-form>
          </el-container>
        </el-card>
      </el-col>
    </el-row>

    <el-dialog
      :visible.sync="dialogUploadConfigVisible"
      :show-close="false"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      width="700px"
    >
      <div slot="title">
        Remote config upload
        <el-popover trigger="hover" placement="right" style="margin-left: 10px">
          <el-link type="primary" :href="sampleConfig" target="_blank" icon="el-icon-info">参考配置</el-link>
          <i slot="reference" class="el-icon-question" />
        </el-popover>
      </div>
      <el-form label-position="left">
        <el-form-item prop="uploadConfig">
          <el-input
            v-model="uploadConfig"
            type="textarea"
            :autosize="{ minRows: 15, maxRows: 15}"
            maxlength="3000"
            show-word-limit
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="uploadConfig = ''; dialogUploadConfigVisible = false">取 消</el-button>
        <el-button
          type="primary"
          :disabled="uploadConfig.length === 0"
          @click="confirmUploadConfig"
        >确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import clip from '@/utils/clipboard' // use clipboard directly

const remoteConfigSample = 'https://raw.githubusercontent.com/tindy2013/subconverter/master/base/config/example_external_config.ini'

export default {
  name: 'Subconverter',
  data() {
    return {
      backendVersion: '',
      advanced: '2',

      options: {
        clientTypes: {
          Clash: 'clash',
          ClashR: 'clashr',
          Surge2: 'surge&ver=2',
          Surge3: 'surge&ver=3',
          Surge4: 'surge&ver=4',
          Quantumult: 'quan',
          QuantumultX: 'quanx',
          Surfboard: 'surfboard',
          Loon: 'loon',
          ss: 'ss',
          ssr: 'ssr',
          ssd: 'ssd'
        },
        backendOptions: [{ value: process.env.VUE_APP_LOCAL_SUBCONVERTER_API }],
        remoteConfig: [
          {
            label: 'universal',
            options: [
              {
                label: 'No-Urltest',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/universal/no-urltest.ini'
              },
              {
                label: 'Urltest',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/universal/urltest.ini'
              }
            ]
          },
          {
            label: 'customized',
            options: [
              {
                label: 'Maying',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/maying.ini'
              },
              {
                label: 'rixCloud',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/rixcloud.ini'
              },
              {
                label: 'YoYu',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/yoyu.ini'
              },
              {
                label: 'Ytoo',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/ytoo.ini'
              },
              {
                label: 'NyanCAT',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/nyancat.ini'
              },
              {
                label: 'Nexitally',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/nexitally.ini'
              },
              {
                label: '贼船',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/customized/zeichuan.ini'
              }
            ]
          },
          {
            label: 'Special',
            options: [
              {
                label: 'NeteaseUnblock(仅规则，No-Urltest)',
                value:
                    'https://raw.githubusercontent.com/CareyWang/Rules/master/RemoteConfig/special/netease.ini'
              }
            ]
          }
        ]
      },
      form: {
        sourceSubUrl: '',
        clientType: '',
        customBackend: '',
        remoteConfig: '',
        excludeRemarks: '',
        includeRemarks: '',
        filename: '',
        emoji: true,
        nodeList: false,
        sort: false,
        udp: false,
        tfo: false,
        scv: false,
        fdn: false,
        appendType: false
      },

      loading: false,
      customSubUrl: '',
      curtomShortSubUrl: '',

      dialogUploadConfigVisible: false,
      uploadConfig: '',
      uploadPassword: '',
      myBot: process.env.VUE_APP_TELEGRAM_BOT,
      sampleConfig: remoteConfigSample
    }
  },
  mounted() {
    this.form.clientType = 'clash'
    this.notify()
    this.getBackendVersion()
  },
  methods: {
    onCopy() {
      this.$message.success('Copied!')
    },
    goToProject() {
      window.open(project)
    },
    gotoGayhub() {
      window.open(gayhubRelease)
    },
    gotoRemoteConfig() {
      window.open(remoteConfigSample)
    },
    clashInstall() {
      if (this.customSubUrl === '') {
        this.$message.error('请先填写必填项，生成订阅链接')
        return false
      }

      const url = 'clash://install-config?url='
      window.open(url + encodeURIComponent(this.curtomShortSubUrl !== '' ? this.curtomShortSubUrl : this.customSubUrl))
    },
    surgeInstall() {
      if (this.customSubUrl === '') {
        this.$message.error('请先填写必填项，生成订阅链接')
        return false
      }

      const url = 'surge://install-config?url='
      window.open(url + this.customSubUrl)
    },
    makeUrl() {
      if (this.form.sourceSubUrl === '' || this.form.clientType === '') {
        this.$message.error('订阅链接与客户端为必填项')
        return false
      }

      const backend =
        this.form.customBackend === ''
          ? defaultBackend
          : this.form.customBackend

      let sourceSub = this.form.sourceSubUrl
      sourceSub = sourceSub.replace(/[\n|\r|\n\r]/g, '|')

      this.customSubUrl =
        backend +
        'target=' +
        this.form.clientType +
        '&url=' +
        encodeURIComponent(sourceSub)

      if (this.advanced === '2') {
        if (this.form.remoteConfig !== '') {
          this.customSubUrl +=
            '&config=' + encodeURIComponent(this.form.remoteConfig)
        }
        if (this.form.excludeRemarks !== '') {
          this.customSubUrl +=
            '&exclude=' + encodeURIComponent(this.form.excludeRemarks)
        }
        if (this.form.includeRemarks !== '') {
          this.customSubUrl +=
            '&include=' + encodeURIComponent(this.form.includeRemarks)
        }
        if (this.form.filename !== '') {
          this.customSubUrl +=
            '&filename=' + encodeURIComponent(this.form.filename)
        }
        if (this.form.appendType) {
          this.customSubUrl +=
            '&append_type=' + this.form.appendType.toString()
        }

        this.customSubUrl +=
          '&emoji=' +
          this.form.emoji.toString() +
          '&list=' +
          this.form.nodeList.toString() +
          '&udp=' +
          this.form.udp.toString() +
          '&tfo=' +
          this.form.tfo.toString() +
          '&scv=' +
          this.form.scv.toString() +
          '&fdn=' +
          this.form.fdn.toString() +
          '&sort=' +
          this.form.sort.toString()
      }

      this.$copyText(this.customSubUrl)
      this.$message.success('定制订阅已复制到剪贴板')
    },
    makeShortUrl() {
      if (this.customSubUrl === '') {
        this.$message.warning('请先生成订阅链接，再获取对应短链接')
        return false
      }

      this.loading = true

      const data = new FormData()
      data.append('longUrl', btoa(this.customSubUrl))

      this.$axios
        .post(shortUrlBackend, data, {
          header: {
            'Content-Type': 'application/form-data; charset=utf-8'
          }
        })
        .then(res => {
          if (res.data.Code === 1 && res.data.ShortUrl !== '') {
            this.curtomShortSubUrl = res.data.ShortUrl
            this.$copyText(res.data.ShortUrl)
            this.$message.success('短链接已复制到剪贴板')
          } else {
            this.$message.error('短链接获取失败：' + res.data.Message)
          }
        })
        .catch(() => {
          this.$message.error('短链接获取失败')
        })
        .finally(() => {
          this.loading = false
        })
    },
    notify() {
      const h = this.$createElement

      this.$notify({
        title: '隐私提示',
        type: 'warning',
        message: h(
          'i',
          { style: 'color: teal' },
          '各种订阅链接（短链接服务除外）生成纯前端实现，无隐私问题。默认提供后端转换服务，隐私担忧者请自行搭建后端服务。'
        )
      })
    },
    confirmUploadConfig() {
      if (this.uploadConfig === '') {
        this.$message.warning('远程配置不能为空')
        return false
      }

      this.loading = true

      const data = new FormData()
      data.append('password', this.uploadPassword)
      data.append('config', this.uploadConfig)

      this.$axios
        .post(configUploadBackend, data, {
          header: {
            'Content-Type': 'application/form-data; charset=utf-8'
          }
        })
        .then(res => {
          if (res.data.Code === 1 && res.data.url !== '') {
            this.$message.success(
              '远程配置上传成功，配置链接已复制到剪贴板，有效期三个月望知悉'
            )

            // 自动填充至『表单-远程配置』
            this.form.remoteConfig = res.data.Url
            this.$copyText(this.form.remoteConfig)

            this.dialogUploadConfigVisible = false
          } else {
            this.$message.error('远程配置上传失败：' + res.data.Message)
          }
        })
        .catch(() => {
          this.$message.error('远程配置上传失败')
        })
        .finally(() => {
          this.loading = false
        })
    },
    backendSearch(queryString, cb) {
      const backends = this.options.backendOptions

      const results = queryString
        ? backends.filter(this.createFilter(queryString))
        : backends

      // 调用 callback 返回建议列表的数据
      cb(results)
    },
    createFilter(queryString) {
      return candidate => {
        return (
          candidate.value.toLowerCase().indexOf(queryString.toLowerCase()) === 0
        )
      }
    },
    getBackendVersion() {
      this.request()
      this.$axios.get(defaultBackend.substring(0, defaultBackend.length - 5) + '/version').then(res => {
        this.backendVersion = res.data.replace(/backend\n$/gm, '')
        this.backendVersion = this.backendVersion.replace('subconverter', '')
      })
    }
  }
}
</script>

<style scoped>

</style>
