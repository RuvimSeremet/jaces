"name": "jace-middleman-bot",
"version": "1.0.0",
"description": "Jace â€” Discord middleman bot (Auto MM + Real MM)",
"main": "src/index.js",
"type": "module",
"scripts": {
"start": "node src/index.js",
"register": "node src/register-commands.js"
},
"dependencies": {
"better-sqlite3": "^11.3.0",
"discord.js": "^14.16.3",
"dotenv": "^16.4.5",
"node-fetch": "^3.3.2"
}
}
# Discord
DISCORD_TOKEN=MTUxMjIyNjI3MjIyNTM5ODk3Nw.GcMF4b._ZFQc1GZlTUWUm5ut-_Mmk2HE1UYng9LQXeQOU
CLIENT_ID=1512226272225398977
GUILD_ID=1509795860828000267
# Roles & channels
MM_STAFF_ROLE_ID=1512231410084348035
TICKETS_CATEGORY_ID=1512231178227417180
LOG_CHANNEL_ID=1512231215460122755
# Auto-MM crypto receiving addresses (bot watches these)
BTC_ADDRESS=bc1q083msj73mmjjrenfcd9tvrfken62v6cy00k44u
LTC_ADDRESS=ltc1q27tzx3a0ped2lw6nhajcq7t6nn04efaln9gdv7
# Min confirmations before auto-release
BTC_MIN_CONF=2
LTC_MIN_CONF=4


import Database from 'better-sqlite3';
import { mkdirSync } from 'node:fs';
import { dirname } from 'node:path';
const DB_PATH = process.env.DB_PATH || './data/jace.db';
mkdirSync(dirname(DB_PATH), { recursive: true });
export const db = new Database(DB_PATH);
db.pragma('journal_mode = WAL');
db.exec(`
CREATE TABLE IF NOT EXISTS deals (
id INTEGER PRIMARY KEY AUTOINCREMENT,
channel_id TEXT UNIQUE NOT NULL,
guild_id TEXT NOT NULL,
mode TEXT NOT NULL, -- 'auto' | 'real'
kind TEXT NOT NULL, -- 'crypto' | 'virtual'
buyer_id TEXT NOT NULL,
seller_id TEXT NOT NULL,
amount TEXT, -- e.g. "0.005"
currency TEXT, -- 'BTC' | 'LTC' | 'ITEM'
item TEXT, -- description for virtual goods
deposit_address TEXT,
deposit_txid TEXT,
deposit_confs INTEGER DEFAULT 0,
status TEXT NOT NULL DEFAULT 'open', -- open | funded | released | cancelled | disputed
created_at INTEGER NOT NULL,
updated_at INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS confirmations (
deal_id INTEGER NOT NULL,
user_id TEXT NOT NULL,
kind TEXT NOT NULL, -- 'release' | 'cancel'
PRIMARY KEY (deal_id, user_id, kind)
);
`);
export const insertDeal = db.prepare(`
INSERT INTO deals (channel_id, guild_id, mode, kind, buyer_id, seller_id, amount, currency, item, deposit_address, created_at, updated_at)
VALUES (@channel_id, @guild_id, @mode, @kind, @buyer_id, @seller_id, @amount, @currency, @item, @deposit_address, @now, @now)
`);
export const getDealByChannel = db.prepare(`SELECT * FROM deals WHERE channel_id = ?`);
export const updateDealStatus = db.prepare(`UPDATE deals SET status = ?, updated_at = ? WHERE id = ?`);
export const updateDeposit = db.prepare(`UPDATE deals SET deposit_txid = ?, deposit_confs = ?, status = ?, updated_at = ? WHERE id = ?`);
export const listOpenCryptoDeals = db.prepare(`SELECT * FROM deals WHERE kind='crypto' AND status IN ('open','funded')`);
export const addConfirmation = db.prepare(`INSERT OR IGNORE INTO confirmations (deal_id, user_id, kind) VALUES (?, ?, ?)`);
export const countConfirmations = db.prepare(`SELECT COUNT(*) AS n FROM confirmations WHERE deal_id = ? AND kind = ?`);
export const clearConfirmations = db.prepare(`DELETE FROM confirmations WHERE deal_id = ?`);

