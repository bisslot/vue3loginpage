<template>
  <div class="login-container">
    <!-- 登录头部 -->
    <div class="login-header animate-item">
      <div class="login-logo">
        <UserIcon class="logo-icon" />
      </div>
      <h1 class="login-title">用户登录</h1>
      <p class="login-subtitle">请输入您的账号信息</p>
    </div>

    <!-- 登录表单 -->
    <form class="login-form" @submit.prevent="handleLogin">
      <!-- 用户名输入框 -->
      <div class="form-item animate-item" style="animation-delay: 100ms;">
        <label for="username">
          用户名
          <span class="input-type-hint" v-if="usernameInputType !== 'unknown'">
            ({{ usernameInputType === 'email' ? '邮箱' : '手机号' }})
          </span>
        </label>
        <Input
          id="username"
          v-model="formData.username"
          :placeholder="usernamePlaceholder"
          size="large"
          :status="usernameStatus"
          :tips="usernameTips"
          clearable
          @focus="handleUsernameFocus"
          @blur="handleUsernameBlur"
          :disabled="isLoginBlocked"
        >
          <template #prefix-icon>
            <UserIcon />
          </template>
          <template #suffix-icon>
            <CheckCircleIcon v-if="usernameStatus === 'success'" class="success-icon" />
            <ErrorCircleIcon v-if="usernameStatus === 'error' && !usernameInputFocused" class="error-icon" />
          </template>
        </Input>
      </div>

      <!-- 密码输入框 -->
      <div class="form-item animate-item" style="animation-delay: 200ms;">
        <label for="password">密码</label>
        <Input
          id="password"
          v-model="formData.password"
          :type="showPassword ? 'text' : 'password'"
          placeholder="请输入密码"
          size="large"
          :status="passwordStatus"
          :tips="passwordTips"
          clearable
          @focus="handlePasswordFocus"
          @blur="handlePasswordBlur"
          :disabled="isLoginBlocked"
        >
          <template #prefix-icon>
            <LockOnIcon />
          </template>
          <template #suffix-icon>
            <div class="password-actions">
              <CheckCircleIcon v-if="passwordStatus === 'success'" class="success-icon" />
              <ErrorCircleIcon v-if="passwordStatus === 'error' && !passwordInputFocused" class="error-icon" />
              <span class="password-toggle" @click.stop="togglePassword">
                <BrowseIcon 
                  v-if="!showPassword" 
                  class="password-eye"
                />
                <BrowseOffIcon 
                  v-else 
                  class="password-eye"
                />
              </span>
            </div>
          </template>
        </Input>
      </div>

      <!-- 滑块验证码区域 -->
      <div class="captcha-container">
        <div class="form-item animate-item" style="animation-delay: 300ms;">
          <label>安全验证</label>
          <SliderCaptcha 
            v-model:verified="captchaVerified"
            @success="handleCaptchaSuccess"
            @reset="handleCaptchaReset"
            ref="captchaRef"
            :disabled="isLoginBlocked"
          />
        </div>
      </div>

      <!-- 登录按钮 -->
      <Button
        type="submit"
        theme="primary"
        size="large"
        :loading="isLoading"
        :disabled="!isFormValid || isLoginBlocked"
        class="login-button animate-item"
        style="animation-delay: 400ms;"
        block
        @click="handleLogin"
      >
        <template v-if="!isLoading && !loginSuccess && !isLoginBlocked">
          登录
        </template>
        <template v-else-if="isLoading">
          <span class="loading-text">登录中<span class="dot-animation">...</span></span>
        </template>
        <template v-else-if="loginSuccess">
          <span class="success-text">
            <CheckCircleIcon class="success-icon-btn" />
            登录成功
          </span>
        </template>
        <template v-else-if="isLoginBlocked">
          <span class="blocked-text">
            请等待 {{ blockTimeRemaining }}秒
          </span>
        </template>
      </Button>
      
      <!-- 登录错误提示 -->
      <div v-if="loginError" class="login-error-message">
        <ErrorCircleIcon class="error-icon-msg" />
        {{ loginErrorMessage }}
        
        <!-- 网络错误时显示重试按钮 -->
        <div v-if="showRetryButton" class="error-actions">
          <Button size="small" theme="danger" variant="outline" @click="handleRetry">
            重试
          </Button>
        </div>
        
        <!-- 账号锁定时显示联系客服选项 -->
        <div v-if="showContactSupport" class="error-actions">
          <Button size="small" theme="primary" variant="outline" @click="handleContactSupport">
            联系客服
          </Button>
        </div>
      </div>
      
      <!-- 登录尝试次数提示 -->
      <div v-if="loginAttempts > 1 && !loginSuccess" class="login-attempts-warning">
        <InfoCircleIcon class="info-icon" />
        您已尝试登录 {{ loginAttempts }} 次，连续 5 次失败将暂时锁定账号
      </div>
    </form>

    <!-- 页脚链接 -->
    <div class="login-footer animate-item" style="animation-delay: 500ms;">
      <div class="footer-links">
        <a href="#" @click.prevent>忘记密码？</a>
        <a href="#" @click.prevent>注册账号</a>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import { Input, Button } from 'tdesign-vue-next'
