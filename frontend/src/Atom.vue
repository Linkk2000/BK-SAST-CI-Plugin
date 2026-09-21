<template>
    <section class="bk-form bk-form-vertical atom-form">
        <!-- 第一部分：服务器信息 -->
        <div class="form-section">
            <h3 class="section-title">服务器信息</h3>
            
            <!-- SAST服务器地址 -->
            <div class="form-group" :class="{ 'has-error': fieldErrors.server.show }">
                <label class="form-label required">SAST服务器地址</label>
                <input 
                    type="text" 
                    class="form-input"
                    v-model="sastTask.server"
                    @blur="handleBlur('server')"
                    @input="updateTaskField('server', $event.target.value)"
                    placeholder="请输入SAST服务器地址"
                    :disabled="atomPropsDisabled"
                />
                <div class="error-message" v-if="fieldErrors.server.show">
                    {{ fieldErrors.server.message }}
                </div>
            </div>

            <!-- Token -->
            <div class="form-group" :class="{ 'has-error': fieldErrors.token.show }">
                <label class="form-label required">Token</label>
                <input 
                    type="password" 
                    class="form-input"
                    v-model="sastTask.token"
                    @blur="handleBlur('token')"
                    @input="updateTaskField('token', $event.target.value)"
                    placeholder="请输入Token"
                    :disabled="atomPropsDisabled"
                />
                <div class="error-message" v-if="fieldErrors.token.show">
                    {{ fieldErrors.token.message }}
                </div>
            </div>

            <!-- 测试连接按钮 -->
            <div class="test-connection-wrapper">
                <button 
                    class="test-btn"
                    :class="{ 'is-loading': isLoading }"
                    @click="testConnection"
                    :disabled="atomPropsDisabled || isLoading"
                >
                    <span v-if="isLoading">测试中...</span>
                    <span v-else>测试连接</span>
                </button>
                
                <transition name="fade">
                    <div class="test-result success" v-if="testResult.show && testResult.type === 'success'">
                        <span class="message">连接成功</span>
                    </div>
                </transition>
                
                <transition name="fade">
                    <div class="test-result error" v-if="testResult.show && testResult.type === 'error'">
                        <span class="message">{{ testResult.message }}</span>
                    </div>
                </transition>
            </div>
        </div>

        <!-- 第二部分：扫描任务信息 -->
        <div class="form-section">
            <h3 class="section-title">扫描任务信息</h3>
            
            <!-- 项目选择 -->
            <div class="form-group" :class="{ 'has-error': fieldErrors.projectName.show }">
                <label class="form-label required">项目</label>
                <bk-select 
                    v-model="sastTask.projectId"
                    :searchable="true"
                    :loading="projectLoading"
                    :disabled="atomPropsDisabled || !sastTask.server || !sastTask.token"
                    placeholder="请选择或搜索项目"
                    @change="handleProjectChange"
                    @clear="resetProjectData"
                    @toggle="handleProjectToggle"
                >
                    <bk-option 
                        v-for="project in projectList" 
                        :key="project.id" 
                        :id="project.id" 
                        :name="project.name"
                    >
                    </bk-option>
                </bk-select>
                <div class="error-message" v-if="fieldErrors.projectName.show">
                    {{ fieldErrors.projectName.message }}
                </div>
                <div class="field-tip" v-if="!sastTask.server || !sastTask.token">
                    请先完成服务器信息配置并测试连接
                </div>
            </div>

            <!-- 应用选择 -->
            <div class="form-group" :class="{ 'has-error': fieldErrors.appName.show }">
                <label class="form-label required">应用</label>
                <bk-select 
                    v-model="sastTask.appId"
                    :searchable="true"
                    :loading="appLoading"
                    :disabled="atomPropsDisabled || !sastTask.projectId"
                    placeholder="请选择或搜索应用"
                    @change="handleAppChange"
                    @clear="resetAppData"
                    @toggle="handleAppToggle"
                >
                    <bk-option 
                        v-for="app in appList" 
                        :key="app.id" 
                        :id="app.id" 
                        :name="app.name"
                    >
                    </bk-option>
                </bk-select>
                <div class="error-message" v-if="fieldErrors.appName.show">
                    {{ fieldErrors.appName.message }}
                </div>
                <div class="field-tip" v-if="!sastTask.projectId">
                    请先选择关联项目
                </div>
            </div>
        </div>

        <!-- 底部操作栏 -->
        <div class="form-actions">
            <button 
                class="save-btn" 
                @click="saveConfiguration"
                :disabled="atomPropsDisabled"
            >
                保存配置
            </button>
            <transition name="fade">
                <div class="save-status" v-if="saveStatus.show">
                    {{ saveStatus.message }}
                </div>
            </transition>
        </div>
    </section>
