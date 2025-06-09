<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Smart Sockets Dashboard</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <link href="https://fonts.googleapis.com/css?family=Roboto:400,500&display=swap" rel="stylesheet">
  <style>
    body {
      font-family: 'Roboto', Arial, sans-serif;
      background: linear-gradient(120deg, #e0eafc 0%, #cfdef3 100%);
      margin: 0; padding: 0;
      min-height: 100vh;
    }
    .container {
      max-width: 560px;
      margin: 0 auto;
      padding: 1.5em;
    }
    h2 {
      color: #1976d2;
      text-align: center;
      font-weight: 500;
      letter-spacing: 1px;
      margin-bottom: 1em;
    }
    .socket-card {
      background: #fff;
      border-radius: 18px;
      box-shadow: 0 2px 10px #b2bec3;
      padding: 1.25em;
      margin-bottom: 1.5em;
      border-left: 8px solid #1976d2;
      transition: border-color 0.3s;
    }
    .socket-card.master { border-color: #43a047; }
    .socket-title {
      font-size: 1.2em;
      font-weight: 500;
      color: #1976d2;
      margin-bottom: 0.7em;
      letter-spacing: 1px;
    }
    .socket-card.master .socket-title {
      color: #43a047;
    }
    .status-row {
      display: flex;
      justify-content: space-between;
      margin: 0.3em 0;
      font-size: 1em;
    }
    .status-row span {
      color: #222;
    }
    .relay-btn {
      display: block;
      width: 100%;
      margin: 1em 0 0.5em 0;
      padding: 0.9em;
      border-radius: 8px;
      font-size: 1em;
      font-weight: 500;
      border: none;
      cursor: pointer;
      transition: background 0.2s, color 0.2s;
    }
    .relay-btn.on {
      background: linear-gradient(90deg, #43a047, #66bb6a);
      color: #fff;
      box-shadow: 0 2px 8px #43a04777;
    }
    .relay-btn.off {
      background: linear-gradient(90deg, #c62828, #e57373);
      color: #fff;
      box-shadow: 0 2px 8px #c6282877;
    }
    .cutoff {
      color: #c62828;
      font-weight: bold;
      text-align: right;
    }
    .slider-row, .input-row, .presence-row, .schedule-row, .reset-row {
      display: flex;
      align-items: center;
      margin: 0.4em 0;
    }
    .slider-row input[type="number"], .input-row input[type="number"], .schedule-row input[type="time"] {
      width: 100px;
      margin-left: 0.6em;
      padding: 0.3em 0.5em;
      border: 1px solid #b2bec3;
      border-radius: 5px;
      font-size: 1em;
      background: #f8fafd;
    }
    .slider-row button, .input-row button, .schedule-row button, .reset-row button {
      margin-left: 0.7em;
      padding: 0.5em 1em;
      border-radius: 6px;
      border: none;
      background: #1976d2;
      color: #fff;
      font-weight: 500;
      cursor: pointer;
      transition: background 0.2s;
    }
    .slider-row button:hover, .input-row button:hover, .schedule-row button:hover, .reset-row button:hover {
      background: #1254a2;
    }
    .presence-row label {
      margin-left: 0.7em;
      font-weight: 500;
      color: #1976d2;
      cursor: pointer;
    }
    .presence-row input[type="checkbox"] {
      transform: scale(1.3);
      accent-color: #43a047;
    }
    @media (max-width: 600px) {
      .container { padding: 0.2em; }
      .socket-card { padding: 0.7em; }
      .slider-row input[type="number"], .input-row input[type="number"], .schedule-row input[type="time"] {
        width: 80px;
      }
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Smart Sockets Dashboard</h2>
    <div id="dashboard"></div>
  </div>
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/9.19.1/firebase-app.js";
    import { getDatabase, ref, onValue, set, get, update } from "https://www.gstatic.com/firebasejs/9.19.1/firebase-database.js";

    // --- EDIT THIS: List your outlets here ---
    const socketIds = ['master', 'slave_1']; // add more like 'slave_3', etc

    // --- REPLACE WITH YOUR FIREBASE CONFIG ---
    const firebaseConfig = {
      apiKey: "AIzaSyBzH4if3zg3eKt9nku2aughJqV6FVSbAjo",
      authDomain: "smart-e343e.firebaseapp.com",
      databaseURL: "https://smart-e343e-default-rtdb.europe-west1.firebasedatabase.app",
      projectId: "smart-e343e",
      appId: "1:1234567890:web:abcdefg"
    };
    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    // --- Dashboard rendering ---
    function createSocketCard(id) {
      const isMaster = id === "master";
      const card = document.createElement("div");
      card.className = "socket-card" + (isMaster ? " master" : "");
      card.innerHTML = `
        <div class="socket-title">${isMaster ? "MASTER" : id.toUpperCase()}</div>
        <div class="status-row"><span>Voltage:</span><span id="v-${id}">-</span></div>
        <div class="status-row"><span>Current:</span><span id="c-${id}">-</span></div>
        <div class="status-row"><span>Power:</span><span id="p-${id}">-</span></div>
        <div class="status-row"><span>kWh Used:</span><span id="kwh-${id}">-</span></div>
        <div class="status-row"><span>Relay:</span><span id="relay-${id}">-</span></div>
        <div class="cutoff" id="cutoff-${id}"></div>
        <button class="relay-btn" id="relay-btn-${id}">Toggle</button>
        <div class="slider-row">
          <span>kWh Limit:</span>
          <input type="number" min="0" step="0.01" id="limit-${id}">
          <button id="limit-btn-${id}">Set</button>
        </div>
        <div class="reset-row">
          <button id="resetkwh-btn-${id}" style="background:#c62828;">Reset kWh</button>
        </div>
        <div class="presence-row">
          <input type="checkbox" id="presence-${id}">
          <label for="presence-${id}">Presence Sensing</label>
        </div>
        <div class="schedule-row">
          <span>ON:</span>
          <input type="time" id="onTime-${id}">
          <span>OFF:</span>
          <input type="time" id="offTime-${id}">
          <button id="schedule-btn-${id}">Set</button>
        </div>
      `;
      return card;
    }

    function updateCard(id) {
      // Live status
      const statusBase = ref(db, "/status/" + id);
      onValue(ref(db, `/status/${id}/voltage`), s => document.getElementById(`v-${id}`).textContent = s.val() ? s.val().toFixed(2) + " V" : "-");
      onValue(ref(db, `/status/${id}/current`), s => document.getElementById(`c-${id}`).textContent = s.val() ? s.val().toFixed(2) + " A" : "-");
      onValue(ref(db, `/status/${id}/power`), s => document.getElementById(`p-${id}`).textContent = s.val() ? s.val().toFixed(2) + " W" : "-");
      onValue(ref(db, `/kwh/${id}`), s => document.getElementById(`kwh-${id}`).textContent = s.val() ? (+s.val()).toFixed(3) + " kWh" : "-");
      onValue(ref(db, `/relayState/${id}`), s => {
        const v = !!s.val();
        document.getElementById(`relay-${id}`).textContent = v ? "ON" : "OFF";
        const btn = document.getElementById(`relay-btn-${id}`);
        btn.className = "relay-btn " + (v ? "on" : "off");
        btn.textContent = v ? "Turn OFF" : "Turn ON";
      });
      onValue(ref(db, `/cutoff/${id}`), s => {
        document.getElementById(`cutoff-${id}`).textContent = s.val() ? "RELAY CUT OFF (kWh limit reached)" : "";
      });

      // Relay toggle
      document.getElementById(`relay-btn-${id}`).onclick = async function() {
        const relayRef = ref(db, `/relayState/${id}`);
        const relayCmdRef = ref(db, `/remoteRelayCommand/${id}`);
        const v = await get(relayRef).then(snap => !!snap.val());
        set(relayCmdRef, !v);
      };

      // kWh limit
      get(ref(db, `/threshold_kwh/${id}`)).then(snap => {
        if (snap.exists()) document.getElementById(`limit-${id}`).value = snap.val();
      });
      document.getElementById(`limit-btn-${id}`).onclick = function() {
        const val = parseFloat(document.getElementById(`limit-${id}`).value);
        if (!isNaN(val)) set(ref(db, `/threshold_kwh/${id}`), val);
      };

      // Reset kWh
      document.getElementById(`resetkwh-btn-${id}`).onclick = function() {
        set(ref(db, `/reset_kwh/${id}`), true);
      };

      // Presence sensing
      get(ref(db, `/presence/${id}`)).then(snap => {
        document.getElementById(`presence-${id}`).checked = snap.exists() ? !!snap.val() : true;
      });
      document.getElementById(`presence-${id}`).onchange = function() {
        set(ref(db, `/presence/${id}`), this.checked);
      };

      // Schedule
      get(ref(db, `/schedule/${id}/onTime`)).then(snap => {
        if (snap.exists()) document.getElementById(`onTime-${id}`).value = snap.val();
      });
      get(ref(db, `/schedule/${id}/offTime`)).then(snap => {
        if (snap.exists()) document.getElementById(`offTime-${id}`).value = snap.val();
      });
      document.getElementById(`schedule-btn-${id}`).onclick = function() {
        const onTime = document.getElementById(`onTime-${id}`).value;
        const offTime = document.getElementById(`offTime-${id}`).value;
        update(ref(db, `/schedule/${id}`), { onTime, offTime });
      };
    }

    // --- Render ---
    const dash = document.getElementById("dashboard");
    socketIds.forEach(id => {
      const card = createSocketCard(id);
      dash.appendChild(card);
      updateCard(id);
    });
  </script>
</body>
</html>
