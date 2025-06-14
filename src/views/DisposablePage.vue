<template>
  <v-container>
    <v-row>
      <v-col cols="12" md="8" class="mx-auto">
        <!-- 消息进度条列表，只显示标题 -->
        <div
          v-for="(draft, idx) in drafts"
          :key="draft.title"
          class="mb-2 position-relative"
          style="width: 100%;"
        >
          <progressLine
            :hight="4"
            :ma="6"
            :duration="1000"
            :payload="idx"
            @endPress="preview"
            @mousedown="addToBuffer(idx)"
          >
            <div class="d-flex flex-column">
              <div class="d-flex justify-space-between align-center">
                <span class="text-subtitle-1">{{ draft.title }}</span>
              </div>
            </div>
          </progressLine>
          <div class="action-buttons">
            <v-btn
              icon="mdi-send"
              variant="text"
              size="small"
              color="primary"
              class="transfer-btn"
              @click.stop="startTransfer(idx)"
            ></v-btn>
            <v-btn
              icon="mdi-delete"
              variant="text"
              size="small"
              color="error"
              class="delete-btn"
              @click.stop="deleteDraft(idx)"  
            ></v-btn>
          </div>
        </div>
        

        <!-- 空状态时显示默认的进度条 -->
        <template v-if="drafts.length === 0">
          <div style="width: 100%;">
            <progressLine :hight="4" :ma="6">这里还没有消息噢</progressLine>
          </div>
        </template>

        <!-- 加号输入框（不传 payload，不绑定 endPress） -->
        <div class="mb-2" style="width: 100%;">
          <v-btn 
            color="primary" 
            block 
            @click="navigateToEdit"
            class="add-btn"
          >
            新建消息
          </v-btn>
        </div>

        <!-- 预览内容弹窗 -->
        <v-dialog v-model="previewDialog" max-width="500px">
          <v-card v-if="previewIndex !== null && drafts[previewIndex]">
            <v-card-title>
              {{ drafts[previewIndex].title }}
              <v-spacer></v-spacer>
              <v-btn icon="mdi-close" variant="text" @click="closePreview"></v-btn>
            </v-card-title>
            <v-card-text>
              {{ drafts[previewIndex].body }}
            </v-card-text>
          </v-card>
        </v-dialog>

        <!-- 添加碰一碰提示组件 -->
        <touch-prompt v-model="touching" :prompt="touchingPrompt"></touch-prompt>

        <!-- 添加成功/失败提示 -->
        <v-snackbar :timeout="2000" color="green-lighten-3" v-model="transferSuccess">
          消息传递成功
        </v-snackbar>
        <v-snackbar :timeout="2000" color="red-lighten-1" v-model="transferError">
          消息传递失败
        </v-snackbar>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup lang="ts">
import { ref, onMounted, watchEffect } from 'vue'
import progressLine from "@/components/progress-linear.vue"
import touchPrompt from "@/components/touch-prompt.vue"
import { useRouter } from 'vue-router'  // 添加路由导入
// import MessageInput from "../components/MessageInput.vue"
// import EditView from "./EditView.vue"  // 添加 EditView 导入
import { MessageDraft } from '@/types'
import { try_invoke } from '@/utils/utils'
import { listen } from '@tauri-apps/api/event'

const drafts = ref<MessageDraft[]>([])
const router = useRouter()  // 获取路由实例
// const newMessageId = ref(randomUUID())

const navigateToEdit = () => {
  router.push({ 
    name: 'edit',  // 路由名称
    params: { 
      type: 'Disposable',
      id: String(drafts.value.length)  // 使用数组长度作为 id
    }
  })
}

async function saveDrafts() {
  await try_invoke('store_disposable_drafts', { data: { drafts: drafts.value } })
}

async function loadDrafts() {
  try {
    const result = await try_invoke('load_disposable_drafts') as { drafts: MessageDraft[] }
    drafts.value = result.drafts || []
  } catch (e) {
    drafts.value = []
  }
}
const deleteDraft = async (index: number) => {
  console.log(index)
  if (index >= 0 && index < drafts.value.length) {
    // 从数组中删除指定索引的元素
    drafts.value.splice(index, 1)
    // 保存更新后的草稿
    await saveDrafts()
  }
}

// const addMessage = async (message: { title: string; content: string }) => {
//   const draft: MessageDraft = {
//     title: message.title,
//     body: message.content
//   }
//   drafts.value.unshift(draft)
//   await saveDrafts()
// }

const previewDialog = ref(false)
const previewIndex = ref<number | null>(null)

const preview = (payload: unknown) => {
  if (typeof payload === 'number' && payload >= 0 && payload < drafts.value.length) {
    previewIndex.value = payload
    previewDialog.value = true
  }
}

const closePreview = () => {
  previewDialog.value = false
  previewIndex.value = null
}

// const deleteDraft = async (title: string) => {
//   drafts.value = drafts.value.filter(draft => draft.title !== title)
//   await saveDrafts()
// }

const touching = ref(false)
const touchingPrompt = ref("")
const transferSuccess = ref(false)
const transferError = ref(false)
const transferringIndex = ref<number | null>(null)

const addToBuffer = async (idx: number) => {
  if (idx >= 0 && idx < drafts.value.length) {
    const selectedDraft = drafts.value[idx];
    try {
      await try_invoke('set_disposable_msg', { msg: selectedDraft.body });
    } catch (e) {
      console.error('Failed to set message:', e);
    }
  }
}

const startTransfer = async (idx: number) => {
  if (idx >= 0 && idx < drafts.value.length) {
    const selectedDraft = drafts.value[idx];
    try {
      await try_invoke('set_disposable_msg', { msg: selectedDraft.body });
      touchingPrompt.value = "传递一次性消息";
      touching.value = true;
      transferringIndex.value = idx;
    } catch (e) {
      console.error('Failed to set message:', e);
      transferError.value = true;
    }
  }
}

// 监听消息接收
(async () => {
  await listen<string>("recv-disposable-msg", async (event) => {
    if (touching.value && transferringIndex.value !== null) {
      // 消息传递成功，删除该消息
      drafts.value.splice(transferringIndex.value, 1);
      await saveDrafts();
      touching.value = false;
      transferringIndex.value = null;
      transferSuccess.value = true;
    }
  });
})();

// 清理消息
watchEffect(async () => {
  if (touching.value == false) {
    await try_invoke('clear_msg', {});
    transferringIndex.value = null;
  }
});

onMounted(() => {
  loadDrafts()
})
</script>

<style scoped>
.mb-2 {
  margin-bottom: 8px;
}
.action-buttons {
  position: absolute;
  right: 5px;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
  display: flex;
  gap: 4px;
}
.transfer-btn {
  opacity: 0.8;
  transition: opacity 0.2s;
}
.transfer-btn:hover {
  opacity: 1;
}
.delete-btn {
  position: absolute;
  right: 35px;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
}
.add-btn {
  margin-top: 10px;
}
</style>
