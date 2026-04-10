<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { sm2, sm3, sm4 } from 'sm-crypto'
import { 
  Shield, 
  Key, 
  FileSignature, 
  Hash, 
  Lock, 
  Unlock, 
  Copy, 
  RefreshCw, 
  CheckCircle2, 
  AlertCircle 
} from 'lucide-vue-next'
import { Toaster, toast } from 'vue-sonner'
import 'vue-sonner/style.css'

// SM2 状态
const sm2Keys = ref({ publicKey: '', privateKey: '' })
const sm2Message = ref('你好，国密算法')
const sm2Signature = ref('')
const sm2VerifyResult = ref<boolean | null>(null)
const sm2CryptoInput = ref('你好，国密算法')
const sm2CryptoResult = ref('')
const sm2CryptoMode = ref<'encrypt' | 'decrypt'>('encrypt')

// SM3 状态
const sm3Input = ref('')
const sm3Hash = ref('')

// SM4 状态
const sm4Key = ref('0123456789abcdeffedcba9876543210')
const sm4Input = ref('')
const sm4Result = ref('')
const sm4Mode = ref<'encrypt' | 'decrypt'>('encrypt')

// 标签页状态
const activeTab = ref('sm2')

const generateSM2Keys = () => {
  const keypair = sm2.generateKeyPairHex()
  sm2Keys.value = keypair
  toast.success('已生成新的 SM2 密钥对')
}

onMounted(() => {
  generateSM2Keys()
})

// 自动执行逻辑
import { watch } from 'vue'

// SM2 自动签名
watch([sm2Message, () => sm2Keys.value.privateKey], () => {
  if (sm2Message.value && sm2Keys.value.privateKey.length === 64) {
    try {
      sm2Signature.value = sm2.doSignature(sm2Message.value, sm2Keys.value.privateKey)
    } catch (e) {
      // 正在输入中，暂不报错
    }
  } else if (!sm2Message.value) {
    sm2Signature.value = ''
  }
}, { immediate: true })

// SM2 自动验签
watch([sm2Message, sm2Signature, () => sm2Keys.value.publicKey], () => {
  if (sm2Message.value && sm2Signature.value && sm2Keys.value.publicKey.length === 130) {
    try {
      sm2VerifyResult.value = sm2.doVerifySignature(sm2Message.value, sm2Signature.value, sm2Keys.value.publicKey)
    } catch (e) {
      sm2VerifyResult.value = false
    }
  } else {
    sm2VerifyResult.value = null
  }
}, { immediate: true })

// SM2 自动加解密 (参照 SM4 风格)
watch([sm2CryptoInput, sm2CryptoMode, () => sm2Keys.value.publicKey, () => sm2Keys.value.privateKey], () => {
  if (!sm2CryptoInput.value) {
    sm2CryptoResult.value = ''
    return
  }

  try {
    if (sm2CryptoMode.value === 'encrypt') {
      if (sm2Keys.value.publicKey.length === 130) {
        sm2CryptoResult.value = sm2.doEncrypt(sm2CryptoInput.value, sm2Keys.value.publicKey)
      }
    } else {
      if (sm2Keys.value.privateKey.length === 64) {
        sm2CryptoResult.value = sm2.doDecrypt(sm2CryptoInput.value, sm2Keys.value.privateKey)
      }
    }
  } catch (e) {
    sm2CryptoResult.value = ''
  }
}, { immediate: true })

// SM3 自动哈希
watch(sm3Input, () => {
  if (sm3Input.value) {
    sm3Hash.value = sm3(sm3Input.value)
  } else {
    sm3Hash.value = ''
  }
}, { immediate: true })

// SM4 自动加解密
watch([sm4Input, sm4Key, sm4Mode], () => {
  if (sm4Input.value && sm4Key.value.length === 32) {
    try {
      if (sm4Mode.value === 'encrypt') {
        sm4Result.value = sm4.encrypt(sm4Input.value, sm4Key.value)
      } else {
        sm4Result.value = sm4.decrypt(sm4Input.value, sm4Key.value)
      }
    } catch (e) {
      sm4Result.value = ''
    }
  } else {
    sm4Result.value = ''
  }
}, { immediate: true })

