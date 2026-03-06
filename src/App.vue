<template>
  <div>
    <!-- 静态布局容器，包含不需要过渡效果的菜单和按钮 -->
    <div class="static-layout" v-if="$route.meta.requiresAuth">
      <!-- 网站名称 -->
      <div class="site-logo">
        <img v-if="siteConfig.showLogo" src="/images/logo.png" alt="Logo" class="site-logo-img" />
        {{ siteConfig.siteName }}
      </div>

      <!-- 顶部导航栏 - 保持不变 -->
      <SlideTabsNav />

      <!-- 顶部工具栏：语言选择器、主题切换和用户头像 -->
      <div class="top-toolbar">
        <Announcement @popup-opened="handleAnnouncementPopup" />
        <!-- Desktop version -->
        <div class="desktop-toolbar">
          <ThemeToggle />
          <LanguageSelector />
          <button
            v-if="SHOP_CONFIG.enableCoupons"
            class="coupon-btn"
            id="coupon-btn"
            @click="openCouponModal"
            title="优惠券"
          >
            <IconTag :size="20" />
          </button>
          <button
            v-if="PROFILE_CONFIG.showGiftCardRedeem"
            class="gift-btn"
            @click="$router.push('/profile')"
          >
            <IconGift :size="20" />
          </button>
          <UserAvatar :username="username" :avatarUrl="avatarUrl" id="user-avatar" />
        </div>
        <!-- Mobile version -->
        <div class="mobile-toolbar">
          <button
            v-if="SHOP_CONFIG.enableCoupons"
            class="coupon-btn"
            id="mobile-coupon-btn"
            @click="openCouponModal"
            title="优惠券"
          >
            <IconTag :size="20" />
          </button>
          <button
            v-if="PROFILE_CONFIG.showGiftCardRedeem"
            class="gift-btn"
            @click="$router.push('/profile')"
          >
            <IconGift :size="20" />
          </button>
          <MobileToolbar />
        </div>
      </div>
    </div>

    <!-- 认证页面顶部工具栏，确保认证页面也有语言切换器 -->
    <div class="auth-toolbar" v-if="!$route.meta.requiresAuth && $route.path.includes('/auth')">
      <div class="top-toolbar">
        <div class="desktop-toolbar">
          <ThemeToggle />
          <LanguageSelector />
        </div>
      </div>
    </div>

    <!-- 路由视图只对内容部分应用过渡效果 -->
    <router-view v-slot="{ Component, route }">
      <transition
        name="page-transition"
        mode="out-in"
        appear
      >
        <keep-alive :include="cachedRoutes" :max="5">
          <component
            :is="Component"
            :key="route.path"
            :is-active="true"
          />
        </keep-alive>
      </transition>
    </router-view>

    <!-- 全局Toast通知 - 放在最外层，确保不受页面切换影响 -->
    <Toast />

    <!-- 优惠券模态框 -->
    <CouponModal />

    <!-- 返回顶部按钮 -->
    <BackToTop />

    <!-- 自定义鼠标右键菜单 -->
    <CustomContextMenu />

    <!-- 客服图标 -->
    <CustomerServiceIcon v-if="$route.path !== '/customer-service'" />

    <!-- Crisp嵌入组件 -->
    <CrispEmbed v-if="customerServiceConfig.embedMode === 'embed' && customerServiceConfig.type === 'crisp'" />

    <!-- Chatwoot嵌入组件 -->
    <ChatwootEmbed v-if="customerServiceConfig.embedMode === 'embed' && customerServiceConfig.type === 'chatwoot'" />

    <!-- 资源预加载组件 -->
    <ResourcePreloader />

    <!-- SVG图标定义 -->
    <IconDefinitions />
  </div>
</template>

