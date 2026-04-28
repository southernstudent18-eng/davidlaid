<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>The David Blueprint — Training Tracker</title>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,600;1,300;1,400&family=DM+Mono:wght@300;400;500&display=swap" rel="stylesheet">
<!-- Firebase -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getAuth, GoogleAuthProvider, signInWithPopup, signOut, onAuthStateChanged }
  from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import { getFirestore, doc, setDoc, getDoc, collection, addDoc, serverTimestamp }
  from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyBTgBMA0YLvU0BPhYJXZyXcREF8rOfApTs",
  authDomain: "david-blueprint.firebaseapp.com",
  databaseURL: "https://david-blueprint-default-rtdb.firebaseio.com",
  projectId: "david-blueprint",
  storageBucket: "david-blueprint.firebasestorage.app",
  messagingSenderId: "190439970987",
  appId: "1:190439970987:web:e7a987e11153b4dad13fee"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// Expose to global scope so non-module functions can call them
window._fbAuth = auth;
window._fbDb = db;
window._fbDoc = doc;
window._fbSetDoc = setDoc;
window._fbGetDoc = getDoc;
window._fbCollection = collection;
window._fbAddDoc = addDoc;
window._fbServerTimestamp = serverTimestamp;
window._fbGoogleProvider = new GoogleAuthProvider();
window._fbSignInWithPopup = signInWithPopup;
window._fbSignOut = signOut;

// Watch auth state
onAuthStateChanged(auth, async (user) => {
  window._currentUser = user || null;
  if (user) {
    renderUserHeader(user);
    await loadFromFirebase(user.uid);
  } else {
    renderSignedOutHeader();
  }
});
</script>
<style>
:root {
  --bg: #f7f3ec;
  --bg2: #efe9df;
  --card: #fdfaf5;
  --text: #1c1713;
  --muted: #7a6e63;
  --accent: #8b6f47;
  --accent-dark: #5c4a2e;
  --accent-light: #c9a97a;
  --gold: #b8985f;
  --green: #5c7a3a;
  --green-bg: #eaf2e4;
  --green-light: #c8dbb8;
  --border: #d8d0c4;
  --border-light: #ece6dc;
  --red: #9b3a2a;
  --red-bg: #fdf0ed;
  --shadow: 0 2px 12px rgba(92,74,46,0.10);
  --shadow-lg: 0 8px 32px rgba(92,74,46,0.14);
}

* { box-sizing: border-box; margin: 0; padding: 0; }
html { scroll-behavior: smooth; }

body {
  font-family: 'DM Mono', monospace;
  background: var(--bg);
  color: var(--text);
  min-height: 100vh;
  font-size: 13px;
  line-height: 1.6;
}

/* Grain overlay */
body::before {
  content: '';
  position: fixed;
  inset: 0;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)' opacity='0.04'/%3E%3C/svg%3E");
  pointer-events: none;
  z-index: 1000;
  opacity: 0.4;
}

