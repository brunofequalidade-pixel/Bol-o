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
        .duplicate-bg { background-color: #fee2e2; border: 2px solid #ef4444; } /* Novo estilo para duplicados */
        .hit-number { background-color: #16a34a; color: white; font-weight: bold; border-color: #16a34a; }
    </style>
</head>
<body class="bg-gray-100 min-h-screen pb-20">

    <nav class="bg-green-700 text-white p-4 shadow-lg sticky top-0 z-50">
        <div class="container mx-auto flex flex-col md:flex-row justify-between items-center gap-4">
            <h1 class="text-2xl font-bold flex items-center gap-2"><i class="fas fa-clover"></i> Bolão</h1>
            <input type="text" id="searchInput" placeholder="Pesquisar por nome ou dezenas (ex: 1020 ou 10 20)..." class="w-full md:w-1/2 p-2 rounded text-gray-800 focus:outline-none">
            <button onclick="toggleAdminPanel()" class="bg-green-800 px-3 py-1 rounded text-xs border border-green-600">Admin</button>
        </div>
    </nav>

    <div id="passwordModal" class="hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div class="bg-white rounded-lg p-6 max-w-sm w-full mx-4 shadow-xl">
            <h3 class="text-lg font-bold mb-4 text-gray-800">Acesso Administrativo</h3>
            <input type="password" id="adminPassword" placeholder="Digite a senha" class="w-full p-3 border border-gray-300 rounded mb-4 focus:outline-none focus:border-green-500">
            <div class="flex gap-2">
                <button onclick="closePasswordModal()" class="flex-1 bg-gray-300 text-gray-700 px-4 py-2 rounded hover:bg-gray-400 transition">Cancelar</button>
                <button onclick="checkPassword()" class="flex-1 bg-green-600 text-white px-4 py-2 rounded hover:bg-green-700 transition">Entrar</button>
            </div>
            <p id="passwordError" class="text-red-500 text-sm mt-2 hidden">Senha incorreta!</p>
        </div>
    </div>

    <div class="container mx-auto mt-6 px-4">
        <div class="bg-white p-4 rounded-lg shadow-md grid grid-cols-2 md:grid-cols-4 gap-2 text-center font-bold">
            <div class="bg-gray-100 p-2 rounded border border-gray-400 text-gray-700"><div class="text-[10px] uppercase">Participantes</div><span id="countTotal">0</span></div>
            <div class="bg-green-50 p-2 rounded border border-green-500 text-green-700"><div class="text-[10px] uppercase">Sena</div><span id="countSena">0</span></div>
            <div class="bg-yellow-50 p-2 rounded border border-yellow-500 text-yellow-700"><div class="text-[10px] uppercase">Quina</div><span id="countQuina">0</span></div>
            <div class="bg-blue-50 p-2 rounded border border-blue-500 text-blue-700"><div class="text-[10px] uppercase">Quadra</div><span id="countQuadra">0</span></div>
        </div>
    </div>

    <div id="adminPanel" class="hidden container mx-auto mt-6 px-4">
        <div class="bg-gray-200 p-4 rounded-lg border border-gray-300 shadow-inner">
            <div class="mb-4 bg-white p-3 rounded shadow-sm">
                <label class="block font-bold mb-1 text-sm text-gray-700">Resultado do Sorteio:</label>
                <div class="flex gap-2">
                    <input type="text" id="drawInput" placeholder="Ex: 01 02 03 04 05 06" class="flex-1 p-2 rounded border border-gray-300">
                    <button onclick="saveDrawResult()" class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 transition">Salvar</button>
                </div>
            </div>
            <div class="bg-white p-3 rounded shadow-sm">
                <label class="block font-bold mb-1 text-sm text-gray-700">Importar Excel:</label>
                <input type="file" id="fileInput" accept=".xlsx, .csv" class="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:bg-green-50 file:text-green-700 hover:file:bg-green-100"/>
                <p class="text-xs text-gray-500 mt-2">💡 Formato: 1ª coluna = Nome | Demais colunas = Números dos jogos (6 números por jogo)</p>
            </div>
        </div>
    </div>

    <div class="container mx-auto mt-6 px-4">
        <div id="loading" class="text-center py-10 text-gray-500"><i class="fas fa-spinner fa-spin fa-2x"></i><p class="mt-2">Carregando bolão...</p></div>
        <div id="betsList" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"></div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, getDocs, doc, setDoc, onSnapshot, writeBatch } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

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
        let isAdminAuthenticated = false;

        // Funções de autenticação admin
        window.toggleAdminPanel = () => {
            if (isAdminAuthenticated) {
                document.getElementById('adminPanel').classList.toggle('hidden');
            } else {
                document.getElementById('passwordModal').classList.remove('hidden');
                document.getElementById('adminPassword').focus();
            }
        };

        window.closePasswordModal = () => {
            document.getElementById('passwordModal').classList.add('hidden');
            document.getElementById('adminPassword').value = '';
            document.getElementById('passwordError').classList.add('hidden');
        };

        window.checkPassword = () => {
            const password = document.getElementById('adminPassword').value;
            if (password === 'bolao2025') {
                isAdminAuthenticated = true;
                closePasswordModal();
                document.getElementById('adminPanel').classList.remove('hidden');
            } else {
                document.getElementById('passwordError').classList.remove('hidden');
                document.getElementById('adminPassword').value = '';
                document.getElementById('adminPassword').focus();
            }
        };

        // Enter no campo de senha
        document.addEventListener('DOMContentLoaded', () => {
            document.getElementById('adminPassword').addEventListener('keypress', (e) => {
                if (e.key === 'Enter') checkPassword();
            });
        });

        function parseNumbersFromLine(row) {
            let allNumbers = [];
            for (let i = 1; i < row.length; i++) {
                if (row[i] !== null && row[i] !== undefined && row[i] !== '') {
                    const cellValue = row[i].toString();
                    const nums = cellValue.match(/\d+/g);
                    if (nums) {
                        allNumbers.push(...nums.map(n => parseInt(n, 10)));
                    }
                }
            }
            return allNumbers.filter(n => n > 0 && n <= 60);
        }

        document.getElementById('fileInput').addEventListener('change', (e) => {
            const file = e.target.files[0];
            const reader = new FileReader();
            reader.onload = async (evt) => {
                try {
                    const data = new Uint8Array(evt.target.result);
                    const workbook = XLSX.read(data, { type: 'array' });
                    const rows = XLSX.utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]], { header: 1 });
                    
                    const snap = await getDocs(collection(db, "participants"));
                    const batch = writeBatch(db);
                    snap.forEach(d => batch.delete(d.ref));

                    let count = 0;
                    let skipped = 0;
                    
                    rows.forEach((row, index) => {
                        if (!row || row.length === 0 || row.every(cell => !cell)) {
                            skipped++;
                            return;
                        }

                        const nome = row[0] ? row[0].toString().trim() : "";
                        const nomeLower = nome.toLowerCase();
                        if (!nome || nomeLower === "nome" || nomeLower === "participante" || nomeLower === "apostador") {
                            skipped++;
                            return;
                        }

                        const todosNumeros = parseNumbersFromLine(row);
                        
                        if (todosNumeros.length < 6) {
                            skipped++;
                            return;
                        }
                        
                        let validBets = [];
                        for (let i = 0; i < todosNumeros.length; i += 6) {
                            const grupo = todosNumeros.slice(i, i + 6);
                            if (grupo.length === 6) validBets.push(grupo.sort((a, b) => a - b));
                        }

                        if (validBets.length > 0) {
                            const betsForFirestore = validBets.map(bet => ({ numbers: bet }));
                            const docRef = doc(collection(db, "participants"));
                            batch.set(docRef, { name: nome, bets: betsForFirestore });
                            count++;
                        } else {
                            skipped++;
                        }
                    });

                    if (count > 0) {
                        await batch.commit();
                        alert(`✅ ${count} participantes carregados com sucesso!`);
                        location.reload();
                    } else {
                        alert(`⚠️ Nenhum participante importado.`);
                    }
                } catch (err) { 
                    console.error('Erro na importação:', err);
                    alert("Erro: " + err.message); 
                }
            };
            reader.readAsArrayBuffer(file);
        });

        window.saveDrawResult = async () => {
            const input = document.getElementById('drawInput').value;
            const nums = input.match(/\d+/g);
            if (!nums || nums.length < 6) return alert("Insira os 6 números do sorteio!");
            const sorted = nums.map(n => parseInt(n)).slice(0, 6).sort((a,b)=>a-b);
            await setDoc(doc(db, "settings", "drawResult"), { numbers: sorted });
            alert("Resultado salvo!");
        };

        onSnapshot(doc(db, "settings", "drawResult"), (s) => {
            currentDraw = s.exists() ? s.data().numbers : [];
            document.getElementById('drawInput').value = currentDraw.join(' ');
            render();
        });

        onSnapshot(collection(db, "participants"), (s) => {
            allParticipants = s.docs.map(d => d.data());
            document.getElementById('loading').classList.add('hidden');
            render();
        });

        function render() {
            const container = document.getElementById('betsList');
            container.innerHTML = '';
            
            // Atualiza total de participantes na tela
            document.getElementById('countTotal').innerText = allParticipants.length;

            const rawSearch = document.getElementById('searchInput').value.toLowerCase();
            let stats = { sena: 0, quina: 0, quadra: 0 };
            let foundCount = 0;

            // 1. Mapeamento de Jogos Duplicados
            const betCounts = {};
            allParticipants.forEach(p => {
                p.bets.forEach(b => {
                    const nums = b.numbers || b;
                    // Cria uma chave única para o jogo (ex: "1,10,20,30,40,50")
                    const key = nums.slice().sort((a,b)=>a-b).join(',');
                    betCounts[key] = (betCounts[key] || 0) + 1;
                });
            });

            const searchNums = rawSearch.match(/\d{1,2}/g) || [];
            
            const sortedList = [...allParticipants].sort((a, b) => {
                const maxA = Math.max(...a.bets.map(bet => {
                    const nums = bet.numbers || bet;
                    return nums.filter(n => currentDraw.includes(n)).length;
                }));
                const maxB = Math.max(...b.bets.map(bet => {
                    const nums = bet.numbers || bet;
                    return nums.filter(n => currentDraw.includes(n)).length;
                }));
                return maxB - maxA;
            });

            sortedList.forEach(p => {
                const matchesName = p.name.toLowerCase().includes(rawSearch);
                
                const matchesBets = searchNums.length > 0 && p.bets.some(b => {
                    const nums = b.numbers || b;
                    return searchNums.every(searchNum => {
                        const sInt = parseInt(searchNum);
                        return nums.includes(sInt);
                    });
                });

                if (rawSearch && !matchesName && !matchesBets) return;

                foundCount++;

                const card = document.createElement('div');
                card.className = "bg-white p-4 rounded-lg shadow-sm hover:shadow-md transition border border-gray-200";
                
                let betsHtml = p.bets.map((bet, idx) => {
                    const nums = bet.numbers || bet;
                    const hits = nums.filter(n => currentDraw.includes(n)).length;
                    
                    // Verifica duplicidade usando a chave gerada anteriormente
                    const key = nums.slice().sort((a,b)=>a-b).join(',');
                    const isDuplicate = betCounts[key] > 1;

                    if (hits === 6) stats.sena++; else if (hits === 5) stats.quina++; else if (hits === 4) stats.quadra++;
                    
                    // Lógica de cores: Prioriza vitória, senão verifica duplicidade
                    let bg = hits === 6 ? 'sena-bg' : 
                             hits === 5 ? 'quina-bg' : 
                             hits === 4 ? 'quadra-bg' : 
                             isDuplicate ? 'duplicate-bg' : 'bg-gray-50';

                    let duplicateIcon = isDuplicate ? '<span class="text-red-600 ml-2 text-[10px]"><i class="fas fa-exclamation-circle"></i> Duplicado</span>' : '';

                    return `
                        <div class="p-2 rounded mb-2 ${bg} transition-colors">
                            <div class="text-[10px] text-gray-400 mb-1 font-bold uppercase flex justify-between">
                                <span>Jogo ${idx+1} • ${hits} acertos</span>
                                ${duplicateIcon}
                            </div>
                            <div class="flex flex-wrap gap-1.5 justify-center">
                                ${nums.map(n => `<span class="w-7 h-7 flex items-center justify-center rounded-full text-xs border ${currentDraw.includes(n) ? 'hit-number' : 'bg-white border-gray-200'}">${n.toString().padStart(2,'0')}</span>`).join('')}
                            </div>
                        </div>`;
                }).join('');

                card.innerHTML = `<h3 class="font-bold text-gray-700 border-b mb-3 pb-1 truncate">${p.name}</h3>${betsHtml}`;
                container.appendChild(card);
            });

            console.log(`📈 Total encontrados: ${foundCount}`);
            
            if (foundCount === 0) {
                container.innerHTML = '<div class="col-span-full text-center py-10 text-gray-500"><i class="fas fa-search fa-2x mb-2"></i><p>Nenhum resultado encontrado</p></div>';
            }

            updateStats(stats);
        }

        function updateStats(stats) {
            document.getElementById('countSena').innerText = stats.sena;
            document.getElementById('countQuina').innerText = stats.quina;
            document.getElementById('countQuadra').innerText = stats.quadra;
        }

        document.getElementById('searchInput').addEventListener('input', render);
    </script>
</body>
</html>
