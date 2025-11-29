// ==UserScript==
// @name         WhatsApp 翻译助手 v12·Fix9（翻译+快捷话术+T一键翻译+锁定弹窗）
// @namespace    https://chat.openai.com/
// @version      12.9
// @description  WhatsApp Web 自动翻译消息 & 输入框翻译（A/B）、切换聊天自动翻译、历史消息、新消息、GAS + OpenAI、多语言、T 按钮、Lexical 输入框修复 + 右中悬浮翻译图标一键翻译 + 翻译面板锁定 + 右下快捷话术（英文+中文+批量导入）
// @match        https://web.whatsapp.com/*
// @grant        GM_xmlhttpRequest
// @grant        GM_addStyle
// @connect      script.google.com
// @connect      script.googleusercontent.com
// @connect      api.openai.com
// @run-at       document-end
// ==/UserScript==

(function () {
  "use strict";

  console.log(
    "%c[WA Translate v12·Fix9 + QuickPhrases v3.3] loaded",
    "color:#00e676;font-weight:bold"
  );

  /************ 配置区（翻译模块） ************/
  const GAS_API_URL =
    "https://script.google.com/macros/s/AKfycbzn_MolHdZiMxN5fIezjJndbY7vbzUhFLJoNb5_FmpNee-2iHm9Rc8Abh1AhdB70_m-/exec";

  const OPENAI_API_KEY = ""; // ← 有需要就填
  const OPENAI_MODEL = "gpt-4o-mini";
  const OPENAI_URL = "https://api.openai.com/v1/chat/completions";

  const TRANSLATION_COLOR = "#3fa9f5";

  const ENGINE = {
    GAS: "gas",
    OPENAI: "openai",
  };

  /*********** 多语言 ***********/
  const LANGS = {
    English: { zh: "英语", iso: "en" },
    Spanish: { zh: "西班牙语", iso: "es" },
    Japanese: { zh: "日语", iso: "ja" },
    Korean: { zh: "韩语", iso: "ko" },
    French: { zh: "法语", iso: "fr" },
    German: { zh: "德语", iso: "de" },
    Arabic: { zh: "阿拉伯语", iso: "ar" },
    Portuguese: { zh: "葡萄牙语", iso: "pt" },
    Russian: { zh: "俄语", iso: "ru" },
    Italian: { zh: "意大利语", iso: "it" },
    Dutch: { zh: "荷兰语", iso: "nl" },
    Turkish: { zh: "土耳其语", iso: "tr" },
    Vietnamese: { zh: "越南语", iso: "vi" },
    Thai: { zh: "泰语", iso: "th" },
    Indonesian: { zh: "印尼语", iso: "id" },
    Chinese: { zh: "中文", iso: "zh-CN" },
  };

  /*********** 状态 / 本地存储 ***********/
  const STORAGE = {
    engine: "wa_v12_engine",
    lang: "wa_v12_partner_lang",
    auto: "wa_v12_auto",
    mode: "wa_v12_inputMode",
    cache: "wa_v12_cache",
  };

  let currentEngine = localStorage.getItem(STORAGE.engine) || ENGINE.GAS;
  let currentPartnerLang =
    localStorage.getItem(STORAGE.lang) || "English";
  let autoTranslate = localStorage.getItem(STORAGE.auto) === "1";
  let inputMode = localStorage.getItem(STORAGE.mode) || "A"; // A 覆盖 / B 原文+译文

  const translatedSet = new WeakSet(); // 已翻译消息
  let pmTimer = null; // 消息处理防抖

  // 翻译面板锁定状态（保持展开），默认 false（你选 A）
  let transPanelLock = false;

  /************ 工具函数（翻译模块） ************/
  function hasChinese(t) {
    return /[\u4e00-\u9fa5]/.test(t);
  }

  function getCache() {
    try {
      return JSON.parse(localStorage.getItem(STORAGE.cache) || "{}");
    } catch {
      return {};
    }
  }

  function setCache(obj) {
    try {
      const keys = Object.keys(obj);
      if (keys.length > 2000) {
        const del = keys.slice(0, keys.length - 2000);
        del.forEach((k) => delete obj[k]);
      }
      localStorage.setItem(STORAGE.cache, JSON.stringify(obj));
    } catch {}
  }

  function cacheGet(text, iso) {
    const c = getCache();
    return c[iso + "|" + text];
  }

  function cacheSet(text, iso, v) {
    const c = getCache();
    const k = iso + "|" + text;
    if (!c[k]) {
      c[k] = v;
      setCache(c);
    }
  }

  /*********** 翻译链路 ************/
  function translateViaGas(text, iso) {
    return new Promise((resolve, reject) => {
      GM_xmlhttpRequest({
        method: "GET",
        url:
          GAS_API_URL +
          "?text=" +
          encodeURIComponent(text) +
          "&target=" +
          encodeURIComponent(iso),
        onload: (res) => {
          const t = (res.responseText || "").trim();
          if (!t || t.startsWith("<")) reject("GAS 返回异常");
          else resolve(t);
        },
        onerror: () => reject("GAS 网络错误"),
      });
    });
  }

  function translateViaOpenAI(text, langName) {
    return new Promise((resolve, reject) => {
      if (!OPENAI_API_KEY) {
        reject("未配置 OpenAI Key");
        return;
      }
      GM_xmlhttpRequest({
        method: "POST",
        url: OPENAI_URL,
        headers: {
          "Content-Type": "application/json",
          Authorization: "Bearer " + OPENAI_API_KEY,
        },
        data: JSON.stringify({
          model: OPENAI_MODEL,
          messages: [
            {
              role: "system",
              content:
                "You are a professional translator. Translate the text into " +
                langName +
                " naturally and concisely. Do not add explanations.",
            },
            { role: "user", content: text },
          ],
        }),
        onload: (res) => {
          try {
            const j = JSON.parse(res.responseText);
            const t =
              j.choices?.[0]?.message?.content?.trim() || "";
            if (!t) reject("OpenAI 空结果");
            else resolve(t);
          } catch {
            reject("OpenAI 解析失败");
          }
        },
        onerror: () => reject("OpenAI 网络错误"),
      });
    });
  }

  function doTranslate(text, langName) {
    const cfg = LANGS[langName] || LANGS.English;
    const iso = cfg.iso;
    const cached = cacheGet(text, iso);
    if (cached) return Promise.resolve(cached);

    let p;
    if (currentEngine === ENGINE.OPENAI) {
      p = translateViaOpenAI(text, langName).catch(() =>
        translateViaGas(text, iso)
      );
    } else {
      p = translateViaGas(text, iso);
    }

    return p.then((t) => {
      cacheSet(text, iso, t);
      return t;
    });
  }

  /*********** 消息文本提取 ************/
  function getMessageText(msg) {
    const lex = msg.querySelector("span[__lexicalText]");
    if (lex && lex.textContent.trim()) return lex.textContent.trim();

    const legacy = msg.querySelector("span.selectable-text");
    if (legacy && legacy.innerText.trim()) return legacy.innerText.trim();

    const walker = document.createTreeWalker(
      msg,
      NodeFilter.SHOW_TEXT,
      {
        acceptNode(node) {
          return node.textContent.trim()
            ? NodeFilter.FILTER_ACCEPT
            : NodeFilter.REJECT;
        },
      }
    );
    let txt = "";
    while (walker.nextNode()) {
      txt += walker.currentNode.textContent;
    }
    return txt.trim();
  }

  /*********** 显示译文（彩色 + 原生风） ***********/
  function createTranslationBlock(msg, translated, langName) {
    const old = msg.querySelector(".wa-trans-block");
    if (old) old.remove();

    const cfg = LANGS[langName] || LANGS.English;

    const wrap = document.createElement("div");
    wrap.className = "wa-trans-block";
    wrap.innerHTML = `
      <div class="wa-trans-header">
        <span class="wa-trans-title-text">译文（${cfg.zh}）</span>
        <span class="wa-trans-actions">
          <span class="wa-copy">复制</span>
          <span class="wa-dot">·</span>
          <span class="wa-toggle">折叠</span>
        </span>
      </div>
      <div class="wa-trans-body"></div>
    `;

    const body = wrap.querySelector(".wa-trans-body");
    body.textContent = translated;
    body.style.color = TRANSLATION_COLOR;

    wrap.querySelector(".wa-copy").onclick = (e) => {
      e.stopPropagation();
      navigator.clipboard?.writeText(translated);
    };

    wrap.querySelector(".wa-toggle").onclick = (e) => {
      e.stopPropagation();
      const hidden = body.style.display === "none";
      body.style.display = hidden ? "" : "none";
      e.target.textContent = hidden ? "折叠" : "展开";
    };

    msg.appendChild(wrap);
  }

  /*********** 自动翻译 & T 按钮 ***********/
  function autoTranslateMessage(msg) {
    if (translatedSet.has(msg)) return;

    const txt = getMessageText(msg);
    if (!txt) return;

    const lang = hasChinese(txt) ? currentPartnerLang : "Chinese";

    translatedSet.add(msg);

    doTranslate(txt, lang)
      .then((t) => createTranslationBlock(msg, t, lang))
      .catch((e) =>
        createTranslationBlock(msg, "翻译失败：" + e, lang)
      );
  }

  function manualTranslateMessage(msg) {
    const txt = getMessageText(msg);
    if (!txt) return;

    const lang = hasChinese(txt) ? currentPartnerLang : "Chinese";

    createTranslationBlock(msg, "翻译中…", lang);

    doTranslate(txt, lang)
      .then((t) => createTranslationBlock(msg, t, lang))
      .catch((e) =>
        createTranslationBlock(msg, "翻译失败：" + e, lang)
      );
  }

  function attachTinyButton(msg) {
    if (msg.querySelector(".wa-mini-btn")) return;

    const btn = document.createElement("div");
    btn.className = "wa-mini-btn";
    btn.textContent = "T";

    let timer = null;

    btn.onclick = (e) => {
      e.stopPropagation();
      const body = msg.querySelector(".wa-trans-body");
      if (body) {
        body.style.display = body.style.display === "none" ? "" : "none";
      } else {
        manualTranslateMessage(msg);
      }
    };

    btn.onmousedown = () => {
      timer = setTimeout(() => manualTranslateMessage(msg), 600);
    };
    btn.onmouseup = btn.onmouseleave = () => {
      clearTimeout(timer);
      timer = null;
    };

    msg.style.position = "relative";
    msg.appendChild(btn);
  }

  /*********** 扫描消息 ***********/
  function processMessages() {
    const msgs = document.querySelectorAll("div.message-in, div.message-out");
    msgs.forEach((msg) => {
      attachTinyButton(msg);
      if (autoTranslate) {
        autoTranslateMessage(msg);
      }
    });
  }

  function processMessagesDebounced(delay = 120) {
    clearTimeout(pmTimer);
    pmTimer = setTimeout(processMessages, delay);
  }

  /*********** 输入框：查找 + Lexical 展开 + 读取 + 写入 ***********/
  function getInputEditable() {
    const main = document.querySelector("#main");
    if (!main) return null;

    const footer = main.querySelector("footer");
    if (!footer) return null;

    const lexical = footer.querySelector(
      "div[contenteditable='true'][data-lexical-editor='true']"
    );
    if (lexical) return lexical;

    const boxes = footer.querySelectorAll("div[contenteditable='true']");
    if (!boxes.length) return null;
    return boxes[boxes.length - 1];
  }

  function lexicalForceExpand(editable) {
    try {
      editable.focus();
      const sel = window.getSelection();
      const range = document.createRange();
      range.selectNodeContents(editable);
      range.collapse(false);
      sel.removeAllRanges();
      sel.addRange(range);
      editable.dispatchEvent(new InputEvent("input", { bubbles: true }));
    } catch (e) {
      console.warn("lexicalForceExpand error:", e);
    }
  }

  function getEditableTextForce(editable) {
    try {
      const walker = document.createTreeWalker(
        editable,
        NodeFilter.SHOW_TEXT,
        {
          acceptNode(node) {
            return node.textContent.trim()
              ? NodeFilter.FILTER_ACCEPT
              : NodeFilter.REJECT;
          },
        }
      );
      let txt = "";
      while (walker.nextNode()) {
        txt += walker.currentNode.textContent;
      }
      return txt.trim();
    } catch (e) {
      console.warn("getEditableTextForce error:", e);
      return (editable.textContent || editable.innerText || "").trim();
    }
  }

  function setEditableText(editable, text) {
    editable.focus();

    const sel = window.getSelection();
    const range = document.createRange();
    range.selectNodeContents(editable);
    sel.removeAllRanges();
    sel.addRange(range);

    const ok = document.execCommand("insertText", false, text);
    if (!ok) {
      editable.innerHTML = "";
      editable.appendChild(document.createTextNode(text));
    }

    editable.dispatchEvent(
      new InputEvent("input", { bubbles: true })
    );
  }

  // 可复用：一键翻译输入框（也给 T 按钮用）
  async function translateInputBox() {
    const editable = getInputEditable();
    if (!editable) {
      alert("找不到输入框，请先打开一个聊天窗口。");
      return;
    }

    lexicalForceExpand(editable);
    await new Promise((r) => requestAnimationFrame(r));

    const text = getEditableTextForce(editable);
    if (!text) {
      alert("输入框为空");
      return;
    }

    const lang = hasChinese(text) ? currentPartnerLang : "Chinese";

    const btn = document.querySelector(".wa-trans-btn");
    let old = "翻译输入内容";
    if (btn) {
      old = btn.textContent;
      btn.disabled = true;
      btn.textContent = "翻译中…";
    }

    try {
      const t = await doTranslate(text, lang);
      const finalText =
        inputMode === "A" ? t : text + "\n———\n" + t;

      setEditableText(editable, finalText);
    } catch (e) {
      alert("输入框翻译失败：" + e);
    } finally {
      if (btn) {
        btn.disabled = false;
        btn.textContent = old;
      }
      const sel = window.getSelection();
      sel.removeAllRanges();
    }
  }

  /*********** 控制面板（翻译助手） ***********/
  function createControlPanel() {
    if (document.querySelector(".wa-trans-panel")) return;

    const panel = document.createElement("div");
    panel.className = "wa-trans-panel";

    panel.innerHTML = `
      <div class="wa-panel-header">
        <span class="wa-panel-title">翻译助手</span>
        <div class="wa-panel-header-right">
          <button class="wa-panel-lock" title="保持展开">🔓</button>
          <button class="wa-panel-toggle" title="折叠/展开">⋮</button>
        </div>
      </div>

      <div class="wa-panel-body">
        <div class="wa-row">
          <div class="wa-label">引擎</div>
          <select class="wa-engine wa-input">
            <option value="gas">GAS 免费翻译</option>
            <option value="openai">OpenAI 高质量</option>
          </select>
        </div>

        <div class="wa-row">
          <div class="wa-label">对方语言</div>
          <select class="wa-lang wa-input"></select>
        </div>

        <div class="wa-row">
          <div class="wa-label">自动翻译</div>
          <label class="wa-switch">
            <input type="checkbox" class="wa-auto">
            <span class="wa-slider"></span>
          </label>
        </div>

        <div class="wa-row">
          <div class="wa-label">输入模式</div>
          <select class="wa-mode wa-input">
            <option value="A">A 覆盖</option>
            <option value="B">B 原文+译文</option>
          </select>
        </div>

        <div class="wa-row">
          <button class="wa-trans-btn">翻译输入内容</button>
        </div>
      </div>
    `;

    const langSel = panel.querySelector(".wa-lang");
    Object.keys(LANGS).forEach((k) => {
      if (k === "Chinese") return;
      const opt = document.createElement("option");
      opt.value = k;
      opt.textContent = LANGS[k].zh;
      langSel.appendChild(opt);
    });

    const engineSel = panel.querySelector(".wa-engine");
    if (!OPENAI_API_KEY && currentEngine === ENGINE.OPENAI) {
      currentEngine = ENGINE.GAS;
    }
    engineSel.value = currentEngine;
    engineSel.onchange = (e) => {
      currentEngine = e.target.value;
      localStorage.setItem(STORAGE.engine, currentEngine);
    };

    langSel.value = currentPartnerLang;
    langSel.onchange = (e) => {
      currentPartnerLang = e.target.value;
      localStorage.setItem(STORAGE.lang, currentPartnerLang);
    };

    const autoChk = panel.querySelector(".wa-auto");
    autoChk.checked = autoTranslate;
    autoChk.onchange = (e) => {
      autoTranslate = e.target.checked;
      localStorage.setItem(STORAGE.auto, autoTranslate ? "1" : "0");
      processMessages();
    };

    const modeSel = panel.querySelector(".wa-mode");
    modeSel.value = inputMode;
    modeSel.onchange = (e) => {
      inputMode = e.target.value;
      localStorage.setItem(STORAGE.mode, inputMode);
    };

    panel.querySelector(".wa-trans-btn").onclick = translateInputBox;

    panel.querySelector(".wa-panel-toggle").onclick = () => {
      panel.classList.toggle("wa-collapsed");
    };

    // 🔒 保持展开按钮
    const lockBtn = panel.querySelector(".wa-panel-lock");
    function refreshLockVisual() {
      if (transPanelLock) {
        lockBtn.textContent = "🔒";
        lockBtn.classList.add("wa-lock-on");
      } else {
        lockBtn.textContent = "🔓";
        lockBtn.classList.remove("wa-lock-on");
      }
    }
    refreshLockVisual();
    lockBtn.onclick = () => {
      transPanelLock = !transPanelLock;
      refreshLockVisual();
    };

    document.body.appendChild(panel);
  }

  /*********** 消息 DOM 监听 + 历史 / 新消息 ***********/
  function getMessagesContainer() {
    const main = document.querySelector("#main");
    if (!main) return null;

    return (
      main.querySelector('[data-testid="conversation-panel-messages"]') ||
      main.querySelector("[role='region'] div[tabindex]") ||
      main
    );
  }

  function startMessageObserver() {
    const main = document.querySelector("#main");
    if (!main) return;

    const obs = new MutationObserver(() =>
      processMessagesDebounced(80)
    );
    obs.observe(main, { childList: true, subtree: true });

    const msgContainer = getMessagesContainer();
    if (msgContainer) {
      msgContainer.addEventListener("scroll", () =>
        processMessagesDebounced(180)
      );
    }

    processMessages();
  }

  /*********** 聊天切换检测（URL + 点击） ***********/
  let lastURL = location.href;

  setInterval(() => {
    if (location.href !== lastURL) {
      lastURL = location.href;
      processMessagesDebounced(300);
    }
  }, 500);

  document.addEventListener("pointerup", (e) => {
    if (
      e.target.closest(
        "[aria-selected], div[data-testid='cell-frame-container']"
      )
    ) {
      processMessagesDebounced(300);
    }
  });

  /*********** UI 样式（翻译 + 翻译悬浮按钮） ***********/
  GM_addStyle(`
    .wa-mini-btn {
      position:absolute;
      top:2px;
      right:4px;
      padding:0 4px;
      font-size:10px;
      border-radius:999px;
      background:rgba(0,0,0,0.35);
      color:#fff;
      cursor:pointer;
      z-index:5;
    }
    .wa-mini-btn:hover { background:rgba(0,0,0,0.6); }

    .wa-trans-block {
      margin-top:4px;
      padding:4px 6px;
      border-radius:6px;
      background:rgba(255,255,255,0.9);
      font-size:12px;
      border-left:3px solid ${TRANSLATION_COLOR};
    }

    .wa-trans-header {
      display:flex;
      justify-content:space-between;
      align-items:center;
      color:#667781;
      font-size:11px;
      margin-bottom:2px;
    }
    .wa-trans-actions span { cursor:pointer; }
    .wa-trans-actions span:hover { text-decoration:underline; }

    .wa-trans-body {
      white-space:pre-wrap;
      margin-top:2px;
    }

    .wa-trans-panel {
      position:fixed;
      top:72px;
      right:16px;
      width:240px;
      background:#202c33;
      border-radius:12px;
      border:1px solid rgba(134,150,160,0.25);
      box-shadow:0 2px 8px rgba(0,0,0,0.3);
      color:#e9edef;
      font-size:12px;
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      z-index:99999;
      transform:translateX(120%);
      opacity:0;
      pointer-events:none;
      transition:transform .18s ease-out, opacity .18s ease-out;
    }
    .wa-trans-panel.wa-trans-open {
      transform:translateX(0%);
      opacity:1;
      pointer-events:auto;
    }
    .wa-trans-panel.wa-collapsed .wa-panel-body { display:none; }

    .wa-panel-header {
      display:flex;
      justify-content:space-between;
      align-items:center;
      padding:6px 8px;
      border-bottom:1px solid rgba(134,150,160,0.2);
    }
    .wa-panel-title {
      font-size:13px;
      font-weight:600;
    }
    .wa-panel-header-right {
      display:flex;
      gap:4px;
      align-items:center;
    }
    .wa-panel-toggle,
    .wa-panel-lock {
      background:transparent;
      border:none;
      color:#8696a0;
      cursor:pointer;
      font-size:14px;
      padding:0 4px;
    }
    .wa-panel-toggle:hover,
    .wa-panel-lock:hover { color:#e9edef; }

    .wa-panel-lock.wa-lock-on {
      color:#00a884;
    }

    .wa-panel-body { padding:8px; }

    .wa-row {
      display:flex;
      align-items:center;
      margin-top:6px;
      gap:6px;
    }
    .wa-label {
      width:64px;
      color:#8696a0;
      font-size:12px;
    }

    .wa-engine, .wa-lang, .wa-mode {
      flex:1;
      background:#111b21;
      border-radius:6px;
      border:1px solid rgba(134,150,160,0.4);
      padding:3px 6px;
      color:#e9edef;
      font-size:12px;
      outline:none;
    }
    .wa-engine:focus, .wa-lang:focus, .wa-mode:focus {
      border-color:#00a884;
    }

    .wa-trans-btn {
      flex:1;
      background:#00a884;
      border:none;
      color:#111b21;
      padding:5px 6px;
      border-radius:6px;
      font-size:12px;
      cursor:pointer;
      font-weight:500;
    }
    .wa-trans-btn:hover { filter:brightness(1.05); }

    .wa-switch {
      position:relative;
      display:inline-block;
      width:34px;
      height:18px;
    }
    .wa-switch input {
      opacity:0;
      width:0;
      height:0;
    }
    .wa-slider {
      position:absolute;
      cursor:pointer;
      top:0; left:0; right:0; bottom:0;
      background-color:#3b4a54;
      transition:.2s;
      border-radius:999px;
    }
    .wa-slider:before {
      position:absolute;
      content:"";
      height:14px;
      width:14px;
      left:2px;
      bottom:2px;
      background-color:#e9edef;
      transition:.2s;
      border-radius:50%;
    }
    .wa-switch input:checked + .wa-slider {
      background-color:#00a884;
    }
    .wa-switch input:checked + .wa-slider:before {
      transform:translateX(16px);
    }

    /* 右中悬浮翻译图标（圆形 T） - 永远显示 */
    #wa-trans-float-btn {
      position:fixed;
      right:16px;
      top:50%;
      transform:translateY(-50%);
      width:38px;
      height:38px;
      border-radius:50%;
      background:#00a884;
      box-shadow:0 2px 8px rgba(0,0,0,0.4);
      color:#111b21;
      display:flex;
      align-items:center;
      justify-content:center;
      cursor:pointer;
      z-index:99998;
      font-size:18px;
      font-weight:600;
    }
    #wa-trans-float-btn:hover {
      filter:brightness(1.05);
    }
  `);

  /*********** 翻译面板悬浮按钮逻辑（鼠标悬停 + 点击一键翻译） ***********/
  function setupTransFloatButton() {
    if (document.querySelector("#wa-trans-float-btn")) return;
    const panel = document.querySelector(".wa-trans-panel");
    if (!panel) return;

    const btn = document.createElement("div");
    btn.id = "wa-trans-float-btn";
    btn.textContent = "T";
    document.body.appendChild(btn);

    let transHoverTimer = null;
    let overBtn = false;
    let overPanel = false;

    function openPanel() {
      panel.classList.add("wa-trans-open");
    }
    function closePanel() {
      panel.classList.remove("wa-trans-open");
    }
    function cancelTimer() {
      if (transHoverTimer) {
        clearTimeout(transHoverTimer);
        transHoverTimer = null;
      }
    }
    function scheduleClose() {
      cancelTimer();
      transHoverTimer = setTimeout(() => {
        if (transPanelLock) return; // 🔒 开启时，不自动关闭
        if (!overBtn && !overPanel) {
          closePanel();
        }
      }, 350); // 稍微加长时间，避免你说的“还没点到就缩回去”
    }

    // 悬停：只展开/关闭（方便你只改设置）
    btn.addEventListener("mouseenter", () => {
      overBtn = true;
      cancelTimer();
      openPanel();
    });
    btn.addEventListener("mouseleave", () => {
      overBtn = false;
      scheduleClose();
    });

    panel.addEventListener("mouseenter", () => {
      overPanel = true;
      cancelTimer();
    });
    panel.addEventListener("mouseleave", () => {
      overPanel = false;
      scheduleClose();
    });

    // 点击：一键翻译输入框 + 展开面板（你选的 B）
    btn.addEventListener("click", (e) => {
      e.preventDefault();
      e.stopPropagation();
      translateInputBox(); // 自动把输入框中文翻译成英文/目标语言
      openPanel(); // 同时展开面板
    });
  }

  /*********** 快捷话术模块：双语两行 + 右下悬浮按钮 ***********/
  (function initQuickPhrases() {
    const QP_STORE_KEY = "WA_BI_QUICK_PHRASES_V32";

    const defaultData = {
      "问候": [
        { en: "Hello, how can I help you today?", zh: "你好，请问今天有什么可以帮到你？" },
        { en: "Nice to meet you.", zh: "很高兴认识你。" }
      ],
      "报价": [
        { en: "I will calculate the best price for you.", zh: "我会为你核算一个最优惠的价格。" },
        { en: "This is our latest quotation. Please check it.", zh: "这是我们的最新报价，请查收。" }
      ],
      "售后": [
        { en: "Don't worry, we will handle this for you.", zh: "请放心，我们会为你妥善处理。" },
        { en: "Please send me your order number.", zh: "请把你的订单号发给我，我来帮你查询。" }
      ],
      "常用语": [
        { en: "Okay, got it.", zh: "好的，明白了。" },
        { en: "Thank you for your trust.", zh: "感谢你的信任。" }
      ]
    };

    function loadQPData() {
      try {
        const raw = localStorage.getItem(QP_STORE_KEY);
        if (!raw) return structuredClone(defaultData);
        const obj = JSON.parse(raw);
        if (!obj || typeof obj !== "object") return structuredClone(defaultData);
        return obj;
      } catch {
        return structuredClone(defaultData);
      }
    }

    let QP_DATA = loadQPData();
    let qpCurrentCategory = Object.keys(QP_DATA)[0] || "问候";

    function saveQPData() {
      localStorage.setItem(QP_STORE_KEY, JSON.stringify(QP_DATA));
    }

    function qpInsertToWAInput(text) {
      const box =
        document.querySelector("footer div[contenteditable='true']") ||
        document.querySelector("div[contenteditable='true'][data-tab='10']");
      if (!box) {
        alert("未找到 WhatsApp 输入框，请先打开一个聊天窗口。");
        return;
      }

      box.focus();

      const sel = window.getSelection();
      const range = document.createRange();
      range.selectNodeContents(box);
      sel.removeAllRanges();
      sel.addRange(range);

      document.execCommand("delete", false, null);

      const ok = document.execCommand("insertText", false, text);
      if (!ok) {
        box.innerHTML = "";
        box.appendChild(document.createTextNode(text));
      }

      box.dispatchEvent(new InputEvent("input", { bubbles: true }));
    }

    const qpFloatBtn = document.createElement("div");
    qpFloatBtn.id = "wa-biqp-float-btn";
    qpFloatBtn.textContent = "💬";
    document.body.appendChild(qpFloatBtn);

    const qpPanel = document.createElement("div");
    qpPanel.id = "wa-biqp-panel";
    qpPanel.innerHTML = `
      <div class="wa-biqp-header">
        <span>快捷话术</span>
      </div>

      <div class="wa-biqp-section">
        <div class="wa-biqp-row">
          <select id="wa-biqp-category"></select>
          <button id="wa-biqp-add-cat" class="wa-biqp-btn-sm">新分类</button>
          <button id="wa-biqp-del-cat" class="wa-biqp-btn-sm wa-biqp-btn-danger">删类</button>
        </div>
      </div>

      <div class="wa-biqp-section">
        <div class="wa-biqp-row">
          <button id="wa-biqp-add-phrase" class="wa-biqp-btn-main" style="flex:1;">添加话术（两行）</button>
          <button id="wa-biqp-batch" class="wa-biqp-btn-sm" style="white-space:nowrap;">批量导入</button>
        </div>
      </div>

      <div class="wa-biqp-section wa-biqp-list-section">
        <div id="wa-biqp-list"></div>
      </div>
    `;
    document.body.appendChild(qpPanel);

    const qpBatchModal = document.createElement("div");
    qpBatchModal.id = "wa-biqp-batch-modal";
    qpBatchModal.innerHTML = `
      <div class="wa-biqp-batch-backdrop"></div>
      <div class="wa-biqp-batch-dialog">
        <div class="wa-biqp-batch-title">批量导入话术（当前分类）</div>
        <div class="wa-biqp-batch-tip">
          规则：每两行是一条话术<br>
          第1行英文（点击插入这一行）<br>
          第2行中文备注（仅显示，不插入）<br>
          空行会被自动忽略。
        </div>
        <textarea id="wa-biqp-batch-text"></textarea>
        <div class="wa-biqp-batch-actions">
          <button id="wa-biqp-batch-cancel" class="wa-biqp-btn-sm">取消</button>
          <button id="wa-biqp-batch-ok" class="wa-biqp-btn-main">导入</button>
        </div>
      </div>
    `;
    qpBatchModal.style.display = "none";
    document.body.appendChild(qpBatchModal);

    const qpStyle = document.createElement("style");
    qpStyle.textContent = `
      #wa-biqp-float-btn {
        position: fixed;
        right: 20px;
        bottom: 20px;
        width: 42px;
        height: 42px;
        border-radius: 50%;
        background: #00a884;
        box-shadow: 0 2px 8px rgba(0,0,0,0.4);
        color: #111b21;
        display:flex;
        align-items:center;
        justify-content:center;
        cursor:pointer;
        z-index:99998;
        font-size:20px;
      }
      #wa-biqp-float-btn:hover {
        filter: brightness(1.05);
      }

      #wa-biqp-panel {
        position: fixed;
        right: 20px;
        bottom: 70px;
        width: 300px;
        max-height: 65%;
        background: #202c33;
        color: #e9edef;
        border-radius: 10px;
        box-shadow: 0 2px 8px rgba(0,0,0,0.4);
        font-size: 12px;
        z-index: 99999;
        display: flex;
        flex-direction: column;
        transform: translateX(120%);
        transition: transform 0.18s ease-out;
        overflow: hidden;
      }
      #wa-biqp-panel.wa-biqp-open {
        transform: translateX(0%);
      }

      .wa-biqp-header {
        padding: 8px 10px;
        border-bottom: 1px solid #394047;
        font-size: 13px;
        font-weight: 600;
      }
      .wa-biqp-section {
        padding: 8px 10px;
        border-bottom: 1px solid #394047;
      }
      .wa-biqp-section:last-child {
        border-bottom: none;
        flex: 1;
        overflow-y: auto;
      }

      .wa-biqp-row {
        display:flex;
        gap:6px;
        align-items:center;
      }

      #wa-biqp-category {
        flex:1;
        padding:4px 6px;
        border-radius:6px;
        border:1px solid #394047;
        background:#111b21;
        color:#e9edef;
        outline:none;
      }

      .wa-biqp-btn-sm {
        border:none;
        padding:3px 6px;
        border-radius:6px;
        cursor:pointer;
        font-size:11px;
        background:#444f5a;
        color:#e9edef;
        white-space:nowrap;
      }
      .wa-biqp-btn-sm:hover {
        filter:brightness(1.1);
      }
      .wa-biqp-btn-danger {
        background:#b33;
      }

      .wa-biqp-btn-main {
        border:none;
        padding:6px;
        border-radius:6px;
        cursor:pointer;
        font-size:12px;
        background:#00a884;
        color:#111b21;
        font-weight:600;
      }
      .wa-biqp-btn-main:hover {
        filter:brightness(1.05);
      }

      #wa-biqp-list {
        display:flex;
        flex-direction:column;
        gap:6px;
      }

      .wa-biqp-item {
        background:#111b21;
        border-radius:6px;
        padding:6px;
        cursor:pointer;
        position:relative;
      }
      .wa-biqp-item:hover {
        background:#18252f;
      }

      .wa-biqp-en {
        color:#e9edef;
        font-size:13px;
        margin-bottom:2px;
        white-space:pre-wrap;
      }
      .wa-biqp-zh {
        color:#8696a0;
        font-size:11px;
        white-space:pre-wrap;
      }

      .wa-biqp-item-btns {
        position:absolute;
        top:6px;
        right:6px;
        display:flex;
        gap:4px;
      }
      .wa-biqp-item-btns button {
        border:none;
        font-size:10px;
        padding:1px 5px;
        border-radius:4px;
        cursor:pointer;
        background:#444f5a;
        color:#e9edef;
      }
      .wa-biqp-item-btns button:hover {
        filter:brightness(1.1);
      }
      .wa-biqp-item-btns .wa-biqp-del {
        background:#b33;
      }

      #wa-biqp-batch-modal {
        position: fixed;
        inset: 0;
        z-index: 100000;
      }
      .wa-biqp-batch-backdrop {
        position:absolute;
        inset:0;
        background:rgba(0,0,0,0.4);
      }
      .wa-biqp-batch-dialog {
        position:absolute;
        top:50%;
        left:50%;
        transform:translate(-50%, -50%);
        width:360px;
        max-width:90%;
        background:#202c33;
        color:#e9edef;
        border-radius:10px;
        box-shadow:0 2px 10px rgba(0,0,0,0.5);
        padding:10px;
        display:flex;
        flex-direction:column;
        gap:6px;
      }
      .wa-biqp-batch-title {
        font-size:14px;
        font-weight:600;
      }
      .wa-biqp-batch-tip {
        font-size:11px;
        color:#8696a0;
        line-height:1.4;
      }
      #wa-biqp-batch-text {
        margin-top:4px;
        width:100%;
        height:160px;
        resize:vertical;
        border-radius:6px;
        border:1px solid #394047;
        background:#111b21;
        color:#e9edef;
        padding:6px;
        outline:none;
        font-size:12px;
        font-family:inherit;
      }
      .wa-biqp-batch-actions {
        margin-top:6px;
        display:flex;
        justify-content:flex-end;
        gap:8px;
      }
    `;
    document.head.appendChild(qpStyle);

    let qpHoverTimer = null;
    let qpOverBtn = false;
    let qpOverPanel = false;

    function qpOpenPanel() {
      qpPanel.classList.add("wa-biqp-open");
    }
    function qpClosePanel() {
      qpPanel.classList.remove("wa-biqp-open");
    }
    function qpCancelTimer() {
      if (qpHoverTimer) {
        clearTimeout(qpHoverTimer);
        qpHoverTimer = null;
      }
    }
    function qpScheduleClose() {
      qpCancelTimer();
      qpHoverTimer = setTimeout(() => {
        if (!qpOverBtn && !qpOverPanel) {
          qpClosePanel();
        }
      }, 200);
    }

    qpFloatBtn.addEventListener("mouseenter", () => {
      qpOverBtn = true;
      qpCancelTimer();
      qpOpenPanel();
    });
    qpFloatBtn.addEventListener("mouseleave", () => {
      qpOverBtn = false;
      qpScheduleClose();
    });

    qpPanel.addEventListener("mouseenter", () => {
      qpOverPanel = true;
      qpCancelTimer();
    });
    qpPanel.addEventListener("mouseleave", () => {
      qpOverPanel = false;
      qpScheduleClose();
    });

    const qpCatSelect = qpPanel.querySelector("#wa-biqp-category");
    const qpBtnAddCat = qpPanel.querySelector("#wa-biqp-add-cat");
    const qpBtnDelCat = qpPanel.querySelector("#wa-biqp-del-cat");
    const qpBtnAddPhrase = qpPanel.querySelector("#wa-biqp-add-phrase");
    const qpBtnBatch = qpPanel.querySelector("#wa-biqp-batch");
    const qpListDiv = qpPanel.querySelector("#wa-biqp-list");

    function loadQPDataSafe() {
      return QP_DATA;
    }

    function qpRenderCategorySelect() {
      qpCatSelect.innerHTML = "";
      const keys = Object.keys(loadQPDataSafe());
      if (!keys.length) {
        qpCurrentCategory = "";
        return;
      }
      if (!QP_DATA[qpCurrentCategory]) {
        qpCurrentCategory = keys[0];
      }
      keys.forEach((cat) => {
        const opt = document.createElement("option");
        opt.value = cat;
        opt.textContent = cat;
        qpCatSelect.appendChild(opt);
      });
      qpCatSelect.value = qpCurrentCategory;
    }

    function qpEscapeHTML(str) {
      return String(str)
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;");
    }

    function qpRenderPhraseList() {
      qpListDiv.innerHTML = "";
      if (!qpCurrentCategory || !QP_DATA[qpCurrentCategory]) {
        qpListDiv.innerHTML = `<div style="color:#8696a0;">暂无分类，请先添加分类。</div>`;
        return;
      }

      const arr = QP_DATA[qpCurrentCategory];
      if (!arr || arr.length === 0) {
        qpListDiv.innerHTML = `<div style="color:#8696a0;">该分类暂无话术，点击上方「添加话术」或「批量导入」。</div>`;
        return;
      }

      arr.forEach((item, idx) => {
        const en = item.en || "";
        const zh = item.zh || "";

        const div = document.createElement("div");
        div.className = "wa-biqp-item";
        div.innerHTML = `
          <div class="wa-biqp-en">${qpEscapeHTML(en)}</div>
          <div class="wa-biqp-zh">${qpEscapeHTML(zh)}</div>
          <div class="wa-biqp-item-btns">
            <button class="wa-biqp-edit">编辑</button>
            <button class="wa-biqp-del">删</button>
          </div>
        `;

        div.addEventListener("click", (e) => {
          if (e.target.closest(".wa-biqp-item-btns")) return;
          const text = (QP_DATA[qpCurrentCategory][idx].en || "").trim();
          if (!text) return;
          qpInsertToWAInput(text);
        });

        div.querySelector(".wa-biqp-edit").addEventListener("click", (e) => {
          e.stopPropagation();
          const old = QP_DATA[qpCurrentCategory][idx];
          const newEn =
            prompt("编辑英文（插入输入框的内容）：", old.en || "") ??
            old.en;
          const newZh =
            prompt(
              "编辑中文备注（仅用于显示，不插入）：",
              old.zh || ""
            ) ?? old.zh;
          QP_DATA[qpCurrentCategory][idx] = {
            en: (newEn || "").trim(),
            zh: (newZh || "").trim(),
          };
          saveQPData();
          qpRenderPhraseList();
        });

        div.querySelector(".wa-biqp-del").addEventListener("click", (e) => {
          e.stopPropagation();
          if (!confirm("确定删除该话术？")) return;
          QP_DATA[qpCurrentCategory].splice(idx, 1);
          saveQPData();
          qpRenderPhraseList();
        });

        qpListDiv.appendChild(div);
      });
    }

    qpCatSelect.addEventListener("change", () => {
      qpCurrentCategory = qpCatSelect.value;
      qpRenderPhraseList();
    });

    qpBtnAddCat.addEventListener("click", () => {
      const name = prompt("输入新分类名称：");
      if (!name || !name.trim()) return;
      const n = name.trim();
      if (!QP_DATA[n]) {
        QP_DATA[n] = [];
        qpCurrentCategory = n;
        saveQPData();
        qpRenderCategorySelect();
        qpRenderPhraseList();
      }
    });

    qpBtnDelCat.addEventListener("click", () => {
      if (!qpCurrentCategory) return;
      if (
        !confirm(
          `确定删除分类「${qpCurrentCategory}」以及其中所有话术？`
        )
      )
        return;
      delete QP_DATA[qpCurrentCategory];
      const keys = Object.keys(QP_DATA);
      qpCurrentCategory = keys[0] || "";
      saveQPData();
      qpRenderCategorySelect();
      qpRenderPhraseList();
    });

    qpBtnAddPhrase.addEventListener("click", () => {
      if (!qpCurrentCategory) {
        alert("请先创建一个分类。");
        return;
      }
      const en = prompt(
        "请输入英文（点击话术时会插入这行到输入框）："
      );
      if (!en || !en.trim()) return;
      const zh =
        prompt(
          "请输入中文备注（用于在列表中展示，不会插入输入框）："
        ) || "";
      QP_DATA[qpCurrentCategory].push({
        en: en.trim(),
        zh: zh.trim(),
      });
      saveQPData();
      qpRenderPhraseList();
    });

    const qpBatchText = qpBatchModal.querySelector(
      "#wa-biqp-batch-text"
    );
    const qpBatchCancel = qpBatchModal.querySelector(
      "#wa-biqp-batch-cancel"
    );
    const qpBatchOK = qpBatchModal.querySelector("#wa-biqp-batch-ok");

    qpBtnBatch.addEventListener("click", () => {
      if (!qpCurrentCategory) {
        alert("请先创建一个分类。");
        return;
      }
      qpBatchText.value = "";
      qpBatchModal.style.display = "block";
    });

    qpBatchCancel.addEventListener("click", () => {
      qpBatchModal.style.display = "none";
    });
    qpBatchModal
      .querySelector(".wa-biqp-batch-backdrop")
      .addEventListener("click", () => {
        qpBatchModal.style.display = "none";
      });

    qpBatchOK.addEventListener("click", () => {
      const raw = qpBatchText.value || "";
      const lines = raw
        .split(/\r?\n/)
        .map((l) => l.trim())
        .filter((l) => l.length > 0);

      if (lines.length < 2) {
        alert("至少需要 2 行（英文 + 中文）才能形成一条话术。");
        return;
      }

      const added = [];
      for (let i = 0; i < lines.length; i += 2) {
        const en = lines[i] || "";
        const zh = lines[i + 1] || "";
        if (!en.trim() && !zh.trim()) continue;
        added.push({ en: en.trim(), zh: zh.trim() });
      }

      if (!added.length) {
        alert("未识别到有效话术，请检查格式。");
        return;
      }

      if (!QP_DATA[qpCurrentCategory]) QP_DATA[qpCurrentCategory] = [];
      QP_DATA[qpCurrentCategory] =
        QP_DATA[qpCurrentCategory].concat(added);
      saveQPData();
      qpRenderPhraseList();
      qpBatchModal.style.display = "none";
    });

    qpRenderCategorySelect();
    qpRenderPhraseList();
  })();

  /*********** 启动 ***********/
  (async function init() {
    let tries = 0;
    while (!document.querySelector("#main") && tries < 40) {
      await new Promise((r) => setTimeout(r, 200));
      tries++;
    }
    createControlPanel();
    startMessageObserver();
    setupTransFloatButton();
  })();
})();
