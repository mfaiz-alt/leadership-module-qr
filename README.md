<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#17152f">
  <title>LEADERSHIP MODULE · QR Scanner</title>
  <style>
    :root { color-scheme: light; --ink:#20243a; --muted:#687086; --purple:#6851dc; --line:#e5e6ef; }
    * { box-sizing:border-box; }
    body { margin:0; min-height:100vh; padding:22px 16px; background:#f3f4fa; color:var(--ink); font:16px/1.5 Arial,sans-serif; }
    main { width:min(100%,560px); margin:0 auto; background:#fff; border:1px solid var(--line); border-radius:22px; padding:24px; box-shadow:0 12px 36px #1d214015; }
    .eyebrow { color:#7968c9; font-size:12px; font-weight:800; letter-spacing:2px; }
    h1 { margin:8px 0 6px; font-size:27px; line-height:1.2; }
    .context { color:var(--muted); margin:0 0 18px; overflow-wrap:anywhere; }
    #reader { width:100%; min-height:280px; border:1px dashed #c9c7db; border-radius:16px; overflow:hidden; background:#f7f6fc; }
    #reader video { object-fit:cover; }
    .status { min-height:48px; margin:14px 0; color:#5d4c9e; font-weight:700; overflow-wrap:anywhere; }
    label { display:block; margin:18px 0 7px; color:#555d70; font-weight:700; }
    input { width:100%; border:1px solid var(--line); border-radius:12px; padding:14px; font-size:17px; }
    button { border:0; border-radius:12px; padding:14px 18px; font-size:16px; font-weight:700; cursor:pointer; }
    button:disabled { opacity:.5; cursor:not-allowed; }
    .primary { width:100%; margin-top:10px; color:#fff; background:var(--purple); }
    .secondary { color:#4c5264; background:#eef0f5; }
    .actions { display:flex; gap:10px; margin-top:12px; }
    .actions button { flex:1; }
    .note { margin-top:20px; color:#7b8190; font-size:13px; }
    .error { color:#a12d45; }
  </style>
</head>
<body>
<main>
  <span class="eyebrow">ATTENDANCE QUICK SCAN</span>
  <h1>Scan Student QR Code</h1>
  <p class="context" id="context">Loading selected session…</p>

  <div id="reader"></div>
  <p id="status" class="status" role="status" aria-live="polite">Preparing camera…</p>

  <label for="studentId">Or enter Student ID manually</label>
  <input id="studentId" inputmode="numeric" autocomplete="off" placeholder="Enter student ID">
  <button id="check" class="primary" type="button">Check Student ID</button>

  <div class="actions">
    <button id="restart" class="secondary" type="button">Restart camera</button>
    <button id="back" class="secondary" type="button">Back to attendance</button>
  </div>
  <p class="note">Keep this page open while scanning. The scanned Student ID will be sent back to LEADERSHIP MODULE for verification against the selected module, date and group.</p>
</main>

<script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js"></script>
<script>
(() => {
  const params = new URLSearchParams(location.search);
  const session = {
    module: params.get('module') || '',
    date: params.get('date') || '',
    group: params.get('group') || ''
  };
  const $ = id => document.getElementById(id);
  let scanner = null;
  let busy = false;

  $('context').textContent =
    `${session.module || 'Module not selected'} · ${session.date || 'Date not selected'} · ${session.group ? 'Group ' + session.group : 'Group not selected'}`;

  function setStatus(message, isError = false) {
    $('status').textContent = message;
    $('status').classList.toggle('error', isError);
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
    if (!scanner) return;
    const current = scanner;
    scanner = null;
    try { if (current.isScanning) await current.stop(); } catch (e) {}
    try { current.clear(); } catch (e) {}
  }

  async function sendStudentId(raw) {
    if (busy) return;
    let studentId;
    try { studentId = extractStudentId(raw); }
    catch (error) { setStatus(error.message, true); return; }

    if (!session.module || !session.date || !session.group) {
      setStatus('Session details are missing. Close this page and select module, date and group again.', true);
      return;
    }

    busy = true;
    $('check').disabled = true;
    setStatus('Student ID captured. Returning to LEADERSHIP MODULE…');
    await stopCamera();

    const message = {
      type: 'LEADERSHIP_MODULE_QR_RESULT',
      studentId,
      module: session.module,
      date: session.date,
      group: session.group
    };

    if (window.opener && !window.opener.closed) {
      window.opener.postMessage(message, '*');
      setStatus('Student ID sent. You can return to the LEADERSHIP MODULE tab.');
      setTimeout(() => window.close(), 500);
    } else {
      busy = false;
      $('check').disabled = false;
      $('studentId').value = studentId;
      setStatus('The original attendance page is not connected. Return to it and try Scan Student QR again.', true);
    }
  }

  async function startCamera() {
    await stopCamera();
    if (typeof Html5Qrcode === 'undefined') {
      setStatus('QR scanner library did not load. Check your internet connection or enter the Student ID manually.', true);
      return;
    }

    setStatus('Requesting camera access…');
    try {
      scanner = new Html5Qrcode('reader');
      const cameras = await Html5Qrcode.getCameras();
      if (!cameras || !cameras.length) throw new Error('No camera was detected on this device.');

      const preferred = cameras.find(camera => /back|rear|environment|world/i.test(camera.label)) || cameras[cameras.length - 1];
      await scanner.start(
        preferred.id,
        { fps: 10, qrbox: { width: 245, height: 245 }, aspectRatio: 1 },
        decodedText => sendStudentId(decodedText),
        () => {}
      );
      setStatus('Camera ready. Point it at the QR code on the student card.');
    } catch (error) {
      await stopCamera();
      setStatus('Unable to start camera: ' + (error && error.message ? error.message : String(error)) +
        '. Check Chrome camera permission, then use manual Student ID entry if needed.', true);
    }
  }

  $('check').addEventListener('click', () => sendStudentId($('studentId').value));
  $('studentId').addEventListener('keydown', event => {
    if (event.key === 'Enter') sendStudentId($('studentId').value);
  });
  $('restart').addEventListener('click', () => {
    busy = false;
    $('check').disabled = false;
    startCamera();
  });
  $('back').addEventListener('click', () => {
    stopCamera().finally(() => {
      if (window.opener && !window.opener.closed) window.close();
      else history.back();
    });
  });
  window.addEventListener('pagehide', stopCamera);

  if (!session.module || !session.date || !session.group) {
    setStatus('Missing session details. Open the scanner from LEADERSHIP MODULE after selecting module, date and a specific group.', true);
  } else {
    startCamera();
  }
})();
</script>
</body>
</html>
