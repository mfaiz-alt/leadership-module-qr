<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#24104f">
  <title>LEADERSHIP MODULE · QR Scanner</title>
  <style>
    :root { color-scheme: light; --ink:#20243a; --muted:#687086; --purple:#6851dc; --line:#e5e6ef; }
    * { box-sizing:border-box; }
    body { margin:0; min-height:100vh; padding:16px; background:#f3f4fa; color:var(--ink); font:16px/1.45 Arial,sans-serif; }
    main { width:min(100%,520px); margin:0 auto; background:#fff; border:1px solid var(--line); border-radius:20px; padding:20px; box-shadow:0 12px 36px #1d214015; }
    .top { display:flex; align-items:center; justify-content:space-between; gap:12px; }
    .eyebrow { color:#7968c9; font-size:11px; font-weight:800; letter-spacing:1.8px; }
    .live { font-size:11px; font-weight:800; color:#176b43; background:#e8f7ef; padding:5px 9px; border-radius:999px; }
    h1 { margin:8px 0 4px; font-size:25px; line-height:1.2; }
    .context { color:var(--muted); margin:0 0 16px; overflow-wrap:anywhere; font-size:14px; }
    #reader { width:100%; min-height:260px; border:1px dashed #c9c7db; border-radius:15px; overflow:hidden; background:#f7f6fc; }
    #reader video { object-fit:cover; }
    .status { min-height:44px; margin:12px 0; color:#5d4c9e; font-weight:700; white-space:pre-line; overflow-wrap:anywhere; font-size:14px; }
    .error { color:#a12d45; }
    .waiting { color:#875400; }
    label { display:block; margin:15px 0 6px; color:#555d70; font-weight:700; font-size:13px; }
    input { width:100%; border:1px solid var(--line); border-radius:11px; padding:13px; font-size:17px; }
    button { border:0; border-radius:11px; padding:12px 15px; font-size:14px; font-weight:700; cursor:pointer; }
    button:disabled { opacity:.45; cursor:not-allowed; }
    .primary { width:100%; margin-top:9px; color:#fff; background:var(--purple); }
    .secondary { color:#4c5264; background:#eef0f5; }
    .actions { display:flex; gap:9px; margin-top:12px; }
    .actions button { flex:1; }
    .note { margin:16px 0 0; color:#7b8190; font-size:12px; }
    .status-card { border-radius:12px; padding:10px 12px; background:#f4f2fc; }
  </style>
</head>
<body>
  <main>
    <div class="top">
      <span class="eyebrow">LEADERSHIP MODULE · ATTENDANCE</span>
      <span class="live" id="connectionState">CONNECTING</span>
    </div>
    <h1>Scan Student QR Code</h1>
    <p class="context" id="context">Loading selected session…</p>

    <div id="reader"></div>
    <p id="status" class="status status-card" role="status" aria-live="polite">Preparing camera…</p>

    <label for="studentId">Manual Student ID entry</label>
    <input id="studentId" inputmode="numeric" autocomplete="off" placeholder="Enter Student ID">
    <button id="check" class="primary" type="button">Check Student ID</button>

    <div class="actions">
      <button id="restart" class="secondary" type="button">Restart camera</button>
      <button id="back" class="secondary" type="button">Close scanner</button>
    </div>
    <p class="note">Keep this scanner window open during attendance. After you save or cancel a student's update in LEADERSHIP MODULE, scanning will resume automatically. Close this window manually when the session ends.</p>
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
      let cameraStarting = false;

      $('context').textContent = `${session.module || 'Module not selected'} · ${session.date || 'Date not selected'} · ${session.group ? 'Group ' + session.group : 'Group not selected'}`;

      function setStatus(message, type) {
        $('status').textContent = message;
        $('status').classList.toggle('error', type === 'error');
        $('status').classList.toggle('waiting', type === 'waiting');
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
        try { if (current.isScanning) await current.stop(); } catch (error) {}
        try { current.clear(); } catch (error) {}
      }

      async function startCamera() {
        if (busy || cameraStarting) return;
        cameraStarting = true;
        await stopCamera();
        if (typeof Html5Qrcode === 'undefined') {
          setStatus('QR scanner library did not load. Check your internet connection or use manual Student ID entry.', 'error');
          cameraStarting = false;
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
          $('connectionState').textContent = 'READY';
          setStatus('Camera ready. Point it at the QR code on the student card.');
        } catch (error) {
          await stopCamera();
          $('connectionState').textContent = 'CAMERA ERROR';
          setStatus('Unable to start camera: ' + (error && error.message ? error.message : String(error)) + '\nCheck Chrome camera permission, then use manual Student ID entry if needed.', 'error');
        } finally {
          cameraStarting = false;
        }
      }

      async function sendStudentId(raw) {
        if (busy) return;
        let studentId;
        try { studentId = extractStudentId(raw); }
        catch (error) { setStatus(error.message, 'error'); return; }

        if (!session.module || !session.date || !session.group) {
          setStatus('Session details are missing. Close this scanner and select module, date and group again.', 'error');
          return;
        }
        if (!window.opener || window.opener.closed) {
          setStatus('The LEADERSHIP MODULE window is no longer connected. Close this scanner and reopen it from the attendance page.', 'error');
          return;
        }

        busy = true;
        $('check').disabled = true;
        $('restart').disabled = true;
        $('connectionState').textContent = 'WAITING';
        setStatus('Student ID ' + studentId + ' captured.\nComplete the attendance or registration in LEADERSHIP MODULE. This scanner will restart automatically afterwards.', 'waiting');
        await stopCamera();

        window.opener.postMessage({
          type: 'LEADERSHIP_MODULE_QR_RESULT',
          studentId,
          module: session.module,
          date: session.date,
          group: session.group
        }, '*');
      }

      window.addEventListener('message', async event => {
        if (event.source !== window.opener) return;
        const message = event.data;
        if (!message || typeof message !== 'object') return;

        if (message.type === 'LEADERSHIP_MODULE_QR_RESUME') {
          busy = false;
          $('check').disabled = false;
          $('restart').disabled = false;
          $('studentId').value = '';
          $('connectionState').textContent = 'READY';
          setStatus('Ready for the next student. Point the camera at the next QR code.');
          await startCamera();
        }

        if (message.type === 'LEADERSHIP_MODULE_QR_CLOSE') {
          await stopCamera();
          window.close();
        }
      });

      $('check').addEventListener('click', () => sendStudentId($('studentId').value));
      $('studentId').addEventListener('keydown', event => {
        if (event.key === 'Enter') sendStudentId($('studentId').value);
      });
      $('restart').addEventListener('click', () => {
        if (busy) return;
        startCamera();
      });
      $('back').addEventListener('click', async () => {
        await stopCamera();
        if (window.opener && !window.opener.closed) {
          window.opener.postMessage({ type: 'LEADERSHIP_MODULE_QR_CLOSED' }, '*');
        }
        window.close();
      });
      window.addEventListener('pagehide', stopCamera);

      if (!session.module || !session.date || !session.group) {
        $('connectionState').textContent = 'SESSION ERROR';
        setStatus('Missing session details. Open the scanner from LEADERSHIP MODULE after selecting module, date and a specific group.', 'error');
      } else if (!window.opener) {
        $('connectionState').textContent = 'NOT CONNECTED';
        setStatus('This page was opened directly. Close it and launch Scan Student QR from LEADERSHIP MODULE.', 'error');
      } else {
        startCamera();
      }
    })();
  </script>
</body>
</html>
