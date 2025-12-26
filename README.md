<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bolão Mega da Virada 2025</title>

    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">

    <style>
        .sena-bg { background-color: #dcfce7; border: 2px solid #16a34a; }
        .quina-bg { background-color: #fef9c3; border: 2px solid #ca8a04; }
        .quadra-bg { background-color: #dbeafe; border: 2px solid #2563eb; }
        .hit-number { background-color: #16a34a; color: white; font-weight: bold; border-color: #16a34a; }
    </style>
</head>

<body class="bg-gray-100 min-h-screen pb-20">

<!-- NAV -->
<nav class="bg-green-700 text-white p-4 shadow-lg sticky top-0 z-50">
    <div class="container mx-auto flex flex-col md:flex-row justify-between items-center gap-4">
        <h1 class="text-2xl font-bold flex items-center gap-2">
            <i class="fas fa-clover"></i> Bolão Mega da Virada
        </h1>

        <input type="text" id="searchInput"
               placeholder="Pesquisar por nome ou dezenas (ex: 05 12 33)"
               class="w-full md:w-1/2 p-2 rounded text-gray-800 focus:outline-none">

        <button onclick="toggleAdmin()"
                class="bg-green-800 px-3 py-1 rounded text-xs border border-green-600">
            Admin
        </button>
    </div>
</nav>

<!-- RESUMO -->
<div class="container mx-auto mt-6 px-4">
    <div class="bg-white p-4 rounded-lg shadow-md grid grid-cols-3 gap-2 text-center font-bold">
        <div class="bg-green-50 p-2 rounded border border-green-500 text-green-700">
            <div class="text-[10px] uppercase">Sena</div>
            <span id="countSena">0</span>
        </div>
        <div class="bg-yellow-50 p-2 rounded border border-yellow-500 text-yellow-700">
            <div class="text-[10px] uppercase">Quina</div>
            <span id="countQuina">0</span>
        </div>
        <div class="bg-blue-50 p-2 rounded border border-blue-500 text-blue-700">
            <div class="text-[10px] uppercase">Quadra</div>
            <span id="countQuadra">0</span>
        </div>
    </div>
</div>

<!-- ADMIN -->
<div id="adminPanel" class="hidden container mx-auto mt-6 px-4">
    <div class="bg-gray-200 p-4 rounded-lg border border-gray-300 shadow-inner">

        <div class="mb-4 bg-white p-3 rounded shadow-sm">
            <label class="block font-bold mb-1 text-sm text-gray-700">Resultado do Sorteio:</label>
            <div class="flex gap-2">
                <input type="text" id="drawInput"
                       placeholder="Ex: 01 02 03 04 05 06"
                       class="flex-1 p-2 rounded border border-gray-300">
                <button onclick="saveDrawResult()"
                        class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
                    Salvar
                </button>
            </div>
        </div>

        <div class="bg-white p-3 rounded shadow-sm">
            <label class="block font-bold mb-1 text-sm text-gray-700">Importar Excel:</label>
            <input type="file" id="fileInput" accept=".xlsx"
                   class="block w-full text-sm text-gray-500
                   file:mr-4 file:py-2 file:px-4
                   file:rounded-full file:border-0
                   file:bg-green-50 file:text-green-700
                   hover:file:bg-green-100"/>
        </div>
    </div>
</div>

<!-- LISTA -->
<div class="container mx-auto mt-6 px-4">
    <div id="loading" class="text-center py-10 text-gray-500">
        <i class="fas fa-spinner fa-spin fa-2x"></i>
        <p class="mt-2">Carregando bolão...</p>
    </div>
    <div id="betsList" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"></div>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import {
    getFirestore, collection, getDocs, doc,
    setDoc, onSnapshot, writeBatch
} from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

const firebaseConfig = {
    apiKey: "AIzaSyBo8G3ZcWk4EepN0cHdVBtXc7tGOfcw-yg",
    authDomain: "inscricaosinuca.firebaseapp.com",
    projectId: "inscricaosinuca",
    storageBucket: "inscricaosinuca.firebasestorage.app",
    messagingSenderId: "338241576305",
    appId: "1:338241576305:web:288b6124384c6be4f76ad0"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

let allParticipants = [];
let currentDraw = [];

/* ADMIN */
window.toggleAdmin = () => {
    if (!localStorage.getItem("admin")) {
        const pass = prompt("Senha do administrador:");
        if (pass !== "bolao2025") return;
        localStorage.setItem("admin", "ok");
    }
    document.getElementById("adminPanel").classList.toggle("hidden");
};

/* IMPORTAÇÃO EXCEL */
document.getElementById("fileInput").addEventListener("change", async e => {
    const file = e.target.files[0];
    const reader = new FileReader();

    reader.onload = async evt => {
        const data = new Uint8Array(evt.target.result);
        const wb = XLSX.read(data, { type: "array" });
        const rows = XLSX.utils.sheet_to_json(wb.Sheets[wb.SheetNames[0]], { header: 1 });

        const snap = await getDocs(collection(db, "participants"));
        const batch = writeBatch(db);
        snap.forEach(d => batch.delete(d.ref));

        let count = 0;

        rows.forEach((row, index) => {
            if (index === 0) return;

            const name = String(row[0] || "").trim();
            if (!name) return;

            const extract = txt =>
                String(txt || "").match(/\d+/g)
                    ?.map(n => parseInt(n))
                    .filter(n => n >= 1 && n <= 60) || [];

            const nums1 = extract(row[1]);
            const nums2 = extract(row[2]);

            let bets = [];
            if (nums1.length === 6) bets.push(nums1.sort((a,b)=>a-b));
            if (nums2.length === 6) bets.push(nums2.sort((a,b)=>a-b));

            if (bets.length) {
                batch.set(doc(collection(db, "participants")), { name, bets });
                count++;
            }
        });

        await batch.commit();
        alert(`✅ ${count} participantes importados`);
        location.reload();
    };

    reader.readAsArrayBuffer(file);
});

/* RESULTADO */
window.saveDrawResult = async () => {
    const nums = document.getElementById("drawInput").value.match(/\d+/g);
    if (!nums || nums.length < 6) return alert("Informe 6 números");
    const draw = nums.map(n => parseInt(n)).slice(0,6).sort((a,b)=>a-b);
    await setDoc(doc(db,"settings","draw"), { numbers: draw });
};

onSnapshot(doc(db,"settings","draw"), s => {
    currentDraw = s.exists() ? s.data().numbers : [];
    document.getElementById("drawInput").value = currentDraw.join(" ");
    render();
});

onSnapshot(collection(db,"participants"), s => {
    allParticipants = s.docs.map(d => d.data());
    document.getElementById("loading").classList.add("hidden");
    render();
});

/* RENDER */
function render() {
    const container = document.getElementById("betsList");
    container.innerHTML = "";

    const searchText = document.getElementById("searchInput").value.toLowerCase();
    const searchNums = searchText.match(/\d+/g)?.map(n=>parseInt(n)) || [];

    let stats = { sena:0, quina:0, quadra:0 };

    allParticipants.forEach(p => {
        p.bets.forEach(b => {
            if (!currentDraw.length) return;
            const hits = b.filter(n=>currentDraw.includes(n)).length;
            if (hits===6) stats.sena++;
            else if (hits===5) stats.quina++;
            else if (hits===4) stats.quadra++;
        });
    });

    document.getElementById("countSena").innerText = stats.sena;
    document.getElementById("countQuina").innerText = stats.quina;
    document.getElementById("countQuadra").innerText = stats.quadra;

    allParticipants.forEach(p => {
        const match =
            !searchText ||
            p.name.toLowerCase().includes(searchText) ||
            p.bets.some(b => searchNums.every(n => b.includes(n)));

        if (!match) return;

        const card = document.createElement("div");
        card.className = "bg-white p-4 rounded-lg shadow border";

        let html = `<h3 class="font-bold mb-3">${p.name}</h3>`;

        p.bets.forEach((b, i) => {
            const hits = currentDraw.length
                ? b.filter(n=>currentDraw.includes(n)).length
                : 0;

            const bg =
                hits===6?'sena-bg':
                hits===5?'quina-bg':
                hits===4?'quadra-bg':
                'bg-gray-50';

            html += `
            <div class="p-2 mb-2 rounded ${bg}">
                <div class="text-xs mb-1">
                    Jogo ${i+1} • ${currentDraw.length ? hits + ' acertos' : 'aguardando sorteio'}
                </div>
                <div class="flex flex-wrap gap-1 justify-center">
                    ${b.map(n=>`
                        <span class="w-7 h-7 rounded-full flex items-center justify-center text-xs border
                        ${currentDraw.includes(n)?'hit-number':'bg-white'}">
                        ${String(n).padStart(2,'0')}
                        </span>`).join("")}
                </div>
            </div>`;
        });

        card.innerHTML = html;
        container.appendChild(card);
    });
}

document.getElementById("searchInput").addEventListener("input", render);
</script>

</body>
</html>
