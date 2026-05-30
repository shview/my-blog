---
title: 说说
date: 2026-05-30 18:45:00
layout: page
description: 记录日常动态与碎片想法
---

<!-- 将 data-qexo-api-base 改成你的 Qexo 站点地址，例如 https://qexo.example.com -->
<div class="talks-page" data-qexo-api-base="https://qexo.shview.top">
  <div class="talks-toolbar">
    <div>
      <p class="talks-kicker">Moments</p>
      <h1>说说</h1>
    </div>
    <button id="talks-refresh" class="talks-icon-button" type="button" aria-label="刷新说说" title="刷新">
      <span aria-hidden="true">&#8635;</span>
    </button>
  </div>

  <div id="talks-status" class="talks-status">正在加载说说...</div>
  <div id="talks-list" class="talks-list" aria-live="polite"></div>
  <button id="talks-more" class="talks-more" type="button" hidden>加载更多</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/marked@12.0.2/marked.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dompurify@3.1.6/dist/purify.min.js"></script>

<script>
(function () {
  var pageRoot = document.querySelector(".talks-page");
  var list = document.getElementById("talks-list");
  var status = document.getElementById("talks-status");
  var moreButton = document.getElementById("talks-more");
  var refreshButton = document.getElementById("talks-refresh");
  var apiBase = (pageRoot.dataset.qexoApiBase || "").replace(/\/$/, "");
  var page = 1;
  var limit = 8;
  var total = 0;
  var loading = false;

  function setStatus(message, isError) {
    status.textContent = message || "";
    status.classList.toggle("is-error", Boolean(isError));
    status.hidden = !message;
  }

  function formatTime(value) {
    var timestamp = Number(value);
    if (!Number.isFinite(timestamp)) return value || "";
    var date = new Date(timestamp * 1000);
    return date.toLocaleString("zh-CN", {
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
      hour: "2-digit",
      minute: "2-digit"
    });
  }

  function fallbackMarkdown(source) {
    var html = String(source || "")
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/\n/g, "<br>");
    return html.replace(/(https?:\/\/[^\s<]+)/g, '<a href="$1" target="_blank" rel="noopener noreferrer">$1</a>');
  }

  function renderMarkdown(source) {
    if (window.marked && window.DOMPurify) {
      marked.setOptions({ breaks: true, gfm: true });
      return DOMPurify.sanitize(marked.parse(source || ""), {
        ADD_ATTR: ["target", "rel", "loading"]
      });
    }
    return fallbackMarkdown(source);
  }

  function enhanceContent(node) {
    node.querySelectorAll("a[href]").forEach(function (link) {
      link.target = "_blank";
      link.rel = "noopener noreferrer";
    });
    node.querySelectorAll("img").forEach(function (image) {
      image.loading = "lazy";
      if (!image.alt) image.alt = "说说图片";
    });
  }

  function renderTalk(talk) {
    var article = document.createElement("article");
    article.className = "talk-card";
    article.dataset.id = talk.id || "";

    var time = document.createElement("time");
    time.className = "talk-time";
    var timestamp = Number(talk.time);
    time.dateTime = Number.isFinite(timestamp) ? new Date(timestamp * 1000).toISOString() : "";
    time.textContent = formatTime(talk.time);

    var content = document.createElement("div");
    content.className = "talk-content markdown-body";
    content.innerHTML = renderMarkdown(talk.content || "");
    enhanceContent(content);

    var meta = document.createElement("div");
    meta.className = "talk-meta";

    if (Array.isArray(talk.tags) && talk.tags.length) {
      var tags = document.createElement("div");
      tags.className = "talk-tags";
      talk.tags.forEach(function (tag) {
        var span = document.createElement("span");
        span.textContent = "#" + tag;
        tags.appendChild(span);
      });
      meta.appendChild(tags);
    }

    if (talk.id) {
      var like = document.createElement("button");
      like.className = "talk-like";
      like.type = "button";
      like.textContent = (talk.liked ? "已赞 " : "赞 ") + (talk.like || 0);
      like.dataset.liked = talk.liked ? "true" : "false";
      like.addEventListener("click", function () {
        likeTalk(talk.id, like);
      });
      meta.appendChild(like);
    }

    article.appendChild(time);
    article.appendChild(content);
    article.appendChild(meta);
    return article;
  }

  function updateMoreButton() {
    moreButton.hidden = list.children.length >= total;
  }

  function loadTalks(reset) {
    if (loading) return;
    if (!apiBase) {
      setStatus("请先在 source/talks/index.md 里填写 Qexo 站点地址。", true);
      return;
    }

    loading = true;
    if (reset) {
      page = 1;
      total = 0;
      list.innerHTML = "";
      moreButton.hidden = true;
    }
    setStatus("正在加载说说...");

    fetch(apiBase + "/pub/talks/?page=" + page + "&limit=" + limit, {
      method: "GET",
      mode: "cors",
      credentials: "omit"
    })
      .then(function (response) {
        if (!response.ok) throw new Error("HTTP " + response.status);
        return response.json();
      })
      .then(function (result) {
        if (!result.status) throw new Error(result.msg || "Qexo 返回失败");
        total = Number(result.count || 0);
        (result.data || []).forEach(function (talk) {
          list.appendChild(renderTalk(talk));
        });
        page += 1;
        updateMoreButton();
        setStatus(total ? "" : "还没有说说。");
      })
      .catch(function (error) {
        setStatus("说说加载失败：" + error.message, true);
      })
      .finally(function () {
        loading = false;
      });
  }

  function likeTalk(id, button) {
    if (!apiBase) return;
    var body = new URLSearchParams();
    body.set("id", id);
    button.disabled = true;

    fetch(apiBase + "/pub/like_talk/", {
      method: "POST",
      mode: "cors",
      credentials: "omit",
      headers: { "Content-Type": "application/x-www-form-urlencoded;charset=UTF-8" },
      body: body.toString()
    })
      .then(function (response) {
        if (!response.ok) throw new Error("HTTP " + response.status);
        return response.json();
      })
      .then(function () {
        loadTalks(true);
      })
      .catch(function () {
        button.disabled = false;
      });
  }

  moreButton.addEventListener("click", function () {
    loadTalks(false);
  });
  refreshButton.addEventListener("click", function () {
    loadTalks(true);
  });

  loadTalks(true);
})();
</script>

