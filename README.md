<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Energy Efficient Home Automation Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      margin: 0;
      background: #f0f4f8;
      color: #333;
      max-width: 900px;
      margin-left: auto;
      margin-right: auto;
      padding: 30px;
    }
    header {
      background-color: #00796b;
      color: white;
      padding: 15px 30px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-radius: 6px;
    }
    header h1 { margin: 0; font-weight: normal; }
    button.logout-btn {
      background: #004d40;
      border: none;
      color: white;
      padding: 8px 14px;
      cursor: pointer;
      border-radius: 4px;
      font-size: 14px;
    }
    section {
      background: white;
      padding: 20px 30px;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
      margin-bottom: 30px;
    }
    section h2 { margin-top: 0; color: #00796b; }
    .device-control {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 15px;
    }
    .device-control label { font-weight: 600; flex: 1; }
    .device-control input[type="checkbox"],
    .device-control input[type="number"] {
      width: 60px; height: 24px; cursor: pointer;
    }
    .energy-stats { font-size: 1.2em; margin-bottom: 5px; }
    #weeklyChart { max-width: 100%; margin-top: 20px; }
    .details {
      background: #e8f5e9;
      padding: 15px;
      border-radius: 6px;
      margin-top: 15px;
      display: none;
    }
    .details h3 { margin-top: 0; color: #004d40; }
    .device { display: flex; justify-content: space-between; margin: 6px 0; }
  </style>
</head>
<body>

<header>
  <h1>Welcome, <span id="user-name">User</span></h1>
  <button class="logout-btn" id="logoutBtn">Logout</button>
</header>

<section>
  <h2>Device Controls</h2>
  <div id="deviceControls"></div>
</section>

<section>
  <h2>Current Energy Consumption</h2>
  <p class="energy-stats">Total Power Usage: <strong id="powerUsage">0</strong> Watts</p>
  <p class="energy-stats">Estimated Monthly Consumption: <strong id="monthlyUsage">0</strong> kWh</p>
  <p class="energy-stats">Estimated Monthly Cost: <strong id="monthlyCost">₹0.00</strong></p>
</section>

<section>
  <h2>Weekly Energy Consumption</h2>
  <canvas id="weeklyChart" height="100"></canvas>
  <div class="details" id="dayDetails">
    <h3 id="dayTitle">Devices Used on [Day]</h3>
    <div id="deviceUsageList"></div>
  </div>
</section>

<script>
  const userName = localStorage.getItem('username') || 'User';
  document.getElementById('user-name').textContent = userName;

  document.getElementById('logoutBtn').addEventListener('click', () => {
    localStorage.removeItem('username');
    window.location.href = 'index.html';
  });

  const electricityRate = 8; // Rs per kWh

  const devices = [
    { id: 'fan1', name: "Fan 1", power: 75 },
    { id: 'fan2', name: "Fan 2", power: 75 },
    { id: 'light1', name: "Light 1", power: 60 },
    { id: 'light2', name: "Light 2", power: 60 },
    { id: 'ac1', name: "AC 1", power: 1200 },
    { id: 'heater1', name: "Heater 1", power: 1500 }
  ];

  const deviceControlsDiv = document.getElementById('deviceControls');
  const deviceState = {};

  devices.forEach(device => {
    deviceState[device.id] = { on: false, hours: 0 };

    const div = document.createElement('div');
    div.className = 'device-control';

    const label = document.createElement('label');
    label.htmlFor = device.id;
    label.textContent = device.name;

    const checkbox = document.createElement('input');
    checkbox.type = 'checkbox';
    checkbox.id = device.id;

    const hoursInput = document.createElement('input');
    hoursInput.type = 'number';
    hoursInput.min = 0;
    hoursInput.max = 24;
    hoursInput.value = 1;
    hoursInput.title = "Daily usage in hours";
    hoursInput.style.marginLeft = '10px';
    hoursInput.disabled = true;

    div.appendChild(label);
    div.appendChild(checkbox);
    div.appendChild(hoursInput);
    deviceControlsDiv.appendChild(div);

    checkbox.addEventListener('change', e => {
      deviceState[device.id].on = e.target.checked;
      hoursInput.disabled = !e.target.checked;
      deviceState[device.id].hours = e.target.checked ? 1 : 0;
      hoursInput.value = e.target.checked ? 1 : 0;
      updateEnergyStats();
    });

    hoursInput.addEventListener('input', e => {
      const val = Number(e.target.value);
      e.target.value = Math.max(0, Math.min(24, val));
      deviceState[device.id].hours = Number(e.target.value);
      updateEnergyStats();
    });
  });

  function updateEnergyStats() {
    let totalPowerWatts = 0;
    let totalDailyEnergyKWh = 0;

    devices.forEach(device => {
      const state = deviceState[device.id];
      if (state.on && state.hours > 0) {
        totalPowerWatts += device.power;
        totalDailyEnergyKWh += (device.power * state.hours) / 1000;
      }
    });

    const monthlyEnergy = totalDailyEnergyKWh * 30;
    const monthlyCost = monthlyEnergy * electricityRate;

    document.getElementById('powerUsage').textContent = totalPowerWatts.toFixed(0);
    document.getElementById('monthlyUsage').textContent = monthlyEnergy.toFixed(2);
    document.getElementById('monthlyCost').textContent = `₹${monthlyCost.toFixed(2).toLocaleString()}`;

    generateWeeklyData(totalDailyEnergyKWh);
  }

  const days = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];
  const weeklyDeviceUsage = [];

  function generateWeeklyData(dailyTotalEnergy) {
    for (let i = 0; i < 7; i++) {
      const dayData = [];
      devices.forEach(device => {
        let hours = 0;
        const state = deviceState[device.id];
        if (state.on) hours = Math.min(24, Math.max(0, state.hours * (0.5 + Math.random())));
        const energyKWh = +(device.power * hours / 1000).toFixed(2);
        dayData.push({ name: device.name, power: device.power, hours: +hours.toFixed(1), energy: energyKWh });
      });
      weeklyDeviceUsage[i] = dayData;
    }

    const dailyTotals = weeklyDeviceUsage.map(day =>
      +(day.reduce((sum, d) => sum + d.energy, 0)).toFixed(2)
    );
    updateChart(dailyTotals);
  }

  const ctx = document.getElementById("weeklyChart").getContext("2d");
  let weeklyChart = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: days,
      datasets: [{
        label: "Total Energy (kWh)",
        data: [],
        backgroundColor: '#66bb6a',
        borderRadius: 6
      }]
    },
    options: {
      onClick: (event, elements) => {
        if (elements.length > 0) showDayDetails(elements[0].index);
      },
      plugins: {
        tooltip: {
          callbacks: {
            label: ctx => `${ctx.dataset.data[ctx.dataIndex]} kWh`
          }
        }
      },
      scales: {
        y: {
          beginAtZero: true,
          title: { display: true, text: "Energy (kWh)" }
        }
      }
    }
  });

  function updateChart(dailyTotals) {
    weeklyChart.data.datasets[0].data = dailyTotals;
    weeklyChart.update();
  }

  function showDayDetails(dayIndex) {
    const dayName = days[dayIndex];
    const deviceList = weeklyDeviceUsage[dayIndex];
    const output = deviceList
      .filter(d => d.energy > 0)
      .map(d => `
        <div class="device">
          <div>${d.name}</div>
          <div>${d.hours} hrs × ${d.power}W = <strong>${d.energy} kWh</strong></div>
        </div>
      `).join("");

    document.getElementById("dayTitle").textContent = `Devices Used on ${dayName}`;
    document.getElementById("deviceUsageList").innerHTML = output || "<p>No devices used on this day.</p>";
    document.getElementById("dayDetails").style.display = "block";
  }

  updateEnergyStats();
</script>

</body>
</html>
