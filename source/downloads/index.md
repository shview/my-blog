---
title: 资料
date: 2026-06-27 15:30:00
layout: page
description: 课程资料、文件说明与公开下载
---

<div class="resource-page"
     data-openlist-url="https://pan.shview.top/"
     data-catalog-url="https://raw.githubusercontent.com/shview/NKU-study-resources/main/manifest.json"
     data-catalog-api-url="https://api.github.com/repos/shview/NKU-study-resources/contents/manifest.json?ref=main"
     data-catalog-fallback-url="https://cdn.jsdelivr.net/gh/shview/NKU-study-resources@main/manifest.json"
     data-catalog-backup-url="https://fastly.jsdelivr.net/gh/shview/NKU-study-resources@main/manifest.json"
     data-local-catalog-url="http://localhost:4020/manifest.json">
  <div class="resource-toolbar">
    <div>
      <p class="resource-kicker">Resources</p>
      <h1>资料</h1>
    </div>
    <div class="resource-actions">
      <a class="resource-action-button" href="/downloads/manage/" target="_self" title="管理资料">管理</a>
      <a class="resource-action-button" href="https://pan.shview.top/" target="_blank" rel="noopener noreferrer" title="打开 OpenList">OpenList</a>
    </div>
  </div>

  <div class="resource-search">
    <input id="resource-query" type="search" placeholder="搜索课程、文件、说明、标签" autocomplete="off">
    <button id="resource-clear" type="button" title="清空搜索" aria-label="清空搜索">&times;</button>
  </div>

  <div id="resource-status" class="resource-status">正在加载资料索引...</div>

  <div class="resource-layout" hidden>
    <aside class="resource-sidebar" aria-label="资料分类">
      <div id="resource-tree"></div>
    </aside>
    <section class="resource-main" aria-live="polite">
      <div id="resource-breadcrumb" class="resource-breadcrumb"></div>
      <div id="resource-content"></div>
    </section>
  </div>
</div>

