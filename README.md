<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Painel Bancário - RioFly Aviation</title>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@400;600&display=swap" rel="stylesheet">
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
    @media (max-width: 600px) {
      main {
        margin: 1rem;
        padding: 1rem;
      }
      table {
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
            <th>Lucro</th>
          </tr>
        </thead>
        <tbody id="tabelaTransacoes"></tbody>
      </table>
    </div>
  </main>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-app.js";
    import { getDatabase, ref, onValue } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-database.js";

    const firebaseConfig = {
      apiKey: "AIzaSyD7SE9XR48nqXKS_vvmk6c4cJ9ITJAumko",
      authDomain: "riofly-aviation.firebaseapp.com",
      projectId: "riofly-aviation",
      storageBucket: "riofly-aviation.firebasestorage.app",
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

    onValue(saldoRef, (snapshot) => {
      const saldo = snapshot.val();
      saldoEl.textContent = `R$ ${saldo.toLocaleString('pt-BR')}`;
    });

    let dadosGrafico = [];
    let rotulosGrafico = [];

    onValue(diarioRef, (snapshot) => {
      const data = snapshot.val();
      tabela.innerHTML = "";
      dadosGrafico = [];
      rotulosGrafico = [];

      const ordenado = Object.entries(data || {}).sort((a, b) => a[1].timestamp - b[1].timestamp);

      ordenado.forEach(([id, voo]) => {
        const tr = document.createElement("tr");
        tr.innerHTML = `
          <td>${voo.data}</td>
          <td>${voo.comandante} (${voo.vid})</td>
          <td>${voo.aeronave}</td>
          <td>R$ ${voo.lucro.toLocaleString('pt-BR')}</td>
        `;
        tabela.appendChild(tr);

        rotulosGrafico.push(voo.data);
        dadosGrafico.push(voo.lucro);
      });

      renderizarGrafico();
    });

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
