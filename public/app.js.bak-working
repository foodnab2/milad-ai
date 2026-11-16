const storageKey = "milad-ai-conversations-v1";
const elements = {
  sidebar: document.querySelector("#sidebar"),
  historyList: document.querySelector("#historyList"),
  welcome: document.querySelector("#welcome"),
  messages: document.querySelector("#messages"),
  composer: document.querySelector("#composer"),
  input: document.querySelector("#messageInput"),
  sendButton: document.querySelector("#sendButton"),
  attachButton: document.querySelector("#attachButton"),
  fileInput: document.querySelector("#fileInput"),
  attachmentPreview: document.querySelector("#attachmentPreview"),
  composerStatus: document.querySelector("#composerStatus"),
  voiceButton: document.querySelector("#voiceButton"),
  autoVoiceToggle: document.querySelector("#autoVoiceToggle"),
  statusDot: document.querySelector("#statusDot"),
  statusText: document.querySelector("#statusText"),
  title: document.querySelector("#conversationTitle"),
  sidebarScrim: document.querySelector("#sidebarScrim"),
  accessGate: document.querySelector("#accessGate"),
  accessMessage: document.querySelector("#accessMessage"),
  accessChoices: document.querySelector("#accessChoices"),
  accessFootnote: document.querySelector("#accessFootnote"),
  adultConsent: document.querySelector("#adultConsent"),
  professionalConsent: document.querySelector("#professionalConsent"),
  regionConsent: document.querySelector("#regionConsent"),
  accessContinue: document.querySelector("#accessContinue")
};

function loadConversations() {
  try {
    const saved = JSON.parse(localStorage.getItem(storageKey) || "[]");
    return Array.isArray(saved) ? saved : [];
  } catch {
    return [];
  }
}

let conversations = loadConversations();
let activeId = conversations[0]?.id || null;
let sending = false;
let chatAvailable = false;
let pendingFiles = [];
let voiceChoice = localStorage.getItem("milad-ai-voice-choice") === "male" ? "male" : "female";
let autoVoiceEnabled = localStorage.getItem("milad-ai-auto-voice") === "1";
let activeAudio = null;
let attachmentDatabasePromise;
const speechUrls = new Map();
let liveSocket = null;
let liveAudioContext = null;
let liveMicrophoneStream = null;
let liveMicrophoneSource = null;
let liveProcessor = null;
let liveMutedOutput = null;
let liveAudioSources = new Set();
let liveStarting = false;
let liveActive = false;
let liveAttempt = 0;
let liveSessionTimeout = null;
let liveFinalizeTimeout = null;
let liveUserTranscript = "";
let liveAssistantTranscript = "";
let liveNextAudioTime = 0;

const MAX_FILE_BYTES = 6 * 1024 * 1024;
const MAX_FILES_PER_MESSAGE = 6;
const MAX_MESSAGE_FILE_BYTES = 10 * 1024 * 1024;
const supportedMimeTypes = new Set([
  "image/png", "image/jpeg", "image/webp", "image/heic", "image/heif",
  "audio/webm", "audio/mp4", "audio/m4a", "audio/ogg", "audio/mpeg", "audio/mp3", "audio/wav", "audio/aac", "audio/opus",
  "application/pdf", "text/plain", "text/markdown", "text/csv",
  "application/json", "text/html", "application/xml", "text/xml",
  "text/yaml", "application/yaml",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  "application/vnd.openxmlformats-officedocument.presentationml.presentation"
]);

function openAttachmentDatabase() {
  if (attachmentDatabasePromise) return attachmentDatabasePromise;
  attachmentDatabasePromise = new Promise((resolve, reject) => {
    const request = indexedDB.open("milad-ai-local-attachments", 1);
    request.onupgradeneeded = () => request.result.createObjectStore("files");
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(new Error("ذخیرهٔ امن فایل در این مرورگر در دسترس نیست."));
  });
  return attachmentDatabasePromise;
}

async function storeAttachment(id, file) {
  const database = await openAttachmentDatabase();
  return new Promise((resolve, reject) => {
    const transaction = database.transaction("files", "readwrite");
    transaction.objectStore("files").put(file, id);
    transaction.oncomplete = resolve;
    transaction.onerror = () => reject(new Error("ذخیرهٔ فایل در مرورگر انجام نشد."));
  });
}

async function readStoredAttachment(id) {
  const database = await openAttachmentDatabase();
  return new Promise((resolve, reject) => {
    const request = database.transaction("files", "readonly").objectStore("files").get(id);
    request.onsuccess = () => resolve(request.result || null);
    request.onerror = () => reject(new Error("خواندن فایل ذخیره‌شده انجام نشد."));
  });
}

async function removeStoredAttachment(id) {
  if (!id || !window.indexedDB) return;
  try {
    const database = await openAttachmentDatabase();
    await new Promise(resolve => {
      const transaction = database.transaction("files", "readwrite");
      transaction.objectStore("files").delete(id);
      transaction.oncomplete = resolve;
      transaction.onerror = resolve;
    });
  } catch {
    return;
  }
}

function allConversationAttachmentIds(conversation) {
  return (conversation?.messages || []).flatMap(message => (message.attachments || []).map(file => file.id));
}

function bytesToBase64(bytes) {
  let binary = "";
  const chunkSize = 0x8000;
  for (let offset = 0; offset < bytes.length; offset += chunkSize) {
    binary += String.fromCharCode(...bytes.subarray(offset, offset + chunkSize));
  }
  return btoa(binary);
}

async function blobToBase64(blob) {
  return bytesToBase64(new Uint8Array(await blob.arrayBuffer()));
}

function clearComposerStatus() {
  elements.composerStatus.hidden = true;
  elements.composerStatus.textContent = "";
  elements.composerStatus.classList.remove("error");
}

function showComposerStatus(message, isError = true) {
  elements.composerStatus.textContent = message;
  elements.composerStatus.hidden = !message;
  elements.composerStatus.classList.toggle("error", isError);
}

function activeConversation() {
  return conversations.find(item => item.id === activeId) || null;
}

