# base-erros
# Base Interactions Monitor

Monitora interações onchain na **Base (chainId 8453)** para uma lista de endereços
e aponta **falhas** (transações revertidas), incluindo hash, bloco, contrato alvo e gas usado.
Usa a API Etherscan-compatível do **Basescan** e RPC para detalhes opcionais.

## Como funciona
1. Busca as últimas `N` transações por endereço via Basescan (`account/txlist`).
2. Filtra **interações com contrato** (tem `input` ≠ `0x`).
3. Marca como **falha** quando `isError = "1"` (ou `receiptStatus = "0"`).
4. (Opcional) consulta o recibo via RPC para gas usado.
5. (Opcional) envia alerta para Webhook (Discord/Telegram/etc).

## Uso local
```bash
git clone <seu-repo>
cd <seu-repo>
npm i
cp .env.example .env
# Edite .env com suas chaves e endereços
node src/checker.js



---

# 2) `package.json`
```json
{
  "name": "base-interactions-monitor",
  "version": "1.0.0",
  "type": "module",
  "main": "src/checker.js",
  "scripts": {
    "start": "node src/checker.js"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "dotenv": "^16.4.5",
    "ethers": "^6.13.2"
  }
}
# API Etherscan-compatível (Basescan)
BASESCAN_API_KEY=COLOQUE_SUA_CHAVE

# Endereços separados por vírgula
ADDRESSES=0xSEUENDERECO1,0xSEUENDERECO2

# Quantidade de transações por endereço (padrão 50)
NUM_TX=50

# RPC público da Base (ou seu provedor)
RPC_URL=https://mainnet.base.org

# (Opcional) Webhook para alertas (Discord/Telegram/Slack)
WEBHOOK_URL=

import 'dotenv/config';
import axios from 'axios';
import { JsonRpcProvider } from 'ethers';

const BASESCAN_API = 'https://api.basescan.org/api';

const {
  BASESCAN_API_KEY,
  ADDRESSES,
  NUM_TX = '50',
  RPC_URL,
  WEBHOOK_URL
} = process.env;

if (!BASESCAN_API_KEY) throw new Error('Defina BASESCAN_API_KEY no .env');
if (!ADDRESSES) throw new Error('Defina ADDRESSES no .env (separado por vírgula)');

const provider = RPC_URL ? new JsonRpcProvider(RPC_URL) : null;

function fmtWei(gasUsed, gasPrice) {
  try {
    const gu = BigInt(gasUsed ?? 0n);
    const gp = BigInt(gasPrice ?? 0n);
    const wei = gu * gp;
    // gwei approx
    const gwei = Number(wei) / 1e9;
    return `${gwei.toFixed(2)} gwei-wei`;
  } catch {
    return '-';
  }
}

async function fetchTxList(address, limit) {
  const url = `${BASESCAN_API}?module=account&action=txlist&address=${address}&page=1&offset=${limit}&sort=desc&apikey=${BASESCAN_API_KEY}`;
  const { data } = await axios.get(url, { timeout: 20000 });
  if (data.status !== '1' && !Array.isArray(data.result)) {
    console.warn(`[WARN] Basescan retornou vazio para ${address}:`, data.message || data.result);
    return [];
  }
  return data.result;
}

async function getReceipt(hash) {
  if (!provider) return null;
  try {
    return await provider.getTransactionReceipt(hash);
  } catch {
    return null;
  }
}

async function sendWebhook(message) {
  if (!WEBHOOK_URL) return;
  try {
    await axios.post(WEBHOOK_URL, { content: message });
  } catch (e) {
    console.warn('[WARN] Falha ao enviar webhook:', e.message);
  }
}

function row(obj) {
  return [
    obj.address,
    obj.hash,
    obj.blockNumber,
    obj.to,
    obj.status,
    obj.gasUsed || '-'
  ].join(' | ');
}