/* ======== HEADER ======== */
.header {
  background: var(--accent-dark);
  color: #faf8f3;
  padding: 0;
  position: sticky;
  top: 0;
  z-index: 100;
  box-shadow: 0 2px 20px rgba(0,0,0,0.2);
}
.header-inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 28px;
  max-width: 1100px;
  margin: 0 auto;
}
.header-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: 22px;
  font-weight: 400;
  letter-spacing: 0.5px;
  color: #fdf7ed;
}
.header-title span {
  color: var(--accent-light);
  font-style: italic;
}
.header-meta {
  display: flex;
  align-items: center;
  gap: 20px;
}
.week-display {
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(255,255,255,0.08);
  border: 1px solid rgba(255,255,255,0.15);
  border-radius: 6px;
  padding: 6px 14px;
}
.week-display label {
  font-size: 10px;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: var(--accent-light);
}
.week-display select {
  background: transparent;
  border: none;
  color: #faf8f3;
  font-family: 'DM Mono', monospace;
  font-size: 13px;
  cursor: pointer;
  outline: none;
}
.week-display select option { background: var(--accent-dark); color: #faf8f3; }

/* ======== MAIN LAYOUT ======== */
.main {
  max-width: 1100px;
  margin: 0 auto;
  padding: 28px 24px;
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: 24px;
  align-items: start;
}

/* ======== SIDEBAR ======== */
.sidebar {
  position: sticky;
  top: 80px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.panel {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 8px;
  overflow: hidden;
  box-shadow: var(--shadow);
}
.panel-head {
  padding: 12px 18px;
  background: var(--bg2);
  border-bottom: 1px solid var(--border);
  font-size: 10px;
  letter-spacing: 2.5px;
  text-transform: uppercase;
  color: var(--accent-dark);
  font-weight: 500;
}
.panel-body { padding: 14px 16px; }

/* Day selector */
.day-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.day-btn {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 10px 8px;
  cursor: pointer;
  text-align: center;
  transition: all 0.15s;
  font-family: 'DM Mono', monospace;
}
.day-btn:hover { border-color: var(--accent); background: var(--border-light); }
.day-btn.active {
  background: var(--accent-dark);
  border-color: var(--accent-dark);
  color: #fdf7ed;
}
.day-btn .day-label {
  display: block;
  font-size: 10px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: var(--muted);
  margin-bottom: 3px;
}
.day-btn.active .day-label { color: var(--accent-light); }
.day-btn .day-name {
  display: block;
  font-size: 12px;
  font-weight: 500;
  color: var(--text);
}
.day-btn.active .day-name { color: #fdf7ed; }
.day-btn .day-tag {
  display: inline-block;
  font-size: 9px;
  letter-spacing: 1px;
  text-transform: uppercase;
  margin-top: 4px;
  color: var(--muted);
  background: var(--bg2);
  padding: 1px 6px;
  border-radius: 3px;
}
.day-btn.active .day-tag { background: rgba(255,255,255,0.12); color: var(--accent-light); }
.day-btn.completed { border-color: var(--green); }
.day-btn.completed .day-label { color: var(--green); }

/* Progress stats */
.stat-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-bottom: 12px;
}
.stat-box {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 10px 12px;
  text-align: center;
}
.stat-val {
  font-family: 'Cormorant Garamond', serif;
  font-size: 26px;
  font-weight: 600;
  color: var(--accent-dark);
  line-height: 1;
}
.stat-label {
  font-size: 9px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: var(--muted);
  margin-top: 3px;
}

/* Week progress bar */
.week-bar-wrap { margin-top: 4px; }
.week-bar-label {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: var(--muted);
  margin-bottom: 5px;
}
.week-bar {
  height: 6px;
  background: var(--bg2);
  border-radius: 3px;
  overflow: hidden;
  border: 1px solid var(--border);
}
.week-bar-fill {
  height: 100%;
  background: linear-gradient(90deg, var(--accent), var(--gold));
  border-radius: 3px;
  transition: width 0.4s ease;
}

/* Phase progress */
.phase-track {
  display: flex;
  gap: 4px;
  margin-top: 8px;
}
.phase-dot {
  flex: 1;
  height: 4px;
  background: var(--bg2);
  border-radius: 2px;
  border: 1px solid var(--border);
  transition: background 0.3s;
}
.phase-dot.done { background: var(--green); border-color: var(--green); }
.phase-dot.current { background: var(--gold); border-color: var(--gold); }

/* ======== MAIN CONTENT ======== */
.content { }

.session-header {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 20px 24px;
  margin-bottom: 18px;
  box-shadow: var(--shadow);
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
}
.session-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: 30px;
  font-weight: 400;
  color: var(--accent-dark);
  line-height: 1.1;
}
.session-subtitle {
  font-size: 11px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: var(--muted);
  margin-top: 4px;
}
.session-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 8px;
  flex-shrink: 0;
}
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 18px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-family: 'DM Mono', monospace;
  font-size: 11px;
  letter-spacing: 1px;
  text-transform: uppercase;
  transition: all 0.15s;
}
.btn-primary {
  background: var(--accent-dark);
  color: #fdf7ed;
}
.btn-primary:hover { background: var(--accent); }
.btn-secondary {
  background: var(--bg2);
  border: 1px solid var(--border);
  color: var(--text);
}
.btn-secondary:hover { border-color: var(--accent); }
.btn-success {
  background: var(--green);
  color: #fff;
}
.btn-success:hover { background: #4a6b2c; }

/* Volume note banner */
.volume-note {
  background: var(--green-bg);
  border: 1px solid var(--green-light);
  border-radius: 6px;
  padding: 10px 16px;
  margin-bottom: 16px;
  font-size: 12px;
  color: var(--green);
  display: flex;
  align-items: center;
  gap: 10px;
}
.volume-note .vn-icon {
  font-size: 16px;
  flex-shrink: 0;
}
.volume-note strong { color: #4a6b2c; }

/* ======== EXERCISE TABLE ======== */
.exercise-card {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 8px;
  margin-bottom: 14px;
  box-shadow: var(--shadow);
  overflow: hidden;
  transition: box-shadow 0.2s;
}
.exercise-card:hover { box-shadow: var(--shadow-lg); }

.ex-head {
  display: flex;
  align-items: center;
  padding: 13px 18px;
  background: var(--bg2);
  border-bottom: 1px solid var(--border);
  gap: 12px;
  cursor: pointer;
  user-select: none;
}
.ex-num {
  font-family: 'Cormorant Garamond', serif;
  font-size: 18px;
  color: var(--accent-light);
  min-width: 22px;
  line-height: 1;
}
.ex-name {
  font-family: 'Cormorant Garamond', serif;
  font-size: 17px;
  font-weight: 400;
  color: var(--accent-dark);
  flex: 1;
}
.ex-tags {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}
.tag {
  font-size: 9px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  padding: 2px 8px;
  border-radius: 3px;
  font-weight: 500;
}
.tag-primary { background: #e8ddd0; color: var(--accent-dark); }
.tag-weak { background: #fdebd0; color: #9b6a1a; }
.tag-new { background: var(--green-bg); color: var(--green); border: 1px solid var(--green-light); }
.tag-rpe { background: var(--bg); color: var(--muted); border: 1px solid var(--border); }
.ex-toggle { color: var(--muted); font-size: 12px; transition: transform 0.2s; }
.ex-head.open .ex-toggle { transform: rotate(180deg); }

.ex-body { padding: 16px 18px; display: none; }
.ex-body.open { display: block; }

.ex-note {
  font-size: 11px;
  color: var(--muted);
  font-style: italic;
  margin-bottom: 12px;
  line-height: 1.5;
  padding-left: 2px;
  border-left: 2px solid var(--border);
  padding-left: 10px;
}

/* Sets table */
.sets-table {
  width: 100%;
  border-collapse: collapse;
}
.sets-table th {
  font-size: 9px;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: var(--muted);
  padding: 6px 10px;
  text-align: left;
  border-bottom: 1px solid var(--border);
  background: var(--bg);
}
.sets-table td {
  padding: 8px 10px;
  border-bottom: 1px solid var(--border-light);
  vertical-align: middle;
}
.sets-table tr:last-child td { border-bottom: none; }

.set-num {
  font-size: 11px;
  color: var(--muted);
  width: 30px;
}
.set-target {
  font-size: 11px;
  color: var(--accent-dark);
  font-weight: 500;
  width: 80px;
}
.weight-input {
  width: 80px;
  padding: 5px 8px;
  border: 1px solid var(--border);
  border-radius: 4px;
  background: var(--bg);
  font-family: 'DM Mono', monospace;
  font-size: 12px;
  color: var(--text);
  text-align: center;
  transition: border-color 0.15s;
}
.weight-input:focus { outline: none; border-color: var(--accent); background: #fff; }
.reps-input {
  width: 60px;
  padding: 5px 8px;
  border: 1px solid var(--border);
  border-radius: 4px;
  background: var(--bg);
  font-family: 'DM Mono', monospace;
  font-size: 12px;
  color: var(--text);
  text-align: center;
  transition: border-color 0.15s;
}
.reps-input:focus { outline: none; border-color: var(--accent); background: #fff; }
.set-check {
  width: 28px;
  height: 28px;
  border: 2px solid var(--border);
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
  background: var(--bg);
  font-size: 12px;
}
.set-check:hover { border-color: var(--green); }
.set-check.done { background: var(--green); border-color: var(--green); color: #fff; }
.set-check.done::after { content: '✓'; }
.set-pr { font-size: 10px; color: var(--green); font-weight: 500; }

/* ======== DELOAD BANNER ======== */
.deload-banner {
  background: linear-gradient(135deg, #f7f0e6, #ede4d4);
  border: 1px solid var(--gold);
  border-radius: 8px;
  padding: 20px 24px;
  margin-bottom: 18px;
  text-align: center;
}
.deload-banner h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 22px;
  color: var(--accent-dark);
  margin-bottom: 6px;
}
.deload-banner p {
  color: var(--muted);
  font-size: 12px;
  line-height: 1.6;
}
.deload-banner .deload-rules {
  margin-top: 12px;
  display: flex;
  gap: 16px;
  justify-content: center;
  flex-wrap: wrap;
}
.deload-rule {
  background: rgba(255,255,255,0.6);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 8px 14px;
  font-size: 11px;
  color: var(--accent-dark);
}

/* ======== LOG HISTORY ======== */
.log-entry {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 12px 16px;
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}
.log-date {
  font-size: 10px;
  color: var(--muted);
  letter-spacing: 1px;
}
.log-day { font-size: 12px; color: var(--accent-dark); font-weight: 500; }
.log-sets { font-size: 11px; color: var(--muted); }
.log-delete {
  background: none;
  border: 1px solid var(--border);
  border-radius: 4px;
  padding: 3px 8px;
  cursor: pointer;
  font-size: 10px;
  color: var(--muted);
  font-family: 'DM Mono', monospace;
}
.log-delete:hover { border-color: var(--red); color: var(--red); }

/* ======== BENCHMARKS ======== */
.benchmark-item {
  display: flex;
  align-items: center;
  padding: 9px 0;
  border-bottom: 1px solid var(--border-light);
  gap: 10px;
}
.benchmark-item:last-child { border: none; }
.bm-check {
  width: 18px;
  height: 18px;
  border: 2px solid var(--border);
  border-radius: 3px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  flex-shrink: 0;
  transition: all 0.15s;
}
.bm-check.done { background: var(--green); border-color: var(--green); color: #fff; }
.bm-name { flex: 1; font-size: 11px; color: var(--text); }
.bm-target { font-size: 11px; color: var(--gold); font-weight: 500; }

/* ======== TOAST ======== */
.toast {
  position: fixed;
  bottom: 24px;
  right: 24px;
  background: var(--accent-dark);
  color: #fdf7ed;
  padding: 12px 20px;
  border-radius: 8px;
  font-size: 12px;
  letter-spacing: 0.5px;
  box-shadow: var(--shadow-lg);
  z-index: 500;
  transform: translateY(80px);
  opacity: 0;
  transition: all 0.25s ease;
}
.toast.show { transform: translateY(0); opacity: 1; }
.toast.success { background: var(--green); }

/* ======== RESPONSIVE ======== */
@media (max-width: 768px) {
  .main { grid-template-columns: 1fr; padding: 14px; gap: 14px; }
  .sidebar { position: static; }
  .header-inner { padding: 12px 16px; }
  .header-title { font-size: 17px; }
  .week-display { display: none; }
  .session-header { flex-direction: column; }
  .session-actions { align-items: flex-start; flex-direction: row; flex-wrap: wrap; }
}

/* No-session state */
.no-session {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 60px 40px;
  text-align: center;
  box-shadow: var(--shadow);
}
.no-session h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 26px;
  color: var(--accent-dark);
  margin-bottom: 8px;
  font-style: italic;
}
.no-session p { font-size: 12px; color: var(--muted); }

/* ======== AUTH ======== */
.auth-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 7px 16px;
  border-radius: 6px;
  border: 1px solid rgba(255,255,255,0.25);
  background: rgba(255,255,255,0.1);
  color: #fdf7ed;
  font-family: 'DM Mono', monospace;
  font-size: 11px;
  letter-spacing: 0.5px;
  cursor: pointer;
  transition: all 0.15s;
}
.auth-btn:hover { background: rgba(255,255,255,0.18); }
.auth-btn img {
  width: 18px;
  height: 18px;
  border-radius: 50%;
}
.auth-btn.signout {
  background: rgba(155,58,42,0.3);
  border-color: rgba(155,58,42,0.5);
}
.auth-btn.signout:hover { background: rgba(155,58,42,0.5); }
.sync-indicator {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  color: var(--accent-light);
  letter-spacing: 1px;
}
.sync-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--green);
  animation: pulse 2s infinite;
}
.sync-dot.syncing { background: var(--gold); animation: none; }
.sync-dot.offline { background: var(--muted); animation: none; }
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}
.login-overlay {
  position: fixed;
  inset: 0;
  background: rgba(28,23,19,0.92);
  z-index: 999;
  display: flex;
  align-items: center;
  justify-content: center;
}
.login-card {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 48px 44px;
  text-align: center;
  max-width: 380px;
  width: 90%;
  box-shadow: 0 20px 60px rgba(0,0,0,0.4);
}
.login-card h2 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 32px;
  color: var(--accent-dark);
  font-weight: 400;
  margin-bottom: 6px;
}
.login-card h2 span { font-style: italic; color: var(--gold); }
.login-card p {
  font-size: 12px;
  color: var(--muted);
  margin-bottom: 28px;
  line-height: 1.6;
}
.google-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  width: 100%;
  padding: 14px 20px;
  background: var(--accent-dark);
  color: #fdf7ed;
  border: none;
  border-radius: 8px;
  font-family: 'DM Mono', monospace;
  font-size: 13px;
  letter-spacing: 0.5px;
  cursor: pointer;
  transition: background 0.15s;
}
.google-btn:hover { background: var(--accent); }
.google-btn svg { flex-shrink: 0; }
.login-skip {
  margin-top: 14px;
  font-size: 11px;
  color: var(--muted);
  cursor: pointer;
  text-decoration: underline;
  background: none;
  border: none;
  font-family: 'DM Mono', monospace;
}
.login-skip:hover { color: var(--accent); }

/* ======== CALENDAR ======== */
.cal-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}
.cal-nav button {
  background: none;
  border: 1px solid var(--border);
  border-radius: 4px;
  width: 28px;
  height: 28px;
  cursor: pointer;
  font-size: 13px;
  color: var(--accent-dark);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s;
}
.cal-nav button:hover { background: var(--bg2); border-color: var(--accent); }
.cal-month {
  font-family: 'Cormorant Garamond', serif;
  font-size: 16px;
  color: var(--accent-dark);
  font-weight: 600;
}
.cal-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 3px;
}
.cal-dow {
  text-align: center;
  font-size: 9px;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: var(--muted);
  padding: 3px 0 5px;
}
.cal-day {
  aspect-ratio: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 5px;
  font-size: 11px;
  cursor: default;
  position: relative;
  border: 1px solid transparent;
  transition: all 0.12s;
}
.cal-day.empty { background: none; }
.cal-day.other-month { color: var(--border); }
.cal-day.today {
  border-color: var(--gold);
  color: var(--accent-dark);
  font-weight: 600;
  background: #fdf5e4;
}
.cal-day.trained {
  background: var(--green-bg);
  border-color: var(--green-light);
}
.cal-day.trained .cal-day-num { color: var(--green); font-weight: 600; }
.cal-day.rest { background: var(--bg2); }
.cal-dot-row {
  display: flex;
  gap: 2px;
  margin-top: 1px;
  flex-wrap: wrap;
  justify-content: center;
}
.cal-dot {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: var(--green);
}
.cal-legend {
  display: flex;
  gap: 12px;
  margin-top: 10px;
  font-size: 9px;
  color: var(--muted);
}
.cal-legend-item { display: flex; align-items: center; gap: 5px; }
.cal-legend-dot { width: 10px; height: 10px; border-radius: 3px; }
.cal-stats-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 6px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid var(--border-light);
}
.cal-stat {
  text-align: center;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: 5px;
  padding: 8px 4px;
}
.cal-stat-val {
  font-family: 'Cormorant Garamond', serif;
  font-size: 20px;
  color: var(--accent-dark);
  line-height: 1;
}
.cal-stat-label {
  font-size: 8px;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: var(--muted);
  margin-top: 2px;
}
.heatmap-wrap {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid var(--border-light);
}
.heatmap-title {
  font-size: 10px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: var(--muted);
  margin-bottom: 6px;
}
.heatmap-row {
  display: flex;
  gap: 2px;
  flex-wrap: wrap;
}
.heatmap-cell {
  width: 9px;
  height: 9px;
  border-radius: 2px;
  background: var(--bg2);
  border: 1px solid var(--border-light);
}
.heatmap-cell.d1 { background: #d6eacc; border-color: #c8dbb8; }
.heatmap-cell.d2 { background: #a8d08a; border-color: #90c06e; }
.heatmap-cell.d3 { background: #7ab852; border-color: #62a03a; }
.heatmap-cell.d4 { background: var(--green); border-color: #4a6b2c; }

/* ======== Progression badge ======== */
.prog-badge {
  display: inline-block;
  background: var(--green-bg);
  border: 1px solid var(--green-light);
  color: var(--green);
  font-size: 9px;
  letter-spacing: 1px;
  text-transform: uppercase;
  padding: 2px 8px;
  border-radius: 3px;
  margin-left: 8px;
}

.session-complete-banner {
  background: linear-gradient(135deg, var(--green-bg), #d6eacc);
  border: 1px solid var(--green-light);
  border-radius: 8px;
  padding: 16px 22px;
  margin-bottom: 18px;
  display: flex;
  align-items: center;
  gap: 14px;
}
.sc-icon { font-size: 28px; }
.sc-text h4 { font-family: 'Cormorant Garamond', serif; font-size: 20px; color: var(--green); }
.sc-text p { font-size: 11px; color: var(--muted); margin-top: 2px; }
</style>
</head>
<body>

<header class="header">
  <div class="header-inner">
    <div class="header-title">The <span>David</span> Blueprint</div>
    <div class="header-meta">
      <div class="sync-indicator" id="syncIndicator" style="display:none;">
        <div class="sync-dot" id="syncDot"></div>
        <span id="syncLabel">Synced</span>
      </div>
      <div class="week-display">
        <label>Week</label>
        <select id="weekSelect" onchange="setWeek(this.value)">
          <option value="1">1 — Foundation</option>
          <option value="2">2 — Build</option>
          <option value="3">3 — Volume</option>
          <option value="4">4 — Push</option>
          <option value="5">5 — Peak</option>
          <option value="6">6 — Deload</option>
        </select>
      </div>
      <div id="authArea">
        <!-- rendered by JS -->
      </div>
    </div>
  </div>
</header>

<!-- Login overlay -->
<div class="login-overlay" id="loginOverlay">
  <div class="login-card">
    <h2>The <span>David</span><br>Blueprint</h2>
    <p>Sign in to sync your training data across all devices. Your weights, progress, and calendar are saved to your account automatically.</p>
    <button class="google-btn" onclick="signInGoogle()">
      <svg width="18" height="18" viewBox="0 0 18 18"><path fill="#fff" d="M9 3.48c1.69 0 2.83.73 3.48 1.34l2.54-2.48C13.46.89 11.43 0 9 0 5.48 0 2.44 2.02.96 4.96l2.91 2.26C4.6 5.05 6.62 3.48 9 3.48z"/><path fill="#fff" d="M17.64 9.2c0-.74-.06-1.28-.19-1.84H9v3.34h4.96c-.1.83-.64 2.08-1.84 2.92l2.84 2.2c1.7-1.57 2.68-3.88 2.68-6.62z"/><path fill="#fff" d="M3.88 10.78A5.54 5.54 0 0 1 3.58 9c0-.62.11-1.22.29-1.78L.96 4.96A9 9 0 0 0 0 9c0 1.45.35 2.82.96 4.04l2.92-2.26z"/><path fill="#fff" d="M9 18c2.43 0 4.47-.8 5.96-2.18l-2.84-2.2c-.76.53-1.78.9-3.12.9-2.38 0-4.4-1.57-5.12-3.74L.96 13.04C2.44 15.98 5.48 18 9 18z"/></svg>
      Sign in with Google
    </button>
    <button class="login-skip" onclick="skipLogin()">Continue without signing in</button>
  </div>
</div>

<div class="main">

  <!-- ======= SIDEBAR ======= -->
  <aside class="sidebar">

    <div class="panel">
      <div class="panel-head">Select Day</div>
      <div class="panel-body">
        <div class="day-grid" id="dayGrid"></div>
        <div style="margin-top:12px;">
          <button class="btn btn-secondary" style="width:100%; justify-content:center;" onclick="pickMakeupDay()">
            ↩ Make Up Missed Day
          </button>
        </div>
      </div>
    </div>

    <div class="panel">
      <div class="panel-head">Mesocycle Progress</div>
      <div class="panel-body">
        <div class="stat-row">
          <div class="stat-box">
            <div class="stat-val" id="statWeek">1</div>
            <div class="stat-label">Current Week</div>
          </div>
          <div class="stat-box">
            <div class="stat-val" id="statSessions">0</div>
            <div class="stat-label">Sessions Done</div>
          </div>
        </div>
        <div class="stat-row">
          <div class="stat-box">
            <div class="stat-val" id="statTonnage">0</div>
            <div class="stat-label">Total Tons</div>
          </div>
          <div class="stat-box">
            <div class="stat-val" id="statStreak">0</div>
            <div class="stat-label">Day Streak</div>
          </div>
        </div>
        <div class="week-bar-wrap">
          <div class="week-bar-label">
            <span>Week progress</span>
            <span id="weekPct">0%</span>
          </div>
          <div class="week-bar"><div class="week-bar-fill" id="weekFill" style="width:0%"></div></div>
        </div>
        <div style="margin-top:12px;">
          <div style="font-size:10px; letter-spacing:1.5px; text-transform:uppercase; color:var(--muted); margin-bottom:6px;">Mesocycle (6 weeks)</div>
          <div class="phase-track" id="phaseDots"></div>
        </div>
      </div>
    </div>

    <div class="panel">
      <div class="panel-head">Training Calendar</div>
      <div class="panel-body" id="calendarPanel">
        <!-- rendered by JS -->
      </div>
    </div>

    <div class="panel">
      <div class="panel-head">Phase 3 Benchmarks</div>
      <div class="panel-body" id="benchmarkList">
      </div>
    </div>

    <div class="panel">
      <div class="panel-head">Recent Sessions</div>
      <div class="panel-body" id="recentLog" style="padding:0;">
        <div style="padding:14px; font-size:11px; color:var(--muted); text-align:center;">No sessions logged yet.</div>
      </div>
    </div>

  </aside>

  <!-- ======= CONTENT ======= -->
  <main class="content" id="mainContent">
    <div class="no-session">
      <h3>Choose a day to begin</h3>
      <p>Select a training day from the sidebar<br>to load your workout.</p>
    </div>
  </main>

</div>

<div class="toast" id="toast"></div>

<script>
// =============================================
// DATA — Full program with week-based progressions
// =============================================

const DAYS = [
  { id: 'U1', label: 'Day 1', name: 'Upper A', tag: 'Chest · Back' },
  { id: 'L1', label: 'Day 2', name: 'Lower A', tag: 'Quads · Hams' },
  { id: 'U2', label: 'Day 3', name: 'Upper B', tag: 'Shoulder · Back' },
  { id: 'L2', label: 'Day 4', name: 'Lower B', tag: 'Glutes · Hinge' },
  { id: 'P5', label: 'Day 5', name: 'Priority', tag: 'Weak Points' },
];

// Returns list of exercises for a given day and week
function getExercises(dayId, week) {
  const w = parseInt(week);

  const programs = {
    U1: [
      { name: 'Incline DB Press @ 30°', sets: getSets('U1', 'incline_press', w), reps: '8–10', rir: 2, rest: '2.5 min', tags: ['primary', 'weak-upper-chest'], note: 'Palms facing, elbows 45–75°. Touch upper chest/clavicle line. Scapulae retracted and depressed — "shoulders in back pockets." Lower on the chest than you think.' },
      { name: 'Weighted / Assisted Pull-up (neutral grip)', sets: getSets('U1', 'pullup', w), reps: '6–10', rir: 2, rest: '2.5 min', tags: ['primary', 'weak-lats'], note: 'Dead hang at bottom. Drive elbows down and back, not hands up. Chin over bar. 2s eccentric. Neutral grip reduces shoulder strain for long arms.' },
      { name: 'Chest-Supported DB Row', sets: getSets('U1', 'csrow', w), reps: '10–12', rir: 1, rest: '2 min', tags: ['primary'], note: 'Elbows flared ~60°. Pull elbow past ribs, not hand to belly. 1s squeeze at top, full stretch at bottom (scapular protraction).' },
      { name: 'Low-to-High Cable Fly', sets: getSets('U1', 'cable_fly', w), reps: '12–15', rir: 1, rest: '90s', tags: ['isolation'], note: 'Cable at lowest setting. Arc upward and inward — line of pull follows clavicular pec fiber direction. Full stretch at bottom every rep.' },
      { name: 'Cable Lateral Raise (lean-away)', sets: getSets('U1', 'lat_raise_cable', w), reps: '12–15', rir: 1, rest: '90s', tags: ['primary', 'weak-side-delt'], note: 'PRIORITY #1. Lead with elbow, pinky slightly higher than thumb. Trap relaxed — do NOT shrug. 1s pause at top, 2–3s eccentric. Stop at shoulder height.' },
      { name: 'Reverse Pec Deck', sets: getSets('U1', 'rev_pec', w), reps: '12–20', rir: 0, rest: '60s', tags: ['isolation', 'weak-rear-delt'], note: 'Rear delt. Light weight, full arc. Do not round forward at the end — this turns it into a back exercise.' },
      { name: 'Overhead Rope Triceps Extension', sets: getSets('U1', 'oh_tri', w), reps: '10–12', rir: 1, rest: '90s', tags: ['isolation'], note: 'Long head is only fully stretched with arm overhead. Full extension at top, deep stretch at bottom. Do not flare elbows out.' },
      { name: 'Incline DB Curl @ 45°', sets: getSets('U1', 'incline_curl', w), reps: '8–10', rir: 1, rest: '90s', tags: ['isolation'], note: 'Sato 2021: shoulder-extended position stretches the long head maximally. Palms supinated (up). Do NOT swing — slow, controlled, full ROM.' },
      ...(w >= 3 ? [{ name: 'Pronated Wrist Curl', sets: 3, reps: '15–20', rir: 1, rest: '45s', tags: ['forearm'], note: 'Forearms flat on thighs, wrists hang off knees, palms DOWN. Lower until wrists fully extend, curl up using only wrist extension. 2–3s eccentric. Light weight.' }] : []),
      ...(w >= 4 ? [{ name: 'Hammer Curl', sets: 3, reps: '10–12', rir: 1, rest: '90s', tags: ['isolation', 'new'], note: 'Neutral grip (palms facing each other). Hits brachialis + brachioradialis — builds arm THICKNESS and pushes the bicep peak up. Added Week 4.' }] : []),
      ...(w >= 5 ? [{ name: 'Dead Hang', sets: 3, reps: 'Max time (30–60s)', rir: null, rest: '60s', tags: ['forearm', 'grip'], note: 'Passive hang from pull-up bar. Shoulders at ears, no active engagement. Builds grip + thickens flexor tendons. Keep every week permanently.' }] : []),
    ],

    L1: [
      { name: 'Hack Squat Machine', sets: getSets('L1', 'hack_squat', w), reps: '6–10', rir: 2, rest: '3 min', tags: ['primary', 'weak-quads'], note: 'Feet mid-platform, shoulder-width, toes slightly out. Drive knees OVER toes — not back. Full depth (hamstrings to calves). Back pad removes the long-femur problem. This is your primary quad builder.' },
      { name: 'Romanian Deadlift (Smith or BB)', sets: getSets('L1', 'rdl', w), reps: '8–10', rir: 2, rest: '2.5 min', tags: ['primary'], note: 'HINGE, not squat. Push hips straight back. Bar path vertical, against shins the whole way. Stop just below the knee — mid-shin typically means lumbar is rounding. Neutral spine, neutral neck.' },
      { name: 'Leg Press (feet low, narrow)', sets: getSets('L1', 'leg_press', w), reps: '10–12', rir: 1, rest: '2 min', tags: ['primary'], note: 'Feet low and narrow (shoulder-width) for quad bias. Full depth — controlled slight pelvic tilt at bottom is fine. Zero spinal load. Safe to push near failure here.' },
      { name: 'Seated Leg Curl', sets: getSets('L1', 'seated_curl', w), reps: '10–12', rir: 1, rest: '2 min', tags: ['primary', 'weak-hamstrings'], note: 'Maeo 2021: 2.2× more biceps femoris long-head growth vs. lying curl. Hip-flexed position pre-stretches the biarticular hamstring. PRIMARY hamstring exercise.' },
      { name: 'Standing Calf Raise (Smith + plate)', sets: getSets('L1', 'stand_calf', w), reps: '8–12', rir: 1, rest: '90s', tags: ['primary', 'weak-calves'], note: 'Plate under forefoot for extended ROM. 2s pause at full stretch (bottom). Kinoshita/Maeo 2023: standing produces 12.4% gastroc growth vs 1.7% seated. PRIMARY calf exercise.' },
      { name: 'Hanging Knee / Leg Raise', sets: 3, reps: '12–15', rir: 1, rest: '60s', tags: ['core'], note: 'Classic David definition. Knee raises first, progress to straight-leg when you can complete 3×15 knee raises. No swinging.' },
      ...(w >= 2 ? [] : []),
      ...(w >= 5 ? [{ name: 'Bulgarian Split Squat', sets: 2, reps: '10/leg', rir: 1, rest: '90s', tags: ['primary', 'new'], note: 'Vertical torso = quad-biased. Long legs love BSS — extended ROM maximizes stretch-mediated hypertrophy. Added as peak-week finisher.' }] : []),
    ],

    U2: [
      { name: 'Seated DB Overhead Press (bench ~80°)', sets: getSets('U2', 'ohp', w), reps: '8–10', rir: 2, rest: '2.5 min', tags: ['primary'], note: 'DB over BB — long arms allow natural arc. 80° bench angle (not fully upright). Glutes squeezed, ribs down. Stop at near-lockout — last 10° is triceps not delt.' },
      { name: 'Wide-Grip Lat Pulldown (pronated)', sets: getSets('U2', 'pulldown', w), reps: '8–12', rir: 2, rest: '2 min', tags: ['primary', 'weak-lats'], note: 'Grip ~1.5× biacromial width. Pull chest to bar, elbows drive down and back. Full stretch at top — scapula elevates. The V-taper builder.' },
      { name: 'Flat DB Press', sets: getSets('U2', 'flat_db', w), reps: '8–10', rir: 2, rest: '2.5 min', tags: ['primary'], note: 'Elbows 45–75° from torso. Touch just below nipple line. Full stretch at bottom — this is where growth happens. Arch upper back, ribcage high.' },
      { name: 'Single-Arm Cable Row', sets: getSets('U2', 'cable_row', w), reps: '10–12', rir: 1, rest: '90s', tags: ['primary'], note: 'Half-kneeling or seated. Full scapular ROM — let it protract at the front, retract at the back. Pull elbow to hip/lower rib. Your long arms print money here.' },
      { name: 'DB Lateral Raise (strict)', sets: getSets('U2', 'db_lateral', w), reps: '10–15', rir: 1, rest: '90s', tags: ['primary', 'weak-side-delt'], note: 'Shortened-bias complement to lean-away cable. 10–15° forward torso lean. Lead with elbow. Pinky up slightly. TRAPS RELAXED. Stop at shoulder height.' },
      { name: 'Cable Rear Delt Crossover (high pulleys)', sets: getSets('U2', 'rear_delt', w), reps: '12–20', rir: 0, rest: '60s', tags: ['isolation', 'weak-rear-delt'], note: 'Cables at highest position. Cross-body pull, slight bend in elbow, shoulder internally rotated. Deep lengthened position for rear delts.' },
      { name: 'Preacher Curl (EZ bar or machine)', sets: getSets('U2', 'preacher', w), reps: '8–10', rir: 1, rest: '90s', tags: ['isolation'], note: 'Short-head biceps + stretched elbow position. Full ROM — all the way up and all the way down. Do not let elbows leave the pad.' },
      { name: 'Close-Grip Smith Bench', sets: getSets('U2', 'cg_bench', w), reps: '8–10', rir: 1, rest: '90s', tags: ['isolation'], note: 'Compound triceps. Hands ~14" apart. Lower to sternum, press through the bar. Heavy compound triceps work > pushdowns for overall mass.' },
      ...(w >= 3 ? [{ name: 'Supinated Wrist Curl', sets: 3, reps: '15–20', rir: 1, rest: '45s', tags: ['forearm'], note: 'Same setup as pronated but palms face UP. Forearm flexors (underside of forearm — thickness). Can go slightly heavier than pronated. Full ROM at bottom. Added Week 3.' }] : []),
      ...(w >= 4 ? [{ name: 'Cable Curl (straight bar)', sets: 3, reps: '12–15', rir: 1, rest: '90s', tags: ['isolation', 'new'], note: 'Constant tension from full stretch to full contraction — different strength curve than preacher. Total weekly bicep direct volume now at MAV. Added Week 4.' }] : []),
      ...(w >= 4 ? [{ name: 'Reverse Curl (EZ bar)', sets: 3, reps: '10–12', rir: 1, rest: '90s', tags: ['forearm', 'new'], note: 'Pronated grip (palms down), curl normally. Brachioradialis — the thick muscle on the outer forearm near the elbow. Visible from the front. Wrists stay neutral throughout. Added Week 4.' }] : []),
      ...(w >= 5 ? [{ name: 'Dead Hang', sets: 3, reps: 'Max time (30–60s)', rir: null, rest: '60s', tags: ['forearm', 'grip'], note: 'Passive hang. Builds grip + flexor tendons. Keep every week permanently.' }] : []),
    ],

    L2: [
      { name: 'Machine or Smith Hip Thrust', sets: getSets('L2', 'hip_thrust', w), reps: '8–12', rir: 2, rest: '2.5 min', tags: ['primary', 'weak-glutes'], note: 'Contreras 2015: highest mean glute EMG of any exercise (~229% MVC). Shoulders on bench, bar/pad over hips. Drive through heels, SQUEEZE at top. Full hip extension — but do not hyperextend lumbar.' },
      { name: 'Front Squat (heels elevated) OR Bulgarian Split Squat', sets: getSets('L2', 'front_sq', w), reps: '6–10 / 8–12', rir: 2, rest: '2.5 min', tags: ['primary', 'weak-quads'], note: 'Front squat: heels on 1" plate/board. Upright torso bypasses the long-femur problem. BSS: vertical torso = quad-biased; 15–20° lean = glute-biased. Your choice each session.' },
      { name: 'Lying Leg Curl', sets: getSets('L2', 'lying_curl', w), reps: '10–12', rir: 1, rest: '2 min', tags: ['primary'], note: 'Secondary hamstring angle. Hits BF short head differently than seated curl. Together with seated curl you cover the full hamstring.' },
      { name: '45° Back Extension (glute-biased)', sets: getSets('L2', 'back_ext', w), reps: '12–15', rir: 1, rest: '90s', tags: ['primary', 'weak-glutes'], note: 'CUES: upper back slightly rounded (chin tucked), weight held at chest, hinge from the hip. This makes it glute-dominant rather than lower-back dominant. Full stretch at bottom.' },
      { name: 'Hip Abduction Machine (forward lean ~30°)', sets: getSets('L2', 'hip_abduct', w), reps: '15–20', rir: 0, rest: '60s', tags: ['isolation', 'weak-glutes'], note: 'Lean forward 30° in the seat. This biases glute medius + upper glute max — the "shelf" at the top of the glute. Non-negotiable for the classical silhouette.' },
      { name: 'Seated Calf Raise', sets: getSets('L2', 'seat_calf', w), reps: '15–25', rir: 1, rest: '60s', tags: ['isolation', 'weak-calves'], note: 'Soleus (85% slow-twitch). High reps, constant stimulus. Full ROM — deep stretch at bottom, full contraction at top. 2s pause at stretch.' },
      { name: 'Cable Crunch OR Hanging Leg Raise', sets: 3, reps: '12–15', rir: 1, rest: '60s', tags: ['core'], note: 'Cable crunch: round the spine, not hip-hinge. Leg raise: straight legs, controlled — hip flexors should not be the limiting factor.' },
      ...(w >= 5 ? [{ name: 'DB Glute Bridge (banded)', sets: 2, reps: '15–20', rir: 0, rest: '60s', tags: ['isolation', 'new', 'weak-glutes'], note: 'Peak-week glute burnout. DB on hips, band above knees for abduction activation. Contract hard at top. Skip if hips are beat up.' }] : []),
    ],

    P5: [
      { name: 'Cable Lateral Raise (behind-back, single-arm)', sets: 4, reps: '12–15', rir: 1, rest: '90s', tags: ['primary', 'weak-side-delt'], note: 'Third weekly side-delt hit. Lengthened-position side delt — critical for your shoulder-to-waist ratio. Cable behind back gives a unique stretched-position stimulus.' },
      { name: 'Incline DB Press @ 30° (lighter, rep-focused)', sets: 3, reps: '12–15', rir: 1, rest: '2 min', tags: ['primary', 'weak-upper-chest'], note: 'Third upper chest stimulus. Less load, more mind-muscle. Focus on feeling the clavicular head — place a finger on your upper chest to feel it fire.' },
      { name: 'Cable Pullover (straight-arm, high pulley)', sets: 3, reps: '12–15', rir: 1, rest: '90s', tags: ['primary', 'weak-lats'], note: 'Lat stretch movement. Standing, cable at highest point, straight arms arc down. Full shoulder flexion at start = full lat stretch. Third weekly lat stimulus.' },
      { name: 'Reverse Pec Deck', sets: 3, reps: '15–20', rir: 0, rest: '60s', tags: ['isolation', 'weak-rear-delt'], note: 'Third weekly rear delt hit. Light, high reps. Focus on the rear delt contraction — not the traps.' },
      { name: 'Leg Press Calf Raise (lengthened partials finisher)', sets: 4, reps: '10–15', rir: 1, rest: '90s', tags: ['primary', 'weak-calves'], note: 'Kassiano 2023: bottom-half partials produce +14.9% lateral gastroc vs +7.3% full ROM. Do 1 full ROM set, then 2–3 bottom-half-only sets. Toes on the edge of the platform.' },
      { name: 'Incline DB Curl @ 45°', sets: 3, reps: '10–12', rir: 1, rest: '90s', tags: ['isolation'], note: 'Third weekly bicep hit on this day. Stretched long head. Slow and controlled.' },
      { name: 'Overhead Rope Triceps Extension', sets: 3, reps: '12–15', rir: 1, rest: '90s', tags: ['isolation'], note: 'Third weekly long-head triceps hit. Full stretch at the bottom — feel the tricep stretch behind the elbow.' },
      { name: 'Hip Abduction Machine (forward lean)', sets: 3, reps: '15–20', rir: 0, rest: '60s', tags: ['isolation', 'weak-glutes'], note: 'Third weekly glute medius/upper glute hit. The outer glute shelf that defines the classical silhouette from the front.' },
    ],
  };

  return programs[dayId] || [];
}

// Set count progression logic
function getSets(day, exercise, week) {
  const w = parseInt(week);
  // Base sets for week 1, accumulate over mesocycle
  const base = {
    U1: { incline_press: 3, pullup: 3, csrow: 3, cable_fly: 3, lat_raise_cable: 3, rev_pec: 3, oh_tri: 3, incline_curl: 3 },
    L1: { hack_squat: 3, rdl: 3, leg_press: 3, seated_curl: 3, stand_calf: 3 },
    U2: { ohp: 3, pulldown: 3, flat_db: 3, cable_row: 3, db_lateral: 3, rear_delt: 3, preacher: 3, cg_bench: 3 },
    L2: { hip_thrust: 3, front_sq: 3, lying_curl: 3, back_ext: 3, hip_abduct: 3, seat_calf: 3 },
  };
  // Progression: specific muscles get sets added earlier
  const progression = {
    // Side delts get volume first
    lat_raise_cable: [3, 4, 4, 4, 4],
    db_lateral: [3, 4, 4, 4, 4],
    // Week 3 compounds and key muscles
    incline_press: [3, 3, 4, 4, 4],
    csrow: [3, 3, 4, 4, 4],
    flat_db: [3, 3, 4, 4, 4],
    cable_row: [3, 3, 4, 4, 4],
    pulldown: [3, 4, 4, 4, 4],
    // Week 2 priority muscles
    hack_squat: [3, 4, 4, 4, 4],
    stand_calf: [3, 4, 4, 5, 5],
    seated_curl: [3, 3, 4, 4, 4],
    seat_calf: [3, 4, 4, 4, 4],
    hip_thrust: [3, 4, 4, 4, 4],
    hip_abduct: [3, 3, 4, 4, 4],
    // Week 5 peak bumps
    rev_pec: [3, 3, 3, 3, 4],
    rear_delt: [3, 3, 3, 3, 4],
    oh_tri: [3, 3, 3, 3, 4],
    cg_bench: [3, 3, 3, 3, 4],
    back_ext: [3, 3, 3, 4, 4],
    front_sq: [3, 3, 3, 4, 4],
    // Standard
    pullup: [3, 3, 3, 4, 4],
    leg_press: [3, 3, 3, 4, 4],
    lying_curl: [3, 3, 3, 4, 4],
    rdl: [3, 3, 4, 4, 4],
    ohp: [3, 3, 4, 4, 4],
    preacher: [3, 3, 3, 3, 3],
    cable_fly: [3, 3, 3, 3, 3],
    incline_curl: [3, 3, 3, 3, 3],
  };

  if (w === 6) {
    // Deload: half sets
    const prog = progression[exercise];
    const maxSets = prog ? prog[4] : (base[day] ? base[day][exercise] : 3);
    return Math.ceil(maxSets / 2);
  }

  const prog = progression[exercise];
  if (prog) return prog[Math.min(w - 1, 4)];
  return base[day] ? (base[day][exercise] || 3) : 3;
}

// =============================================
// STATE
// =============================================
let state = JSON.parse(localStorage.getItem('david_state') || 'null') || {
  currentWeek: 1,
  currentDay: null,
  completedDays: {}, // { "2024-01-15": ["U1"] }
  logs: [], // { date, dayId, dayName, totalSets, totalTonnage }
  benchmarks: {
    bench_185: false,
    pullup_45: false,
    bw_175: false,
    hack_2x: false,
    rdl_175: false,
    incline_85: false,
    calf_4p: false,
  },
  weights: {}, // { "U1_0_0": "60" } (day_exIndex_setIndex)
  reps: {},
  setsDone: {}, // { "U1_0_0": true }
  streak: 0,
  lastTrainDate: null,
  calYear: new Date().getFullYear(),
  calMonth: new Date().getMonth(),
  startDate: null, // first ever training date
};

function save() {
  localStorage.setItem('david_state', JSON.stringify(state));
  if (window._currentUser) {
    syncToFirebase();
  }
}

// =============================================
// FIREBASE SYNC
// =============================================
let _syncTimer = null;

function syncToFirebase() {
  // Debounce — wait 1.5s after last change before writing
  clearTimeout(_syncTimer);
  setSyncStatus('syncing');
  _syncTimer = setTimeout(async () => {
    try {
      const uid = window._currentUser.uid;
      const db = window._fbDb;
      const docRef = window._fbDoc(db, 'users', uid, 'data', 'state');
      await window._fbSetDoc(docRef, {
        state: JSON.stringify(state),
        updatedAt: window._fbServerTimestamp()
      });
      setSyncStatus('synced');
    } catch (e) {
      console.error('Sync error:', e);
      setSyncStatus('offline');
    }
  }, 1500);
}

async function loadFromFirebase(uid) {
  try {
    const db = window._fbDb;
    const docRef = window._fbDoc(db, 'users', uid, 'data', 'state');
    const snap = await window._fbGetDoc(docRef);
    if (snap.exists()) {
      const remoteState = JSON.parse(snap.data().state);
      // Merge remote into local — remote wins for cross-device data
      state = { ...state, ...remoteState };
      localStorage.setItem('david_state', JSON.stringify(state));
      // Re-render everything with loaded data
      document.getElementById('weekSelect').value = state.currentWeek;
      renderDayGrid();
      renderPhaseDots();
      renderStats();
      renderBenchmarks();
      renderRecentLog();
      renderCalendar();
      if (state.currentDay) renderSession(state.currentDay);
      setSyncStatus('synced');
      showToast('Progress loaded from your account ☁️', true);
    } else {
      // First time — push local state to Firebase
      syncToFirebase();
    }
  } catch (e) {
    console.error('Load error:', e);
    setSyncStatus('offline');
  }
}

async function logSessionToFirebase(sessionData) {
  if (!window._currentUser) return;
  try {
    const uid = window._currentUser.uid;
    const db = window._fbDb;
    const colRef = window._fbCollection(db, 'users', uid, 'sessions');
    await window._fbAddDoc(colRef, {
      ...sessionData,
      loggedAt: window._fbServerTimestamp()
    });
  } catch (e) {
    console.error('Session log error:', e);
  }
}

function setSyncStatus(status) {
  const indicator = document.getElementById('syncIndicator');
  const dot = document.getElementById('syncDot');
  const label = document.getElementById('syncLabel');
  if (!indicator) return;
  indicator.style.display = 'flex';
  dot.className = 'sync-dot';
  if (status === 'syncing') {
    dot.classList.add('syncing');
    label.textContent = 'Saving...';
  } else if (status === 'synced') {
    label.textContent = 'Synced';
  } else {
    dot.classList.add('offline');
    label.textContent = 'Offline';
  }
}

// =============================================
// AUTH UI
// =============================================
function renderUserHeader(user) {
  document.getElementById('loginOverlay').style.display = 'none';
  document.getElementById('syncIndicator').style.display = 'flex';
  document.getElementById('authArea').innerHTML = `
    <button class="auth-btn signout" onclick="signOutUser()">
      <img src="${user.photoURL || ''}" onerror="this.style.display='none'" alt="">
      ${user.displayName ? user.displayName.split(' ')[0] : 'You'} · Sign out
    </button>
  `;
}

function renderSignedOutHeader() {
  document.getElementById('syncIndicator').style.display = 'none';
  document.getElementById('authArea').innerHTML = `
    <button class="auth-btn" onclick="signInGoogle()">Sign in with Google</button>
  `;
}

async function signInGoogle() {
  try {
    await window._fbSignInWithPopup(window._fbAuth, window._fbGoogleProvider);
    // onAuthStateChanged will handle the rest
  } catch (e) {
    console.error('Sign in error:', e);
    showToast('Sign in failed — try again.');
  }
}

async function signOutUser() {
  if (!confirm('Sign out? Your data is saved to your account.')) return;
  await window._fbSignOut(window._fbAuth);
  window._currentUser = null;
  showToast('Signed out.');
}

function skipLogin() {
  document.getElementById('loginOverlay').style.display = 'none';
  showToast('Running without sync. Data saves locally only.');
}

// =============================================
// INIT
// =============================================
function init() {
  renderDayGrid();
  renderPhaseDots();
  renderStats();
  renderBenchmarks();
  renderRecentLog();
  renderCalendar();

  document.getElementById('weekSelect').value = state.currentWeek;
  document.getElementById('statWeek').textContent = state.currentWeek;

  if (state.currentDay) {
    renderSession(state.currentDay);
    highlightDay(state.currentDay);
  }

  // Login overlay stays visible until Firebase resolves auth state
  // onAuthStateChanged will hide it if user is signed in
  // skipLogin() hides it if user chooses to continue without signing in
  // Give Firebase 3 seconds to resolve before allowing skip
  setTimeout(() => {
    const overlay = document.getElementById('loginOverlay');
    if (overlay && overlay.style.display !== 'none') {
      // Auth is slow or user not signed in — keep overlay, just ensure skip is visible
    }
  }, 3000);
}

// =============================================
// DAY GRID
// =============================================
function renderDayGrid() {
  const grid = document.getElementById('dayGrid');
  const today = todayStr();
  grid.innerHTML = DAYS.map(d => {
    const completedToday = state.completedDays[today] && state.completedDays[today].includes(d.id);
    return `
      <div class="day-btn ${state.currentDay === d.id ? 'active' : ''} ${completedToday ? 'completed' : ''}"
           onclick="selectDay('${d.id}')">
        <span class="day-label">${d.label}</span>
        <span class="day-name">${d.name}</span>
        <span class="day-tag">${d.tag}</span>
        ${completedToday ? '<span style="font-size:10px;color:var(--green);display:block;margin-top:3px;">✓ Done today</span>' : ''}
      </div>
    `;
  }).join('');
}

function highlightDay(dayId) {
  document.querySelectorAll('.day-btn').forEach(b => b.classList.remove('active'));
  document.querySelectorAll('.day-btn').forEach(b => {
    if (b.querySelector('.day-name') && DAYS.find(d => d.id === dayId && b.querySelector('.day-name').textContent === d.name)) {
      b.classList.add('active');
    }
  });
}

function selectDay(dayId) {
  state.currentDay = dayId;
  save();
  renderDayGrid();
  renderSession(dayId);
  window.scrollTo({ top: 0, behavior: 'smooth' });
}

function pickMakeupDay() {
  const day = prompt('Enter day ID to make up (U1, L1, U2, L2, P5):');
  if (day && DAYS.find(d => d.id === day.toUpperCase())) {
    selectDay(day.toUpperCase());
    showToast('Makeup day loaded — crush it!');
  }
}

// =============================================
// SESSION RENDER
// =============================================
function renderSession(dayId) {
  const day = DAYS.find(d => d.id === dayId);
  const week = state.currentWeek;
  const exercises = getExercises(dayId, week);
  const isDeload = parseInt(week) === 6;

  const content = document.getElementById('mainContent');

  const completedSetsCount = exercises.reduce((acc, ex, ei) => {
    for (let si = 0; si < ex.sets; si++) {
      if (state.setsDone[`${dayId}_${ei}_${si}`]) acc++;
    }
    return acc;
  }, 0);

  const totalSets = exercises.reduce((acc, ex) => acc + ex.sets, 0);
  const sessionComplete = completedSetsCount === totalSets && totalSets > 0;

  // Volume note for current week
  const volumeNotes = {
    1: 'Week 1 — Establish baseline. No failure. Focus on form and feeling each muscle.',
    2: 'Week 2 — First volume addition. Side delts and calves get an extra set. Add reps before adding weight.',
    3: 'Week 3 — Compounds get their second set addition. Forearm work begins.',
    4: 'Week 4 — Second direct bicep exercise added. Reverse curl for brachioradialis. Approaching MAV.',
    5: 'Week 5 — Peak volume. Push near failure on isolations. Expect to feel cooked.',
    6: 'Week 6 — DELOAD. 50% sets, 60% load. This is mandatory recovery.',
  };

  let html = `
    <div class="session-header">
      <div>
        <div class="session-title">${day.name}</div>
        <div class="session-subtitle">${day.tag} · Week ${week} · ${totalSets} total working sets</div>
      </div>
      <div class="session-actions">
        <button class="btn btn-primary" onclick="markSessionComplete('${dayId}')">Mark Complete ✓</button>
        <button class="btn btn-secondary" onclick="clearSession('${dayId}')">Clear Session</button>
      </div>
    </div>
  `;

  if (sessionComplete) {
    html += `
      <div class="session-complete-banner">
        <div class="sc-icon">🏛️</div>
        <div class="sc-text">
          <h4>Session Complete</h4>
          <p>Log it and move on. Recovery starts now.</p>
        </div>
      </div>
    `;
  }

  if (isDeload) {
    html += `
      <div class="deload-banner">
        <h3>⚡ Deload Week</h3>
        <p>This is not optional. Your connective tissue and CNS need this to grow.</p>
        <div class="deload-rules">
          <div class="deload-rule">50% of normal sets</div>
          <div class="deload-rule">60% of normal load</div>
          <div class="deload-rule">RIR 4+ always</div>
          <div class="deload-rule">No failure</div>
        </div>
      </div>
    `;
  } else {
    html += `<div class="volume-note"><div class="vn-icon">📈</div><div><strong>Week ${week}:</strong> ${volumeNotes[week]}</div></div>`;
  }

  exercises.forEach((ex, ei) => {
    const isNew = ex.tags && ex.tags.includes('new');
    const isWeak = ex.tags && ex.tags.some(t => t.startsWith('weak-'));
    const isForearm = ex.tags && ex.tags.includes('forearm');

    let tagHtml = '';
    if (isNew) tagHtml += '<span class="tag tag-new">+ Added this week</span>';
    if (isWeak) tagHtml += '<span class="tag tag-weak">Priority</span>';
    if (isForearm) tagHtml += '<span class="tag tag-primary">Forearm</span>';

    const rirText = ex.rir !== null && ex.rir !== undefined ? `RIR ${ex.rir}` : 'Max effort';
    tagHtml += `<span class="tag tag-rpe">${rirText} · ${ex.rest}</span>`;

    // Build sets rows
    let setsHtml = `
      <table class="sets-table">
        <thead>
          <tr>
            <th>Set</th>
            <th>Target</th>
            <th>Weight (lbs)</th>
            <th>Reps done</th>
            <th>PR?</th>
            <th>✓</th>
          </tr>
        </thead>
        <tbody>
    `;

    for (let si = 0; si < ex.sets; si++) {
      const key = `${dayId}_${ei}_${si}`;
      const wKey = `w_${dayId}_${ei}_${si}`;
      const rKey = `r_${dayId}_${ei}_${si}`;
      const prevKey = `prev_${dayId}_${ei}_${si}`;

      const savedWeight = state.weights[wKey] || '';
      const savedReps = state.reps[rKey] || '';
      const isDone = state.setsDone[key] || false;

      // Check for PR (weight higher than any previous for this exercise/set)
      const prevBest = state.weights[prevKey] || 0;
      const isPR = savedWeight && parseInt(savedWeight) > parseInt(prevBest);

      const deloadTarget = isDeload ? ` (${Math.round((parseInt(savedWeight) || 100) * 0.6)} lbs)` : '';

      setsHtml += `
        <tr style="${isDone ? 'background:rgba(92,122,58,0.04);' : ''}">
          <td class="set-num">${si + 1}</td>
          <td class="set-target">${ex.sets}×${ex.reps}${deloadTarget}</td>
          <td>
            <input type="number" class="weight-input" placeholder="lbs"
              value="${savedWeight}"
              onchange="saveWeight('${dayId}', ${ei}, ${si}, this.value)"
              oninput="saveWeight('${dayId}', ${ei}, ${si}, this.value)">
          </td>
          <td>
            <input type="number" class="reps-input" placeholder="reps"
              value="${savedReps}"
              onchange="saveReps('${dayId}', ${ei}, ${si}, this.value)"
              oninput="saveReps('${dayId}', ${ei}, ${si}, this.value)">
          </td>
          <td class="set-pr">${isPR ? '🏆 PR!' : ''}</td>
          <td>
            <div class="set-check ${isDone ? 'done' : ''}" onclick="toggleSet('${dayId}', ${ei}, ${si})"></div>
          </td>
        </tr>
      `;
    }

    setsHtml += '</tbody></table>';

    html += `
      <div class="exercise-card" id="ex_${dayId}_${ei}">
        <div class="ex-head" onclick="toggleEx(this)">
          <span class="ex-num">${ei + 1}</span>
          <span class="ex-name">${ex.name}</span>
          <div class="ex-tags">${tagHtml}</div>
          <span class="ex-toggle">▼</span>
        </div>
        <div class="ex-body open" id="exbody_${dayId}_${ei}">
          <p class="ex-note">${ex.note}</p>
          ${setsHtml}
        </div>
      </div>
    `;
  });

  content.innerHTML = html;
}

// =============================================
// INTERACTIONS
// =============================================
function toggleEx(head) {
  const body = head.nextElementSibling;
  head.classList.toggle('open');
  body.classList.toggle('open');
}

function saveWeight(dayId, ei, si, val) {
  const wKey = `w_${dayId}_${ei}_${si}`;
  state.weights[wKey] = val;
  // Check for PR
  const prevKey = `prev_${dayId}_${ei}_${si}`;
  if (!state.weights[prevKey] || parseInt(val) > parseInt(state.weights[prevKey])) {
    // Will be checked on render
  }
  save();
}

function saveReps(dayId, ei, si, val) {
  const rKey = `r_${dayId}_${ei}_${si}`;
  state.reps[rKey] = val;
  save();
}

function toggleSet(dayId, ei, si) {
  const key = `${dayId}_${ei}_${si}`;
  state.setsDone[key] = !state.setsDone[key];
  save();

  // Re-render just the check
  const check = document.querySelector(`#ex_${dayId}_${ei} .sets-table tbody tr:nth-child(${si + 1}) .set-check`);
  if (check) {
    check.classList.toggle('done', state.setsDone[key]);
  }

  // Check if PR
  const wKey = `w_${dayId}_${ei}_${si}`;
  const prevKey = `prev_${dayId}_${ei}_${si}`;
  const currentWeight = parseInt(state.weights[wKey] || 0);
  const prevBest = parseInt(state.weights[prevKey] || 0);
  if (state.setsDone[key] && currentWeight > prevBest) {
    state.weights[prevKey] = currentWeight;
    save();
    if (currentWeight > 0) showToast(`🏆 New PR — ${currentWeight} lbs!`, true);
  }

  // Update session stats
  renderStats();
}

function markSessionComplete(dayId) {
  const today = todayStr();
  if (!state.completedDays[today]) state.completedDays[today] = [];
  if (!state.completedDays[today].includes(dayId)) {
    state.completedDays[today].push(dayId);
  }

  // Record the very first training date
  if (!state.startDate) state.startDate = today;

  // Calculate tonnage for this session
  const exercises = getExercises(dayId, state.currentWeek);
  let tonnage = 0;
  exercises.forEach((ex, ei) => {
    for (let si = 0; si < ex.sets; si++) {
      const wKey = `w_${dayId}_${ei}_${si}`;
      const rKey = `r_${dayId}_${ei}_${si}`;
      const w = parseFloat(state.weights[wKey] || 0);
      const r = parseFloat(state.reps[rKey] || 0);
      tonnage += w * r;
    }
  });

  // Update streak
  const lastDate = state.lastTrainDate;
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  const yStr = yesterday.toISOString().split('T')[0];

  if (lastDate === yStr || lastDate === today) {
    if (lastDate !== today) state.streak = (state.streak || 0) + 1;
  } else {
    state.streak = 1;
  }
  state.lastTrainDate = today;

  // Log entry
  const day = DAYS.find(d => d.id === dayId);
  state.logs.unshift({
    date: today,
    dayId,
    dayName: day.name,
    week: state.currentWeek,
    totalSets: exercises.reduce((a, e) => a + e.sets, 0),
    tonnage: Math.round(tonnage / 2000 * 10) / 10,
  });

  // Keep last 20 logs
  if (state.logs.length > 20) state.logs = state.logs.slice(0, 20);

  // Log detailed session to Firebase sessions collection
  logSessionToFirebase({
    date: today,
    dayId,
    dayName: day.name,
    week: state.currentWeek,
    totalSets: exercises.reduce((a, e) => a + e.sets, 0),
    tonnage: Math.round(tonnage / 2000 * 10) / 10,
    exercises: exercises.map((ex, ei) => ({
      name: ex.name,
      sets: Array.from({ length: ex.sets }, (_, si) => ({
        set: si + 1,
        weight: state.weights[`w_${dayId}_${ei}_${si}`] || null,
        reps: state.reps[`r_${dayId}_${ei}_${si}`] || null,
      }))
    }))
  });

  save();
  renderDayGrid();
  renderStats();
  renderRecentLog();
  renderCalendar();
  renderSession(dayId);
  showToast('Session logged. Rest, eat, grow. 🏛️', true);
}

function clearSession(dayId) {
  if (!confirm('Clear all weights and reps for this session?')) return;
  const exercises = getExercises(dayId, state.currentWeek);
  exercises.forEach((ex, ei) => {
    for (let si = 0; si < ex.sets; si++) {
      delete state.weights[`w_${dayId}_${ei}_${si}`];
      delete state.reps[`r_${dayId}_${ei}_${si}`];
      delete state.setsDone[`${dayId}_${ei}_${si}`];
    }
  });
  save();
  renderSession(dayId);
  showToast('Session cleared.');
}

// =============================================
// WEEK CONTROL
// =============================================
function setWeek(w) {
  state.currentWeek = parseInt(w);
  document.getElementById('statWeek').textContent = w;
  save();
  renderStats();
  renderPhaseDots();
  if (state.currentDay) renderSession(state.currentDay);
  showToast(`Week ${w} loaded.`);
}

// =============================================
// STATS
// =============================================
function renderStats() {
  const totalSessions = state.logs.length;
  const totalTonnage = state.logs.reduce((a, l) => a + (l.tonnage || 0), 0);

  document.getElementById('statSessions').textContent = totalSessions;
  document.getElementById('statTonnage').textContent = Math.round(totalTonnage * 10) / 10;
  document.getElementById('statStreak').textContent = state.streak || 0;
  document.getElementById('statWeek').textContent = state.currentWeek;

  const weekPct = Math.round((state.currentWeek / 6) * 100);
  document.getElementById('weekPct').textContent = weekPct + '%';
  document.getElementById('weekFill').style.width = weekPct + '%';
}

function renderPhaseDots() {
  const dots = document.getElementById('phaseDots');
  dots.innerHTML = [1,2,3,4,5,6].map(w => {
    let cls = '';
    if (w < state.currentWeek) cls = 'done';
    else if (w == state.currentWeek) cls = 'current';
    return `<div class="phase-dot ${cls}" title="Week ${w}${w===6?' (Deload)':''}"></div>`;
  }).join('');
}

// =============================================
// BENCHMARKS
// =============================================
const BENCHMARKS = [
  { key: 'bench_185', name: 'Bench Press', target: '185 lb × 5 @ ≤2 RIR' },
  { key: 'pullup_45', name: 'Weighted Pull-up', target: 'BW + 45 lb × 5' },
  { key: 'bw_175', name: 'Bodyweight', target: '175+ lb lean (~12% BF)' },
  { key: 'hack_2x', name: 'Hack Squat', target: '2× BW × 8 @ 1 RIR' },
  { key: 'rdl_175', name: 'RDL', target: '1.75× BW × 8' },
  { key: 'incline_85', name: 'Incline DB Press', target: '85 lb × 8' },
  { key: 'calf_4p', name: 'Standing Calf Raise', target: '4 plates × 12 (2s pause)' },
];

function renderBenchmarks() {
  const container = document.getElementById('benchmarkList');
  container.innerHTML = BENCHMARKS.map(b => `
    <div class="benchmark-item">
      <div class="bm-check ${state.benchmarks[b.key] ? 'done' : ''}"
           onclick="toggleBenchmark('${b.key}')">${state.benchmarks[b.key] ? '✓' : ''}</div>
      <div class="bm-name">${b.name}</div>
      <div class="bm-target">${b.target}</div>
    </div>
  `).join('');

  const done = Object.values(state.benchmarks).filter(Boolean).length;
  if (done === BENCHMARKS.length) {
    container.innerHTML += `<div style="text-align:center;padding:12px;font-size:11px;color:var(--green);border-top:1px solid var(--border);margin-top:8px;">🏛️ Phase 3 Ready</div>`;
  } else {
    container.innerHTML += `<div style="text-align:center;padding:8px;font-size:10px;color:var(--muted);">${done}/${BENCHMARKS.length} benchmarks hit</div>`;
  }
}

function toggleBenchmark(key) {
  state.benchmarks[key] = !state.benchmarks[key];
  save();
  renderBenchmarks();
  if (state.benchmarks[key]) showToast('Benchmark achieved! 🏆', true);
}

// =============================================
// RECENT LOG
// =============================================
function renderRecentLog() {
  const container = document.getElementById('recentLog');
  if (state.logs.length === 0) {
    container.innerHTML = '<div style="padding:14px; font-size:11px; color:var(--muted); text-align:center;">No sessions logged yet.</div>';
    return;
  }
  container.innerHTML = state.logs.slice(0, 5).map((l, i) => `
    <div class="log-entry">
      <div>
        <div class="log-day">${l.dayName} · Wk ${l.week}</div>
        <div class="log-date">${l.date} · ${l.totalSets} sets · ${l.tonnage}t</div>
      </div>
      <button class="log-delete" onclick="deleteLog(${i})">✕</button>
    </div>
  `).join('');
}

function deleteLog(i) {
  state.logs.splice(i, 1);
  save();
  renderRecentLog();
  renderStats();
}

// =============================================
// UTILS
// =============================================
function todayStr() {
  return new Date().toISOString().split('T')[0];
}

function showToast(msg, success = false) {
  const t = document.getElementById('toast');
  t.textContent = msg;
  t.className = 'toast show' + (success ? ' success' : '');
  setTimeout(() => { t.className = 'toast'; }, 2800);
}

// =============================================
// CALENDAR
// =============================================

const MONTH_NAMES = ['January','February','March','April','May','June','July','August','September','October','November','December'];
const DOW = ['Su','Mo','Tu','We','Th','Fr','Sa'];

// Day type color map for dots
const DAY_COLORS = { U1:'#5c7a3a', L1:'#8b6f47', U2:'#4a7a8b', L2:'#8b4a6f', P5:'#9b6a1a' };
const DAY_ABBR = { U1:'UA', L1:'LA', U2:'UB', L2:'LB', P5:'P5' };

function renderCalendar() {
  const panel = document.getElementById('calendarPanel');
  if (!panel) return;

  // Ensure state has cal props
  if (state.calYear == null) state.calYear = new Date().getFullYear();
  if (state.calMonth == null) state.calMonth = new Date().getMonth();

  const year = state.calYear;
  const month = state.calMonth;
  const today = new Date();
  const todayISO = todayStr();

  const firstDay = new Date(year, month, 1);
  const lastDay = new Date(year, month + 1, 0);
  const startDow = firstDay.getDay(); // 0=Sun
  const daysInMonth = lastDay.getDate();

  // Build day cells
  let cells = '';
  // Day-of-week headers
  DOW.forEach(d => { cells += `<div class="cal-dow">${d}</div>`; });

  // Empty cells before month start
  for (let i = 0; i < startDow; i++) {
    cells += `<div class="cal-day empty"></div>`;
  }

  for (let d = 1; d <= daysInMonth; d++) {
    const iso = `${year}-${String(month+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
    const isToday = iso === todayISO;
    const trainedDays = state.completedDays[iso] || [];
    const wasTrained = trainedDays.length > 0;
    const isFuture = iso > todayISO;

    let cls = 'cal-day';
    if (isToday) cls += ' today';
    else if (wasTrained) cls += ' trained';
    else if (!isFuture) cls += ' rest';

    // Dots for each day type done
    const dots = trainedDays.map(id =>
      `<div class="cal-dot" style="background:${DAY_COLORS[id]||'var(--green)'}" title="${id}"></div>`
    ).join('');

    const tooltip = wasTrained ? trainedDays.map(id => DAY_ABBR[id] || id).join(', ') : (isFuture ? '' : 'Rest');

    cells += `
      <div class="${cls}" title="${tooltip}">
        <span class="cal-day-num">${d}</span>
        ${wasTrained ? `<div class="cal-dot-row">${dots}</div>` : ''}
      </div>`;
  }

  // Fill remaining cells to complete last row
  const totalCells = startDow + daysInMonth;
  const remainder = totalCells % 7;
  if (remainder !== 0) {
    for (let i = 0; i < 7 - remainder; i++) {
      cells += `<div class="cal-day empty"></div>`;
    }
  }

  // ---- Stats ----
  // Total training days this month
  let monthTrained = 0;
  for (let d = 1; d <= daysInMonth; d++) {
    const iso = `${year}-${String(month+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
    if ((state.completedDays[iso] || []).length > 0) monthTrained++;
  }

  // Days since first session
  let daysSinceStart = '—';
  if (state.startDate) {
    const start = new Date(state.startDate);
    const diff = Math.floor((today - start) / (1000*60*60*24));
    daysSinceStart = diff;
  }

  // Total lifetime training days
  const lifetimeDays = Object.values(state.completedDays).filter(v => v.length > 0).length;

  // ---- 90-day heatmap ----
  let heatCells = '';
  for (let i = 89; i >= 0; i--) {
    const d = new Date(today);
    d.setDate(d.getDate() - i);
    const iso = d.toISOString().split('T')[0];
    const count = (state.completedDays[iso] || []).length;
    let intensity = 'd0';
    if (count === 1) intensity = 'd1';
    else if (count === 2) intensity = 'd2';
    else if (count === 3) intensity = 'd3';
    else if (count >= 4) intensity = 'd4';
    const label = count > 0 ? `${iso}: ${count} session(s)` : iso;
    heatCells += `<div class="heatmap-cell ${intensity}" title="${label}"></div>`;
  }

  panel.innerHTML = `
    <div class="cal-nav">
      <button onclick="calPrev()">‹</button>
      <span class="cal-month">${MONTH_NAMES[month]} ${year}</span>
      <button onclick="calNext()">›</button>
    </div>
    <div class="cal-grid">${cells}</div>
    <div class="cal-legend">
      <div class="cal-legend-item">
        <div class="cal-legend-dot" style="background:var(--green-bg);border:1px solid var(--green-light);"></div>
        Trained
      </div>
      <div class="cal-legend-item">
        <div class="cal-legend-dot" style="background:var(--bg2);border:1px solid var(--border);"></div>
        Rest
      </div>
      <div class="cal-legend-item">
        <div class="cal-legend-dot" style="background:#fdf5e4;border:1px solid var(--gold);"></div>
        Today
      </div>
    </div>
    <div class="cal-stats-row">
      <div class="cal-stat">
        <div class="cal-stat-val">${monthTrained}</div>
        <div class="cal-stat-label">This Month</div>
      </div>
      <div class="cal-stat">
        <div class="cal-stat-val">${lifetimeDays}</div>
        <div class="cal-stat-label">Total Days</div>
      </div>
      <div class="cal-stat">
        <div class="cal-stat-val">${daysSinceStart}</div>
        <div class="cal-stat-label">Day ${typeof daysSinceStart === 'number' ? daysSinceStart + 1 : '1'}</div>
      </div>
    </div>
    <div class="heatmap-wrap">
      <div class="heatmap-title">Last 90 Days</div>
      <div class="heatmap-row">${heatCells}</div>
    </div>
  `;
}

function calPrev() {
  if (state.calMonth === 0) { state.calMonth = 11; state.calYear--; }
  else state.calMonth--;
  save();
  renderCalendar();
}

function calNext() {
  if (state.calMonth === 11) { state.calMonth = 0; state.calYear++; }
  else state.calMonth++;
  save();
  renderCalendar();
}

// =============================================
// BOOT
// =============================================
init();
</script>
</body>
</html>
