<script setup>
import { ref, onMounted, onUnmounted, computed, nextTick } from 'vue';
import { IconBell, IconBellRinging, IconX } from "@tabler/icons-vue";
import { getNotices } from '@/api/dashboard';
import { useToast } from '@/composables/useToast';
import { marked } from 'marked';
import DOMPurify from 'dompurify';

// 配置 marked
marked.setOptions({
  breaks: true,
  gfm: true
});

// 定义组件属性
const props = defineProps({
  autoRotate: {
    type: Boolean,
    default: true
  },
  rotateInterval: {
    type: Number,
    default: 8000
  }
});

// 定义emit事件
const emit = defineEmits(['popupOpened']);

// 定义响应式数据
const isOpen = ref(false);
const dropdown = ref(null);
const hasUnread = ref(true);
const notices = ref([]);
const showNoticeDetails = ref(false);
const loading = ref(true);
const selectedNotice = ref(null);

// 使用toast提示
const { showToast } = useToast();

// 公告列表
const announcementList = computed(() => {
  return notices.value.data || [];
});

// 是否有公告
const hasNotices = computed(() => {
  return announcementList.value.length > 0;
});

// 切换下拉菜单显示状态
const toggleDropdown = () => {
  isOpen.value = !isOpen.value;
  if (isOpen.value) {
    // 打开时标记为已读
    hasUnread.value = false;
  }
};

// 处理点击外部区域关闭下拉菜单
const handleClickOutside = (event) => {
  const selector = document.querySelector('.announcement');
  if (selector && !selector.contains(event.target)) {
    isOpen.value = false;
  }
};

// 获取公告列表
const fetchNotices = async () => {
  try {
    loading.value = true;
    const response = await getNotices();
    if (response && response.data) {
      notices.value = response;
      // 如果有公告，标记为未读
      if (response.data.length > 0) {
        hasUnread.value = true;
        // 检查是否有需要自动弹窗的公告
        checkForPopupNotices(response.data);
      }
    }
  } catch (error) {
    console.error('获取公告失败:', error);
    showToast('获取公告失败', 'error');
  } finally {
    loading.value = false;
  }
};

// 检查是否有需要自动弹窗的公告
const checkForPopupNotices = (noticesData) => {
  if (!noticesData || noticesData.length === 0) return;

  // 查找包含"弹窗"标签的公告（使用Unicode判断）
  const popupNoticeIndex = noticesData.findIndex(notice =>
    notice.tags && Array.isArray(notice.tags) && notice.tags.some(tag =>
      tag.includes('\u5f39\u7a97') // "弹窗"的Unicode编码
    )
  );

  if (popupNoticeIndex !== -1) {
    const noticeId = noticesData[popupNoticeIndex].id;
    const popupShownKey = `popup_notice_shown_${noticeId}`;

    // 检查是否已经显示过（使用sessionStorage避免重复弹窗）
    if (!sessionStorage.getItem(popupShownKey)) {
      // 设置已显示标记
      sessionStorage.setItem(popupShownKey, 'true');
      // 发出事件通知父组件公告弹窗已打开
      emit('popupOpened', true);
      // 显示弹窗
      showNoticeDetail(noticesData[popupNoticeIndex]);
    }
  }
};

// 将 Markdown 内容解析为安全的 HTML
const parsedContent = computed(() => {
  if (!selectedNotice.value?.content) return '';
  const rawHtml = marked(selectedNotice.value.content);
  return DOMPurify.sanitize(rawHtml);
});

// 显示公告详情弹窗
const showNoticeDetail = (notice) => {
  selectedNotice.value = notice;
  showNoticeDetails.value = true;
  // 标记为已读
  hasUnread.value = false;
  nextTick(() => {
    updateModalHeight();
  });
};

// 关闭公告详情弹窗
const closeNoticeModal = () => {
  showNoticeDetails.value = false;
  selectedNotice.value = null;
  // 发出事件通知父组件公告弹窗已关闭
  emit('popupOpened', false);
};