const handleSM2Sign = () => {
  if (!sm2Message.value) return toast.warning('请输入待签名消息')
  try {
    const sig = sm2.doSignature(sm2Message.value, sm2Keys.value.privateKey)
    sm2Signature.value = sig
    toast.success('签名成功')
  } catch (error) {
    toast.error('签名失败: ' + (error as Error).message)
  }
}

const handleSM2Verify = () => {
  if (!sm2Message.value || !sm2Signature.value) return toast.warning('请输入消息和签名值')
  try {
    const isVerified = sm2.doVerifySignature(sm2Message.value, sm2Signature.value, sm2Keys.value.publicKey)
    sm2VerifyResult.value = isVerified
    if (isVerified) {
      toast.success('验签成功！')
    } else {
      toast.error('验签失败')
    }
  } catch (error) {
    sm2VerifyResult.value = false
    toast.error('验签出错: ' + (error as Error).message)
  }
}

const handleSM2CryptoAction = () => {
  if (!sm2CryptoInput.value) return toast.warning('请输入待处理内容')
  
  try {
    if (sm2CryptoMode.value === 'encrypt') {
      if (sm2Keys.value.publicKey.length !== 130) return toast.error('请输入有效的 130 位公钥')
      const encrypted = sm2.doEncrypt(sm2CryptoInput.value, sm2Keys.value.publicKey)
      if (encrypted) {
        sm2CryptoResult.value = encrypted
        toast.success('SM2 加密成功')
      } else {
        toast.error('SM2 加密失败')
      }
    } else {
      if (sm2Keys.value.privateKey.length !== 64) return toast.error('请输入有效的 64 位私钥')
      const decrypted = sm2.doDecrypt(sm2CryptoInput.value, sm2Keys.value.privateKey)
      if (decrypted) {
        sm2CryptoResult.value = decrypted
        toast.success('SM2 解密成功')
      } else {
        sm2CryptoResult.value = ''
        toast.error('SM2 解密失败：请检查密文或私钥是否正确')
      }
    }
  } catch (error) {
    toast.error('SM2 操作失败: ' + (error as Error).message)
  }
}

const handleSM3Hash = () => {
  if (!sm3Input.value) return toast.warning('请输入待哈希文本')
  const hash = sm3(sm3Input.value)
  sm3Hash.value = hash
  toast.success('哈希计算完成')
}

const handleSM4Action = () => {
  if (!sm4Input.value) return toast.warning('请输入待处理内容')
  try {
    if (sm4Mode.value === 'encrypt') {
      const encrypted = sm4.encrypt(sm4Input.value, sm4Key.value)
      sm4Result.value = encrypted
      toast.success('SM4 加密成功')
    } else {
      const decrypted = sm4.decrypt(sm4Input.value, sm4Key.value)
      sm4Result.value = decrypted
      toast.success('SM4 解密成功')
    }
  } catch (error) {
    toast.error('SM4 操作失败: ' + (error as Error).message)
  }
}

const copyToClipboard = (text: string, label: string) => {
  if (!text) return
  navigator.clipboard.writeText(text)
  toast.info(`已复制 ${label} 到剪贴板`)
}
</script>

