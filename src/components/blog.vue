<template>
  <div class="blog-container">
    <!-- 回到顶部按钮 - 放在最外层确保固定在视口右下角 -->
    <v-btn
      fab
      color="var(--leleo-vcard-color)"
      :class="['back-to-top-btn', { 'show': showBackToTop }]"
      @click="backToTop"
      icon
    >
      <v-icon>mdi-arrow-up</v-icon>
    </v-btn>
    
    <v-card class="pa-3 blog-main-card">
      
      <transition name="blog-fade" mode="out-in">
        <v-container v-if="!selectedPost" key="list" class="blog-list-container">
          <div v-if="selectedTag" class="filter-info mb-2">
            <v-chip color="var(--leleo-vcard-color)" closable @click:close="$emit('clearTag')">
              标签: {{ selectedTag }}
            </v-chip>
          </div>
          <v-card
            v-for="post in filteredPosts"
            :key="post.id"
            class="ma-2 cursor-pointer blog-card"
            hover
            @click="selectPost(post)"
          >
            <v-card-title style="color: var(--leleo-vcard-color); font-weight: 600;">{{ post.title }}</v-card-title>
            <v-card-subtitle style="color: rgba(255, 255, 255, 0.7);">
              {{ post.date }} · {{ post.author }}
            </v-card-subtitle>
            <v-card-text style="color: rgba(255, 255, 255, 0.85);">{{ post.excerpt }}</v-card-text>
            <v-card-actions v-if="post.tags && post.tags.length > 0">
              <v-chip
                v-for="tag in post.tags"
                :key="tag"
                size="small"
                class="ma-1"
                color="var(--leleo-vcard-color)"
                variant="outlined"
              >
                {{ tag }}
              </v-chip>
            </v-card-actions>
          </v-card>
        </v-container>
        
        <v-container v-else key="content">
          <div class="d-flex align-center mb-4">
            <v-btn
              variant="tonal"
              color="var(--leleo-vcard-color)"
              class="back-btn"
              @click="backToList"
            >
              <v-icon left>mdi-arrow-left</v-icon>
              返回博客列表
            </v-btn>
            <h1 class="blog-header-title ml-4">{{ selectedPost.title }}</h1>
          </div>
          
          <div class="blog-content-container">
            <div class="blog-content" v-html="selectedPost.content"></div>
          </div>
        </v-container>
      </transition>
    </v-card>
  </div>
</template>