function saveConversations() {
  try {
    localStorage.setItem(storageKey, JSON.stringify(conversations.slice(0, 40)));
  } catch {
    setStatus("فضای ذخیره‌سازی مرورگر پر شده است.", false);
  }
}

function setStatus(message, ready) {
  elements.statusText.textContent = message;
  elements.statusDot.className = "status-dot" + (ready === true ? " ready" : ready === false ? " waiting" : "");
}

function renderHistory() {
  elements.historyList.replaceChildren();
  for (const conversation of conversations) {
    const row = document.createElement("div");
    row.className = "history-row";

    const button = document.createElement("button");
    button.type = "button";
    button.className = "history-item" + (conversation.id === activeId ? " active" : "");
    button.textContent = conversation.title || "گفتگوی جدید";
    button.title = button.textContent;
    button.addEventListener("click", () => {
      activeId = conversation.id;
      render();
      closeSidebar();
    });

    const deleteButton = document.createElement("button");
    deleteButton.type = "button";
    deleteButton.className = "history-delete";
    deleteButton.setAttribute("aria-label", `حذف گفت‌وگو: ${button.textContent}`);
    deleteButton.title = "حذف گفت‌وگو";
    deleteButton.innerHTML = '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M4 7h16M10 11v6m4-6v6M6 7l1 13h10l1-13M9 7V4h6v3"/></svg>';
    deleteButton.addEventListener("click", event => {
      event.stopPropagation();
      if (!window.confirm(`گفت‌وگوی «${button.textContent}» و پیام‌هایش حذف شود؟`)) return;
      deleteConversation(conversation.id);
    });

    row.append(button, deleteButton);
    elements.historyList.append(row);
  }
}

function deleteConversation(conversationId) {
  const index = conversations.findIndex(conversation => conversation.id === conversationId);
  if (index < 0) return;
  allConversationAttachmentIds(conversations[index]).forEach(removeStoredAttachment);
  conversations = conversations.filter(conversation => conversation.id !== conversationId);
  if (activeId === conversationId) activeId = conversations[0]?.id || null;
  saveConversations();
  render();
}

function escapeHtml(value) {
  return value.replace(/[&<>"']/g, character => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;"
  })[character]);
}

function assistantMarkdown(value) {
  const codeBlocks = [];
  let html = escapeHtml(value).replace(/\x60\x60\x60(?:[a-z0-9_-]+)?\r?\n([\s\S]*?)\x60\x60\x60/gi, (_, code) => {
    const marker = "MILADCODEBLOCK" + codeBlocks.length + "END";
    codeBlocks.push("<pre><code>" + code + "</code></pre>");
    return marker;
  });
  html = html
    .replace(/\x60([^\x60\n]+)\x60/g, "<code>$1</code>")
    .replace(/\*\*([^*\n]+)\*\*/g, "<strong>$1</strong>")
    .replace(/(^|[\s(])\*([^*\n]+)\*(?=$|[\s).,!?:;])/g, "$1<em>$2</em>")
    .replace(/\r?\n/g, "<br>");
  return html.replace(/MILADCODEBLOCK(\d+)END/g, (_, index) => codeBlocks[Number(index)] || "");
}

function renderMessages() {
  const conversation = activeConversation();
  const messages = conversation?.messages || [];
  elements.welcome.hidden = messages.length > 0;
  elements.messages.hidden = messages.length === 0;
  elements.messages.replaceChildren();

  for (const [messageIndex, message] of messages.entries()) {
    const row = document.createElement("article");
    row.className = "message-row " + message.role + (message.error ? " error" : "");
    row.dataset.messageKey = message.id || `${conversation.id}-${messageIndex}`;

    const avatar = document.createElement("div");
    avatar.className = "message-avatar";
    avatar.setAttribute("aria-hidden", "true");
    avatar.textContent = message.role === "user" ? "●" : "✦";

    const content = document.createElement("div");
    content.className = "message-content";

    const name = document.createElement("div");
    name.className = "message-name";
    name.textContent = message.role === "user" ? "تو" : "MILAD AI";

    const body = document.createElement("div");
    body.className = "message-body";
    if (message.role === "assistant" && !message.error) body.innerHTML = assistantMarkdown(message.content);
    else body.textContent = message.content;
    content.append(name, body);

    if (message.attachments?.length) {
      const attachmentList = document.createElement("div");
      attachmentList.className = "message-attachments";
      for (const file of message.attachments) {
        const chip = document.createElement("div");
        chip.className = "message-attachment";
        const icon = document.createElement("span");
        icon.className = "file-type-icon";
        icon.textContent = fileTypeLabel(file.mimeType);
        const label = document.createElement("span");
        label.className = "file-name";
        label.textContent = file.name;
        label.title = file.name;
        const size = document.createElement("small");
        size.textContent = formatFileSize(file.size);
        chip.append(icon, label, size);
        attachmentList.append(chip);
      }
      content.append(attachmentList);
    }

    if (message.role === "assistant" && message.content && !message.streaming && !message.error) {
      const actions = document.createElement("div");
      actions.className = "message-actions";
      const speakButton = document.createElement("button");
      speakButton.type = "button";
      speakButton.className = "speak-button";
      speakButton.innerHTML = '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M11 5 6 9H3v6h3l5 4zM15.5 8.5a5 5 0 0 1 0 7m3-10a9 9 0 0 1 0 13"/></svg><span>شنیدن پاسخ</span>';
      speakButton.addEventListener("click", () => playMessageSpeech(message, row, speakButton, audio));
      const audio = document.createElement("audio");
      audio.className = "message-audio";
      audio.controls = true;
      audio.preload = "none";
      audio.hidden = true;
      audio.addEventListener("play", () => {
        if (activeAudio && activeAudio !== audio) activeAudio.pause();
        activeAudio = audio;
      });
      actions.append(speakButton, audio);
      content.append(actions);
    }

    if (message.streaming && !message.content) {
      const typing = document.createElement("div");
      typing.className = "typing";
      typing.setAttribute("aria-label", "در حال نوشتن");
      for (let index = 0; index < 3; index += 1) typing.append(document.createElement("i"));
      content.append(typing);
    }

    row.append(avatar, content);
    elements.messages.append(row);
  }

  elements.messages.scrollTop = elements.messages.scrollHeight;
}