// Lightweight on-chain watcher using public Blockstream (BTC) and Blockcypher (LTC) APIs.
// No API key required. Polled periodically by index.js.
import fetch from 'node-fetch';
export async function getBtcAddressTxs(address) {
const r = await fetch(`https://blockstream.info/api/address/${address}/txs`);
if (!r.ok) throw new Error(`BTC API ${r.status}`);
return r.json();
}
export async function getBtcTipHeight() {
const r = await fetch('https://blockstream.info/api/blocks/tip/height');
if (!r.ok) throw new Error(`BTC tip ${r.status}`);
return parseInt(await r.text(), 10);
}
export async function getLtcAddress(address) {
const r = await fetch(`https://api.blockcypher.com/v1/ltc/main/addrs/${address}/full?limit=10`);
if (!r.ok) throw new Error(`LTC API ${r.status}`);
return r.json();
}
// Returns { txid, amountBtc, confirmations } for first tx paying >= minAmountBtc, or null.
export async function findBtcPayment(address, minAmountBtc) {
const [txs, tip] = await Promise.all([getBtcAddressTxs(address), getBtcTipHeight()]);
for (const tx of txs) {
const paid = tx.vout
.filter(v => v.scriptpubkey_address === address)
.reduce((s, v) => s + v.value, 0) / 1e8;
if (paid + 1e-9 >= minAmountBtc) {
const confirmations = tx.status.confirmed ? (tip - tx.status.block_height + 1) : 0;
return { txid: tx.txid, amountBtc: paid, confirmations };
}
}
return null;
}
export async function findLtcPayment(address, minAmountLtc) {
const data = await getLtcAddress(address);
for (const tx of (data.txs || [])) {
const paid = (tx.outputs || [])
.filter(o => (o.addresses || []).includes(address))
.reduce((s, o) => s + o.value, 0) / 1e8;
if (paid + 1e-9 >= minAmountLtc) {
return { txid: tx.hash, amountLtc: paid, confirmations: tx.confirmations || 0 };
}
}
return null;
}

import 'dotenv/config';
import { REST, Routes, SlashCommandBuilder, PermissionFlagsBits } from 'discord.js';
const commands = [
new SlashCommandBuilder()
.setName('deal')
.setDescription('Open a new middleman deal ticket')
.addUserOption(o => o.setName('partner').setDescription('The other trader').setRequired(true))
.addStringOption(o => o.setName('mode').setDescription('Auto or Real MM').setRequired(true)
.addChoices({ name: 'Auto MM (crypto, automated)', value: 'auto' },
{ name: 'Real MM (human middleman)', value: 'real' }))
.addStringOption(o => o.setName('kind').setDescription('What is being traded').setRequired(true)
.addChoices({ name: 'Crypto', value: 'crypto' },
{ name: 'Virtual goods / items', value: 'virtual' }))
.addStringOption(o => o.setName('amount').setDescription('Amount (e.g. 0.005)').setRequired(false))
.addStringOption(o => o.setName('currency').setDescription('BTC, LTC, or item label').setRequired(false))
.addStringOption(o => o.setName('item').setDescription('Description of item(s) being traded').setRequired(false)),
new SlashCommandBuilder()
.setName('release')
.setDescription('(Buyer) Confirm goods received â€” release the deal')
.setDefaultMemberPermissions(PermissionFlagsBits.SendMessages),
new SlashCommandBuilder()
.setName('cancel')
.setDescription('Cancel the deal (both parties must confirm)'),
new SlashCommandBuilder()
.setName('dispute')
.setDescription('Open a dispute and ping staff'),
new SlashCommandBuilder()
.setName('status')
.setDescription('Show this deal\'s current status'),
new SlashCommandBuilder()
.setName('close')
.setDescription('(Staff) Close & archive this deal ticket')
.setDefaultMemberPermissions(PermissionFlagsBits.ManageChannels),
].map(c => c.toJSON());
const rest = new REST({ version: '10' }).setToken(process.env.DISCORD_TOKEN);
const route = process.env.GUILD_ID
? Routes.applicationGuildCommands(process.env.CLIENT_ID, process.env.GUILD_ID)
: Routes.applicationCommands(process.env.CLIENT_ID);
await rest.put(route, { body: commands });
console.log(`Registered ${commands.length} commands ${process.env.GUILD_ID ? 'to guild' : 'globally'}.`);


