---
title: 资料管理
date: 2026-06-27 18:20:00
layout: page
description: 可视化编辑资料索引与上传公开文件
---

{% raw %}
<div class="resource-manager"
     data-catalog-url="https://raw.githubusercontent.com/shview/NKU-study-resources/main/manifest.json"
     data-catalog-api-url="https://api.github.com/repos/shview/NKU-study-resources/contents/manifest.json?ref=main"
     data-catalog-fallback-url="https://cdn.jsdelivr.net/gh/shview/NKU-study-resources@main/manifest.json"
     data-catalog-backup-url="https://fastly.jsdelivr.net/gh/shview/NKU-study-resources@main/manifest.json"
     data-local-catalog-url="http://localhost:4020/manifest.json"
     data-default-owner="shview"
     data-default-repo="NKU-study-resources"
     data-default-branch="main">
  <div class="manager-toolbar">
    <div>
      <p class="manager-kicker">Resource Studio</p>
      <h1>资料管理</h1>
    </div>
    <div class="manager-actions">
      <button id="manager-load" type="button">刷新</button>
      <label class="manager-import">导入<input id="manager-import" type="file" accept="application/json,.json"></label>
      <button id="manager-export" type="button">导出</button>
      <button id="manager-commit" type="button">提交</button>
    </div>
  </div>

  <div id="manager-status" class="manager-status">正在加载资料索引...</div>

  <div class="manager-help">
    <strong>怎么用</strong>
    <span>先等待自动读取索引；如果一直失败，点击“导入”选择 NKU-study-resources 仓库里的 manifest.json。左侧可搜索课程，右侧可以编辑课程说明、来源、贡献者、适用年级、分组说明和每个文件说明；拖拽文件到分组后，填写 GitHub token 再点“提交”。没有站长 token 的协作者请看 <a href="https://github.com/shview/NKU-study-resources/blob/main/CONTRIBUTING.md" target="_blank" rel="noopener noreferrer">协作说明</a>。</span>
  </div>

  <div class="manager-github">
    <label>Owner<input id="github-owner" type="text"></label>
    <label>Repo<input id="github-repo" type="text"></label>
    <label>Branch<input id="github-branch" type="text"></label>
    <label>Token<input id="github-token" type="password" placeholder="Fine-grained token，不会上传到本站"></label>
  </div>

  <div class="manager-layout" hidden>
    <aside class="manager-sidebar">
      <div class="manager-side-head">
        <strong>课程</strong>
        <button id="course-add" type="button">新增</button>
      </div>
      <input id="course-filter" class="course-filter" type="search" placeholder="搜索课程、学期、板块" autocomplete="off">
      <div id="course-list" class="course-list"></div>
    </aside>

    <div class="manager-resizer" role="separator" aria-label="调整左右栏宽度"></div>

    <main class="manager-main">
      <section class="manager-panel">
        <h2>课程信息</h2>
        <div class="manager-grid">
          <label>ID<input id="course-id" type="text"></label>
          <label>学期<input id="course-term" type="text"></label>
          <label>板块<input id="course-group" type="text"></label>
          <label>课程名<input id="course-title" type="text"></label>
          <label>来源<input id="course-source" type="text"></label>
          <label>更新时间<input id="course-updated" type="date"></label>
          <label>适用年级<input id="course-grades" type="text" placeholder="用顿号或逗号分隔"></label>
          <label>贡献者<input id="course-contributors" type="text" placeholder="用顿号或逗号分隔"></label>
          <label>标签<input id="course-tags" type="text" placeholder="用顿号或逗号分隔"></label>
          <label>资源路径<input id="course-base-path" type="text"></label>
        </div>
        <label class="manager-wide">说明<textarea id="course-summary" rows="3"></textarea></label>
        <div class="manager-row-actions">
          <button id="course-autofill" type="button">自动填写</button>
          <button id="course-save" type="button">保存课程</button>
          <button id="course-delete" type="button">删除课程</button>
        </div>
      </section>

      <section class="manager-panel">
        <div class="manager-section-head">
          <h2>分组与文件</h2>
          <button id="section-add" type="button">新增分组</button>
        </div>
        <div id="section-list"></div>
      </section>
    </main>
  </div>
</div>