function render() {
  const conversation = activeConversation();
  elements.title.textContent = conversation?.title || "گفتگوی جدید";
  renderHistory();
  renderMessages();
}

function createConversation() {
  const conversation = {
    id: crypto.randomUUID(),
    title: "گفتگوی جدید",
    messages: [],
    updatedAt: Date.now()
  };
  conversations.unshift(conversation);
  activeId = conversation.id;
  saveConversations();
  render();
  return conversation;
}

function closeSidebar() {
  document.body.classList.remove("sidebar-open");
}

function resizeInput() {
  elements.input.style.height = "auto";
  elements.input.style.height = Math.min(elements.input.scrollHeight, 170) + "px";
}

function readEvent(eventText, assistantMessage, conversation) {
  const data = eventText
    .split(/\r?\n/)
    .filter(line => line.startsWith("data:"))
    .map(line => line.slice(5).trim())
    .join("\n");
  if (!data || data === "[DONE]") return;
  try {
    const payload = JSON.parse(data);
    if (typeof payload.content === "string") {
      assistantMessage.content += payload.content;
      conversation.updatedAt = Date.now();
      saveConversations();
      renderMessages();
    }
  } catch {
    return;
  }
}

function formatFileSize(bytes) {
  if (!Number.isFinite(bytes) || bytes <= 0) return "فایل";
  return bytes < 1024 * 1024
    ? `${Math.max(1, Math.round(bytes / 1024))} کیلوبایت`
    : `${(bytes / (1024 * 1024)).toLocaleString("fa-IR", { maximumFractionDigits: 1 })} مگابایت`;
}

function fileTypeLabel(mimeType) {
  if (mimeType === "application/pdf") return "PDF";
  if (mimeType?.startsWith("image/")) return "▧";
  if (mimeType?.startsWith("audio/")) return "ویس";
  if (mimeType?.includes("wordprocessingml")) return "DOCX";
  if (mimeType?.includes("spreadsheetml")) return "XLSX";
  if (mimeType?.includes("presentationml")) return "PPTX";
  return "TXT";
}

function fileMimeType(file) {
  const reportedType = file.type ? file.type.toLowerCase().split(";")[0] : "";
  if (reportedType && supportedMimeTypes.has(reportedType)) return reportedType;
  const extension = file.name.split(".").pop().toLowerCase();
  const mimeByExtension = {
    png: "image/png", jpg: "image/jpeg", jpeg: "image/jpeg", webp: "image/webp",
    heic: "image/heic", heif: "image/heif", pdf: "application/pdf",
    webm: "audio/webm", m4a: "audio/m4a", mp4: "audio/mp4", ogg: "audio/ogg",
    mp3: "audio/mpeg", wav: "audio/wav", aac: "audio/aac", opus: "audio/opus",
    txt: "text/plain", md: "text/markdown", csv: "text/csv", json: "application/json",
    html: "text/html", htm: "text/html", xml: "application/xml", log: "text/plain",
    yaml: "text/yaml", yml: "text/yaml",
    docx: "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    xlsx: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    pptx: "application/vnd.openxmlformats-officedocument.presentationml.presentation"
  };
  return mimeByExtension[extension] || "";
}

function renderPendingFiles() {
  elements.attachmentPreview.replaceChildren();
  elements.attachmentPreview.hidden = pendingFiles.length === 0;
  for (const pending of pendingFiles) {
    const card = document.createElement("div");
    card.className = "pending-file";
    if (pending.mimeType.startsWith("audio/") && pending.previewUrl) {
      card.classList.add("pending-audio");
      const audio = document.createElement("audio");
      audio.className = "pending-audio-player";
      audio.controls = true;
      audio.preload = "metadata";
      audio.src = pending.previewUrl;
      audio.setAttribute("aria-label", `پخش ${pending.name}`);
      card.append(audio);
    } else if (pending.previewUrl) {
      const image = document.createElement("img");
      image.src = pending.previewUrl;
      image.alt = `پیش‌نمایش ${pending.name}`;
      card.append(image);
    } else {
      const icon = document.createElement("span");
      icon.className = "pending-file-icon";
      icon.textContent = fileTypeLabel(pending.mimeType);
      card.append(icon);
    }
    const details = document.createElement("div");
    details.className = "pending-file-details";
    const name = document.createElement("span");
    name.className = "file-name";
    name.textContent = pending.name;
    name.title = pending.name;
    const size = document.createElement("small");
    size.textContent = formatFileSize(pending.size);
    details.append(name, size);
    if (pending.mimeType.startsWith("audio/")) {
      const transcribeButton = document.createElement("button");
      transcribeButton.type = "button";
      transcribeButton.className = "pending-audio-transcribe";
      transcribeButton.textContent = "تبدیل گفتار به متن";
      transcribeButton.addEventListener("click", () => transcribeRecording(pending.file, transcribeButton));
      details.append(transcribeButton);
    }
    const remove = document.createElement("button");
    remove.type = "button";
    remove.className = "remove-pending-file";
    remove.setAttribute("aria-label", `حذف پیوست ${pending.name}`);
    remove.textContent = "×";
    remove.addEventListener("click", () => {
      if (pending.previewUrl) URL.revokeObjectURL(pending.previewUrl);
      pendingFiles = pendingFiles.filter(file => file.id !== pending.id);
      renderPendingFiles();
      clearComposerStatus();
    });
    card.append(details, remove);
    elements.attachmentPreview.append(card);
  }
}