<script>
import { onMounted, onUnmounted, ref, computed, provide, watch, nextTick } from 'vue';
import { useStore } from 'vuex';
import { useTheme } from '@/composables/useTheme';
import { useRouter, useRoute } from 'vue-router';
import { SITE_CONFIG, PROFILE_CONFIG, CUSTOMER_SERVICE_CONFIG, SHOP_CONFIG } from '@/utils/baseConfig';
import { checkAuthAndReloadMessages } from '@/utils/authUtils';
import { checkUserLoginStatus } from '@/api/auth';
import { handleRedirectPath } from '@/utils/redirectHandler';
import { createDriver, shouldShowTour, announcementSteps, couponSteps, themeToggleSteps, languageSelectorSteps, mobileMoreSteps, helpDocumentSteps, mobileUserMenuSteps } from '@/utils/driver';
import Toast from '@/components/common/Toast.vue';
import CouponModal from '@/components/common/CouponModal.vue';
import IconDefinitions from '@/components/icons/IconDefinitions.vue';
import SlideTabsNav from '@/components/common/SlideTabsNav.vue';
import ThemeToggle from '@/components/common/ThemeToggle.vue';
import LanguageSelector from '@/components/common/LanguageSelector.vue';
import UserAvatar from '@/components/common/UserAvatar.vue';
import MobileToolbar from '@/components/common/MobileToolbar.vue';
import BackToTop from '@/components/common/BackToTop.vue';
import Announcement from '@/components/common/Announcement.vue';
import CustomContextMenu from '@/components/common/CustomContextMenu.vue';
import CustomerServiceIcon from '@/components/common/CustomerServiceIcon.vue';
import CrispEmbed from '@/components/common/CrispEmbed.vue';
import ChatwootEmbed from '@/components/common/ChatwootEmbed.vue';
import ResourcePreloader from '@/components/common/ResourcePreloader.vue';
import { IconGift, IconTag } from '@tabler/icons-vue';
import NProgress from 'nprogress';
import 'nprogress/nprogress.css';
import pageCache from '@/utils/pageCache';

NProgress.configure({
  showSpinner: true,
  easing: 'ease',
  speed: 400,
  minimum: 0.2
});

