<template>
  <v-app class="vapp-fullscreen-background" style="overflow: hidden;" :class="{ 'radius-before': !xs }"
  :style="xs?{height: '100%',width: '100%',top: '0',left:'0'}:(sm?{height: '98%',width: '98%',top: '1%',left:' 1%'}:{height: '96.6%',width: '99%',top: '1.7%',left:' 0.5%'})">
    <transition name="fade">
      <div class="loading" v-show="isloading">
        <loader></loader>
      </div>
    </transition>

    <video autoplay loop muted class="video-bg" id="bg-video" ref="VdPlayer"
    :style="xs?{height: '100%',width: '100%',top: '0',left:'0'}:(sm?{height: '98%',width: '98%',top: '1%',left:' 1%','border-radius': '16px'}:{height: '96.6%',width: '99%',top: '1.7%',left:' 0.5%','border-radius': '16px',})">
        <source :src=videosrc type="video/mp4">
    </video>

    <div class="zzj-nav-container">
      <v-btn
        variant="tonal"
        color="var(--leleo-vcard-color)"
        class="zzj-nav-btn mr-3"
        :href="'https://zzjjack.us.kg'"
        target="_blank"
        icon
      >
        <v-icon>mdi-home</v-icon>
      </v-btn>
      <v-btn
        variant="tonal"
        color="var(--leleo-vcard-color)"
        class="zzj-nav-btn"
        :href="'https://sites.zzjjack.us.kg'"
        target="_blank"
        icon
      >
        <v-icon>mdi-web</v-icon>
      </v-btn>
      <span class="blog-title">{{ configdata.metaData.title }}</span>
    </div>

    <div class="floating-switch-container">
      <v-switch
        v-model="isClearScreen"
        inset
        :style="xs?{'transform':'scale(0.6) translateX(15%)'}:{}"
        class="floating-switch"
        @mouseover="expandSwitch"
        @mouseleave="collapseSwitch"
      ></v-switch>
    </div>
    
    <div v-show="!isloading && !isClearScreen" style="height: 100vh; overflow: hidden;">
        <v-row style="height: 100%; margin: 0;">
            <v-col cols="12" md="4" lg="3" class="leleo-left" align="center" style="height: 100%; overflow-y: auto;">
              <div :style="xs||sm?{'font-size':'2.3rem'}:{'display':'none'}" class="leleo-left-welcome">{{ configdata.welcometitle }}</div>  
              <v-avatar class="leleo-left-avatar avatar-animate" :size="xs||sm?170:200" :style="xs||sm?{'margin-top': '0'}:{'margin-top': '2rem'}">
                  <v-img
                  alt="Leleo"
                  :src=configdata.avatar
                  class="avatar-img"
                  ></v-img>
                </v-avatar>

                <v-card class="mx-5 mb-5 mt-0 pa-2 leleo-left-card tag-card-animate" variant="tonal" :max-width="xs?320:380" style="text-align: center;">
                    <template v-slot:title>
                    <span>博客标签</span>
                    </template>
                    <div v-if="blogTags.length > 0" class="tag-scroll-container">
                        <v-chip 
                            v-for="tag in blogTags" 
                            :key="tag"
                            density="comfortable" 
                            class="ma-1 tag-chip-animate" 
                            size="default"
                            :color="selectedBlogTag === tag ? 'var(--leleo-vcard-color)' : ''"
                            :variant="selectedBlogTag === tag ? 'flat' : 'outlined'"
                            @click="selectBlogTag(tag)"
                            style="cursor: pointer; font-size: 14px;"
                        >
                            {{ tag }}
                        </v-chip>
                    </div>
                    <div v-else class="text-caption text-grey" style="min-height: 100px; display: flex; align-items: center; justify-content: center;">
                        暂无标签
                    </div>
                    <div style="min-height: 36px;">
                        <v-btn
                            v-if="selectedBlogTag"
                            variant="text"
                            size="small"
                            color="var(--leleo-vcard-color)"
                            class="mt-2"
                            @click="clearBlogTag"
                        >
                            <v-icon left size="small">mdi-refresh</v-icon>
                            重置标签
                        </v-btn>
                    </div>
                </v-card>

                <div class="leleo-left-typewriter">
                    <typewriter :configdata="configdata" />
                </div>

                <v-container class="leleo-left-socialIconsContainer">
                    <v-row align="center" justify="center">
                    <v-col class="pa-1" cols="auto" v-for="item in socialPlatformIcons">
                        <v-btn :size="xs?25:33" variant="tonal" color="var(--leleo-vcard-color)"
                        class="ma-1 leleo-social-bticon"
                        icon
                        :href="item.link" target="_blank"
                        >
                    <v-icon :icon=item.icon :size="xs?20:25" class="social-bticon-icon"></v-icon></v-btn>
                    </v-col>
                    </v-row>

                    <v-row align="center" justify="center" class="setting">
                    <v-col class="ma-1" cols="auto">
                        <v-speed-dial
                            :location="xs||sm ?'top center':'right center'"
                            transition="slide-y-transition"
                        >
                        <template v-slot:activator="{ props: activatorProps }">
                            <v-fab style="width: 2.5rem;height: 2.5rem;" color="var(--leleo-vcard-color)"
                            variant="tonal"
                            v-bind="activatorProps"
                            icon="mdi-cog"
                            ></v-fab>
                        </template>
                        <v-btn variant="tonal" class="setbtn" key="1" icon="mdi-key-chain" @click="dialog1 = true" size="31" color="var(--leleo-vcard-color)"></v-btn>
                        <v-btn variant="tonal" class="setbtn" key="2" icon="mdi-information" @click="dialog2 = true" size="31" color="var(--leleo-vcard-color)"></v-btn>
                        <v-btn variant="tonal" class="setbtn" key="3" icon="$error" size="31" color="var(--leleo-vcard-color)"></v-btn>
                        </v-speed-dial>
                    </v-col>
                    </v-row>
                </v-container>
            </v-col>

            <v-col cols="12" md="7" lg="9" style="height: 100%; padding: 1rem; overflow-y: auto;">
                <blog 
                    ref="blogComponent"
                    :configdata="configdata" 
                    :selectedTag="selectedBlogTag"
                    @clearTag="clearBlogTag"
                    @updateTags="updateBlogTags"
                ></blog>
            </v-col>
        </v-row>
    </div>

    <v-dialog
        v-model="dialog1"
        width="1000"
        heihght="700"
      >
      <v-card elevation="3" style="backdrop-filter: blur(10px);">
        <v-tabs
          v-model="tab"
          :items="tabs"
          align-tabs="center"
          height="60"
          slider-color=var(--leleo-vcard-color)
        >
          <template v-slot:tab="{ item }">
            <v-tab
              :prepend-icon="item.icon"
              :text="item.text"
              :value="item.value"
              class="text-none"
            ></v-tab>
          </template>
          
          <template v-slot:item="{ item }">
            <v-tabs-window-item :value="item.value" class="pa-4">
              <!-- 通过组件绑定不同tab项的组件 -->
              <component :is=item.component @cancel="handleCancel"></component>
            </v-tabs-window-item>
          </template>
        </v-tabs>
      </v-card>
      </v-dialog>

      <v-dialog
        v-model="dialog2"
        width="700"
        heihght="500"
      >
      <v-card class="ma-3 pa-2" hover
          variant="tonal"
          rounded="lg"
          style="text-align: center;backdrop-filter: blur(10px);"
        >
          <template v-slot:title >
            <span class="leleo-card-title">关于</span>
          </template>
          <div style="display: flex;flex-direction: column;align-items: center;">
            <v-card class="ma-3 pa-2" hover
              variant="tonal"
              max-width="400"
              rounded="lg"
              style="text-align: center;"
              >
              <template v-slot:subtitle>
                <span class="leleo-card-subtitle">本页基于以下技术及服务搭建</span>
              </template>
              <div>
                <v-tooltip  v-for="item in stackicons" v-model="item.model" location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn icon v-bind="props" :color=item.color rounded="lg" class="ma-1 stack-btn" size="35">
                      <v-icon size="25" color="white">{{item.icon}}</v-icon>
                    </v-btn>
                  </template>
                  <span>{{item.tip}}</span>
                </v-tooltip>
                <!-- 自定义 -->
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35">
                      <v-avatar image="/img/stackicon/vite.svg" rounded="0" size="23"></v-avatar>
                    </v-btn>
                  </template>
                  <span>vite</span>
                </v-tooltip>
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35" color="#254B7C">
                      <span style="font-size: 8px;font-weight: bolder;">{less}</span>
                    </v-btn>
                  </template>
                  <span>less</span>
                </v-tooltip>
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35">
                      <v-avatar image="/img/stackicon/mdi.svg" rounded="0" size="35"></v-avatar>
                    </v-btn>
                  </template>
                  <span>mdi</span>
                </v-tooltip>
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35">
                      <v-avatar image="/img/stackicon/chartjs.png" rounded="0" size="23"></v-avatar>
                    </v-btn>
                  </template>
                  <span>chartjs</span>
                </v-tooltip>
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35" color="#0F1225">
                      <v-avatar image="/img/stackicon/meting.png" rounded="0" size="23"></v-avatar>
                    </v-btn>
                  </template>
                  <span>meting</span>
                </v-tooltip>
                <v-tooltip location="top">
                  <template v-slot:activator="{ props }">
                    <v-btn v-bind="props" rounded="lg" class="ma-1 stack-btn" size="35" color="#070707">
                      <v-avatar image="/img/stackicon/uiverse.png" rounded="0" size="23"></v-avatar>
                    </v-btn>
                  </template>
                  <span>uiverse</span>
                </v-tooltip>
              </div>
            </v-card>

            <p class="ma-6">
                <span v-for="item in configdata.statement">
                  {{ item }}<br>
                </span>
            </p>
          </div>
        </v-card>
    </v-dialog>
  </v-app>