<script>
export default {
  props: ['configdata', 'selectedTag'],
  data() {
    return {
      posts: [],
      selectedPost: null,
      marked: null,
      showBackToTop: false,
      scrollHandler: null,
      scrollHandlers: []
    };
  },
  computed: {
    blurValue() {
      return this.configdata?.blur || 5;
    },
    filteredPosts() {
      if (!this.selectedTag) return this.posts;
      return this.posts.filter(post => post.tags && post.tags.includes(this.selectedTag));
    },
    allTags() {
      const tags = new Set();
      this.posts.forEach(post => {
        if (post.tags) {
          post.tags.forEach(tag => tags.add(tag));
        }
      });
      return Array.from(tags);
    }
  },
  mounted() {
    // 确保 marked 库加载完成后再加载文章
    if (window.marked) {
      this.loadMarked();
      this.loadPosts();
    } else {
      // 如果 marked 库还没有加载，等待一下再尝试
      setTimeout(() => {
        this.loadMarked();
        this.loadPosts();
      }, 100);
    }
    // 添加代码复制功能
    this.$nextTick(() => {
      this.addCopyButtons();
      // 添加滚动事件监听器到内容容器
      this.setupScrollListeners();
    });
  },
  updated() {
    // 在 DOM 更新后添加复制按钮
    this.$nextTick(() => {
      this.addCopyButtons();
      // 重新设置滚动事件监听器
      this.setupScrollListeners();
    });
  },
  beforeUnmount() {
    // 移除滚动事件监听器
    this.removeScrollListeners();
  },
  methods: {
    loadMarked() {
      // 直接使用 window.marked
      this.marked = window.marked;
    },
    async loadPosts() {
      try {
        // 尝试加载索引文件
        let fileList = [];
        try {
          const indexResponse = await fetch('/post/index.json');
          if (indexResponse.ok) {
            fileList = await indexResponse.json();
          }
        } catch (e) {
          // 索引文件失败
        }
        
        const posts = [];
        
        for (const filename of fileList) {
          const response = await fetch(`/post/${filename}`);
          if (response.ok) {
            const content = await response.text();
            
            // 解析 Front Matter 和内容
            const { frontMatter, body } = this.parseFrontMatter(content);
            
            // 提取标题（优先使用 Front Matter 中的 title）
            let title = frontMatter.title;
            if (!title) {
              title = this.extractTitle(body);
            }
            
            // 提取日期（优先使用 Front Matter 中的 date）
            let date = frontMatter.date;
            if (!date) {
              // 尝试从文件名中提取日期
              const dateMatch = filename.match(/^(\d{4}-\d{2}-\d{2})/);
              date = dateMatch ? dateMatch[1] : new Date().toLocaleDateString();
            }
            
            // 提取摘要
            const excerpt = this.extractExcerpt(body);
            
            // 跳过无标题的文章
            if (title && title !== '无标题') {
              // 使用 marked 解析 markdown
              let parsedContent = body;
              if (window.marked && window.marked.parse) {
                parsedContent = window.marked.parse(body);
              }
              
              posts.push({
                id: filename.replace(/\.(md|markdown)$/i, ''),
                title: title,
                content: parsedContent,
                excerpt: excerpt,
                date: date,
                author: frontMatter.author || 'ZZJ',
                tags: frontMatter.tags || []
              });
            }
          }
        }
        
        // 按日期排序（最新的在前）
        posts.sort((a, b) => new Date(b.date) - new Date(a.date));
        
        // 如果没有找到任何文章，显示一个提示
        if (posts.length === 0) {
          posts.push({
            id: 'no-posts',
            title: '暂无博客文章',
            content: '<p>请在 public/post 目录下添加 markdown 文件来创建博客文章。</p>',
            excerpt: '暂无博客文章',
            date: new Date().toLocaleDateString()
          });
        }
        
        this.posts = posts;
        
        // 更新标签列表（在 posts 赋值后）
        this.$nextTick(() => {
          this.$emit('updateTags', this.allTags);
        });
      } catch (error) {
        console.error('加载博客文章失败:', error);
        
        // 显示错误信息
        this.posts = [{
          id: 'error',
          title: '加载失败',
          content: `<p>加载博客文章时出错: ${error.message}</p>`,
          excerpt: '加载失败',
          date: new Date().toLocaleDateString()
        }];
      }
    },
    parseFrontMatter(content) {
      const frontMatter = {};
      let body = content;
      
      // 检查是否有 Front Matter
      const match = content.match(/^---\s*\n([\s\S]*?)\n---\s*\n([\s\S]*)$/);
      if (match) {
        const fmText = match[1];
        body = match[2];
        
        // 解析 YAML 格式的 Front Matter
        const lines = fmText.split('\n');
        let currentKey = '';
        let currentValue = '';
        let inArray = false;
        
        for (const line of lines) {
          const trimmed = line.trim();
          
          // 检查是否是数组项
          if (trimmed.startsWith('- ')) {
            if (!inArray) {
              inArray = true;
              frontMatter[currentKey] = [];
            }
            frontMatter[currentKey].push(trimmed.substring(2).trim());
          } else {
            inArray = false;
            const colonIndex = trimmed.indexOf(':');
            if (colonIndex > 0) {
              currentKey = trimmed.substring(0, colonIndex).trim();
              currentValue = trimmed.substring(colonIndex + 1).trim();
              // 去除引号
              if ((currentValue.startsWith('"') && currentValue.endsWith('"')) ||
                  (currentValue.startsWith("'") && currentValue.endsWith("'"))) {
                currentValue = currentValue.slice(1, -1);
              }
              frontMatter[currentKey] = currentValue;
            }
          }
        }
      }
      
      return { frontMatter, body };
    },
    extractTitle(content) {
      const match = content.match(/^#\s+(.+)$/m);
      return match ? match[1] : '无标题';
    },
    extractExcerpt(content) {
      // 移除 markdown 标记，提取纯文本
      let text = content
        .replace(/^---[\s\S]*?---/m, '') // 移除 Front Matter
        .replace(/^#{1,6}\s+.+$/gm, '') // 移除所有标题 (h1-h6)
        .replace(/```[\s\S]*?```/g, '') // 移除代码块
        .replace(/`([^`]+)`/g, '$1') // 移除行内代码标记
        .replace(/\*\*([^*]+)\*\*/g, '$1') // 移除加粗
        .replace(/\*([^*]+)\*/g, '$1') // 移除斜体
        .replace(/__([^_]+)__/g, '$1') // 移除下划线加粗
        .replace(/_([^_]+)_/g, '$1') // 移除下划线斜体
        .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1') // 移除链接，保留文本
        .replace(/!\[([^\]]*)\]\([^)]+\)/g, '') // 移除图片
        .replace(/>\s?/g, '') // 移除引用标记
        .replace(/\n/g, ' ') // 换行转空格
        .replace(/\s+/g, ' ') // 多个空格合并为一个
        .trim();
      
      return text.length > 120 ? text.substring(0, 120) + '...' : text;
    },
    selectPost(post) {
      this.selectedPost = post;
      // 重置回到顶部按钮状态
      this.showBackToTop = false;
      
      // 选择文章后延迟添加复制按钮和滚动监听器，确保 DOM 已更新
      this.$nextTick(() => {
        setTimeout(() => {
          this.addCopyButtons();
          this.setupContentScrollListener();
        }, 100);
      });
    },
    addCopyButtons() {
      // 获取博客内容容器中的所有 pre 元素（代码块）
      const container = this.$el.querySelector('.blog-content');
      if (!container) return;
      
      const codeBlocks = container.querySelectorAll('pre');
      codeBlocks.forEach(pre => {
        // 检查是否已经添加了复制按钮
        if (pre.querySelector('.copy-btn')) return;
        
        // 确保 pre 元素有相对定位
        pre.style.position = 'relative';
        
        // 创建复制按钮
        const copyBtn = document.createElement('button');
        copyBtn.className = 'copy-btn';
        copyBtn.innerHTML = '复制';
        copyBtn.title = '复制代码';
        
        // 添加点击事件
        copyBtn.addEventListener('click', (e) => {
          e.stopPropagation();
          e.preventDefault();
          const code = pre.querySelector('code') || pre;
          const text = code.innerText || code.textContent;
          
          // 复制到剪贴板
          navigator.clipboard.writeText(text).then(() => {
            copyBtn.innerHTML = '已复制';
            copyBtn.classList.add('copied');
            setTimeout(() => {
              copyBtn.innerHTML = '复制';
              copyBtn.classList.remove('copied');
            }, 2000);
          }).catch(err => {
            console.error('复制失败:', err);
            // 降级方案
            const textarea = document.createElement('textarea');
            textarea.value = text;
            document.body.appendChild(textarea);
            textarea.select();
            document.execCommand('copy');
            document.body.removeChild(textarea);
            copyBtn.innerHTML = '已复制';
            copyBtn.classList.add('copied');
            setTimeout(() => {
              copyBtn.innerHTML = '复制';
              copyBtn.classList.remove('copied');
            }, 2000);
          });
        });
        
        // 将按钮添加到 pre 元素中
        pre.appendChild(copyBtn);
      });
    },
    backToTop() {
      // 平滑滚动到顶部
      const container = document.querySelector('.blog-list-container, .blog-content-container');
      if (container) {
        container.scrollTo({
          top: 0,
          behavior: 'smooth'
        });
      } else {
        window.scrollTo({
          top: 0,
          behavior: 'smooth'
        });
      }
    },
    backToList() {
      // 重置回到顶部按钮状态
      this.showBackToTop = false;
      // 返回博客列表
      this.selectedPost = null;
    },
    setupScrollListeners() {
      // 移除旧的监听器
      this.removeScrollListeners();
      if (!this.scrollHandlers) this.scrollHandlers = [];
      
      // 使用一个统一的处理函数
      this._scrollHandler = () => this.handleScroll();
      
      // 监听 window 的滚动事件（捕获阶段）
      window.addEventListener('scroll', this._scrollHandler, true);
      this.scrollHandlers.push({ container: window, handler: this._scrollHandler, useCapture: true });
      
      // 监听博客列表容器
      const listContainer = document.querySelector('.blog-list-container');
      if (listContainer) {
        listContainer.addEventListener('scroll', this._scrollHandler);
        this.scrollHandlers.push({ container: listContainer, handler: this._scrollHandler });
      }
    },
    setupContentScrollListener() {
      // 专门用于监听博文内容容器的滚动
      if (this._contentScrollHandler) return; // 避免重复添加
      
      const contentContainer = document.querySelector('.blog-content-container');
      if (contentContainer) {
        this._contentScrollHandler = () => this.handleScroll();
        contentContainer.addEventListener('scroll', this._contentScrollHandler);
        if (!this.scrollHandlers) this.scrollHandlers = [];
        this.scrollHandlers.push({ container: contentContainer, handler: this._contentScrollHandler });
      }
    },
    removeScrollListeners() {
      if (this.scrollHandlers) {
        this.scrollHandlers.forEach(({ container, handler, useCapture }) => {
          if (container && handler) {
            container.removeEventListener('scroll', handler, useCapture || false);
          }
        });
        this.scrollHandlers = [];
      }
      this._scrollHandler = null;
      this._contentScrollHandler = null;
    },
    handleScroll() {
      // 获取当前显示的容器
      const listContainer = document.querySelector('.blog-list-container');
      const contentContainer = document.querySelector('.blog-content-container');
      
      let currentScrollTop = 0;
      
      // 根据当前显示的视图获取对应的滚动位置
      if (this.selectedPost && contentContainer) {
        // 在博文内容视图中
        currentScrollTop = contentContainer.scrollTop;
      } else if (listContainer) {
        // 在博客列表视图中
        currentScrollTop = listContainer.scrollTop;
      }
      
      // 只有当滚动超过 100px 时才显示按钮
      const shouldShow = currentScrollTop > 100;
      
      // 避免不必要的更新
      if (this.showBackToTop !== shouldShow) {
        this.showBackToTop = shouldShow;
      }
    }
  }
};
</script>

<style>
/* 引入字魂白鸽天行体 */
@font-face {
  font-family: '字魂白鸽天行体';
  src: url('/fonts/字魂白鸽天行体.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}

.blog-container {
  width: 100%;
  min-height: 100%;
}

/* 回到顶部按钮样式 */
.back-to-top-btn {
  position: fixed !important;
  bottom: 20px !important;
  right: 20px !important;
  z-index: 9999 !important;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3) !important;
  transition: all 0.3s ease !important;
  opacity: 0;
  transform: translateY(100px) scale(0.8);
  visibility: hidden;
  background-color: rgba(255, 255, 255, 0.6) !important;
  backdrop-filter: blur(var(--leleo-blur)) !important;
  border: 1px solid rgba(255, 255, 255, 0.7) !important;
  color: var(--leleo-vcard-color) !important;
}

.back-to-top-btn.show {
  opacity: 1;
  transform: translateY(0) scale(1);
  visibility: visible;
}

.back-to-top-btn:hover {
  transform: translateY(-5px) scale(1.05);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
  background-color: rgba(255, 255, 255, 0.3) !important;
}

/* 博客主卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-main-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.15) !important;
  border-radius: 12px !important;
  border: 1px solid rgba(255, 255, 255, 0.3) !important;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1) !important;
}

/* 博客主卡片滚动条 */
.blog-main-card::-webkit-scrollbar {
  width: 8px;
}

.blog-main-card::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-main-card::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-main-card::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

/* 博客列表容器 */
.blog-list-container {
  max-height: calc(100vh - 90px);
  overflow-y: auto;
  padding-right: 8px;
}

/* 博客列表滚动条 */
.blog-list-container::-webkit-scrollbar {
  width: 6px;
}

.blog-list-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.4);
}

/* 博客列表滚动条 */
.blog-list-container::-webkit-scrollbar {
  width: 6px;
}

.blog-list-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.blog-list-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.4);
}

.cursor-pointer {
  cursor: pointer;
}

/* 博客列表和内容切换动画 */
.blog-fade-enter-active,
.blog-fade-leave-active {
  transition: all 0.5s cubic-bezier(0.25, 0.8, 0.25, 1);
}

.blog-fade-enter-from {
  opacity: 0;
  transform: translateX(30px);
}

.blog-fade-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}

/* 博客卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.2) !important;
  border-radius: 8px !important;
  border: 1px solid rgba(255, 255, 255, 0.2) !important;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1) !important;
  transition: all 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

.blog-card:hover {
  transform: translateY(-3px) scale(1.01);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15) !important;
  background-color: rgba(255, 255, 255, 0.28) !important;
  border-color: rgba(255, 255, 255, 0.35) !important;
}

/* 返回按钮动画 */
.back-btn {
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
}

.back-btn:hover {
  transform: translateX(-4px);
}

/* 博文头部标题样式 */
.blog-header-title {
  font-family: '字魂白鸽天行体', sans-serif;
  font-size: 2.5rem;
  font-weight: normal;
  color: var(--leleo-vcard-color);
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
  letter-spacing: 1px;
  margin: 0;
  margin-top: -8px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 博客内容容器样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-content-container {
  backdrop-filter: blur(var(--leleo-blur));
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  padding: 2rem;
  max-height: calc(90vh - 98px);
  overflow-y: auto;
}

/* 博客内容滚动条 */
.blog-content-container::-webkit-scrollbar {
  width: 8px;
}

.blog-content-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

/* 自定义滚动条样式 */
.blog-content-container::-webkit-scrollbar {
  width: 8px;
}

.blog-content-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

.blog-content-container::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

.blog-content {
  line-height: 1.6;
  color: rgba(255, 255, 255, 0.9);
}

/* 博客内容大标题 */
.blog-content-title {
  font-size: 4rem;
  font-weight: normal;
  margin: 0 0 2rem 0;
  color: var(--leleo-vcard-color);
  font-family: '字魂白鸽天行体', sans-serif;
  text-align: center;
  padding-bottom: 1rem;
  border-bottom: 2px solid rgba(255, 255, 255, 0.2);
}

.blog-content h1 {
  font-size: 2rem;
  margin: 1.5rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content h2 {
  font-size: 1.5rem;
  margin: 1.2rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content h3 {
  font-size: 1.2rem;
  margin: 1rem 0;
  color: var(--leleo-vcard-color);
}

.blog-content p {
  margin: 1rem 0;
}

.blog-content ul, .blog-content ol {
  margin: 1rem 0 1rem 2rem;
}

.blog-content li {
  margin: 0.5rem 0;
}

/* 表格样式 */
.blog-content table {
  width: 100%;
  border-collapse: collapse;
  margin: 1.5rem 0;
  background-color: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  overflow: hidden;
}

.blog-content th,
.blog-content td {
  padding: 12px 16px;
  text-align: left;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.blog-content th {
  background-color: rgba(255, 255, 255, 0.1);
  font-weight: 600;
  color: var(--leleo-vcard-color);
}

.blog-content tr:hover {
  background-color: rgba(255, 255, 255, 0.05);
}

.blog-content tr:last-child td {
  border-bottom: none;
}

.blog-content code {
  background-color: #f5f5f5;
  padding: 0.2rem 0.4rem;
  border-radius: 4px;
  font-family: monospace;
}

/* 代码框容器 */
.blog-content pre {
  background-color: #1e1e1e;
  border-radius: 10px;
  overflow: hidden;
  margin: 1.5rem 0;
  position: relative;
  border: 1px solid rgba(255, 255, 255, 0.08);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
}

/* 代码框标题栏 */
.blog-content pre::before {
  content: '';
  display: block;
  height: 32px;
  background: linear-gradient(180deg, #2d2d2d 0%, #252525 100%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

/* 代码框标题栏圆点 */
.blog-content pre::after {
  content: '';
  position: absolute;
  top: 10px;
  left: 12px;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: linear-gradient(135deg, #ff5f56, #ff4438);
  box-shadow: 20px 0 0 #ffbd2e, 40px 0 0 #27ca40;
}

/* 代码内容区域 */
.blog-content pre code {
  display: block;
  background-color: transparent;
  padding: 16px 20px;
  color: #d4d4d4;
  font-family: 'Consolas', 'Monaco', 'Courier New', 'Fira Code', monospace;
  font-size: 14px;
  line-height: 1.6;
  overflow-x: auto;
}

/* 行内代码 */
.blog-content code {
  background-color: rgba(255, 255, 255, 0.1);
  color: #e6e6e6;
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'Consolas', 'Monaco', 'Courier New', 'Fira Code', monospace;
  font-size: 0.9em;
}

/* 代码复制按钮样式 */
.copy-btn {
  position: absolute;
  top: 1px;
  right: 10px;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 5px;
  padding: 4px 10px;
  cursor: pointer;
  color: #a0a0a0;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.copy-btn:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.2);
  color: #d0d0d0;
}

.copy-btn:active {
  background: rgba(255, 255, 255, 0.1);
}

.copy-btn.copied {
  background: rgba(76, 175, 80, 0.2);
  border-color: rgba(76, 175, 80, 0.3);
  color: #81c784;
}

.blog-content blockquote {
  border-left: 4px solid var(--leleo-vcard-color);
  padding-left: 1rem;
  margin: 1rem 0;
  font-style: italic;
}

.blog-content a {
  color: var(--leleo-vcard-color);
  text-decoration: none;
}

.blog-content a:hover {
  text-decoration: underline;
}
</style>