function asMarkdownTable(items) {
  const header = '| address | txHash | block | to(contract) | status | gasUsed |\n|---|---|---:|---|---|---:|';
  const lines = items.map(i =>
    `| ${i.address} | \`${i.hash}\` | ${i.blockNumber} | \`${i.to}\` | ${i.status} | ${i.gasUsed || '-'} |`
  );
  return [header, ...lines].join('\n');
}

async function main() {
  const addresses = ADDRESSES.split(',').map(s => s.trim()).filter(Boolean);
  const failures = [];

  for (const address of addresses) {
    console.log(`\n=== Analisando ${address} ===`);
    const txs = await fetchTxList(address, Number(NUM_TX));

    const interactions = txs.filter(tx => tx.input && tx.input !== '0x'); // chama contrato
    for (const tx of interactions) {
      const failed = tx.isError === '1' || tx.txreceipt_status === '0';
      let gasUsed = null;

      if (provider) {
        const receipt = await getReceipt(tx.hash);
        if (receipt) gasUsed = receipt.gasUsed?.toString();
      }

      const item = {
        address,
        hash: tx.hash,
        blockNumber: tx.blockNumber,
        to: tx.to,
        status: failed ? 'FAILED' : 'OK',
        gasUsed
      };

      console.log(row(item));

      if (failed) failures.push(item);
    }
  }

  // Gera relatório
  let report = `# Base Interactions Report\n\nGerado em: ${new Date().toISOString()}\n\n`;
  if (failures.length === 0) {
    report += '✅ Nenhuma falha encontrada nas interações analisadas.\n';
  } else {
    report += `⚠️ Falhas encontradas: **${failures.length}**\n\n`;
    report += asMarkdownTable(failures) + '\n';
  }

  // salva em arquivo
  const fs = await import('node:fs/promises');
  await fs.writeFile('report.md', report);

  // envia alerta resumido
  if (failures.length > 0) {
    const msg = [
      `⚠️ *Falhas onchain na Base*: ${failures.length}`,
      ...failures.slice(0, 5).map(f => `• ${f.address} → \`${f.hash}\` (block ${f.blockNumber})`)
    ].join('\n');
    await sendWebhook(msg);
  }

  console.log('\n---\nRelatório salvo em report.md');
}

main().catch(err => {
  console.error('[ERROR]', err.message);
  process.exit(1);
});

name: Base Monitor

on:
  schedule:
    - cron: "*/30 * * * *"   # a cada 30 minutos
  workflow_dispatch:

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install
        run: npm ci || npm i

      - name: Run checker
        env:
          BASESCAN_API_KEY: ${{ secrets.BASESCAN_API_KEY }}
          ADDRESSES: ${{ secrets.ADDRESSES }}
          NUM_TX: ${{ secrets.NUM_TX }}
          RPC_URL: ${{ secrets.RPC_URL }}
          WEBHOOK_URL: ${{ secrets.WEBHOOK_URL }}
        run: npm start

      - name: Upload report artifact
        uses: actions/upload-artifact@v4
        with:
          name: base-report
          path: report.md

bora criar um repositório “monitor” que **verifica interações na rede Base** para 1+ endereços e **aponta falhas** (tx revertida, gas usado, contrato alvo), com opção de **alerta no Discord/Telegram** e **GitHub Actions** rodando automático.
Abaixo estão **todos os arquivos** pra você copiar/colar.

---

# 1) `README.md`

````markdown
# Base Interactions Monitor

Monitora interações onchain na **Base (chainId 8453)** para uma lista de endereços
e aponta **falhas** (transações revertidas), incluindo hash, bloco, contrato alvo e gas usado.
Usa a API Etherscan-compatível do **Basescan** e RPC para detalhes opcionais.

## Como funciona
1. Busca as últimas `N` transações por endereço via Basescan (`account/txlist`).
2. Filtra **interações com contrato** (tem `input` ≠ `0x`).
3. Marca como **falha** quando `isError = "1"` (ou `receiptStatus = "0"`).
4. (Opcional) consulta o recibo via RPC para gas usado.
5. (Opcional) envia alerta para Webhook (Discord/Telegram/etc).