<template>
  <Toaster position="top-right" richColors :containerStyle="{ position: 'fixed', zIndex: '9999' }" />
  <div class="min-h-screen bg-[#F8F9FA] text-[#212529] font-sans selection:bg-blue-100 p-4 md:p-8">
    
    <div class="max-w-5xl mx-auto space-y-8">
      <!-- 顶部 -->
      <header class="flex flex-col md:flex-row md:items-center justify-between gap-4 border-b border-gray-200 pb-6">
        <div class="flex items-center gap-3">
          <div class="bg-blue-600 p-2.5 rounded-xl shadow-lg shadow-blue-200">
            <Shield class="w-6 h-6 text-white" />
          </div>
          <div>
            <h1 class="text-2xl font-bold tracking-tight text-gray-900">国密算法工具箱</h1>
            <p class="text-sm text-gray-500 font-medium">支持 SM2 (非对称) / SM3 (哈希) / SM4 (对称)</p>
          </div>
        </div>
        <div class="flex items-center gap-2">
          <button 
            @click="generateSM2Keys"
            class="inline-flex items-center gap-2 px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-200 rounded-lg hover:bg-gray-50 transition-colors shadow-sm"
          >
            <RefreshCw class="w-4 h-4" />
            生成新 SM2 密钥
          </button>
        </div>
      </header>

      <!-- 导航 -->
      <div class="flex p-1 bg-gray-100 rounded-xl">
        <button 
          v-for="tab in [
            { id: 'sm2', label: 'SM2 (非对称加密/签名)' },
            { id: 'sm3', label: 'SM3 (杂凑/哈希算法)' },
            { id: 'sm4', label: 'SM4 (分组/对称加密)' }
          ]"
          :key="tab.id"
          @click="activeTab = tab.id"
          :class="[
            'flex-1 py-2 text-sm font-medium rounded-lg transition-all duration-200',
            activeTab === tab.id ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500 hover:text-gray-700'
          ]"
        >
          {{ tab.label }}
        </button>
      </div>

      <!-- SM2 模块 -->
      <div v-if="activeTab === 'sm2'" class="grid grid-cols-1 lg:grid-cols-2 gap-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
        <!-- 密钥管理 -->
        <div class="bg-white rounded-2xl border border-gray-200 shadow-sm overflow-hidden">
          <div class="px-6 py-4 bg-gray-50/50 border-b border-gray-100">
            <h3 class="text-lg font-bold flex items-center gap-2">
              <Key class="w-5 h-5 text-blue-600" />
              SM2 密钥对
            </h3>
            <p class="text-xs text-gray-500 mt-1">用于签名验签及非对称加解密</p>
          </div>
          <div class="p-6 space-y-4">
            <div class="space-y-2">
              <div class="flex items-center justify-between">
                <label class="text-xs font-bold uppercase tracking-wider text-gray-400">公钥 (Hex)</label>
                <button @click="copyToClipboard(sm2Keys.publicKey, '公钥')" class="p-1 hover:bg-gray-100 rounded transition-colors text-gray-400">
                  <Copy class="w-3.5 h-3.5" />
                </button>
              </div>
              <textarea 
                v-model="sm2Keys.publicKey" 
                class="w-full font-mono text-xs p-3 bg-white rounded-lg border border-gray-200 h-20 resize-none focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all"
                placeholder="输入 130 位公钥..."
              ></textarea>
            </div>
            <div class="space-y-2">
              <div class="flex items-center justify-between">
                <label class="text-xs font-bold uppercase tracking-wider text-gray-400">私钥 (Hex)</label>
                <button @click="copyToClipboard(sm2Keys.privateKey, '私钥')" class="p-1 hover:bg-gray-100 rounded transition-colors text-gray-400">
                  <Copy class="w-3.5 h-3.5" />
                </button>
              </div>
              <textarea 
                v-model="sm2Keys.privateKey" 
                class="w-full font-mono text-xs p-3 bg-white rounded-lg border border-gray-200 h-12 resize-none focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all"
                placeholder="输入 64 位私钥..."
              ></textarea>
            </div>
          </div>
        </div>

        <!-- 签名与验签 -->
        <div class="bg-white rounded-2xl border border-gray-200 shadow-sm overflow-hidden">
          <div class="px-6 py-4 bg-gray-50/50 border-b border-gray-100">
            <h3 class="text-lg font-bold flex items-center gap-2">
              <FileSignature class="w-5 h-5 text-blue-600" />
              签名与验签
            </h3>
            <p class="text-xs text-gray-500 mt-1">验证消息的完整性与真实性</p>
          </div>
          <div class="p-6 space-y-4">
            <div class="space-y-2">
              <label class="text-sm font-medium text-gray-700">待签名消息</label>
              <input 
                v-model="sm2Message" 
                type="text"
                class="w-full px-3 py-2 bg-white border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all"
                placeholder="输入消息..."
              />
            </div>
            <div class="flex gap-2">
              <button @click="handleSM2Sign" class="flex-1 py-2 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700 transition-colors shadow-sm">签名</button>
              <button @click="handleSM2Verify" class="flex-1 py-2 bg-white text-gray-700 border border-gray-200 rounded-lg font-medium hover:bg-gray-50 transition-colors shadow-sm">验签</button>
            </div>
            <div class="space-y-2">
              <label class="text-xs font-bold uppercase tracking-wider text-gray-400">签名值 (Hex)</label>
              <textarea 
                v-model="sm2Signature" 
                class="w-full font-mono text-xs p-3 bg-gray-50 rounded-lg border border-gray-200 h-24 resize-none focus:outline-none"
                placeholder="签名结果将在此显示..."
              ></textarea>
            </div>
            <div v-if="sm2VerifyResult !== null" 
              :class="[
                'p-3 rounded-lg flex items-center gap-2 text-sm font-medium border animate-in zoom-in-95 duration-200',
                sm2VerifyResult ? 'bg-green-50 text-green-700 border-green-100' : 'bg-red-50 text-red-700 border-red-100'
              ]"
            >
              <CheckCircle2 v-if="sm2VerifyResult" class="w-4 h-4" />
              <AlertCircle v-else class="w-4 h-4" />
              {{ sm2VerifyResult ? '签名验证通过' : '签名验证失败' }}
            </div>
          </div>
        </div>

        <!-- 加解密 -->
        <div class="lg:col-span-2 bg-white rounded-2xl border border-gray-200 shadow-sm overflow-hidden">
          <div class="px-6 py-4 bg-gray-50/50 border-b border-gray-100">
            <h3 class="text-lg font-bold flex items-center gap-2">
              <Lock class="w-5 h-5 text-blue-600" />
              SM2 加解密
            </h3>
            <p class="text-xs text-gray-500 mt-1">公钥加密，私钥解密</p>
          </div>
          <div class="p-6 grid grid-cols-1 md:grid-cols-2 gap-8">
            <div class="space-y-6">
              <div class="space-y-2">
                <label class="text-sm font-medium text-gray-700">输入内容</label>
                <textarea 
                  v-model="sm2CryptoInput" 
                  class="w-full px-3 py-2 bg-white border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all min-h-[120px]"
                  :placeholder="sm2CryptoMode === 'encrypt' ? '输入明文...' : '输入密文 (Hex)...'"
                ></textarea>
              </div>
              <div class="flex p-1 bg-gray-100 rounded-lg">
                <button 
                  @click="sm2CryptoMode = 'encrypt'"
                  :class="['flex-1 py-1.5 text-xs font-bold rounded-md transition-all', sm2CryptoMode === 'encrypt' ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500']"
                >加密模式</button>
                <button 
                  @click="sm2CryptoMode = 'decrypt'"
                  :class="['flex-1 py-1.5 text-xs font-bold rounded-md transition-all', sm2CryptoMode === 'decrypt' ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500']"
                >解密模式</button>
              </div>
              <button @click="handleSM2CryptoAction" class="w-full py-3 bg-blue-600 text-white rounded-lg font-bold hover:bg-blue-700 transition-colors shadow-md flex items-center justify-center gap-2">
                <Lock v-if="sm2CryptoMode === 'encrypt'" class="w-4 h-4" />
                <Unlock v-else class="w-4 h-4" />
                {{ sm2CryptoMode === 'encrypt' ? '执行加密' : '执行解密' }}
              </button>
            </div>
            <div class="space-y-4">
              <div class="space-y-2">
                <div class="flex items-center justify-between">
                  <label class="text-xs font-bold uppercase tracking-wider text-gray-400">处理结果</label>
                  <button @click="copyToClipboard(sm2CryptoResult, '结果')" class="p-1 hover:bg-gray-100 rounded transition-colors text-gray-400">
                    <Copy class="w-3.5 h-3.5" />
                  </button>
                </div>
                <textarea 
                  v-model="sm2CryptoResult" 
                  readonly 
                  class="w-full font-mono text-sm p-4 bg-gray-50 rounded-xl border border-gray-200 min-h-[240px] focus:outline-none"
                  placeholder="结果将在此显示..."
                ></textarea>
              </div>
              <div class="p-4 bg-blue-50/50 rounded-xl border border-blue-100 text-sm text-blue-800">
                <p class="font-bold mb-2">算法说明：</p>
                <ul class="list-disc list-inside space-y-1.5 text-xs opacity-90">
                  <li>SM2 加密结果包含 C1 (椭圆曲线点), C2 (密文), C3 (杂凑值)。</li>
                  <li>加密过程需要 130 位的公钥。</li>
                  <li>解密过程需要 64 位的私钥。</li>
                  <li>通常用于小数据块加密或密钥交换。</li>
                </ul>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- SM3 模块 -->
      <div v-if="activeTab === 'sm3'" class="animate-in fade-in slide-in-from-bottom-4 duration-500">
        <div class="bg-white rounded-2xl border border-gray-200 shadow-sm overflow-hidden">
          <div class="px-6 py-4 bg-gray-50/50 border-b border-gray-100">
            <h3 class="text-lg font-bold flex items-center gap-2">
              <Hash class="w-5 h-5 text-blue-600" />
              SM3 杂凑算法
            </h3>
            <p class="text-xs text-gray-500 mt-1">256 位定长输出，不可逆</p>
          </div>
          <div class="p-6 space-y-6">
            <div class="space-y-2">
              <label class="text-sm font-medium text-gray-700">输入文本</label>
              <textarea 
                v-model="sm3Input" 
                class="w-full px-3 py-2 bg-white border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all min-h-[120px]"
                placeholder="输入需要计算哈希的文本..."
              ></textarea>
            </div>
            <button @click="handleSM3Hash" class="w-full py-3 bg-blue-600 text-white rounded-lg font-bold hover:bg-blue-700 transition-colors shadow-md">计算 SM3 哈希</button>
            <div class="space-y-2">
              <div class="flex items-center justify-between">
                <label class="text-xs font-bold uppercase tracking-wider text-gray-400">哈希结果 (Hex)</label>
                <button @click="copyToClipboard(sm3Hash, 'SM3 哈希')" class="p-1 hover:bg-gray-100 rounded transition-colors text-gray-400">
                  <Copy class="w-3.5 h-3.5" />
                </button>
              </div>
              <div class="p-4 bg-gray-50 rounded-xl border border-gray-200 font-mono text-sm break-all min-h-[60px] flex items-center justify-center">
                <span v-if="sm3Hash" class="text-gray-800">{{ sm3Hash }}</span>
                <span v-else class="text-gray-400 italic">等待计算...</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- SM4 模块 -->
      <div v-if="activeTab === 'sm4'" class="animate-in fade-in slide-in-from-bottom-4 duration-500">
        <div class="bg-white rounded-2xl border border-gray-200 shadow-sm overflow-hidden">
          <div class="px-6 py-4 bg-gray-50/50 border-b border-gray-100">
            <h3 class="text-lg font-bold flex items-center gap-2">
              <Lock class="w-5 h-5 text-blue-600" />
              SM4 分组加密
            </h3>
            <p class="text-xs text-gray-500 mt-1">128 位密钥，对称加密算法</p>
          </div>
          <div class="p-6 grid grid-cols-1 md:grid-cols-2 gap-8">
            <div class="space-y-6">
              <div class="space-y-2">
                <label class="text-sm font-medium text-gray-700">加密密钥 (32 位十六进制 / 128 位)</label>
                <input v-model="sm4Key" class="w-full px-3 py-2 font-mono text-sm bg-white border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all" />
              </div>
              <div class="space-y-2">
                <label class="text-sm font-medium text-gray-700">输入内容</label>
                <textarea 
                  v-model="sm4Input" 
                  class="w-full px-3 py-2 bg-white border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 outline-none transition-all min-h-[120px]"
                  :placeholder="sm4Mode === 'encrypt' ? '输入明文...' : '输入密文 (Hex)...'"
                ></textarea>
              </div>
              <div class="flex p-1 bg-gray-100 rounded-lg">
                <button 
                  @click="sm4Mode = 'encrypt'"
                  :class="['flex-1 py-1.5 text-xs font-bold rounded-md transition-all', sm4Mode === 'encrypt' ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500']"
                >加密模式</button>
                <button 
                  @click="sm4Mode = 'decrypt'"
                  :class="['flex-1 py-1.5 text-xs font-bold rounded-md transition-all', sm4Mode === 'decrypt' ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500']"
                >解密模式</button>
              </div>
              <button @click="handleSM4Action" class="w-full py-3 bg-blue-600 text-white rounded-lg font-bold hover:bg-blue-700 transition-colors shadow-md flex items-center justify-center gap-2">
                <Lock v-if="sm4Mode === 'encrypt'" class="w-4 h-4" />
                <Unlock v-else class="w-4 h-4" />
                {{ sm4Mode === 'encrypt' ? '执行加密' : '执行解密' }}
              </button>
            </div>
            <div class="space-y-4">
              <div class="space-y-2">
                <div class="flex items-center justify-between">
                  <label class="text-xs font-bold uppercase tracking-wider text-gray-400">处理结果</label>
                  <button @click="copyToClipboard(sm4Result, '结果')" class="p-1 hover:bg-gray-100 rounded transition-colors text-gray-400">
                    <Copy class="w-3.5 h-3.5" />
                  </button>
                </div>
                <textarea 
                  v-model="sm4Result" 
                  readonly 
                  class="w-full font-mono text-sm p-4 bg-gray-50 rounded-xl border border-gray-200 min-h-[240px] focus:outline-none"
                  placeholder="结果将在此显示..."
                ></textarea>
              </div>
              <div class="p-4 bg-amber-50 rounded-xl border border-amber-100 text-xs text-amber-800">
                <p class="font-bold mb-1 flex items-center gap-1">
                  <AlertCircle class="w-3 h-3" /> 注意：
                </p>
                <p>当前实现默认使用 ECB 模式。在生产环境中，建议使用带 IV 的 CBC 或 GCM 模式以增强安全性。</p>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 页脚 -->
      <footer class="text-center py-8 text-gray-400 text-sm">
        <p>© 2026 国密算法工具箱 • 基于 Vue 3 & sm-crypto 构建</p>
        <div class="flex justify-center gap-4 mt-3">
          <span class="px-2 py-0.5 bg-gray-100 rounded border border-gray-200 text-[10px] font-bold uppercase tracking-widest text-gray-500">SM2</span>
          <span class="px-2 py-0.5 bg-gray-100 rounded border border-gray-200 text-[10px] font-bold uppercase tracking-widest text-gray-500">SM3</span>
          <span class="px-2 py-0.5 bg-gray-100 rounded border border-gray-200 text-[10px] font-bold uppercase tracking-widest text-gray-500">SM4</span>
        </div>
      </footer>
    </div>
  </div>
</template>

<style>
@import "tailwindcss";

/* 强制 Toaster 固定在视口，防止随页面滚动 */
[data-sonner-toaster] {
  position: fixed !important;
  z-index: 9999 !important;
}

.animate-in {
  animation-duration: 0.5s;
  animation-fill-mode: both;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slide-in-from-bottom {
  from { transform: translateY(1rem); }
  to { transform: translateY(0); }
}

.fade-in {
  animation-name: fade-in;
}

.slide-in-from-bottom-4 {
  animation-name: slide-in-from-bottom;
}
</style>
