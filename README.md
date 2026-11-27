<script>
let DATA = null;
let selectedStudent = null;
let attendance = {};
let directorySort = { key: 'facilitator', dir: 1 };
let viewRequest = 0;
let isViewingArchive = false;
let directoryDetailsOverride = null;
let qrScanner = null;
let qrScanBusy = false;
let qrPopupWindow = null;
let qrSessionToken = '';
let pendingScanRegistration = false;
let pendingScannedStudentId = '';

const $ = id => document.getElementById(id);

function esc(value) {
  return String(value ?? '').replace(/[&<>"']/g, char => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  }[char]));
}

function key(value) {
  return String(value || '').trim().toUpperCase();
}

function expandStudentModules(modules) {
  const result = [];

  (modules || []).forEach(value => {
    const module = String(value || '')
      .replace(/\u00A0/g, ' ')
      .replace(/\s+/g, ' ')
      .trim()
      .toUpperCase();

    if (!module) return;

    const match = module.match(
      /^(MODUL ASPIRASI UTAMA)\s+(\d+)\s*&\s*(\d+)$/
    );

    if (match) {
      result.push(`${match[1]} ${match[2]}`);
      result.push(`${match[1]} ${match[3]}`);
    } else {
      result.push(module);
    }
  });

  return [...new Set(result)];
}

function normalizeModule(value) {
  return String(value || '')
    .replace(/\u00A0/g, ' ')
    .replace(/\s+/g, ' ')
    .trim()
    .toUpperCase();
}

function moduleMatches(moduleA, moduleB) {
  const a = expandStudentModules([moduleA])
    .map(normalizeModule);

  const b = expandStudentModules([moduleB])
    .map(normalizeModule);

  return a.some(value => b.includes(value));
}

function normalizeGroup(value) {
  return String(value || '')
    .replace(/\u00A0/g, ' ')
    .trim()
    .toUpperCase()
    .replace(/^GROUP\s*/, '')
    .replace(/\s+/g, '');
}

function init() {
  google.script.run
    .withSuccessHandler(data => {
      DATA = data || { students: [], modules: [], details: [], statuses: [] };

      $('today').textContent = new Date().toLocaleDateString(undefined, {
        weekday: 'long', day: 'numeric', month: 'short', year: 'numeric'
      });

      $('status').innerHTML = '';
      (DATA.statuses || []).forEach(status => $('status').add(new Option(status, status)));

      const moduleNames = [...new Set(
        (DATA.modules || []).map(item => item.module).filter(Boolean)
      )].sort((a, b) => a.localeCompare(b));
      $('module').innerHTML = '<option value="">All modules</option>';
      moduleNames.forEach(module => $('module').add(new Option(module, module)));

      // If DETAILS has SESSION_DATE for module/group rows, automatically
      // open the nearest upcoming event on the dashboard without manual filters.
      const upcoming = getNextUpcomingSession_();
      $('module').value = upcoming ? upcoming.module : '';
      $('date').value = upcoming ? upcoming.date : '';
      $('group').value = '';

      refreshGroups();
      loadView();
      refreshMpkAttachmentAccess();
      updateQrScanButton();
    })
    .withFailureHandler(showMainError)
    .getInitialData();
}

function getNextUpcomingSession_() {
  if (!DATA) return null;
  const today = DATA.sessionDate || new Date().toISOString().slice(0, 10);
  const upcoming = (DATA.details || [])
    .map(item => ({
      module: String(item.module || '').trim(),
      group: String(item.group || '').trim(),
      date: String(item.sessionDate || '').trim()
    }))
    .filter(item => item.module && item.group && /^\d{4}-\d{2}-\d{2}$/.test(item.date) && item.date >= today)
    .sort((a, b) => a.date.localeCompare(b.date) ||
      a.module.localeCompare(b.module) ||
      a.group.localeCompare(b.group, undefined, { numeric: true }));
  return upcoming.length ? upcoming[0] : null;
}

function updateQrScanButton() {
  const button = $('scanStudentBtn');
  if (!button) return;
  // Admin scanning is across all groups; only module and date are required.
  const ready = Boolean($('module').value && $('date').value);
  button.disabled = !ready;
  button.title = ready
    ? 'Scan students across all groups for the selected module and date. A specific group filter is optional.'
    : 'Select a module and session date first.';
}

function extractStudentIdFromQr(rawValue) {
  const text = String(rawValue || '').trim();
  if (!text) throw new Error('QR code is empty.');

  // Accept a plain student number or common URL/query-style QR payload.
  const labeled = text.match(/(?:STUDENT[_\s-]*ID|STUDENTID|ID)\s*[:=#/ ]\s*([0-9]{6,15})/i);
  if (labeled) return labeled[1];

  const query = text.match(/[?&](?:student[_-]?id|studentid|id)=([0-9]{6,15})(?:&|$)/i);
  if (query) return query[1];

  const plain = text.match(/^([0-9]{6,15})$/);
  if (plain) return plain[1];

  const numeric = text.match(/\b([0-9]{6,15})\b/);
  if (numeric) return numeric[1];

  throw new Error('QR code does not contain a valid student ID number.');
}


function openQrScanner() {
  const module = $('module').value;
  const group = $('group').value || '';
  const date = $('date').value;

  if (!module || !date) {
    alert('Please select a Module and Session Date first. Group is optional; All groups is allowed for administrators.');
    updateQrScanButton();
    return;
  }

  if (qrPopupWindow && !qrPopupWindow.closed) {
    qrPopupWindow.focus();
    return;
  }

  // The same token must be present in every scanner message. This prevents
  // stale scanner tabs from updating a different module/session.
  qrSessionToken = (window.crypto && crypto.randomUUID)
    ? crypto.randomUUID()
    : String(Date.now()) + '-' + Math.random().toString(36).slice(2);

  const scannerUrl =
    'https://mfaiz-alt.github.io/leadership-module-qr/' +
    '?module=' + encodeURIComponent(module) +
    '&date=' + encodeURIComponent(date) +
    '&group=' + encodeURIComponent(group) +
    '&token=' + encodeURIComponent(qrSessionToken);

  const width = 470;
  const height = 760;
  const left = Math.max(0, Math.round((screen.availWidth - width) / 2));
  const top = Math.max(0, Math.round((screen.availHeight - height) / 2));
  const features = [
    'popup=yes', 'resizable=yes', 'scrollbars=yes',
    'width=' + width, 'height=' + height,
    'left=' + left, 'top=' + top
  ].join(',');

  qrPopupWindow = window.open(
    scannerUrl,
    'LEADERSHIP_MODULE_QR_SCANNER',
    features
  );

  if (!qrPopupWindow) {
    qrSessionToken = '';
    alert('The scanner popup was blocked. Allow pop-ups for LEADERSHIP MODULE and try again.');
    return;
  }

  qrPopupWindow.focus();
  qrScanBusy = false;
}

function focusLeadershipModule_() {
  // Apps Script HTML Service runs inside an iframe. Focus the top-level
  // browser window as well as the iframe so the attendance form is visible.
  try { window.top.focus(); } catch (error) {}
  try { window.focus(); } catch (error) {}
  setTimeout(() => {
    try { window.top.focus(); } catch (error) {}
  }, 120);
}

function handleExternalQrMessage_(event) {
  if (event.origin !== 'https://mfaiz-alt.github.io') return;
  if (!qrPopupWindow || qrPopupWindow.closed) return;

  const data = event.data;
  if (!data || typeof data !== 'object' || data.token !== qrSessionToken) return;

  if (data.type === 'LEADERSHIP_MODULE_QR_CLOSED') {
    qrPopupWindow = null;
    qrSessionToken = '';
    qrScanBusy = false;
    return;
  }

  if (data.type !== 'LEADERSHIP_MODULE_QR_RESULT') return;

  // Acknowledge with the same token. The GitHub scanner requires this exact
  // acknowledgement before showing its paused/waiting state.
  try {
    event.source.postMessage({
      type: 'LEADERSHIP_MODULE_QR_RECEIVED',
      token: qrSessionToken,
      studentId: data.studentId
    }, event.origin);
  } catch (messageError) {}

  focusLeadershipModule_();

  if (
    data.module !== $('module').value ||
    data.date !== $('date').value ||
    String(data.group || '') !== String($('group').value || '')
  ) {
    alert('The selected module or session date has changed. Please close and reopen the scanner.');
    resumeExternalQrScanner_();
    return;
  }

  if (qrScanBusy) return;
  qrScanBusy = true;
  lookupScannedStudent_(String(data.studentId || '').trim());
}

function resumeExternalQrScanner_() {
  qrScanBusy = false;

  if (qrPopupWindow && !qrPopupWindow.closed && qrSessionToken) {
    try {
      qrPopupWindow.postMessage({
        type: 'LEADERSHIP_MODULE_QR_RESUME',
        token: qrSessionToken
      }, 'https://mfaiz-alt.github.io');
    } catch (error) {}

    setTimeout(() => {
      try {
        if (qrPopupWindow && !qrPopupWindow.closed) qrPopupWindow.focus();
      } catch (error) {}
    }, 220);
  } else {
    qrPopupWindow = null;
    qrSessionToken = '';
  }
}

function startQrCamera_() {
  $('qrScanStatus').textContent = 'Starting camera. Please allow camera access.';
  if (qrScanner) {
    stopQrCamera_().then(startQrCamera_);
    return;
  }

  qrScanner = new Html5Qrcode('qrReader');
  const config = { fps: 10, qrbox: { width: 245, height: 245 }, aspectRatio: 1.0 };

  qrScanner.start(
    { facingMode: 'environment' },
    config,
    decodedText => processScannedQr_(decodedText),
    () => {}
  ).then(() => {
    $('qrScanStatus').textContent = 'Camera ready. Point it at the QR code on the student card.';
  }).catch(error => {
    $('qrScanStatus').textContent =
      'Unable to start camera: ' + (error && error.message ? error.message : String(error)) +
      '. Check browser camera permission or enter the Student ID manually below.';
  });
}

function stopQrCamera_() {
  if (!qrScanner) return Promise.resolve();
  const instance = qrScanner;
  return instance.stop().catch(() => {}).then(() => {
    try { instance.clear(); } catch (error) {}
    if (qrScanner === instance) qrScanner = null;
  });
}

function closeQrScanner() {
  if (qrPopupWindow && !qrPopupWindow.closed) {
    try {
      qrPopupWindow.postMessage({
        type: 'LEADERSHIP_MODULE_QR_CLOSE',
        token: qrSessionToken
      }, 'https://mfaiz-alt.github.io');
      qrPopupWindow.close();
    } catch (error) {}
  }
  qrPopupWindow = null;
  qrSessionToken = '';
  qrScanBusy = false;

  const modal = $('qrScannerModal');
  if (modal) {
    modal.classList.add('hidden');
    modal.setAttribute('aria-hidden', 'true');
  }
  stopQrCamera_();
}

function processScannedQr_(rawValue) {
  if (qrScanBusy) return;
  qrScanBusy = true;

  let studentId;
  try {
    studentId = extractStudentIdFromQr(rawValue);
  } catch (error) {
    qrScanBusy = false;
    $('qrScanStatus').textContent = error.message || String(error);
    return;
  }

  $('qrScanStatus').textContent = 'Checking Student ID ' + studentId + ' against the selected session…';
  stopQrCamera_().then(() => lookupScannedStudent_(studentId));
}

function lookupScannedStudent_(studentId) {
  const payload = {
    studentId: studentId,
    module: $('module').value,
    group: $('group').value || '',
    sessionDate: $('date').value
  };

  if (!studentId) {
    alert('The scanner did not return a Student ID.');
    resumeExternalQrScanner_();
    return;
  }

  google.script.run
    .withSuccessHandler(result => {
      if (result && result.groupMismatch) {
        const currentGroup = result.currentGroup || 'another group';
        const moveStudent = confirm(
          'This student is already listed under ' + currentGroup +
          ', but the selected filter is Group ' + payload.group + '.\\n\\n' +
          'Move the student to Group ' + payload.group + '?\\n\\n' +
          'For current/future sessions, NAME_LIST has one shared GROUP field, so this may affect the student’s other modules.\\n\\n' +
          'Choose Cancel to keep the current group and return to scanning.'
        );

        if (!moveStudent) {
          resumeExternalQrScanner_();
          return;
        }

        google.script.run
          .withSuccessHandler(moveResult => {
            if (moveResult && moveResult.warning) alert(moveResult.warning);
            lookupScannedStudent_(studentId);
          })
          .withFailureHandler(error => {
            alert(error.message || String(error));
            resumeExternalQrScanner_();
          })
          .moveScannedStudentToSessionGroup(payload);
        return;
      }

      if (result && result.found && result.student) {
        attendance = result.attendance || {};
        openModal(encodeURIComponent(JSON.stringify(result.student)).replace(/'/g, '%27'));
        return;
      }

      // No confirmation prompt: open the registration form immediately when
      // the ID is not enrolled in the selected module/session.
      const profile = result && result.profile ? result.profile : { id: studentId };
      const allowedGroups = result && Array.isArray(result.allowedGroups)
        ? result.allowedGroups : [];
      openScannedStudentRegistration(studentId, profile, allowedGroups);
    })
    .withFailureHandler(error => {
      alert(error.message || String(error));
      resumeExternalQrScanner_();
    })
    .scanStudentForSession(payload);
}

function lookupManualQrStudent() {
  const studentId = String($('manualQrStudentId').value || '').trim();
  if (!studentId) {
    $('qrScanStatus').textContent = 'Enter a Student ID first.';
    return;
  }
  processScannedQr_(studentId);
}

function openScannedStudentRegistration(studentId, profile, allowedGroups) {
  focusLeadershipModule_();
  pendingScanRegistration = true;
  pendingScannedStudentId = studentId;
  $('newStudentID').value = studentId;
  $('newStudentEmail').value = profile.email || '';
  $('newStudentName').value = profile.name || '';
  $('newProgramme').value = profile.programme || '';
  $('newSemester').value = profile.semester || '';
  $('newGender').value = profile.gender || '';
  populateNewStudentGroupOptions_($('group').value, allowedGroups);

  $('addStudentModal').querySelector('h2').textContent = 'Add Student to Group';
  $('addStudentContext').textContent =
    'Student ID: ' + studentId + ' | Module: ' + $('module').value +
    ' | Date: ' + $('date').value +
    ($('group').value ? ' | Filter: Group ' + $('group').value : ' | Filter: All groups — select the destination group below');
  $('addStudentError').textContent = '';
  $('addStudentModal').classList.remove('hidden');
  focusLeadershipModule_();
}

function finishScannedStudentAddition(studentId) {
  // Registration is complete: close the form, refresh the roster and return
  // directly to the waiting scanner. Do not open a second attendance modal.
  pendingScanRegistration = false;
  pendingScannedStudentId = '';
  $('addStudentModal').classList.add('hidden');
  const heading = $('addStudentModal').querySelector('h2');
  if (heading) heading.textContent = 'Add Student to Group';

  google.script.run
    .withSuccessHandler(data => {
      DATA = data || DATA;
      refreshGroups();
      loadView();
      resumeExternalQrScanner_();
    })
    .withFailureHandler(error => {
      alert('Student was registered, but the directory refresh failed: ' + (error.message || String(error)));
      resumeExternalQrScanner_();
    })
    .getInitialData();
}

function showMainError(error) {
  $('students').innerHTML =
    '<div class="empty">' + esc(error && error.message ? error.message : error) + '</div>';
}

function refreshGroups() {
  if (!DATA) return;

  const module = $('module').value;
  const date = $('date').value;
  const previous = $('group').value;
  const hasScheduledDates = (DATA.details || []).some(item => item.sessionDate);

  const currentGroups = (DATA.students || [])
    .filter(student => {
      const matchesModule = !module ||
        student.modules.some(studentModule => moduleMatches(studentModule, module));
      const matchesDate = !date || !hasScheduledDates ||
        (student.assignments || []).some(item =>
          (!module || moduleMatches(item.module, module)) &&
          String(item.sessionDate || '') === date
        );
      return matchesModule && matchesDate;
    })
    .map(student => student.group)
    .filter(Boolean);

  const archivedGroups = (DATA.historicalSessions || [])
    .filter(session => !module || moduleMatches(session.module, module))
    .map(session => session.group)
    .filter(Boolean);

  const groups = [...new Map(
    [...currentGroups, ...archivedGroups].map(value => [normalizeGroup(value), value])
  ).values()].sort((a, b) =>
    a.localeCompare(b, undefined, { numeric: true })
  );

  $('group').innerHTML = '<option value="">All groups</option>';

  groups.forEach(group => {
    $('group').add(new Option(group, group));
  });

  const matchingPrevious = groups.find(group =>
    normalizeGroup(group) === normalizeGroup(previous)
  );

  if (matchingPrevious) {
    $('group').value = matchingPrevious;
  }
}

function sortDirectory(field) {
  if (directorySort.key === field) directorySort.dir *= -1;
  else directorySort = { key: field, dir: 1 };
  renderDirectory();
}

function initials(value) {
  return String(value || '?').trim().split(/\s+/).slice(0, 2)
    .map(part => part[0] || '').join('').toUpperCase();
}

function updateDirectorySelect_(id, values, allLabel) {
  const element = $(id);
  const previous = element.value;
  const unique = [...new Set((values || []).map(value => String(value || '').trim()).filter(Boolean))]
    .sort((a, b) => a.localeCompare(b));
  element.innerHTML = '';
  element.add(new Option(allLabel, ''));
  unique.forEach(value => element.add(new Option(value, value)));
  element.value = unique.includes(previous) ? previous : '';
}

function renderDirectory(detailsOverride) {
  if (!DATA) return;
  if (Array.isArray(detailsOverride)) {
    directoryDetailsOverride = detailsOverride;
  } else if (!isViewingArchive) {
    directoryDetailsOverride = null;
  }

  const module = $('module').value;
  const group = $('group').value;
  const date = $('date').value;
  const sourceDetails = directoryDetailsOverride || (DATA.details || []);
  const hasScheduledDates = !isViewingArchive && sourceDetails.some(item => item.sessionDate);

  let baseRows = sourceDetails.filter(item =>
    (!module || moduleMatches(item.module, module)) &&
    (!group || normalizeGroup(item.group) === normalizeGroup(group)) &&
    (!hasScheduledDates || !date || String(item.sessionDate || '') === date)
  );

  updateDirectorySelect_(
    'directoryFacilitator',
    baseRows.map(item => item.facilitator),
    'All facilitators'
  );
  updateDirectorySelect_(
    'directoryAssistant',
    baseRows.map(item => item.assistant),
    'All assistants'
  );

  const search = key($('directorySearch').value);
  const selectedFacilitator = key($('directoryFacilitator').value);
  const selectedAssistant = key($('directoryAssistant').value);

  let rows = baseRows.filter(item => {
    const matchFacilitator = !selectedFacilitator || key(item.facilitator) === selectedFacilitator;
    const matchAssistant = !selectedAssistant || key(item.assistant) === selectedAssistant;
    const searchable = [
      item.facilitator, item.staffId, item.assistant,
      item.module, item.group, item.venue, item.time
    ].map(key).join(' ');
    const matchSearch = !search || searchable.includes(search);
    return matchFacilitator && matchAssistant && matchSearch;
  });

  const sortValue = item => String(item[directorySort.key] || '').toLocaleLowerCase();
  rows.sort((a, b) =>
    directorySort.dir * sortValue(a).localeCompare(sortValue(b), undefined, { numeric: true }) ||
    String(a.assistant || '').localeCompare(String(b.assistant || '')) ||
    String(a.module || '').localeCompare(String(b.module || '')) ||
    String(a.group || '').localeCompare(String(b.group || ''), undefined, { numeric: true })
  );

  const columns = [
    ['facilitator', 'FACILITATOR'],
    ['assistant', 'ASSISTANT'],
    ['module', 'MODULE'],
    ['group', 'GROUP'],
    ['venue', 'VENUE'],
    ['time', 'TIME']
  ];

  $('directoryCount').textContent = rows.length + ' assignments';
  $('directoryTable').innerHTML = rows.length ? `
    <div class="table-wrap">
      <table class="directory-table">
        <thead><tr>
          ${columns.map(([field, label]) => `
            <th><button class="sort-btn" onclick="sortDirectory('${field}')">
              ${label}
              <span>${directorySort.key === field ? (directorySort.dir === 1 ? '↑' : '↓') : '↕'}</span>
            </button></th>
          `).join('')}
        </tr></thead>
        <tbody>
          ${rows.map(item => `
            <tr>
              <td><div class="person-cell">
                <span class="avatar faci">${esc(initials(item.facilitator))}</span>
                <span><b>${esc(item.facilitator || 'Not assigned')}</b>
                <small>${esc(item.staffId || 'FACI')}</small></span>
              </div></td>
              <td><div class="person-cell">
                <span class="avatar assistant">${esc(initials(item.assistant))}</span>
                <span><b>${esc(item.assistant || 'Not assigned')}</b><small>Assistant</small></span>
              </div></td>
              <td><span class="module-tag">${esc(item.module)}</span></td>
              <td><span class="group-pill">${esc(item.group)}</span></td>
              <td><span class="venue-text">⌖ ${esc(item.venue || 'Venue TBC')}</span></td>
              <td><span class="schedule-time">${esc(item.time || 'Time not set')}</span></td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    </div>
  ` : '<div class="empty">No facilitator or assistant assignments match these filters.</div>';
}

function getFilteredStudents() {
  if (!DATA) return [];

  const module = $('module').value;
  const group = $('group').value;
  const date = $('date').value;
  const hasScheduledDates = (DATA.details || []).some(item => item.sessionDate);

  return (DATA.students || []).filter(student => {
    const matchesModule = !module ||
      student.modules.some(studentModule => moduleMatches(studentModule, module));
    const matchesGroup = !group ||
      normalizeGroup(student.group) === normalizeGroup(group);
    const matchesScheduledDate = !date || !hasScheduledDates ||
      (student.assignments || []).some(item =>
        (!module || moduleMatches(item.module, module)) &&
        String(item.sessionDate || '') === date
      );
    return matchesModule && matchesGroup && matchesScheduledDate;
  });
}

function updateSummary(students, module, date) {
  const uniqueGroups = [...new Set((students || []).map(student => student.group).filter(Boolean))];
  $('summary').innerHTML = `
    <div class="stat"><span class="stat-icon">♙</span><strong>${(students || []).length}</strong><span>Students found</span></div>
    <div class="stat"><span class="stat-icon">▦</span><strong>${uniqueGroups.length}</strong><span>Groups</span></div>
    <div class="stat"><span class="stat-icon">◈</span><strong>${module ? 1 : '—'}</strong><span>Modules selected</span></div>
    <div class="stat"><span class="stat-icon">◷</span><strong>${date ? new Date(date + 'T00:00:00').toLocaleDateString() : '—'}</strong><span>Session date</span></div>
  `;
}

/** Keep the front dashboard clear until a specific module and session date are selected. */
function showSessionSelectionPrompt_(module, date) {
  isViewingArchive = false;
  attendance = {};
  $('archiveNotice').classList.add('hidden');
  updateSummary([], module, date);

  const message = !module && !date
    ? 'No upcoming session date is configured yet. Add SESSION_DATE to the relevant DETAILS rows, or select a module and date manually. Past sessions remain searchable by date.'
    : !module
      ? 'Select a module to view the student list for this session date.'
      : 'Select a session date to view current or archived student records.';

  $('directoryCount').textContent = '0 assignments';
  $('directoryTable').innerHTML = '<div class="empty">' + message + '</div>';
  $('count').textContent = '0 students';
  $('students').innerHTML = '<div class="empty">' + message + '</div>';
}

function loadView() {
  if (!DATA) return;
  const requestId = ++viewRequest;
  const module = $('module').value;
  const group = $('group').value;
  const date = $('date').value;
  const students = getFilteredStudents();
  const todayKey = DATA.sessionDate || '';

  // No student roster or assignment directory is shown on the landing dashboard.
  // Users must choose a module and a date; past dates load from SESSION_ARCHIVE.
  if (!module || !date) {
    showSessionSelectionPrompt_(module, date);
    return;
  }

  const isPastDate = Boolean(todayKey && date < todayKey);
  isViewingArchive = false;
  $('archiveNotice').classList.add('hidden');
  updateSummary(students, module, date);
  renderDirectory();

  const matchingUpcomingDetails = (DATA.details || []).filter(item =>
    moduleMatches(item.module, module) &&
    String(item.sessionDate || '') === date &&
    (!group || normalizeGroup(item.group) === normalizeGroup(group))
  );
  if (!isPastDate && matchingUpcomingDetails.length) {
    $('archiveNotice').textContent =
      'UPCOMING / CURRENT SESSION · ' + date + ' · ' + module +
      ' · ' + matchingUpcomingDetails.length + ' group assignment(s)';
    $('archiveNotice').classList.remove('hidden');
  }

  if (isPastDate) {
    $('students').innerHTML = '<div class="loading">Retrieving archived session…</div>';
    $('directoryTable').innerHTML = '<div class="loading">Retrieving archived facilitator and venue details…</div>';

    google.script.run
      .withSuccessHandler(result => {
        if (requestId !== viewRequest) return;
        isViewingArchive = true;
        attendance = (result && result.attendance) || {};
        const archivedStudents = (result && result.students) || [];
        updateSummary(archivedStudents, module, date);
        renderDirectory((result && result.details) || []);
        $('archiveNotice').textContent =
          'ARCHIVED SESSION · EDITABLE BY ASSIGNED FACILITATOR / ASSISTANT · ' + date + ' · ' + module;
        $('archiveNotice').classList.remove('hidden');
        if (!archivedStudents.length) {
          $('count').textContent = '0 students';
          $('students').innerHTML =
            '<div class="empty">No historical roster or attendance records were found for this module, group and date.</div>';
          return;
        }
        renderStudents(archivedStudents);
      })
      .withFailureHandler(error => {
        if (requestId !== viewRequest) return;
        isViewingArchive = false;
        showMainError(error);
      })
      .getSessionArchive(module, group, date);
    return;
  }

  $('students').innerHTML = '<div class="loading">Loading attendance…</div>';
  google.script.run
    .withSuccessHandler(records => {
      if (requestId !== viewRequest) return;
      attendance = records || {};
      renderStudents(students);
    })
    .withFailureHandler(error => {
      if (requestId !== viewRequest) return;
      showMainError(error);
    })
    .getAttendance(module, group, date);
}

function renderStudents(list) {
  if (!DATA) return;
  const selectedModule = $('module').value;
  const selectedDate = $('date').value;
  if (!selectedModule || !selectedDate) {
    showSessionSelectionPrompt_(selectedModule, selectedDate);
    return;
  }
  const query = key($('search').value);
  const selectedStatus = $('statusFilter').value;
  const module = $('module').value;
  const groupSelected = $('group').value;
  const date = $('date').value;
  const canUpdate = Boolean(module && date);

  const filtered = list.filter(student => {
    const matchesSearch = !query ||
      key(student.name).includes(query) || key(student.id).includes(query);
    const record = attendance[String(student.id)] || {};
    const status = record.status || 'NOT UPDATED';
    const matchesStatus = !selectedStatus || status === selectedStatus;
    return matchesSearch && matchesStatus;
  });

  $('count').textContent = filtered.length + ' students';
  if (!filtered.length) {
    $('students').innerHTML = '<div class="empty">No students match your filters.</div>';
    return;
  }

  $('students').innerHTML = filtered.map((student, index) => {
    const placements = (student.assignments || [])
  .filter(item => !module || moduleMatches(item.module, module));
    const record = attendance[String(student.id)] || {};
    const status = record.status || 'NOT UPDATED';
    const statusClass = status.toLowerCase().replace(/[^a-z0-9]+/g, '-');

    const placementHtml = placements.map(item => `
      <div class="placement-row">
        <div class="placement-module">
          <span class="mini-label">MODULE</span>
          <b>${esc(item.module)}</b>
          <span class="group-pill">Group ${esc(item.group)}</span>
        </div>
        <div class="placement-venue">
          <span class="venue-pin">⌖</span>
          <div><span class="mini-label">ASSIGNED VENUE</span>
          <b>${esc(item.venue || 'Venue not assigned')}</b></div>
        </div>
        <div class="placement-staff">
  <div>
    <span class="mini-label">SESSION TIME</span>
    <b>${esc(item.time || 'Time not set')}</b>
  </div>

  <div>
    <span class="mini-label">FACILITATOR</span>
    <b>${esc(item.facilitator || 'Not assigned')}</b>
  </div>

  <div>
    <span class="mini-label">ASSISTANT</span>
    <b>${esc(item.assistant || 'Not assigned')}</b>
  </div>
</div>
      </div>
    `).join('');

    return `
      <article class="student-card student-placement-card">
        <div class="student-number">${index + 1}</div>
        <div class="student-main">
          <div class="student-name">${esc(student.name)}</div>
          <div class="student-meta">
            ID: ${esc(student.id)} · ${esc(student.programme)} ·
            Semester ${esc(student.semester)} · Group ${esc(student.group)}
          </div>
        </div>
        <span class="status status-${statusClass}">${esc(status)}</span>
        <div class="student-placements">
          ${placementHtml || '<div class="no-placement">No venue assignment found. Please check the DETAILS sheet.</div>'}
        </div>
        <div class="student-card-actions">
          ${canUpdate ? `
            <button class="update" onclick="openModal('${encodeURIComponent(JSON.stringify(student)).replace(/'/g, '%27')}')">
              ✎ ${isViewingArchive ? 'Update archived attendance' : 'Update attendance'}
            </button>
          ` : `
            <button class="update" disabled title="Select a module, group and session date first">
              Select module, group &amp; date first
            </button>
          `}
        </div>
      </article>
    `;
  }).join('');
}

function openModal(encodedStudent) {
  // Apps Script is framed; focus the top-level attendance window.
  focusLeadershipModule_();
  selectedStudent = JSON.parse(decodeURIComponent(encodedStudent));
  $('modalName').textContent = selectedStudent.name;
  $('modalMeta').textContent =
    selectedStudent.id + ' · ' + $('module').value +
    ' · Group ' + selectedStudent.group + ' · ' + $('date').value;

  const record = attendance[String(selectedStudent.id)] || {};
  $('status').value = record.status || 'REGISTERED';
  $('remarks').value = record.remarks || '';
  $('modalError').textContent = '';
  updateRemarksHint();
  $('modal').classList.remove('hidden');
  focusLeadershipModule_();
}

function closeModal(resumeScanner = true) {
  $('modal').classList.add('hidden');
  selectedStudent = null;
  if (resumeScanner) resumeExternalQrScanner_();
}

function updateRemarksHint() {
  $('remarks').placeholder = $('status').value === 'REGISTERED'
    ? 'Optional remarks…'
    : 'Enter explanation or relevant details…';
}

function save() {
  if (!$('date').value) {
  $('modalError').textContent = 'Please select a session date before saving attendance.';
  return;
}
  if (!selectedStudent) return;
  const module = $('module').value;
  const group = String(selectedStudent.group || $('group').value || '').trim();
  const date = $('date').value;
  const payload = {
    studentId: String(selectedStudent.id),
    module: module,
    group: group,
    sessionDate: date,
    status: $('status').value,
    remarks: $('remarks').value
  };

  $('save').disabled = true;
  $('modalError').textContent = 'Saving attendance…';

  google.script.run
    .withSuccessHandler(result => {
      if (!result || result.success !== true) {
        $('save').disabled = false;
        $('modalError').textContent = 'Attendance save was not confirmed.';
        return;
      }
      $('modalError').textContent = 'Saved. Verifying attendance…';

      google.script.run
        .withSuccessHandler(records => {
          attendance = records || {};
          const saved = attendance[String(selectedStudent.id)];
          if (!saved || key(saved.status) !== key(payload.status)) {
            $('save').disabled = false;
            $('modalError').textContent =
              'The save request completed, but the saved status could not be verified. Please check the ATTENDANCE sheet.';
            return;
          }
          $('save').disabled = false;
          closeModal(false);
          loadView();
          if (result.message) alert(result.message);
          resumeExternalQrScanner_();
        })
        .withFailureHandler(error => {
          $('save').disabled = false;
          $('modalError').textContent =
            'Saved, but refresh failed: ' + (error.message || String(error));
        })
        .getAttendance(module, group, date);
    })
    .withFailureHandler(error => {
      $('save').disabled = false;
      $('modalError').textContent = error.message || String(error);
    })
    .updateStudentStatus(payload);
}

function populateNewStudentGroupOptions_(preferredGroup, allowedGroupsOverride) {
  const select = $('newStudentGroup');
  if (!select) return;

  const module = $('module').value;
  const date = $('date').value;
  const details = (DATA && DATA.details) || [];
  const hasServerGroupScope = Array.isArray(allowedGroupsOverride);
  let groups = hasServerGroupScope
    ? allowedGroupsOverride.slice()
    : details.filter(item =>
        moduleMatches(item.module, module) &&
        (!item.sessionDate || item.sessionDate === date)
      ).map(item => String(item.group || '').trim()).filter(Boolean);

  // For the ordinary Add Student button, fall back to all configured groups
  // when DETAILS has no date-specific row. QR registration uses the server's
  // authorized group list and must not widen that list in the browser.
  if (!hasServerGroupScope) {
    if (!groups.length) {
      groups = details.filter(item => moduleMatches(item.module, module))
        .map(item => String(item.group || '').trim()).filter(Boolean);
    }
    (DATA && DATA.historicalSessions || []).forEach(item => {
      if (moduleMatches(item.module, module) && item.date === date) {
        groups.push(String(item.group || '').trim());
      }
    });
  }

  groups = [...new Map(groups.filter(Boolean).map(group => [normalizeGroup(group), group])).values()]
    .sort((a, b) => a.localeCompare(b, undefined, { numeric: true }));

  select.innerHTML = '<option value="">Select destination group</option>';
  groups.forEach(group => select.add(new Option(group, group)));

  const wanted = preferredGroup || $('group').value || '';
  const match = groups.find(group => normalizeGroup(group) === normalizeGroup(wanted));
  select.value = match || '';
}

function openAddStudent() {
  pendingScanRegistration = false;
  pendingScannedStudentId = '';

  const module = $('module').value;
  const date = $('date').value;
  if (!module || !date) {
    alert('Please select a module and session date first.');
    return;
  }

  $('addStudentContext').textContent =
    'Module: ' + module + ' | Date: ' + date +
    ($('group').value ? ' | Main filter: Group ' + $('group').value : ' | Main filter: All groups');

  $('addStudentError').textContent = '';
  [
    'newStudentID', 'newStudentEmail', 'newStudentName',
    'newProgramme', 'newSemester', 'newGender'
  ].forEach(id => {
    if ($(id)) $(id).value = '';
  });

  populateNewStudentGroupOptions_($('group').value);
  $('addStudentModal').classList.remove('hidden');
}

function closeAddStudent() {
  $('addStudentModal').classList.add('hidden');

  if (pendingScanRegistration) {
    pendingScanRegistration = false;
    pendingScannedStudentId = '';
    const heading = $('addStudentModal').querySelector('h2');
    if (heading) heading.textContent = 'Add Student to Group';
    resumeExternalQrScanner_();
  }
}

function saveNewStudent() {
  const payload = {
    studentId: $('newStudentID').value.trim(),
    name: $('newStudentName').value.trim(),
    programme: $('newProgramme').value.trim(),
    semester: $('newSemester').value.trim(),
    gender: $('newGender').value,
    email: $('newStudentEmail').value.trim(),
    module: $('module').value,
    group: ($('newStudentGroup') && $('newStudentGroup').value) || $('group').value,
    sessionDate: $('date').value
  };

  if (!payload.module || !payload.group || !payload.sessionDate) {
    $('addStudentError').textContent =
      'Select module, session date and the destination group in this form.';
    return;
  }

  $('saveNewStudent').disabled = true;
  $('addStudentError').textContent = pendingScanRegistration
    ? 'Adding scanned student to the selected session…'
    : 'Adding student…';

  const isPastSession = Boolean(DATA && DATA.sessionDate && payload.sessionDate < DATA.sessionDate);
  if (pendingScanRegistration || isPastSession) {
    google.script.run
      .withSuccessHandler(result => {
        $('saveNewStudent').disabled = false;
        if (!result || !result.success) {
          $('addStudentError').textContent = 'Student registration could not be confirmed.';
          return;
        }

        if (pendingScanRegistration) {
          const id = payload.studentId;
          const heading = $('addStudentModal').querySelector('h2');
          if (heading) heading.textContent = 'Add Student to Group';
          finishScannedStudentAddition(id);
          return;
        }

        closeAddStudent();
        loadView();
      })
      .withFailureHandler(error => {
        $('saveNewStudent').disabled = false;
        $('addStudentError').textContent = error.message || String(error);
      })
      .addScannedStudentToSession(payload);
    return;
  }

  google.script.run
    .withSuccessHandler(result => {
      $('saveNewStudent').disabled = false;
      if (!result || !result.success) {
        $('addStudentError').textContent = 'Student registration could not be confirmed.';
        return;
      }

      closeAddStudent();
      google.script.run
        .withSuccessHandler(data => {
          DATA = data;
          refreshGroups();
          loadView();
        })
        .withFailureHandler(error => {
          alert('Student added, but refresh failed: ' + (error.message || String(error)));
        })
        .getInitialData();
    })
    .withFailureHandler(error => {
      $('saveNewStudent').disabled = false;
      $('addStudentError').textContent = error.message || String(error);
    })
    .addStudentToGroup(payload);
}

/* =========================================================
   DOWNLOADS - PHYSICAL PORTRAIT DOCUMENT FLIP FILE
   One realistic illustrative document per portrait page.
   ========================================================= */
const FAIL_TEMUDUGA_ITEMS = [
  { name: 'RESUME / CV', type: 'cv' },
  { name: 'SALINAN IC', type: 'identity' },
  { name: 'SIJIL KELAHIRAN (ASAL)', type: 'birth' },
  { name: 'SIJIL BERHENTI SEKOLAH', type: 'school' },
  { name: 'SIJIL STPM', type: 'certificate-stpm' },
  { name: 'TRANSKRIP MATRIKULASI', type: 'transcript-matric' },
  { name: 'SIJIL DIPLOMA', type: 'certificate-diploma' },
  { name: 'TRANSKRIP DIPLOMA', type: 'transcript-diploma' },
  { name: 'SIJIL IJAZAH SARJANA MUDA', type: 'certificate-degree' },
  { name: 'TRANSKRIP IJAZAH SARJANA MUDA', type: 'transcript-degree' },
  { name: 'LESEN-LESEN BERDAFTAR (IKUT KELAYAKAN/BIDANG)', type: 'licences' },
  { name: 'SIJIL BAHASA (ENGLISH/MANDARIN/LAIN-LAIN)', type: 'language' },
  { name: 'SIJIL PENGIKTIRAFAN KELAYAKAN BIASISWA/TAJAAN', type: 'scholarship' },
  { name: 'SIJIL KEPUJIAN DEKAN', type: 'dean' },
  { name: 'SIJIL AKADEMIK / ANUGERAH DEKAN /ANUGERAH BERKAITAN', type: 'awards' },
  { name: 'SIJIL PENGLIBATAN KOKURIKULUM (UTAMAKAN YANG BERJAWATAN)', type: 'cocurricular' }
];

let failTemudugaPageIndex = 0;
let failTemudugaAnimating = false;
let failTemudugaFlipHandler = null;
let failTemudugaFlipTimer = null;

function refreshMpkAttachmentAccess() {
  const button = $('downloadsBtn');
  if (!button) return;

  // Keep hidden until the server verifies the signed-in account.
  button.classList.add('hidden');
  button.disabled = true;

  google.script.run
    .withSuccessHandler(access => {
      button.disabled = false;
      if (access && access.allowed === true) {
        button.classList.remove('hidden');
        button.title = 'MPK ATTACHMENT – ' + access.role;
      } else {
        button.classList.add('hidden');
      }
    })
    .withFailureHandler(error => {
      button.disabled = false;
      button.classList.add('hidden');
      console.error('MPK attachment access check failed:', error);
    })
    .getMpkAttachmentAccess();
}

function showDownloadsModal_() {
  $('downloadsModal').classList.remove('hidden');
  $('downloadsModal').setAttribute('aria-hidden', 'false');
  $('downloadsHome').classList.remove('hidden');
  $('failTemudugaBook').classList.add('hidden');
  $('realBinder').classList.remove('is-open', 'turn-next', 'turn-prev');
}

function openDownloads() {
  const button = $('downloadsBtn');
  if (button) button.disabled = true;

  // Recheck on every open; hiding the button is not the permission check.
  google.script.run
    .withSuccessHandler(access => {
      if (button) button.disabled = false;

      if (!access || access.allowed !== true) {
        alert('Access denied. MPK ATTACHMENT is restricted to authorised users.');
        closeDownloads();
        return;
      }

      showDownloadsModal_();
    })
    .withFailureHandler(error => {
      if (button) button.disabled = false;
      alert('Unable to verify MPK attachment access: ' +
        (error && error.message ? error.message : String(error)));
    })
    .getMpkAttachmentAccess();
}

function closeDownloads() {
  $('downloadsModal').classList.add('hidden');
  $('downloadsModal').setAttribute('aria-hidden', 'true');
  $('realBinder').classList.remove('is-open', 'turn-next', 'turn-prev');
}

function openFailTemuduga() {
  google.script.run
    .withSuccessHandler(access => {
      if (!access || access.allowed !== true) {
        alert('Access denied. You are not listed for MODUL PERKEMBANGAN KERJAYA.');
        closeDownloads();
        return;
      }

      failTemudugaPageIndex = 0;
      failTemudugaAnimating = false;
      $('downloadsHome').classList.add('hidden');
      $('failTemudugaBook').classList.remove('hidden');
      $('realBinder').classList.remove('is-open', 'turn-next', 'turn-prev');
      renderPhysicalPage();
    })
    .withFailureHandler(error => {
      alert('Unable to verify MPK attachment access: ' +
        (error && error.message ? error.message : String(error)));
      closeDownloads();
    })
    .getMpkAttachmentAccess();
}

function openPhysicalBinder() {
  $('realBinder').classList.add('is-open');
}

function docHeader(label) {
  return `
    <div class="document-brand-row">
      <div class="document-brand-mark">UKP</div>
      <div class="document-brand-copy">
        <b>UNIT KEPIMPINAN PELAJAR</b>
        <span>UiTM CAWANGAN PULAU PINANG · KAMPUS BERTAM</span>
      </div>
    </div>
    <div class="document-rule"></div>
    <div class="document-kicker">CONTOH SUSUN ATUR · ${esc(label)}</div>
  `;
}

function signatureBlock(leftLabel, rightLabel) {
  return `
    <div class="document-signatures">
      <div><span></span><small>${esc(leftLabel)}</small></div>
      <div><span></span><small>${esc(rightLabel)}</small></div>
    </div>
  `;
}

function certificateBody(title, qualification, note) {
  return `
    <div class="certificate-document">
      <div class="certificate-frame">
        <div class="certificate-topline">UNIVERSITI / INSTITUSI PENGANUGERAH</div>
        <div class="certificate-seal">CONTOH</div>
        <div class="certificate-heading">${esc(title)}</div>
        <div class="certificate-small">DENGAN INI DIPERAKUKAN BAHAWA</div>
        <div class="certificate-recipient">NAMA PENERIMA</div>
        <div class="certificate-small">NO. KAD PENGENALAN: XXXX-XX-XXXX</div>
        <p class="certificate-copy">telah memenuhi syarat dan berjaya menamatkan</p>
        <div class="certificate-qualification">${esc(qualification)}</div>
        <p class="certificate-copy">${esc(note)}</p>
        <div class="certificate-date">Dikeluarkan pada: [TARIKH] · [TAHUN]</div>
        ${signatureBlock('PENDAFTAR / PEGAWAI', 'WAKIL INSTITUSI')}
      </div>
    </div>
  `;
}

function transcriptBody(level) {
  const rows = [
    ['KOD 101', 'MATA PELAJARAN A', 'A'],
    ['KOD 102', 'MATA PELAJARAN B', 'A-'],
    ['KOD 103', 'MATA PELAJARAN C', 'B+'],
    ['KOD 104', 'MATA PELAJARAN D', 'B'],
    ['KOD 105', 'MATA PELAJARAN E', 'A']
  ];
  return `
    ${docHeader('TRANSKRIP AKADEMIK')}
    <h2 class="document-main-title">TRANSKRIP AKADEMIK</h2>
    <div class="transcript-student">
      <div><small>NAMA PELAJAR</small><b>NAMA PENUH PELAJAR</b></div>
      <div><small>NO. PELAJAR</small><b>XXXXXXXX</b></div>
      <div><small>PROGRAM</small><b>${esc(level)}</b></div>
      <div><small>SESI AKADEMIK</small><b>20XX / 20XX</b></div>
    </div>
    <table class="transcript-table">
      <thead><tr><th>KOD</th><th>NAMA KURSUS</th><th>GRED</th></tr></thead>
      <tbody>${rows.map(row => `<tr><td>${row[0]}</td><td>${row[1]}</td><td>${row[2]}</td></tr>`).join('')}</tbody>
    </table>
    <div class="transcript-summary">
      <div><small>JUMLAH KREDIT</small><b>XX</b></div>
      <div><small>PNG / GPA</small><b>X.XX</b></div>
      <div><small>STATUS</small><b>BERJAYA</b></div>
    </div>
    <p class="document-note">Nota: Maklumat di atas ialah contoh ilustrasi dan bukan rekod akademik sebenar.</p>
    ${signatureBlock('PEGAWAI AKADEMIK', 'PENDAFTAR')}
  `;
}

function documentBody(item, index) {
  switch (item.type) {
    case 'cv':
      return `
        <div class="cv-document">
          <div class="cv-top">
            <div class="cv-photo">GAMBAR<br>PASPORT</div>
            <div class="cv-name"><h2>NAMA PENUH CALON</h2><p>PROFIL KERJAYA · BIDANG / JAWATAN DIPOHON</p><small>emel@example.com · 01X-XXX XXXX · Malaysia</small></div>
          </div>
          <section class="cv-section"><h3>PROFIL RINGKAS</h3><p>Individu yang bermotivasi, berdisiplin dan komited untuk menyumbang kemahiran serta pengalaman dalam organisasi.</p></section>
          <section class="cv-section"><h3>PENDIDIKAN</h3><div class="cv-entry"><b>Nama Kelayakan / Program</b><span>Nama Institusi · 20XX – 20XX</span><small>Keputusan / PNGK: X.XX</small></div><div class="cv-entry"><b>Kelayakan Terdahulu</b><span>Nama Institusi · 20XX – 20XX</span></div></section>
          <section class="cv-section"><h3>PENGALAMAN / LATIHAN</h3><div class="cv-entry"><b>Jawatan / Peranan</b><span>Organisasi · Tempoh</span><small>• Skop tugas dan pencapaian utama</small><small>• Kemahiran atau tanggungjawab berkaitan</small></div></section>
          <section class="cv-section"><h3>KEMAHIRAN</h3><div class="cv-tags"><span>Komunikasi</span><span>Kerja Berpasukan</span><span>Pengurusan</span><span>Komputer</span></div></section>
          <section class="cv-section"><h3>RUJUKAN</h3><p>Disediakan atas permintaan.</p></section>
        </div>
      `;
    case 'identity':
      return `
        ${docHeader('SALINAN KAD PENGENALAN')}
        <h2 class="document-main-title">SALINAN KAD PENGENALAN</h2>
        <p class="document-intro">Sertakan salinan bahagian hadapan dan belakang kad pengenalan.</p>
        <div class="ic-card ic-front">
          <div class="ic-header">KAD PENGENALAN · CONTOH ILUSTRASI</div>
          <div class="ic-content"><div class="ic-photo">FOTO</div><div class="ic-fields"><b>NAMA PEMEGANG</b><strong>XXXXXX-XX-XXXX</strong><span>WARGANEGARA</span><span>TEMPAT LAHIR: [TEMPAT]</span></div></div>
          <div class="ic-bottom">CONTOH VISUAL · BUKAN KAD PENGENALAN</div>
        </div>
        <div class="ic-card ic-back">
          <div class="ic-header">BAHAGIAN BELAKANG · CONTOH</div>
          <div class="ic-back-lines"><i></i><i></i><i></i></div>
          <div class="ic-note">[MAKLUMAT / ALAMAT PEMEGANG DI SINI]</div>
          <div class="ic-bottom">SALINAN UNTUK SEMAKAN DOKUMEN</div>
        </div>
        <p class="document-note">Pastikan salinan jelas, lengkap dan tidak terpotong.</p>
      `;
    case 'birth':
      return `
        ${docHeader('SIJIL KELAHIRAN')}
        <div class="civil-document">
          <div class="civil-emblem">DOKUMEN<br>CONTOH</div>
          <h2 class="document-main-title">SIJIL KELAHIRAN</h2>
          <p class="document-centered">BUTIRAN KELAHIRAN · CONTOH PAPARAN</p>
          <div class="official-fields">
            <div><small>NAMA</small><b>NAMA PENUH</b></div>
            <div><small>NO. PENDAFTARAN</small><b>XXXXXXXXXX</b></div>
            <div><small>TARIKH LAHIR</small><b>DD / MM / TAHUN</b></div>
            <div><small>MASA LAHIR</small><b>00:00</b></div>
            <div><small>TEMPAT LAHIR</small><b>[HOSPITAL / TEMPAT]</b></div>
            <div><small>JANTINA</small><b>[JANTINA]</b></div>
            <div><small>NAMA BAPA</small><b>[NAMA BAPA]</b></div>
            <div><small>NAMA IBU</small><b>[NAMA IBU]</b></div>
          </div>
          <div class="civil-bottom">CONTOH REKAAN · BUKAN SIJIL RASMI</div>
        </div>
      `;
    case 'school':
      return `
        ${docHeader('SIJIL BERHENTI SEKOLAH')}
        <div class="letter-document">
          <div class="letter-ref">Ruj. Kami: [NO. RUJUKAN] <span>Tarikh: [TARIKH]</span></div>
          <h2 class="document-main-title">SIJIL BERHENTI SEKOLAH</h2>
          <p>Kepada pihak yang berkenaan,</p>
          <p>Dengan ini disahkan bahawa:</p>
          <div class="letter-highlight">NAMA PENUH PELAJAR</div>
          <p>No. Kad Pengenalan: <b>XXXXXX-XX-XXXX</b></p>
          <p>telah bersekolah di <b>NAMA SEKOLAH</b> dan tamat / berhenti persekolahan pada <b>[TARIKH / TAHUN]</b>.</p>
          <p>Surat pengesahan ini disediakan sebagai contoh susun atur dokumen untuk tujuan semakan fail temuduga.</p>
          ${signatureBlock('PENGETUA / GURU BESAR', 'COP SEKOLAH')}
        </div>
      `;
    case 'certificate-stpm':
      return certificateBody('SIJIL STPM', 'SIJIL TINGGI PERSEKOLAHAN MALAYSIA (STPM)', 'telah menduduki dan memenuhi syarat peperiksaan yang ditetapkan.');
    case 'transcript-matric':
      return transcriptBody('PROGRAM MATRIKULASI');
    case 'certificate-diploma':
      return certificateBody('SIJIL DIPLOMA', 'DIPLOMA [NAMA BIDANG]', 'telah memenuhi semua keperluan akademik program pengajian.');
    case 'transcript-diploma':
      return transcriptBody('DIPLOMA [NAMA PROGRAM]');
    case 'certificate-degree':
      return certificateBody('SIJIL IJAZAH SARJANA MUDA', 'IJAZAH SARJANA MUDA [NAMA BIDANG]', 'telah dianugerahkan kelayakan akademik tersebut.');
    case 'transcript-degree':
      return transcriptBody('IJAZAH SARJANA MUDA [NAMA PROGRAM]');
    case 'licences':
      return `
        ${docHeader('LESEN / PENDAFTARAN PROFESIONAL')}
        <h2 class="document-main-title">LESEN-LESEN BERDAFTAR</h2>
        <p class="document-intro">Lampirkan lesen atau bukti pendaftaran yang berkaitan dengan kelayakan dan bidang.</p>
        <div class="licence-card"><div class="licence-ribbon">PROFESSIONAL REGISTRATION · SAMPLE</div><h3>LESEN / SIJIL PENDAFTARAN</h3><div class="licence-number">NO. PENDAFTARAN: XXXXXXXX</div><div class="licence-fields"><span>Nama pemegang</span><b>NAMA PENUH</b><span>Badan profesional</span><b>[NAMA BADAN]</b><span>Bidang / kategori</span><b>[BIDANG]</b><span>Tarikh luput</span><b>[TARIKH]</b></div></div>
        <div class="document-check-lines"><b>Semakan dokumen</b><p>□ Salinan lesen jelas</p><p>□ Tempoh sah masih berkuat kuasa</p><p>□ Kelayakan / bidang berkaitan</p></div>
      `;
    case 'language':
      return `
        ${docHeader('SIJIL BAHASA')}
        <div class="language-document"><div class="language-band">LANGUAGE PROFICIENCY · SAMPLE</div><h2 class="document-main-title">SIJIL PENCAPAIAN BAHASA</h2><p class="document-centered">CERTIFICATE OF LANGUAGE PROFICIENCY</p><div class="certificate-recipient">NAMA PENUH CALON</div><p class="document-centered">telah menduduki penilaian bahasa berikut:</p><div class="language-result"><div><small>BAHASA</small><b>ENGLISH / MANDARIN / LAIN-LAIN</b></div><div><small>UJIAN / TAHAP</small><b>[NAMA UJIAN / TAHAP]</b></div><div><small>SKOR / GRED</small><b>[SKOR / GRED]</b></div><div><small>TARIKH</small><b>[TARIKH]</b></div></div>${signatureBlock('PENILAI / PENGUJI', 'PENGELUAR SIJIL')}</div>
      `;
    case 'scholarship':
      return `
        ${docHeader('PENGIKTIRAFAN KELAYAKAN BIASISWA / TAJAAN')}
        <div class="recognition-document"><div class="recognition-label">SURAT / SIJIL PENGIKTIRAFAN</div><h2 class="document-main-title">PENGESAHAN KELAYAKAN TAJAAN</h2><p class="document-centered">Dengan ini disahkan bahawa</p><div class="certificate-recipient">NAMA PENERIMA</div><p class="document-centered">telah menerima pengiktirafan / tawaran biasiswa atau tajaan seperti berikut:</p><div class="official-fields"><div><small>NAMA BIASISWA / TAJAAN</small><b>[NAMA TAJAAN]</b></div><div><small>AGENSI / PENYUMBANG</small><b>[NAMA AGENSI]</b></div><div><small>PERINGKAT PENGAJIAN</small><b>[PERINGKAT]</b></div><div><small>TEMPOH</small><b>[TEMPOH TAJAAN]</b></div></div>${signatureBlock('PEGAWAI TAJAAN', 'PENERIMA')}</div>
      `;
    case 'dean':
      return certificateBody('SIJIL KEPUJIAN DEKAN', 'PENCAPAIAN AKADEMIK CEMERLANG', 'telah mencapai keputusan akademik yang melayakkan penerima menerima Anugerah Dekan.');
    case 'awards':
      return certificateBody('SIJIL ANUGERAH AKADEMIK', 'PENGHARGAAN / ANUGERAH BERKAITAN', 'telah menerima pengiktirafan atas pencapaian dan sumbangan yang diberikan.');
    case 'cocurricular':
      return `
        ${docHeader('PENGLIBATAN KOKURIKULUM')}
        <div class="certificate-document"><div class="certificate-frame"><div class="certificate-seal">UKP</div><div class="certificate-heading">SIJIL PENGHARGAAN</div><div class="certificate-small">DENGAN INI DIANUGERAHKAN KEPADA</div><div class="certificate-recipient">NAMA PENUH PELAJAR</div><p class="certificate-copy">atas penglibatan dan sumbangan dalam aktiviti kokurikulum seperti berikut:</p><div class="co-details"><b>[NAMA PROGRAM / PERSATUAN]</b><span>Jawatan: [JAWATAN / PERANAN]</span><span>Tempoh: [TAHUN / TEMPOH]</span><span>Peringkat: [FAKULTI / UNIVERSITI / NEGERI / KEBANGSAAN]</span></div>${signatureBlock('PENASIHAT / PENYELARAS', 'PENGANJUR')}</div></div>
      `;
    default:
      return `<h2 class="document-main-title">${esc(item.name)}</h2><p class="document-centered">CONTOH DOKUMEN UNTUK SEMAKAN</p>`;
  }
}

function createPhysicalPage(pageIndex) {
  const item = FAIL_TEMUDUGA_ITEMS[pageIndex];
  if (!item) return '';

  return `
    <article class="document-sheet">
      <div class="document-watermark">CONTOH VISUAL</div>
      <div class="document-index">DOKUMEN ${String(pageIndex + 1).padStart(2, '0')} / 16</div>
      ${documentBody(item, pageIndex)}
      <div class="document-disclaimer">CONTOH REKAAN PAPARAN · BUKAN DOKUMEN RASMI ATAU SALINAN SAH</div>
      <div class="document-sheet-footer"><span>FAIL TEMUDUGA · UNIT KEPIMPINAN PELAJAR</span><b>${pageIndex + 1}</b></div>
    </article>
  `;
}

function renderPhysicalPage() {
  const totalPages = FAIL_TEMUDUGA_ITEMS.length;
  const nextIndex = Math.min(failTemudugaPageIndex + 1, totalPages - 1);
  const previousIndex = Math.max(failTemudugaPageIndex - 1, 0);

  $('pageFront').innerHTML = createPhysicalPage(failTemudugaPageIndex);
  $('pageBack').innerHTML = createPhysicalPage(nextIndex);
  $('pageUnder').innerHTML = createPhysicalPage(nextIndex);

  $('failPageNumber').textContent =
    'Page ' + (failTemudugaPageIndex + 1) + ' / ' + totalPages +
    ' · ' + FAIL_TEMUDUGA_ITEMS[failTemudugaPageIndex].name;

  $('prevFailPage').disabled = failTemudugaPageIndex === 0;
  $('nextFailPage').disabled = failTemudugaPageIndex === totalPages - 1;
  $('pagePrevHotspot').disabled = failTemudugaPageIndex === 0;
  $('pageNextHotspot').disabled = failTemudugaPageIndex === totalPages - 1;
  $('pageUnder').innerHTML = createPhysicalPage(nextIndex);
}

function flipPhysicalPage(direction) {
  const binder = $('realBinder');
  const sheet = $('turnSheet');
  if (!binder.classList.contains('is-open') || failTemudugaAnimating) return;

  const totalPages = FAIL_TEMUDUGA_ITEMS.length;
  const targetIndex = failTemudugaPageIndex + direction;
  if (targetIndex < 0 || targetIndex >= totalPages) return;

  failTemudugaAnimating = true;
  $('pageFront').innerHTML = createPhysicalPage(failTemudugaPageIndex);
  $('pageBack').innerHTML = createPhysicalPage(targetIndex);
  $('pageUnder').innerHTML = createPhysicalPage(targetIndex);

  binder.classList.remove('turn-next', 'turn-prev');
  void sheet.offsetWidth;
  binder.classList.add(direction > 0 ? 'turn-next' : 'turn-prev');

  const finishFlip = function(event) {
    if (event && (event.target !== sheet || event.propertyName !== 'transform')) return;
    if (failTemudugaFlipHandler) {
      sheet.removeEventListener('transitionend', failTemudugaFlipHandler);
      failTemudugaFlipHandler = null;
    }
    if (failTemudugaFlipTimer) {
      clearTimeout(failTemudugaFlipTimer);
      failTemudugaFlipTimer = null;
    }

    // Remove the turn class without animating the sheet back to its resting position.
    sheet.style.transition = 'none';
    binder.classList.remove('turn-next', 'turn-prev');
    failTemudugaPageIndex = targetIndex;
    renderPhysicalPage();
    void sheet.offsetWidth;
    sheet.style.transition = '';
    failTemudugaAnimating = false;
  };

  failTemudugaFlipHandler = finishFlip;
  sheet.addEventListener('transitionend', finishFlip);
  failTemudugaFlipTimer = setTimeout(() => finishFlip(), 950);
}

function bindEvents() {
window.addEventListener('message', handleExternalQrMessage_);
  $('module').addEventListener('change', () => {
    refreshGroups();
    loadView();
    updateQrScanButton();
  });
  $('group').addEventListener('change', () => { loadView(); updateQrScanButton(); });
  $('date').addEventListener('change', () => { loadView(); updateQrScanButton(); });
  $('search').addEventListener('input', () => renderStudents(getFilteredStudents()));
  $('statusFilter').addEventListener('change', () => renderStudents(getFilteredStudents()));
  $('directorySearch').addEventListener('input', () => renderDirectory());
  $('directoryFacilitator').addEventListener('change', () => renderDirectory());
  $('directoryAssistant').addEventListener('change', () => renderDirectory());
  $('clearDirectoryFilters').addEventListener('click', () => {
    $('directorySearch').value = '';
    $('directoryFacilitator').value = '';
    $('directoryAssistant').value = '';
    renderDirectory();
  });

  $('close').addEventListener('click', closeModal);
  $('cancel').addEventListener('click', closeModal);
  $('save').addEventListener('click', save);
  $('status').addEventListener('change', updateRemarksHint);

  $('addStudentBtn').addEventListener('click', openAddStudent);
  $('closeAddStudent').addEventListener('click', closeAddStudent);
  $('cancelAddStudent').addEventListener('click', closeAddStudent);
  $('saveNewStudent').addEventListener('click', saveNewStudent);

  // Student QR scanning – filters must be selected before the camera opens.
  $('scanStudentBtn').addEventListener('click', openQrScanner);
  $('closeQrScanner').addEventListener('click', closeQrScanner);
  $('stopQrScanner').addEventListener('click', closeQrScanner);
  $('manualQrLookup').addEventListener('click', lookupManualQrStudent);
  $('manualQrStudentId').addEventListener('keydown', event => {
    if (event.key === 'Enter') lookupManualQrStudent();
  });
  $('qrScannerModal').addEventListener('click', event => {
    if (event.target === $('qrScannerModal')) closeQrScanner();
  });

  // Downloads and physical Fail Temuduga binder
  $('downloadsBtn').addEventListener('click', openDownloads);
  $('closeDownloads').addEventListener('click', closeDownloads);
  $('openFailTemuduga').addEventListener('click', openFailTemuduga);
  $('backDownloads').addEventListener('click', () => {
    $('failTemudugaBook').classList.add('hidden');
    $('downloadsHome').classList.remove('hidden');
    $('realBinder').classList.remove('is-open', 'turn-next', 'turn-prev');
  });
  $('openBinderBtn').addEventListener('click', openPhysicalBinder);
  $('prevFailPage').addEventListener('click', () => flipPhysicalPage(-1));
  $('nextFailPage').addEventListener('click', () => flipPhysicalPage(1));
  $('pagePrevHotspot').addEventListener('click', () => flipPhysicalPage(-1));
  $('pageNextHotspot').addEventListener('click', () => flipPhysicalPage(1));

  // Close the downloads overlay when the shaded area is clicked.
  $('downloadsModal').addEventListener('click', event => {
    if (event.target === $('downloadsModal')) closeDownloads();
  });
}

document.addEventListener('DOMContentLoaded', () => {
  bindEvents();
  init();
});

</script>
