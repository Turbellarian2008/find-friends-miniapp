<template>
  <view class="register-container">
    <view class="register-header">
      <text class="title">用户注册</text>
    </view>
    
    <view class="register-form">
      <view class="form-item">
        <text class="label">用户名</text>
        <input 
          class="input" 
          v-model="form.username" 
          placeholder="请输入用户名（3-20位,不可更改）"
          maxlength="20"
        />
      </view>
      
      <view class="form-item input-with-suffix">
        <text class="label">密码</text>
        <view class="input-wrapper">
          <input 
            class="input" 
            v-model="form.password" 
            :password="!regPwdVisible"
            placeholder="请输入密码（6-20位）"
            maxlength="20"
          />
          <view class="suffix-group">
            <view class="clear-toggle" v-if="form.password" @click.stop="form.password = ''">
              <text class="icon-clear">✕</text>
            </view>
            <view class="eye-toggle" @click.stop="regPwdVisible = !regPwdVisible">
              <text class="icon-eye">{{ regPwdVisible ? '👁️' : '🙈' }}</text>
            </view>
          </view>
        </view>
      </view>
      
      <view class="form-item input-with-suffix">
        <text class="label">确认密码</text>
        <view class="input-wrapper">
          <input 
            class="input" 
            v-model="form.confirmPassword" 
            :password="!regConfirmVisible"
            placeholder="请再次输入密码"
            maxlength="20"
          />
          <view class="suffix-group">
            <view class="clear-toggle" v-if="form.confirmPassword" @click.stop="form.confirmPassword = ''">
              <text class="icon-clear">✕</text>
            </view>
            <view class="eye-toggle" @click.stop="regConfirmVisible = !regConfirmVisible">
              <text class="icon-eye">{{ regConfirmVisible ? '👁️' : '🙈' }}</text>
            </view>
          </view>
        </view>
      </view>
      
      <!-- 隐私政策同意勾选 -->
      <view class="form-item privacy-row">
        <view class="checkbox" :class="{ checked: agreePrivacy }" @click="toggleAgree">
          <text v-if="agreePrivacy">✔</text>
        </view>
        <text class="privacy-tip">打勾代表同意</text>
        <text class="privacy-link" @click="navigateToPrivacy">隐私政策</text>
      </view>
      
      <button class="register-btn" @click="handleRegister" :disabled="loading">
        {{ loading ? '注册中...' : '注册' }}
      </button>
      
      <view class="login-tip">
        已有账号？<text class="login-link" @click="navigateToLogin">去登录</text>
      </view>

      <!-- 注册前确认弹窗（3秒倒计时后方可确认） -->
      <view v-if="showConfirmDialog" class="modal-mask">
        <view class="modal-dialog">
          <view class="modal-title">重要提醒</view>
          <view class="modal-content">
            请截图保存用户名与密码，用户名不可更改
          </view>
          <view class="modal-actions">
            <button class="btn-cancel" @click="cancelConfirm" :disabled="confirmLoading">取消</button>
            <button class="btn-ok" @click="confirmRegister" :disabled="!confirmEnabled" :class="{ disabled: !confirmEnabled }">
              {{ confirmEnabled ? '确认' : `确认（${confirmCountdown}s）` }}
            </button>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      form: {
        username: '',
        password: '',
        confirmPassword: ''
      },
      loading: false,
      regPwdVisible: false,
      regConfirmVisible: false,
      agreePrivacy: false,
      policyVersion: '1.0.0',
      // 确认弹窗控制
      showConfirmDialog: false,
      confirmCountdown: 3,
      confirmEnabled: false,
      confirmTimer: null,
      confirmLoading: false
    }
  },
  methods: {
    validateForm() {
      if (!this.form.username.trim()) {
        uni.showToast({
          title: '请输入用户名',
          icon: 'none'
        });
        return false;
      }
      
      if (this.form.username.trim().length < 3) {
        uni.showToast({
          title: '用户名长度不能少于3位',
          icon: 'none'
        });
        return false;
      }
      
      if (!this.form.password) {
        uni.showToast({
          title: '请输入密码',
          icon: 'none'
        });
        return false;
      }

      
      
      if (this.form.password.length < 6) {
        uni.showToast({
          title: '密码长度不能少于6位',
          icon: 'none'
        });
        return false;
      }
      
      if (this.form.password !== this.form.confirmPassword) {
        uni.showToast({
          title: '两次输入的密码不一致',
          icon: 'none'
        });
        return false;
      }
      // 必须勾选隐私政策
      if (!this.agreePrivacy) {
        uni.showToast({
          title: '请勾选并同意隐私政策',
          icon: 'none'
        });
        return false;
      }
      
      return true;
    },
    
    handleRegister() {
      if (!this.validateForm()) return;
      // 打开确认弹窗并开始 3 秒倒计时
      this.openConfirmDialog();
    },

    openConfirmDialog() {
      this.showConfirmDialog = true;
      this.confirmEnabled = false;
      this.confirmLoading = false;
      this.confirmCountdown = 3;
      // 清理旧计时器
      if (this.confirmTimer) {
        clearInterval(this.confirmTimer);
        this.confirmTimer = null;
      }
      this.confirmTimer = setInterval(() => {
        if (this.confirmCountdown > 1) {
          this.confirmCountdown -= 1;
        } else {
          this.confirmCountdown = 0;
          this.confirmEnabled = true;
          clearInterval(this.confirmTimer);
          this.confirmTimer = null;
        }
      }, 1000);
    },

    cancelConfirm() {
      if (this.confirmTimer) {
        clearInterval(this.confirmTimer);
        this.confirmTimer = null;
      }
      this.showConfirmDialog = false;
      this.confirmEnabled = false;
      this.confirmLoading = false;
    },

    async confirmRegister() {
      if (!this.confirmEnabled || this.confirmLoading) return;
      this.confirmLoading = true;
      await this.doRegister();
      this.cancelConfirm();
    },

    async doRegister() {
      try {
        this.loading = true;
        // 调用云函数进行注册
        const res = await uniCloud.callFunction({
          name: 'userRegister',
          data: {
            username: this.form.username.trim(),
            password: this.form.password,
            agreePrivacy: this.agreePrivacy,
            policyVersion: this.policyVersion
          },
          timeout: 10000
        });
        if (res.result.code === 0) {
          uni.showToast({ title: '注册成功', icon: 'success' });
          setTimeout(() => {
            uni.redirectTo({ url: '/pages/login/login' });
          }, 1500);
        } else if (res.result.code === 409) {
          uni.showToast({ title: res.result.message || '用户名重复', icon: 'none' });
          return;
        } else {
          throw new Error(res.result.message || '注册失败');
        }
      } catch (err) {
        console.error('注册失败:', err);
        uni.showToast({ title: err.message || '注册失败，请重试', icon: 'none' });
      } finally {
        this.loading = false;
        this.confirmLoading = false;
      }
    },
    
    navigateToLogin() {
      // 使用 redirectTo 替换当前页，避免登录成功后 navigateBack 回到注册页
      uni.redirectTo({
        url: '/pages/login/login'
      });
    },
    navigateToPrivacy() {
      uni.navigateTo({ url: '/pages/privacy/privacy' });
    },
    toggleAgree() {
      this.agreePrivacy = !this.agreePrivacy;
    }
  }
}
</script>