// 更新弹窗高度
const updateModalHeight = () => {
  // 这里可以添加弹窗高度调整逻辑
};

// 截取内容摘要
const getContentSummary = (content, length = 100) => {
  if (!content) return '';
  // 去除HTML标签
  const text = content.replace(/<[^>]*>/g, '');
  // 截取指定长度
  return text.length > length ? text.substring(0, length) + '...' : text;
};

// 组件挂载时添加事件监听器
onMounted(() => {
  document.addEventListener('click', handleClickOutside);
  fetchNotices();
});

// 组件卸载时移除事件监听器和定时器
onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});
</script>

<template>
  <div class="announcement">
    <button
        class="announcement-btn"
        id="announcement-btn"
        :class="{ shake: hasUnread && hasNotices }"
        @click="toggleDropdown"
        :title="$t('dashboard.siteAnnouncement')"
    >
      <span class="bell-icon">
        <transition name="bell-fade" mode="out-in">
          <IconBellRinging v-if="hasUnread && hasNotices" :size="20" class="alert-icon" />
          <IconBell v-else :size="20" class="alert-icon" />
        </transition>
      </span>
      <!-- 红点提醒 -->
      <span v-if="hasUnread && hasNotices" class="unread-dot"></span>
    </button>

    <transition name="fade">
      <div class="announcement-dropdown" v-if="isOpen" ref="dropdown">
        <div class="announcement-header">
          <h3>{{ $t('dashboard.siteAnnouncement') }}</h3>
        </div>
        <div class="announcement-list">
          <div v-if="loading" class="loading">
            <div class="skeleton-row"></div>
            <div class="skeleton-row"></div>
            <div class="skeleton-row"></div>
          </div>
          <template v-else>
            <div
                v-for="(notice, index) in announcementList"
                :key="index"
                class="announcement-item"
                @click="showNoticeDetail(notice)"
            >
              <div class="announcement-content">
                <h4>{{ notice.title }}</h4>
                <p>{{ getContentSummary(notice.content, 100) }}</p>
              </div>
              <div class="announcement-date">
                {{ new Date(notice.created_at * 1000).toLocaleDateString() }}
              </div>
            </div>
            <div v-if="announcementList.length === 0" class="no-announcements">
              {{ $t('dashboard.noAnnouncements') }}
            </div>
          </template>
        </div>
      </div>
    </transition>

    <!-- 公告弹窗 -->
    <Teleport to="body">
      <transition name="fade">
        <div v-if="showNoticeDetails" class="notice-modal-overlay" @click="closeNoticeModal">
          <transition name="popup-slide">
            <div v-if="showNoticeDetails" class="notice-modal" @click.stop>
              <div class="notice-modal-header">
                <h2 class="popup-title">{{ selectedNotice?.title }}</h2>
                <button class="popup-close-btn" @click="closeNoticeModal">
                  <IconX :size="20"/>
                </button>
              </div>
              <div class="notice-modal-content">
                <div class="notice-content markdown-body" v-html="parsedContent"></div>
              </div>
              <div class="notice-modal-footer">
                <button class="popup-action-btn adaptive-btn" @click="closeNoticeModal">
                  {{ $t('common.close') }}
                </button>
              </div>
            </div>
          </transition>
        </div>
      </transition>
    </Teleport>
  </div>
</template>


<style lang="scss" scoped>
.announcement {
  position: relative;
  display: inline-block;
}

.announcement-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: var(--card-background);
  border: 1px solid var(--border-color);
  color: var(--text-color);
  cursor: pointer;
  transition: all 0.3s ease;
  padding: 6px;
  overflow: hidden;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
  position: relative;

  &:hover {
    background-color: rgba(var(--theme-color-rgb), 0.1);
    border-color: var(--theme-color);
    transform: translateY(-2px);
    box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  }

  .bell-icon {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  /* 铃铛震动动画 */
  &.shake {
    animation: shake 0.5s ease-in-out;
  }
}

