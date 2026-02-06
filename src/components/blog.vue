<template>
  <div class="blog-container">
    <v-card class="pa-4 blog-main-card">
      
      <v-container v-if="!selectedPost">
        <v-card
          v-for="post in posts"
          :key="post.id"
          class="ma-2 cursor-pointer blog-card"
          hover
          @click="selectPost(post)"
        >
          <v-card-title style="color: #333;">{{ post.title }}</v-card-title>
          <v-card-subtitle style="color: #666;">{{ post.date }}</v-card-subtitle>
          <v-card-text style="color: #444;">{{ post.excerpt }}</v-card-text>
        </v-card>
      </v-container>
      
      <v-container v-else>
        <v-btn
          variant="tonal"
          color="var(--leleo-vcard-color)"
          class="mb-4"
          @click="selectedPost = null"
        >
          <v-icon left>mdi-arrow-left</v-icon>
          返回博客列表
        </v-btn>
        
        <div class="blog-content-container">
          <div class="blog-content" v-html="selectedPost.content"></div>
        </div>
      </v-container>
    </v-card>
  </div>
</template>

<script>
export default {
  props: ['configdata'],
  data() {
    return {
      posts: [],
      selectedPost: null,
      marked: null
    };
  },
  computed: {
    blurValue() {
      return this.configdata?.blur || 5;
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
    });
  },
  updated() {
    // 在 DOM 更新后添加复制按钮
    this.$nextTick(() => {
      this.addCopyButtons();
    });
  },
  methods: {
    loadMarked() {
      if (window.marked) {
        this.marked = window.marked;
        // 检查 marked 是否是一个函数
        if (typeof this.marked === 'function') {
          // 对于较新版本的 marked
          this.marked.setOptions({
            breaks: true,
            gfm: true
          });
        } else if (window.marked.parse) {
          // 对于较新版本的 marked，API 可能有所不同
          this.marked = window.marked.parse;
        }
      }
    },
    async loadPosts() {
      try {
        // 尝试加载 public/post 目录下的 markdown 文件
        // 由于是静态网站，我们使用常见的命名格式来尝试加载
        const commonNames = ['index', 'post1', 'post2', 'article1', 'article2', 'example1', 'example2'];
        const posts = [];
        
        for (const name of commonNames) {
          const file = `${name}.md`;
          const response = await fetch(`/post/${file}`);
          if (response.ok) {
            const content = await response.text();
            const title = this.extractTitle(content);
            const excerpt = this.extractExcerpt(content);
            
            // 检查是否已经添加过相同的文章，并跳过无标题的文章
            const existingPost = posts.find(post => post.id === name);
            if (!existingPost && title !== '无标题') {
              let parsedContent = content;
              if (this.marked) {
                if (typeof this.marked === 'function') {
                  // 对于较新版本的 marked，直接调用 parse 方法
                  parsedContent = this.marked.parse ? this.marked.parse(content) : this.marked(content);
                } else if (this.marked.parse) {
                  // 对于对象形式的 marked，使用 parse 方法
                  parsedContent = this.marked.parse(content);
                }
              }
              
              posts.push({
                id: name,
                title: title,
                content: parsedContent,
                excerpt: excerpt,
                date: new Date().toLocaleDateString()
              });
            }
          }
        }
        
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
    extractTitle(content) {
      const match = content.match(/^#\s+(.+)$/m);
      return match ? match[1] : '无标题';
    },
    extractExcerpt(content) {
      const text = content.replace(/^#\s+.+$/m, '').replace(/\n/g, ' ').trim();
      return text.length > 100 ? text.substring(0, 100) + '...' : text;
    },
    selectPost(post) {
      this.selectedPost = post;
      // 选择文章后延迟添加复制按钮，确保 DOM 已更新
      setTimeout(() => {
        this.addCopyButtons();
      }, 100);
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
    }
  }
};
</script>

<style scoped>
.blog-container {
  width: 100%;
}

/* 博客主卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-main-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.15) !important;
  border-radius: 12px !important;
  border: 1px solid rgba(255, 255, 255, 0.3) !important;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1) !important;
}

.cursor-pointer {
  cursor: pointer;
}

/* 博客卡片样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-card {
  backdrop-filter: blur(var(--leleo-blur)) !important;
  background-color: rgba(255, 255, 255, 0.2) !important;
  border-radius: 8px !important;
  border: 1px solid rgba(255, 255, 255, 0.2) !important;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1) !important;
  transition: all 0.3s ease;
}

.blog-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.15) !important;
}

/* 博客内容容器样式 - 使用 CSS 变量实现毛玻璃效果 */
.blog-content-container {
  backdrop-filter: blur(var(--leleo-blur));
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  padding: 2rem;
}

.blog-content {
  line-height: 1.6;
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
  top: 5px;
  right: 10px;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 5px;
  padding: 4px 10px;
  cursor: pointer;
  color: #a0a0a0;
  font-size: 11px;
  font-weight: 500;
  transition: all 0.2s ease;
  display: flex;
  align-items