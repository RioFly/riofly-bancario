
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Painel Bancário - RioFly Aviation</title>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@400;600&display=swap" rel="stylesheet" />
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: 'Outfit', sans-serif;
    }
    body {
      background: #eaf0f5;
      color: #333;
    }
    header {
      background-color: #003366;
      color: white;
      padding: 2rem;
      text-align: center;
    }
    main {
      max-width: 1000px;
      margin: 2rem auto;
      background: white;
      padding: 2rem;
      border-radius: 16px;
      box-shadow: 0 6px 18px rgba(0,0,0,0.1);
    }
    h2 {
      color: #003366;
      margin-bottom: 1rem;
    }
    .saldo {
      font-size: 1.5rem;
      font-weight: 600;
      color: green;
      margin-bottom: 2rem;
    }
    .grafico-container {
      margin-bottom: 2rem;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-bottom: 2rem;
    }
    th, td {
      padding: 0.75rem;
      border: 1px solid #ccc;
      text-align: left;
      vertical-align: middle;
    }
    th {
      background-color: #f4f4f4;
    }
    button.btn-apagar {
      background-color: #cc3300;
      border: none;
      padding: 0.3rem 0.6rem;
      color: white;
      border-radius: 6px;
      cursor: pointer;
      transition: background 0.3s;
    }
    button.btn-apagar:hover {
      background-color: #991f00;
    }
    @media (max-width: 600px) {
      main {
        margin: 1rem;
        padding: 1rem;
      }
      table {
        font-size: 0.85rem;
      }
      button.btn-apagar {
        padding: 0.2rem 0.4rem;
        font-size: 0.85rem;
      }
    }
  </style>