/* 红点提醒 */
.unread-dot {
  position: absolute;
  top: 5px;
  right: 5px;
  width: 8px;
  height: 8px;
  background-color: #ff4757;
  border-radius: 50%;
  box-shadow: 0 0 0 2px var(--card-background);
  animation: pulse 1.5s infinite;
}

/* 震动动画 */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  10%, 30%, 50%, 70%, 90% { transform: translateX(-2px); }
  20%, 40%, 60%, 80% { transform: translateX(2px); }
}

/* 红点脉冲动画 */
@keyframes pulse {
  0% { box-shadow: 0 0 0 0 rgba(255, 71, 87, 0.7); }
  70% { box-shadow: 0 0 0 5px rgba(255, 71, 87, 0); }
  100% { box-shadow: 0 0 0 0 rgba(255, 71, 87, 0); }
}

.announcement-dropdown {
  position: fixed;
  top: 68px;
  right: 20px;
  width: min(480px, calc(100vw - 40px));
  background: rgba(var(--card-background-rgb), 0.9);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  border: 1px solid var(--border-color);
  z-index: 200;
  overflow: hidden;
  max-height: 400px;
  display: flex;
  flex-direction: column;
}

.announcement-header {
  padding: 16px;
  border-bottom: 1px solid var(--border-color);
  display: flex;
  justify-content: space-between;
  align-items: center;

  h3 {
    margin: 0;
    font-size: 16px;
    font-weight: 600;
    color: var(--text-color);
  }
}

.announcement-list {
  max-height: 300px;
  overflow-y: auto;
  flex: 1;
}

.announcement-item {
  padding: 16px;
  border-bottom: 1px solid var(--border-color);
  cursor: pointer;
  transition: all 0.2s ease;

  &:last-child {
    border-bottom: none;
  }

  &:hover {
    background-color: rgba(var(--theme-color-rgb), 0.05);
  }

  h4 {
    margin: 0 0 8px 0;
    font-size: 15px;
    font-weight: 600;
    color: var(--text-color);
  }

  p {
    margin: 0 0 12px 0;
    font-size: 13px;
    color: var(--text-muted);
    line-height: 1.4;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
}

.announcement-date {
  font-size: 12px;
  color: var(--text-muted);
  text-align: right;
}

.no-announcements {
  padding: 24px;
  text-align: center;
  color: var(--text-muted);
  font-size: 14px;
}

.loading {
  padding: 16px;

  .skeleton-row {
    height: 16px;
    margin-bottom: 12px;
    background-color: rgba(0, 0, 0, 0.05);
    border-radius: 4px;

    &:last-child {
      margin-bottom: 0;
    }
  }

  .skeleton-row:first-child {
    width: 80%;
  }

  .skeleton-row:nth-child(2) {
    width: 100%;
  }

  .skeleton-row:last-child {
    width: 60%;
  }
}

/* 弹窗样式 */
.notice-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  box-sizing: border-box;
  backdrop-filter: blur(4px);
}

.notice-modal {
  width: 100%;
  max-width: 500px;
  background-color: rgba(var(--card-background-rgb, 255, 255, 255), 1);
  border-radius: 16px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.15);
  border: 1px solid rgba(var(--theme-color-rgb), 0.15);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 80vh;
  animation: modal-in 0.3s cubic-bezier(0.16, 1, 0.3, 1);

  @media (prefers-color-scheme: dark) {
    background-color: rgba(var(--card-background-rgb, 30, 30, 30), 1);
  }
}

.notice-modal-header {
  padding: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid var(--border-color);
  background-color: rgba(var(--theme-color-rgb), 0.03);

  .popup-title {
    margin: 0;
    font-size: 18px;
    font-weight: 600;
    color: var(--text-color);
  }

  .popup-close-btn {
    background: none;
    border: none;
    cursor: pointer;
    color: var(--secondary-text-color);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 8px;
    margin: -8px;
    border-radius: 50%;
    transition: all 0.3s ease;

    &:hover {
      background-color: rgba(0, 0, 0, 0.05);
      color: var(--text-color);
      transform: rotate(90deg);
    }
  }
}

