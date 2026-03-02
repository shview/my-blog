---
title: 碎碎念
layout: page
---

<div id="qexo_talks"></div>

<script src="/js/qexo_talks.js"></script>

<script>
  (function init() {
    // 确保页面加载完成后执行
    window.addEventListener('load', function() {
      // 检查 renderTalks 是否可用
      if (typeof renderTalks === 'function') {
        renderTalks({
          container: '#qexo_talks',
          url: 'https://qexo.shview.top',
          limit: 10
        });
      } else {
        console.error("Qexo 脚本加载失败，请检查 /js/qexo_talks.js 是否存在");
      }
    });
  })();
</script>

<style>
  /* 你的自定义样式保持不变 */
  .qexo_talks_item {
    background: var(--board-bg-color);
    padding: 1.2rem;
    border-radius: 0.5rem;
    margin-bottom: 1.5rem;
    border: 1px solid var(--border-color);
  }
</style>
