<template>
    <section class="bk-form bk-form-vertical atom-form">
        <!-- SAST服务器地址 -->
        <div class="form-group" :class="{ 'has-error': errors.server.show }">
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
            <div class="error-message" v-if="errors.server.show">
                {{ errors.server.message }}
            </div>
        </div>

        <!-- Token -->
        <div class="form-group" :class="{ 'has-error': errors.token.show }">
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
            <div class="error-message" v-if="errors.token.show">
                {{ errors.token.message }}
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
            
            <!-- 成功提示 -->
            <transition name="fade">
                <div class="test-result success" v-if="testResult.show && testResult.type === 'success'">
                    <span class="icon">✓</span>
                    <span class="message">连接成功！</span>
                </div>
            </transition>
            
            <!-- 失败提示 -->
            <transition name="fade">
                <div class="test-result error" v-if="testResult.show && testResult.type === 'error'">
                    <span class="icon">✗</span>
                    <span class="message">{{ testResult.message }}</span>
                </div>
            </transition>
        </div>

        <!-- 项目选择 -->
        <div class="form-group" :class="{ 'has-error': errors.projectName.show }">
            <label class="form-label required">项目</label>
            <select 
                class="form-select"
                v-model="formData.projectId"
                @change="handleProjectChange(formData.projectId)"
                @focus="fetchProjectList()"
                :disabled="atomPropsDisabled || !formData.server || !formData.token"
            >
                <option value="">请选择项目</option>
                <option 
                    v-for="project in projectList" 
                    :key="project.id" 
                    :value="project.id"
                >
                    {{ project.name }}
                </option>
            </select>
            <div class="error-message" v-if="errors.projectName.show">
                {{ errors.projectName.message }}
            </div>
            <div class="field-tip" v-if="!formData.server || !formData.token">
                请先填写服务器地址和Token
            </div>
            <div class="field-tip" v-if="projectLoading">
                加载中...
            </div>
        </div>

        <!-- 应用选择 -->
        <div class="form-group" :class="{ 'has-error': errors.appName.show }">
            <label class="form-label required">应用</label>
            <select 
                class="form-select"
                v-model="formData.appId"
                @change="handleAppChange(formData.appId)"
                @focus="fetchAppList()"
                :disabled="atomPropsDisabled || !formData.projectId"
            >
                <option value="">请选择应用</option>
                <option 
                    v-for="app in appList" 
                    :key="app.id" 
                    :value="app.id"
                >
                    {{ app.name }}
                </option>
            </select>
            <div class="error-message" v-if="errors.appName.show">
                {{ errors.appName.message }}
            </div>
            <div class="field-tip" v-if="!formData.projectId">
                请先选择项目
            </div>
            <div class="field-tip" v-if="appLoading">
                加载中...
            </div>
        </div>
    </section>
</template>

<script>
    // 需引用atomMixin
    import { atomMixin } from 'bkci-atom-components'

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
                errors: {
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
        mounted() {
            // 从 atomValue 中初始化表单数据
            if (this.atomValue) {
                this.formData.server = this.atomValue.server || ''
                this.formData.token = this.atomValue.token || ''
                this.formData.projectId = this.atomValue.projectId || ''
                this.formData.projectName = this.atomValue.projectName || ''
                this.formData.appId = this.atomValue.appId || ''
                this.formData.appName = this.atomValue.appName || ''
            }
        },
        watch: {
            'formData.server'(newVal) {
                this.atomValue.server = newVal
                this.clearTestResult()
            },
            'formData.token'(newVal) {
                this.atomValue.token = newVal
                this.clearTestResult()
            },
            'formData.projectId'(newVal) {
                this.atomValue.projectId = newVal
            },
            'formData.projectName'(newVal) {
                this.atomValue.projectName = newVal
            },
            'formData.appId'(newVal) {
                this.atomValue.appId = newVal
            },
            'formData.appName'(newVal) {
                this.atomValue.appName = newVal
            }
        },
        methods: {
            // 验证单个字段
            validateField(fieldName) {
                const value = this.formData[fieldName]
                
                // 特殊处理：projectName 和 appName 的验证基于对应的 ID
                if (fieldName === 'projectName') {
                    if (!this.formData.projectId) {
                        this.errors.projectName = {
                            show: true,
                            message: '请选择项目'
                        }
                        return false
                    } else {
                        this.errors.projectName = {
                            show: false,
                            message: ''
                        }
                        return true
                    }
                }
                
                if (fieldName === 'appName') {
                    if (!this.formData.appId) {
                        this.errors.appName = {
                            show: true,
                            message: '请选择应用'
                        }
                        return false
                    } else {
                        this.errors.appName = {
                            show: false,
                            message: ''
                        }
                        return true
                    }
                }
                
                if (!value || value.trim() === '') {
                    this.errors[fieldName] = {
                        show: true,
                        message: '字段不能为空'
                    }
                    return false
                } else {
                    this.errors[fieldName] = {
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
            validateAll() {
                let isValid = true
                Object.keys(this.formData).forEach(key => {
                    if (!this.validateField(key)) {
                        isValid = false
                    }
                })
                return isValid
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
                // 验证所有必填字段
                if (!this.validateAll()) {
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
                    const response = await this.$ajax({
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
                    
                    if (response.data && response.data.data && response.data.data.duration === "1") {
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
                    
                    const response = await this.$ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token
                        },
                        timeout: 10000
                    })
                    
                    if (response.data && response.data.code === 0 && response.data.data && response.data.data.records) {
                        this.projectList = response.data.data.records.map(item => ({
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
                    
                    const response = await this.$ajax({
                        url: url,
                        method: 'GET',
                        headers: {
                            'Sast-Token': token
                        },
                        timeout: 10000
                    })
                    
                    if (response.data && response.data.code === 0 && response.data.data && response.data.data.records) {
                        let apps = response.data.data.records
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
                    // 清空应用选择
                    this.formData.appId = ''
                    this.formData.appName = ''
                    this.appList = []
                    // 自动加载应用列表
                    this.fetchAppList()
                }
            },
            
            // 应用选择变化
            handleAppChange(appId) {
                const app = this.appList.find(a => a.id === appId)
                if (app) {
                    this.formData.appId = app.id
                    this.formData.appName = app.name
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
            }
        },
        beforeDestroy() {
            if (this.successTimer) {
                clearTimeout(this.successTimer)
            }
        }
    }
</script>

<style lang="scss" scoped>
    .atom-form {
        padding: 20px 0;
        
        .form-group {
            margin-bottom: 20px;
            
            &.has-error {
                .form-input {
                    border-color: #ff5656;
                    
                    &:focus {
                        border-color: #ff5656;
                        box-shadow: 0 0 0 2px rgba(255, 86, 86, 0.1);
                    }
                }
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
                
                .icon {
                    font-size: 16px;
                    font-weight: bold;
                }
                
                &.success {
                    color: #2dcb56;
                    
                    .icon {
                        color: #2dcb56;
                    }
                }
                
                &.error {
                    color: #ff5656;
                    
                    .icon {
                        color: #ff5656;
                    }
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