</template>

<script>
    // 需引用atomMixin
    import { atomMixin } from 'bkci-atom-components'
    import { mockAjax } from '@/utils/mock'

    export default {
        name: 'atom',
        mixins: [atomMixin],    // 需引用atomMixin
        props: {
            atomPropsContainerInfo: {
                type: Object,
                default: () => ({})
            },
            atomPropsDisabled: {
                type: Boolean,
                default: false
            },
            currentUserInfo: {
                type: Object,
                default: () => ({})
            },
            envConf: {
                type: Object,
                default: () => ({})
            }
        },
        data() {
            return {
                // 核心：本地数据副本，所有 UI 绑定均基于此
                sastTask: {
                    server: '',
                    token: '',
                    projectId: '',
                    projectName: '',
                    appId: '',
                    appName: ''
                },
                // 校验状态保持独立
                fieldErrors: {
                    server: { show: false, message: '' },
                    token: { show: false, message: '' },
                    projectName: { show: false, message: '' },
                    appName: { show: false, message: '' }
                },
                touched: {
                    server: false,
                    token: false,
                    projectName: false,
                    appName: false
                },
                isLoading: false,
                testResult: { show: false, type: '', message: '' },
                saveStatus: { show: false, message: '' },
                successTimer: null,
                projectList: [],
                projectLoading: false,
                projectSearchKeyword: '',
                appList: [],
                appLoading: false,
                appSearchKeyword: ''
            }
        },
        computed: {
            isLocalDev() {
                // 读取 main.js 中设置的环境标识
                return window.__ATOM_ENV__ && window.__ATOM_ENV__.isLocal === true
            },
            useMock() {
                return this.isLocalDev
            },
            ajax() {
                return this.useMock ? mockAjax : this.$ajax
            }
        },
        mounted() {
            console.log('Atom component mounted')
            this.initSastTask()
            
            // 暴露组件实例到全局，方便调试
            window.__ATOM_INSTANCE__ = this
            console.log('%c[调试提示] 组件实例已暴露到 window.__ATOM_INSTANCE__', 'color: #2dcb56; font-weight: bold;')
            console.log('可在控制台使用: __ATOM_INSTANCE__.sastTask 或 __ATOM_INSTANCE__.atomValue')
        },
        methods: {
            // 统一规范化 server 输入，避免出现 http://https//xxx 这类错误 URL
            // 约束：必须显式以 http:// 或 https:// 开头；否则判定为不合法（不做任何自动修复/补全）
            normalizeServerUrl(rawServer) {
                // 打印url
                console.log('url', rawServer)
                let server = (rawServer || '').trim()
                if (!server) return ''

                // 不使用正则，严格要求协议前缀
                const hasValidPrefix = server.startsWith('http://') || server.startsWith('https://')
                if (!hasValidPrefix) return ''
                // 说明url合格
                console.log("Url is effective.")
                // 去掉末尾 /
                if (server.endsWith('/')) server = server.substring(0, server.length - 1)
                return server
            },

            // 封装一：初始化逻辑
            initSastTask() {
                console.log('%c[初始化] 开始初始化 sastTask', 'color: #ff9800; font-weight: bold;')
                console.log('[初始化] atomValue 原始数据:', JSON.parse(JSON.stringify(this.atomValue)))
                
                if (this.atomValue) {
                    // 深拷贝平台数据到本地副本
                    const platformData = JSON.parse(JSON.stringify(this.atomValue))
                    Object.keys(this.sastTask).forEach(key => {
                        this.sastTask[key] = platformData[key] || ''
                    })
                    
                    console.log('[初始化] 拷贝后的 sastTask:', JSON.parse(JSON.stringify(this.sastTask)))

                    // 初始化回显列表
                    if (this.sastTask.server && this.sastTask.token) {
                        console.log('[初始化] 准备获取项目列表...')
                        this.fetchProjectList()
                        if (this.sastTask.projectId) {
                            console.log('[初始化] 准备获取应用列表...')
                            this.fetchAppList()
                        }
                    }
                }
                
                console.log('%c[初始化] 完成', 'color: #ff9800; font-weight: bold;')
            },

            // 封装二：统一字段更新入口（替代 watch）
            updateTaskField(field, value) {
                this.sastTask[field] = value
                
                // 显式联动逻辑
                if (field === 'server' || field === 'token') {
                    this.clearTestResult()
                    this.resetProjectData() 
                } else if (field === 'projectId') {
                    this.resetAppData()
                }

                if (this.touched[field]) {
                    this.validateField(field)
                }
            },

            // 封装三：同步回平台
            syncToPlatform() {
                Object.keys(this.sastTask).forEach(key => {
                    this.$set(this.atomValue, key, this.sastTask[key])
                })
            },

            // 重置项目及以下所有数据
            resetProjectData() {
                console.warn('%c[重置] resetProjectData() 被调用', 'color: #f44336; font-weight: bold;')
                console.trace('[重置] 调用堆栈')
                this.sastTask.projectId = ''
                this.sastTask.projectName = ''
                this.projectList = []
                this.resetAppData()
            },
            
            // 重置应用数据
            resetAppData() {
                console.warn('%c[重置] resetAppData() 被调用', 'color: #f44336; font-weight: bold;')
                console.trace('[重置] 调用堆栈')
                this.sastTask.appId = ''
                this.sastTask.appName = ''
                this.appList = []
            },
            // 验证单个字段
            validateField(fieldName) {
                const value = this.sastTask[fieldName]

                if (fieldName === 'server') {
                    const server = (value || '').trim()
                    if (!server) {
                        this.fieldErrors.server = { show: true, message: '字段不能为空' }
                        return false
                    }
                    const isValid = server.startsWith('http://') || server.startsWith('https://')
                    if (!isValid) {
                        this.fieldErrors.server = { show: true, message: '服务器地址必须以 http:// 或 https:// 开头' }
                        return false
                    }
                    this.fieldErrors.server = { show: false, message: '' }
                    return true
                }
                
                if (fieldName === 'projectName') {
                    if (!this.sastTask.projectId) {
                        this.fieldErrors.projectName = { show: true, message: '请选择项目' }
                        return false
                    }
                    this.fieldErrors.projectName = { show: false, message: '' }
                    return true
                }
                
                if (fieldName === 'appName') {
                    if (!this.sastTask.appId) {
                        this.fieldErrors.appName = { show: true, message: '请选择应用' }
                        return false
                    }
                    this.fieldErrors.appName = { show: false, message: '' }
                    return true
                }
                
                if (!value || (typeof value === 'string' && value.trim() === '')) {
                    this.fieldErrors[fieldName] = { show: true, message: '字段不能为空' }
                    return false
                }
                this.fieldErrors[fieldName] = { show: false, message: '' }
                return true
            },
            
            // 失焦验证
            handleBlur(fieldName) {
                this.touched[fieldName] = true
                this.validateField(fieldName)
            },
            
            // 验证所有字段
            validateAll(showErrors = true) {
                let isValid = true
                // 仅验证我们需要同步到平台的字段
                const fieldsToValidate = ['server', 'token', 'projectName', 'appName']
                fieldsToValidate.forEach(key => {
                    const fieldValid = this.checkFieldValid(key)
                    if (!fieldValid) {
                        isValid = false
                    }
                    if (showErrors && this.fieldErrors[key]) {
                        this.fieldErrors[key].show = !fieldValid
                    }
                })
                return isValid
            },

            // 内部纯校验逻辑
            checkFieldValid(fieldName) {
                const value = this.sastTask[fieldName]
                if (fieldName === 'projectName') return !!this.sastTask.projectId
                if (fieldName === 'appName') return !!this.sastTask.appId
                return !!(value && typeof value === 'string' && value.trim() !== '')
            },
            
            // 清除测试结果
            clearTestResult() {
                this.testResult.show = false
                this.testResult.type = ''
                this.testResult.message = ''
                if (this.successTimer) {
                    clearTimeout(this.successTimer)
                    this.successTimer = null
                }
            },
            
            // 测试连接
            async testConnection() {
                const isServerValid = this.validateField('server')
                const isTokenValid = this.validateField('token')
                
                if (!isServerValid || !isTokenValid) return
                
                this.isLoading = true
                this.clearTestResult()
                
                try {
                    const rawServer = this.sastTask.server
                    const server = this.normalizeServerUrl(rawServer)
                    if (!server) throw new Error('SAST服务器地址不能为空')
                    
                    const apiPath = '/sast/api-v1/open-api/system/user/token/connect'
                    const token = this.sastTask.token.trim()
                    const url = `${server}${apiPath}?token=${encodeURIComponent(token)}`

                    console.warn('[SAST][testConnection] rawServer=', rawServer)
                    console.warn('[SAST][testConnection] normalizedServer=', server)
                    console.warn('[SAST][testConnection] url=', url)
                    
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token
                        },
                        timeout: 10000
                    })
                    
                    if (response && response.code === 0 && response.data && response.data.duration === "1") {
                        this.testResult = { show: true, type: 'success', message: '连接成功！' }
                        this.successTimer = setTimeout(() => this.clearTestResult(), 3000)
                    } else {
                        throw new Error('连接测试失败：响应异常')
                    }
                } catch (error) {
                    this.testResult = { show: true, type: 'error', message: error.message || '连接失败' }
                } finally {
                    this.isLoading = false
                }
            },
            
            // 获取项目列表
            async fetchProjectList(keyword = '') {
                console.log('%c[获取项目] fetchProjectList() 开始', 'color: #2196f3; font-weight: bold;')
                console.log('[获取项目] 当前 sastTask.projectId=', this.sastTask.projectId)
                console.log('[获取项目] 当前 sastTask.appId=', this.sastTask.appId)
                
                const isServerValid = this.validateField('server')
                const isTokenValid = this.validateField('token')
                if (!isServerValid || !isTokenValid) {
                    console.warn('[获取项目] 校验失败，退出')
                    return
                }
                
                this.projectLoading = true
                try {
                    const rawServer = this.sastTask.server
                    const server = this.normalizeServerUrl(rawServer)
                    if (!server) return
                    const url = `${server}/sast/api-v1/open-api/project/page?contParam=${encodeURIComponent(keyword)}&pageSize=20`

                    console.warn('[SAST][fetchProjectList] rawServer=', rawServer)
                    console.warn('[SAST][fetchProjectList] normalizedServer=', server)
                    console.warn('[SAST][fetchProjectList] url=', url)
                    
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: { 'Sast-Token': this.sastTask.token.trim() }
                    })
                    
                    if (response && response.code === 0 && response.data && response.data.records) {
                        this.projectList = response.data.records.map(item => ({
                            id: item.projectId,
                            name: item.projectName
                        }))
                        console.log('[获取项目] 成功获取项目列表，数量:', this.projectList.length)
                        console.log('[获取项目] 完成后 sastTask.projectId=', this.sastTask.projectId)
                        console.log('[获取项目] 完成后 sastTask.appId=', this.sastTask.appId)
                    }
                } catch (error) {
                    console.error('[获取项目] 失败:', error)
                } finally {
                    this.projectLoading = false
                }
            },
            
            // 获取应用列表
            async fetchAppList(keyword = '') {
                console.log('%c[获取应用] fetchAppList() 开始', 'color: #2196f3; font-weight: bold;')
                console.log('[获取应用] 当前 sastTask.projectId=', this.sastTask.projectId)
                console.log('[获取应用] 当前 sastTask.appId=', this.sastTask.appId)
                
                const isServerValid = this.validateField('server')
                const isTokenValid = this.validateField('token')
                if (!isServerValid || !isTokenValid || !this.sastTask.projectId) {
                    console.warn('[获取应用] 校验失败或 projectId 为空，退出')
                    return
                }
                
                this.appLoading = true
                try {
                    const rawServer = this.sastTask.server
                    const server = this.normalizeServerUrl(rawServer)
                    if (!server) return
                    const url = `${server}/sast/api-v1/app/info/${this.sastTask.projectId}`

                    console.warn('[SAST][fetchAppList] rawServer=', rawServer)
                    console.warn('[SAST][fetchAppList] normalizedServer=', server)
                    console.warn('[SAST][fetchAppList] url=', url)
                    
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: { 'Sast-Token': this.sastTask.token.trim() }
                    })
                    
                    if (response && response.code === 0 && response.data && response.data.records) {
                        this.appList = response.data.records.map(item => ({
                            id: item.appId,
                            name: item.appName
                        }))
                        console.log('[获取应用] 成功获取应用列表，数量:', this.appList.length)
                        console.log('[获取应用] 完成后 sastTask.appId=', this.sastTask.appId)
                    }
                } catch (error) {
                    console.error('[获取应用] 失败:', error)
                } finally {
                    this.appLoading = false
                }
            },
            
            // 项目选择变化
            handleProjectChange(projectId) {
                console.log('%c[事件] handleProjectChange 被触发', 'color: #9c27b0; font-weight: bold;')
                console.log('[事件] projectId=', projectId)
                console.log('[事件] projectList.length=', this.projectList.length)
                
                const project = this.projectList.find(p => p.id === projectId)
                if (project) {
                    console.log('[事件] 找到项目:', project.name)
                    this.updateTaskField('projectId', project.id)
                    this.updateTaskField('projectName', project.name)
                    this.fetchAppList()
                } else {
                    // 🔥 修复：只有在列表不为空时才重置（避免初始化时因列表未加载而误清空）
                    if (this.projectList.length > 0) {
                        console.warn('[事件] 项目列表不为空但未找到匹配项，重置数据')
                        this.resetProjectData()
                    } else {
                        console.log('[事件] 项目列表为空，跳过重置（可能是初始化中）')
                    }
                }
            },
            
            // 应用选择变化
            handleAppChange(appId) {
                console.log('%c[事件] handleAppChange 被触发', 'color: #9c27b0; font-weight: bold;')
                console.log('[事件] appId=', appId)
                console.log('[事件] appList.length=', this.appList.length)
                
                const app = this.appList.find(a => a.id === appId)
                if (app) {
                    console.log('[事件] 找到应用:', app.name)
                    this.updateTaskField('appId', app.id)
                    this.updateTaskField('appName', app.name)
                } else {
                    // 🔥 修复：只有在列表不为空时才重置（避免初始化时因列表未加载而误清空）
                    if (this.appList.length > 0) {
                        console.warn('[事件] 应用列表不为空但未找到匹配项，重置数据')
                        this.resetAppData()
                    } else {
                        console.log('[事件] 应用列表为空，跳过重置（可能是初始化中）')
                    }
                }
            },

            handleProjectToggle(isOpen) {
                if (isOpen && this.projectList.length === 0) this.fetchProjectList()
                if (!isOpen) {
                    this.touched.projectName = true
                    this.validateField('projectName')
                }
            },

            handleAppToggle(isOpen) {
                if (isOpen && this.appList.length === 0 && this.sastTask.projectId) this.fetchAppList()
                if (!isOpen) {
                    this.touched.appName = true
                    this.validateField('appName')
                }
            },
            
            // 项目搜索
            handleProjectSearch(keyword) {
                this.projectSearchKeyword = keyword
                this.fetchProjectList(keyword)
            },
            
            // 应用搜索
            handleAppSearch(keyword) {
                this.appSearchKeyword = keyword
                this.fetchAppList(keyword)
            },

            // 保存配置
            saveConfiguration() {
                const isValid = this.validateAll()
                if (isValid) {
                    this.syncToPlatform() // 显式同步到 atomValue
                    this.setAtomIsError(false)
                    this.saveStatus = { show: true, message: '保存成功' }
                } else {
                    this.setAtomIsError(true)
                    this.saveStatus = { show: true, message: '请完善必填信息' }
                }
                setTimeout(() => this.saveStatus.show = false, 3000)
            }
        },
        beforeDestroy() {
            if (this.successTimer) {
                clearTimeout(this.successTimer)
            }
            
            // 清理全局引用
            if (window.__ATOM_INSTANCE__ === this) {
                window.__ATOM_INSTANCE__ = null
            }
            
            // 💡 只有在点击保存按钮时才同步到 atomValue
            // 侧边栏关闭时仅回传当前的校验状态，不强制覆盖数据，保护已保存的数据不被中间态破坏
            const isFinalValid = this.validateAll(false)
            this.setAtomIsError(!isFinalValid)
            
            console.log('[BKCI-ATOM] Cleanup completed. Valid:', isFinalValid)
        }
    }
