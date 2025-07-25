<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <title>D2B1 - Rotation en ligne</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="theme-color" content="#004080" />
  <link rel="icon" type="image/png" href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAOAAAADgCAYAAACJ4C9RAAAACXBIWXMAAAsTAAALEwEAmpwYAAA...=="/>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; background: #f0f4f9; padding: 20px; margin: 0; }
    .card { background: white; padding: 20px; margin: 20px auto; max-width: 600px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    h1 { text-align: center; color: #004080; font-size: 1.5em; }
    input, button { width: 100%; padding: 10px; margin-top: 10px; font-size: 16px; box-sizing: border-box; cursor: pointer; }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; font-size: 0.9em; }
    th, td { border: 1px solid #ccc; padding: 6px; text-align: center; }
    th { background-color: #004080; color: white; }
    .btn-danger { background-color: #e74c3c; color: white; }
    .btn-primary { background-color: #004080; color: white; }
    .btn-secondary { background-color: #2980b9; color: white; }
    #forgotSection { text-align: center; margin-top: 10px; font-size: 0.9em; color: #555; cursor: pointer; }
    @media (max-width: 600px) {
      input, button { font-size: 14px; padding: 8px; }
      table { font-size: 0.8em; }
    }
  </style>
</head>
<body>
  <h1>Système Rotatif Dream2Build1</h1>

  <div class="card" id="authCard">
    <input type="password" id="adminPass" placeholder="Mot de passe admin" autocomplete="current-password" />
    <button onclick="checkAdminStep1()" class="btn-primary">🔐 Connexion</button>
    <div id="forgotSection" onclick="forgotPassword()">Mot de passe oublié ?</div>
  </div>

  <div class="card" id="codeCard" style="display:none;">
    <input type="text" id="code2FA" placeholder="Code de vérification (123456)" autocomplete="one-time-code" />
    <button onclick="checkAdminStep2()" class="btn-primary">✅ Vérifier</button>
  </div>

  <div class="card" id="mainCard" style="display:none;">
    <input type="text" id="participantName" placeholder="Nom du participant" />
    <button onclick="addParticipant()" class="btn-primary">➕ Ajouter</button>
    <button onclick="validatePayments()" class="btn-secondary">✅ Valider Paiement</button>
    <button onclick="nextDay()" class="btn-secondary">🔁 Jour Suivant</button>
    <button onclick="exportToExcel()" class="btn-primary">📤 Export Historique</button>
    <button onclick="resetData()" class="btn-danger">🗑️ Réinitialiser Données</button>

    <div id="stats" style="margin-top:20px;"></div>
    <div id="historyTable" style="margin-top:20px;"></div>
    <div id="participantsList" style="margin-top:20px;"></div>
  </div>

<script>
const API_URL = "https://script.google.com/macros/s/AKfycbzVSp0Mr7uRkJLqpOVloqaBJHmDCGK1zUgQPBRK5EqA4gzHsZF3KM2WLwkmIzD2kiI9/exec";
const ADMIN_PASSWORD = "d2b1admin";
const SECOND_CODE = "123456";
let participants = [], history = [], currentDay = 1;

function checkAdminStep1() {
  const pass = document.getElementById("adminPass").value;
  if (pass === ADMIN_PASSWORD) {
    document.getElementById("authCard").style.display = "none";
    document.getElementById("codeCard").style.display = "block";
  } else alert("Mot de passe incorrect");
}

function checkAdminStep2() {
  const code = document.getElementById("code2FA").value.trim();
  if (code === SECOND_CODE) {
    document.getElementById("codeCard").style.display = "none";
    document.getElementById("mainCard").style.display = "block";
    loadData();
  } else alert("Code de vérification invalide");
}

function forgotPassword() {
  window.location.href = "mailto:emmanueljeambas@gmail.com?subject=Réinitialisation%20mot%20de%20passe";
}

async function loadData() {
  const res = await fetch(`${API_URL}?action=get`);
  const data = await res.json();
  participants = data.participants || [];
  history = data.history || [];
  currentDay = history.length + 1;
  renderParticipants();
  renderHistory();
  updateStats();
}

function renderParticipants() {
  const ul = document.getElementById("participantsList");
  ul.innerHTML = participants.length
    ? "<h3>Participants</h3><ul>" + participants.map(p => `<li>Jour ${p.day} – ${p.name}</li>`).join('') + "</ul>"
    : "<p>Aucun participant.</p>";
}

function renderHistory() {
  const div = document.getElementById("historyTable");
  if (!history.length) return div.innerHTML = "<p>Aucun historique.</p>";
  let html = "<h3>Historique</h3><table><tr><th>Jour</th><th>Bénéficiaire</th><th>Total</th><th>1/3</th><th>10%</th><th>Retenu</th><th>Payé</th><th>Date & Heure</th></tr>";
  history.forEach(r => html += `<tr><td>${r.jour}</td><td>${r.beneficiary}</td><td>${r.total}</td><td>${r.oneThird}</td><td>${r.tenPercent}</td><td>${r.retained}</td><td>${r.payout}</td><td>${r.dateTime}</td></tr>`);
  div.innerHTML = html + "</table>";
}

async function addParticipant() {
  const name = document.getElementById("participantName").value.trim();
  if (!name) return alert("Nom requis");
  const day = participants.length + 1;
  await fetch(API_URL, {
    method: "POST",
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: "addParticipant", name, day })
  });
  document.getElementById("participantName").value = "";
  loadData();
}

async function validatePayments() {
  if (!participants.length) return alert("Aucun participant");
  const jour = currentDay, total = participants.length * 10;
  const oneThird = total / 3, tenPercent = oneThird * 0.1;
  const retained = oneThird - tenPercent, payout = total - retained;
  const beneficiary = participants[(jour - 1) % participants.length].name;
  const dateTime = new Date().toLocaleString("fr-FR", { hour12: false });
  await fetch(API_URL, {
    method: "POST",
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: "addHistory", jour, beneficiary, total: total.toFixed(2), oneThird: oneThird.toFixed(2), tenPercent: tenPercent.toFixed(2), retained: retained.toFixed(2), payout: payout.toFixed(2), dateTime })
  });
  currentDay++;
  loadData();
}

function nextDay() {
  currentDay++;
  updateStats();
}

function updateStats() {
  const stats = document.getElementById("stats");
  if (!participants.length) return stats.innerHTML = "<p>Aucune donnée</p>";
  const jour = currentDay, total = participants.length * 10;
  const oneThird = total / 3, tenPercent = oneThird * 0.1;
  const retained = oneThird - tenPercent, payout = total - retained;
  const beneficiary = participants[(jour - 1) % participants.length]?.name || "—";
  stats.innerHTML = `<strong>Jour ${jour} :</strong> ${beneficiary} (prévu)<br>
    Total: $${total.toFixed(2)}, 1/3: $${oneThird.toFixed(2)}, 10%: $${tenPercent.toFixed(2)}, Retenu: $${retained.toFixed(2)}, Payé: $${payout.toFixed(2)}`;
}

function exportToExcel() {
  if (!history.length) return alert("Aucun historique");
  const ws = XLSX.utils.json_to_sheet(history);
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, "Historique");
  XLSX.writeFile(wb, "rotation_d2b1_online.xlsx");
}

async function resetData() {
  if (!confirm("Réinitialiser toutes les données ?")) return;
  await fetch(API_URL, {
    method: "POST",
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: "resetAll" })
  });
  participants = []; history = []; currentDay = 1;
  renderParticipants(); renderHistory(); updateStats();
  alert("Données réinitialisées.");
}

// Service Worker PWA
if ('serviceWorker' in navigator) {
  const swBlob = new Blob([`
    const CACHE_NAME = 'd2b1-cache';
    const urlsToCache = ['index.html'];

    self.addEventListener('install', e => {
      e.waitUntil(caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache)));
    });

    self.addEventListener('fetch', e => {
      e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
    });
  `], { type: "application/javascript" });

  const swUrl = URL.createObjectURL(swBlob);
  navigator.serviceWorker.register(swUrl).then(() => console.log("Service worker actif"));
}
</script>
</body>
</html>