<script>
(function () {
  var pageRoot = document.querySelector(".resource-page");
  var layout = document.querySelector(".resource-layout");
  var status = document.getElementById("resource-status");
  var catalog = [];
  var resourceRoot = "";

  var tree = document.getElementById("resource-tree");
  var content = document.getElementById("resource-content");
  var breadcrumb = document.getElementById("resource-breadcrumb");
  var queryInput = document.getElementById("resource-query");
  var clearButton = document.getElementById("resource-clear");

  function setStatus(message, isError) {
    status.textContent = message || "";
    status.classList.toggle("is-error", Boolean(isError));
    status.hidden = !message;
  }

  function catalogUrls() {
    var isLocal = ["localhost", "127.0.0.1"].indexOf(location.hostname) !== -1;
    var urls = [];
    if (isLocal && pageRoot.dataset.localCatalogUrl) urls.push(pageRoot.dataset.localCatalogUrl);
    [pageRoot.dataset.catalogUrl, pageRoot.dataset.catalogApiUrl, pageRoot.dataset.catalogFallbackUrl, pageRoot.dataset.catalogBackupUrl].forEach(function (url) {
      if (url && urls.indexOf(url) === -1) urls.push(url);
    });
    return urls;
  }

  function fetchJson(url, timeout) {
    var controller = window.AbortController ? new AbortController() : null;
    var timer = controller ? setTimeout(function () { controller.abort(); }, timeout || 8000) : null;
    return fetch(url, { mode: "cors", cache: "no-cache", signal: controller ? controller.signal : undefined })
      .then(function (response) {
        if (!response.ok) throw new Error("HTTP " + response.status);
        return response.json();
      })
      .then(function (body) {
        if (body && body.encoding === "base64" && body.content) {
          return JSON.parse(decodeURIComponent(escape(atob(String(body.content).replace(/\s/g, "")))));
        }
        return body;
      })
      .finally(function () {
        if (timer) clearTimeout(timer);
      });
  }

  function loadFromUrls(urls, index, errors) {
    errors = errors || [];
    if (index >= urls.length) {
      throw new Error(errors.join("；") || "没有可用索引地址");
    }
    var url = urls[index];
    return fetchJson(url, 8000).catch(function (error) {
      errors.push(url + "：" + (error.name === "AbortError" ? "超时" : error.message));
      return loadFromUrls(urls, index + 1, errors);
    });
  }

  function loadCatalog() {
    setStatus("正在加载资料索引...");
    loadFromUrls(catalogUrls(), 0)
      .then(function (manifest) {
        catalog = manifest.courses || [];
        resourceRoot = ["localhost", "127.0.0.1"].indexOf(location.hostname) !== -1
          ? "http://localhost:4020/resources/"
          : (manifest.resourceRoot || "");
        if (!catalog.length) throw new Error("资料索引为空");
        layout.hidden = false;
        setStatus("");
        renderTree();
        refresh();
      })
      .catch(function (error) {
        layout.hidden = true;
        setStatus("资料索引加载失败：" + error.message, true);
      });
  }

  function fileCount(course) {
    return course.sections.reduce(function (sum, section) {
      return sum + section.files.length;
    }, 0);
  }

  function formatSize(bytes) {
    if (bytes >= 1024 * 1024) return (bytes / 1024 / 1024).toFixed(1) + " MB";
    if (bytes >= 1024) return Math.round(bytes / 1024) + " KB";
    return bytes + " B";
  }

  function fileType(name) {
    var match = name.match(/\.([^.]+)$/);
    return match ? match[1].toUpperCase() : "FILE";
  }

  function slug(value) {
    return encodeURIComponent(String(value || "")).replace(/%/g, "");
  }

  function activeCourse() {
    var id = decodeURIComponent((location.hash || "").replace(/^#\/?/, ""));
    return catalog.find(function (course) { return course.id === id; }) || catalog[0];
  }

  function searchableText(course) {
    return [
      course.term,
      course.group,
      course.title,
      course.summary,
      course.source || "",
      (course.contributors || []).join(" "),
      (course.grades || []).join(" "),
      course.tags.join(" "),
      course.sections.map(function (section) {
        return section.title + " " + section.note + " " + section.files.map(function (file) {
          return file.title + " " + (file.description || "");
        }).join(" ");
      }).join(" ")
    ].join(" ").toLowerCase();
  }

  function renderTree() {
    var terms = {};
    catalog.forEach(function (course) {
      terms[course.term] = terms[course.term] || {};
      terms[course.term][course.group] = terms[course.term][course.group] || [];
      terms[course.term][course.group].push(course);
    });

    tree.innerHTML = Object.keys(terms).map(function (term) {
      var groups = terms[term];
      var termId = "resource-tree-term-" + slug(term);
      return '<div class="resource-tree-term">' +
        '<button class="resource-tree-toggle" type="button" aria-expanded="true" aria-controls="' + termId + '">' + term + '</button>' +
        '<div id="' + termId + '" class="resource-tree-body">' +
        Object.keys(groups).map(function (group) {
          var groupId = "resource-tree-group-" + slug(term) + "-" + slug(group);
          return '<div class="resource-tree-group">' +
            '<button class="resource-tree-toggle resource-tree-toggle-sub" type="button" aria-expanded="true" aria-controls="' + groupId + '">' + group + '</button>' +
            '<div id="' + groupId + '" class="resource-tree-body">' +
            groups[group].map(function (course) {
              return '<a class="resource-course-link" data-id="' + course.id + '" href="#/' + course.id + '">' +
                '<span>' + course.title + '</span><small>' + fileCount(course) + ' 个文件</small></a>';
            }).join("") +
            '</div>' +
          '</div>';
        }).join("") +
        '</div>' +
      '</div>';
    }).join("");
  }

  function renderCourse(course, query) {
    var normalizedQuery = (query || "").trim().toLowerCase();
    var sections = course.sections.map(function (section) {
      var files = section.files.filter(function (file) {
        if (!normalizedQuery) return true;
        return [course.term, course.group, course.title, section.title, section.note, file.title, file.description || "", course.tags.join(" ")]
          .join(" ").toLowerCase().indexOf(normalizedQuery) !== -1;
      });
      return Object.assign({}, section, { files: files });
    }).filter(function (section) {
      return section.files.length || (!normalizedQuery && section.note);
    });

    breadcrumb.textContent = course.term + " / " + course.group + " / " + course.title;

    content.innerHTML =
      '<article class="resource-course">' +
        '<div class="resource-course-head">' +
          '<div><h2>' + course.title + '</h2><p>' + course.summary + '</p></div>' +
          '<div class="resource-count">' + fileCount(course) + '<span>文件</span></div>' +
        '</div>' +
        renderMeta(course) +
        '<div class="resource-tags">' + course.tags.map(function (tag) { return '<span>#' + tag + '</span>'; }).join("") + '</div>' +
        (sections.length ? sections.map(function (section) { return renderSection(section, course); }).join("") : '<div class="resource-empty">没有找到匹配的资料。</div>') +
      '</article>';
  }

  function renderMeta(course) {
    var meta = [
      ["来源", course.source || "未填写"],
      ["贡献者", (course.contributors || []).join("、") || "未填写"],
      ["更新时间", course.updated || "未填写"],
      ["适用年级", (course.grades || []).join("、") || "未填写"]
    ];
    return '<dl class="resource-meta">' + meta.map(function (item) {
      return '<div><dt>' + item[0] + '</dt><dd>' + item[1] + '</dd></div>';
    }).join("") + '</dl>';
  }

  function renderSection(section, course) {
    var shouldCollapse = section.collapsed === true || section.files.length > 8;
    var sectionId = "resource-section-" + course.id + "-" + slug(section.title);
    return '<section class="resource-section">' +
      '<button class="resource-section-toggle" type="button" aria-expanded="' + (!shouldCollapse) + '" aria-controls="' + sectionId + '">' +
        '<span><strong>' + section.title + '</strong><small>' + section.note + '</small></span>' +
        '<em>' + section.files.length + ' 个文件</em>' +
      '</button>' +
      '<div class="resource-files" id="' + sectionId + '"' + (shouldCollapse ? ' hidden' : '') + '>' +
        section.files.map(function (file) { return renderFile(file, course); }).join("") +
      '</div>' +
    '</section>';
  }

  function renderFile(file, course) {
    var title = file.title;
    var path = file.path;
    var size = file.size;
    return '<a class="resource-file" href="' + makeHref(path, course) + '" download>' +
      '<span class="resource-file-type">' + fileType(title) + '</span>' +
      '<span class="resource-file-main"><strong>' + title + '</strong><small>' + sizeLabel(size) + (file.description ? " / " + file.description : "") + '</small></span>' +
      '<span class="resource-file-action">下载</span>' +
    '</a>';
  }

  function sizeLabel(size) {
    return formatSize(size);
  }

  function renderSearch(query) {
    var normalizedQuery = query.trim().toLowerCase();
    var matches = normalizedQuery ? catalog.filter(function (course) {
      return searchableText(course).indexOf(normalizedQuery) !== -1;
    }) : catalog;

    tree.querySelectorAll(".resource-course-link").forEach(function (link) {
      var isActive = link.dataset.id === activeCourse().id;
      link.classList.toggle("is-active", isActive);
    });

    if (normalizedQuery && matches.length) {
      renderCourse(matches[0], query);
      return;
    }

    if (normalizedQuery && !matches.length) {
      breadcrumb.textContent = "搜索";
      content.innerHTML = '<div class="resource-empty">没有找到匹配的资料。</div>';
      return;
    }

    renderCourse(activeCourse(), "");
  }

  function refresh() {
    renderSearch(queryInput.value);
  }

  content.addEventListener("click", function (event) {
    var button = event.target.closest(".resource-section-toggle");
    if (!button) return;
    var target = document.getElementById(button.getAttribute("aria-controls"));
    if (!target) return;
    var expanded = button.getAttribute("aria-expanded") === "true";
    button.setAttribute("aria-expanded", String(!expanded));
    target.hidden = expanded;
  });

  tree.addEventListener("click", function (event) {
    var button = event.target.closest(".resource-tree-toggle");
    if (!button) return;
    var target = document.getElementById(button.getAttribute("aria-controls"));
    if (!target) return;
    var expanded = button.getAttribute("aria-expanded") === "true";
    button.setAttribute("aria-expanded", String(!expanded));
    target.hidden = expanded;
  });

  queryInput.addEventListener("input", refresh);
  clearButton.addEventListener("click", function () {
    queryInput.value = "";
    queryInput.focus();
    refresh();
  });
  window.addEventListener("hashchange", refresh);

  function makeHref(path, course) {
    return encodeURI(resourceRoot + (course.basePath || "") + path);
  }

  loadCatalog();
})();
</script>

<style>
.resource-page {
  max-width: 1060px;
  margin: 0 auto;
}

.resource-toolbar {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 22px;
}

.resource-kicker {
  margin: 0 0 6px;
  color: #5d7b74;
  font-size: 13px;
  letter-spacing: 0;
}

.resource-toolbar h1 {
  margin: 0;
  font-size: 30px;
  line-height: 1.2;
}

.resource-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: flex-end;
}

.resource-action-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 104px;
  height: 40px;
  padding: 0 16px;
  border: 1px solid rgba(63, 132, 119, 0.34);
  border-radius: 8px;
  color: #2f6d62;
  background: rgba(63, 132, 119, 0.08);
  text-decoration: none;
}