</script>

<style lang="scss" scoped>
    .atom-form {
        padding: 10px 0;

        .form-section {
            background: #fff;
            border: 1px solid #dcdee5;
            border-radius: 2px;
            padding: 20px;
            margin-bottom: 20px;

            .section-title {
                margin: 0 0 20px 0;
                padding-bottom: 10px;
                border-bottom: 1px solid #f0f1f5;
                font-size: 16px;
                color: #313238;
                font-weight: bold;
            }
        }
        
        .form-group {
            margin-bottom: 20px;
            
            &:last-child {
                margin-bottom: 0;
            }
            
            &.has-error {
                .form-input,
                .bk-select {
                    border-color: #ff5656 !important;
                    
                    &:focus,
                    &.is-focus {
                        border-color: #ff5656 !important;
                        box-shadow: 0 0 0 2px rgba(255, 86, 86, 0.1);
                    }
                }
            }
        }

        .form-actions {
            margin-top: 30px;
            padding: 20px 0;
            border-top: 1px solid #f0f1f5;
            display: flex;
            align-items: center;
            gap: 15px;

            .save-btn {
                height: 40px;
                padding: 0 40px;
                font-size: 14px;
                color: #fff;
                background-color: #3a84ff;
                border: none;
                border-radius: 2px;
                cursor: pointer;
                font-weight: bold;
                transition: background-color 0.2s;
                
                &:hover:not(:disabled) {
                    background-color: #4e94ff;
                }
                
                &:disabled {
                    background-color: #dcdee5;
                    cursor: not-allowed;
                }
            }

            .save-status {
                font-size: 14px;
                color: #63656e;
                animation: fadeIn 0.3s ease-in;
            }
        }
        
        .form-label {
            display: block;
            margin-bottom: 8px;
            font-size: 14px;
            color: #63656e;
            font-weight: 500;
            
            &.required::before {
                content: '*';
                color: #ff5656;
                margin-right: 4px;
            }
        }
        
        .form-input,
        .form-select {
            width: 100%;
            height: 36px;
            padding: 0 12px;
            font-size: 14px;
            line-height: 36px;
            color: #63656e;
            background-color: #fff;
            border: 1px solid #c4c6cc;
            border-radius: 2px;
            outline: none;
            transition: border-color 0.2s, box-shadow 0.2s;
            
            &:hover {
                border-color: #979ba5;
            }
            
            &:focus {
                border-color: #3a84ff;
                box-shadow: 0 0 0 2px rgba(58, 132, 255, 0.1);
            }
            
            &:disabled {
                background-color: #fafbfd;
                color: #c4c6cc;
                cursor: not-allowed;
            }
            
            &::placeholder {
                color: #c4c6cc;
            }
        }
        
        .form-select {
            cursor: pointer;
            padding-right: 30px;
            background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%2363656e' d='M6 9L1 4h10z'/%3E%3C/svg%3E");
            background-repeat: no-repeat;
            background-position: right 10px center;
            background-size: 12px;
            appearance: none;
            
            &:disabled {
                background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23c4c6cc' d='M6 9L1 4h10z'/%3E%3C/svg%3E");
            }
        }
        
        .field-tip {
            margin-top: 6px;
            font-size: 12px;
            color: #979ba5;
            line-height: 1.5;
        }
        
        .error-message {
            margin-top: 6px;
            font-size: 12px;
            color: #ff5656;
            line-height: 1.5;
        }
        
        .test-connection-wrapper {
            margin-top: 24px;
            display: flex;
            align-items: center;
            gap: 12px;
            
            .test-btn {
                height: 36px;
                padding: 0 24px;
                font-size: 14px;
                color: #fff;
                background-color: #3a84ff;
                border: none;
                border-radius: 2px;
                cursor: pointer;
                outline: none;
                transition: background-color 0.2s;
                
                &:hover:not(:disabled) {
                    background-color: #4e94ff;
                }
                
                &:active:not(:disabled) {
                    background-color: #2c6be6;
                }
                
                &:disabled {
                    background-color: #dcdee5;
                    cursor: not-allowed;
                }
                
                &.is-loading {
                    background-color: #699df4;
                }
            }
            
            .test-result {
                display: flex;
                align-items: center;
                gap: 6px;
                font-size: 14px;
                animation: fadeIn 0.3s ease-in;
                
                &.success {
                    color: #2dcb56;
                }
                
                &.error {
                    color: #ff5656;
                }
            }
        }
    }
    
    // 淡入淡出动画
    .fade-enter-active, .fade-leave-active {
        transition: opacity 0.3s ease;
    }
    
    .fade-enter, .fade-leave-to {
        opacity: 0;
    }
    
    @keyframes fadeIn {
        from {
            opacity: 0;
            transform: translateX(-10px);
        }
        to {
            opacity: 1;
            transform: translateX(0);
        }
    }
</style>