# Jace â€” Discord Middleman Bot
A real middleman/escrow bot for Discord. Two modes:
- **Auto MM** â€” bot watches a BTC/LTC address on-chain and auto-marks the deal "funded" once the agreed amount lands with N confirmations. For virtual goods, buyer confirms with `/release`.
- **Real MM** â€” bot opens a private ticket and pings your human middleman staff role.
## Setup
```bash
cd bot
npm install
cp .env.example .env # fill in tokens, role ids, BTC/LTC addresses
npm run register # register slash commands (per-guild = instant)
npm start
```
### Discord setup
1. https://discord.com/developers/applications â†’ New Application â†’ Bot. Copy the token.
2. Enable **Message Content Intent**.
3. Invite with scopes `bot applications.commands` and perms: Manage Channels, Send Messages, Read Message History, Manage Messages, Mention Everyone.
4. In your server create a category for tickets and a staff role, put the ids in `.env`.
### Crypto setup
Put your own receiving BTC / LTC address in `.env`. The bot polls the public Blockstream (BTC) and Blockcypher (LTC) APIs once a minute â€” no API key needed. **You** control the wallet; the bot only watches it.
> Limitation: a single receiving address per coin means concurrent auto-deals must use **different amounts** so the bot can tell deposits apart. For per-deal addresses use a wallet that supports HD-derivation (out of scope here).
## Commands
| Command | Who | What |
|---|---|---|
| `/deal partner:@user mode:auto|real kind:crypto|virtual amount currency item` | Anyone | Opens a private ticket |
| `/release` | Buyer | Releases the deal (funds/goods go to seller) |
| `/cancel` | Buyer or Seller | Both must run it to cancel |
| `/dispute` | Anyone | Pings staff, marks deal disputed |
| `/status` | Anyone | Show deal info |
| `/close` | Staff | Delete the ticket |
## Hosting
Runs anywhere Node 18+ runs: your PC, a VPS, Railway, Fly.io, a Raspberry Pi. `data/jace.db` is a SQLite file â€” back it up.
## Security notes
- This bot does **not** custody crypto. Funds go to **your** address; the bot only signals "the buyer paid".
- Always read pinned rules before trading. Auto-MM cannot recover wrong-amount or wrong-coin deposits.
- For real-money trades enable 2FA on the bot account and lock down the staff role.


# Jace â€” Discord Middleman Bot
A real middleman/escrow bot for Discord. Two modes:
- **Auto MM** â€” bot watches a BTC/LTC address on-chain and auto-marks the deal "funded" once the agreed amount lands with N confirmations. For virtual goods, buyer confirms with `/release`.
- **Real MM** â€” bot opens a private ticket and pings your human middleman staff role.
## Setup
```bash
cd bot
npm install
cp .env.example .env # fill in tokens, role ids, BTC/LTC addresses
npm run register # register slash commands (per-guild = instant)
npm start
```
### Discord setup
1. https://discord.com/developers/applications â†’ New Application â†’ Bot. Copy the token.
2. Enable **Message Content Intent**.
3. Invite with scopes `bot applications.commands` and perms: Manage Channels, Send Messages, Read Message History, Manage Messages, Mention Everyone.
4. In your server create a category for tickets and a staff role, put the ids in `.env`.
### Crypto setup
Put your own receiving BTC / LTC address in `.env`. The bot polls the public Blockstream (BTC) and Blockcypher (LTC) APIs once a minute â€” no API key needed. **You** control the wallet; the bot only watches it.
> Limitation: a single receiving address per coin means concurrent auto-deals must use **different amounts** so the bot can tell deposits apart. For per-deal addresses use a wallet that supports HD-derivation (out of scope here).
## Commands
| Command | Who | What |
|---|---|---|
| `/deal partner:@user mode:auto|real kind:crypto|virtual amount currency item` | Anyone | Opens a private ticket |
| `/release` | Buyer | Releases the deal (funds/goods go to seller) |
| `/cancel` | Buyer or Seller | Both must run it to cancel |
| `/dispute` | Anyone | Pings staff, marks deal disputed |
| `/status` | Anyone | Show deal info |
| `/close` | Staff | Delete the ticket |
## Hosting
Runs anywhere Node 18+ runs: your PC, a VPS, Railway, Fly.io, a Raspberry Pi. `data/jace.db` is a SQLite file â€” back it up.
## Security notes
- This bot does **not** custody crypto. Funds go to **your** address; the bot only signals "the buyer paid".
- Always read pinned rules before trading. Auto-MM cannot recover wrong-amount or wrong-coin deposits.
- For real-money trades enable 2FA on the bot account and lock down the staff role.