</head>
<body>
  <header>
    <h1>Banco RioFly</h1>
    <p>Controle Financeiro</p>
  </header>
  <main>
    <h2>Saldo Atual</h2>
    <div class="saldo" id="saldoAtual">Carregando...</div>

    <div class="grafico-container">
      <h2>Histórico de Lucros (Voos)</h2>
      <canvas id="graficoLucro" height="100"></canvas>
    </div>

    <section>
      <h2>Transações de Voos Recentes</h2>
      <table>
        <thead>
          <tr>
            <th>Data</th>
            <th>Comandante</th>
            <th>Aeronave</th>
            <th>Matrícula</th>
            <th>Lucro (R$)</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody id="tabelaVoos"></tbody>
      </table>
    </section>

    <section>
      <h2>Transações de Manutenção Recentes</h2>
      <table>
        <thead>
          <tr>
            <th>Data</th>
            <th>Aeronave</th>
            <th>Descrição</th>
            <th>Custo (R$)</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody id="tabelaManutencao"></tbody>
      </table>
    </section>
  </main>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-app.js";
    import { getDatabase, ref, onValue, remove, runTransaction } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-database.js";

    const firebaseConfig = {
      apiKey: "AIzaSyD7SE9XR48nqXKS_vvmk6c4cJ9ITJAumko",
      authDomain: "riofly-aviation.firebaseapp.com",
      projectId: "riofly-aviation",
      storageBucket: "riofly-aviation.appspot.com",
      messagingSenderId: "162992793097",
      appId: "1:162992793097:web:d4ee41e33dd948d0a0bad8",
      databaseURL: "https://riofly-aviation-default-rtdb.firebaseio.com"
    };

    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    const saldoEl = document.getElementById("saldoAtual");
    const tabelaVoos = document.getElementById("tabelaVoos");
    const tabelaManutencao = document.getElementById("tabelaManutencao");
    const graficoCanvas = document.getElementById("graficoLucro");

    const saldoRef = ref(db, 'banco/saldo');
    const diarioRef = ref(db, 'diario_bordo');
    const manutencaoRef = ref(db, 'manutencao');

    let dadosGrafico = [];
    let rotulosGrafico = [];
    let voosAtuais = {};
    let manutencoesAtuais = {};
    let chartInstance = null;

    // Atualiza o saldo exibido
    onValue(saldoRef, (snapshot) => {
      const saldo = snapshot.val();
      saldoEl.textContent = saldo !== null ? `R$ ${saldo.toLocaleString('pt-BR')}` : "R$ 0";
    });

    // Atualiza voos
    onValue(diarioRef, (snapshot) => {
      const data = snapshot.val();
      tabelaVoos.innerHTML = "";
      dadosGrafico = [];
      rotulosGrafico = [];
      voosAtuais = data || {};

      // Ordena voos por timestamp
      const ordenado = Object.entries(data || {}).sort((a, b) => a[1].timestamp - b[1].timestamp);

      ordenado.forEach(([id, voo]) => {
        let aeronave = voo.aeronave;
        let modelo = aeronave;
        let matricula = "";

        const partes = aeronave.split(" ");
        if (partes.length >= 2) {
          modelo = partes.slice(0, -1).join(" ");
          matricula = partes[partes.length - 1];
        }

        const tr = document.createElement("tr");
        tr.innerHTML = `
          <td>${voo.data}</td>
          <td>${voo.comandante} (${voo.vid})</td>
          <td>${modelo}</td>
          <td>${matricula}</td>
          <td>R$ ${voo.lucro.toLocaleString('pt-BR')}</td>
          <td><button class="btn-apagar" data-id="${id}" data-tipo="voo">Apagar</button></td>
        `;
        tabelaVoos.appendChild(tr);

        rotulosGrafico.push(voo.data);
        dadosGrafico.push(voo.lucro);
      });

      renderizarGrafico();
      adicionarEventosApagar();
    });

    // Atualiza manutenções
    onValue(manutencaoRef, (snapshot) => {
      const data = snapshot.val();
      tabelaManutencao.innerHTML = "";
      manutencoesAtuais = data || {};

      // Ordena manutenções por timestamp
      const ordenado = Object.entries(data || {}).sort((a, b) => a[1].timestamp - b[1].timestamp);

      ordenado.forEach(([id, manut]) => {
        const tr = document.createElement("tr");
        tr.innerHTML = `
          <td>${manut.data}</td>
          <td>${manut.aeronave}</td>
          <td>${manut.descricao || ''}</td>
          <td>R$ ${manut.custo.toLocaleString('pt-BR')}</td>
          <td><button class="btn-apagar" data-id="${id}" data-tipo="manutencao">Apagar</button></td>
        `;
        tabelaManutencao.appendChild(tr);
      });

      adicionarEventosApagar();
    });

    // Função para adicionar eventos aos botões apagar (tanto voos quanto manutenção)
    function adicionarEventosApagar() {
      document.querySelectorAll("button.btn-apagar").forEach(btn => {
        btn.onclick = async (e) => {
          const id = btn.getAttribute("data-id");
          const tipo = btn.getAttribute("data-tipo");

          if (tipo === "voo") {
            const voo = voosAtuais[id];
            if (!voo) return alert("Voo não encontrado.");
            if (!confirm(`Deseja apagar o voo do comandante ${voo.comandante} na data ${voo.data}?`)) return;

            try {
              await remove(ref(db, `diario_bordo/${id}`));
              await runTransaction(saldoRef, (currentSaldo) => {
                if (currentSaldo === null) return 0;
                return currentSaldo - (voo.lucro || 0);
              });
              alert("Voo apagado e saldo atualizado com sucesso!");
            } catch (error) {
              alert("Erro ao apagar voo: " + error.message);
            }

          } else if (tipo === "manutencao") {
            const manut = manutencoesAtuais[id];
            if (!manut) return alert("Transação de manutenção não encontrada.");
            if (!confirm(`Deseja apagar a manutenção da aeronave ${manut.aeronave} na data ${manut.data}?`)) return;

            try {
              await remove(ref(db, `manutencao/${id}`));
              await runTransaction(saldoRef, (currentSaldo) => {
                if (currentSaldo === null) return 0;
                return currentSaldo + (manut.custo || 0); // somar custo pois é despesa
              });
              alert("Manutenção apagada e saldo atualizado com sucesso!");
            } catch (error) {
              alert("Erro ao apagar manutenção: " + error.message);
            }
          }
        };
      });
    }

    // Renderiza gráfico e previne múltiplas instâncias
    function renderizarGrafico() {
      if(chartInstance) {
        chartInstance.destroy();
      }
      chartInstance = new Chart(graficoCanvas, {
        type: 'line',
        data: {
          labels: rotulosGrafico,
          datasets: [{
            label: 'Lucro (R$)',
            data: dadosGrafico,
            fill: false,
            borderColor: 'green',
            tension: 0.2
          }]
        },
        options: {
          responsive: true,
          plugins: {
            legend: { display: true }
          }
        }
      });
    }
  </script>
</body>
</html>