function queueFiles(files) {
  clearComposerStatus();
  const accepted = [];
  for (const file of files) {
    const mimeType = fileMimeType(file);
    if (!mimeType) {
      showComposerStatus(`فرمت «${file.name}» پشتیبانی نمی‌شود. عکس، صدا، PDF یا فایل متنی انتخاب کن.`);
      continue;
    }
    if (!file.size || file.size > MAX_FILE_BYTES) {
      showComposerStatus(`حجم هر فایل باید حداکثر ۶ مگابایت باشد: ${file.name}`);
      continue;
    }
    if (pendingFiles.length + accepted.length >= MAX_FILES_PER_MESSAGE) {
      showComposerStatus("در هر پیام حداکثر ۶ فایل می‌توانی پیوست کنی.");
      break;
    }
    const currentSize = pendingFiles.reduce((sum, item) => sum + item.size, 0)
      + accepted.reduce((sum, item) => sum + item.size, 0);
    if (currentSize + file.size > MAX_MESSAGE_FILE_BYTES) {
      showComposerStatus("حجم پیوست‌های این پیام حداکثر ۱۰ مگابایت است.");
      break;
    }
    accepted.push({
      id: crypto.randomUUID(),
      file,
      name: file.name.slice(0, 180),
      mimeType,
      size: file.size,
      previewUrl: mimeType.startsWith("image/") || mimeType.startsWith("audio/") ? URL.createObjectURL(file) : ""
    });
  }
  pendingFiles.push(...accepted);
  renderPendingFiles();
  elements.fileInput.value = "";
}

function updateVoiceControls() {
  document.querySelectorAll(".voice-choice").forEach(button => {
    const selected = button.dataset.voice === voiceChoice;
    button.classList.toggle("active", selected);
    button.setAttribute("aria-pressed", String(selected));
  });
  elements.autoVoiceToggle.classList.toggle("active", autoVoiceEnabled);
  elements.autoVoiceToggle.setAttribute("aria-pressed", String(autoVoiceEnabled));
  elements.autoVoiceToggle.textContent = autoVoiceEnabled ? "پاسخ صوتی روشن" : "پاسخ صوتی خاموش";
}

function setLiveUi(active, starting = false) {
  elements.voiceButton.classList.toggle("live-active", active);
  elements.voiceButton.classList.toggle("recording", active);
  elements.voiceButton.disabled = starting || sending;
  elements.voiceButton.setAttribute("aria-pressed", String(active || starting));
  elements.voiceButton.setAttribute("aria-label", active ? "پایان مکالمهٔ زنده" : "شروع مکالمهٔ زنده");
  elements.voiceButton.title = active ? "برای پایان مکالمه لمس کن" : "شروع مکالمهٔ زندهٔ فارسی";
}

async function transcribeRecording(blob, actionButton = elements.voiceButton) {
  if (!blob.size || blob.size > 5 * 1024 * 1024) {
    showComposerStatus("ضبط بیش از حد بزرگ شد. لطفاً کوتاه‌تر بگو و دوباره ضبط کن.");
    return false;
  }
  const previousDisabled = actionButton.disabled;
  actionButton.disabled = true;
  showComposerStatus("در حال تبدیل گفتار فارسی به متن…", false);
  try {
    const mimeType = (blob.type || "audio/webm").toLowerCase().split(";")[0];
    const response = await fetch("/api/transcribe", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ mimeType, data: await blobToBase64(blob) })
    });
    const result = await response.json().catch(() => ({}));
    if (!response.ok) throw new Error(result.error || "تبدیل گفتار انجام نشد.");
    const transcript = (result.transcript || "").trim();
    if (!transcript) throw new Error("گفتاری تشخیص داده نشد؛ دوباره امتحان کن.");
    elements.input.value = [elements.input.value.trim(), transcript].filter(Boolean).join(" ");
    resizeInput();
    elements.input.focus();
    autoVoiceEnabled = true;
    localStorage.setItem("milad-ai-auto-voice", "1");
    updateVoiceControls();
    showComposerStatus(pendingFiles.some(file => file.mimeType.startsWith("audio/"))
      ? "متن آماده شد و ویس هم پیوست است؛ برای فرستادن فقط متن، ویس را با × حذف کن."
      : "متن تشخیص‌داده‌شده را مرور کن؛ با دکمهٔ ارسال بفرست.", false);
    return true;
  } catch (error) {
    showComposerStatus(error.message || "تبدیل گفتار انجام نشد.");
    return false;
  } finally {
    actionButton.disabled = previousDisabled;
  }
}

function mergeLiveTranscript(current, addition) {
  const next = String(addition || "").trim();
  if (!next || current === next || current.endsWith(next)) return current;
  if (!current || next.startsWith(current)) return next;
  return `${current} ${next}`.trim();
}

function finishLiveTurn() {
  clearTimeout(liveFinalizeTimeout);
  liveFinalizeTimeout = null;
  const userText = liveUserTranscript.trim();
  const assistantText = liveAssistantTranscript.trim();
  liveUserTranscript = "";
  liveAssistantTranscript = "";
  if (!userText && !assistantText) return;

  const conversation = activeConversation() || createConversation();
  if (userText) conversation.messages.push({ id: crypto.randomUUID(), role: "user", content: userText });
  if (assistantText) conversation.messages.push({ id: crypto.randomUUID(), role: "assistant", content: assistantText });
  if (conversation.title === "گفتگوی جدید" && userText) conversation.title = userText.slice(0, 36);
  conversation.updatedAt = Date.now();
  saveConversations();
  render();
}

function stopLivePlayback() {
  for (const source of liveAudioSources) {
    try { source.stop(); } catch { /* already ended */ }
  }
  liveAudioSources.clear();
  liveNextAudioTime = liveAudioContext?.currentTime || 0;
}

