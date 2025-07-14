
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
    }
    th, td {
      padding: 0.75rem;
      border: 1px solid #ccc;
      text-align: left;
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
      <h2>Histórico de Lucros</h2>
      <canvas id="graficoLucro" height="100"></canvas>
    </div>

    <div>
      <h2>Transações Recentes</h2>
      <table>
        <thead>
          <tr>
            <th>Data</th>
            <th>Comandante</th>
            <th>Aeronave</th>
            <th>Matrícula</th>
            <th>Lucro</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody id="tabelaTransacoes"></tbody>
      </table>
    </div>
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
    const tabela = document.getElementById("tabelaTransacoes");
    const graficoCanvas = document.getElementById("graficoLucro");

    const saldoRef = ref(db, 'banco/saldo');
    const diarioRef = ref(db, 'diario_bordo');

    let dadosGrafico = [];
    let rotulosGrafico = [];
    let voosAtuais = {};

    // Atualiza o saldo exibido
    onValue(saldoRef, (snapshot) => {
      const saldo = snapshot.val();
      saldoEl.textContent = saldo !== null ? `R$ ${saldo.toLocaleString('pt-BR')}` : "R$ 0";
    });

    // Atualiza a tabela de voos e gráfico
    onValue(diarioRef, (snapshot) => {
      const data = snapshot.val();
      tabela.innerHTML = "";
      dadosGrafico = [];
      rotulosGrafico = [];
      voosAtuais = data || {};

      // Ordena voos por timestamp (assumindo que cada voo tem timestamp)
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
          <td><button class="btn-apagar" data-id="${id}">Apagar</button></td>
        `;
        tabela.appendChild(tr);
      });

      renderizarGrafico();

      // Adiciona evento para apagar voos
      document.querySelectorAll(".btn-apagar").forEach(btn => {
        btn.addEventListener("click", async (e) => {
          const id = e.target.getAttribute("data-id");
          const voo = voosAtuais[id];
          if (!voo) {
            alert("Voo não encontrado.");
            return;
          }
          if (confirm(`Deseja apagar o voo do comandante ${voo.comandante} na data ${voo.data}?`)) {
            try {
              // Remove o voo do banco
              await remove(ref(db, `diario_bordo/${id}`));
              // Atualiza o saldo subtraindo o lucro do voo removido
              await runTransaction(saldoRef, (currentSaldo) => {
                if (currentSaldo === null) return 0;
                return currentSaldo - (voo.lucro || 0);
              });
              alert("Voo apagado e saldo atualizado com sucesso!");
            } catch (error) {
              alert("Erro ao apagar voo: " + error.message);
            }
          }
        });
      });
    });

    // Função para renderizar gráfico dos lucros
    function renderizarGrafico() {
      new Chart(graficoCanvas, {
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
            legend: {
              display: true
            }
          }
        }
      });
    }
  </script>
</body>
</html>