<style>
page {
  background-color: #f5f5f5;
  height: 100%;
}

.register-container {
  padding: 60rpx 60rpx 0;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.register-header {
  margin-bottom: 80rpx;
  text-align: center;
}

.register-header .title {
  font-size: 48rpx;
  font-weight: bold;
  color: #333;
}

.register-form {
  flex: 1;
}

.form-item {
  margin-bottom: 40rpx;
}

.form-item .label {
  display: block;
  font-size: 28rpx;
  color: #333;
  margin-bottom: 16rpx;
}

.form-item .input {
  width: 100%;
  height: 88rpx;
  background-color: #fff;
  border-radius: 8rpx;
  padding: 0 24rpx;
  box-sizing: border-box;
  font-size: 28rpx;
  border: 1px solid #eee;
}

.register-btn {
  width: 100%;
  height: 88rpx;
  line-height: 88rpx;
  background-color: #007AFF;
  color: #fff;
  font-size: 32rpx;
  border-radius: 44rpx;
  margin-top: 60rpx;
  text-align: center;
}

.register-btn[disabled] {
  background-color: #a0cfff;
  color: #fff;
}

.login-tip {
  margin-top: 40rpx;
  text-align: center;
  font-size: 26rpx;
  color: #999;
}

.login-link {
  color: #007AFF;
  margin-left: 10rpx;
}

/* 自定义弹窗 */
.modal-mask {
  position: fixed;
  left: 0; right: 0; top: 0; bottom: 0;
  background: rgba(0,0,0,0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10000;
}
.modal-dialog {
  width: 640rpx;
  background: #fff;
  border-radius: 20rpx;
  overflow: hidden;
  min-height: 100rpx; /* 增加弹窗高度 */
  display: flex;
  flex-direction: column;
  padding-bottom: 0; /* 减少底部预留，整体高度减少 */
  transform: translateY(-100rpx); /* 整体上移 100rpx */
}
.modal-title {
  padding: 28rpx 32rpx 8rpx;
  font-size: 50rpx; /* 放大标题字号 */
  font-weight: 600;
  color: #333;
  text-align: center; /* 标题居中 */
}
.modal-content {
  padding: 0 32rpx 12rpx;
  font-size: 40rpx;
  color: #666;
  flex: 1; /* 占据中间可用空间 */
  display: flex;
  align-items: center; /* 垂直居中内容 */
  justify-content: center; /* 水平居中内容 */
  text-align: center;
  transform: translateY(50rpx); /* 下移减半 */
}
.modal-actions {
  display: flex;
  gap: 16rpx;
  padding: 24rpx 24rpx calc(24rpx + env(safe-area-inset-bottom));
  /* margin-top: auto; 按钮固定在弹窗底部 */
  transform: translateY(70rpx); /* 上移 100rpx，配合 padding-bottom 一起总计减少约200rpx */
}
.btn-cancel {
  flex: 1;
  background: #f5f5f5;
  color: #666;
  border-radius: 12rpx;
  height: 80rpx;
  line-height: 80rpx;
}
.btn-ok {
  flex: 1;
  background: #007AFF;
  color: #fff;
  border-radius: 12rpx;
  height: 80rpx;
  line-height: 80rpx;
}
.btn-ok.disabled {
  background: #a0cfff;
}
/* 密码显示/隐藏输入框样式 */
.input-with-suffix .input-wrapper {
  position: relative;
}
.input-with-suffix .input {
  padding-right: 80rpx; /* 预留眼睛按钮空间 */
}
.eye-toggle {
  position: absolute;
  right: 20rpx;
  top: 0;
  height: 88rpx; /* 与输入框等高 */
  display: flex;
  align-items: center;
}

/* 隐私政策勾选行 */
.privacy-row {
  display: flex;
  align-items: center;
  gap: 16rpx;
  margin-top: 8rpx;
}
.checkbox {
  width: 32rpx;
  height: 32rpx;
  border-radius: 6rpx;
  border: 2rpx solid #c0c4cc;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #007AFF;
  box-sizing: border-box;
}
.checkbox.checked {
  border-color: #007AFF;
  background: #e8f1ff;
}
.privacy-tip {
  font-size: 24rpx;
  color: #666;
}
.privacy-link {
  font-size: 24rpx;
  color: #007AFF;
}
</style>
