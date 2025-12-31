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
                    v-model="formData.server"
                    @blur="handleBlur('server')"
                    @input="handleInput('server')"
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
                    v-model="formData.token"
                    @blur="handleBlur('token')"
                    @input="handleInput('token')"
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
                    v-model="formData.projectId"
                    :searchable="true"
                    :loading="projectLoading"
                    :disabled="atomPropsDisabled || !formData.server || !formData.token"
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
                <div class="field-tip" v-if="!formData.server || !formData.token">
                    请先完成服务器信息配置并测试连接
                </div>
            </div>

            <!-- 应用选择 -->
            <div class="form-group" :class="{ 'has-error': fieldErrors.appName.show }">
                <label class="form-label required">应用</label>
                <bk-select 
                    v-model="formData.appId"
                    :searchable="true"
                    :loading="appLoading"
                    :disabled="atomPropsDisabled || !formData.projectId"
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
                <div class="field-tip" v-if="!formData.projectId">
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
                formData: {
                    server: '',
                    token: '',
                    projectId: '',
                    projectName: '',
                    appId: '',
                    appName: ''
                },
                fieldErrors: {
                    server: {
                        show: false,
                        message: ''
                    },
                    token: {
                        show: false,
                        message: ''
                    },
                    projectName: {
                        show: false,
                        message: ''
                    },
                    appName: {
                        show: false,
                        message: ''
                    }
                },
                touched: {
                    server: false,
                    token: false,
                    projectName: false,
                    appName: false
                },
                isLoading: false,
                testResult: {
                    show: false,
                    type: '', // 'success' or 'error'
                    message: ''
                },
                saveStatus: {
                    show: false,
                    message: ''
                },
                successTimer: null,
                // 项目相关
                projectList: [],
                projectLoading: false,
                projectSearchKeyword: '',
                // 应用相关
                appList: [],
                appLoading: false,
                appSearchKeyword: ''
            }
        },
        computed: {
            // 判断是否为本地开发环境
            isLocalDev() {
                return typeof ISLOCAL !== 'undefined' && ISLOCAL === true
            },
            // 是否启用 Mock 数据（本地开发时自动启用）
            useMock() {
                return this.isLocalDev
            },
            // 决定使用真实 Ajax 还是 Mock Ajax
            ajax() {
                return this.useMock ? mockAjax : this.$ajax
            }
        },
        created() {
            if (this.useMock) {
                console.warn('[BKCI-ATOM] MOCK MODE ENABLED')
            }
        },
        mounted() {
            console.log('Atom component mounted')
            
            // 从 atomValue 中初始化表单数据（仅作为本地备份操作）
            if (this.atomValue) {
                // 1. 将平台数据备份到本地 formData
                this.formData.server = this.atomValue.server || ''
                this.formData.token = this.atomValue.token || ''
                this.formData.projectId = this.atomValue.projectId || ''
                this.formData.projectName = this.atomValue.projectName || ''
                this.formData.appId = this.atomValue.appId || ''
                this.formData.appName = this.atomValue.appName || ''

                // 2. 核心：如果已有配置，拉取列表以供回显
                if (this.formData.server && this.formData.token) {
                    this.fetchProjectList()
                    if (this.formData.projectId) {
                        this.fetchAppList()
                    }
                }
            }
        },
        watch: {
            'formData.server'(newVal) {
                this.clearTestResult()
                // 仅操作本地重置，不污染 atomValue
                this.resetProjectData()
            },
            'formData.token'(newVal) {
                this.clearTestResult()
                this.resetProjectData()
            },
            'formData.projectId'(newVal) {
                // 仅操作本地重置
                this.resetAppData()
            }
        },
        methods: {
            // 重置项目及以下所有数据
            resetProjectData() {
                this.formData.projectId = ''
                this.formData.projectName = ''
                this.projectList = []
                this.resetAppData()
            },
            
            // 重置应用数据
            resetAppData() {
                this.formData.appId = ''
                this.formData.appName = ''
                this.appList = []
            },
            // 验证单个字段
            validateField(fieldName) {
                const value = this.formData[fieldName]
                
                // 特殊处理：projectName 和 appName 的验证基于对应的 ID
                if (fieldName === 'projectName') {
                    if (!this.formData.projectId) {
                        this.fieldErrors.projectName = {
                            show: true,
                            message: '请选择项目'
                        }
                        return false
                    } else {
                        this.fieldErrors.projectName = {
                            show: false,
                            message: ''
                        }
                        return true
                    }
                }
                
                if (fieldName === 'appName') {
                    if (!this.formData.appId) {
                        this.fieldErrors.appName = {
                            show: true,
                            message: '请选择应用'
                        }
                        return false
                    } else {
                        this.fieldErrors.appName = {
                            show: false,
                            message: ''
                        }
                        return true
                    }
                }
                
                if (!value || value.trim() === '') {
                    this.fieldErrors[fieldName] = {
                        show: true,
                        message: '字段不能为空'
                    }
                    return false
                } else {
                    this.fieldErrors[fieldName] = {
                        show: false,
                        message: ''
                    }
                    return true
                }
            },
            
            // 失焦验证
            handleBlur(fieldName) {
                this.touched[fieldName] = true
                this.validateField(fieldName)
            },
            
            // 输入时验证
            handleInput(fieldName) {
                if (this.touched[fieldName]) {
                    this.validateField(fieldName)
                }
            },
            
            // 验证所有字段
            validateAll(showErrors = true) {
                let isValid = true
                Object.keys(this.formData).forEach(key => {
                    const fieldValid = this.checkFieldValid(key)
                    if (!fieldValid) {
                        isValid = false
                    }
                    // 🚨 安全检查：只有在 fieldErrors 对象中存在的字段才进行 UI 状态更新
                    if (showErrors && this.fieldErrors[key]) {
                        this.fieldErrors[key].show = !fieldValid
                    }
                })
                return isValid
            },

            // 内部纯校验逻辑（不操作 UI）
            checkFieldValid(fieldName) {
                const value = this.formData[fieldName]
                if (fieldName === 'projectName') return !!this.formData.projectId
                if (fieldName === 'appName') return !!this.formData.appId
                return !!(value && value.trim() !== '')
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
                // 仅验证服务器和 Token
                const isServerValid = this.validateField('server')
                const isTokenValid = this.validateField('token')
                
                if (!isServerValid || !isTokenValid) {
                    return
                }
                
                this.isLoading = true
                this.clearTestResult()
                
                try {
                    // 处理 server 地址
                    let server = this.formData.server.trim()
                    
                    // 自动添加 http:// 前缀
                    if (!server.startsWith('http://') && !server.startsWith('https://')) {
                        server = 'http://' + server
                    }
                    
                    // 移除末尾的 /
                    if (server.endsWith('/')) {
                        server = server.substring(0, server.length - 1)
                    }
                    
                    // 构建完整 URL（与后端保持一致）
                    const apiPath = '/sast/api-v1/open-api/system/user/token/connect'
                    const token = this.formData.token.trim()
                    const url = `${server}${apiPath}?token=${encodeURIComponent(token)}`
                    
                    console.log('Testing connection to:', url)
                    
                    // 发送 GET 请求，Header 也带上 token（与后端保持一致）
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token,
                            'User-Agent': 'bkci-custom-atom-frontend/1.0'
                        },
                        timeout: 10000 // 10秒超时
                    })
                    
                    // 检查响应中的 duration 字段
                    console.log('Connection test response:', response)
                    
                    if (response && response.code === 0 && response.data && response.data.duration === "1") {
                        // duration === "1" 表示连接成功
                        this.testResult = {
                            show: true,
                            type: 'success',
                            message: '连接成功！'
                        }
                        
                        // 3秒后自动隐藏成功提示
                        this.successTimer = setTimeout(() => {
                            this.clearTestResult()
                        }, 3000)
                    } else {
                        // duration 不为 1，视为失败
                        throw new Error('连接测试失败：duration 字段校验不通过')
                    }
                    
                } catch (error) {
                    // 连接失败
                    console.error('Connection test failed:', error)
                    
                    let errorMessage = '连接失败，请检查服务器地址和Token是否正确'
                    
                    if (error.message) {
                        errorMessage = error.message
                    } else if (error.response) {
                        const status = error.response.status
                        const data = error.response.data
                        
                        if (data && data.message) {
                            errorMessage = `连接失败 (${status}): ${data.message}`
                        } else {
                            errorMessage = `连接失败，HTTP 状态码: ${status}`
                        }
                    }
                    
                    this.testResult = {
                        show: true,
                        type: 'error',
                        message: errorMessage
                    }
                } finally {
                    this.isLoading = false
                }
            },
            
            // 当用户输入相关参数后，把字段写入到this.atomValue
            handleUpdate(name, value) {
                this.atomValue[name] = value
            },
            
            // 获取项目列表
            async fetchProjectList(keyword = '') {
                if (!this.formData.server || !this.formData.token) {
                    console.warn('Server or token not set, cannot fetch project list')
                    return
                }
                
                this.projectLoading = true
                try {
                    let server = this.formData.server.trim()
                    if (!server.startsWith('http://') && !server.startsWith('https://')) {
                        server = 'http://' + server
                    }
                    if (server.endsWith('/')) {
                        server = server.substring(0, server.length - 1)
                    }
                    
                    const token = this.formData.token.trim()
                    const url = `${server}/sast/api-v1/open-api/project/page?contParam=${encodeURIComponent(keyword)}&roleId=&status=&sort=&order=&pageNum=1&pageSize=20`
                    
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token
                        },
                        timeout: 10000
                    })
                    
                    if (response && response.code === 0 && response.data && response.data.records) {
                        this.projectList = response.data.records.map(item => ({
                            id: item.projectId,
                            name: item.projectName
                        }))
                    } else {
                        this.projectList = []
                    }
                } catch (error) {
                    console.error('Failed to fetch project list:', error)
                    this.projectList = []
                } finally {
                    this.projectLoading = false
                }
            },
            
            // 获取应用列表
            async fetchAppList(keyword = '') {
                if (!this.formData.server || !this.formData.token || !this.formData.projectId) {
                    console.warn('Server, token or projectId not set, cannot fetch app list')
                    return
                }
                
                this.appLoading = true
                try {
                    let server = this.formData.server.trim()
                    if (!server.startsWith('http://') && !server.startsWith('https://')) {
                        server = 'http://' + server
                    }
                    if (server.endsWith('/')) {
                        server = server.substring(0, server.length - 1)
                    }
                    
                    const token = this.formData.token.trim()
                    const projectId = this.formData.projectId
                    const url = `${server}/sast/api-v1/app/info/${projectId}?sort=&order=&projectId=${projectId}&pageNum=1&pageSize=20`
                    
                    const response = await this.ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token
                        },
                        timeout: 10000
                    })
                    
                    if (response && response.code === 0 && response.data && response.data.records) {
                        let apps = response.data.records
                        // 如果有搜索关键字，进行前端过滤
                        if (keyword) {
                            apps = apps.filter(item => item.appName && item.appName.toLowerCase().includes(keyword.toLowerCase()))
                        }
                        this.appList = apps.map(item => ({
                            id: item.appId,
                            name: item.appName
                        }))
                    } else {
                        this.appList = []
                    }
                } catch (error) {
                    console.error('Failed to fetch app list:', error)
                    this.appList = []
                } finally {
                    this.appLoading = false
                }
            },
            
            // 项目选择变化
            handleProjectChange(projectId) {
                const project = this.projectList.find(p => p.id === projectId)
                if (project) {
                    this.formData.projectId = project.id
                    this.formData.projectName = project.name
                    // 自动重置并加载应用列表
                    this.resetAppData()
                    this.fetchAppList()
                } else {
                    this.resetProjectData()
                }
                
                // 值变动后如果已触摸，则触发校验
                if (this.touched.projectName) {
                    this.validateField('projectName')
                }
            },
            
            // 应用选择变化
            handleAppChange(appId) {
                const app = this.appList.find(a => a.id === appId)
                if (app) {
                    this.formData.appId = app.id
                    this.formData.appName = app.name
                } else {
                    this.resetAppData()
                }
                
                // 值变动后如果已触摸，则触发校验
                if (this.touched.appName) {
                    this.validateField('appName')
                }
            },

            // 展开项目下拉框时，如果列表为空则获取
            handleProjectToggle(isOpen) {
                if (isOpen && this.projectList.length === 0) {
                    this.fetchProjectList()
                }
                // 下拉框关闭时标记为已触摸，并触发红框校验
                if (!isOpen) {
                    this.touched.projectName = true
                    this.validateField('projectName')
                }
            },

            // 展开应用下拉框时，如果列表为空则获取
            handleAppToggle(isOpen) {
                if (isOpen && this.appList.length === 0 && this.formData.projectId) {
                    this.fetchAppList()
                }
                // 下拉框关闭时标记为已触摸，并触发红框校验
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
                // 1. 执行全量验证
                const isValid = this.validateAll()
                
                if (isValid) {
                    // 2. 强制全量同步数据到 atomValue，确保平台能立即拿到最新值
                    Object.keys(this.formData).forEach(key => {
                        this.$set(this.atomValue, key, this.formData[key])
                    })

                    // 3. 通知平台上层：插件状态正常，解锁流水线保存按钮
                    this.setAtomIsError(false)

                    this.saveStatus = {
                        show: true,
                        message: '保存成功'
                    }
                } else {
                    // 4. 通知平台上层：插件状态异常，标红插件并拦截流水线保存
                    this.setAtomIsError(true)

                    this.saveStatus = {
                        show: true,
                        message: '请完善必填信息'
                    }
                }

                setTimeout(() => {
                    this.saveStatus.show = false
                }, 3000)
            }
        },
        beforeDestroy() {
            if (this.successTimer) {
                clearTimeout(this.successTimer)
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
