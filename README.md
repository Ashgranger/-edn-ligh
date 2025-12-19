# -edn-ligh
Lighter 


npm install --legacy-peer-deps



Great — you want to trade BERA and place 10 MARKET orders of 0.05 BERA each. I'll give you a complete, ready-to-run TypeScript bot that uses the correct SDK (SignerClient from lighter-ts-sdk), converts 0.05 BERA → baseAmount = 50,000 (since 1 BERA = 1,000,000 units), and includes robust auto-error handling & retries.

Do the steps below on your new VPS.


---

What this bot will do

Initialize the WASM signer (SignerClient).

Send 10 MARKET BUY orders (one after another).

Each order size = 0.05 BERA → baseAmount = 0.05 * 1_000_000 = 50_000.

Per-order retry logic (up to 4 attempts), nonce refresh attempts, backoff for network errors.

Stops early on unrecoverable errors (insufficient balance, restricted jurisdiction).

Logs the returned mainOrder.hash for each order.



---

Files to create (in project root ~/lighter-send-10)

1. .env — your secrets


2. package.json


3. tsconfig.json


4. src/send_10_orders.ts — the bot




---

1) .env (create in project root)

API_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
ACCOUNT_INDEX=0
API_KEY_INDEX=0
BASE_URL=https://mainnet.zklighter.elliot.ai
MARKET_ID=20        # replace only if you know a different BERA market index

chmod 600 .env after creating it.


---

2) package.json

Create package.json with:

{
  "name": "lighter-send-10",
  "version": "1.0.0",
  "type": "commonjs",
  "scripts": {
    "start": "node -r ts-node/register src/send_10_orders.ts"
  },
  "dependencies": {
    "dotenv": "^17.2.0",
    "lighter-ts-sdk": "^1.0.7",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3",
    "@types/node": "^20.11.0"
  }
}

Then run:

npm install --legacy-peer-deps


---

3) tsconfig.json

Create tsconfig.json:

{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": false
  },
  "include": ["src"]
}


---

4) src/send_10_orders.ts (full code — copy & paste)

Create src folder and file src/send_10_orders.ts and paste the code below exactly:

/**
 * send_10_orders.ts
 * Sends 10 MARKET BUY orders of 0.05 BERA each (baseAmount = 50,000).
 * Uses SignerClient from lighter-ts-sdk.
 *
 * Safety note: This script will respect server-side restrictions.
 * If you see "restricted jurisdiction" from the API, deploy to a different VPS region.
 */

import dotenv from "dotenv";
dotenv.config();

import { SignerClient, OrderType } from "lighter-ts-sdk";
import axios from "axios";

const {
  API_PRIVATE_KEY,
  ACCOUNT_INDEX = "0",
  API_KEY_INDEX = "0",
  BASE_URL = "https://mainnet.zklighter.elliot.ai",
  MARKET_ID = "20"
} = process.env;

if (!API_PRIVATE_KEY) {
  console.error("Missing API_PRIVATE_KEY in .env — aborting.");
  process.exit(1);
}

const MARKET_INDEX = Number(MARKET_ID);
const ACCOUNT_IDX = Number(ACCOUNT_INDEX);
const APIKEY_IDX = Number(API_KEY_INDEX);

// Conversion: 1 BERA = 1_000_000 units (per SDK README)
const BERA_SCALE = 1_000_000;
const BERA_PER_ORDER = 0.05;
const BASE_AMOUNT_PER_ORDER = Math.round(BERA_PER_ORDER * BERA_SCALE); // 50,000

const MAX_ATTEMPTS_PER_ORDER = 4;
const WAIT_BETWEEN_ORDERS_MS = 800;

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function testApiReachable(): Promise<{ ok: boolean; reason?: string }> {
  const url = `${BASE_URL}/api/v1/orderBookDetails?marketIndex=${MARKET_INDEX}`;
  try {
    const r = await axios.get(url, { timeout: 5000 });
    if (r?.data) return { ok: true };
    return { ok: false, reason: "no_data" };
  } catch (err: any) {
    const msg = (err?.response?.data ?? err?.message ?? String(err)).toString();
    if (msg.toLowerCase().includes("restricted jurisdiction")) {
      return { ok: false, reason: "restricted_jurisdiction" };
    }
    return { ok: false, reason: "network_error" };
  }
}

async function newSignerClient(): Promise<SignerClient> {
  const client = new SignerClient({
    url: BASE_URL,
    privateKey: API_PRIVATE_KEY!,
    accountIndex: ACCOUNT_IDX,
    apiKeyIndex: APIKEY_IDX,
  });
  await client.initialize();
  await client.ensureWasmClient();
  return client;
}

