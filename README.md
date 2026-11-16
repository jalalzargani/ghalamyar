<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="theme-color" content="#0f172a">
<title>قلم‌یار — دستیار نوشتاری</title>
<style>
:root{--bg:#0f172a;--card:#1e293b;--card2:#273449;--txt:#e2e8f0;--muted:#94a3b8;--accent:#38bdf8;--rec:#ef4444}
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
body{margin:0;background:var(--bg);color:var(--txt);font-family:Tahoma,"Segoe UI",system-ui,sans-serif;padding:12px 12px 40px;min-height:100vh}
h1{font-size:20px;margin:6px 0 14px;text-align:center}
h1 span{color:var(--accent)}
.card{background:var(--card);border-radius:16px;padding:12px;margin-bottom:12px}
.row{display:flex;gap:8px;align-items:center;flex-wrap:wrap}
select,button{font-family:inherit;font-size:15px;border-radius:12px;border:0;padding:12px 14px}
select{background:var(--card2);color:var(--txt);flex:1;min-width:130px}
button{background:var(--card2);color:var(--txt);cursor:pointer;transition:.15s}
button:active{transform:scale(.97)}
button.primary{background:var(--accent);color:#06202f;font-weight:bold}
button.rec{background:var(--rec);color:#fff;font-weight:bold;width:100%;padding:18px;font-size:18px;border-radius:18px}
button.rec.on{background:#7f1d1d;animation:pulse 1.4s infinite}
@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(239,68,68,.6)}50%{box-shadow:0 0 0 14px rgba(239,68,68,0)}}
.status{font-size:13px;color:var(--muted);text-align:center;margin-top:8px;min-height:18px}
textarea{width:100%;min-height:240px;background:var(--card);color:var(--txt);border:1px solid #334155;border-radius:16px;padding:14px;font-size:17px;line-height:2;font-family:inherit;resize:vertical}
textarea:focus{outline:2px solid var(--accent)}
.interim{color:var(--muted);font-style:italic;font-size:15px;min-height:22px;padding:4px 8px}
.tools{display:grid;grid-template-columns:1fr 1fr;gap:8px}
.meta{font-size:13px;color:var(--muted);display:flex;justify-content:space-between;padding:0 6px 6px}
.note{display:flex;gap:8px;align-items:center;background:var(--card2);border-radius:12px;padding:8px 10px;margin-bottom:8px;font-size:14px}
.note .t{flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.note button{padding:8px 10px;font-size:13px}
.hint{font-size:12px;color:var(--muted);line-height:1.9;margin-top:10px}
</style>
</head>
<body>

<h1>قلم‌یار <span>✍️</span></h1>

<div class="card">
  <div class="row">
    <select id="lang">
      <option value="fa-IR" selected>فارسی</option>
      <option value="en-US">English</option>
      <option value="ar-SA">العربية</option>
      <option value="tr-TR">Türkçe</option>
    </select>
    <button id="clearBtn">🗑 پاک کردن</button>
  </div>
</div>

<div class="card">
  <button id="recBtn" class="rec">🎤 شروع گفتار</button>
  <div class="status" id="status">آماده</div>
  <div class="interim" id="interim"></div>
</div>

<div class="meta">
  <span id="count">۰ کلمه</span>
  <span id="timer">۰۰:۰۰</span>
</div>

<textarea id="editor" placeholder="هر چه می‌گویی اینجا نوشته می‌شود... با دست هم می‌توانی ویرایش کنی."></textarea>

<div class="card" style="margin-top:12px">
  <div class="tools">
    <button id="saveBtn" class="primary">💾 ذخیره یادداشت</button>
    <button id="copyBtn">📋 کپی متن</button>
    <button id="wordBtn">📄 خروجی Word</button>
    <button id="pdfBtn">🖨 خروجی PDF</button>
  </div>
  <div class="hint">برای PDF، پنجره چاپ باز می‌شود؛ گزینه «Save as PDF / ذخیره به‌صورت PDF» را بزن.</div>
</div>

<div class="card">
  <div style="font-size:14px;color:var(--muted);margin-bottom:8px">یادداشت‌های ذخیره‌شده</div>
  <div id="notes"></div>
</div>

<script>
const $ = id => document.getElementById(id);
const editor=$('editor'), recBtn=$('recBtn'), statusEl=$('status'), interimEl=$('interim');
const langSel=$('lang'), countEl=$('count'), timerEl=$('timer'), notesEl=$('notes');

const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
let recog=null, listening=false, restart=false, startTime=0, timerInt=null;

const faDigits='۰۱۲۳۴۵۶۷۸۹';
const toFa = s => String(s).replace(/\d/g, d => faDigits[d]);

/* ---------- درج متن ---------- */
function insertText(t){
  if(!t) return;
  const pos = (document.activeElement===editor) ? editor.selectionStart : editor.value.length;
  const before = editor.value.slice(0,pos);
  const after  = editor.value.slice(pos);
  const sep = (before && !/\s$/.test(before)) ? ' ' : '';
  const ins = sep + t;
  editor.value = before + ins + after;
  const np = (before+ins).length;
  editor.selectionStart = editor.selectionEnd = np;
  editor.scrollTop = editor.scrollHeight;
  updateCount(); autosave();
}

function updateCount(){
  const w = editor.value.trim().split(/\s+/).filter(Boolean).length;
  countEl.textContent = toFa(w) + ' کلمه';
}

/* ---------- تشخیص گفتار ---------- */
function buildRecog(){
  const r = new SR();
  r.lang = langSel.value;
  r.continuous = true;
  r.interimResults = true;
  r.maxAlternatives = 1;

  r.onstart = () => { statusEl.textContent = '🔴 در حال شنیدن...'; };

  r.onresult = (e) => {
    let interim='';
    for(let i=e.resultIndex; i<e.results.length; i++){
      const res=e.results[i], txt=res[0].transcript;
      if(res.isFinal) insertText(txt.trim());
      else interim += txt;
    }
    interimEl.textContent = interim;
  };

  r.onerror = (e) => {
    if(e.error==='not-allowed' || e.error==='service-not-allowed'){
      statusEl.textContent='❌ دسترسی میکروفون داده نشد';
      hardStop();
    } else if(e.error==='no-speech'){
      statusEl.textContent='سکوت بود، ادامه بده...';
    } else if(e.error==='network'){
      statusEl.textContent='❌ اینترنت قطع است (تشخیص گفتار به اینترنت نیاز دارد)';
    } else {
      statusEl.textContent='خطا: '+e.error;
    }
  };

  r.onend = () => {
    if(listening && restart){
      setTimeout(()=>{ try{ r.start(); }catch(_){} }, 250);
    }
  };
  return r;
}

function startRec(){
  if(!SR){ statusEl.textContent='❌ این مرورگر پشتیبانی نمی‌کند؛ Chrome را باز کن.'; return; }
  if(!recog) recog = buildRecog();
  recog.lang = langSel.value;
  listening = true; restart = true;
  try{ recog.start(); }catch(_){}
  recBtn.textContent='⏹ توقف';
  recBtn.classList.add('on');
  startTimer();
}

function stopRec(){
  listening = false; restart = false;
  if(recog){ try{ recog.stop(); }catch(_){} }
  recBtn.textContent='🎤 شروع گفتار';
  recBtn.classList.remove('on');
  stopTimer();
  interimEl.textContent='';
  statusEl.textContent='متوقف شد — متن ذخیره شد';
  autosave();
}

function hardStop(){ listening=false; restart=false; recBtn.textContent='🎤 شروع گفتار'; recBtn.classList.remove('on'); stopTimer(); }

recBtn.onclick = () => listening ? stopRec() : startRec();

langSel.onchange = () => {
  if(listening){ stopRec(); setTimeout(startRec, 400); }
};

/* ---------- زمان‌سنج ---------- */
function startTimer(){
  if(timerInt) return;
  startTime = Date.now();
  timerInt = setInterval(()=>{
    const s = Math.floor((Date.now()-startTime)/1000);
    const m = String(Math.floor(s/60)).padStart(2,'0');
    const ss = String(s%60).padStart(2,'0');
    timerEl.textContent = toFa(m+':'+ss);
  }, 500);
}
function stopTimer(){ clearInterval(timerInt); timerInt=null; }

/* ---------- ذخیره خودکار پیش‌نویس ---------- */
let saveT;
function autosave(){
  clearTimeout(saveT);
  saveT = setTimeout(()=>{ try{ localStorage.setItem('ghalamyar_draft', editor.value); }catch(_){}} , 400);
}

/* ---------- یادداشت‌ها ---------- */
const KEY='ghalamyar_notes_v1';
function getNotes(){ try{ return JSON.parse(localStorage.getItem(KEY))||[]; }catch(_){ return []; } }
function setNotes(n){ localStorage.setItem(KEY, JSON.stringify(n)); renderNotes(); }

function saveNote(){
  const text = editor.value.trim();
  if(!text){ alert('متنی برای ذخیره نیست.'); return; }
  const notes = getNotes();
  if(notes.length && notes[0].text === text){ statusEl.textContent='قبلاً ذخیره شده'; return; }
  notes.unshift({ title: text.slice(0,45).replace(/\n/g,' '), text, date: Date.now() });
  setNotes(notes);
  statusEl.textContent='💾 ذخیره شد';
}

function renderNotes(){
  const notes = getNotes();
  notesEl.innerHTML='';
  if(!notes.length){ notesEl.innerHTML='<div class="hint">هنوز چیزی ذخیره نکرده‌ای.</div>'; return; }
  notes.forEach((n,i)=>{
    const d=document.createElement('div'); d.className='note';
    const t=document.createElement('div'); t.className='t'; t.textContent=n.title||'بدون عنوان';
    const open=document.createElement('button'); open.textContent='باز کردن';
    open.onclick=()=>{ editor.value=n.text; updateCount(); autosave(); window.scrollTo({top:0,behavior:'smooth'}); };
    const del=document.createElement('button'); del.textContent='🗑';
    del.onclick=()=>{ if(confirm('حذف شود؟')){ const a=getNotes(); a.splice(i,1); setNotes(a);} };
    d.append(t,open,del);
    notesEl.appendChild(d);
  });
}

/* ---------- خروجی‌ها ---------- */
const esc = s => s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');

function exportPDF(){
  const text = editor.value.trim();
  if(!text){ alert('متنی نیست!'); return; }
  const w = window.open('', '_blank');
  if(!w){ alert('پنجره باز نشد؛ اجازه پاپ‌آپ بده.'); return; }
  w.document.open();
  w.document.write('<!DOCTYPE html><html dir="rtl" lang="fa"><head><meta charset="utf-8"><title>قلم‌یار</title><style>body{font-family:Tahoma,Arial,sans-serif;font-size:15pt;line-height:2;padding:24px;color:#000;white-space:pre-wrap;word-wrap:break-word}@page{size:A4;margin:2cm}</style></head><body>'+esc(text)+'</body></html>');
  w.document.close();
  w.focus();
  setTimeout(()=>{ try{ w.print(); }catch(_){}} , 700);
}

function exportWord(){
  const text = editor.value.trim();
  if(!text){ alert('متنی نیست!'); return; }
  const html='<html xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:w="urn:schemas-microsoft-com:office:word" xmlns="http://www.w3.org/TR/REC-html40"><head><meta charset="utf-8"><style>body{font-family:Tahoma;font-size:14pt;line-height:1.9}</style></head><body dir="rtl">'+esc(text).replace(/\n/g,'<br>')+'</body></html>';
  const blob=new Blob(['\ufeff',html],{type:'application/msword'});
  const a=document.createElement('a');
  a.href=URL.createObjectURL(blob);
  a.download='ghalamyar-'+Date.now()+'.doc';
  a.click();
  setTimeout(()=>URL.revokeObjectURL(a.href),3000);
  statusEl.textContent='📄 فایل Word ساخته شد';
}

async function copyText(){
  if(!editor.value) return;
  try{ await navigator.clipboard.writeText(editor.value); }
  catch(_){ editor.select(); document.execCommand('copy'); }
  statusEl.textContent='📋 کپی شد';
}

/* ---------- اتصال دکمه‌ها ---------- */
$('saveBtn').onclick = saveNote;
$('copyBtn').onclick = copyText;
$('wordBtn').onclick = exportWord;
$('pdfBtn').onclick = exportPDF;
$('clearBtn').onclick = ()=>{ if(editor.value && !confirm('متن پاک شود؟')) return; editor.value=''; updateCount(); autosave(); interimEl.textContent=''; };

editor.addEventListener('input', ()=>{ updateCount(); autosave(); });

/* ---------- راه‌اندازی ---------- */
try{ const d=localStorage.getItem('ghalamyar_draft'); if(d) editor.value=d; }catch(_){}
updateCount();
renderNotes();

if(!SR){
  statusEl.textContent='⚠️ این مرورگر تشخیص گفتار ندارد. Chrome را باز کن.';
  recBtn.disabled=true; recBtn.style.opacity=.5;
}
</script>
</body>
</html>