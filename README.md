<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Scorecard Dashboard</title>
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<style>
  select[multiple] {
    min-width: 150px;
    max-width: 250px;
    height: 100px;
    border: 1px solid #03befc;
    border-radius: 0.5rem;
    padding: 0.25rem;
    background-color: white;
  }
  textarea {
    font-family: inherit;
  }
</style>
</head>
<body class="bg-gradient-to-r from-gray-100 to-gray-200 min-h-screen p-6 font-sans">

<!-- Top filter line -->
<div class="flex space-x-4 items-start mb-4" id="filterLine">
  <input type="file" id="fileInput" class="rounded-lg shadow-md p-2 border border-blue-500" accept=".xlsx,.xls,.csv"/>
  <select id="userSelect" class="rounded-lg shadow-md p-2 border border-blue-500 h-10">
    <option value="All">All Users</option>
  </select>
  <select id="monthSelect" multiple></select>
  <button id="downloadBtn" class="ml-auto bg-blue-500 text-white px-4 py-2 rounded-lg shadow-md hover:bg-blue-600">Download Dashboard</button>
</div>

<!-- Main Dashboard container -->
<div id="dashboard" class="max-w-7xl mx-auto space-y-6">

  <!-- Header -->
  <div class="text-center">
    <h1 class="text-3xl font-bold text-gray-800">Scorecard</h1>
    <h2 id="userMonthHeader" class="text-lg font-bold text-gray-700 mt-1"></h2>
  </div>

  <!-- Top Info: Designation, LOB, TL -->
  <div id="topInfo" class="grid grid-cols-3 gap-4"></div>

  <!-- Metrics Cards -->
  <div id="metricsContainer" class="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-6 gap-4"></div>

  <!-- Charts Section -->
  <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
    <div class="rounded-2xl border border-blue-500 p-4 bg-white">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">Attendance</h3>
      <canvas id="attendanceChart" height="120"></canvas>
    </div>
    <div class="rounded-2xl border border-blue-500 p-4 bg-white">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">Quality</h3>
      <canvas id="qualityChart" height="120"></canvas>
    </div>
    <div class="rounded-2xl border border-blue-500 p-4 bg-white">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">PKT</h3>
      <canvas id="pktChart" height="120"></canvas>
    </div>
  </div>

  <!-- Feedback / Area of Improvements / Additional Activities as editable textareas -->
  <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
    <div class="rounded-2xl border border-blue-500 p-4 bg-white h-64">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">Feedback</h3>
      <textarea id="feedbackContent" class="w-full h-full p-2 border border-gray-300 rounded resize-none"></textarea>
    </div>
    <div class="rounded-2xl border border-blue-500 p-4 bg-white h-64">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">Area of Improvements</h3>
      <textarea id="areaImprovementContent" class="w-full h-full p-2 border border-gray-300 rounded resize-none"></textarea>
    </div>
    <div class="rounded-2xl border border-blue-500 p-4 bg-white h-64">
      <h3 class="text-lg font-semibold mb-2 text-gray-800">Additional Activities</h3>
      <textarea id="additionalActivityContent" class="w-full h-full p-2 border border-gray-300 rounded resize-none"></textarea>
    </div>
  </div>

</div>

<script>
let rawData = [];
let attendanceChart, qualityChart, pktChart;

document.getElementById('fileInput').addEventListener('change', handleFile);
document.getElementById('downloadBtn').addEventListener('click', downloadDashboard);

function handleFile(e){
  const file = e.target.files[0];
  const reader = new FileReader();
  reader.onload = function(evt){
    const data = evt.target.result;
    const workbook = XLSX.read(data,{type:'binary'});
    const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
    rawData = XLSX.utils.sheet_to_json(firstSheet,{defval:''});
    populateFilters();
    updateDashboard();
  };
  reader.readAsBinaryString(file);
}

function populateFilters(){
  const userSelect = document.getElementById('userSelect');
  const monthSelect = document.getElementById('monthSelect');
  userSelect.innerHTML='<option value="All">All Users</option>';
  monthSelect.innerHTML='';

  const users = [...new Set(rawData.map(r=>r.Name))];
  users.forEach(u=>{
    const opt=document.createElement('option'); opt.value=u; opt.textContent=u;
    userSelect.appendChild(opt);
  });

  const months = [...new Set(rawData.map(r=>r.Month))];
  months.forEach(m=>{
    const opt=document.createElement('option'); opt.value=m; opt.textContent=m;
    monthSelect.appendChild(opt);
  });

  userSelect.addEventListener('change', updateDashboard);
  monthSelect.addEventListener('change', updateDashboard);
}

function getSelectedMonths(){
  return Array.from(document.getElementById('monthSelect').selectedOptions).map(o=>o.value);
}

function updateDashboard(){
  const user = document.getElementById('userSelect').value;
  const months = getSelectedMonths();
  let filtered = rawData;
  if(user!=='All') filtered = filtered.filter(r=>r.Name===user);
  if(months.length>0) filtered = filtered.filter(r=>months.includes(r.Month));

  updateHeader(filtered, months, user);
  updateTopInfo(filtered);
  updateMetrics(filtered);
  updateCharts(filtered);
  updateFeedbackArea(filtered);
}

function updateHeader(data, months, user){
  const header = document.getElementById('userMonthHeader');
  if(data.length>0){
    const name = (user!=='All')? user : 'Multiple Users';
    const monthText = months.length>0? months.join(', ') : 'All Months';
    header.textContent=`${name} - ${monthText}`;
  } else header.textContent='';
}

