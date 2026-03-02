---
title: 碎碎念
layout: page
---

<div id="qexo_talks"></div>
<script src="https://cdn.jsdelivr.net/npm/qexo-static/hexo/talks.js"></script>

<script>
  (function initQexo() {
    // 尝试不同的初始化函数名
    if (typeof renderTalks === 'function') {
      renderTalks({
        container: '#qexo_talks',
        url: 'https://qexo.shview.top',
        limit: 10
      });
      console.log("Qexo 说说已通过 renderTalks 加载");
    } else if (typeof QexoTalks === 'function') {
      new QexoTalks({
        container: '#qexo_talks',
        url: 'https://qexo.shview.top',
        limit: 10
      });
      console.log("Qexo 说说已通过 QexoTalks 加载");
    } else {
      // 如果都没找到，等待 500ms 重试
      console.log("脚本尚未就绪，准备重试...");
      setTimeout(initQexo, 500);
    }
  })();
</script>

<style>
  /* 让说说列表更美观的样式 */
  #qexo_talks {
    margin-top: 1.5rem;
  }
  .qexo_talks_item {
    background: var(--board-bg-color);
    padding: 1.2rem;
    border-radius: 0.5rem;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    margin-bottom: 1.5rem;
    border: 1px solid var(--border-color);
  }
  .qexo_talks_date {
    font-size: 0.85rem;
    color: var(--sec-text-color);
    margin-bottom: 0.5rem;
    display: block;
  }
  .qexo_talks_content {
    line-height: 1.6;
    color: var(--text-color);
  }
</style>
