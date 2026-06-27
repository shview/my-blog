---
title: 资料管理
date: 2026-06-27 18:20:00
layout: page
description: 可视化编辑资料索引与上传公开文件
---

<div class="resource-manager"
     data-catalog-url="https://raw.githubusercontent.com/shview/NKU-study-resources/main/manifest.json"
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
      <button id="manager-export" type="button">导出</button>
      <button id="manager-commit" type="button">提交</button>
    </div>
  </div>

  <div id="manager-status" class="manager-status">正在加载资料索引...</div>

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
      <div id="course-list" class="course-list"></div>
    </aside>

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
  var courseList = document.getElementById("course-list");
  var sectionList = document.getElementById("section-list");
  var manifest = null;
  var selectedCourseId = "";
  var pendingUploads = [];

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

  function catalogUrl() {
    var isLocal = ["localhost", "127.0.0.1"].indexOf(location.hostname) !== -1;
    return isLocal ? root.dataset.localCatalogUrl : root.dataset.catalogUrl;
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

  function fileSize(bytes) {
    if (!bytes) return "0 B";
    if (bytes >= 1024 * 1024) return (bytes / 1024 / 1024).toFixed(1) + " MB";
    if (bytes >= 1024) return Math.round(bytes / 1024) + " KB";
    return bytes + " B";
  }

  function loadManifest() {
    setStatus("正在加载资料索引...");
    fetch(catalogUrl(), { cache: "no-cache" })
      .then(function (response) {
        if (!response.ok) throw new Error("HTTP " + response.status);
        return response.json();
      })
      .then(function (data) {
        manifest = data;
        manifest.courses = manifest.courses || [];
        selectedCourseId = manifest.courses[0] ? manifest.courses[0].id : "";
        layout.hidden = false;
        setStatus("");
        render();
      })
      .catch(function (error) {
        setStatus("加载失败：" + error.message, true);
      });
  }

  function render() {
    renderCourses();
    renderCourseEditor();
    renderSections();
  }

  function renderCourses() {
    courseList.innerHTML = manifest.courses.map(function (course) {
      return '<button class="course-item' + (course.id === selectedCourseId ? " is-active" : "") + '" type="button" data-id="' + escapeHtml(course.id) + '">' +
        '<strong>' + escapeHtml(course.title || "未命名课程") + '</strong>' +
        '<small>' + escapeHtml([course.term, course.group].filter(Boolean).join(" / ")) + '</small>' +
      '</button>';
    }).join("");
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
          '<input class="section-title" type="text" value="' + escapeHtml(section.title || "") + '" placeholder="分组标题">' +
          '<label class="section-collapse"><input class="section-collapsed" type="checkbox"' + (section.collapsed ? " checked" : "") + '> 默认收起</label>' +
          '<button class="section-up" type="button">上移</button>' +
          '<button class="section-down" type="button">下移</button>' +
          '<button class="section-delete" type="button">删除</button>' +
        '</div>' +
        '<textarea class="section-note" rows="2" placeholder="分组说明">' + escapeHtml(section.note || "") + '</textarea>' +
        '<div class="drop-zone" data-section="' + sectionIndex + '">拖拽文件到这里，或点击选择<input class="file-picker" type="file" multiple></div>' +
        '<div class="file-list">' + files.map(function (file, fileIndex) {
          return '<div class="file-row" data-file="' + fileIndex + '">' +
            '<input class="file-title" type="text" value="' + escapeHtml(file.title || "") + '" placeholder="文件名">' +
            '<input class="file-path" type="text" value="' + escapeHtml(file.path || "") + '" placeholder="路径">' +
            '<input class="file-description" type="text" value="' + escapeHtml(file.description || "") + '" placeholder="说明">' +
            '<span>' + fileSize(file.size) + '</span>' +
            '<button class="file-up" type="button">上移</button>' +
            '<button class="file-down" type="button">下移</button>' +
            '<button class="file-delete" type="button">删除</button>' +
          '</div>';
        }).join("") + '</div>' +
      '</article>';
    }).join("");
  }

  function saveCourseFromForm() {
    var course = courseById(selectedCourseId);
    if (!course) return;
    var oldId = course.id;
    course.id = fields.id.value.trim() || slug(fields.title.value);
    course.term = fields.term.value.trim();
    course.group = fields.group.value.trim();
    course.title = fields.title.value.trim();
    course.summary = fields.summary.value.trim();
    course.source = fields.source.value.trim();
    course.updated = fields.updated.value || today();
    course.grades = splitList(fields.grades.value);
    course.contributors = splitList(fields.contributors.value);
    course.tags = splitList(fields.tags.value);
    course.basePath = fields.basePath.value.trim() || normalizeBasePath(course);
    selectedCourseId = course.id;
    pendingUploads.forEach(function (upload) {
      if (upload.courseId === oldId) upload.courseId = course.id;
    });
    render();
    setStatus("课程已保存。");
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
        file.title = fileNode.querySelector(".file-title").value.trim();
        file.path = fileNode.querySelector(".file-path").value.trim();
        file.description = fileNode.querySelector(".file-description").value.trim();
      });
    });
  }

  function addCourse() {
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
        description: ""
      });
      pendingUploads.push({
        courseId: course.id,
        sectionIndex: sectionIndex,
        path: relativePath,
        file: file
      });
    });
    renderSections();
    setStatus("已加入 " + files.length + " 个待上传文件。");
  }

  function exportManifest() {
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

  sectionList.addEventListener("input", function () {
    saveSectionsFromDom();
  });

  sectionList.addEventListener("click", function (event) {
    var course = courseById(selectedCourseId);
    var sectionNode = event.target.closest(".section-editor");
    if (!course || !sectionNode) return;
    var sectionIndex = Number(sectionNode.dataset.section);
    var fileNode = event.target.closest(".file-row");

    saveSectionsFromDom();
    if (event.target.closest(".section-delete")) course.sections.splice(sectionIndex, 1);
    if (event.target.closest(".section-up")) moveItem(course.sections, sectionIndex, sectionIndex - 1);
    if (event.target.closest(".section-down")) moveItem(course.sections, sectionIndex, sectionIndex + 1);
    if (fileNode) {
      var fileIndex = Number(fileNode.dataset.file);
      var files = course.sections[sectionIndex].files;
      if (event.target.closest(".file-delete")) files.splice(fileIndex, 1);
      if (event.target.closest(".file-up")) moveItem(files, fileIndex, fileIndex - 1);
      if (event.target.closest(".file-down")) moveItem(files, fileIndex, fileIndex + 1);
    }
    renderSections();
  });

  sectionList.addEventListener("change", function (event) {
    if (!event.target.classList.contains("file-picker")) return;
    var sectionNode = event.target.closest(".section-editor");
    addFiles(Number(sectionNode.dataset.section), event.target.files);
    event.target.value = "";
  });

  sectionList.addEventListener("dragover", function (event) {
    if (event.target.closest(".drop-zone")) event.preventDefault();
  });

  sectionList.addEventListener("drop", function (event) {
    var zone = event.target.closest(".drop-zone");
    if (!zone) return;
    event.preventDefault();
    addFiles(Number(zone.dataset.section), event.dataTransfer.files);
  });

  document.getElementById("manager-load").addEventListener("click", loadManifest);
  document.getElementById("manager-export").addEventListener("click", exportManifest);
  document.getElementById("manager-commit").addEventListener("click", commitToGithub);
  document.getElementById("course-add").addEventListener("click", addCourse);
  document.getElementById("section-add").addEventListener("click", addSection);
  document.getElementById("course-save").addEventListener("click", saveCourseFromForm);
  document.getElementById("course-delete").addEventListener("click", function () {
    manifest.courses = manifest.courses.filter(function (course) { return course.id !== selectedCourseId; });
    selectedCourseId = manifest.courses[0] ? manifest.courses[0].id : "";
    render();
  });

  loadManifest();
})();
</script>