import { UserIcon, LockOnIcon, BrowseIcon, BrowseOffIcon, CheckCircleIcon, ErrorCircleIcon, InfoCircleIcon } from 'tdesign-icons-vue-next'
import SliderCaptcha from './SliderCaptcha.vue'

// 表单数据
const formData = ref({
  username: '',
  password: ''
})

// 控制状态
const showPassword = ref(false)
const isLoading = ref(false)
const captchaVerified = ref(false)
const usernameInputFocused = ref(false)
const passwordInputFocused = ref(false)
const loginSuccess = ref(false)
const loginError = ref(false)
const loginErrorMessage = ref('')
const showRetryButton = ref(false)
const showContactSupport = ref(false)
const loginAttempts = ref(0) // 登录尝试次数
const isLoginBlocked = ref(false) // 是否暂时阻止登录
const blockTimeRemaining = ref(0) // 阻止登录的剩余时间（秒）
const blockTimer = ref<number | null>(null) // 计时器引用

// 表单验证状态
const usernameStatus = ref<'default' | 'success' | 'warning' | 'error'>('default')
const passwordStatus = ref<'default' | 'success' | 'warning' | 'error'>('default')
const usernameTips = ref('')
const passwordTips = ref('')

// 用户名输入类型检测
const usernameInputType = ref<'email' | 'phone' | 'unknown'>('unknown')

// 切换密码显示
const togglePassword = () => {
  showPassword.value = !showPassword.value
}

// 检测用户名输入类型
const detectUsernameType = (username: string) => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  const phoneRegex = /^1[3-9]\d{9}$/
  const partialEmailRegex = /^[^\s@]*@?[^\s@]*\.?[^\s@]*$/
  const partialPhoneRegex = /^1[3-9]?\d{0,9}$/
  
  if (emailRegex.test(username)) {
    return 'email'
  } else if (phoneRegex.test(username)) {
    return 'phone'
  } else if (partialEmailRegex.test(username) && username.includes('@')) {
    return 'email'
  } else if (partialPhoneRegex.test(username) && username.startsWith('1')) {
    return 'phone'
  }
  
  return 'unknown'
}

// 增强的用户名验证
const validateUsername = (username: string, showDetailedTips = false) => {
  if (!username.trim()) {
    return { 
      valid: false, 
      message: '请输入邮箱或手机号',
      type: 'unknown'
    }
  }
  
  const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
  const phoneRegex = /^1[3-9]\d{9}$/
  const type = detectUsernameType(username)
  
  // 邮箱验证
  if (type === 'email') {
    if (emailRegex.test(username)) {
      return { 
        valid: true, 
        message: showDetailedTips ? '邮箱格式正确' : '',
        type: 'email'
      }
    } else {
      // 详细的邮箱格式提示
      if (!username.includes('@')) {
        return { 
          valid: false, 
          message: '邮箱格式不正确，缺少@符号',
          type: 'email'
        }
      } else if (!username.includes('.') || username.indexOf('.') < username.indexOf('@')) {
        return { 
          valid: false, 
          message: '邮箱格式不正确，请检查域名部分',
          type: 'email'
        }
      } else {
        return { 
          valid: false, 
          message: '邮箱格式不正确',
          type: 'email'
        }
      }
    }
  }
  
  // 手机号验证
  if (type === 'phone') {
    if (phoneRegex.test(username)) {
      return { 
        valid: true, 
        message: showDetailedTips ? '手机号格式正确' : '',
        type: 'phone'
      }
    } else {
      if (username.length < 11) {
        return { 
          valid: false, 
          message: `手机号长度不足，还需${11 - username.length}位数字`,
          type: 'phone'
        }
      } else if (username.length > 11) {
        return { 
          valid: false, 
          message: '手机号长度超出，请检查输入',
          type: 'phone'
        }
      } else {
        return { 
          valid: false, 
          message: '手机号格式不正确，请以1开头',
          type: 'phone'
        }
      }
    }
  }
  
  // 未知类型，给出引导提示
  if (username.length < 3) {
    return { 
      valid: false, 
      message: '请输入完整的邮箱或手机号',
      type: 'unknown'
    }
  }
  
  return { 
    valid: false, 
    message: '请输入正确的邮箱格式（如：user@example.com）或手机号（如：13812345678）',
    type: 'unknown'
  }
}