</template>

<script src="./app.js"></script>
<style scoped>
  @import url(/css/app.less);
  @import url(/css/mobile.less);

  .zzj-nav-container {
    position: absolute;
    top: 1rem;
    left: 1rem;
    z-index: 1000;
  }

  .zzj-nav-btn {
    width: 40px;
    height: 40px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
    transition: all 0.3s ease;
  }

  .zzj-nav-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  }

  .blog-title {
    font-family: '字魂白鸽天行体', sans-serif;
    font-size: 2rem;
    font-weight: normal;
    color: var(--leleo-vcard-color);
    text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
    letter-spacing: 2px;
    margin-left: 1rem;
    margin-top: -5px;
    vertical-align: middle;
    display: inline-block;
  }

  .leleo-left {
    padding-top: 3rem !important;
  }

  /* 打字机容器样式 */
  .leleo-left-typewriter {
    margin: 1rem 0;
    padding: 0 1rem;
    text-align: center;
  }

  /* 头像动画样式 */
  .avatar-animate {
    transition: all 0.5s cubic-bezier(0.25, 0.8, 0.25, 1);
    cursor: pointer;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  }

  .avatar-animate:hover {
    transform: scale(1.08) translateY(-5px);
    box-shadow: 0 12px 30px rgba(0, 0, 0, 0.3);
  }

  .avatar-img {
    transition: all 0.5s cubic-bezier(0.25, 0.8, 0.25, 1);
  }

  .avatar-animate:hover .avatar-img {
    transform: scale(1.05);
  }

  /* 博客标签卡片动画 */
  .tag-card-animate {
    transition: all 0.5s cubic-bezier(0.25, 0.8, 0.25, 1);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  .tag-card-animate:hover {
    transform: translateY(-3px);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
    background-color: rgba(255, 255, 255, 0.25) !important;
  }

  /* 标签 chip 动画 */
  .tag-chip-animate {
    transition: all 0.4s cubic-bezier(0.25, 0.8, 0.25, 1);
    font-weight: 500;
    color: rgba(255, 255, 255, 0.9) !important;
  }

  .tag-chip-animate:hover {
    transform: scale(1.15) translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25);
    background-color: rgba(255, 255, 255, 0.3) !important;
    border-color: var(--leleo-vcard-color) !important;
    color: #ffffff !important;
  }

  /* 被选中的标签样式 - 使用深色背景配主题色边框 */
  .tag-chip-animate.v-chip--variant-flat {
    background: rgba(20, 20, 30, 0.95) !important;
    color: #ffffff !important;
    font-weight: 700 !important;
    font-size: 15px !important;
    text-shadow: 0 0 0 transparent, 0 2px 4px rgba(0, 0, 0, 0.8);
    border: 2px solid var(--leleo-vcard-color) !important;
    box-shadow: 0 0 10px var(--leleo-vcard-color), 0 4px 12px rgba(0, 0, 0, 0.5) !important;
    transform: scale(1.08);
  }

  /* 标签滚动容器 */
  .tag-scroll-container {
    max-height: 130px;
    overflow-y: auto;
    overflow-x: hidden;
    padding: 4px;
    margin-bottom: 8px;
  }

  .tag-scroll-container::-webkit-scrollbar {
    width: 4px;
  }

  .tag-scroll-container::-webkit-scrollbar-track {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 2px;
  }

  .tag-scroll-container::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.3);
    border-radius: 2px;
  }

  .tag-scroll-container::-webkit-scrollbar-thumb:hover {
    background: rgba(255, 255, 255, 0.5);
  }
</style>