<style>
.resource-manager {
  max-width: 1120px;
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

.resource-manager button {
  min-height: 36px;
  border: 1px solid rgba(63, 132, 119, 0.34);
  border-radius: 8px;
  color: #2f6d62;
  background: rgba(63, 132, 119, 0.08);
  cursor: pointer;
}

.resource-manager button:hover {
  background: rgba(63, 132, 119, 0.14);
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

.manager-github,
.manager-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
}

.manager-github {
  margin-bottom: 18px;
}

.manager-layout {
  display: grid;
  grid-template-columns: 280px minmax(0, 1fr);
  gap: 18px;
  align-items: start;
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

.resource-manager input,
.resource-manager textarea {
  width: 100%;
  min-height: 38px;
  padding: 8px 10px;
  border: 1px solid rgba(96, 117, 138, 0.26);
  border-radius: 8px;
  background: var(--board-bg-color, #fff);
  color: var(--text-color, #333);
}

.manager-wide {
  margin: 12px 0;
}

.section-editor {
  padding: 14px;
  margin-top: 12px;
}

.section-editor-head {
  grid-template-columns: minmax(0, 1fr) auto auto auto auto;
}

.section-title {
  flex: 1;
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
}

.file-row {
  display: grid;
  grid-template-columns: minmax(120px, 1fr) minmax(160px, 1fr) minmax(160px, 1fr) 72px auto auto auto;
  gap: 8px;
  align-items: center;
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

  .manager-sidebar {
    position: static;
  }

  .file-row {
    grid-template-columns: 1fr;
  }

  .section-editor-head {
    flex-wrap: wrap;
    justify-content: flex-start;
  }
}
</style>