// 验证密码
const validatePassword = (password: string) => {
  if (!password) {
    return { valid: false, message: '请输入密码' }
  }
  
  if (password.length < 6) {
    return { valid: false, message: `密码长度至少6位，当前${password.length}位` }
  }
  
  if (password.length > 20) {
    return { valid: false, message: '密码长度不能超过20位' }
  }
  
  return { valid: true, message: '密码格式正确' }
}

// 实时验证用户名
watch(() => formData.value.username, (newUsername) => {
  if (!newUsername) {
    usernameStatus.value = 'default'
    usernameTips.value = ''
    usernameInputType.value = 'unknown'
    return
  }
  
  const validation = validateUsername(newUsername, true)
  usernameInputType.value = validation.type as 'email' | 'phone' | 'unknown'
  
  if (validation.valid) {
    usernameStatus.value = 'success'
    usernameTips.value = validation.message
  } else {
    // 如果用户正在输入且输入框聚焦，显示警告而不是错误
    if (usernameInputFocused.value && newUsername.length > 0) {
      usernameStatus.value = 'warning'
    } else {
      usernameStatus.value = 'error'
    }
    usernameTips.value = validation.message
  }
}, { immediate: true })

// 实时验证密码
watch(() => formData.value.password, (newPassword) => {
  if (!newPassword) {
    passwordStatus.value = 'default'
    passwordTips.value = ''
    return
  }
  
  const validation = validatePassword(newPassword)
  
  if (validation.valid) {
    passwordStatus.value = 'success'
    passwordTips.value = validation.message
  } else {
    // 如果用户正在输入且输入框聚焦，显示警告而不是错误
    if (passwordInputFocused.value && newPassword.length > 0) {
      passwordStatus.value = 'warning'
    } else {
      passwordStatus.value = 'error'
    }
    passwordTips.value = validation.message
  }
}, { immediate: true })

// 表单是否有效
const isFormValid = computed(() => {
  const usernameValid = validateUsername(formData.value.username).valid
  const passwordValid = validatePassword(formData.value.password).valid
  return usernameValid && passwordValid && captchaVerified.value
})

// 获取用户名输入框占位符
const usernamePlaceholder = computed(() => {
  switch (usernameInputType.value) {
    case 'email':
      return '请输入邮箱地址'
    case 'phone':
      return '请输入手机号码'
    default:
      return '请输入邮箱或手机号'
  }
})

// 处理用户名输入框聚焦
const handleUsernameFocus = () => {
  usernameInputFocused.value = true
}

// 处理用户名输入框失焦
const handleUsernameBlur = () => {
  usernameInputFocused.value = false
  // 失焦时重新验证，显示最终状态
  if (formData.value.username) {
    const validation = validateUsername(formData.value.username)
    usernameStatus.value = validation.valid ? 'success' : 'error'
    usernameTips.value = validation.message
  }
}

// 处理密码输入框聚焦
const handlePasswordFocus = () => {
  passwordInputFocused.value = true
}

// 处理密码输入框失焦
const handlePasswordBlur = () => {
  passwordInputFocused.value = false
  // 失焦时重新验证，显示最终状态
  if (formData.value.password) {
    const validation = validatePassword(formData.value.password)
    passwordStatus.value = validation.valid ? 'success' : 'error'
    passwordTips.value = validation.message
  }
}

