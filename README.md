<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>BMD | Analisador Contábil</title>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

<style>

/* BASE */
body {
  margin: 0;
  font-family: "Segoe UI", Arial;
  background: linear-gradient(135deg, #0f172a, #020617);
  color: #e2e8f0;
}

/* HEADER */
.header {
  background: #020617;
  padding: 20px 30px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 2px solid #00aa88;
}

.logo {
  font-size: 30px;
  font-weight: bold;
  color: #00ffcc;
  letter-spacing: 4px;
}

.sub {
  font-size: 12px;
  color: #94a3b8;
}

/* CONTAINER */
.container {
  padding: 30px;
}

/* CARDS */
.card {
  background: #020617;
  border: 1px solid #1e293b;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 25px;
}

/* INPUT */
input {
  padding: 10px;
  border-radius: 6px;
  border: 1px solid #334155;
  background: #020617;
  color: white;
}

/* BOTÕES */
button {
  padding: 10px 18px;
  margin: 10px;
  border-radius: 6px;
  border: none;
  background: linear-gradient(135deg, #00aa88, #007766);
  color: white;
  font-weight: bold;
  cursor: pointer;
}

button:hover {
  opacity: 0.9;
}

/* LISTAS */
.box {
  display: flex;
  gap: 20px;
}

.coluna {
  flex: 1;
  padding: 15px;
  border-radius: 10px;
}

.positivo { background: #064e3b; }
.negativo { background: #7f1d1d; }

/* TABELA */
table {
  width: 100%;
  border-collapse: collapse;
}

th {
  background: #00aa88;
  padding: 10px;
}

td {
  padding: 8px;
  border: 1px solid #334155;
  background: #022c22;
}

.maior { background: #1d4ed8; }
.menor { background: #991b1b; }

</style>
</head>

<body>

<!-- HEADER -->
<div class="header">
  <div>
    <div class="logo">BMD</div>
    <div class="sub">Business Management & Data</div>
  </div>
  <div>📊 Analisador Contábil</div>
</div>

<div class="container">

<!-- UPLOAD -->
<div class="card">
  <h2>📂 Importar PDFs</h2>
  <input type="file" id="pdfInput" multiple accept="application/pdf">
  <br>
  <button onclick="baixarNegativosZIP()">⬇️ Negativos</button>
  <button onclick="baixarPositivosZIP()">⬇️ Positivos</button>
  <button onclick="exportarExcel()">📊 Excel</button>
</div>

<!-- LISTAS -->
<div class="card">
  <h2>📑 Classificação</h2>
  <div class="box">
    <div class="coluna positivo">
      <h3>✅ Positivos</h3>
      <ul id="positivos"></ul>
    </div>
    <div class="coluna negativo">
      <h3>❌ Negativos</h3>
      <ul id="negativos"></ul>
    </div>
  </div>
</div>

<!-- TABELA -->
<div class="card">
  <h2>📊 Resumo Contábil</h2>
  <table id="tabela">
    <thead>
      <tr>
        <th>Arquivo</th>
        <th>Empresa</th>
        <th>Resultado</th>
        <th>Produtos</th>
        <th>Mercadoria</th>
        <th>Serviços</th>
        <th>Simples</th>
        <th>Serv+Simples</th>
        <th>Comp</th>
        <th>Total</th>
        <th>Comp</th>
      </tr>
    </thead>
    <tbody id="tabelaResumo"></tbody>
  </table>
</div>

</div>

<script>

pdfjsLib.GlobalWorkerOptions.workerSrc =
"https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js";

const input = document.getElementById("pdfInput");

let arquivosNegativos = [];
let arquivosPositivos = [];

input.addEventListener("change", async (event) => {

  positivos.innerHTML = "";
  negativos.innerHTML = "";
  tabelaResumo.innerHTML = "";

  arquivosNegativos = [];
  arquivosPositivos = [];

  const arquivos = event.target.files;

  for (let file of arquivos) {

    const textoSimples = await lerPDFSimples(file);
    const ehNegativo = analisarTexto(textoSimples, file.name, file);

    if (!ehNegativo) {
      const textoTabela = await lerPDFTabela(file);
      extrairInformacoes(textoTabela, file.name);
    }
  }
});

/* LEITURA SIMPLES */
async function lerPDFSimples(file) {
  const reader = new FileReader();
  return new Promise((resolve) => {
    reader.onload = async function () {
      const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;
      let texto = "";

      for (let i = 1; i <= pdf.numPages; i++) {
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();
        content.items.forEach(i => texto += i.str + " ");
      }

      resolve(texto.toLowerCase());
    };
    reader.readAsArrayBuffer(file);
  });
}

function analisarTexto(texto, nome, file) {
  texto = texto.replace(/\s+/g, " ");
  const idx = texto.indexOf("resultado do período");

  if (idx !== -1) {
    const trecho = texto.substring(idx, idx + 300);
    const valores = trecho.match(/\(?\d{1,3}(?:\.\d{3})*,\d{2}\)?/g);

    if (valores && valores.length >= 4) {
      const saldo = valores[3];
      const negativo = saldo.includes("(");

      if (negativo) {
        adicionarLista("negativos", nome, saldo);
        arquivosNegativos.push(file);
        return true;
      } else {
        adicionarLista("positivos", nome, saldo);
        arquivosPositivos.push(file);
        return false;
      }
    }
  }
  return true;
}

function adicionarLista(tipo, nome, saldo) {
  const li = document.createElement("li");
  li.textContent = `${nome} → ${saldo}`;
  document.getElementById(tipo).appendChild(li);
}

/* TABELA (mantida lógica) */
async function lerPDFTabela(file) {
  const reader = new FileReader();
  return new Promise((resolve) => {
    reader.onload = async function () {
      const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;
      let linhas = [];

      for (let i = 1; i <= pdf.numPages; i++) {
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();

        let linha = "";
        content.items.forEach(i => linha += i.str + " ");
        linhas.push(linha);
      }

      resolve(linhas.join("\n").toLowerCase());
    };
    reader.readAsArrayBuffer(file);
  });
}

function extrairInformacoes(texto, nomeArquivo) {
  const tr = document.createElement("tr");
  tr.innerHTML = `<td>${nomeArquivo}</td><td>OK</td>`;
  tabelaResumo.appendChild(tr);
}

/* DOWNLOAD */
async function baixarNegativosZIP() {
  const zip = new JSZip();
  arquivosNegativos.forEach(f => zip.file(f.name, f));
  const blob = await zip.generateAsync({ type: "blob" });
  download(blob, "negativos.zip");
}

async function baixarPositivosZIP() {
  const zip = new JSZip();
  arquivosPositivos.forEach(f => zip.file(f.name, f));
  const blob = await zip.generateAsync({ type: "blob" });
  download(blob, "positivos.zip");
}

function download(blob, nome) {
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = nome;
  a.click();
}

function exportarExcel() {
  const wb = XLSX.utils.table_to_book(tabela);
  XLSX.writeFile(wb, "Resumo.xlsx");
}

</script>

</body>
</html>
