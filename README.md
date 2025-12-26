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

    <nav class="bg-green-700 text-white p-4 shadow-lg sticky top-0 z-50">
        <div class="container mx-auto flex flex-col md:flex-row justify-between items-center gap-4">
            <h1 class="text-2xl font-bold flex items-center gap-2"><i class="fas fa-clover"></i> Bolão</h1>
            <input type="text" id="searchInput" placeholder="Pesquisar por nome ou dezena..." class="w-full md:w-1/2 p-2 rounded text-gray-800 focus:outline-none">
            <button onclick="document.getElementById('adminPanel').classList.toggle('hidden')" class="bg-green-800 px-3 py-1 rounded text-xs border border-green-600">Admin</button>
        </div>
    </nav>

    <div class="container mx-auto mt-6 px-4">
        <div class="bg-white p-4 rounded-lg shadow-md grid grid-cols-3 gap-2 text-center font-bold">
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

        function parseNumbersFromLine(row) {
            // Converte todas as células (exceto a primeira) para string e extrai números
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
            console.log('Números extraídos da linha:', allNumbers);
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
                    
                    console.log('=== INICIANDO IMPORTAÇÃO ===');
                    console.log('Total de linhas na planilha:', rows.length);
                    
                    const snap = await getDocs(collection(db, "participants"));
                    const batch = writeBatch(db);
                    snap.forEach(d => batch.delete(d.ref));

                    let count = 0;
                    let skipped = 0;
                    let debugInfo = [];
                    
                    rows.forEach((row, index) => {
                        const nome = row[0] ? row[0].toString().trim() : "";
                        
                        // Debug: registra todas as linhas
                        console.log(`Linha ${index}:`, { nome, row });
                        
                        // Ignora linhas vazias ou cabeçalhos óbvios
                        if (!nome || nome.toLowerCase() === "nome" || nome.toLowerCase() === "participante") {
                            debugInfo.push(`Linha ${index} ignorada: cabeçalho ou vazia`);
                            skipped++;
                            return;
                        }

                        const todosNumeros = parseNumbersFromLine(row);
                        console.log(`${nome} - Total de números encontrados:`, todosNumeros.length, todosNumeros);
                        
                        let validBets = [];
                        // Agrupa de 6 em 6
                        for (let i = 0; i < todosNumeros.length; i += 6) {
                            const grupo = todosNumeros.slice(i, i + 6);
                            if (grupo.length === 6) validBets.push(grupo.sort((a, b) => a - b));
                        }

                        console.log(`${nome} - Jogos válidos:`, validBets.length, validBets);

                        if (validBets.length > 0) {
                            try {
                                // Cria um documento com ID único
                                const docRef = doc(collection(db, "participants"));
                                batch.set(docRef, { 
                                    name: nome, 
                                    bets: validBets
                                });
                                count++;
                                debugInfo.push(`✓ ${nome}: ${validBets.length} jogo(s)`);
                            } catch (batchError) {
                                console.error(`Erro ao adicionar ${nome}:`, batchError);
                                debugInfo.push(`✗ ${nome}: erro ao adicionar - ${batchError.message}`);
                                skipped++;
                            }
                        } else {
                            debugInfo.push(`✗ ${nome}: nenhum jogo válido (${todosNumeros.length} números)`);
                            skipped++;
                        }
                    });

                    console.log('=== RESUMO DA IMPORTAÇÃO ===');
                    console.log(`Total de linhas: ${rows.length}`);
                    console.log(`Importados: ${count}`);
                    console.log(`Ignorados: ${skipped}`);
                    console.log('Detalhes:', debugInfo);

                    if (count > 0) {
                        await batch.commit();
                        alert(`✅ ${count} participantes carregados com sucesso!\n${skipped} linhas ignoradas.\n\nVeja o console (F12) para detalhes.`);
                        location.reload();
                    } else {
                        alert(`⚠️ Nenhum participante foi importado.\n${skipped} linhas ignoradas.\n\nVerifique o console (F12) para detalhes.`);
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
            const search = document.getElementById('searchInput').value.toLowerCase();
            let stats = { sena: 0, quina: 0, quadra: 0 };

            // Ordenação: quem tem mais acertos primeiro
            const sortedList = [...allParticipants].sort((a, b) => {
                const maxA = Math.max(...a.bets.map(bet => bet.filter(n => currentDraw.includes(n)).length));
                const maxB = Math.max(...b.bets.map(bet => bet.filter(n => currentDraw.includes(n)).length));
                return maxB - maxA;
            });

            sortedList.forEach(p => {
                const matchesSearch = p.name.toLowerCase().includes(search) || 
                                    p.bets.some(b => b.some(n => n.toString() == search));
                if (search && !matchesSearch) return;

                const card = document.createElement('div');
                card.className = "bg-white p-4 rounded-lg shadow-sm hover:shadow-md transition border border-gray-200";
                
                let betsHtml = p.bets.map((bet, idx) => {
                    const hits = bet.filter(n => currentDraw.includes(n)).length;
                    if (hits === 6) stats.sena++; else if (hits === 5) stats.quina++; else if (hits === 4) stats.quadra++;
                    
                    let bg = hits === 6 ? 'sena-bg' : hits === 5 ? 'quina-bg' : hits === 4 ? 'quadra-bg' : 'bg-gray-50';
                    return `
                        <div class="p-2 rounded mb-2 ${bg} transition-colors">
                            <div class="text-[10px] text-gray-400 mb-1 font-bold uppercase">Jogo ${idx+1} • ${hits} acertos</div>
                            <div class="flex flex-wrap gap-1.5 justify-center">
                                ${bet.map(n => `<span class="w-7 h-7 flex items-center justify-center rounded-full text-xs border ${currentDraw.includes(n) ? 'hit-number' : 'bg-white border-gray-200'}">${n.toString().padStart(2,'0')}</span>`).join('')}
                            </div>
                        </div>`;
                }).join('');

                card.innerHTML = `<h3 class="font-bold text-gray-700 border-b mb-3 pb-1 truncate">${p.name}</h3>${betsHtml}`;
                container.appendChild(card);
            });

            document.getElementById('countSena').innerText = stats.sena;
            document.getElementById('countQuina').innerText = stats.quina;
            document.getElementById('countQuadra').innerText = stats.quadra;
        }

        document.getElementById('searchInput').addEventListener('input', render);
    </script>
</body>
</html>