function closeLiveResources(closeSocket = true) {
  clearTimeout(liveSessionTimeout);
  clearTimeout(liveFinalizeTimeout);
  liveSessionTimeout = null;
  liveFinalizeTimeout = null;
  const socket = liveSocket;
  liveSocket = null;
  if (closeSocket && socket && socket.readyState < WebSocket.CLOSING) {
    try {
      if (socket.readyState === WebSocket.OPEN) socket.send(JSON.stringify({ realtimeInput: { audioStreamEnd: true } }));
      socket.close(1000, "Session ended");
    } catch { /* the connection may already be closed */ }
  }
  stopLivePlayback();
  if (liveProcessor) {
    liveProcessor.onaudioprocess = null;
    try { liveProcessor.disconnect(); } catch { /* already disconnected */ }
  }
  try { liveMicrophoneSource?.disconnect(); } catch { /* already disconnected */ }
  try { liveMutedOutput?.disconnect(); } catch { /* already disconnected */ }
  liveProcessor = null;
  liveMicrophoneSource = null;
  liveMutedOutput = null;
  liveMicrophoneStream?.getTracks().forEach(track => track.stop());
  liveMicrophoneStream = null;
  const context = liveAudioContext;
  liveAudioContext = null;
  if (context && context.state !== "closed") context.close().catch(() => {});
}

function pcm16Base64(samples, inputRate) {
  const ratio = inputRate / 16000;
  const outputLength = Math.floor(samples.length / ratio);
  const bytes = new Uint8Array(outputLength * 2);
  const view = new DataView(bytes.buffer);
  for (let index = 0; index < outputLength; index += 1) {
    const start = Math.floor(index * ratio);
    const end = Math.max(start + 1, Math.floor((index + 1) * ratio));
    let total = 0;
    for (let sourceIndex = start; sourceIndex < end && sourceIndex < samples.length; sourceIndex += 1) {
      total += samples[sourceIndex];
    }
    const sample = Math.max(-1, Math.min(1, total / Math.max(1, end - start)));
    view.setInt16(index * 2, sample < 0 ? sample * 0x8000 : sample * 0x7fff, true);
  }
  return bytesToBase64(bytes);
}

function playLiveAudio(part) {
  const inline = part.inlineData || part.inline_data;
  if (!inline?.data || !liveAudioContext) return;
  try {
    const binary = atob(inline.data);
    const sampleCount = Math.floor(binary.length / 2);
    if (!sampleCount) return;
    const bytes = new Uint8Array(sampleCount * 2);
    for (let index = 0; index < bytes.length; index += 1) bytes[index] = binary.charCodeAt(index);
    const view = new DataView(bytes.buffer);
    const rate = Number(inline.mimeType?.match(/rate=(\d+)/i)?.[1]) || 24000;
    const audioBuffer = liveAudioContext.createBuffer(1, sampleCount, rate);
    const channel = audioBuffer.getChannelData(0);
    for (let index = 0; index < sampleCount; index += 1) channel[index] = view.getInt16(index * 2, true) / 32768;
    const source = liveAudioContext.createBufferSource();
    source.buffer = audioBuffer;
    source.connect(liveAudioContext.destination);
    liveAudioSources.add(source);
    source.addEventListener("ended", () => liveAudioSources.delete(source), { once: true });
    const startAt = Math.max(liveNextAudioTime, liveAudioContext.currentTime + 0.025);
    source.start(startAt);
    liveNextAudioTime = startAt + audioBuffer.duration;
  } catch {
    showComposerStatus("پخش صدای مکالمهٔ زنده ممکن نشد.");
  }
}

function handleLiveMessage(payload) {
  const serverContent = payload.serverContent;
  if (!serverContent) return;
  if (serverContent.interrupted) stopLivePlayback();

  const inputText = serverContent.inputTranscription?.text;
  const interimText = serverContent.interimInputTranscription?.text;
  const outputText = serverContent.outputTranscription?.text;
  if (inputText) {
    clearTimeout(liveFinalizeTimeout);
    liveUserTranscript = mergeLiveTranscript(liveUserTranscript, inputText);
    showComposerStatus("صدایت را می‌شنوم؛ پاسخ مستقیم پخش می‌شود.", false);
  } else if (interimText) {
    showComposerStatus("دارم گوش می‌دهم…", false);
  }
  if (outputText) {
    clearTimeout(liveFinalizeTimeout);
    liveAssistantTranscript = mergeLiveTranscript(liveAssistantTranscript, outputText);
  }

  for (const part of serverContent.modelTurn?.parts || []) playLiveAudio(part);
  if (serverContent.turnComplete) {
    liveFinalizeTimeout = setTimeout(() => {
      finishLiveTurn();
      if (liveActive) showComposerStatus("مکالمه زنده است؛ فارسی صحبت کن. برای پایان، میکروفن را بزن.", false);
    }, 450);
  }
}

function attachLiveMicrophone() {
  if (!liveAudioContext || !liveMicrophoneStream || !liveSocket) throw new Error("میکروفن یا اتصال مکالمه آماده نشد.");
  liveMicrophoneSource = liveAudioContext.createMediaStreamSource(liveMicrophoneStream);
  liveProcessor = liveAudioContext.createScriptProcessor(4096, 1, 1);
  liveMutedOutput = liveAudioContext.createGain();
  liveMutedOutput.gain.value = 0;
  liveProcessor.onaudioprocess = event => {
    if (!liveActive || liveSocket?.readyState !== WebSocket.OPEN) return;
    const data = pcm16Base64(event.inputBuffer.getChannelData(0), liveAudioContext.sampleRate);
    if (data) liveSocket.send(JSON.stringify({ realtimeInput: { audio: { data, mimeType: "audio/pcm;rate=16000" } } }));
  };
  liveMicrophoneSource.connect(liveProcessor);
  liveProcessor.connect(liveMutedOutput);
  liveMutedOutput.connect(liveAudioContext.destination);
}

