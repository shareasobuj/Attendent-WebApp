# Attendent-WebApp


<html lang="bn">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>কর্মচারী হাজিরা অ্যাপ</title>
  <style>
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:"Segoe UI",Tahoma,Geneva,Verdana,sans-serif;padding:2rem;background:#fafafa;color:#333;line-height:1.5}
    h1{font-size:2rem;margin-bottom:1rem;display:inline-block;padding:.25rem .75rem;background:#ffeb3b;border-radius:.5rem;box-shadow:0 3px 6px rgba(0,0,0,.1)}
    #officeNameInput{font-size:1rem;padding:.3rem .5rem;border:1px solid #ccc;border-radius:.375rem;margin-left:1rem;width:220px}
    label{font-weight:600}
    input[type=text],input[type=date],select{padding:.5rem .75rem;margin-left:.25rem;border:1px solid #ccc;border-radius:.375rem;min-width:200px}
    button{padding:.5rem 1rem;border:none;border-radius:.375rem;background:#ffeb3b;cursor:pointer;font-weight:600;margin-left:.5rem;box-shadow:0 2px 4px rgba(0,0,0,.15);transition:transform .1s ease}
    button:hover{transform:translateY(-2px)}
    .table-wrapper{overflow-x:auto;margin-top:2rem}
    table{width:100%;border-collapse:collapse;min-width:400px}
    th,td{border:1px solid #ddd;padding:.5rem;text-align:center}
    th{background:#fff9c4}
    /* ছুটি দিন (সাপ্তাহিক + সরকারি) */
    th.holiday,td.holiday{background:#ffcdd2!important;color:#b71c1c;font-weight:700}
    tr.present{background:#e8f5e9}
    tr.absent{background:#ffebee}
    .controls{display:grid;grid-template-columns:repeat(auto-fit,minmax(260px,1fr));gap:1rem;align-items:center;margin-bottom:1.5rem}
    .actions{margin-top:1.5rem;display:flex;flex-wrap:wrap;gap:.5rem}
    @media print{body{padding:0}.controls,.actions,#officeNameInput{display:none!important}h1{box-shadow:none}th,td{font-size:12px;padding:4px}}
  </style>
</head>
<body>
  <h1>
    🌼 <span id="officeTitle">অফিসের নাম</span>
    <input type="text" id="officeNameInput" placeholder="অফিসের নাম লিখুন" oninput="updateOfficeName()" />
  </h1>

  <div class="controls">
    <div>
      <label for="date">তারিখ:</label>
      <input type="date" id="date" />
    </div>
    <div>
      <label for="viewSelect">ভিউ:</label>
      <select id="viewSelect">
        <option value="single">একদিন</option>
        <option value="week">সপ্তাহ</option>
        <option value="month">মাস</option>
      </select>
    </div>
    <div>
      <label for="employeeName">নতুন কর্মচারী যোগ করুন:</label>
      <input type="text" id="employeeName" placeholder="নাম লিখুন" />
      <button id="addEmployeeBtn">যোগ করুন</button>
    </div>
  </div>
  <div class="table-wrapper">
    <table id="attendanceTable">
      <thead id="attendanceHead"></thead>
      <tbody id="attendanceBody"></tbody>
    </table>
  </div>
  <div class="actions">
    <button id="saveBtn">সংরক্ষণ করুন</button>
    <button id="exportBtn">CSV ডাউনলোড</button>
    <button id="printBtn">প্রিন্ট</button>
    <button id="clearBtn">সব ডেটা মুছুন</button>
  </div>
  <script defer>
    /* ======== ডেটা ======== */
    let attendance=JSON.parse(localStorage.getItem("attendance"))||{};
    let employees=JSON.parse(localStorage.getItem("employees"))||[];

    /* ======== বাংলাদেশ সরকারি ছুটি 2025 (প্রয়োজনে আপডেট করুন) ======== */
    const GOV_HOLIDAYS=new Set([
      "2025-02-21","2025-03-17","2025-03-26","2025-04-14","2025-04-21","2025-05-01","2025-05-05","2025-06-13","2025-06-14","2025-06-15","2025-07-28","2025-08-15","2025-08-26","2025-10-02","2025-10-07","2025-12-16","2025-12-25"
    ]);

    /* ======== ইউটিল ======== */
    const dateInput=document.getElementById("date");
    const viewSelect=document.getElementById("viewSelect");
    const employeeNameInp=document.getElementById("employeeName");
    const addEmployeeBtn=document.getElementById("addEmployeeBtn");
    const attendanceHead=document.getElementById("attendanceHead");
    const attendanceBody=document.getElementById("attendanceBody");

    function formatDateISO(d){return d.toISOString().slice(0,10)}
    function getWeekDates(base){const b=new Date(base);const mon=new Date(b);mon.setDate(b.getDate()-((b.getDay()+6)%7));const arr=[];for(let i=0;i<7;i++){const d=new Date(mon);d.setDate(mon.getDate()+i);arr.push(formatDateISO(d))}return arr}
    function getMonthDates(base){const d=new Date(base);d.setDate(1);const arr=[];while(d.getMonth()===new Date(base).getMonth()){arr.push(formatDateISO(d));d.setDate(d.getDate()+1)}return arr}
    const isWeekend=dt=>{const day=new Date(dt).getDay();return day===5||day===6};
    const isGovHoliday=dt=>GOV_HOLIDAYS.has(dt);

    /* ======== অফিস নাম ======== */
    function updateOfficeName(){document.getElementById("officeTitle").textContent=document.getElementById("officeNameInput").value||"অফিসের নাম"}

    /* ======== টেবিল রেন্ডার ======== */
    function renderTable(){const view=viewSelect.value;const base=dateInput.value||formatDateISO(new Date());let range=[base];if(view==="week")range=getWeekDates(base);if(view==="month")range=getMonthDates(base);
      attendanceHead.innerHTML="";const hr=document.createElement("tr");const nameTh=document.createElement("th");nameTh.textContent="কর্মচারীর নাম";hr.appendChild(nameTh);
      range.forEach(dt=>{const th=document.createElement("th");th.textContent=dt.split("-")[2];if(isWeekend(dt)||isGovHoliday(dt))th.classList.add("holiday");hr.appendChild(th)});
      const optTh=document.createElement("th");optTh.textContent="অপশন";hr.appendChild(optTh);attendanceHead.appendChild(hr);
      attendanceBody.innerHTML="";
      employees.forEach((name,idx)=>{const tr=document.createElement("tr");const tdName=document.createElement("td");tdName.textContent=name;tr.appendChild(tdName);
        range.forEach(dt=>{const td=document.createElement("td");if(isWeekend(dt)||isGovHoliday(dt))td.classList.add("holiday");const cb=document.createElement("input");cb.type="checkbox";cb.checked=attendance[dt]?.[name]||false;cb.onchange=()=>{setStatus(name,cb.checked,dt);saveAttendance()};td.appendChild(cb);tr.appendChild(td)});
        const tdOpt=document.createElement("td");const delBtn=document.createElement("button");delBtn.textContent="ডিলিট";delBtn.onclick=()=>{if(confirm("মুছে ফেলতে চান?")){employees.splice(idx,1);saveEmployees();renderTable()}};tdOpt.appendChild(delBtn);tr.appendChild(tdOpt);
        attendanceBody.appendChild(tr)});
    }

    function setStatus(name,st,dt){if(!attendance[dt])attendance[dt]={};attendance[dt][name]=st}
    const saveAttendance=()=>localStorage.setItem("attendance",JSON.stringify(attendance));
    const saveEmployees=()=>localStorage.setItem("employees",JSON.stringify(employees));

    /* ======== ইভেন্ট ======== */
    addEmployeeBtn.onclick=()=>{const n=employeeNameInp.value.trim();if(!n)return;if(employees.includes(n))return alert("এই নাম ইতিমধ্যে আছে");employees.push(n);saveEmployees();employeeNameInp.value="";renderTable()};
    document.getElementById("saveBtn").onclick=()=>{saveAttendance();alert("সংরক্ষণ হয়েছে")};
    document.getElementById("exportBtn").onclick=()=>{let csv="Date,"+employees.join(",")+"\n";const keys=Object.keys(attendance).sort();keys.forEach(d=>{const row=[d];employees.forEach(e=>row.push(attendance[d]?.[e]?"Present":""));csv+=row.join(",")+"\n"});const blob=new Blob([csv],{type:"text/csv"});const a=document.createElement("a");a.href=URL.createObjectURL(blob);a.download="attendance.csv";a.click();URL.revokeObjectURL(a.href)};
    document.getElementById("printBtn").onclick=()=>window.print();
    document.getElementById("clearBtn").onclick=()=>{if(confirm("সব মুছবেন?")){localStorage.clear();location.reload()}};
    dateInput.onchange=renderTable;viewSelect.onchange=renderTable;

    /* ======== Init ======== */
    dateInput.valueAsDate=new Date();renderTable();
  </script>
</body>
</html>
