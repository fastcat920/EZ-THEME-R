<template>
  <!-- Chatwoot 将通过 SDK 自动注入到页面 -->
  <div class="chatwoot-embed-container" v-if="CONFIG.enabled">
    <!-- 通过JavaScript初始化Chatwoot，不需要视图内容 -->
  </div>
</template>

<script>
import { onMounted, onUnmounted, computed, watch, ref } from 'vue';
import { useStore } from 'vuex';
import { getUserInfo, getCommConfig, getUserSubscribe } from '@/api/user';
import { CUSTOMER_SERVICE_CONFIG } from '@/utils/baseConfig';
import { formatDate } from '@/utils/formatters';

if (typeof window !== 'undefined') {
  window.CHATWOOT_INITIALIZED = window.CHATWOOT_INITIALIZED || false;
}

export default {
  name: 'ChatwootEmbed',
  
  setup() {
    const store = useStore();
    const userInfo = ref(null);
    const userSubscribe = ref(null);
    const currencySymbol = ref('¥');
    const chatwootInitialized = ref(window.CHATWOOT_INITIALIZED || false);
    const isMobile = ref(false);
    
    const CONFIG = computed(() => {
      return {
        enabled: CUSTOMER_SERVICE_CONFIG.enabled && CUSTOMER_SERVICE_CONFIG.type === 'chatwoot' && CUSTOMER_SERVICE_CONFIG.embedMode === 'embed',
        iconPosition: CUSTOMER_SERVICE_CONFIG.iconPosition || {
          desktop: { left: '20px', bottom: '20px' },
          mobile: { right: '20px', bottom: '100px' }
        }
      };
    });

    /**
     * 从配置中提取 Chatwoot 的参数
     * 支持格式: CHATWOOT_TOKEN="xxx" CHATWOOT_BASE_URL="https://app.chatwoot.com"
     */
    const extractChatwootConfig = () => {
      const customHtml = CUSTOMER_SERVICE_CONFIG.customHtml || '';
      
      const tokenMatch = customHtml.match(/CHATWOOT_TOKEN="([^"]*)"/);
      const baseUrlMatch = customHtml.match(/CHATWOOT_BASE_URL="([^"]*)"/);
      
      return {
        token: tokenMatch ? tokenMatch[1] : '',
        baseUrl: baseUrlMatch ? baseUrlMatch[1] : 'https://app.chatwoot.com'
      };
    };
    
    const initChatwoot = async () => {
      if (!CONFIG.value.enabled) return;
      
      if (window.CHATWOOT_INITIALIZED && chatwootInitialized.value) {
        try {
          updateChatwootUser();
          return;
        } catch (error) {
          console.error('更新Chatwoot配置失败:', error);
        }
      }
      
      try {
        const { token, baseUrl } = extractChatwootConfig();
        
        if (!token) {
          console.error('无法从配置中提取Chatwoot Token');
          return;
        }

        // 设置 Chatwoot 全局配置（含暗色模式）
        const storedTheme = localStorage.getItem('theme');
        window.chatwootSettings = {
          position: isMobile.value ? 'right' : 'left',
          type: 'standard',
          launcherTitle: '在线客服',
          darkMode: storedTheme === 'dark' ? 'dark' : 'light'
        };

        // 加载 Chatwoot SDK
        await loadChatwootScript(baseUrl, token);
        
        // 等待 SDK 加载完成
        window.addEventListener('chatwoot:ready', () => {
          window.CHATWOOT_INITIALIZED = true;
          chatwootInitialized.value = true;
          
          // 设置暗色主题
          setChatwootTheme();
          
          // 设置移动端样式
          if (isMobile.value) {
            setChatwootMobileStyles();
          }
          
          // 设置用户数据
          if (store.getters.isLoggedIn) {
            fetchUserData().then(() => {
              setUserDataToChatwoot();
            });
          }
        });
        
      } catch (error) {
        console.error('初始化Chatwoot客服系统失败:', error);
      }
    };
    
    const loadChatwootScript = (baseUrl, token) => {
      return new Promise((resolve, reject) => {
        // 避免重复加载
        if (document.getElementById('chatwoot-sdk')) {
          resolve();
          return;
        }
        
        const script = document.createElement('script');
        script.id = 'chatwoot-sdk';
        script.src = `${baseUrl}/packs/js/sdk.js`;
        script.async = true;
        script.defer = true;
        
        script.onload = () => {
          window.chatwootSDK.run({
            websiteToken: token,
            baseUrl: baseUrl
          });
          resolve();
        };
        
        script.onerror = () => {
          reject(new Error('Chatwoot SDK 加载失败'));
        };
        
        document.head.appendChild(script);
      });
    };
    
    const setChatwootTheme = () => {
      const storedTheme = localStorage.getItem('theme');
      const isDark = storedTheme === 'dark';
      
      // SDK 正确的 API 是 setColorScheme（支持 'light' / 'auto' / 'dark'）
      if (window.$chatwoot && typeof window.$chatwoot.setColorScheme === 'function') {
        window.$chatwoot.setColorScheme(isDark ? 'dark' : 'light');
      }
    };
    
    const setChatwootMobileStyles = () => {
      const BOTTOM = '140px';
      
      const applyBottom = () => {
        // SDK 源码中气泡容器 class 为 woot--bubble-holder
        const bubbles = document.getElementsByClassName('woot--bubble-holder');
        for (let i = 0; i < bubbles.length; i++) {
          bubbles[i].style.setProperty('bottom', BOTTOM, 'important');
        }
        // 聊天窗口容器 id 为 cw-widget-holder
        const widget = document.getElementById('cw-widget-holder');
        if (widget) {
          widget.style.setProperty('bottom', BOTTOM, 'important');
        }
      };
      
      // SDK 的 onLoad 回调中才创建气泡，需要等元素出现
      const observer = new MutationObserver((mutations) => {
        for (const mutation of mutations) {
          // 监听新节点插入（气泡创建）和样式变化（SDK 重置）
          if (mutation.type === 'childList' || mutation.type === 'attributes') {
            applyBottom();
          }
        }
      });
      
      observer.observe(document.body, {
        childList: true,
        subtree: true,
        attributes: true,
        attributeFilter: ['style', 'class']
      });
      
      // 延迟也尝试一下，双保险
      setTimeout(applyBottom, 500);
      setTimeout(applyBottom, 2000);
      setTimeout(applyBottom, 5000);
      
      onUnmounted(() => observer.disconnect());
    };

    const fetchUserData = async () => {
      try {
        const [userInfoResponse, commConfigResponse, subscribeResponse] = await Promise.all([
          getUserInfo(),
          getCommConfig(),
          getUserSubscribe()
        ]);
        
        userInfo.value = userInfoResponse.data ? userInfoResponse.data : userInfoResponse;
        
        const commConfigData = commConfigResponse.data ? commConfigResponse.data : commConfigResponse;
        if (commConfigData && commConfigData.currency_symbol) {
          currencySymbol.value = commConfigData.currency_symbol;
        }
        
        userSubscribe.value = subscribeResponse.data ? subscribeResponse.data : subscribeResponse;
      } catch (error) {
        console.error('获取用户信息失败:', error);
      }
    };
    
    const updateChatwootUser = () => {
      if (store.getters.isLoggedIn) {
        setUserDataToChatwoot();
      }
    };
    
    const setUserDataToChatwoot = () => {
      if (!window.$chatwoot) return;
      
      try {
        const userEmail = extractUserEmail();
        
        if (userEmail) {
          window.$chatwoot.setUser(userEmail, {
            email: userEmail,
            name: userEmail.split('@')[0]
          });
        }
        
        // 从配置中读取属性 key 映射
        const attrKeys = CUSTOMER_SERVICE_CONFIG.chatwootAttributes || {};
        
        // 提取所有用户数据
        const planName = extractPlanName();
        const expireDate = extractExpireDate();
        const remainingGB = calculateRemainingTraffic();
        const balance = extractBalance();
        const trafficDetails = calculateTrafficDetails();
        const userMeta = extractUserMeta();
        
        // 构建自定义属性（仅推送配置中值不为空的属性）
        const customAttrs = {};
        
        if (attrKeys.plan) customAttrs[attrKeys.plan] = planName;
        if (attrKeys.expires) customAttrs[attrKeys.expires] = expireDate;
        if (attrKeys.traffic) customAttrs[attrKeys.traffic] = remainingGB + ' GB';
        if (attrKeys.balance) customAttrs[attrKeys.balance] = balance + ' ' + currencySymbol.value;
        if (attrKeys.uuid) customAttrs[attrKeys.uuid] = userMeta.uuid || '未知';
        if (attrKeys.created_at) customAttrs[attrKeys.created_at] = userMeta.created_at || '未知';
        if (attrKeys.used_traffic) customAttrs[attrKeys.used_traffic] = trafficDetails.usedGB + ' GB';
        if (attrKeys.total_traffic) customAttrs[attrKeys.total_traffic] = trafficDetails.totalGB + ' GB';
        if (attrKeys.upload) customAttrs[attrKeys.upload] = trafficDetails.uploadGB + ' GB';
        if (attrKeys.download) customAttrs[attrKeys.download] = trafficDetails.downloadGB + ' GB';
        if (attrKeys.telegram_id) customAttrs[attrKeys.telegram_id] = userMeta.telegram_id || '未绑定';
        if (attrKeys.invite_code) customAttrs[attrKeys.invite_code] = userMeta.invite_code || '无';
        if (attrKeys.commission_balance) customAttrs[attrKeys.commission_balance] = (userMeta.commission_balance || 0) + ' ' + currencySymbol.value;
        
        if (Object.keys(customAttrs).length > 0) {
          window.$chatwoot.setCustomAttributes(customAttrs);
        }
      } catch (error) {
        console.error('设置Chatwoot用户数据失败:', error);
      }
    };
    
    const extractUserEmail = () => {
      let userEmail = '';
      
      if (userInfo.value) {
        if (typeof userInfo.value === 'object') {
          if (userInfo.value.email) {
            userEmail = userInfo.value.email;
          } else if (userInfo.value.data && userInfo.value.data.email) {
            userEmail = userInfo.value.data.email;
          }
        }
      }
      
      if (!userEmail && userSubscribe.value) {
        if (typeof userSubscribe.value === 'object') {
          if (userSubscribe.value.email) {
            userEmail = userSubscribe.value.email;
          } else if (userSubscribe.value.data && userSubscribe.value.data.email) {
            userEmail = userSubscribe.value.data.email;
          }
        }
      }
      
      if (!userEmail) {
        const storedUser = localStorage.getItem('user');
        if (storedUser) {
          try {
            const parsedUser = JSON.parse(storedUser);
            if (parsedUser && parsedUser.email) {
              userEmail = parsedUser.email;
            }
          } catch (e) {
            console.error('解析localStorage用户数据失败:', e);
          }
        }
      }
      
      return userEmail;
    };
    
    const extractPlanName = () => {
      let planName = "未知套餐";
      
      if (userSubscribe.value && userSubscribe.value.plan && userSubscribe.value.plan.name) {
        planName = userSubscribe.value.plan.name;
      } else if (userSubscribe.value && userSubscribe.value.data && userSubscribe.value.data.plan && userSubscribe.value.data.plan.name) {
        planName = userSubscribe.value.data.plan.name;
      } else if (userInfo.value && userInfo.value.plan_name && userInfo.value.plan_name.trim() !== '') {
        planName = userInfo.value.plan_name;
      } else if (userInfo.value && userInfo.value.group && userInfo.value.group.name && userInfo.value.group.name.trim() !== '') {
        planName = userInfo.value.group.name;
      }
      
      return planName;
    };
    
    const extractExpireDate = () => {
      let expireDate = "无限期";
      
      if (userSubscribe.value && userSubscribe.value.expired_at) {
        expireDate = formatDate(userSubscribe.value.expired_at);
      } else if (userSubscribe.value && userSubscribe.value.data && userSubscribe.value.data.expired_at) {
        expireDate = formatDate(userSubscribe.value.data.expired_at);
      } else if (userInfo.value && userInfo.value.expired_at) {
        expireDate = formatDate(userInfo.value.expired_at);
      } else if (userInfo.value && userInfo.value.data && userInfo.value.data.expired_at) {
        expireDate = formatDate(userInfo.value.data.expired_at);
      }
      
      return expireDate;
    };
    
    const calculateRemainingTraffic = () => {
      let transferEnable = 0;
      let u = 0;
      let d = 0;
      
      if (userSubscribe.value) {
        if (typeof userSubscribe.value === 'object') {
          if (userSubscribe.value.transfer_enable !== undefined) {
            transferEnable = userSubscribe.value.transfer_enable || 0;
            u = userSubscribe.value.u || 0;
            d = userSubscribe.value.d || 0;
          } else if (userSubscribe.value.data && userSubscribe.value.data.transfer_enable !== undefined) {
            transferEnable = userSubscribe.value.data.transfer_enable || 0;
            u = userSubscribe.value.data.u || 0;
            d = userSubscribe.value.data.d || 0;
          }
        }
      } 
      
      if (transferEnable === 0 && userInfo.value) {
        if (typeof userInfo.value === 'object') {
          if (userInfo.value.transfer_enable !== undefined) {
            transferEnable = userInfo.value.transfer_enable || 0;
            u = userInfo.value.u || 0;
            d = userInfo.value.d || 0;
          } else if (userInfo.value.data && userInfo.value.data.transfer_enable !== undefined) {
            transferEnable = userInfo.value.data.transfer_enable || 0;
            u = userInfo.value.data.u || 0;
            d = userInfo.value.data.d || 0;
          }
        }
      }
      
      const remainingBytes = transferEnable - (u + d);
      return (remainingBytes / (1024 * 1024 * 1024)).toFixed(2);
    };
    
    const extractBalance = () => {
      let balance = 0;
      
      if (userInfo.value) {
        if (typeof userInfo.value === 'object') {
          if (userInfo.value.balance !== undefined) {
            balance = userInfo.value.balance || 0;
          } else if (userInfo.value.data && userInfo.value.data.balance !== undefined) {
            balance = userInfo.value.data.balance || 0;
          }
        }
      }
      
      return balance;
    };
    
    const calculateTrafficDetails = () => {
      let transferEnable = 0;
      let u = 0;
      let d = 0;
      
      // 优先从 subscribe 数据获取
      if (userSubscribe.value) {
        if (typeof userSubscribe.value === 'object') {
          if (userSubscribe.value.transfer_enable !== undefined) {
            transferEnable = userSubscribe.value.transfer_enable || 0;
            u = userSubscribe.value.u || 0;
            d = userSubscribe.value.d || 0;
          } else if (userSubscribe.value.data && userSubscribe.value.data.transfer_enable !== undefined) {
            transferEnable = userSubscribe.value.data.transfer_enable || 0;
            u = userSubscribe.value.data.u || 0;
            d = userSubscribe.value.data.d || 0;
          }
        }
      }
      
      if (transferEnable === 0 && userInfo.value) {
        if (typeof userInfo.value === 'object') {
          if (userInfo.value.transfer_enable !== undefined) {
            transferEnable = userInfo.value.transfer_enable || 0;
            u = userInfo.value.u || 0;
            d = userInfo.value.d || 0;
          } else if (userInfo.value.data && userInfo.value.data.transfer_enable !== undefined) {
            transferEnable = userInfo.value.data.transfer_enable || 0;
            u = userInfo.value.data.u || 0;
            d = userInfo.value.data.d || 0;
          }
        }
      }
      
      const toGB = (bytes) => (bytes / (1024 * 1024 * 1024)).toFixed(2);
      
      return {
        totalGB: toGB(transferEnable),
        usedGB: toGB(u + d),
        uploadGB: toGB(u),
        downloadGB: toGB(d)
      };
    };
    
    const extractUserMeta = () => {
      const meta = {
        uuid: '',
        created_at: '',
        telegram_id: '',
        invite_code: '',
        commission_balance: 0
      };
      
      const source = userInfo.value;
      if (!source || typeof source !== 'object') return meta;
      
      // 兼容 data 嵌套和扁平两种结构
      const data = source.data || source;
      
      if (data.uuid) meta.uuid = data.uuid;
      if (data.created_at) meta.created_at = formatDate(data.created_at);
      if (data.telegram_id) meta.telegram_id = String(data.telegram_id);
      if (data.invite_user_id) meta.invite_code = String(data.invite_user_id);
      if (data.commission_balance !== undefined) meta.commission_balance = data.commission_balance || 0;
      
      return meta;
    };
    
    const checkIfMobile = () => {
      isMobile.value = window.innerWidth < 768;
    };
    
    const handleResize = () => {
      checkIfMobile();
      
      if (chatwootInitialized.value && window.$chatwoot) {
        setChatwootMobileStyles();
      }
    };
    
    watch(() => store.getters.isLoggedIn, async (newVal) => {
      if (newVal && chatwootInitialized.value) {
        await fetchUserData();
        setUserDataToChatwoot();
      }
    });
    
    // 监听主题切换（body 的 dark-theme class 变化），实时同步 Chatwoot 配色
    let themeObserver = null;
    
    onMounted(async () => {
      checkIfMobile();
      await initChatwoot();
      window.addEventListener('resize', handleResize);
      
      // 用 MutationObserver 监听 body 的 class 变化（useTheme 通过 classList 切换主题）
      themeObserver = new MutationObserver(() => {
        if (chatwootInitialized.value) {
          setChatwootTheme();
        }
      });
      themeObserver.observe(document.body, {
        attributes: true,
        attributeFilter: ['class']
      });
    });
    
    onUnmounted(() => {
      window.removeEventListener('resize', handleResize);
      if (themeObserver) themeObserver.disconnect();
    });
    
    return {
      CONFIG
    };
  }
};
</script>

<style lang="scss" scoped>
.chatwoot-embed-container {
  
}
</style>