async function startLiveConversation() {
  if (!chatAvailable || sending || liveStarting || liveActive) return;
  const AudioContextClass = window.AudioContext || window.webkitAudioContext;
  if (!navigator.mediaDevices?.getUserMedia || !AudioContextClass || typeof WebSocket === "undefined") {
    showComposerStatus("مکالمهٔ زنده در این مرورگر در دسترس نیست. مرورگر را به‌روز کن یا ویس را به‌صورت فایل پیوست کن.");
    return;
  }

  const attempt = ++liveAttempt;
  liveStarting = true;
  setLiveUi(false, true);
  showComposerStatus("برای مکالمهٔ زنده، صدای میکروفن به Gemini فرستاده می‌شود؛ در حال اتصال…", false);
  try {
    const microphonePromise = navigator.mediaDevices.getUserMedia({
      audio: { channelCount: 1, echoCancellation: true, noiseSuppression: true, autoGainControl: true }
    }).then(stream => (liveMicrophoneStream = stream));
    liveAudioContext = new AudioContextClass();
    await Promise.all([liveAudioContext.resume(), microphonePromise]);
    if (attempt !== liveAttempt) return;

    const tokenResponse = await fetch("/api/live-token", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ voice: voiceChoice })
    });
    const session = await tokenResponse.json().catch(() => ({}));
    if (!tokenResponse.ok || !session.token || !session.model || !session.config) {
      throw new Error(session.error || "دریافت دسترسی امن مکالمه ممکن نشد.");
    }
    if (attempt !== liveAttempt) return;

    liveUserTranscript = "";
    liveAssistantTranscript = "";
    const socketUrl = "wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1alpha.GenerativeService.BidiGenerateContentConstrained?access_token="
      + encodeURIComponent(session.token);
    const socket = new WebSocket(`${location.protocol === "https:" ? "wss:" : "ws:"}//${location.host}/api/live-ws`);
    liveSocket = socket;

    await new Promise((resolve, reject) => {
      let setupFinished = false;
      const timeout = setTimeout(() => {
        if (setupFinished) return;
        setupFinished = true;
        reject(new Error("پاسخی برای شروع مکالمه از Gemini نرسید. دوباره تلاش کن."));
      }, 20000);
      const failSetup = error => {
        if (setupFinished) return;
        setupFinished = true;
        clearTimeout(timeout);
        reject(error);
      };

      socket.onopen = () => {
        socket.send(JSON.stringify({ setup: { model: session.model, ...session.config } }));
      };
      socket.onmessage = event => {
        let payload;
        try { payload = JSON.parse(event.data); } catch { return; }
        if (payload.setupComplete && !setupFinished) {
          setupFinished = true;
          clearTimeout(timeout);
          resolve();
          return;
        }
        if (payload.error && !setupFinished) {
          failSetup(new Error(payload.error.message || "شروع مکالمهٔ زنده پذیرفته نشد."));
          return;
        }
        if (setupFinished) handleLiveMessage(payload);
      };
      socket.onerror = () => failSetup(new Error("اتصال امن به Gemini Live برقرار نشد."));
      socket.onclose = event => {
        if (!setupFinished) {
          failSetup(new Error(`اتصال مکالمه برقرار نشد (${event.code}: ${event.reason}).`));
          return;
        }
        if (liveSocket === socket) {
          finishLiveTurn();
          liveActive = false;
          liveStarting = false;
          closeLiveResources(false);
          setLiveUi(false);
          showComposerStatus("اتصال مکالمهٔ زنده پایان یافت. برای شروع دوباره میکروفن را بزن.");
        }
      };
    });

    if (attempt !== liveAttempt) return;
    if (liveSocket !== socket || socket.readyState !== WebSocket.OPEN) throw new Error("اتصال مکالمه پیش از آماده‌شدن بسته شد.");
    attachLiveMicrophone();
    liveStarting = false;
    liveActive = true;
    setLiveUi(true);
    showComposerStatus("مکالمهٔ زنده فعال است؛ فارسی صحبت کن. برای پایان، میکروفن را بزن.", false);
    liveSessionTimeout = setTimeout(() => stopLiveConversation("مدت این نوبت مکالمه تمام شد؛ برای ادامه دوباره میکروفن را بزن."), 8 * 60 * 1000);
  } catch (error) {
    if (attempt !== liveAttempt) {
      closeLiveResources();
      return;
    }
    liveStarting = false;
    liveActive = false;
    closeLiveResources();
    setLiveUi(false);
    showComposerStatus(error.name === "NotAllowedError"
      ? "اجازهٔ میکروفن داده نشد. دسترسی میکروفن را در تنظیمات مرورگر فعال کن."
      : error.message || "شروع مکالمهٔ زنده ممکن نشد.");
  }
}

function stopLiveConversation(statusMessage = "مکالمهٔ زنده پایان یافت.") {
  if (!liveActive && !liveStarting) return;
  liveAttempt += 1;
  finishLiveTurn();
  liveActive = false;
  liveStarting = false;
  closeLiveResources();
  setLiveUi(false);
  showComposerStatus(statusMessage, false);
}

function toggleLiveConversation() {
  if (liveActive) stopLiveConversation();
  else startLiveConversation();
}

async function playMessageSpeech(message, row, button, audio) {
  if (!message.content || message.content.length > 6000) {
    showComposerStatus("برای شنیدن پاسخ‌های خیلی بلند، بخش کوتاه‌تری را انتخاب کن.");
    return;
  }
  button.disabled = true;
  const messageKey = row.dataset.messageKey;
  const speechCacheKey = `${messageKey}-${voiceChoice}`;
  try {
    let url = speechUrls.get(speechCacheKey);
    if (!url) {
      button.querySelector("span").textContent = "در حال ساخت صدا…";
      const response = await fetch("/api/speech", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ text: message.content, voice: voiceChoice })
      });
      const result = await response.json().catch(() => ({}));
      if (!response.ok) throw new Error(result.error || "ساخت صدای پاسخ انجام نشد.");
      const raw = atob(result.audio);
      const bytes = new Uint8Array(raw.length);
      for (let index = 0; index < raw.length; index += 1) bytes[index] = raw.charCodeAt(index);
      url = URL.createObjectURL(new Blob([bytes], { type: result.mimeType || "audio/wav" }));
      speechUrls.set(speechCacheKey, url);
      audio.src = url;
      audio.hidden = false;
    }
    audio.src = url;
    audio.hidden = false;
    await audio.play();
  } catch (error) {
    showComposerStatus(error.message || "پخش صدای پاسخ ممکن نشد.");
  } finally {
    button.disabled = false;
    button.querySelector("span").textContent = "شنیدن پاسخ";
  }
}