## Uso local
```bash
git clone <seu-repo>
cd <seu-repo>
npm i
cp .env.example .env
# Edite .env com suas chaves e endereços
node src/checker.js
````

## Variáveis (.env)

* `BASESCAN_API_KEY=` sua chave do [https://basescan.org](https://basescan.org) (menu API Keys)
* `ADDRESSES=` endereços separados por vírgula (ex: 0xabc...,0xdef...)
* `NUM_TX=` quantas TX por endereço analisar (padrão 50)
* `RPC_URL=` RPC da Base (ex: [https://mainnet.base.org](https://mainnet.base.org) ou Alchemy/Infura)
* `WEBHOOK_URL=` (opcional) URL de webhook para alertas

## Saída

* Tabela no console
* Arquivo `report.md` com falhas do período
* (Opcional) Alerta no seu Webhook

## Automação (GitHub Actions)

O workflow `.github/workflows/monitor.yml` roda **a cada 30 min**.
Guarde as secrets no repositório:

* `BASESCAN_API_KEY`, `ADDRESSES`, `RPC_URL`, `WEBHOOK_URL` (opcional)

````

---

# 2) `package.json`
```json
{
  "name": "base-interactions-monitor",
  "version": "1.0.0",
  "type": "module",
  "main": "src/checker.js",
  "scripts": {
    "start": "node src/checker.js"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "dotenv": "^16.4.5",
    "ethers": "^6.13.2"
  }
}
````

---

# 3) `.env.example`

```dotenv
# API Etherscan-compatível (Basescan)
BASESCAN_API_KEY=COLOQUE_SUA_CHAVE

# Endereços separados por vírgula
ADDRESSES=0xSEUENDERECO1,0xSEUENDERECO2

# Quantidade de transações por endereço (padrão 50)
NUM_TX=50

# RPC público da Base (ou seu provedor)
RPC_URL=https://mainnet.base.org

# (Opcional) Webhook para alertas (Discord/Telegram/Slack)
WEBHOOK_URL=
```

---

# 4) `src/checker.js`

```js
import 'dotenv/config';
import axios from 'axios';
import { JsonRpcProvider } from 'ethers';

const BASESCAN_API = 'https://api.basescan.org/api';

const {
  BASESCAN_API_KEY,
  ADDRESSES,
  NUM_TX = '50',
  RPC_URL,
  WEBHOOK_URL
} = process.env;

if (!BASESCAN_API_KEY) throw new Error('Defina BASESCAN_API_KEY no .env');
if (!ADDRESSES) throw new Error('Defina ADDRESSES no .env (separado por vírgula)');

const provider = RPC_URL ? new JsonRpcProvider(RPC_URL) : null;

function fmtWei(gasUsed, gasPrice) {
  try {
    const gu = BigInt(gasUsed ?? 0n);
    const gp = BigInt(gasPrice ?? 0n);
    const wei = gu * gp;
    // gwei approx
    const gwei = Number(wei) / 1e9;
    return `${gwei.toFixed(2)} gwei-wei`;
  } catch {
    return '-';
  }
}

async function fetchTxList(address, limit) {
  const url = `${BASESCAN_API}?module=account&action=txlist&address=${address}&page=1&offset=${limit}&sort=desc&apikey=${BASESCAN_API_KEY}`;
  const { data } = await axios.get(url, { timeout: 20000 });
  if (data.status !== '1' && !Array.isArray(data.result)) {
    console.warn(`[WARN] Basescan retornou vazio para ${address}:`, data.message || data.result);
    return [];
  }
  return data.result;
}

async function getReceipt(hash) {
  if (!provider) return null;
  try {
    return await provider.getTransactionReceipt(hash);
  } catch {
    return null;
  }
}

async function sendWebhook(message) {
  if (!WEBHOOK_URL) return;
  try {
    await axios.post(WEBHOOK_URL, { content: message });
  } catch (e) {
    console.warn('[WARN] Falha ao enviar webhook:', e.message);
  }
}

function row(obj) {
  return [
    obj.address,
    obj.hash,
    obj.blockNumber,
    obj.to,
    obj.status,
    obj.gasUsed || '-'
  ].join(' | ');
}

function asMarkdownTable(items) {
  const header = '| address | txHash | block | to(contract) | status | gasUsed |\n|---|---|---:|---|---|---:|';
  const lines = items.map(i =>
    `| ${i.address} | \`${i.hash}\` | ${i.blockNumber} | \`${i.to}\` | ${i.status} | ${i.gasUsed || '-'} |`
  );
  return [header, ...lines].join('\n');
}

