---
title: 碎碎念
layout: page
---

<div id="qexo_talks"></div>
<script src="https://cdn.jsdelivr.net/npm/qexo-static@1.6.0/talks.js"></script>
<script>
  // 等待 window 加载，确保上面的 js 资源已就绪
  window.addEventListener('load', function() {
    if (typeof renderTalks === 'function') {
      renderTalks({
        container: '#qexo_talks',
        url: 'https://qexo.shview.top',
        limit: 10
      });
    } else {
      // 如果还报错，尝试 QexoTalks (部分版本使用这个名称)
      QexoTalks({
        container: '#qexo_talks',
        url: 'https://qexo.shview.top',
        limit: 10
      });
    }
  });
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