export default {
  name: 'App',
  components: {
    Toast,
    CouponModal,
    IconDefinitions,
    SlideTabsNav,
    ThemeToggle,
    LanguageSelector,
    UserAvatar,
    BackToTop,
    CustomContextMenu,
    CustomerServiceIcon,
    CrispEmbed,
    ChatwootEmbed,
    ResourcePreloader,
    IconGift,
    IconTag,
    Announcement,
    MobileToolbar
  },
  setup() {
    const router = useRouter();
    const route = useRoute();
    const store = useStore();
    const { applyTheme } = useTheme();
    const siteConfig = ref(SITE_CONFIG);
    const cachedRoutes = computed(() => pageCache.getCachedRoutes());

    const customerServiceConfig = computed(() => CUSTOMER_SERVICE_CONFIG);

    router.beforeEach((to, from, next) => {
      if (to.meta.keepAlive && to.name) {
        pageCache.addRouteToCache(to.name);
      }

      if (from.name && from.meta.keepAlive === false) {
        pageCache.removeRouteFromCache(from.name);
      }

      NProgress.start();
      next();
    });

    router.afterEach(() => {
      NProgress.done();
    });

    const handleRedirectParam = () => {
      let redirectParam = null;

      const hashParts = window.location.hash.split('?');
      if (hashParts.length > 1) {
        const hashParams = new URLSearchParams(hashParts[1]);
        redirectParam = hashParams.get('redirect');
      }

      if (!redirectParam) {
        redirectParam = route.query.redirect;
      }

      if (redirectParam && typeof redirectParam === 'string') {
        const targetPath = handleRedirectPath(redirectParam);

        if (route.path !== targetPath) {
          router.replace(targetPath);
        }
      }
    };

    watch(() => route.fullPath, () => {
      handleRedirectParam();
    });

    const username = computed(() => store.getters.username);
    const avatarUrl = computed(() => store.getters.avatarUrl || '');
    const isLoggedIn = computed(() => store.getters.isLoggedIn);

    const openCouponModal = () => {
      // Create a custom event to open coupon modal
      const event = new CustomEvent('openCouponModal');
      window.dispatchEvent(event);
    };

    const languageChangedSignal = ref(0);
    const isAnnouncementPopupOpen = ref(false);
    const driverInstance = ref(null);

    const onLanguageChanged = () => {
      languageChangedSignal.value++;

      setTimeout(() => {
        document.body.classList.add('language-transitioning');
        setTimeout(() => {
          document.body.classList.remove('language-transitioning');
        }, 300);
      }, 0);
    };

    // 处理公告弹窗状态变化
    const handleAnnouncementPopup = (isOpen) => {
      isAnnouncementPopupOpen.value = isOpen;

      // 如果公告弹窗关闭且存在引导实例，则启动引导
      if (!isOpen && driverInstance.value) {
        // 稍微延迟启动引导，确保弹窗完全关闭
        setTimeout(() => {
          driverInstance.value.drive();
        }, 500);
      }
    };

    const handleVisibilityChange = () => {
      if (!document.hidden) {
        checkAuthAndReloadMessages();

        checkUserLoginStatus().then(result => {
          if (result.isLoggedIn === false && result.message) {
            const { showToast } = require('@/composables/useToast').useToast();
            if (showToast) {
              showToast(result.message, 'warning');
            }
          }
        }).catch(err => {
          console.error('检查登录状态出错:', err);
        });
      }
    };

    // 新手引导相关
    const initTour = () => {
      // 检查是否应该显示新手引导
      if (shouldShowTour() && isLoggedIn.value) {
        // 等待DOM更新后再启动引导
        nextTick(() => {
          // 检查是否为移动端
          const isMobile = window.innerWidth <= 768;

          let allSteps = [];

          if (isMobile) {
            // 移动端引导流程：公告 -> 优惠券 -> 更多按钮 -> 帮助文档
            allSteps = [
              ...announcementSteps,
              ...couponSteps.slice(1, 2), // 只添加移动端优惠券步骤
              ...mobileMoreSteps,
              ...helpDocumentSteps
            ];
          } else {
            // 桌面端引导流程：公告 -> 优惠券 -> 主题切换 -> 语言切换 -> 帮助文档
            allSteps = [
              ...announcementSteps,
              ...couponSteps.slice(0, 1), // 只添加桌面端优惠券步骤
              ...themeToggleSteps.slice(0, 1), // 只添加桌面端主题切换步骤
              ...languageSelectorSteps.slice(0, 1), // 只添加桌面端语言切换步骤
              ...helpDocumentSteps // 添加帮助文档步骤
            ];
          }

          // 创建引导实例
          const driverObj = createDriver(allSteps);

          // 将driverObj存储在ref中，以便在其他地方可以访问
          driverInstance.value = driverObj;

          // 延迟启动引导，确保元素已经渲染
          // 但只有当公告弹窗未打开时才启动
          setTimeout(() => {
            if (!isAnnouncementPopupOpen.value) {
              driverObj.drive();
            }
          }, 3000);
        });
      }
    };

    provide('languageChangedSignal', languageChangedSignal);

    const clearCache = () => {
      pageCache.clearCache();
    };

    const removeCachedRoute = (routeName) => {
      pageCache.removeRouteFromCache(routeName);
    };

    provide('clearCache', clearCache);
    provide('removeCachedRoute', removeCachedRoute);

    onMounted(() => {
      window.addEventListener('languageChanged', onLanguageChanged);

      applyTheme(store.getters.currentTheme);

      // 将客服气泡位置配置注入 CSS 变量（复用 iconPosition.mobile）
      const mobilePos = CUSTOMER_SERVICE_CONFIG.iconPosition?.mobile;
      if (mobilePos?.bottom) {
        document.documentElement.style.setProperty('--chatwoot-bubble-bottom', mobilePos.bottom);
      }
      if (mobilePos?.right) {
        document.documentElement.style.setProperty('--chatwoot-bubble-right', mobilePos.right);
      }

      checkAuthAndReloadMessages();

      document.addEventListener('visibilitychange', handleVisibilityChange);

      checkUserLoginStatus().then(result => {
        if (result.isLoggedIn === false && result.message) {
          const { showToast } = require('@/composables/useToast').useToast();
          if (showToast) {
            showToast(result.message, 'warning');
          }
        }
      }).catch(err => {
        console.error('检查登录状态出错:', err);
      });

      handleRedirectParam();

      // 初始化新手引导
      initTour();
    });

    onUnmounted(() => {
      window.removeEventListener('languageChanged', onLanguageChanged);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    });

    return {
      username,
      avatarUrl,
      siteConfig,
      PROFILE_CONFIG,
      SHOP_CONFIG,
      cachedRoutes,
      customerServiceConfig,
      openCouponModal,
      isAnnouncementPopupOpen,
      driverInstance,
      handleAnnouncementPopup,
    };
  }
};
</script>

<style lang="scss">
@use "sass:math";
@use "@/assets/styles/base/variables.scss" as *;
@use "@/assets/styles/base/reset.scss" as *;
@use "@/assets/styles/base/animations.scss" as *;
@use "@/assets/styles/base/scrollbar.scss" as *;


.page-transitioning {
  overflow: hidden;
}


.static-layout {
  position: fixed;
  width: 100%;
  top: 0;
  left: 0;
  z-index: 100;
}


.site-logo {
  position: fixed;
  top: 20px;
  left: 25px;
  font-size: 20px;
  font-weight: 700;
  color: var(--theme-color);
  z-index: 110;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  letter-spacing: -0.5px;
  background-color: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  padding: 6px 14px;
  border-radius: 10px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 10px;

  .site-logo-img {
    height: 24px;
    width: 24px;
    border-radius: 6px;
    object-fit: cover;
  }
}


.dark-theme .site-logo {
  background-color: rgba(30, 30, 30, 0.7);
}