// 处理登录
const handleLogin = async (e: Event) => {
  // 阻止表单默认提交行为
  if (e) e.preventDefault()
  
  // 如果登录被阻止，直接返回
  if (isLoginBlocked.value) return
  
  // 重置错误状态
  loginError.value = false
  loginErrorMessage.value = ''
  loginSuccess.value = false
  showRetryButton.value = false
  showContactSupport.value = false
  
  // 最终验证表单
  const usernameValidation = validateUsername(formData.value.username)
  const passwordValidation = validatePassword(formData.value.password)
  
  usernameStatus.value = usernameValidation.valid ? 'success' : 'error'
  passwordStatus.value = passwordValidation.valid ? 'success' : 'error'
  usernameTips.value = usernameValidation.message
  passwordTips.value = passwordValidation.message
  
  // 验证滑块验证码
  if (!captchaVerified.value) {
    loginError.value = true
    loginErrorMessage.value = '请完成滑块验证'
    return
  }
  
  // 验证表单
  if (!isFormValid.value) {
    // 显示具体的错误信息
    if (!usernameValidation.valid) {
      loginError.value = true
      loginErrorMessage.value = usernameValidation.message
      // 聚焦到用户名输入框
      document.getElementById('username')?.focus()
    } else if (!passwordValidation.valid) {
      loginError.value = true
      loginErrorMessage.value = passwordValidation.message
      // 聚焦到密码输入框
      document.getElementById('password')?.focus()
    }
    return
  }
  
  // 增加登录尝试次数
  loginAttempts.value++
  
  // 检查是否需要暂时阻止登录
  if (loginAttempts.value >= 5) {
    blockLogin(30) // 连续5次失败，阻止登录30秒
    loginError.value = true
    loginErrorMessage.value = '登录失败次数过多，请稍后再试'
    return
  }
  
  // 开始登录流程
  isLoading.value = true
  loginError.value = false
  
  try {
    // 模拟网络延迟和错误情况
    const testCase = Math.random()
    
    await new Promise((resolve, reject) => {
      setTimeout(() => {
        // 模拟不同的登录结果
        if (testCase > 0.7) {
          // 正常登录成功
          resolve('登录成功')
        } else if (testCase > 0.5) {
          // 用户名或密码错误
          reject(new Error('用户名或密码错误'))
        } else if (testCase > 0.3) {
          // 账号被锁定
          reject(new Error('账号已被锁定，请联系管理员'))
        } else if (testCase > 0.1) {
          // 网络错误
          reject(new Error('网络连接异常，请检查网络设置'))
        } else {
          // 服务器错误
          reject(new Error('服务器繁忙，请稍后再试'))
        }
      }, 1500 + Math.random() * 1000) // 随机延迟1.5-2.5秒
    })
    
    // 登录成功
    console.log('登录成功', formData.value)
    loginSuccess.value = true
    loginAttempts.value = 0 // 重置登录尝试次数
    
    // 模拟跳转到首页
    setTimeout(() => {
      alert('登录成功！即将跳转到首页...')
      // 实际项目中这里可以使用路由导航
      // router.push('/dashboard')
    }, 1500)
    
  } catch (error) {
    // 登录失败
    console.error('登录失败', error)
    loginError.value = true
    loginErrorMessage.value = error instanceof Error ? error.message : '登录失败，请稍后重试'
    
    // 根据错误类型执行不同操作
    if (loginErrorMessage.value.includes('网络')) {
      // 网络错误时显示重试按钮
      showRetryButton.value = true
    } else if (loginErrorMessage.value.includes('锁定')) {
      // 账号锁定时显示联系客服选项
      showContactSupport.value = true
    } else {
      // 其他错误重置验证码
      if (captchaRef.value) {
        captchaRef.value.reset()
        captchaVerified.value = false
      }
    }
  } finally {
    isLoading.value = false
  }
}

// 处理重试按钮点击
const handleRetry = () => {
  showRetryButton.value = false
  loginError.value = false
  handleLogin(new Event('click'))
}