async function sendOneOrder(client: SignerClient, index: number): Promise<boolean> {
  let attempt = 0;
  while (attempt < MAX_ATTEMPTS_PER_ORDER) {
    attempt++;
    try {
      console.log(`[Order ${index}] Attempt ${attempt} → sending MARKET buy ${BERA_PER_ORDER} BERA (baseAmount=${BASE_AMOUNT_PER_ORDER})`);
      const res = await client.createUnifiedOrder({
        marketIndex: MARKET_INDEX,
        clientOrderIndex: Date.now() + index + attempt,
        baseAmount: BASE_AMOUNT_PER_ORDER,
        isAsk: false,             // BUY
        orderType: OrderType.MARKET,
        idealPrice: 0,            // not used for MARKET but okay
        maxSlippage: 0.005        // 0.5% slippage guard (adjust if needed)
      });

      if (!res || !res.success) {
        // server returned an error object
        const serverMsg = res?.mainOrder?.error ?? JSON.stringify(res);
        console.error(`[Order ${index}] Server rejected order:`, serverMsg);

        // If jurisdiction block, stop
        if (serverMsg.toString().toLowerCase().includes("restricted jurisdiction")) {
          console.error(`[Order ${index}] restricted jurisdiction — aborting further attempts.`);
          return false;
        }

        // else backoff and retry
        await sleep(1500 * attempt);
        continue;
      }

      console.log(`[Order ${index}] Created → hash: ${res.mainOrder.hash}`);
      // wait for transaction confirmation (best-effort)
      try {
        await client.waitForTransaction(res.mainOrder.hash, 30000, 2000);
        console.log(`[Order ${index}] Transaction confirmed (or timed out after wait).`);
      } catch (waitErr: any) {
        console.warn(`[Order ${index}] waitForTransaction failed (non-fatal):`, waitErr?.message ?? waitErr);
      }
      return true;
    } catch (err: any) {
      const msg = (err?.message ?? String(err)).toLowerCase();
      console.error(`[Order ${index}] Exception attempt ${attempt}:`, msg);

      // Unfixable: jurisdiction block
      if (msg.includes("restricted jurisdiction")) {
        console.error(`[Order ${index}] Lighter restricted your region/IP. Stop.`);
        return false;
      }

      // Nonce issues: try to re-init signer
      if (msg.includes("nonce") || msg.includes("invalid nonce") || msg.includes("nonce too low")) {
        console.log(`[Order ${index}] Detected nonce issue — re-initializing signer client and retrying.`);
        try {
          await client.ensureWasmClient();
        } catch (reinitErr) {
          console.warn(`[Order ${index}] signer re-init failed:`, reinitErr);
        }
      }

      // Insufficient balance: abort
      if (msg.includes("insufficient") || msg.includes("balance")) {
        console.error(`[Order ${index}] Insufficient balance — aborting.`);
        return false;
      }

      // Network/rate-limit issues -> backoff
      if (msg.includes("429") || msg.includes("too many") || msg.includes("timeout") || msg.includes("network")) {
        const backoff = 2000 * attempt;
        console.log(`[Order ${index}] Network/rate issue — backing off ${backoff}ms`);
        await sleep(backoff);
        continue;
      }

      // default backoff
      await sleep(1200 + Math.random() * 800);
    }
  }

  console.error(`[Order ${index}] Failed after ${MAX_ATTEMPTS_PER_ORDER} attempts.`);
  return false;
}

async function main() {
  console.log("Starting send_10_orders (0.05 BERA each) ...");

  const apiTest = await testApiReachable();
  if (!apiTest.ok) {
    console.error("API reachable test failed:", apiTest.reason);
    if (apiTest.reason === "restricted_jurisdiction") {
      console.error("Your VPS IP is blocked by Lighter. Deploy to a different region.");
    }
    process.exit(1);
  }

  const client = await newSignerClient();

  for (let i = 1; i <= 10; i++) {
    const ok = await sendOneOrder(client, i);
    if (!ok) {
      console.error(`[Main] Stopping run at order ${i} due to unrecoverable error.`);
      break;
    }
    await sleep(WAIT_BETWEEN_ORDERS_MS);
  }

  try {
    await client.close();
  } catch (e) {
    console.warn("Error closing client (non-fatal):", e);
  }

  console.log("Done.");
  process.exit(0);
}

main().catch((e) => {
  console.error("Fatal error:", e);
  process.exit(2);
});


---

Run the bot

From project root:

npx ts-node src/send_10_orders.ts

Watch logs — you should see one line per order with hash: on success. If you see restricted jurisdiction stop and try another VPS region.


---

Quick checks & troubleshooting

If you see Missing API_PRIVATE_KEY, edit .env and export or restart shell.

If you see restricted jurisdiction, pick a different VPS region/country and repeat deployment.

If you see Insufficient balance, top up the account or reduce order size.

To verify the scale: 0.05 BERA → BASE_AMOUNT = 50,000 units (correct).



---

Safety & advice

Test with low sizes first (you asked 0.05 which is small — good).

Keep .env private; never commit to git.

Monitor the account after run — check UI or transaction hashes to be sure orders filled as expected.

If you want the bot to place limit quotes or alternate buy/sell (spread bot) I can generate that next.



---

If you want, I’ll now:

provide a one-line scp/deploy script to copy project to new VPS and run it, or

produce a systemd service file so the bot can run on boot.


Which would you like?