.top-toolbar {
  position: fixed;
  top: 20px;
  right: 25px;
  display: flex;
  gap: 12px;
  z-index: 110;

  .coupon-btn,
  .gift-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 38px;
    height: 38px;
    border-radius: 50%;
    background-color: rgba(var(--theme-color-rgb), 0.1);
    border: 1px solid rgba(var(--theme-color-rgb), 0.3);
    color: var(--theme-color);
    cursor: pointer;
    transition: all 0.3s ease;

    &:hover {
      box-shadow: 0 0 0 3px rgba(var(--theme-color-rgb), 0.15);
      transform: translateY(-2px);
    }
  }
}

.desktop-toolbar {
  display: flex;
  gap: 12px;
}

.mobile-toolbar {
  display: none;
  gap: 12px;
}


@media (max-width: 768px) {
  .site-logo {
    top: 12px;
    left: 20px;
    font-size: 20px;
    padding: 5px 10px;
    border-radius: 8px;
  }

  .top-toolbar {
    top: 12px;
    right: 20px;
    gap: 10px;
  }

  .desktop-toolbar {
    display: none;
  }

  .mobile-toolbar {
    display: flex;
    gap: 10px;
  }

  main, .main-content, .content-container {
    padding-bottom: 70px !important;
    margin-bottom: 10px !important;
  }
}


.page-transition-enter-active,
.page-transition-leave-active {
  transition: opacity 0.3s ease;
}

.page-transition-enter-from {
  opacity: 0;
}

.page-transition-leave-to {
  opacity: 0;
}


.language-transitioning .language-transition-item {
  animation: language-fade 0.3s ease-out;
}

@keyframes language-fade {
  0% {
    opacity: 0.2;
  }
  100% {
    opacity: 1;
  }
}


.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}


::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background-color: var(--input-bg-color, rgba(0, 0, 0, 0.05));
  border-radius: 3px;
}

::-webkit-scrollbar-thumb {
  background-color: var(--theme-color);
  border-radius: 3px;
  opacity: 0.7;
  transition: background-color 0.3s ease;
}

::-webkit-scrollbar-thumb:hover {
  background-color: var(--theme-hover-color, rgba(var(--theme-color-rgb), 0.8));
}

::-webkit-scrollbar-corner {
  background-color: transparent;
}


* {
  scrollbar-width: thin;
  scrollbar-color: var(--theme-color) var(--input-bg-color, rgba(0, 0, 0, 0.05));
}


html {
  scroll-behavior: smooth;
}


.auth-toolbar {
  position: fixed;
  top: 0;
  right: 0;
  z-index: 100;

  .top-toolbar {
    position: fixed;
    top: 20px;
    right: 25px;
    display: flex;
    gap: 12px;
    z-index: 110;
  }
}


.eztheme-btn {
  text-decoration: none !important;
  border-bottom: none !important;
  background-image: none !important;
  background-repeat: no-repeat !important;
  background-position: initial !important;
  background-size: initial !important;

  &:hover, &:active, &:focus, &:visited {
    text-decoration: none !important;
    border-bottom: none !important;
  }

  &::after, &::before {
    display: none !important;
    content: none !important;
  }
}


#nprogress {
  pointer-events: none;

  .bar {
    background: var(--theme-color);
    position: fixed;
    z-index: 1031;
    top: 0;
    left: 0;
    width: 100%;
    height: 2px;
    box-shadow: 0 0 10px var(--theme-color), 0 0 5px var(--theme-color);
  }


  .spinner {
    display: block;
    position: fixed;
    z-index: 1031;
    top: 10px;
    left: 10px;

    .spinner-icon {
      width: 18px;
      height: 18px;
      box-sizing: border-box;
      border: solid 2px transparent;
      border-top-color: var(--theme-color);
      border-left-color: var(--theme-color);
      border-radius: 50%;
      animation: nprogress-spinner 400ms linear infinite;
    }
  }
}

@keyframes nprogress-spinner {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}


.nprogress-custom-parent {
  overflow: hidden;
  position: relative;
}

.nprogress-custom-parent #nprogress .bar {
  position: absolute;
}

/* Chatwoot 气泡在移动端上移，通过配置项 chatwootBubbleBottom 控制 */
@media (max-width: 768px) {
  .woot-widget-bubble {
    bottom: var(--chatwoot-bubble-bottom, 20px) !important;
    right: var(--chatwoot-bubble-right, 20px) !important;
  }
  .woot-widget-holder {
    bottom: var(--chatwoot-bubble-bottom, 20px) !important;
  }
}
</style>
