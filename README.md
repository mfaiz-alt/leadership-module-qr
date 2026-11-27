<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#6851dc">
  <title>LEADERSHIP MODULE · QR Scanner</title>
  <style>
    :root{color-scheme:light;--ink:#20243a;--muted:#687086;--purple:#6851dc;--line:#e5e6ef;}
    *{box-sizing:border-box} body{margin:0;min-height:100vh;padding:18px 14px;background:#f3f4fa;color:var(--ink);font:15px/1.5 Arial,sans-serif}
    main{width:min(100%,470px);margin:0 auto;background:#fff;border:1px solid var(--line);border-radius:20px;padding:20px;box-shadow:0 12px 36px #1d214015}
    .eyebrow{color:#7968c9;font-size:11px;font-weight:800;letter-spacing:1.8px} h1{margin:7px 0;font-size:24px;line-height:1.2}
    .context{color:var(--muted);margin:0 0 14px;overflow-wrap:anywhere;font-size:13px}
    #reader{width:100%;min-height:260px;border:1px dashed #c9c7db;border-radius:14px;overflow:hidden;background:#f7f6fc}
    #reader video{object-fit:cover} .status{min-height:48px;margin:12px 0;color:#5d4c9e;font-weight:700;overflow-wrap:anywhere}
    .status.error{color:#a12d45} .status.success{color:#26734d}
    label{display:block;margin:14px 0 6px;color:#555d70;font-weight:700;font-size:13px}
    input{width:100%;border:1px solid var(--line);border-radius:11px;padding:13px;font-size:16px}
    button{border:0;border-radius:11px;padding:13px 15px;font-size:14px;font-weight:700;cursor:pointer}
    button:disabled{opacity:.5;cursor:not-allowed}.primary{width:100%;margin-top:9px;color:#fff;background:var(--purple)}
    .secondary{color:#4c5264;background:#eef0f5}.actions{display:flex;gap:9px;margin-top:12px}.actions button{flex:1}
    .note{margin:16px 0 0;color:#7b8190;font-size:12px}.waiting{display:none;border-radius:13px;background:#f3f0ff;padding:16px;color:#4b3d94;font-weight:700}
    .waiting small{display:block;margin-top:6px;color:#6f6a87;font-weight:400}.hidden{display:none!important}
  </style>
</head>
<body>
<main>
  <span class="eyebrow">ATTENDANCE QUICK SCAN</span>
  <h1>Scan Student QR Code</h1>
  <p class="context" id="context">Loading selected session…</p>
  <section id="scannerPanel">
    <div id="reader"></div>
    <p id="status" class="status" role="status">Preparing camera…</p>
    <label for="studentId">Or enter Student ID manually</label>
    <input id="studentId" inputmode="numeric" autocomplete="off" placeholder="Enter Student ID">
    <button id="check" class="primary" type="button">Check Student ID</button>
    <div class="actions">
      <button id="restart" class="secondary" type="button">Restart camera</button>
      <button id="back" class="secondary" type="button">Close scanner</button>
    </div>
  </section>
  <section id="waiting" class="waiting" aria-live="polite">
    Student ID captured. Complete the attendance update or registration in LEADERSHIP MODULE.
    <small>The camera is paused. It will resume automatically when the attendance form is saved or closed.</small>
  </section>
  <p class="note">Keep this scanner window open during the session. Close it manually when scanning is finished.</p>
</main>
<script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js"></script>
<script>
(() => {
  'use strict';
  const params = new URLSearchParams(location.search);
  const session = {
    module: params.get('module') || '',
    date: params.get('date') || '',
    group: params.get('group') || '',
    token: params.get('token') || ''
  };
  const $ = id => document.getElementById(id);
  let scanner = null;
  let busy = false;
  let starting = false;
  $('context').textContent = `${session.module || 'Module missing'} · ${session.date || 'Date missing'} · ${session.group ? 'Group ' + session.group : 'Group missing'}`;

  function setStatus(message, kind='') {
    $('status').textContent = message;
    $('status').className = 'status' + (kind ? ' ' + kind : '');
  }
  function extractStudentId(raw) {
    const value = String(raw || '').trim();
    const labelled = value.match(/(?:STUDENT[_\s-]*ID|STUDENTID|ID)\s*[:=#/ ]\s*([0-9]{6,15})/i);
    if (labelled) return labelled[1];
    const query = value.match(/[?&](?:student[_-]?id|studentid|id)=([0-9]{6,15})(?:&|$)/i);
    if (query) return query[1];
    const plain = value.match(/^([0-9]{6,15})$/);
    if (plain) return plain[1];
    const numeric = value.match(/\b([0-9]{6,15})\b/);
    if (numeric) return numeric[1];
    throw new Error('QR code does not contain a valid Student ID (6–15 digits).');
  }
  async function stopCamera() {
    const current = scanner;
    if (!current) return;
    scanner = null;
    try { if (current.isScanning) await current.stop(); } catch (error) {}
    try { current.clear(); } catch (error) {}
  }
  function sendClosed() {
    try {
      if (window.opener && !window.opener.closed) {
        window.opener.postMessage({type:'LEADERSHIP_MODULE_QR_CLOSED',token:session.token}, '*');
      }
    } catch (error) {}
  }
  async function pauseAfterScan(studentId) {
    busy = true;
    $('check').disabled = true;
    setStatus('Student ID captured. Sending to LEADERSHIP MODULE…');
    await stopCamera();
    if (!session.module || !session.date || !session.group || !session.token) {
      busy = false;
      $('check').disabled = false;
      setStatus('Scanner session details are incomplete. Close and reopen it from LEADERSHIP MODULE.', 'error');
      return;
    }
    if (!window.opener || window.opener.closed) {
      busy = false;
      $('check').disabled = false;
      $('studentId').value = studentId;
      setStatus('LEADERSHIP MODULE is not connected. Return to the attendance page and reopen the scanner.', 'error');
      return;
    }
    const message = {
      type:'LEADERSHIP_MODULE_QR_RESULT', token:session.token, studentId,
      module:session.module, date:session.date, group:session.group
    };
    try {
      window.opener.postMessage(message, '*');
      try { window.opener.focus(); } catch (error) {}
      setStatus('Student ID sent. Waiting for LEADERSHIP MODULE to confirm…');
    } catch (error) {
      busy = false;
      $('check').disabled = false;
      setStatus('Unable to send the Student ID to LEADERSHIP MODULE: ' + error.message, 'error');
    }
  }
  async function startCamera() {
    if (busy || starting) return;
    starting = true;
    await stopCamera();
    if (typeof Html5Qrcode === 'undefined') {
      starting = false;
      setStatus('QR scanner library did not load. Check your internet connection or enter Student ID manually.', 'error');
      return;
    }
    setStatus('Requesting camera access…');
    try {
      scanner = new Html5Qrcode('reader');
      const cameras = await Html5Qrcode.getCameras();
      if (!cameras || !cameras.length) throw new Error('No camera was detected on this device.');
      const preferred = cameras.find(camera => /back|rear|environment|world/i.test(camera.label)) || cameras[cameras.length-1];
      await scanner.start(preferred.id,
        {fps:10, qrbox:{width:245,height:245}, aspectRatio:1},
        decodedText => { if (!busy) pauseAfterScan(decodedText); },
        () => {}
      );
      setStatus('Camera ready. Point it at the QR code on the student card.', 'success');
    } catch (error) {
      await stopCamera();
      setStatus('Unable to start camera: ' + (error && error.message ? error.message : String(error)) + '. Check Chrome camera permission, then use manual Student ID entry.', 'error');
    } finally {
      starting = false;
    }
  }
  async function resumeFromHost() {
    if (!busy) return;
    busy = false;
    $('scannerPanel').classList.remove('hidden');
    $('waiting').style.display = 'none';
    $('studentId').value = '';
    $('check').disabled = false;
    setStatus('Restarting camera…');
    try { window.focus(); } catch (error) {}
    await startCamera();
  }
  window.addEventListener('message', event => {
    if (!window.opener || event.source !== window.opener) return;
    const data = event.data;
    if (!data || data.token !== session.token) return;
    if (data.type === 'LEADERSHIP_MODULE_QR_RECEIVED') {
      busy = true;
      $('scannerPanel').classList.add('hidden');
      $('waiting').style.display = 'block';
      setStatus('Student ID received. Scanner paused while the record is updated.', 'success');
    } else if (data.type === 'LEADERSHIP_MODULE_QR_RESUME') {
      $('scannerPanel').classList.remove('hidden');
      $('waiting').style.display = 'none';
      resumeFromHost();
    } else if (data.type === 'LEADERSHIP_MODULE_QR_CLOSE') {
      stopCamera().finally(() => window.close());
    }
  });
  $('check').addEventListener('click', () => pauseAfterScan($('studentId').value));
  $('studentId').addEventListener('keydown', event => {
    if (event.key === 'Enter') pauseAfterScan($('studentId').value);
  });
  $('restart').addEventListener('click', () => {
    busy = false;
    $('scannerPanel').classList.remove('hidden');
    $('waiting').style.display = 'none';
    $('check').disabled = false;
    startCamera();
  });
  $('back').addEventListener('click', async () => {
    await stopCamera();
    sendClosed();
    window.close();
  });
  window.addEventListener('pagehide', () => { stopCamera(); sendClosed(); });
  if (!session.module || !session.date || !session.group || !session.token) {
    setStatus('Missing session details. Open the scanner from LEADERSHIP MODULE after selecting module, date and group.', 'error');
  } else {
    startCamera();
  }
})();
</script>
</body>
</html>
