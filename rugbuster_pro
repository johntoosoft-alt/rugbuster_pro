// ============================================================
// RUGBUSTER PRO v3.0 — Complete Multi-User Non-Custodial Bot
// ============================================================
//
// REPLIT SECRETS:
//   TELEGRAM_BOT_TOKEN  — from @BotFather
//   ENCRYPTION_KEY      — any random 32+ char string
//
// INSTALL:
//   npm install node-telegram-bot-api @solana/web3.js @solana/spl-token axios bs58 qrcode
// ============================================================

'use strict';

const TelegramBot     = require('node-telegram-bot-api');
const { Keypair, Connection, PublicKey, Transaction,
        SystemProgram, LAMPORTS_PER_SOL }  = require('@solana/web3.js');
const axios  = require('axios');
const bs58   = require('bs58');
const QRCode = require('qrcode');
const fs     = require('fs');
const path   = require('path');

// ─── ENV ─────────────────────────────────────────────────────
['TELEGRAM_BOT_TOKEN', 'ENCRYPTION_KEY'].forEach(k => {
  if (!process.env[k]) { console.error(`❌ Missing Secret: ${k}`); process.exit(1); }
});
const TOKEN = process.env.TELEGRAM_BOT_TOKEN;

// ─── CONSTANTS ───────────────────────────────────────────────
const SOLANA_RPC   = 'https://api.mainnet-beta.solana.com';
const JUPITER_API  = 'https://quote-api.jup.ag/v6';
const DEXSCREENER  = 'https://api.dexscreener.com';
const RUGCHECK_API = 'https://api.rugcheck.xyz/v1';
const SOL_MINT     = 'So11111111111111111111111111111111111111112';
const WSOL_MINT    = 'So11111111111111111111111111111111111111112';