// 处理联系客服按钮点击
const handleContactSupport = () => {
  alert('即将跳转到客服页面...')
  // 实际项目中这里可以跳转到客服页面
  // window.open('https://support.example.com', '_blank')
}

// 暂时阻止登录
const blockLogin = (seconds: number) => {
  isLoginBlocked.value = true
  blockTimeRemaining.value = seconds
  
  // 清除之前的计时器
  if (blockTimer.value !== null) {
    clearInterval(blockTimer.value)
  }
  
  // 创建新的计时器
  blockTimer.value = setInterval(() => {
    blockTimeRemaining.value--
    
    if (blockTimeRemaining.value <= 0) {
      // 解除登录阻止
      isLoginBlocked.value = false
      clearInterval(blockTimer.value as number)
      blockTimer.value = null
    }
  }, 1000) as unknown as number
}

// 滑块验证码引用
const captchaRef = ref<InstanceType<typeof SliderCaptcha> | null>(null)

// 处理验证码验证成功
const handleCaptchaSuccess = () => {
  console.log('验证码验证成功')
  // 可以在这里添加额外的验证逻辑
}

// 处理验证码重置
const handleCaptchaReset = () => {
  console.log('验证码已重置')
}

// 重置所有表单
const resetForm = () => {
  formData.value.username = ''
  formData.value.password = ''
  usernameStatus.value = 'default'
  passwordStatus.value = 'default'
  usernameTips.value = ''
  passwordTips.value = ''
  loginError.value = false
  loginErrorMessage.value = ''
  showRetryButton.value = false
  showContactSupport.value = false
  
  // 重置验证码
  if (captchaRef.value) {
    captchaRef.value.reset()
    captchaVerified.value = false
  }
}

// 检测设备类型
const isMobile = ref(false)

// 页面加载动画
onMounted(() => {
  // 检测是否为移动设备
  isMobile.value = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
  
  // 添加移动设备类
  if (isMobile.value) {
    document.body.classList.add('mobile-device')
  }
  
  // 添加页面加载完成的类，触发动画
  document.body.classList.add('page-loaded')
  
  // 添加输入框聚焦效果
  const inputs = document.querySelectorAll('input')
  inputs.forEach(input => {
    input.addEventListener('focus', () => {
      document.body.classList.add('input-focused')
      
      // 移动端输入时，滚动到输入框位置
      if (isMobile.value) {
        setTimeout(() => {
          input.scrollIntoView({ behavior: 'smooth', block: 'center' })
        }, 300)
      }
    })
    
    input.addEventListener('blur', () => {
      document.body.classList.remove('input-focused')
    })
  })
  
  // 添加键盘事件监听
  document.addEventListener('keydown', (e) => {
    if (e.key === 'Enter') {
      handleLogin(e)
    }
  })
  
  // 添加触摸反馈
  if (isMobile.value) {
    const touchElements = document.querySelectorAll('button, a, .captcha-slider')
    touchElements.forEach(el => {
      el.addEventListener('touchstart', () => {
        el.classList.add('touch-active')
      })
      
      el.addEventListener('touchend', () => {
        setTimeout(() => {
          el.classList.remove('touch-active')
        }, 150)
      })
    })
  }
})

// 组件卸载时清理
onUnmounted(() => {
  // 清除计时器
  if (blockTimer.value !== null) {
    clearInterval(blockTimer.value)
  }
  
  // 移除事件监听器
  document.removeEventListener('keydown', (e) => {
    if (e.key === 'Enter') {
      handleLogin(e)
    }
  })
})
</script>

<style scoped>
/* 输入框样式增强 */
.form-item {
  position: relative;
}

.input-type-hint {
  font-size: 12px;
  color: var(--td-text-color-secondary);
  font-weight: normal;
  margin-left: 4px;
}

.success-icon {
  color: var(--td-success-color);
}

.error-icon {
  color: var(--td-error-color);
}

.info-icon {
  color: var(--td-warning-color);
}

.password-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.password-toggle {
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 4px;
  border-radius: 50%;
  transition: background-color 0.2s ease;
}

.password-toggle:hover {
  background-color: rgba(0, 0, 0, 0.05);
}

.password-eye {
  color: var(--td-text-color-secondary);
}

/* 输入框聚焦动画 */
:deep(.t-input) {
  transition: all 0.3s ease;
}