async function main() {
  const addresses = ADDRESSES.split(',').map(s => s.trim()).filter(Boolean);
  const failures = [];

  for (const address of addresses) {
    console.log(`\n=== Analisando ${address} ===`);
    const txs = await fetchTxList(address, Number(NUM_TX));

    const interactions = txs.filter(tx => tx.input && tx.input !== '0x'); // chama contrato
    for (const tx of interactions) {
      const failed = tx.isError === '1' || tx.txreceipt_status === '0';
      let gasUsed = null;

      if (provider) {
        const receipt = await getReceipt(tx.hash);
        if (receipt) gasUsed = receipt.gasUsed?.toString();
      }

      const item = {
        address,
        hash: tx.hash,
        blockNumber: tx.blockNumber,
        to: tx.to,
        status: failed ? 'FAILED' : 'OK',
        gasUsed
      };

      console.log(row(item));

      if (failed) failures.push(item);
    }
  }

  // Gera relatório
  let report = `# Base Interactions Report\n\nGerado em: ${new Date().toISOString()}\n\n`;
  if (failures.length === 0) {
    report += '✅ Nenhuma falha encontrada nas interações analisadas.\n';
  } else {
    report += `⚠️ Falhas encontradas: **${failures.length}**\n\n`;
    report += asMarkdownTable(failures) + '\n';
  }

  // salva em arquivo
  const fs = await import('node:fs/promises');
  await fs.writeFile('report.md', report);

  // envia alerta resumido
  if (failures.length > 0) {
    const msg = [
      `⚠️ *Falhas onchain na Base*: ${failures.length}`,
      ...failures.slice(0, 5).map(f => `• ${f.address} → \`${f.hash}\` (block ${f.blockNumber})`)
    ].join('\n');
    await sendWebhook(msg);
  }

  console.log('\n---\nRelatório salvo em report.md');
}

main().catch(err => {
  console.error('[ERROR]', err.message);
  process.exit(1);
});
```

---

# 5) `.github/workflows/monitor.yml`

```yaml
name: Base Monitor

on:
  schedule:
    - cron: "*/30 * * * *"   # a cada 30 minutos
  workflow_dispatch:

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install
        run: npm ci || npm i

      - name: Run checker
        env:
          BASESCAN_API_KEY: ${{ secrets.BASESCAN_API_KEY }}
          ADDRESSES: ${{ secrets.ADDRESSES }}
          NUM_TX: ${{ secrets.NUM_TX }}
          RPC_URL: ${{ secrets.RPC_URL }}
          WEBHOOK_URL: ${{ secrets.WEBHOOK_URL }}
        run: npm start

      - name: Upload report artifact
        uses: actions/upload-artifact@v4
        with:
          name: base-report
          path: report.md
```

---

## Como colocar para rodar agora

1. Crie o repositório novo (`base-interactions-monitor`).
2. Adicione os arquivos acima (mesmos nomes/paths).
3. **Settings → Secrets and variables → Actions → New repository secret** e crie:

   * `BASESCAN_API_KEY`
   * `ADDRESSES` (ex.: `0xSEU1,0xSEU2`)
   * `RPC_URL` (ex.: `https://mainnet.base.org`)
   * (opcional) `WEBHOOK_URL`
4. Vá em **Actions** e rode `Base Monitor` com **Run workflow**.

Se quiser, me diga os endereços que você quer monitorar e te devolvo um `.env` prontinho, ou ajusto o script para também checar **logs de eventos (ERC-20/721)** e separar falhas por tipo (approve, transfer, mint, etc.).