bot/src/index.js
import 'dotenv/config';
import {
Client, GatewayIntentBits, Partials, ChannelType, PermissionFlagsBits,
EmbedBuilder,
} from 'discord.js';
import {
insertDeal, getDealByChannel, updateDealStatus, updateDeposit,
listOpenCryptoDeals, addConfirmation, countConfirmations, clearConfirmations,
} from './db.js';
import { findBtcPayment, findLtcPayment } from './crypto.js';
const {
DISCORD_TOKEN, MM_STAFF_ROLE_ID, TICKETS_CATEGORY_ID, LOG_CHANNEL_ID,
BTC_ADDRESS, LTC_ADDRESS,
BTC_MIN_CONF = '2', LTC_MIN_CONF = '4',
} = process.env;
const client = new Client({
intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages, GatewayIntentBits.MessageContent],
partials: [Partials.Channel],
});
const log = async (guild, content) => {
if (!LOG_CHANNEL_ID) return;
const ch = await guild.channels.fetch(LOG_CHANNEL_ID).catch(() => null);
if (ch) ch.send(content).catch(() => {});
};
client.once('ready', () => {
console.log(`Jace online as ${client.user.tag}`);
if (BTC_ADDRESS || LTC_ADDRESS) setInterval(pollCryptoDeposits, 60_000);
});
// --- Slash commands ---------------------------------------------------------
client.on('interactionCreate', async (i) => {
if (!i.isChatInputCommand()) return;
try {
if (i.commandName === 'deal') return openDeal(i);
if (i.commandName === 'release') return confirmRelease(i);
if (i.commandName === 'cancel') return confirmCancel(i);
if (i.commandName === 'dispute') return openDispute(i);
if (i.commandName === 'status') return showStatus(i);
if (i.commandName === 'close') return closeTicket(i);
} catch (e) {
console.error(e);
if (i.deferred || i.replied) i.followUp({ content: `Error: ${e.message}`, ephemeral: true });
else i.reply({ content: `Error: ${e.message}`, ephemeral: true });
}
});
async function openDeal(i) {
const partner = i.options.getUser('partner', true);
const mode = i.options.getString('mode', true); // auto | real
const kind = i.options.getString('kind', true); // crypto | virtual
const amount = i.options.getString('amount') || null;
const currency = (i.options.getString('currency') || '').toUpperCase() || null;
const item = i.options.getString('item') || null;
if (partner.id === i.user.id) return i.reply({ content: 'Pick someone other than yourself.', ephemeral: true });
if (mode === 'auto' && kind === 'crypto') {
if (!amount || !currency || !['BTC', 'LTC'].includes(currency))
return i.reply({ content: 'Auto crypto deals need `amount` and `currency` (BTC or LTC).', ephemeral: true });
if (currency === 'BTC' && !BTC_ADDRESS) return i.reply({ content: 'BTC auto-MM is not configured.', ephemeral: true });
if (currency === 'LTC' && !LTC_ADDRESS) return i.reply({ content: 'LTC auto-MM is not configured.', ephemeral: true });
}
await i.deferReply({ ephemeral: true });
// Buyer is whoever opened the deal; seller is the partner. (User can swap later in chat â€” keep simple.)
const buyer = i.user;
const seller = partner;
const perms = [
{ id: i.guild.roles.everyone.id, deny: [PermissionFlagsBits.ViewChannel] },
{ id: buyer.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory] },
{ id: seller.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory] },
{ id: client.user.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ManageChannels] },
];
if (MM_STAFF_ROLE_ID) perms.push({
id: MM_STAFF_ROLE_ID,
allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory, PermissionFlagsBits.ManageMessages],
});
const channel = await i.guild.channels.create({
name: `deal-${buyer.username}-${seller.username}`.toLowerCase().slice(0, 90),
type: ChannelType.GuildText,
parent: TICKETS_CATEGORY_ID || undefined,
permissionOverwrites: perms,
topic: `Middleman deal between ${buyer.tag} (buyer) and ${seller.tag} (seller) â€” ${mode.toUpperCase()} MM`,
});
const depositAddress = mode === 'auto' && kind === 'crypto'
? (currency === 'BTC' ? BTC_ADDRESS : LTC_ADDRESS)
: null;
insertDeal.run({
channel_id: channel.id, guild_id: i.guild.id,
mode, kind, buyer_id: buyer.id, seller_id: seller.id,
amount, currency, item, deposit_address: depositAddress,
now: Date.now(),
});
const embed = new EmbedBuilder()
.setTitle(`Jace Middleman â€” ${mode === 'auto' ? 'Auto MM' : 'Real MM'}`)
.setColor(mode === 'auto' ? 0x22c55e : 0x3b82f6)
.setDescription([
`**Buyer:** <@${buyer.id}>`,
`**Seller:** <@${seller.id}>`,
`**Kind:** ${kind}`,
amount ? `**Amount:** ${amount} ${currency || ''}` : null,
item ? `**Item:** ${item}` : null,
].filter(Boolean).join('\n'))
.setFooter({ text: 'Read pinned rules. /release when buyer confirms goods. /dispute pings staff.' });
if (mode === 'auto' && kind === 'crypto') {
embed.addFields({
name: `Send exactly ${amount} ${currency} to:`,
value: `\`${depositAddress}\`\nBot polls the blockchain every ~60s. Status will update here once detected and after **${currency === 'BTC' ? BTC_MIN_CONF : LTC_MIN_CONF}** confirmations.`,
});
} else if (mode === 'real') {
embed.addFields({ name: 'Real Middleman', value: MM_STAFF_ROLE_ID ? `Pinging <@&${MM_STAFF_ROLE_ID}> â€” a human MM will handle this.` : 'Configure MM_STAFF_ROLE_ID to ping staff.' });
} else {
embed.addFields({ name: 'Auto MM â€” virtual goods', value: 'Seller: hand over the item. Buyer: run `/release` only after you verify receipt. `/dispute` to escalate.' });
}
const msg = await channel.send({
content: `<@${buyer.id}> <@${seller.id}>` + (mode === 'real' && MM_STAFF_ROLE_ID ? ` <@&${MM_STAFF_ROLE_ID}>` : ''),
embeds: [embed],
});
await msg.pin().catch(() => {});
await log(i.guild, `ðŸ†• Deal opened ${channel} â€” ${mode}/${kind} â€” buyer <@${buyer.id}> seller <@${seller.id}>`);
i.editReply({ content: `Deal ticket created: ${channel}` });
}
async function getDealOrFail(i) {
const deal = getDealByChannel.get(i.channelId);
if (!deal) { await i.reply({ content: 'This is not a deal channel.', ephemeral: true }); return null; }
return deal;
}
async function confirmRelease(i) {
const deal = await getDealOrFail(i); if (!deal) return;
if (i.user.id !== deal.buyer_id) return i.reply({ content: 'Only the **buyer** can /release.', ephemeral: true });
if (deal.status === 'released') return i.reply({ content: 'Already released.', ephemeral: true });
if (deal.kind === 'crypto' && deal.mode === 'auto' && deal.status !== 'funded')
return i.reply({ content: 'Deposit not confirmed on-chain yet. Wait for the funded notice.', ephemeral: true });
updateDealStatus.run('released', Date.now(), deal.id);
clearConfirmations.run(deal.id);
await i.reply(`âœ… **Released by buyer <@${deal.buyer_id}>.** Seller <@${deal.seller_id}> â€” funds/goods are yours. Staff can /close.`);
await log(i.guild, `âœ… Released ${i.channel} â€” seller <@${deal.seller_id}>`);
}
async function confirmCancel(i) {
const deal = await getDealOrFail(i); if (!deal) return;
if (![deal.buyer_id, deal.seller_id].includes(i.user.id))
return i.reply({ content: 'Only the buyer or seller can /cancel.', ephemeral: true });
if (deal.status === 'released') return i.reply({ content: 'Deal already released â€” open a /dispute instead.', ephemeral: true });
addConfirmation.run(deal.id, i.user.id, 'cancel');
const { n } = countConfirmations.get(deal.id, 'cancel');
if (n >= 2) {
updateDealStatus.run('cancelled', Date.now(), deal.id);
clearConfirmations.run(deal.id);
await i.reply('ðŸ›‘ Both parties confirmed â€” deal cancelled. Staff can /close.');
await log(i.guild, `ðŸ›‘ Cancelled ${i.channel}`);
} else {
await i.reply(`Cancel requested by <@${i.user.id}>. Waiting for the other party to /cancel as well.`);
}
}
async function openDispute(i) {
const deal = await getDealOrFail(i); if (!deal) return;
updateDealStatus.run('disputed', Date.now(), deal.id);
const ping = MM_STAFF_ROLE_ID ? `<@&${MM_STAFF_ROLE_ID}>` : '**Staff**';
await i.reply(`âš ï¸ Dispute opened by <@${i.user.id}>. ${ping} please review.`);
await log(i.guild, `âš ï¸ DISPUTE ${i.channel} by <@${i.user.id}>`);
}
async function showStatus(i) {
const deal = await getDealOrFail(i); if (!deal) return;
const e = new EmbedBuilder()
.setTitle(`Deal #${deal.id} â€” ${deal.status.toUpperCase()}`)
.addFields(
{ name: 'Mode', value: deal.mode, inline: true },
{ name: 'Kind', value: deal.kind, inline: true },
{ name: 'Buyer', value: `<@${deal.buyer_id}>`, inline: true },
{ name: 'Seller', value: `<@${deal.seller_id}>`, inline: true },
...(deal.amount ? [{ name: 'Amount', value: `${deal.amount} ${deal.currency || ''}`, inline: true }] : []),
...(deal.deposit_address ? [{ name: 'Deposit address', value: `\`${deal.deposit_address}\`` }] : []),
...(deal.deposit_txid ? [{ name: 'Deposit txid', value: `\`${deal.deposit_txid}\` (${deal.deposit_confs} confs)` }] : []),
);
await i.reply({ embeds: [e], ephemeral: true });
}
async function closeTicket(i) {
const deal = await getDealOrFail(i); if (!deal) return;
if (MM_STAFF_ROLE_ID && !i.member.roles.cache.has(MM_STAFF_ROLE_ID) && !i.member.permissions.has(PermissionFlagsBits.ManageChannels))
return i.reply({ content: 'Only staff can /close.', ephemeral: true });
await i.reply('Closing this ticket in 5sâ€¦');
setTimeout(() => i.channel.delete().catch(() => {}), 5000);
}
// --- Auto-MM crypto poller --------------------------------------------------
async function pollCryptoDeposits() {
const deals = listOpenCryptoDeals.all().filter(d => d.mode === 'auto');
for (const deal of deals) {
try {
const min = parseFloat(deal.amount);
if (!isFinite(min) || min <= 0) continue;
let payment = null;
if (deal.currency === 'BTC') payment = await findBtcPayment(deal.deposit_address, min);
else if (deal.currency === 'LTC') payment = await findLtcPayment(deal.deposit_address, min);
if (!payment) continue;
const needed = deal.currency === 'BTC' ? parseInt(BTC_MIN_CONF, 10) : parseInt(LTC_MIN_CONF, 10);
const newStatus = payment.confirmations >= needed ? 'funded' : 'open';
if (payment.txid !== deal.deposit_txid || payment.confirmations !== deal.deposit_confs || newStatus !== deal.status) {
updateDeposit.run(payment.txid, payment.confirmations, newStatus, Date.now(), deal.id);
const channel = await client.channels.fetch(deal.channel_id).catch(() => null);
if (!channel) continue;
if (newStatus === 'funded' && deal.status !== 'funded') {
channel.send(`ðŸ’° **Deposit confirmed** â€” \`${payment.txid}\` (${payment.confirmations} confs).\nSeller <@${deal.seller_id}>: deliver goods. Buyer <@${deal.buyer_id}>: \`/release\` once received.`);
} else if (!deal.deposit_txid) {
channel.send(`ðŸ‘€ Detected incoming tx \`${payment.txid}\` â€” waiting for ${needed} confirmations (now ${payment.confirmations}).`);
}
}
} catch (e) {
console.error('poll error', deal.id, e.message);
}
}
}
client.login(DISCORD_TOKEN);