:deep(.t-input:focus-within) {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

/* 验证提示动画 */
:deep(.t-input__tips) {
  transition: opacity 0.3s ease, transform 0.3s ease;
  opacity: 0;
  transform: translateY(-5px);
}

:deep(.t-input--success .t-input__tips),
:deep(.t-input--warning .t-input__tips),
:deep(.t-input--error .t-input__tips) {
  opacity: 1;
  transform: translateY(0);
}

/* 验证码容器样式 */
.captcha-container {
  margin: 8px 0;
}

/* 登录按钮 */
.login-button {
  margin-top: 8px;
  position: relative;
  overflow: hidden;
  transition: all 0.3s ease;
}

.login-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.login-button:active:not(:disabled) {
  transform: translateY(0);
}

/* 登录按钮文字动画 */
.loading-text {
  display: inline-flex;
  align-items: center;
}

.dot-animation {
  display: inline-block;
  width: 16px;
  animation: dotTyping 1.5s infinite linear;
  text-align: left;
}

@keyframes dotTyping {
  0% { content: '.'; }
  25% { content: '..'; }
  50% { content: '...'; }
  75% { content: '..'; }
  100% { content: '.'; }
}

/* 成功状态 */
.success-text {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.success-icon-btn {
  animation: zoomIn 0.5s ease;
}

/* 阻止登录状态 */
.blocked-text {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  color: var(--td-text-color-secondary);
}

@keyframes zoomIn {
  from {
    opacity: 0;
    transform: scale(0.5);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* 错误提示 */
.login-error-message {
  margin-top: 16px;
  padding: 10px;
  border-radius: 6px;
  background-color: rgba(227, 77, 89, 0.1);
  color: var(--td-error-color);
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 8px;
  animation: fadeIn 0.3s ease;
  flex-wrap: wrap;
}

/* 错误操作按钮 */
.error-actions {
  margin-left: auto;
  display: flex;
  gap: 8px;
}

/* 登录尝试警告 */
.login-attempts-warning {
  margin-top: 12px;
  padding: 8px;
  border-radius: 6px;
  background-color: rgba(230, 162, 60, 0.1);
  color: var(--td-warning-color);
  font-size: 13px;
  display: flex;
  align-items: center;
  gap: 8px;
  animation: fadeIn 0.3s ease;
}

/* 页面元素进入动画 */
.animate-item {
  animation: fadeInUp 0.5s ease forwards;
  opacity: 0;
}

/* Logo动画 */
.login-logo {
  position: relative;
  overflow: hidden;
}

.logo-icon {
  animation: pulse 2s infinite ease-in-out;
}

/* 登录标题动画 */
.login-title {
  position: relative;
}

.login-title::after {
  content: '';
  position: absolute;
  bottom: -4px;
  left: 50%;
  transform: translateX(-50%);
  width: 40px;
  height: 2px;
  background-color: var(--td-brand-color);
  transition: width 0.3s ease;
}

.login-container:hover .login-title::after {
  width: 80px;
}

.error-icon-msg {
  flex-shrink: 0;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.02);
  }
  100% {
    transform: scale(1);
  }
}

/* 移动端触摸反馈 */
.touch-active {
  opacity: 0.8;
  transform: scale(0.98);
}

/* 移动端优化 */
@media (max-width: 768px) {
  /* 增大触摸区域 */
  .password-toggle {
    padding: 8px;
  }
  
  .footer-links {
    gap: 32px;
  }
  
  .footer-links a {
    padding: 8px 0;
  }
  
  /* 优化表单间距 */
  .form-item {
    margin-bottom: 20px;
  }
  
  /* 优化动画效果 */
  .animate-item {
    animation-duration: 0.3s;
  }
  
  /* 错误提示优化 */
  .error-actions {
    margin-top: 8px;
    width: 100%;
    justify-content: flex-end;
  }
}

/* 适配iPhone小屏幕 */
@media (max-width: 375px) {
  .login-title {
    font-size: 20px;
  }
  
  .login-subtitle {
    font-size: 13px;
  }
  
  .form-item label {
    font-size: 13px;
  }
  
  .login-attempts-warning,
  .login-error-message {
    font-size: 12px;
  }
}
</style>