async function sendMessage(text) {
  const cleanText = text.trim();
  if (sending || !chatAvailable || (!cleanText && pendingFiles.length === 0)) return;
  const conversation = activeConversation() || createConversation();
  const filesToSend = [...pendingFiles];
  const newFileBytes = filesToSend.reduce((sum, file) => sum + file.size, 0);
  if (newFileBytes > MAX_MESSAGE_FILE_BYTES) {
    showComposerStatus("حجم پیوست‌های هر پیام حداکثر ۱۰ مگابایت است.");
    return;
  }

  const attachmentMetadata = filesToSend.map(file => ({
    id: file.id,
    name: file.name,
    mimeType: file.mimeType,
    size: file.size
  }));
  const userMessage = {
    id: crypto.randomUUID(),
    role: "user",
    content: cleanText || (attachmentMetadata.some(file => file.mimeType.startsWith("audio/"))
      ? "لطفاً پیام صوتی پیوست را گوش بده، منظور را بفهم و پاسخ مناسب بده."
      : "لطفاً فایل پیوست را بررسی کن."),
    attachments: attachmentMetadata
  };
  let requestBody;
  try {
    for (const pending of filesToSend) await storeAttachment(pending.id, pending.file);
    const prospectiveMessages = [...conversation.messages, userMessage]
      .filter(message => !message.streaming && !message.error)
      .slice(-40);
    const apiMessages = [];
    for (const message of prospectiveMessages) {
      const attachmentData = [];
      let content = message.content;
      if (message.id === userMessage.id) {
        for (const file of message.attachments || []) {
          const blob = await readStoredAttachment(file.id);
          if (!blob) throw new Error(`فایل «${file.name}» در حافظهٔ این گوشی پیدا نشد.`);
          attachmentData.push({ name: file.name, mimeType: file.mimeType, data: await blobToBase64(blob) });
        }
      } else if (message.attachments?.length) {
        const names = message.attachments.map(file => file.name).join("، ");
        content += `\n[پیوست قبلی «${names}» در این درخواست همراه نیست؛ برای بررسی دوباره آن را دوباره پیوست کن.]`;
      }
      const apiMessage = { role: message.role, content };
      if (attachmentData.length) apiMessage.attachments = attachmentData;
      apiMessages.push(apiMessage);
    }
    requestBody = JSON.stringify({ messages: apiMessages });
    if (new TextEncoder().encode(requestBody).byteLength > 15 * 1024 * 1024) {
      throw new Error("حجم فایل‌های این گفتگو برای یک درخواست زیاد است. گفتگوی تازه بساز و فایل‌های کمتری بفرست.");
    }
  } catch (error) {
    for (const file of filesToSend) await removeStoredAttachment(file.id);
    showComposerStatus(error.message || "آماده‌سازی فایل برای ارسال انجام نشد.");
    return;
  }

  for (const pending of filesToSend) {
    if (pending.previewUrl) URL.revokeObjectURL(pending.previewUrl);
  }
  pendingFiles = [];
  renderPendingFiles();
  clearComposerStatus();
  elements.input.value = "";
  resizeInput();

  const assistantMessage = { id: crypto.randomUUID(), role: "assistant", content: "", streaming: true };
  conversation.messages.push(userMessage, assistantMessage);
  conversation.title = cleanText
    ? cleanText.slice(0, 36)
    : attachmentMetadata[0]?.name?.slice(0, 36) || "فایل پیوست‌شده";
  conversation.updatedAt = Date.now();
  sending = true;
  elements.sendButton.disabled = true;
  elements.attachButton.disabled = true;
  elements.voiceButton.disabled = true;
  saveConversations();
  render();

  try {
    const response = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: requestBody
    });

    if (!response.ok) {
      const errorBody = await response.json().catch(() => ({}));
      throw new Error(errorBody.error || "پاسخ‌گویی فعلاً ممکن نیست.");
    }
    if (!response.body) throw new Error("جریان پاسخ در دسترس نیست.");

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";
    while (true) {
      const chunk = await reader.read();
      if (chunk.done) break;
      buffer += decoder.decode(chunk.value, { stream: true });
      const events = buffer.split(/\r?\n\r?\n/);
      buffer = events.pop() || "";
      for (const eventText of events) readEvent(eventText, assistantMessage, conversation);
    }
    if (buffer) readEvent(buffer, assistantMessage, conversation);
  } catch (error) {
    assistantMessage.content = error.message || "در دریافت پاسخ مشکلی پیش آمد.";
    assistantMessage.error = true;
  } finally {
    assistantMessage.streaming = false;
    conversation.updatedAt = Date.now();
    sending = false;
    elements.sendButton.disabled = false;
    elements.attachButton.disabled = false;
    elements.voiceButton.disabled = false;
    saveConversations();
    render();
    if (autoVoiceEnabled && !assistantMessage.error && assistantMessage.content) {
      const row = [...elements.messages.querySelectorAll(".message-row")]
        .find(item => item.dataset.messageKey === assistantMessage.id);
      const button = row?.querySelector(".speak-button");
      const audio = row?.querySelector(".message-audio");
      if (row && button && audio) playMessageSpeech(assistantMessage, row, button, audio);
    }
  }
}

document.querySelector("#newChat").addEventListener("click", () => {
  if (liveActive || liveStarting) stopLiveConversation();
  activeId = null;
  render();
  closeSidebar();
  elements.input.focus();
});

document.querySelector("#sidebarToggle").addEventListener("click", () => {
  document.body.classList.toggle("sidebar-open");
});

elements.sidebarScrim.addEventListener("click", closeSidebar);
elements.composer.addEventListener("submit", event => {
  event.preventDefault();
  if (sending) return;
  if (liveActive || liveStarting) stopLiveConversation("مکالمهٔ زنده برای ارسال پیام متنی پایان یافت.");
  sendMessage(elements.input.value);
});