function updateTopInfo(data){
  const topInfo = document.getElementById('topInfo');
  topInfo.innerHTML='';
  if(data.length===0) return;
  const info = ['Designation','LOB','TL'];
  info.forEach(field=>{
    const card = document.createElement('div');
    card.className="rounded-2xl border border-blue-500 p-4 text-center bg-white";
    card.innerHTML=`<div class="text-gray-700 text-sm font-semibold">${field}</div><div class="text-xl font-bold mt-1">${data[0][field]}</div>`;
    topInfo.appendChild(card);
  });
}

function updateMetrics(data){
  const container=document.getElementById('metricsContainer');
  container.innerHTML='';
  if(data.length===0) return;
  const metrics=['Production','Quality','Attendance','Escalations','Compliance','PKT'];
  metrics.forEach(m=>{
    let val;
    if(['Attendance','Quality','PKT'].includes(m)){
      val = (data.reduce((sum,d)=>sum+(Number(d[m]||0)*100),0)/data.length).toFixed(1)+'%';
    } else val = data.reduce((sum,d)=>sum+(Number(d[m]||0)),0);
    const card=document.createElement('div');
    card.className="rounded-2xl p-4 flex flex-col justify-between border border-blue-500 bg-white";
    card.innerHTML=`<div class="text-gray-700 text-sm font-semibold">${m}</div><div class="text-2xl font-bold mt-2">${val}</div>`;
    container.appendChild(card);
  });
}

function updateCharts(data){
  const avg = (field) => data.length>0 ? (data.reduce((s,d)=>s+(Number(d[field]||0)*100),0)/data.length).toFixed(1) : 0;
  const occupied = '#03befc';
  const notOccupied = '#b3e5fc';

  // Pie Chart - Attendance
  const ctxAtt=document.getElementById('attendanceChart').getContext('2d');
  if(attendanceChart) attendanceChart.destroy();
  attendanceChart=new Chart(ctxAtt,{
    type:'pie',
    data:{
      labels:['Occupied','Remaining'],
      datasets:[{
        data:[avg('Attendance'),100-avg('Attendance')],
        backgroundColor:[occupied,notOccupied]
      }]
    },
    plugins: [ChartDataLabels],
    options:{
      plugins:{
        legend:{position:'bottom'},
        datalabels:{color:'#000', font:{weight:'bold'}, formatter:v=>v+'%'},
        tooltip:{callbacks:{label: ctx => ctx.label + ': ' + ctx.raw.toFixed(1) + '%'}}
      }
    }
  });

  // Donut Chart - Quality
  const ctxQual=document.getElementById('qualityChart').getContext('2d');
  if(qualityChart) qualityChart.destroy();
  qualityChart=new Chart(ctxQual,{
    type:'doughnut',
    data:{
      labels:['Occupied','Remaining'],
      datasets:[{
        data:[avg('Quality'),100-avg('Quality')],
        backgroundColor:[occupied,notOccupied]
      }]
    },
    plugins: [ChartDataLabels],
    options:{
      cutout:'60%',
      plugins:{
        legend:{position:'bottom'},
        datalabels:{color:'#000', font:{weight:'bold'}, formatter:v=>v+'%'},
        tooltip:{callbacks:{label: ctx => ctx.label + ': ' + ctx.raw.toFixed(1) + '%'}}
      }
    }
  });

  // Line Chart - PKT
  const ctxPkt=document.getElementById('pktChart').getContext('2d');
  if(pktChart) pktChart.destroy();
  const labels = data.map(d=>d.Month+'-'+d.Name);
  const values = data.map(d=>Number(d.PKT||0)*100);
  pktChart=new Chart(ctxPkt,{
    type:'line',
    data:{
      labels:labels,
      datasets:[{
        label:'PKT %',
        data:values,
        borderColor:'#03befc',
        backgroundColor:'rgba(3,190,252,0.2)',
        tension:0.3,
        fill:true
      }]
    },
    plugins: [ChartDataLabels],
    options:{
      plugins:{
        legend:{position:'bottom'},
        datalabels:{color:'#000', font:{weight:'bold'}, formatter:v=>v+'%'},
        tooltip:{callbacks:{label: ctx => ctx.dataset.label + ': ' + ctx.raw.toFixed(1) + '%'}}
      },
      scales:{y:{beginAtZero:true,max:100}}
    }
  });
}

function updateFeedbackArea(data){
  const feedback = document.getElementById('feedbackContent');
  const area = document.getElementById('areaImprovementContent');
  const addit = document.getElementById('additionalActivityContent');

  if(data.length===0){
    feedback.value = area.value = addit.value = '';
    return;
  }

  feedback.value = data.map(d => d.Feedback || 'N/A').join('\n');
  area.value = data.map(d => d['Area of Improvements'] || 'N/A').join('\n');
  addit.value = data.map(d => d['Additional Activity'] || 'N/A').join('\n');
}

function downloadDashboard(){
  const dashboard=document.getElementById('dashboard');
  html2canvas(dashboard,{useCORS:true,allowTaint:true,scale:2}).then(canvas=>{
    const link=document.createElement('a');
    link.download='scorecard_dashboard.png';
    link.href=canvas.toDataURL();
    link.click();
  });
}
</script>

</body>
</html>