// ─── KNOWN SWAP TOKEN MINTS ──────────────────────────────────
const SWAP_TOKENS = {
  usdc: { mint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v', symbol: 'USDC', label: '💵 USDC' },
  usdt: { mint: 'Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB',  symbol: 'USDT', label: '💵 USDT' },
  sol:  { mint: 'So11111111111111111111111111111111111111112',     symbol: 'SOL',  label: '◎ SOL'  },
  btc:  { mint: '3NZ9JMVBmGAqocybic2c7LQCJScmgsAZ6vQqTDzcqmJh', symbol: 'BTC',  label: '₿ BTC'  },
  eth:  { mint: '7vfCXTUXx5WJV5JADk17DUJ4ksgau7utNKj4b963voxs', symbol: 'ETH',  label: 'Ξ ETH'  },
};

const connection = new Connection(SOLANA_RPC, 'confirmed');
const bot        = new TelegramBot(TOKEN, { polling: true });

// ─── STORAGE ─────────────────────────────────────────────────
const DATA_FILE   = path.join(process.cwd(), 'user_data.json');
const ALERTS_FILE = path.join(process.cwd(), 'alerts.json');
let users  = new Map();
let alerts = []; // { userId, mint, symbol, targetPrice, direction, chatId }

function loadData() {
  try {
    if (fs.existsSync(DATA_FILE)) {
      const raw = JSON.parse(fs.readFileSync(DATA_FILE, 'utf8'));
      users = new Map(raw.map(([k, v]) => [Number(k), v]));
      console.log(`📂 Loaded ${users.size} users`);
    }
    if (fs.existsSync(ALERTS_FILE)) {
      alerts = JSON.parse(fs.readFileSync(ALERTS_FILE, 'utf8'));
    }
  } catch (e) { console.log('Starting fresh'); }
}

function saveData() {
  try {
    fs.writeFileSync(DATA_FILE,   JSON.stringify([...users.entries()], null, 2));
    fs.writeFileSync(ALERTS_FILE, JSON.stringify(alerts, null, 2));
  } catch (e) { console.error('Save failed:', e.message); }
}

loadData();
setInterval(saveData, 30_000);

// ─── DEFAULT USER ────────────────────────────────────────────
const DEFAULT_SETTINGS = {
  slippageBps:       100,
  priorityFee:       0.0005,
  minLiqUsd:         5000,
  minScore:          60,
  quickBuyAmounts:   [0.1, 0.5, 1, 5],
  quickSellPercents: [25, 50, 75, 100],
  tpPercent:         null,
  slPercent:         null,
  copyWallets:       [],
  referralCode:      null,
  referredBy:        null,
  referralCount:     0
};

function getUser(userId) {
  if (!users.has(userId)) {
    users.set(userId, {
      walletAddress:  null,
      onboarded:      false,
      settings:       { ...DEFAULT_SETTINGS },
      positions:      [],
      tradingHistory: [],
      stats:          { totalTrades: 0, totalVolume: 0, totalPnL: 0, wins: 0 },
      referralCode:   genReferralCode(userId)
    });
  }
  return users.get(userId);
}

function genReferralCode(userId) {
  return 'RB' + Math.abs(userId % 100000).toString(36).toUpperCase().padStart(5, '0');
}

// ─── HELPERS ─────────────────────────────────────────────────
function fmt(n) {
  if (!n || isNaN(n)) return '0';
  if (n >= 1e9)  return (n / 1e9).toFixed(2)  + 'B';
  if (n >= 1e6)  return (n / 1e6).toFixed(2)  + 'M';
  if (n >= 1e3)  return (n / 1e3).toFixed(2)  + 'K';
  return n.toFixed(4);
}
function pct(n)     { return (n >= 0 ? '+' : '') + n.toFixed(2) + '%'; }
function isMint(s)  { return /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(s); }
function isAddr(s)  { return isMint(s); }

// ─── API: DEXSCREENER ────────────────────────────────────────
async function getTokenInfo(mint) {
  try {
    const { data } = await axios.get(`${DEXSCREENER}/latest/dex/tokens/${mint}`, { timeout: 8000 });
    const pair = data?.pairs?.[0];
    if (!pair) return null;
    return {
      name:           pair.baseToken?.name    || 'Unknown',
      symbol:         pair.baseToken?.symbol  || '???',
      mint:           pair.baseToken?.address || mint,
      priceUsd:       parseFloat(pair.priceUsd    || 0),
      priceNative:    parseFloat(pair.priceNative  || 0),
      liquidity:      pair.liquidity?.usd          || 0,
      volume24h:      pair.volume?.h24             || 0,
      priceChange1h:  pair.priceChange?.h1         || 0,
      priceChange24h: pair.priceChange?.h24        || 0,
      marketCap:      pair.marketCap               || 0,
      dexId:          pair.dexId                   || '',
      pairAddress:    pair.pairAddress             || ''
    };
  } catch (e) { return null; }
}

// ─── API: RUGCHECK ───────────────────────────────────────────
async function securityScan(mint) {
  try {
    const { data } = await axios.get(`${RUGCHECK_API}/tokens/${mint}/report/summary`, { timeout: 8000 });
    const score = data?.score ?? 50;
    const risks = (data?.risks || []).map(r => `• ${r.name}: ${r.description}`).join('\n') || '• None detected';
    return {
      score,
      risks,
      passed: score >= 50,
      grade:  score >= 80 ? '🟢 SAFE' : score >= 60 ? '🟡 MODERATE' : '🔴 RISKY'
    };
  } catch (e) {
    return { score: 50, risks: '• API unavailable', passed: false, grade: '⚠️ UNKNOWN' };
  }
}

// ─── WALLET: BALANCES ────────────────────────────────────────
async function getSolBalance(address) {
  try {
    return (await connection.getBalance(new PublicKey(address))) / LAMPORTS_PER_SOL;
  } catch (e) { return 0; }
}

async function getTokenBalance(walletAddress, mint) {
  try {
    const accounts = await connection.getParsedTokenAccountsByOwner(
      new PublicKey(walletAddress), { mint: new PublicKey(mint) }
    );
    if (!accounts.value.length) return { uiAmount: 0, decimals: 6, rawAmount: 0 };
    const info = accounts.value[0].account.data.parsed.info.tokenAmount;
    return { uiAmount: info.uiAmount || 0, decimals: info.decimals, rawAmount: parseInt(info.amount) };
  } catch (e) { return { uiAmount: 0, decimals: 6, rawAmount: 0 }; }
}

async function getAllTokenBalances(walletAddress) {
  try {
    const accounts = await connection.getParsedTokenAccountsByOwner(
      new PublicKey(walletAddress),
      { programId: new PublicKey('TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA') }
    );
    return accounts.value
      .map(a => ({
        mint:     a.account.data.parsed.info.mint,
        uiAmount: a.account.data.parsed.info.tokenAmount.uiAmount,
        decimals: a.account.data.parsed.info.tokenAmount.decimals
      }))
      .filter(t => t.uiAmount > 0);
  } catch (e) { return []; }
}

// ─── JUPITER SWAP ─────────────────────────────────────────────
async function jupiterSwap(privateKeyB58, inputMint, outputMint, amount, slippageBps, priorityFee) {
  const keypair = Keypair.fromSecretKey(bs58.decode(privateKeyB58));

  const { data: quote } = await axios.get(
    `${JUPITER_API}/quote?inputMint=${inputMint}&outputMint=${outputMint}` +
    `&amount=${amount}&slippageBps=${slippageBps}`,
    { timeout: 10_000 }
  );
  if (!quote || quote.error) throw new Error(quote?.error || 'No route found');

  const { data: swapData } = await axios.post(`${JUPITER_API}/swap`, {
    quoteResponse:             quote,
    userPublicKey:             keypair.publicKey.toBase58(),
    wrapAndUnwrapSol:          true,
    prioritizationFeeLamports: Math.floor(priorityFee * LAMPORTS_PER_SOL)
  }, { timeout: 10_000 });

  if (!swapData?.swapTransaction) throw new Error('Failed to build swap transaction');

  const tx = Transaction.from(Buffer.from(swapData.swapTransaction, 'base64'));
  tx.sign(keypair);
  const txId = await connection.sendRawTransaction(tx.serialize(), { skipPreflight: false, maxRetries: 3 });
  await connection.confirmTransaction(txId, 'confirmed');

  return { txId, outAmount: parseInt(quote.outAmount), explorerUrl: `https://solscan.io/tx/${txId}` };
}

// ─── SOL TRANSFER ────────────────────────────────────────────
async function sendSol(privateKeyB58, toAddress, amount) {
  const keypair   = Keypair.fromSecretKey(bs58.decode(privateKeyB58));
  const lamports  = Math.floor(amount * LAMPORTS_PER_SOL);
  const toKey     = new PublicKey(toAddress);

  const tx = new Transaction().add(
    SystemProgram.transfer({ fromPubkey: keypair.publicKey, toPubkey: toKey, lamports })
  );
  tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
  tx.feePayer        = keypair.publicKey;
  tx.sign(keypair);

  const txId = await connection.sendRawTransaction(tx.serialize(), { skipPreflight: false });
  await connection.confirmTransaction(txId, 'confirmed');
  return { txId, explorerUrl: `https://solscan.io/tx/${txId}` };
}

// ─── QR CODE ─────────────────────────────────────────────────
async function generateQR(text) {
  return QRCode.toBuffer(text, { type: 'png', width: 300, margin: 2 });
}

// ─── KEYBOARDS ───────────────────────────────────────────────
const kb = {
  // ── Main menu ──
  main: () => ({ inline_keyboard: [
    [{ text: '💼 Wallet',     callback_data: 'wallet_menu'  },
     { text: '🔍 Scan Token', callback_data: 'scan_prompt'  }],
    [{ text: '📊 Positions',  callback_data: 'positions'    },
     { text: '📈 PnL / Stats',callback_data: 'pnl'          }],
    [{ text: '🔔 Alerts',     callback_data: 'alerts_menu'  },
     { text: '👥 Copy Trade', callback_data: 'copy_menu'    }],
    [{ text: '📋 History',    callback_data: 'history'      },
     { text: '🎁 Referral',   callback_data: 'referral'     }],
    [{ text: '⚙️ Settings',   callback_data: 'settings'     }]
  ]}),

  // ── Wallet submenu ──
  wallet: () => ({ inline_keyboard: [
    [{ text: '💰 Portfolio & Tokens', callback_data: 'balance'      }],
    [{ text: '📤 Send',               callback_data: 'send_sol'     },
     { text: '📥 Receive',            callback_data: 'receive'      }],
    [{ text: '💱 Swap',               callback_data: 'swap_menu'    }],
    [{ text: '🔙 Back',               callback_data: 'main_menu'    }]
  ]}),

  // ── Swap coin picker ──
  swapPicker: (fromLabel) => ({ inline_keyboard: [
    [{ text: '💵 USDC',   callback_data: 'swap_to_usdc' },
     { text: '💵 USDT',   callback_data: 'swap_to_usdt' }],
    [{ text: '◎ SOL',    callback_data: 'swap_to_sol'  },
     { text: '₿ BTC',    callback_data: 'swap_to_btc'  }],
    [{ text: 'Ξ ETH',    callback_data: 'swap_to_eth'  },
     { text: '🔵 BASE',  callback_data: 'swap_to_base' }],
    [{ text: '✏️ Custom coin address', callback_data: 'swap_to_custom' }],
    [{ text: '🔙 Back',               callback_data: 'wallet_menu'    }]
  ]}),

  // ── Swap amount picker ──
  swapAmounts: (outputLabel) => ({ inline_keyboard: [
    [{ text: '10%',  callback_data: 'swapamt_10'  },
     { text: '25%',  callback_data: 'swapamt_25'  },
     { text: '50%',  callback_data: 'swapamt_50'  }],
    [{ text: '75%',  callback_data: 'swapamt_75'  },
     { text: '100%', callback_data: 'swapamt_100' }],
    [{ text: '✏️ Custom amount', callback_data: 'swapamt_custom' }],
    [{ text: '🔙 Back',         callback_data: 'swap_menu'       }]
  ]}),

  // ── Disclaimer ──
  disclaimer: () => ({ inline_keyboard: [
    [{ text: '✅ I understand — create my wallet', callback_data: 'confirm_onboard' }],
    [{ text: '❌ Cancel',                          callback_data: 'cancel_onboard'  }]
  ]}),

  // ── Token actions ──
  tokenBuy: (mint, amounts) => ({ inline_keyboard: [
    amounts.map(a => ({ text: `${a} SOL`, callback_data: `buyprompt_${mint}_${a}` })),
    [{ text: '✏️ Custom Amount',    callback_data: `buy_custom_${mint}`  },
     { text: '% of Balance',        callback_data: `buy_pct_${mint}`     }],
    [{ text: '🎯 Set TP/SL + Buy',  callback_data: `buy_tpsl_${mint}`    }],
    [{ text: '🔙 Menu',             callback_data: 'main_menu'           }]
  ]}),

  // ── Sell menu ──
  sellMenu: (mint, symbol, percents) => ({ inline_keyboard: [
    percents.map(p => ({ text: `${p}%`, callback_data: `sellprompt_${mint}_${p}` })),
    [{ text: '✏️ Custom %', callback_data: `sell_custom_${mint}` }],
    [{ text: '🔙 Positions', callback_data: 'positions' }]
  ]}),

  // ── Settings ──
  settings: (s) => ({ inline_keyboard: [
    [{ text: `📉 Slippage: ${s.slippageBps / 100}%`,        callback_data: 'set_slippage'  }],
    [{ text: `⚡ Priority Fee: ${s.priorityFee} SOL`,        callback_data: 'set_priority'  }],
    [{ text: `🎯 Min RugScore: ${s.minScore}`,               callback_data: 'set_minscore'  }],
    [{ text: `💧 Min Liquidity: $${fmt(s.minLiqUsd)}`,       callback_data: 'set_minliq'    }],
    [{ text: `✅ TP: ${s.tpPercent ?? 'OFF'}%  |  🛑 SL: ${s.slPercent ?? 'OFF'}%`,
                                                             callback_data: 'set_tpsl'      }],
    [{ text: '🔙 Menu', callback_data: 'main_menu' }]
  ]}),

  // ── Alerts ──
  alertsMenu: (userAlerts) => {
    const btns = userAlerts.map((a, i) => ([{
      text: `${a.symbol} ${a.direction === 'above' ? '📈' : '📉'} $${a.targetPrice}`,
      callback_data: `del_alert_${i}`
    }]));
    return { inline_keyboard: [
      ...btns,
      [{ text: '➕ New Alert',  callback_data: 'new_alert'  }],
      [{ text: '🔙 Menu',       callback_data: 'main_menu'  }]
    ]};
  },

  // ── Copy trade ──
  copyMenu: (copyWallets) => {
    const btns = copyWallets.map((w, i) => ([{
      text: `🗑 Remove ${w.slice(0, 8)}...`,
      callback_data: `del_copy_${i}`
    }]));
    return { inline_keyboard: [
      ...btns,
      [{ text: '➕ Add Wallet to Copy', callback_data: 'add_copy'   }],
      [{ text: '🔙 Menu',               callback_data: 'main_menu'  }]
    ]};
  },

  back: (dest = 'main_menu', label = '🔙 Back') => ({
    inline_keyboard: [[{ text: label, callback_data: dest }]]
  })
};

// ─── WAITING STATE ────────────────────────────────────────────
const waiting = new Map();

// ─── SEND / EDIT HELPERS ─────────────────────────────────────
const send = (cid, text, opts = {}) =>
  bot.sendMessage(cid, text, { parse_mode: 'Markdown', disable_web_page_preview: true, ...opts });

const edit = (cid, mid, text, opts = {}) =>
  bot.editMessageText(text, {
    chat_id: cid, message_id: mid,
    parse_mode: 'Markdown', disable_web_page_preview: true, ...opts
  });

// ─── KEY PROMPT ──────────────────────────────────────────────
async function promptForKey(chatId, userId, waitingState) {
  waiting.set(userId, waitingState);
  await send(chatId,
    `🔑 *Enter your private key to sign*\n\n` +
    `Used in memory only — never stored. Type /cancel to abort.`
  );
}

// ─── VALIDATE & LOAD KEY ─────────────────────────────────────
function loadAndVerifyKey(privateKeyB58, expectedAddress) {
  const keypair = Keypair.fromSecretKey(bs58.decode(privateKeyB58));
  if (keypair.publicKey.toBase58() !== expectedAddress) {
    throw new Error('Private key does not match your registered wallet');
  }
  return keypair;
}

// ─── ONBOARDING ──────────────────────────────────────────────
async function showDisclaimer(chatId) {
  await send(chatId,
    `⚠️ *Read this before continuing*\n\n` +
    `RugBuster Pro will generate a Solana wallet for you.\n\n` +
    `🔑 Your private key is shown *once and never stored*. If lost, funds are unrecoverable.\n` +
    `💸 This bot executes *real trades with real money*. You can lose funds.\n` +
    `🔒 To sign trades, you paste your key into chat. It is used in memory only and immediately discarded.\n` +
    `👤 Use at your own risk.\n\n` +
    `Do you understand?`,
    { reply_markup: kb.disclaimer() }
  );
}

async function generateAndShowWallet(chatId, userId, referredBy = null) {
  const keypair    = Keypair.generate();
  const address    = keypair.publicKey.toBase58();
  const privateKey = bs58.encode(keypair.secretKey);
  const ud         = getUser(userId);

  ud.walletAddress = address;
  ud.onboarded     = true;
  if (referredBy) {
    ud.settings.referredBy = referredBy;
    // Credit referrer
    for (const [uid, u] of users) {
      if (u.referralCode === referredBy) { u.referralCount = (u.referralCount || 0) + 1; break; }
    }
  }
  saveData();

  await send(chatId,
    `✅ *Wallet Created!*\n\n` +
    `📬 Address:\n\`${address}\`\n\n` +
    `🔑 *Private Key — SAVE THIS NOW (shown once):*\n\`${privateKey}\`\n\n` +
    `⚠️ Screenshot and store offline. This is NOT saved anywhere.\n\n` +
    `Fund your wallet with SOL then paste any token address to start.`,
    { reply_markup: kb.main() }
  );
}

// ─── TOKEN SCAN ──────────────────────────────────────────────
async function handleTokenScan(chatId, mint, ud) {
  const scanning = await send(chatId, '🔍 Scanning token...');
  const [tokenInfo, scan] = await Promise.all([getTokenInfo(mint), securityScan(mint)]);

  if (!tokenInfo) {
    return bot.editMessageText('❌ Token not found on DexScreener', {
      chat_id: chatId, message_id: scanning.message_id
    });
  }

  const liqWarn = tokenInfo.liquidity < ud.settings.minLiqUsd
    ? `\n⚠️ Below your min liquidity ($${fmt(ud.settings.minLiqUsd)})\n` : '';

  const scoreWarn = !scan.passed
    ? `\n⚠️ Below your min score (${ud.settings.minScore})\n` : '';

  const msg =
    `💎 *${tokenInfo.name} (${tokenInfo.symbol})*\n` +
    `\`${mint}\`\n\n` +
    `💰 Price: \`$${tokenInfo.priceUsd.toFixed(8)}\`\n` +
    `💧 Liquidity: \`$${fmt(tokenInfo.liquidity)}\`\n` +
    `📊 Volume 24h: \`$${fmt(tokenInfo.volume24h)}\`\n` +
    `📈 1h: \`${pct(tokenInfo.priceChange1h)}\` | 24h: \`${pct(tokenInfo.priceChange24h)}\`\n` +
    `🏦 MCap: \`$${fmt(tokenInfo.marketCap)}\`\n` +
    `🔗 DEX: ${tokenInfo.dexId}\n\n` +
    `🛡 *Security: ${scan.grade} (${scan.score}/100)*\n${scan.risks}` +
    liqWarn + scoreWarn +
    `\n\n📥 *Select buy option:*`;

  await bot.editMessageText(msg, {
    chat_id:      chatId,
    message_id:   scanning.message_id,
    parse_mode:   'Markdown',
    reply_markup: kb.tokenBuy(mint, ud.settings.quickBuyAmounts)
  });
}

// ─── BUY ─────────────────────────────────────────────────────
async function executeBuy(chatId, userId, privateKeyB58, mint, solAmount, tpPct = null, slPct = null) {
  const ud      = getUser(userId);
  const lamports = Math.floor(solAmount * LAMPORTS_PER_SOL);
  const msg      = await send(chatId, `⏳ Buying ${solAmount} SOL of token... (10–30s)`);

  try {
    loadAndVerifyKey(privateKeyB58, ud.walletAddress); // verify before swap
    const result    = await jupiterSwap(privateKeyB58, SOL_MINT, mint, lamports, ud.settings.slippageBps, ud.settings.priorityFee);
    const tokenInfo = await getTokenInfo(mint);

    const tp = tpPct ?? ud.settings.tpPercent;
    const sl = slPct ?? ud.settings.slPercent;

    const existing = ud.positions.find(p => p.mint === mint);
    if (existing) {
      existing.amount   += result.outAmount;
      existing.solSpent += solAmount;
    } else {
      ud.positions.push({
        mint,
        symbol:           tokenInfo?.symbol      || '???',
        name:             tokenInfo?.name        || 'Unknown',
        entryPrice:       tokenInfo?.priceUsd    || 0,
        entryPriceNative: tokenInfo?.priceNative || 0,
        amount:           result.outAmount,
        solSpent:         solAmount,
        tpPercent:        tp,
        slPercent:        sl,
        entryTime:        new Date().toISOString()
      });
    }

    // Register TP/SL price alerts automatically
    if (tp && tokenInfo) {
      alerts.push({
        userId, chatId, mint,
        symbol:      tokenInfo.symbol,
        targetPrice: tokenInfo.priceUsd * (1 + tp / 100),
        direction:   'above',
        type:        'tp'
      });
    }
    if (sl && tokenInfo) {
      alerts.push({
        userId, chatId, mint,
        symbol:      tokenInfo.symbol,
        targetPrice: tokenInfo.priceUsd * (1 - sl / 100),
        direction:   'below',
        type:        'sl'
      });
    }

    ud.stats.totalTrades++;
    ud.stats.totalVolume += solAmount;
    saveData();

    const tpslStr = tp || sl
      ? `\n🎯 TP: ${tp ?? '—'}% | 🛑 SL: ${sl ?? '—'}% (alerts set)` : '';

    await bot.editMessageText(
      `✅ *BUY CONFIRMED*\n\n` +
      `💸 Spent: ${solAmount} SOL\n` +
      `🎯 Received: ${result.outAmount.toLocaleString()} tokens` +
      tpslStr +
      `\n🔗 [Solscan](${result.explorerUrl})`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  } catch (e) {
    console.error('Buy error:', e.message);
    await bot.editMessageText(
      `❌ *Buy failed*\n\n${e.message}`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  }
}

// ─── SELL ────────────────────────────────────────────────────
async function executeSell(chatId, userId, privateKeyB58, mint, percent) {
  const ud      = getUser(userId);
  const keypair = loadAndVerifyKey(privateKeyB58, ud.walletAddress);
  const bal     = await getTokenBalance(keypair.publicKey.toBase58(), mint);

  if (bal.uiAmount === 0) return send(chatId, '❌ No token balance found', { reply_markup: kb.main() });

  const rawAmount = Math.floor(bal.rawAmount * (percent / 100));
  const msg       = await send(chatId, `⏳ Selling ${percent}% of position... (10–30s)`);

  try {
    const result    = await jupiterSwap(privateKeyB58, mint, SOL_MINT, rawAmount, ud.settings.slippageBps, ud.settings.priorityFee);
    const tokenInfo = await getTokenInfo(mint);
    const position  = ud.positions.find(p => p.mint === mint);

    let pnl = 0;
    if (position) {
      const cp = tokenInfo?.priceNative || 0;
      pnl = (cp - position.entryPriceNative) / (position.entryPriceNative || 1) * position.solSpent * (percent / 100);
      ud.stats.totalPnL += pnl;
      if (pnl > 0) ud.stats.wins++;

      if (percent >= 100) {
        ud.tradingHistory.unshift({ ...position, exitTime: new Date().toISOString(), pnl });
        ud.positions = ud.positions.filter(p => p.mint !== mint);
        // Clear alerts for this token
        alerts = alerts.filter(a => !(a.userId === userId && a.mint === mint));
      } else {
        position.amount   *= (1 - percent / 100);
        position.solSpent *= (1 - percent / 100);
      }
    }

    ud.stats.totalTrades++;
    saveData();

    const pnlStr = pnl >= 0 ? `🟢 +${pnl.toFixed(4)} SOL` : `🔴 ${pnl.toFixed(4)} SOL`;
    await bot.editMessageText(
      `✅ *SELL CONFIRMED (${percent}%)*\n\n` +
      `💰 Received: ${(result.outAmount / LAMPORTS_PER_SOL).toFixed(4)} SOL\n` +
      `${pnlStr}\n🔗 [Solscan](${result.explorerUrl})`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  } catch (e) {
    console.error('Sell error:', e.message);
    await bot.editMessageText(
      `❌ *Sell failed*\n\n${e.message}`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  }
}

// ─── PRICE ALERT MONITOR ─────────────────────────────────────
setInterval(async () => {
  if (!alerts.length) return;
  const fired = [];

  for (let i = 0; i < alerts.length; i++) {
    const a = alerts[i];
    try {
      const info = await getTokenInfo(a.mint);
      if (!info) continue;

      const triggered =
        (a.direction === 'above' && info.priceUsd >= a.targetPrice) ||
        (a.direction === 'below' && info.priceUsd <= a.targetPrice);

      if (triggered) {
        const emoji = a.type === 'tp' ? '🎯' : a.type === 'sl' ? '🛑' : '🔔';
        await send(a.chatId,
          `${emoji} *Alert triggered!*\n\n` +
          `${a.symbol} is now $${info.priceUsd.toFixed(8)}\n` +
          `Target was: $${a.targetPrice.toFixed(8)} (${a.direction})\n\n` +
          `[View on DexScreener](https://dexscreener.com/solana/${a.mint})`,
          { reply_markup: kb.main() }
        );
        fired.push(i);
      }
    } catch (e) { /* skip */ }
  }

  // Remove fired alerts (reverse to preserve indices)
  fired.reverse().forEach(i => alerts.splice(i, 1));
  if (fired.length) saveData();
}, 60_000); // check every 60s

// ─── COPY TRADE MONITOR ──────────────────────────────────────
const seenTxs = new Set();

setInterval(async () => {
  for (const [userId, ud] of users) {
    if (!ud.settings.copyWallets?.length || !ud.walletAddress) continue;

    for (const copyAddr of ud.settings.copyWallets) {
      try {
        const sigs = await connection.getSignaturesForAddress(new PublicKey(copyAddr), { limit: 5 });

        for (const sig of sigs) {
          if (seenTxs.has(sig.signature)) continue;
          seenTxs.add(sig.signature);

          const tx = await connection.getParsedTransaction(sig.signature, { maxSupportedTransactionVersion: 0 });
          if (!tx) continue;

          // Detect Jupiter swaps by checking log messages
          const logs = tx.meta?.logMessages || [];
          const isJupiter = logs.some(l => l.includes('JUP'));
          if (!isJupiter) continue;

          // Notify user
          const chatId = userId; // userId == chatId for private bots
          await send(chatId,
            `👥 *Copy Trade Detected*\n\n` +
            `Wallet \`${copyAddr.slice(0, 8)}...\` made a trade.\n` +
            `[View TX](https://solscan.io/tx/${sig.signature})\n\n` +
            `To copy this trade, scan the token address above.`
          );
        }
      } catch (e) { /* skip */ }
    }
  }
}, 30_000);

// ─── MESSAGE HANDLER ─────────────────────────────────────────
bot.on('message', async (msg) => {
  const userId = msg.from.id;
  const chatId = msg.chat.id;
  const text   = msg.text?.trim();
  if (!text) return;

  const ud = getUser(userId);

  // ── /cancel ──
  if (text === '/cancel') {
    waiting.delete(userId);
    return send(chatId, '❌ Cancelled', { reply_markup: kb.main() });
  }

  // ── /start with optional referral ──
  if (text.startsWith('/start')) {
    const parts   = text.split(' ');
    const refCode = parts[1] || null;
    if (!ud.onboarded) return showDisclaimer(chatId, refCode);
    const bal = await getSolBalance(ud.walletAddress);
    return send(chatId,
      `🚀 *RugBuster Pro v3.0*\n\n` +
      `💳 \`${ud.walletAddress}\`\n` +
      `💰 *${bal.toFixed(4)} SOL*\n\n` +
      `Paste a token address or use the menu.`,
      { reply_markup: kb.main() }
    );
  }

  // ── Handle waiting state ──
  const w = waiting.get(userId);
  if (w) {
    // ── Private key inputs ──
    if (w.type === 'key_for_buy' || w.type === 'key_for_sell' ||
        w.type === 'key_for_send' || w.type === 'key_for_swap') {
      waiting.delete(userId);
      let keypair;
      try {
        keypair = Keypair.fromSecretKey(bs58.decode(text));
      } catch (e) {
        return send(chatId, '❌ Invalid private key. Trade cancelled.', { reply_markup: kb.main() });
      }
      if (keypair.publicKey.toBase58() !== ud.walletAddress) {
        return send(chatId, '❌ Key does not match your wallet. Cancelled.', { reply_markup: kb.main() });
      }
      try { await bot.deleteMessage(chatId, msg.message_id); } catch (_) {}

      if (w.type === 'key_for_buy')       return executeBuy(chatId, userId, text, w.mint, w.amount, w.tp, w.sl);
      if (w.type === 'key_for_sell')      return executeSell(chatId, userId, text, w.mint, w.percent);
      if (w.type === 'key_for_send')      return handleSendSol(chatId, userId, text, w.toAddress, w.amount);
      if (w.type === 'key_for_swap')      return handleSwap(chatId, userId, text, w.inputMint, w.outputMint, w.amount);
      if (w.type === 'key_for_sendtoken') return handleSendToken(chatId, userId, text, w.mint, w.symbol, w.toAddress, w.amount);
    }

    // ── Text inputs ──
    if (w.type === 'scan') {
      waiting.delete(userId);
      if (!isMint(text)) return send(chatId, '❌ Invalid address');
      return handleTokenScan(chatId, text, ud);
    }

    if (w.type === 'buy_custom') {
      waiting.delete(userId);
      const amount = parseFloat(text);
      if (isNaN(amount) || amount <= 0) return send(chatId, '❌ Invalid amount');
      return promptForKey(chatId, userId, { type: 'key_for_buy', mint: w.mint, amount });
    }

    if (w.type === 'buy_pct') {
      waiting.delete(userId);
      const pctVal = parseFloat(text);
      if (isNaN(pctVal) || pctVal <= 0 || pctVal > 100) return send(chatId, '❌ Enter 1–100');
      const bal    = await getSolBalance(ud.walletAddress);
      const amount = parseFloat((bal * pctVal / 100 - 0.01).toFixed(4));
      if (amount <= 0) return send(chatId, '❌ Insufficient balance');
      return promptForKey(chatId, userId, { type: 'key_for_buy', mint: w.mint, amount });
    }

    if (w.type === 'buy_tpsl_amount') {
      const amount = parseFloat(text);
      if (isNaN(amount) || amount <= 0) return send(chatId, '❌ Invalid amount');
      waiting.set(userId, { ...w, type: 'buy_tpsl_tp', amount });
      return send(chatId, `🎯 Enter TP % (e.g. 100 for 2x) or 0 to skip:`);
    }

    if (w.type === 'buy_tpsl_tp') {
      waiting.set(userId, { ...w, type: 'buy_tpsl_sl', tp: parseFloat(text) || null });
      return send(chatId, '🛑 Enter SL % (e.g. 30) or 0 to skip:');
    }

    if (w.type === 'buy_tpsl_sl') {
      waiting.delete(userId);
      const sl = parseFloat(text) || null;
      return promptForKey(chatId, userId, { type: 'key_for_buy', mint: w.mint, amount: w.amount, tp: w.tp, sl });
    }

    if (w.type === 'sell_custom') {
      waiting.delete(userId);
      const p = parseFloat(text);
      if (isNaN(p) || p <= 0 || p > 100) return send(chatId, '❌ Enter 1–100');
      return promptForKey(chatId, userId, { type: 'key_for_sell', mint: w.mint, percent: p });
    }

    if (w.type === 'send_address') {
      if (!isAddr(text)) return send(chatId, '❌ Invalid address. Try again or /cancel');
      waiting.set(userId, { type: 'send_amount', toAddress: text });
      return send(chatId, `💸 How much SOL to send?\nExample: 0.5`);
    }

    if (w.type === 'send_amount') {
      waiting.delete(userId);
      const amount = parseFloat(text);
      if (isNaN(amount) || amount <= 0) return send(chatId, '❌ Invalid amount');
      const bal = await getSolBalance(ud.walletAddress);
      if (bal < amount + 0.001) return send(chatId, `❌ Insufficient balance (${bal.toFixed(4)} SOL)`);
      return promptForKey(chatId, userId, { type: 'key_for_send', toAddress: w.toAddress, amount });
    }

    // ── Swap: custom output token address typed ──
    if (w.type === 'swap_custom_output') {
      if (!isMint(text)) return send(chatId, '❌ Invalid token address. Try again or /cancel');
      const info = await getTokenInfo(text).catch(() => null);
      const toSym = info?.symbol || text.slice(0,8);
      waiting.set(userId, { type: 'swap_pick_amount', fromMint: w.fromMint, fromSym: w.fromSym, toMint: text, toSym });
      return send(chatId,
        `💱 *Swap ${w.fromSym} → ${toSym}*\n\nHow much? (% of ${w.fromSym} balance)`,
        { reply_markup: kb.swapAmounts(toSym) }
      );
    }

    // ── Swap: custom amount typed ──
    if (w.type === 'swap_custom_amount') {
      waiting.delete(userId);
      const amount = parseFloat(text);
      if (isNaN(amount) || amount <= 0) return send(chatId, '❌ Invalid amount. /cancel to abort');
      return promptForKey(chatId, userId, { type: 'key_for_swap', inputMint: w.fromMint, outputMint: w.toMint, amount });
    }

    // ── Send token: destination address typed ──
    if (w.type === 'sendtoken_address') {
      if (!isAddr(text)) return send(chatId, '❌ Invalid address. Try again or /cancel');
      waiting.set(userId, { ...w, type: 'sendtoken_amount', toAddress: text });
      return send(chatId, `💸 How much *${w.symbol}* to send? Enter amount:`);
    }

    if (w.type === 'sendtoken_amount') {
      waiting.delete(userId);
      const amount = parseFloat(text);
      if (isNaN(amount) || amount <= 0) return send(chatId, '❌ Invalid amount');
      return promptForKey(chatId, userId, { type: 'key_for_sendtoken', mint: w.mint, symbol: w.symbol, toAddress: w.toAddress, amount });
    }

    if (w.type === 'new_alert_mint') {
      if (!isMint(text)) return send(chatId, '❌ Invalid mint');
      const info = await getTokenInfo(text);
      if (!info) return send(chatId, '❌ Token not found');
      waiting.set(userId, { type: 'new_alert_price', mint: text, symbol: info.symbol, currentPrice: info.priceUsd });
      return send(chatId, `Current price: \`$${info.priceUsd.toFixed(8)}\`\n\nEnter target price (e.g. 0.0000025):`);
    }

    if (w.type === 'new_alert_price') {
      waiting.delete(userId);
      const targetPrice = parseFloat(text);
      if (isNaN(targetPrice) || targetPrice <= 0) return send(chatId, '❌ Invalid price');
      const direction = targetPrice > w.currentPrice ? 'above' : 'below';
      alerts.push({ userId, chatId, mint: w.mint, symbol: w.symbol, targetPrice, direction, type: 'manual' });
      saveData();
      return send(chatId,
        `🔔 Alert set!\n${w.symbol} ${direction} \`$${targetPrice}\``,
        { reply_markup: kb.main() }
      );
    }

    if (w.type === 'add_copy') {
      waiting.delete(userId);
      if (!isAddr(text)) return send(chatId, '❌ Invalid address');
      ud.settings.copyWallets = ud.settings.copyWallets || [];
      if (ud.settings.copyWallets.includes(text)) return send(chatId, '⚠️ Already tracking this wallet');
      if (ud.settings.copyWallets.length >= 3) return send(chatId, '❌ Max 3 copy wallets');
      ud.settings.copyWallets.push(text);
      saveData();
      return send(chatId, `✅ Now monitoring \`${text.slice(0, 8)}...\` for trades`, { reply_markup: kb.main() });
    }

    // Settings inputs
    const settingMap = {
      set_slippage: { key: 'slippageBps', label: 'Slippage', transform: v => Math.round(v * 100), validate: v => v >= 0.1 && v <= 50  },
      set_priority: { key: 'priorityFee', label: 'Priority fee', transform: v => v, validate: v => v >= 0                             },
      set_minscore: { key: 'minScore', label: 'Min score', transform: v => parseInt(v), validate: v => v >= 0 && v <= 100             },
      set_minliq:   { key: 'minLiqUsd', label: 'Min liquidity', transform: v => v, validate: v => v >= 0                              },
      set_tp:       { key: 'tpPercent', label: 'Take profit', transform: v => v <= 0 ? null : v, validate: () => true                  },
      set_sl:       { key: 'slPercent', label: 'Stop loss', transform: v => v <= 0 ? null : v, validate: () => true                    }
    };

    if (settingMap[w.type]) {
      waiting.delete(userId);
      const s   = settingMap[w.type];
      const val = parseFloat(text);
      if (isNaN(val) || !s.validate(val)) return send(chatId, `❌ Invalid value`);
      ud.settings[s.key] = s.transform(val);
      // If setting TP, now ask for SL
      if (w.type === 'set_tp') {
        waiting.set(userId, { type: 'set_sl' });
        return send(chatId, `✅ TP set to ${val > 0 ? val + '%' : 'OFF'}\n\nNow enter SL % (0 to disable):`);
      }
      saveData();
      return send(chatId, `✅ ${s.label} updated`, { reply_markup: kb.settings(ud.settings) });
    }

    return;
  }

  // ── Token address detection ──
  if (isMint(text)) {
    if (!ud.onboarded) return showDisclaimer(chatId);
    return handleTokenScan(chatId, text, ud);
  }
});

// ─── SEND SOL HANDLER ────────────────────────────────────────
async function handleSendSol(chatId, userId, privateKeyB58, toAddress, amount) {
  const msg = await send(chatId, `⏳ Sending ${amount} SOL...`);
  try {
    const result = await sendSol(privateKeyB58, toAddress, amount);
    await bot.editMessageText(
      `✅ *Sent ${amount} SOL*\n\nTo: \`${toAddress}\`\n🔗 [Solscan](${result.explorerUrl})`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  } catch (e) {
    await bot.editMessageText(`❌ Send failed\n\n${e.message}`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  }
}

// ─── SEND SPL TOKEN HANDLER ──────────────────────────────────
async function handleSendToken(chatId, userId, privateKeyB58, mint, symbol, toAddress, amount) {
  const ud  = getUser(userId);
  const msg = await send(chatId, `⏳ Sending ${amount} ${symbol}...`);
  try {
    // Use Jupiter swap route to send SPL token via transfer
    // For simplicity: swap the token to SOL first if needed, or direct transfer
    const keypair = Keypair.fromSecretKey(require('bs58').decode(privateKeyB58));
    const { Token, TOKEN_PROGRAM_ID } = require('@solana/spl-token');
    
    // Get source token account
    const fromTokenAccounts = await connection.getParsedTokenAccountsByOwner(
      keypair.publicKey,
      { mint: new PublicKey(mint) }
    );
    if (!fromTokenAccounts.value.length) throw new Error('No token account found');
    const fromATA = fromTokenAccounts.value[0].pubkey;
    const decimals = fromTokenAccounts.value[0].account.data.parsed.info.tokenAmount.decimals;
    const rawAmount = Math.floor(amount * Math.pow(10, decimals));

    // Get or create destination token account
    const destPubkey = new PublicKey(toAddress);
    const destTokenAccounts = await connection.getParsedTokenAccountsByOwner(
      destPubkey, { mint: new PublicKey(mint) }
    );

    let destATA;
    if (destTokenAccounts.value.length) {
      destATA = destTokenAccounts.value[0].pubkey;
    } else {
      // Create associated token account
      const { getAssociatedTokenAddress, createAssociatedTokenAccountInstruction } = require('@solana/spl-token');
      destATA = await getAssociatedTokenAddress(new PublicKey(mint), destPubkey);
    }

    // Build transfer instruction
    const { createTransferInstruction } = require('@solana/spl-token');
    const ix = createTransferInstruction(fromATA, destATA, keypair.publicKey, rawAmount);
    const tx = new Transaction().add(ix);
    const { blockhash } = await connection.getLatestBlockhash();
    tx.recentBlockhash = blockhash;
    tx.feePayer = keypair.publicKey;
    tx.sign(keypair);
    const sig = await connection.sendRawTransaction(tx.serialize());
    await connection.confirmTransaction(sig, 'confirmed');

    await bot.editMessageText(
      `✅ *Sent ${amount} ${symbol}*\n\nTo: \`${toAddress}\`\n🔗 [Solscan](https://solscan.io/tx/${sig})`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  } catch (e) {
    await bot.editMessageText(`❌ Send failed\n\n${e.message}`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  }
}

// ─── SWAP HANDLER ────────────────────────────────────────────
async function handleSwap(chatId, userId, privateKeyB58, inputMint, outputMint, uiAmount) {
  const ud  = getUser(userId);
  const msg = await send(chatId, `⏳ Swapping tokens... (10–30s)`);

  try {
    // Fetch decimals for input token
    let decimals = 9; // SOL default
    if (inputMint !== SOL_MINT) {
      const bal = await getTokenBalance(ud.walletAddress, inputMint);
      decimals  = bal.decimals;
    }
    const rawAmount = Math.floor(uiAmount * Math.pow(10, decimals));
    const result    = await jupiterSwap(privateKeyB58, inputMint, outputMint, rawAmount, ud.settings.slippageBps, ud.settings.priorityFee);

    await bot.editMessageText(
      `✅ *Swap Confirmed*\n\nReceived: ${result.outAmount.toLocaleString()} tokens\n🔗 [Solscan](${result.explorerUrl})`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  } catch (e) {
    await bot.editMessageText(`❌ Swap failed\n\n${e.message}`,
      { chat_id: chatId, message_id: msg.message_id, parse_mode: 'Markdown', reply_markup: kb.main() }
    );
  }
}

// ─── CALLBACK HANDLER ────────────────────────────────────────
bot.on('callback_query', async (query) => {
  const userId = query.from.id;
  const chatId = query.message.chat.id;
  const msgId  = query.message.message_id;
  const data   = query.data;
  const ud     = getUser(userId);

  await bot.answerCallbackQuery(query.id);

  try {
    // ── Onboarding ──
    if (data === 'confirm_onboard') return generateAndShowWallet(chatId, userId);
    if (data === 'cancel_onboard')  return edit(chatId, msgId, '❌ Send /start to try again.');

    // ── Guard ──
    if (!ud.onboarded) return send(chatId, 'Send /start first to set up your wallet.');

    // ── Main menu ──
    if (data === 'main_menu') {
      const bal = await getSolBalance(ud.walletAddress);
      return edit(chatId, msgId,
        `🚀 *RugBuster Pro*\n\n💰 Balance: *${bal.toFixed(4)} SOL*`,
        { reply_markup: kb.main() }
      );
    }

    // ── Wallet menu ──
    if (data === 'wallet_menu') {
      return edit(chatId, msgId, '💼 *Wallet*\nWhat would you like to do?',
        { reply_markup: kb.wallet() }
      );
    }

    // ── Balance & tokens ──
    if (data === 'balance') {
      const loadMsg = await edit(chatId, msgId, '📊 Loading your portfolio...');
      const [bal, tokens] = await Promise.all([
        getSolBalance(ud.walletAddress),
        getAllTokenBalances(ud.walletAddress)
      ]);

      // Build token lines with P&L from positions
      const tokenLines = [];
      for (const t of tokens.slice(0, 15)) {
        const pos = ud.positions.find(p => p.mint === t.mint);
        let pnlStr = '';
        if (pos && pos.entryPriceNative > 0) {
          const info = await getTokenInfo(t.mint).catch(() => null);
          if (info) {
            const pnlPct = ((info.priceNative - pos.entryPriceNative) / pos.entryPriceNative * 100);
            const pnlSol = (info.priceNative - pos.entryPriceNative) * (t.uiAmount || 0);
            const emoji  = pnlPct >= 0 ? '🟢' : '🔴';
            pnlStr = ` ${emoji} ${pnlPct >= 0 ? '+' : ''}${pnlPct.toFixed(1)}%`;
          }
        }
        const sym = pos?.symbol || t.mint.slice(0,6)+'...';
        tokenLines.push(`• *${sym}*: ${(t.uiAmount||0).toFixed(4)}${pnlStr}`);
      }

      // Build per-token action buttons
      const tokenBtns = ud.positions.slice(0, 8).map(p => ([
        { text: `💸 Sell ${p.symbol}`,      callback_data: `sellmenu_${p.mint}` },
        { text: `💱 Swap ${p.symbol}`,      callback_data: `swapfrom_${p.mint}` },
        { text: `📤 Send ${p.symbol}`,      callback_data: `sendtoken_${p.mint}` },
      ]));

      return edit(chatId, msgId,
        `💼 *Portfolio*\n` +
        `💳 \`${ud.walletAddress.slice(0,16)}...\`\n\n` +
        `◎ *SOL Balance:* ${bal.toFixed(4)} SOL\n` +
        `━━━━━━━━━━━━━━\n` +
        `🪙 *Your Tokens:*\n` +
        (tokenLines.length ? tokenLines.join('\n') : '_No tokens yet_') +
        `\n\n[🔗 View on Solscan](https://solscan.io/account/${ud.walletAddress})`,
        { reply_markup: { inline_keyboard: [
          ...tokenBtns,
          [{ text: '💱 Swap SOL',   callback_data: 'swapfrom_sol'  },
           { text: '📤 Send SOL',   callback_data: 'send_sol'       }],
          [{ text: '🔙 Back',       callback_data: 'wallet_menu'   }]
        ]}}
      );
    }

    // ── Receive ──
    if (data === 'receive') {
      const qr = await generateQR(ud.walletAddress);
      await bot.sendPhoto(chatId, qr, {
        caption: `📥 *Your Deposit Address*\n\n\`${ud.walletAddress}\`\n\nSend SOL or SPL tokens to this address.`,
        parse_mode: 'Markdown',
        reply_markup: kb.back('wallet_menu')
      });
      return;
    }

    // ── Send SOL ──
    if (data === 'send_sol') {
      waiting.set(userId, { type: 'send_address' });
      return edit(chatId, msgId, '📤 *Send SOL*\n\nEnter destination wallet address:');
    }

    // ── Swap menu — pick what you're swapping FROM ──
    if (data === 'swap_menu') {
      return edit(chatId, msgId,
        `💱 *Swap Tokens*\n\nWhat do you want to swap?`,
        { reply_markup: { inline_keyboard: [
          [{ text: '◎ Swap SOL',  callback_data: 'swapfrom_sol'  }],
          ...ud.positions.slice(0, 5).map(p => ([
            { text: `💱 Swap ${p.symbol}`, callback_data: `swapfrom_${p.mint}` }
          ])),
          [{ text: '🔙 Back', callback_data: 'wallet_menu' }]
        ]}}
      );
    }

    // ── Swap FROM chosen — pick output token ──
    if (data.startsWith('swapfrom_')) {
      const fromKey  = data.replace('swapfrom_', '');
      const fromMint = fromKey === 'sol' ? SOL_MINT : fromKey;
      const fromSym  = fromKey === 'sol' ? 'SOL'
        : (ud.positions.find(p => p.mint === fromKey)?.symbol || fromKey.slice(0,8));
      waiting.set(userId, { type: 'swap_pick_output', fromMint, fromSym });
      return edit(chatId, msgId,
        `💱 *Swap ${fromSym} → ?*\n\nChoose what to swap to:`,
        { reply_markup: kb.swapPicker(fromSym) }
      );
    }

    // ── Swap TO — known token chosen ──
    if (data.startsWith('swap_to_')) {
      const w = waiting.get(userId);
      if (!w || w.type !== 'swap_pick_output') return;
      const key    = data.replace('swap_to_', '');
      if (key === 'custom') {
        waiting.set(userId, { ...w, type: 'swap_custom_output' });
        return edit(chatId, msgId, `💱 Enter the token mint address to swap to:`);
      }
      const toToken = SWAP_TOKENS[key];
      if (!toToken) return;
      waiting.set(userId, { type: 'swap_pick_amount', fromMint: w.fromMint, fromSym: w.fromSym, toMint: toToken.mint, toSym: toToken.symbol });
      return edit(chatId, msgId,
        `💱 *Swap ${w.fromSym} → ${toToken.symbol}*\n\nHow much? (% of your ${w.fromSym} balance)`,
        { reply_markup: kb.swapAmounts(toToken.symbol) }
      );
    }

    // ── Swap amount % chosen ──
    if (data.startsWith('swapamt_')) {
      const w = waiting.get(userId);
      if (!w || w.type !== 'swap_pick_amount') return;
      const pctKey = data.replace('swapamt_', '');
      if (pctKey === 'custom') {
        waiting.set(userId, { ...w, type: 'swap_custom_amount' });
        return edit(chatId, msgId, `✏️ Enter exact amount of *${w.fromSym}* to swap:`);
      }
      const pctVal = parseInt(pctKey);
      let amount;
      if (w.fromMint === SOL_MINT) {
        const bal = await getSolBalance(ud.walletAddress);
        amount = parseFloat((bal * pctVal / 100 - 0.002).toFixed(6));
      } else {
        const tokens = await getAllTokenBalances(ud.walletAddress);
        const tok    = tokens.find(t => t.mint === w.fromMint);
        amount = (tok?.uiAmount || 0) * pctVal / 100;
      }
      if (!amount || amount <= 0) return edit(chatId, msgId, '❌ Insufficient balance', { reply_markup: kb.back('wallet_menu') });
      waiting.delete(userId);
      return promptForKey(chatId, userId, { type: 'key_for_swap', inputMint: w.fromMint, outputMint: w.toMint, amount });
    }

    // ── Send token ──
    if (data.startsWith('sendtoken_')) {
      const mint = data.replace('sendtoken_', '');
      const pos  = ud.positions.find(p => p.mint === mint);
      waiting.set(userId, { type: 'sendtoken_address', mint, symbol: pos?.symbol || 'token' });
      return edit(chatId, msgId, `📤 *Send ${pos?.symbol || 'Token'}*\n\nEnter destination Solana wallet address:`);
    }

    // ── Positions ──
    if (data === 'positions') {
      if (!ud.positions.length) return edit(chatId, msgId, '📊 No open positions', { reply_markup: kb.back() });

      const lines = await Promise.all(ud.positions.map(async p => {
        const info   = await getTokenInfo(p.mint);
        const change = p.entryPriceNative > 0
          ? (((info?.priceNative || 0) - p.entryPriceNative) / p.entryPriceNative * 100) : 0;
        const tpsl   = (p.tpPercent || p.slPercent)
          ? ` [TP:${p.tpPercent ?? '—'}% SL:${p.slPercent ?? '—'}%]` : '';
        return `*${p.symbol}*: ${pct(change)} | ${p.solSpent?.toFixed(3)} SOL${tpsl}`;
      }));

      const sellBtns = ud.positions.map(p => ([
        { text: `💸 Sell ${p.symbol}`, callback_data: `sellmenu_${p.mint}` }
      ]));

      return edit(chatId, msgId,
        `📊 *Open Positions (${ud.positions.length})*\n\n${lines.join('\n')}`,
        { reply_markup: { inline_keyboard: [...sellBtns, [{ text: '🔙 Menu', callback_data: 'main_menu' }]] } }
      );
    }

    // ── Sell menu ──
    if (data.startsWith('sellmenu_')) {
      const mint = data.replace('sellmenu_', '');
      const pos  = ud.positions.find(p => p.mint === mint);
      return edit(chatId, msgId,
        `💸 *Sell ${pos?.symbol || 'token'}*\nSelect percentage:`,
        { reply_markup: kb.sellMenu(mint, pos?.symbol, ud.settings.quickSellPercents) }
      );
    }

    // ── PnL ──
    if (data === 'pnl') {
      const s       = ud.stats;
      const winRate = s.totalTrades > 0 ? ((s.wins / s.totalTrades) * 100).toFixed(1) : '0.0';
      return edit(chatId, msgId,
        `📈 *Trading Stats*\n\n` +
        `📊 Trades: ${s.totalTrades}\n` +
        `💱 Volume: ${s.totalVolume.toFixed(3)} SOL\n` +
        `${s.totalPnL >= 0 ? '🟢' : '🔴'} Total PnL: ${s.totalPnL >= 0 ? '+' : ''}${s.totalPnL.toFixed(4)} SOL\n` +
        `🏆 Win Rate: ${winRate}%\n` +
        `🥇 Wins: ${s.wins} | Losses: ${s.totalTrades - s.wins}`,
        { reply_markup: kb.back() }
      );
    }

    // ── History ──
    if (data === 'history') {
      if (!ud.tradingHistory.length) return edit(chatId, msgId, '📋 No trade history yet', { reply_markup: kb.back() });
      const lines = ud.tradingHistory.slice(0, 15).map((t, i) =>
        `${i + 1}. *${t.symbol}* ${t.pnl >= 0 ? '🟢' : '🔴'} ${t.pnl >= 0 ? '+' : ''}${t.pnl?.toFixed(4)} SOL\n` +
        `   ${new Date(t.exitTime).toLocaleDateString()}`
      ).join('\n\n');
      return edit(chatId, msgId, `📋 *Trade History*\n\n${lines}`, { reply_markup: kb.back() });
    }

    // ── Scan prompt ──
    if (data === 'scan_prompt') {
      waiting.set(userId, { type: 'scan' });
      return edit(chatId, msgId, '🔍 Paste the token mint address:');
    }

    // ── Buy quick ──
    if (data.startsWith('buyprompt_')) {
      const parts  = data.split('_');
      const amount = parseFloat(parts[parts.length - 1]);
      const mint   = parts.slice(1, -1).join('_');
      return promptForKey(chatId, userId, { type: 'key_for_buy', mint, amount });
    }

    // ── Buy custom amount ──
    if (data.startsWith('buy_custom_')) {
      const mint = data.replace('buy_custom_', '');
      waiting.set(userId, { type: 'buy_custom', mint });
      return edit(chatId, msgId, '✏️ Enter SOL amount:\nExample: 0.25');
    }

    // ── Buy % of balance ──
    if (data.startsWith('buy_pct_')) {
      const mint = data.replace('buy_pct_', '');
      waiting.set(userId, { type: 'buy_pct', mint });
      return edit(chatId, msgId, '% Enter % of your SOL balance to use:\nExample: 50');
    }

    // ── Buy with TP/SL ──
    if (data.startsWith('buy_tpsl_')) {
      const mint = data.replace('buy_tpsl_', '');
      const bal  = await getSolBalance(ud.walletAddress);
      waiting.set(userId, { type: 'buy_tpsl_amount', mint });
      return edit(chatId, msgId,
        `🎯 *Buy with TP/SL*\n\nBalance: ${bal.toFixed(4)} SOL\n\nEnter SOL amount to buy:`
      );
    }

    // ── Sell quick ──
    if (data.startsWith('sellprompt_')) {
      const parts   = data.split('_');
      const percent = parseFloat(parts[parts.length - 1]);
      const mint    = parts.slice(1, -1).join('_');
      return promptForKey(chatId, userId, { type: 'key_for_sell', mint, percent });
    }

    // ── Sell custom ──
    if (data.startsWith('sell_custom_')) {
      const mint = data.replace('sell_custom_', '');
      waiting.set(userId, { type: 'sell_custom', mint });
      return edit(chatId, msgId, '✏️ Enter sell percentage (1–100):');
    }

    // ── Alerts ──
    if (data === 'alerts_menu') {
      const userAlerts = alerts.filter(a => a.userId === userId);
      return edit(chatId, msgId,
        `🔔 *Price Alerts (${userAlerts.length})*\n\nTap an alert to delete it.`,
        { reply_markup: kb.alertsMenu(userAlerts) }
      );
    }

    if (data === 'new_alert') {
      waiting.set(userId, { type: 'new_alert_mint' });
      return edit(chatId, msgId, '🔔 Enter the token mint address for your alert:');
    }

    if (data.startsWith('del_alert_')) {
      const idx        = parseInt(data.replace('del_alert_', ''));
      const userAlerts = alerts.filter(a => a.userId === userId);
      if (userAlerts[idx]) {
        const globalIdx = alerts.indexOf(userAlerts[idx]);
        if (globalIdx !== -1) alerts.splice(globalIdx, 1);
        saveData();
      }
      const remaining = alerts.filter(a => a.userId === userId);
      return edit(chatId, msgId,
        `🔔 *Price Alerts (${remaining.length})*\n\nAlert deleted.`,
        { reply_markup: kb.alertsMenu(remaining) }
      );
    }

    // ── Copy trade ──
    if (data === 'copy_menu') {
      const cw = ud.settings.copyWallets || [];
      return edit(chatId, msgId,
        `👥 *Copy Trading (${cw.length}/3)*\n\nYou'll be notified when tracked wallets trade.`,
        { reply_markup: kb.copyMenu(cw) }
      );
    }

    if (data === 'add_copy') {
      waiting.set(userId, { type: 'add_copy' });
      return edit(chatId, msgId, '👥 Enter the wallet address to copy:');
    }

    if (data.startsWith('del_copy_')) {
      const idx = parseInt(data.replace('del_copy_', ''));
      ud.settings.copyWallets?.splice(idx, 1);
      saveData();
      return edit(chatId, msgId,
        `👥 *Copy Trading (${ud.settings.copyWallets?.length}/3)*\n\nWallet removed.`,
        { reply_markup: kb.copyMenu(ud.settings.copyWallets || []) }
      );
    }

    // ── Settings ──
    if (data === 'settings') {
      return edit(chatId, msgId, '⚙️ *Settings*\nTap to change:', { reply_markup: kb.settings(ud.settings) });
    }

    const settingPrompts = {
      set_slippage: `📉 Slippage % (current: ${ud.settings.slippageBps / 100}%)\nExample: 1.5`,
      set_priority: `⚡ Priority fee in SOL (current: ${ud.settings.priorityFee})\nExample: 0.001`,
      set_minscore: `🎯 Min rug score 0–100 (current: ${ud.settings.minScore})`,
      set_minliq:   `💧 Min liquidity USD (current: ${ud.settings.minLiqUsd})\nExample: 10000`,
      set_tpsl:     `🎯 Enter TP % (e.g. 100 = 2x, 0 to disable):`
    };

    if (settingPrompts[data]) {
      waiting.set(userId, { type: data === 'set_tpsl' ? 'set_tp' : data });
      return edit(chatId, msgId, settingPrompts[data]);
    }

    // ── Referral ──
    if (data === 'referral') {
      const botInfo = await bot.getMe();
      const link    = `https://t.me/${botInfo.username}?start=${ud.referralCode}`;
      return edit(chatId, msgId,
        `🎁 *Referral Program*\n\n` +
        `Your code: \`${ud.referralCode}\`\n` +
        `Your link: ${link}\n\n` +
        `👥 Referred users: *${ud.referralCount || 0}*\n\n` +
        `Share your link — every user who signs up via your link is tracked.`,
        { reply_markup: kb.back() }
      );
    }

    // ── Help ──
    if (data === 'help') {
      return edit(chatId, msgId,
        `❓ *RugBuster Pro — Help*\n\n` +
        `*Trading:*\n` +
        `• Paste any token mint address to scan & buy\n` +
        `• Choose quick amounts or enter custom\n` +
        `• Buy with a % of your balance\n` +
        `• Set TP/SL auto-alerts when buying\n\n` +
        `*Wallet:*\n` +
        `• Send SOL to any address\n` +
        `• Receive — get QR code & address\n` +
        `• Swap any token pair via Jupiter\n` +
        `• View all token balances\n\n` +
        `*Alerts:*\n` +
        `• Set price alerts on any token\n` +
        `• Auto-set when buying with TP/SL\n\n` +
        `*Copy Trade:*\n` +
        `• Track up to 3 wallets\n` +
        `• Get notified when they trade\n\n` +
        `*Security:*\n` +
        `• Private key is used in memory only\n` +
        `• Never stored anywhere\n` +
        `• /cancel aborts any action\n\n` +
        `⚠️ Never share your private key with anyone.`,
        { reply_markup: kb.back() }
      );
    }

  } catch (e) {
    console.error('Callback error:', e);
    try { await send(chatId, `❌ Error: ${e.message}`); } catch (_) {}
  }
});

// ─── SHUTDOWN ────────────────────────────────────────────────
process.on('SIGINT',  () => { saveData(); process.exit(0); });
process.on('SIGTERM', () => { saveData(); process.exit(0); });

console.log('🚀 RugBuster Pro v3.0 — Starting...');
bot.getMe().then(me => {
  console.log(`✅ Bot: @${me.username}`);
  console.log(`📡 Listening for users...`);
});