.notice-modal-content {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
  background: linear-gradient(to bottom, rgba(var(--theme-color-rgb), 0.02), transparent);

  .notice-content {
    font-size: 14px;
    line-height: 1.6;
    color: var(--text-color);
  }

  /* Markdown 排版样式 */
  .markdown-body {
    :deep(h1), :deep(h2), :deep(h3), :deep(h4) {
      margin: 16px 0 8px;
      font-weight: 600;
      color: var(--text-color);
    }
    :deep(h1) { font-size: 1.4em; }
    :deep(h2) { font-size: 1.2em; }
    :deep(h3) { font-size: 1.1em; }

    :deep(p) {
      margin: 8px 0;
    }

    :deep(img) {
      max-width: 100%;
      border-radius: 8px;
      margin: 8px 0;
    }

    :deep(a) {
      color: var(--theme-color);
      text-decoration: none;
      &:hover { text-decoration: underline; }
    }

    :deep(code) {
      background: rgba(0, 0, 0, 0.06);
      padding: 2px 6px;
      border-radius: 4px;
      font-size: 0.9em;
    }

    :deep(pre) {
      background: rgba(0, 0, 0, 0.06);
      padding: 12px;
      border-radius: 8px;
      overflow-x: auto;
      code {
        background: none;
        padding: 0;
      }
    }

    :deep(ul), :deep(ol) {
      padding-left: 20px;
      margin: 8px 0;
    }

    :deep(blockquote) {
      border-left: 3px solid var(--theme-color);
      padding-left: 12px;
      margin: 8px 0;
      color: var(--secondary-text-color);
    }

    :deep(hr) {
      border: none;
      border-top: 1px solid var(--border-color);
      margin: 16px 0;
    }

    :deep(table) {
      width: 100%;
      border-collapse: collapse;
      margin: 8px 0;
      th, td {
        border: 1px solid var(--border-color);
        padding: 8px;
        text-align: left;
      }
      th {
        background: rgba(0, 0, 0, 0.03);
      }
    }
  }
}

.notice-modal-footer {
  padding: 15px 20px;
  border-top: 1px solid var(--border-color);
  display: flex;
  justify-content: flex-end;

  .popup-action-btn {
    padding: 8px 20px;
    background-color: var(--theme-color);
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    min-width: 120px;

    &.adaptive-btn {
      min-width: auto;
      padding: 8px 20px;
      display: inline-flex;
      align-items: center;
      justify-content: center;
    }

    &:hover:not(:disabled) {
      transform: translateY(-2px);
      box-shadow: 0 4px 10px rgba(var(--theme-color-rgb), 0.3);
    }

    &:disabled {
      opacity: 0.7;
      cursor: not-allowed;
      background-color: var(--secondary-text-color);
    }
  }
}

.popup-slide-enter-active {
  transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}

.popup-slide-leave-active {
  transition: all 0.2s ease-out;
}

.popup-slide-enter-from {
  opacity: 0;
  transform: translateY(20px) scale(0.98);
}

.popup-slide-leave-to {
  opacity: 0;
  transform: scale(0.95);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.bell-fade-enter-active,
.bell-fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.bell-fade-enter-from {
  opacity: 0;
  transform: scale(0.8);
}

.bell-fade-leave-to {
  opacity: 0;
  transform: scale(1.2);
}

@media (max-width: 576px) {
  .announcement-dropdown {
    width: min(480px, calc(100vw - 40px));
    left: 50%;
    right: auto;
    transform: translateX(-50%);
  }

  .notice-modal-overlay {
    padding: 15px;

    .notice-modal {
      max-width: 100%;
      max-height: 85vh;

      .notice-modal-header {
        padding: 15px;

        .popup-title {
          font-size: 16px;
        }
      }

      .notice-modal-content {
        padding: 15px;
      }

      .notice-modal-footer {
        padding: 12px 15px;
      }
    }
  }
}
</style>