<style>
.talks-page {
  max-width: 760px;
  margin: 0 auto;
}

.talks-toolbar {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 28px;
}

.talks-kicker {
  margin: 0 0 6px;
  color: #60758a;
  font-size: 13px;
  letter-spacing: 0;
}

.talks-toolbar h1 {
  margin: 0;
  font-size: 30px;
  line-height: 1.2;
}

.talks-icon-button {
  width: 40px;
  height: 40px;
  border: 1px solid rgba(96, 117, 138, 0.28);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
  color: #31465a;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
}

.talks-list {
  position: relative;
  display: grid;
  gap: 18px;
}

.talk-card {
  position: relative;
  padding: 18px 20px 18px 24px;
  border: 1px solid rgba(96, 117, 138, 0.18);
  border-left: 4px solid #4f8cc9;
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
  box-shadow: 0 10px 28px rgba(39, 58, 76, 0.07);
}

.talk-time {
  display: block;
  margin-bottom: 12px;
  color: #6c7f92;
  font-size: 13px;
}

.talk-content {
  color: var(--text-color, #333);
  font-size: 16px;
  line-height: 1.8;
  overflow-wrap: anywhere;
}

.talk-content > :first-child {
  margin-top: 0;
}

.talk-content > :last-child {
  margin-bottom: 0;
}

.talk-content img {
  display: block;
  max-width: 100%;
  max-height: 560px;
  margin: 12px 0;
  border-radius: 8px;
  object-fit: contain;
}

.talk-meta {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-top: 16px;
}

.talk-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.talk-tags span {
  color: #60758a;
  font-size: 13px;
}

.talk-like,
.talks-more {
  border: 1px solid rgba(79, 140, 201, 0.32);
  border-radius: 8px;
  background: transparent;
  color: #315f8d;
  cursor: pointer;
}

.talk-like {
  min-width: 72px;
  height: 32px;
  padding: 0 12px;
  font-size: 13px;
}

.talks-more {
  display: block;
  width: 160px;
  height: 40px;
  margin: 24px auto 0;
  font-size: 14px;
}

.talks-status {
  padding: 14px 16px;
  border: 1px solid rgba(96, 117, 138, 0.22);
  border-radius: 8px;
  color: #60758a;
  background: rgba(96, 117, 138, 0.06);
}

.talks-status.is-error {
  border-color: rgba(194, 81, 81, 0.34);
  color: #a33d3d;
  background: rgba(194, 81, 81, 0.08);
}

@media (max-width: 576px) {
  .talks-toolbar {
    align-items: center;
  }

  .talk-card {
    padding: 16px;
  }

  .talk-meta {
    align-items: flex-start;
    flex-direction: column;
  }
}
</style>