elements.attachButton.addEventListener("click", () => {
  if (!sending && chatAvailable) elements.fileInput.click();
});
elements.fileInput.addEventListener("change", () => queueFiles([...elements.fileInput.files]));
elements.voiceButton.addEventListener("click", toggleLiveConversation);
document.addEventListener("visibilitychange", () => {
  if (document.hidden && (liveActive || liveStarting)) stopLiveConversation("مکالمهٔ زنده با خروج از صفحه پایان یافت.");
});
document.querySelectorAll(".voice-choice").forEach(button => {
  button.addEventListener("click", () => {
    voiceChoice = button.dataset.voice === "male" ? "male" : "female";
    localStorage.setItem("milad-ai-voice-choice", voiceChoice);
    updateVoiceControls();
  });
});
elements.autoVoiceToggle.addEventListener("click", () => {
  autoVoiceEnabled = !autoVoiceEnabled;
  localStorage.setItem("milad-ai-auto-voice", autoVoiceEnabled ? "1" : "0");
  updateVoiceControls();
});

elements.input.addEventListener("input", resizeInput);
elements.input.addEventListener("keydown", event => {
  if (event.key === "Enter" && !event.shiftKey) {
    event.preventDefault();
    elements.composer.requestSubmit();
  }
});

function updateConsentButton() {
  elements.accessContinue.disabled = !elements.adultConsent.checked
    || !elements.professionalConsent.checked
    || !elements.regionConsent.checked;
}

async function refreshAccess() {
  elements.input.disabled = true;
  elements.sendButton.disabled = true;
  elements.attachButton.disabled = true;
  elements.voiceButton.disabled = true;
  try {
    const response = await fetch("/api/access", { cache: "no-store" });
    const access = await response.json();
    chatAvailable = access.state === "ready" && access.available === true;
    if (chatAvailable) {
      elements.accessGate.hidden = true;
      elements.input.disabled = false;
      elements.sendButton.disabled = false;
      elements.attachButton.disabled = false;
      elements.voiceButton.disabled = false;
      setStatus(access.message || "سرویس آمادهٔ پاسخ‌گویی است.", true);
      return;
    }

    if (access.state === "consent-required") {
      elements.accessMessage.textContent = access.message;
      elements.accessChoices.hidden = false;
      elements.accessContinue.hidden = false;
      elements.accessFootnote.hidden = false;
      elements.accessGate.hidden = false;
      setStatus("تأیید شرایط استفاده لازم است.", false);
      return;
    }

    elements.accessGate.hidden = true;
    elements.accessMessage.textContent = access.message || "موتور پاسخ‌گویی فعلاً در دسترس نیست.";
    elements.accessChoices.hidden = true;
    elements.accessContinue.hidden = true;
    elements.accessFootnote.hidden = false;
    setStatus(access.message || "موتور پاسخ‌گویی تنظیم نشده است.", false);
  } catch {
    chatAvailable = false;
    setStatus("ارتباط با سرویس برقرار نشد.", false);
  }
}

elements.adultConsent.addEventListener("change", updateConsentButton);
elements.professionalConsent.addEventListener("change", updateConsentButton);
elements.regionConsent.addEventListener("change", updateConsentButton);
elements.accessContinue.addEventListener("click", async () => {
  if (!elements.adultConsent.checked || !elements.professionalConsent.checked || !elements.regionConsent.checked) return;
  elements.accessContinue.disabled = true;
  try {
    const response = await fetch("/api/consent", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ adult: true, professional: true, supportedRegion: true })
    });
    if (!response.ok) {
      const result = await response.json().catch(() => ({}));
      elements.accessMessage.textContent = result.error || "تأیید شرایط انجام نشد.";
      return;
    }
    await refreshAccess();
  } catch {
    elements.accessMessage.textContent = "ارتباط با سرویس برقرار نشد.";
  } finally {
    updateConsentButton();
  }
});

document.querySelectorAll(".suggestion").forEach(button => {
  button.addEventListener("click", () => {
    elements.input.value = button.dataset.prompt || "";
    resizeInput();
    elements.input.focus();
  });
});

render();
updateVoiceControls();
refreshAccess();
;/*liveOverlayModal*/(function(){const ov=document.createElement("div");ov.id="liveOverlay";ov.innerHTML=`<div><div class="live-badge"><span class="live-dot"></span>مکالمه زنده Gemini Live</div><p id="liveOverlaySub" style="margin-top:12px;font-size:13px;color:#94a3b8">در حال گوش دادن به صدای شما...</p></div><div class="live-orb"><div class="live-bars"><span></span><span></span><span></span><span></span></div></div><div><h3 style="font-size:17px;margin-bottom:6px">"سلام، امروز چطور می‌تونم کمکت کنم؟"</h3><p style="font-size:12px;color:#94a3b8;margin-bottom:24px">هر زمان صحبت کنی، پاسخ صوتی بلافاصله پخش می‌شه</p><button id="liveOverlayEnd" class="live-end-btn" type="button">✕ پایان مکالمه</button></div>`;document.body.appendChild(ov);const vb=document.querySelector("#voiceButton");const cs=document.querySelector("#composerStatus");const endBtn=ov.querySelector("#liveOverlayEnd");vb?.addEventListener("click",()=>{setTimeout(()=>{if(vb.classList.contains("recording")||vb.getAttribute("aria-pressed")==="true"||(cs&&cs.textContent.includes("مکالمه"))){ov.classList.add("active")}},120)});endBtn?.addEventListener("click",()=>{ov.classList.remove("active");if(vb&&(vb.classList.contains("recording")||vb.getAttribute("aria-pressed")==="true"))vb.click()});setInterval(()=>{if(!ov.classList.contains("active"))return;const t=cs?cs.textContent.trim():"";if(t)ov.querySelector("#liveOverlaySub").textContent=t;if(t.includes("پایان یافت")||t.includes("خطا")||t.includes("نرسید")||(!vb.classList.contains("recording")&&vb.getAttribute("aria-pressed")!=="true"&&!t.includes("اتصال")&&!t.includes("فعال")))ov.classList.remove("active")},350)})();
