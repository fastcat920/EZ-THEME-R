<script setup>
import { IconArrowsShuffle2, IconMoodCrazyHappy } from "@tabler/icons-vue";
import { checkInReq } from "@/api/user";
import { ref } from "vue";
import { useToast } from '@/composables/useToast';
import LuckyModal from "./LuckyModal.vue";

const emit = defineEmits(['checkinSuccess']);
const { showToast } = useToast();

const loading = ref(false);
const modalRef = ref(null);

const onLuckyCheckinSuccess = () => {
  // 传递幸运签到的成功事件给父组件
  emit('checkinSuccess');
};

const checkin = async (type) => {
  if(type ==1) {
    try {
      loading.value = true;
      const res = await checkInReq({type})
      const { message, data } = res;
      showToast(message, data ? 'success' : 'error', data ? 0 : 3000);
      
      // 签到成功后发出事件通知
      if (data) {
        emit('checkinSuccess');
      }
    } catch (e) {
      console.log(e)
    } finally {
      loading.value = false;
    }
  } else if(type == 2) {
    modalRef.value?.openModal()
  }
}
</script>

<template>
  <div class="checkin-container">
    <button class="btn-primary withdraw-btn" @click="checkin(1)">
      <div v-if="loading" class="loader"></div>
      <IconArrowsShuffle2 v-else class="btn-icon" />
      普通签到
    </button>
    <button class="btn-primary lucky-checkin" @click="checkin(2)">
      <IconMoodCrazyHappy class="btn-icon" />
      运气签到
    </button>

    <LuckyModal ref="modalRef" @checkin-success="onLuckyCheckinSuccess" />
  </div>
</template>

<style scoped lang="scss">
.checkin-container {
  margin-bottom: 16px;
  .btn-primary, .btn-outline, .btn-action {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 8px 16px;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    box-sizing: border-box;

    .btn-icon {
      width: 18px;
      height: 18px;
      display: inline-flex;
      vertical-align: middle;
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }
  }

  .btn-primary {
    height: 42px;
    min-width: 110px;
    padding: 0 16px;
    background-color: var(--theme-color);
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 500;
    box-shadow: 0 2px 5px rgba(var(--theme-color-rgb), 0.2);
    transition: all 0.3s ease;

    &:hover:not(:disabled) {
      background-color: var(--theme-hover-color, #3070e0);
      transform: translateY(-1px);
      box-shadow: 0 4px 8px rgba(var(--theme-color-rgb), 0.3);
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }
  }

  .btn-primary {
    height: 44px;
    font-weight: 600;
    letter-spacing: 0.3px;
    padding: 0 20px;
    border-radius: 10px;
    transition: all 0.3s ease;
    min-width: 140px;
    box-shadow: 0 2px 8px rgba(var(--theme-color-rgb), 0.25);

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(var(--theme-color-rgb), 0.35);
    }
  }

  .withdraw-btn {
    background: transparent !important;
    border: 1.5px solid var(--theme-color) !important;
    color: var(--theme-color) !important;
    transition: all 0.3s ease !important;
    margin-right: 16px;

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
    }

    .btn-icon {
      color: var(--theme-color);
    }
  }

  @media (max-width: 768px) {
    width: 100%;
    margin-top: 10px;
    display: flex;
    gap: 10px;

    .btn-primary {
      flex: 1;
      min-width: 0;
      justify-content: center;
    }

    .withdraw-btn {
      margin-right: 0;
    }
  }
}

.loader {
  width: 18px;
  height: 18px;
  min-width: 18px;
  min-height: 18px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: white;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