<script>
(function () {
  var root = document.querySelector(".resource-manager");
  var status = document.getElementById("manager-status");
  var layout = document.querySelector(".manager-layout");
  var resizer = document.querySelector(".manager-resizer");
  var courseList = document.getElementById("course-list");
  var courseFilter = document.getElementById("course-filter");
  var sectionList = document.getElementById("section-list");
  var manifest = null;
  var selectedCourseId = "";
  var pendingUploads = [];
  var draggingFile = null;

  var fields = {
    id: document.getElementById("course-id"),
    term: document.getElementById("course-term"),
    group: document.getElementById("course-group"),
    title: document.getElementById("course-title"),
    source: document.getElementById("course-source"),
    updated: document.getElementById("course-updated"),
    grades: document.getElementById("course-grades"),
    contributors: document.getElementById("course-contributors"),
    tags: document.getElementById("course-tags"),
    basePath: document.getElementById("course-base-path"),
    summary: document.getElementById("course-summary")
  };

  var github = {
    owner: document.getElementById("github-owner"),
    repo: document.getElementById("github-repo"),
    branch: document.getElementById("github-branch"),
    token: document.getElementById("github-token")
  };

  github.owner.value = root.dataset.defaultOwner;
  github.repo.value = root.dataset.defaultRepo;
  github.branch.value = root.dataset.defaultBranch;

  function setStatus(message, isError) {
    status.textContent = message || "";
    status.classList.toggle("is-error", Boolean(isError));
    status.hidden = !message;
  }

  function catalogUrls() {
    var isLocal = ["localhost", "127.0.0.1"].indexOf(location.hostname) !== -1;
    var urls = [];
    if (isLocal && root.dataset.localCatalogUrl) urls.push(root.dataset.localCatalogUrl);
    [root.dataset.catalogUrl, root.dataset.catalogApiUrl, root.dataset.catalogFallbackUrl, root.dataset.catalogBackupUrl].forEach(function (url) {
      if (url && urls.indexOf(url) === -1) urls.push(url);
    });
    return urls;
  }

  function fetchJson(url, timeout) {
    var controller = window.AbortController ? new AbortController() : null;
    var timer = controller ? setTimeout(function () { controller.abort(); }, timeout || 8000) : null;
    return fetch(url, { cache: "no-cache", signal: controller ? controller.signal : undefined })
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
    setStatus("正在加载资料索引... " + url);
    return fetchJson(url, 8000).catch(function (error) {
      errors.push(url + "：" + (error.name === "AbortError" ? "超时" : error.message));
      return loadFromUrls(urls, index + 1, errors);
    });
  }

  function applyManifest(data) {
    manifest = data;
    manifest.courses = manifest.courses || [];
    sortManifestFiles();
    selectedCourseId = manifest.courses[0] ? manifest.courses[0].id : "";
    layout.hidden = false;
    setStatus("");
    render();
  }

  function splitList(value) {
    return String(value || "").split(/[、,，]/).map(function (item) { return item.trim(); }).filter(Boolean);
  }

  function joinList(value) {
    return (value || []).join("、");
  }

  function today() {
    return new Date().toISOString().slice(0, 10);
  }

  function courseById(id) {
    return (manifest.courses || []).find(function (course) { return course.id === id; });
  }

  function slug(value) {
    return String(value || "").trim().toLowerCase()
      .replace(/\s+/g, "-")
      .replace(/[\\/:*?"<>|]/g, "-")
      .replace(/-+/g, "-")
      .replace(/^-|-$/g, "") || "new-course";
  }

  function escapeHtml(value) {
    return String(value || "").replace(/[&<>"']/g, function (char) {
      return ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" })[char];
    });
  }

  function normalizeBasePath(course) {
    var base = course.basePath || [course.term, course.group, course.title].filter(Boolean).join("/");
    return base.replace(/^\/+/, "").replace(/\/?$/, "/");
  }

  function suggestedCourseId(course) {
    return slug(course.title || [course.term, course.group].filter(Boolean).join("-"));
  }

  function suggestedBasePath(course) {
    return normalizeBasePath({
      term: course.term,
      group: course.group,
      title: course.title,
      basePath: ""
    });
  }

  function stripExtension(name) {
    return String(name || "").replace(/\.[^.]+$/, "");
  }

  function baseName(path) {
    return String(path || "").split("/").pop();
  }

  function safePathPart(value) {
    return String(value || "").trim().replace(/[\\/:*?"<>|]/g, "-").replace(/-+/g, "-");
  }

  function suggestedFilePath(section, file) {
    var title = file.title || baseName(file.path) || "未命名文件";
    var prefix = section.title ? safePathPart(section.title) + "/" : "";
    return prefix + safePathPart(title);
  }

  function fileSortKey(file) {
    return String(file.title || file.path || "").trim();
  }

  function sortFiles(files) {
    return (files || []).sort(function (a, b) {
      return fileSortKey(a).localeCompare(fileSortKey(b), "zh-Hans-CN", {
        numeric: true,
        sensitivity: "base"
      });
    });
  }

  function sortManifestFiles() {
    (manifest.courses || []).forEach(function (course) {
      (course.sections || []).forEach(function (section) {
        section.files = sortFiles(section.files || []);
      });
    });
  }

  function textIncludes(value, query) {
    return String(value || "").toLowerCase().indexOf(query) !== -1;
  }

  function fileSize(bytes) {
    if (!bytes) return "0 B";
    if (bytes >= 1024 * 1024) return (bytes / 1024 / 1024).toFixed(1) + " MB";
    if (bytes >= 1024) return Math.round(bytes / 1024) + " KB";
    return bytes + " B";
  }

  function courseFileCount(course) {
    return (course.sections || []).reduce(function (sum, section) {
      return sum + (section.files || []).length;
    }, 0);
  }

  function loadManifest() {
    loadFromUrls(catalogUrls(), 0)
      .then(applyManifest)
      .catch(function (error) {
        setStatus("加载失败：" + error.message + "。可以点“导入”手动选择 manifest.json 继续编辑。", true);
      });
  }

  function render() {
    renderCourses();
    renderCourseEditor();
    renderSections();
  }

  function renderCourses() {
    var query = courseFilter.value.trim().toLowerCase();
    var courses = (manifest.courses || []).filter(function (course) {
      if (!query) return true;
      return [course.title, course.term, course.group, course.id, course.summary, (course.tags || []).join(" ")]
        .some(function (value) { return textIncludes(value, query); });
    });
    var terms = {};
    courses.forEach(function (course) {
      var term = course.term || "未分类学期";
      var group = course.group || "未分类板块";
      terms[term] = terms[term] || {};
      terms[term][group] = terms[term][group] || [];
      terms[term][group].push(course);
    });
    courseList.innerHTML = Object.keys(terms).map(function (term) {
      return '<details class="course-term" open><summary>' + escapeHtml(term) + '</summary>' +
        Object.keys(terms[term]).map(function (group) {
          return '<details class="course-group" open><summary>' + escapeHtml(group) + '</summary>' +
            terms[term][group].map(function (course) {
              return '<button class="course-item' + (course.id === selectedCourseId ? " is-active" : "") + '" type="button" data-id="' + escapeHtml(course.id) + '">' +
                '<strong>' + escapeHtml(course.title || "未命名课程") + '</strong>' +
                '<small>' + courseFileCount(course) + ' 个文件</small>' +
              '</button>';
            }).join("") +
          '</details>';
        }).join("") +
      '</details>';
    }).join("") || '<div class="manager-empty">没有匹配课程。</div>';
  }

  function renderCourseEditor() {
    var course = courseById(selectedCourseId);
    Object.keys(fields).forEach(function (key) { fields[key].value = ""; });
    if (!course) return;

    fields.id.value = course.id || "";
    fields.term.value = course.term || "";
    fields.group.value = course.group || "";
    fields.title.value = course.title || "";
    fields.source.value = course.source || "";
    fields.updated.value = course.updated || today();
    fields.grades.value = joinList(course.grades);
    fields.contributors.value = joinList(course.contributors);
    fields.tags.value = joinList(course.tags);
    fields.basePath.value = course.basePath || "";
    fields.basePath.dataset.autoValue = suggestedBasePath(course);
    fields.id.dataset.autoValue = suggestedCourseId(course);
    fields.summary.value = course.summary || "";
  }

  function renderSections() {
    var course = courseById(selectedCourseId);
    if (!course) {
      sectionList.innerHTML = '<div class="manager-empty">请先新增或选择课程。</div>';
      return;
    }

    sectionList.innerHTML = (course.sections || []).map(function (section, sectionIndex) {
      var files = section.files || [];
      return '<article class="section-editor" data-section="' + sectionIndex + '">' +
        '<div class="section-editor-head">' +
          '<input class="section-title" type="text" value="' + escapeHtml(section.title || "") + '" title="' + escapeHtml(section.title || "") + '" placeholder="分组标题">' +
          '<label class="section-collapse"><input class="section-collapsed" type="checkbox"' + (section.collapsed ? " checked" : "") + '> 默认收起</label>' +
          '<button class="section-delete" type="button" title="删除分组" aria-label="删除分组">×</button>' +
        '</div>' +
        '<textarea class="section-note" rows="2" title="' + escapeHtml(section.note || "") + '" placeholder="分组说明">' + escapeHtml(section.note || "") + '</textarea>' +
        '<div class="drop-zone" data-section="' + sectionIndex + '">拖拽文件到这里，或点击选择<input class="file-picker" type="file" multiple></div>' +
        '<div class="file-list">' + (files.length ? '<div class="file-list-head"><span></span><span>文件名</span><span>大小</span><span></span></div>' : '') + files.map(function (file, fileIndex) {
          return '<div class="file-row" data-file="' + fileIndex + '">' +
            '<span class="drag-handle" draggable="true" title="拖动排序" aria-label="拖动排序">::</span>' +
            '<label><span>文件名</span><input class="file-title" type="text" value="' + escapeHtml(file.title || "") + '" title="' + escapeHtml(file.title || "") + '" placeholder="文件名"></label>' +
            '<span class="file-size">' + fileSize(file.size) + '</span>' +
            '<button class="file-delete" type="button" title="删除" aria-label="删除">×</button>' +
          '</div>';
        }).join("") + '</div>' +
      '</article>';
    }).join("");
  }

  function saveCourseFromForm() {
    var course = courseById(selectedCourseId);
    if (!course) return;
    var oldId = course.id;
    course.id = fields.id.value.trim() || suggestedCourseId({
      term: fields.term.value.trim(),
      group: fields.group.value.trim(),
      title: fields.title.value.trim()
    });
    course.term = fields.term.value.trim();
    course.group = fields.group.value.trim();
    course.title = fields.title.value.trim();
    course.summary = fields.summary.value.trim();
    course.source = fields.source.value.trim();
    course.updated = fields.updated.value || today();
    course.grades = splitList(fields.grades.value);
    course.contributors = splitList(fields.contributors.value);
    course.tags = splitList(fields.tags.value);
    course.basePath = fields.basePath.value.trim() || suggestedBasePath(course);
    selectedCourseId = course.id;
    pendingUploads.forEach(function (upload) {
      if (upload.courseId === oldId) upload.courseId = course.id;
    });
    render();
    setStatus("课程已保存。");
  }

  function autoFillCourseFields() {
    var draft = {
      term: fields.term.value.trim() || "大一上",
      group: fields.group.value.trim() || "通识必修课",
      title: fields.title.value.trim() || "新课程"
    };
    if (!fields.term.value.trim()) fields.term.value = draft.term;
    if (!fields.group.value.trim()) fields.group.value = draft.group;
    if (!fields.title.value.trim()) fields.title.value = draft.title;
    if (!fields.id.value.trim() || fields.id.value.indexOf("course-") === 0 || fields.id.value === fields.id.dataset.autoValue) {
      fields.id.value = suggestedCourseId(draft);
    }
    if (!fields.basePath.value.trim() || fields.basePath.value === fields.basePath.dataset.autoValue) {
      fields.basePath.value = suggestedBasePath(draft);
    }
    if (!fields.updated.value) fields.updated.value = today();
    fields.id.dataset.autoValue = fields.id.value;
    fields.basePath.dataset.autoValue = fields.basePath.value;
    setStatus("已自动填写 ID、资源路径和更新时间。");
  }

  function autoFillFromTyping() {
    var draft = {
      term: fields.term.value.trim(),
      group: fields.group.value.trim(),
      title: fields.title.value.trim()
    };
    if (draft.title && (!fields.id.value.trim() || fields.id.value.indexOf("course-") === 0 || fields.id.value === fields.id.dataset.autoValue)) {
      fields.id.value = suggestedCourseId(draft);
      fields.id.dataset.autoValue = fields.id.value;
    }
    if ((draft.term || draft.group || draft.title) && (!fields.basePath.value.trim() || fields.basePath.value === fields.basePath.dataset.autoValue)) {
      fields.basePath.value = suggestedBasePath(draft);
      fields.basePath.dataset.autoValue = fields.basePath.value;
    }
    if (!fields.updated.value) fields.updated.value = today();
  }

  function saveSectionsFromDom() {
    var course = courseById(selectedCourseId);
    if (!course) return;
    Array.prototype.forEach.call(sectionList.querySelectorAll(".section-editor"), function (sectionNode) {
      var sectionIndex = Number(sectionNode.dataset.section);
      var section = course.sections[sectionIndex];
      section.title = sectionNode.querySelector(".section-title").value.trim();
      section.note = sectionNode.querySelector(".section-note").value.trim();
      section.collapsed = sectionNode.querySelector(".section-collapsed").checked;
      Array.prototype.forEach.call(sectionNode.querySelectorAll(".file-row"), function (fileNode) {
        var file = section.files[Number(fileNode.dataset.file)];
        var oldPath = file.path;
        file.title = fileNode.querySelector(".file-title").value.trim();
        file.path = suggestedFilePath(section, file);
        if (!file.description) file.description = stripExtension(file.title) + "。";
        pendingUploads.forEach(function (upload) {
          if (upload.courseId === course.id && upload.sectionIndex === sectionIndex && upload.path === oldPath) {
            upload.path = file.path;
          }
        });
      });
    });
  }

  function addCourse() {
    if (!manifest) {
      setStatus("还没有资料索引，请先刷新或导入 manifest.json。", true);
      return;
    }
    var id = "course-" + Date.now();
    manifest.courses.push({
      id: id,
      term: "大一上",
      group: "通识必修课",
      title: "新课程",
      summary: "",
      source: "",
      contributors: [],
      updated: today(),
      grades: [],
      tags: [],
      basePath: "大一上/通识必修课/新课程/",
      sections: []
    });
    selectedCourseId = id;
    render();
  }

  function addSection() {
    if (!manifest) {
      setStatus("还没有资料索引，请先刷新或导入 manifest.json。", true);
      return;
    }
    saveCourseFromForm();
    var course = courseById(selectedCourseId);
    if (!course) return;
    course.sections = course.sections || [];
    course.sections.push({ title: "新分组", note: "", collapsed: false, files: [] });
    renderSections();
  }

  function moveItem(list, from, to) {
    if (to < 0 || to >= list.length) return;
    var item = list.splice(from, 1)[0];
    list.splice(to, 0, item);
  }


  function addFiles(sectionIndex, files) {
    saveCourseFromForm();
    saveSectionsFromDom();
    var course = courseById(selectedCourseId);
    var section = course.sections[sectionIndex];
    var prefix = section.title ? section.title.replace(/[\\/:*?"<>|]/g, "-") + "/" : "";
    Array.prototype.forEach.call(files, function (file) {
      var relativePath = prefix + file.name;
      section.files.push({
        title: file.name,
        path: relativePath,
        size: file.size,
        description: stripExtension(file.name) + "。"
      });
      pendingUploads.push({
        courseId: course.id,
        sectionIndex: sectionIndex,
        path: relativePath,
        file: file
      });
    });
    sortFiles(section.files);
    renderSections();
    setStatus("已加入 " + files.length + " 个待上传文件。");
  }

  function exportManifest() {
    if (!manifest) {
      setStatus("还没有资料索引，请先刷新或导入 manifest.json。", true);
      return;
    }
    saveCourseFromForm();
    saveSectionsFromDom();
    manifest.updated = today();
    var blob = new Blob([JSON.stringify(manifest, null, 2) + "\n"], { type: "application/json" });
    var link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = "manifest.json";
    link.click();
    URL.revokeObjectURL(link.href);
  }

  function toBase64(text) {
    return btoa(unescape(encodeURIComponent(text)));
  }

  function fileToBase64(file) {
    return new Promise(function (resolve, reject) {
      var reader = new FileReader();
      reader.onload = function () {
        resolve(String(reader.result).split(",")[1]);
      };
      reader.onerror = reject;
      reader.readAsDataURL(file);
    });
  }

  function githubRequest(path, options) {
    var token = github.token.value.trim();
    if (!token) throw new Error("请先填写 GitHub token。");
    return fetch("https://api.github.com" + path, Object.assign({
      headers: {
        "Accept": "application/vnd.github+json",
        "Authorization": "Bearer " + token,
        "X-GitHub-Api-Version": "2022-11-28"
      }
    }, options || {})).then(function (response) {
      return response.json().then(function (body) {
        if (!response.ok) throw new Error(body.message || ("HTTP " + response.status));
        return body;
      });
    });
  }

  function getContentSha(owner, repo, branch, filePath) {
    return githubRequest("/repos/" + owner + "/" + repo + "/contents/" + encodeURIComponent(filePath).replace(/%2F/g, "/") + "?ref=" + encodeURIComponent(branch))
      .then(function (body) { return body.sha; })
      .catch(function (error) {
        if (String(error.message).indexOf("Not Found") !== -1) return null;
        throw error;
      });
  }

  function putContent(owner, repo, branch, filePath, contentBase64, message) {
    return getContentSha(owner, repo, branch, filePath).then(function (sha) {
      var payload = {
        message: message,
        content: contentBase64,
        branch: branch
      };
      if (sha) payload.sha = sha;
      return githubRequest("/repos/" + owner + "/" + repo + "/contents/" + encodeURIComponent(filePath).replace(/%2F/g, "/"), {
        method: "PUT",
        headers: {
          "Accept": "application/vnd.github+json",
          "Authorization": "Bearer " + github.token.value.trim(),
          "X-GitHub-Api-Version": "2022-11-28",
          "Content-Type": "application/json"
        },
        body: JSON.stringify(payload)
      });
    });
  }

  function commitToGithub() {
    if (!manifest) {
      setStatus("还没有资料索引，请先刷新或导入 manifest.json。", true);
      return;
    }
    saveCourseFromForm();
    saveSectionsFromDom();
    manifest.updated = today();

    var owner = github.owner.value.trim();
    var repo = github.repo.value.trim();
    var branch = github.branch.value.trim() || "main";
    if (!owner || !repo) {
      setStatus("请填写 GitHub owner 和 repo。", true);
      return;
    }

    setStatus("正在提交 manifest...");
    var queue = Promise.resolve()
      .then(function () {
        return putContent(owner, repo, branch, "manifest.json", toBase64(JSON.stringify(manifest, null, 2) + "\n"), "Update resources manifest");
      });

    pendingUploads.forEach(function (upload) {
      queue = queue.then(function () {
        var course = courseById(upload.courseId);
        if (!course) return null;
        var targetPath = "resources/" + normalizeBasePath(course) + upload.path;
        setStatus("正在上传 " + targetPath);
        return fileToBase64(upload.file).then(function (content) {
          return putContent(owner, repo, branch, targetPath, content, "Upload " + upload.file.name);
        });
      });
    });

    queue.then(function () {
      pendingUploads = [];
      setStatus("提交完成。GitHub Pages 或博客部署可能还需要等待。");
    }).catch(function (error) {
      setStatus("提交失败：" + error.message, true);
    });
  }

  courseList.addEventListener("click", function (event) {
    var button = event.target.closest(".course-item");
    if (!button) return;
    saveCourseFromForm();
    saveSectionsFromDom();
    selectedCourseId = button.dataset.id;
    render();
  });

  sectionList.addEventListener("input", function (event) {
    if (event.target.matches(".section-title, .section-note, .file-title")) {
      event.target.title = event.target.value;
    }
    saveSectionsFromDom();
  });

  courseFilter.addEventListener("input", renderCourses);

  sectionList.addEventListener("click", function (event) {
    var course = courseById(selectedCourseId);
    var sectionNode = event.target.closest(".section-editor");
    if (!course || !sectionNode) return;
    var sectionIndex = Number(sectionNode.dataset.section);
    var fileNode = event.target.closest(".file-row");

    saveSectionsFromDom();
    if (event.target.closest(".section-delete")) course.sections.splice(sectionIndex, 1);
    if (fileNode) {
      var fileIndex = Number(fileNode.dataset.file);
      var files = course.sections[sectionIndex].files;
      if (event.target.closest(".file-delete")) files.splice(fileIndex, 1);
    }
    renderSections();
  });

  sectionList.addEventListener("dragstart", function (event) {
    var handle = event.target.closest(".drag-handle");
    var row = event.target.closest(".file-row");
    var sectionNode = event.target.closest(".section-editor");
    if (!handle || !row || !sectionNode) {
      event.preventDefault();
      return;
    }
    saveSectionsFromDom();
    draggingFile = {
      sectionIndex: Number(sectionNode.dataset.section),
      fileIndex: Number(row.dataset.file)
    };
    row.classList.add("is-dragging");
    event.dataTransfer.effectAllowed = "move";
    event.dataTransfer.setData("text/plain", "file-row");
  });

  sectionList.addEventListener("dragend", function (event) {
    var row = event.target.closest(".file-row");
    if (row) row.classList.remove("is-dragging");
    draggingFile = null;
  });

  sectionList.addEventListener("dragover", function (event) {
    if (event.target.closest(".file-row") && draggingFile) {
      event.preventDefault();
      event.dataTransfer.dropEffect = "move";
      return;
    }
    if (event.target.closest(".drop-zone")) event.preventDefault();
  });

  sectionList.addEventListener("drop", function (event) {
    var row = event.target.closest(".file-row");
    if (row && draggingFile) {
      event.preventDefault();
      var course = courseById(selectedCourseId);
      var sectionNode = row.closest(".section-editor");
      var targetSectionIndex = Number(sectionNode.dataset.section);
      var targetFileIndex = Number(row.dataset.file);
      if (course && targetSectionIndex === draggingFile.sectionIndex && targetFileIndex !== draggingFile.fileIndex) {
        saveSectionsFromDom();
        moveItem(course.sections[targetSectionIndex].files, draggingFile.fileIndex, targetFileIndex);
        renderSections();
      }
      draggingFile = null;
      return;
    }
    var zone = event.target.closest(".drop-zone");
    if (!zone) return;
    event.preventDefault();
    addFiles(Number(zone.dataset.section), event.dataTransfer.files);
  });

  sectionList.addEventListener("change", function (event) {
    if (!event.target.classList.contains("file-picker")) return;
    var sectionNode = event.target.closest(".section-editor");
    addFiles(Number(sectionNode.dataset.section), event.target.files);
    event.target.value = "";
  });

  document.getElementById("manager-load").addEventListener("click", loadManifest);
  document.getElementById("manager-import").addEventListener("change", function (event) {
    var file = event.target.files[0];
    if (!file) return;
    var reader = new FileReader();
    reader.onload = function () {
      try {
        applyManifest(JSON.parse(reader.result));
        setStatus("已导入 " + file.name + "。");
      } catch (error) {
        setStatus("导入失败：" + error.message, true);
      }
    };
    reader.onerror = function () {
      setStatus("导入失败：无法读取文件。", true);
    };
    reader.readAsText(file, "utf-8");
    event.target.value = "";
  });
  document.getElementById("manager-export").addEventListener("click", exportManifest);
  document.getElementById("manager-commit").addEventListener("click", commitToGithub);
  document.getElementById("course-add").addEventListener("click", addCourse);
  document.getElementById("section-add").addEventListener("click", addSection);
  document.getElementById("course-autofill").addEventListener("click", autoFillCourseFields);
  document.getElementById("course-save").addEventListener("click", saveCourseFromForm);
  document.getElementById("course-delete").addEventListener("click", function () {
    if (!manifest) {
      setStatus("还没有资料索引，请先刷新或导入 manifest.json。", true);
      return;
    }
    manifest.courses = manifest.courses.filter(function (course) { return course.id !== selectedCourseId; });
    selectedCourseId = manifest.courses[0] ? manifest.courses[0].id : "";
    render();
  });

  [fields.term, fields.group, fields.title].forEach(function (field) {
    field.addEventListener("input", autoFillFromTyping);
  });

  resizer.addEventListener("mousedown", function (event) {
    event.preventDefault();
    var startX = event.clientX;
    var startWidth = document.querySelector(".manager-sidebar").getBoundingClientRect().width;
    document.body.classList.add("is-resizing-manager");
    function onMove(moveEvent) {
      var nextWidth = Math.max(220, Math.min(520, startWidth + moveEvent.clientX - startX));
      layout.style.setProperty("--manager-sidebar-width", nextWidth + "px");
    }
    function onUp() {
      document.body.classList.remove("is-resizing-manager");
      window.removeEventListener("mousemove", onMove);
      window.removeEventListener("mouseup", onUp);
    }
    window.addEventListener("mousemove", onMove);
    window.addEventListener("mouseup", onUp);
  });

  loadManifest();
})();
</script>

<style>
.resource-manager {
  max-width: 1320px;
  margin: 0 auto;
}

.manager-toolbar,
.manager-section-head,
.manager-side-head,
.section-editor-head,
.manager-row-actions,
.manager-actions {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.manager-kicker {
  margin: 0 0 6px;
  color: #5d7b74;
  font-size: 13px;
  letter-spacing: 0;
}

.manager-toolbar h1,
.manager-panel h2 {
  margin: 0;
}

.manager-actions,
.manager-row-actions {
  flex-wrap: wrap;
  justify-content: flex-start;
}

.resource-manager button,
.manager-import {
  min-height: 36px;
  border: 1px solid rgba(63, 132, 119, 0.34);
  border-radius: 8px;
  color: #2f6d62;
  background: rgba(63, 132, 119, 0.08);
  cursor: pointer;
}

.resource-manager button:hover,
.manager-import:hover {
  background: rgba(63, 132, 119, 0.14);
}

.manager-import {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0 14px;
  font-size: 14px;
}

.manager-import input {
  position: absolute;
  width: 1px;
  height: 1px;
  opacity: 0;
  pointer-events: none;
}

.manager-status {
  margin: 18px 0;
  padding: 12px 14px;
  border: 1px solid rgba(96, 117, 138, 0.22);
  border-radius: 8px;
  color: #60758a;
  background: rgba(96, 117, 138, 0.06);
}

.manager-status.is-error {
  border-color: rgba(194, 81, 81, 0.34);
  color: #a33d3d;
  background: rgba(194, 81, 81, 0.08);
}

.manager-help {
  display: grid;
  gap: 6px;
  margin: 0 0 16px;
  padding: 12px 14px;
  border: 1px solid rgba(63, 132, 119, 0.18);
  border-radius: 8px;
  color: #60758a;
  background: rgba(63, 132, 119, 0.06);
  line-height: 1.7;
}

.manager-help strong {
  color: #2f6d62;
}

.manager-github,
.manager-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(260px, 1fr));
  gap: 12px;
}

.manager-github {
  grid-template-columns: repeat(4, minmax(180px, 1fr));
  margin-bottom: 18px;
}

.manager-layout {
  display: grid;
  grid-template-columns: var(--manager-sidebar-width, 320px) 8px minmax(0, 1fr);
  gap: 12px;
  align-items: start;
}

.manager-resizer {
  align-self: stretch;
  min-height: 240px;
  border-radius: 8px;
  cursor: col-resize;
  background: linear-gradient(to right, transparent 3px, rgba(96, 117, 138, 0.24) 3px, rgba(96, 117, 138, 0.24) 5px, transparent 5px);
}

.manager-resizer:hover,
.is-resizing-manager .manager-resizer {
  background: linear-gradient(to right, transparent 3px, rgba(63, 132, 119, 0.58) 3px, rgba(63, 132, 119, 0.58) 5px, transparent 5px);
}

.is-resizing-manager {
  cursor: col-resize;
  user-select: none;
}

.manager-sidebar,
.manager-panel,
.section-editor {
  border: 1px solid rgba(96, 117, 138, 0.18);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
}

.manager-sidebar {
  position: sticky;
  top: 86px;
  padding: 14px;
}

.manager-panel {
  padding: 16px;
  margin-bottom: 18px;
}

.course-list {
  display: grid;
  gap: 8px;
  margin-top: 12px;
}

.course-filter {
  margin-top: 12px;
}

.course-term,
.course-group {
  display: grid;
  gap: 8px;
}

.course-term summary,
.course-group summary {
  cursor: pointer;
  color: #60758a;
  line-height: 1.6;
}

.course-term summary {
  font-size: 17px;
  font-weight: 700;
}

.course-group {
  margin-left: 10px;
}

.course-group summary {
  font-size: 14px;
  font-weight: 600;
}

.course-item {
  display: block;
  width: 100%;
  padding: 10px 12px;
  text-align: left;
}

.course-item strong,
.course-item small {
  display: block;
}

.course-item small {
  margin-top: 4px;
  color: #60758a;
}

.course-item.is-active {
  background: rgba(63, 132, 119, 0.16);
}

.resource-manager label {
  display: grid;
  gap: 6px;
  color: #60758a;
  font-size: 13px;
}

.resource-manager .manager-import {
  display: inline-flex;
  gap: 0;
  color: #2f6d62;
  font-size: 14px;
}

.resource-manager input,
.resource-manager textarea {
  width: 100%;
  min-height: 46px;
  padding: 10px 12px;
  border: 1px solid rgba(96, 117, 138, 0.26);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
  color: var(--text-color, #333);
  font-size: 15px;
  line-height: 1.5;
}

.resource-manager textarea {
  min-height: 96px;
  resize: vertical;
}

.manager-wide {
  margin: 12px 0;
}

.manager-grid label:nth-child(5),
.manager-grid label:nth-child(7),
.manager-grid label:nth-child(9),
.manager-grid label:nth-child(10) {
  grid-column: span 2;
}

.section-editor {
  padding: 14px;
  margin-top: 12px;
  min-width: 0;
  overflow: hidden;
}

.section-editor-head {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 112px 40px;
  align-items: center;
}

.section-title {
  min-width: 0;
}

.section-collapse {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.section-collapse input {
  width: 18px;
  min-height: 18px;
  flex: 0 0 auto;
}

.section-note {
  margin: 10px 0;
}

.drop-zone {
  position: relative;
  padding: 18px;
  border: 1px dashed rgba(63, 132, 119, 0.42);
  border-radius: 8px;
  color: #60758a;
  background: rgba(63, 132, 119, 0.06);
  text-align: center;
  cursor: pointer;
}

.drop-zone input {
  position: absolute;
  inset: 0;
  opacity: 0;
  cursor: pointer;
}

.file-list {
  display: grid;
  gap: 8px;
  margin-top: 12px;
  min-width: 0;
}

.file-list-head {
  display: grid;
  grid-template-columns: 28px minmax(0, 1fr) 72px 40px;
  gap: 10px;
  padding: 0 2px;
  color: #60758a;
  font-size: 12px;
  line-height: 1.4;
}

.file-row {
  display: grid;
  grid-template-columns: 28px minmax(0, 1fr) 72px 40px;
  gap: 10px;
  align-items: center;
  min-width: 0;
  overflow: visible;
}

.drag-handle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 42px;
  border-radius: 8px;
  color: #60758a;
  cursor: grab;
  font-weight: 700;
  letter-spacing: 0;
  user-select: none;
}

.drag-handle:hover {
  background: rgba(96, 117, 138, 0.1);
  color: #2f6d62;
}

.file-row.is-dragging {
  opacity: 0.55;
}

.file-row label {
  min-width: 0;
}

.file-row label span {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}

.file-row input,
.file-row .file-size,
.file-row input {
  overflow: hidden;
  text-overflow: ellipsis;
  user-select: text;
}

.file-size {
  color: #60758a;
  white-space: nowrap;
}

.file-delete {
  width: 40px;
  min-width: 40px;
  padding: 0;
  font-size: 22px;
  line-height: 1;
}

.section-delete {
  width: 40px;
  min-width: 40px;
  padding: 0;
  font-size: 22px;
  line-height: 1;
}

.manager-empty {
  padding: 16px;
  color: #60758a;
}

@media (max-width: 900px) {
  .manager-github,
  .manager-grid,
  .manager-layout {
    grid-template-columns: 1fr;
  }

  .manager-resizer {
    display: none;
  }

  .manager-grid label:nth-child(5),
  .manager-grid label:nth-child(7),
  .manager-grid label:nth-child(9),
  .manager-grid label:nth-child(10) {
    grid-column: auto;
  }

  .manager-sidebar {
    position: static;
  }

  .file-row {
    grid-template-columns: 28px minmax(0, 1fr) 64px 40px;
  }

  .file-list-head {
    display: none;
  }

  .file-row label span {
    position: static;
    width: auto;
    height: auto;
    overflow: visible;
    clip: auto;
    white-space: normal;
  }

  .section-editor-head {
    grid-template-columns: minmax(0, 1fr) 112px 40px;
  }
}
</style>
{% endraw %}