.resource-action-button:hover {
  color: #24584f;
  text-decoration: none;
  background: rgba(63, 132, 119, 0.14);
}

.resource-search {
  position: relative;
  margin-bottom: 18px;
}

.resource-status {
  margin-bottom: 18px;
  padding: 14px 16px;
  border: 1px solid rgba(96, 117, 138, 0.22);
  border-radius: 8px;
  color: #60758a;
  background: rgba(96, 117, 138, 0.06);
}

.resource-status.is-error {
  border-color: rgba(194, 81, 81, 0.34);
  color: #a33d3d;
  background: rgba(194, 81, 81, 0.08);
}

.resource-search input {
  width: 100%;
  height: 46px;
  padding: 0 46px 0 16px;
  border: 1px solid rgba(96, 117, 138, 0.26);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
  color: var(--text-color, #333);
  outline: none;
}

.resource-search input:focus {
  border-color: rgba(63, 132, 119, 0.62);
  box-shadow: 0 0 0 3px rgba(63, 132, 119, 0.12);
}

.resource-search button {
  position: absolute;
  top: 5px;
  right: 6px;
  width: 36px;
  height: 36px;
  border: 0;
  border-radius: 8px;
  color: #60758a;
  background: transparent;
  font-size: 22px;
  line-height: 1;
  cursor: pointer;
}

.resource-layout {
  display: grid;
  grid-template-columns: 260px minmax(0, 1fr);
  gap: 18px;
  align-items: start;
}

.resource-sidebar {
  position: sticky;
  top: 86px;
  padding: 14px;
  border: 1px solid rgba(96, 117, 138, 0.18);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
}

.resource-tree-group {
  margin-top: 12px;
}

.resource-tree-toggle {
  display: flex;
  align-items: center;
  width: 100%;
  gap: 8px;
  padding: 8px 6px;
  border: 0;
  border-radius: 8px;
  color: var(--text-color, #333);
  background: transparent;
  font-size: 18px;
  font-weight: 600;
  line-height: 1.4;
  text-align: left;
  cursor: pointer;
}

.resource-tree-toggle:hover {
  background: rgba(96, 117, 138, 0.08);
}

.resource-tree-toggle::before {
  content: "▾";
  flex: 0 0 auto;
  color: #60758a;
  font-size: 12px;
  transform: rotate(-90deg);
  transition: transform 0.16s ease;
}

.resource-tree-toggle[aria-expanded="true"]::before {
  transform: rotate(0deg);
}

.resource-tree-toggle-sub {
  margin-bottom: 8px;
  color: #60758a;
  font-size: 14px;
  font-weight: 500;
}

.resource-tree-body {
  display: grid;
  gap: 2px;
}

.resource-course-link {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 10px 12px;
  border-radius: 8px;
  color: var(--text-color, #333);
  text-decoration: none;
}

.resource-course-link:hover,
.resource-course-link.is-active {
  color: #2f6d62;
  text-decoration: none;
  background: rgba(63, 132, 119, 0.09);
}

.resource-course-link small {
  color: #7b8b99;
  font-size: 12px;
}

.resource-breadcrumb {
  margin-bottom: 12px;
  color: #6c7f92;
  font-size: 13px;
}

.resource-course {
  min-width: 0;
}

.resource-course-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 18px;
  margin-bottom: 12px;
}

.resource-course-head h2 {
  margin: 0 0 8px;
  font-size: 26px;
}

.resource-course-head p {
  margin: 0;
  color: #60758a;
  line-height: 1.8;
}

.resource-count {
  min-width: 76px;
  padding: 10px 12px;
  border: 1px solid rgba(63, 132, 119, 0.22);
  border-radius: 8px;
  color: #2f6d62;
  text-align: center;
  font-size: 24px;
  font-weight: 700;
}

.resource-count span {
  display: block;
  margin-top: 2px;
  font-size: 12px;
  font-weight: 400;
}

.resource-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin: 12px 0 22px;
}

.resource-meta {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin: 16px 0 18px;
}

.resource-meta div {
  padding: 10px 12px;
  border: 1px solid rgba(96, 117, 138, 0.16);
  border-radius: 8px;
  background: rgba(96, 117, 138, 0.04);
}

.resource-meta dt,
.resource-meta dd {
  margin: 0;
}

.resource-meta dt {
  color: #6c7f92;
  font-size: 12px;
}

.resource-meta dd {
  margin-top: 4px;
  color: var(--text-color, #333);
  font-size: 14px;
  line-height: 1.5;
  overflow-wrap: anywhere;
}

.resource-tags span {
  color: #60758a;
  font-size: 13px;
}

.resource-section {
  margin-top: 24px;
}

.resource-files {
  display: grid;
  gap: 10px;
}

.resource-section-toggle {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  gap: 14px;
  margin-bottom: 10px;
  padding: 12px 14px;
  border: 1px solid rgba(96, 117, 138, 0.18);
  border-radius: 8px;
  color: var(--text-color, #333);
  background: var(--board-bg-color, #fff);
  text-align: left;
  cursor: pointer;
}

.resource-section-toggle:hover {
  border-color: rgba(63, 132, 119, 0.38);
}

.resource-section-toggle span,
.resource-section-toggle strong,
.resource-section-toggle small {
  display: block;
}

.resource-section-toggle strong {
  font-size: 20px;
  line-height: 1.35;
}

.resource-section-toggle small {
  margin-top: 4px;
  color: #6c7f92;
  font-size: 14px;
  line-height: 1.5;
}

.resource-section-toggle em {
  flex: 0 0 auto;
  color: #2f6d62;
  font-size: 13px;
  font-style: normal;
}

.resource-section-toggle::before {
  content: "▾";
  flex: 0 0 auto;
  color: #60758a;
  transform: rotate(-90deg);
  transition: transform 0.16s ease;
}

.resource-section-toggle[aria-expanded="true"]::before {
  transform: rotate(0deg);
}

.resource-file {
  display: grid;
  grid-template-columns: 58px minmax(0, 1fr) 58px;
  gap: 12px;
  align-items: center;
  padding: 12px 14px;
  border: 1px solid rgba(96, 117, 138, 0.18);
  border-radius: 8px;
  color: var(--text-color, #333);
  background: var(--board-bg-color, #fff);
  text-decoration: none;
}

.resource-file:hover {
  border-color: rgba(63, 132, 119, 0.38);
  color: var(--text-color, #333);
  text-decoration: none;
  box-shadow: 0 10px 24px rgba(39, 58, 76, 0.07);
}

.resource-file-type {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  height: 30px;
  border-radius: 8px;
  color: #2f6d62;
  background: rgba(63, 132, 119, 0.1);
  font-size: 12px;
  font-weight: 700;
}

.resource-file-main {
  min-width: 0;
}

.resource-file-main strong,
.resource-file-main small {
  display: block;
}

.resource-file-main strong {
  overflow-wrap: anywhere;
  line-height: 1.5;
}

.resource-file-main small {
  margin-top: 3px;
  color: #7b8b99;
  font-size: 12px;
}

.resource-file-action {
  color: #2f6d62;
  font-size: 14px;
  text-align: right;
}

.resource-empty {
  padding: 18px;
  border: 1px solid rgba(96, 117, 138, 0.2);
  border-radius: 8px;
  color: #60758a;
  background: rgba(96, 117, 138, 0.06);
}

@media (max-width: 768px) {
  .resource-toolbar {
    align-items: center;
  }

  .resource-layout {
    grid-template-columns: 1fr;
  }

  .resource-sidebar {
    position: static;
  }

  .resource-course-head {
    flex-direction: column;
  }

  .resource-count {
    width: 100%;
  }

  .resource-meta {
    grid-template-columns: 1fr;
  }

  .resource-file {
    grid-template-columns: 52px minmax(0, 1fr);
  }

  .resource-file-action {
    display: none;
  }
}
</style>
