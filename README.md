<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Bolão Mega da Virada</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

<style>
.sena-bg { background:#d1fae5; }
.quina-bg { background:#fef3c7; }
.quadra-bg { background:#e0e7ff; }
.hit-number { background:#16a34a; color:white; }
</style>
</head>

<body class="bg-gray-100 p-4 max-w-4xl mx-auto">

<h1 class="text-2xl font-bold mb-4 text-center">🎰 Bolão Mega da Virada</h1>

<!-- ADMIN -->
<div class="bg-white p-4 rounded shadow mb-4">
  <button onclick="loginAdmin()" class="bg-blue-600 text-white px-4 py-2 rounded">
    Área Administrativa
  </button>

  <div id="adminPanel" class="hidden mt-4 space-y-3">
    <input type="file" id="fileInput" accept=".xlsx" class="block">
    <button onclick="importExcel()" class="bg-green-600 text-white px-4 py-2 rounded">
      Importar Planilha
    </button>

    <input id="drawInput" placeholder="Resultado (ex: 01 02 03 04 05 06)"
      class="border p-2 w-full">
    <button onclick="setDraw()" class="bg-purple-600 text-white px-4 py-2 rounded">
      Salvar Resultado
    </button>
  </div>
</div>

<!-- BUSCA -->
<input id="searchInput" oninput="render()" placeholder="Pesquisar nome ou dezenas"
class="border p-2 w-full mb-4">

<!-- RESUMO -->
<div class="grid grid-cols-3 gap-2 mb-4 text-center">
  <div class="bg-green-100 p-2 rounded">Sena<br><b id="countSena">0</b></div>
  <div class="bg-yellow-100 p-2 rounded">Quina<br><b id="countQuina">0</b></div>
  <div class="bg-indigo-100 p-2 rounded">Quadra<br><b id="countQuadra">0</b></div>
</div>

<!-- LISTA -->
<div id="betsList" class="space-y-4"></div>

<!-- FIREBASE -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getFirestore, collection, getDocs, setDoc, doc, deleteDoc } 
from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyA-S6ncia7aEB3BgtzjFo8mq-C9ATTGYo8",
  authDomain: "bolao-8bd76.firebaseapp.com",
  projectId: "bolao-8bd76",
  storageBucket: "bolao-8bd76.firebasestorage.app",
  messagingSenderId: "209250392992",
  appId: "1:209250392992:web:fd5850c1ffbe8e37a222c5",
  measurementId: "G-73E64DCVRY"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

window.allParticipants = [];
window.currentDraw = [];

window.loginAdmin = () => {
  const pass = prompt("Senha admin:");
  if (pass === "bolao2025") {
    document.getElementById("adminPanel").classList.remove("hidden");
  }
};

window.importExcel = async () => {
  const file = document.getElementById("fileInput").files[0];
  if (!file) return alert("Selecione a planilha");

  const data = await file.arrayBuffer();
  const wb = XLSX.read(data);
  const ws = wb.Sheets[wb.SheetNames[0]];
  const rows = XLSX.utils.sheet_to_json(ws, { header: 1 });

  await Promise.all((await getDocs(collection(db, "participants")))
    .docs.map(d => deleteDoc(d.ref)));

  for (let i = 1; i < rows.length; i++) {
    const [name, b1, b2] = rows[i];
    if (!name) continue;

    const bets = [b1, b2].map(b =>
      b.toString().match(/\d+/g).map(n => parseInt(n))
    );

    await setDoc(doc(db, "participants", name), { name, bets });
  }

  alert("Participantes importados");
  loadData();
};

window.setDraw = async () => {
  currentDraw = document.getElementById("drawInput")
    .value.match(/\d+/g)?.map(n => parseInt(n)) || [];
  render();
};

async function loadData() {
  const snap = await getDocs(collection(db, "participants"));
  allParticipants = snap.docs.map(d => d.data());
  render();
}

window.render = () => {
  const list = document.getElementById("betsList");
  list.innerHTML = "";

  let sena = 0, quina = 0, quadra = 0;
  const search = document.getElementById("searchInput").value.toLowerCase();
  const nums = search.match(/\d+/g)?.map(n => parseInt(n)) || [];

  allParticipants.forEach(p => {
    p.bets.forEach(b => {
      const h = b.filter(n => currentDraw.includes(n)).length;
      if (h === 6) sena++;
      else if (h === 5) quina++;
      else if (h === 4) quadra++;
    });
  });

  countSena.innerText = sena;
  countQuina.innerText = quina;
  countQuadra.innerText = quadra;

  allParticipants.forEach(p => {
    if (
      search &&
      !p.name.toLowerCase().includes(search) &&
      !p.bets.some(b => nums.every(n => b.includes(n)))
    ) return;

    const card = document.createElement("div");
    card.className = "bg-white p-4 rounded shadow";

    let html = `<b>${p.name}</b>`;
    p.bets.forEach((b, i) => {
      const h = b.filter(n => currentDraw.includes(n)).length;
      const bg = h === 6 ? "sena-bg" : h === 5 ? "quina-bg" : h === 4 ? "quadra-bg" : "bg-gray-50";

      html += `
      <div class="p-2 mt-2 rounded ${bg}">
        Jogo ${i + 1} • ${currentDraw.length ? h + " acertos" : "aguardando sorteio"}
        <div class="flex gap-1 mt-1">
          ${b.map(n => `<span class="w-7 h-7 text-xs rounded-full border flex items-center justify-center
          ${currentDraw.includes(n) ? "hit-number" : ""}">${String(n).padStart(2,"0")}</span>`).join("")}
        </div>
      </div>`;
    });

    card.innerHTML = html;
    list.appendChild(card);
  });
};

loadData();
</script>
</body>
</html>
