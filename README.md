const {
    default: makeWASocket,
    useMultiFileAuthState
} = require('@whiskeysockets/baileys');
const fs = require('fs');
const express = require('express');
const app = express();

// ====== KEEPALIVE REPLIT ======
app.get('/', (req, res) => res.send('Nejire Bot Online 💙'));
app.listen(3000, () => console.log('🌐 KeepAlive ativo na porta 3000'));

// ====== FUNÇÕES DB ======
const DB_FILE = './dados.json';
function loadDB() {
    if (!fs.existsSync(DB_FILE)) {
        fs.writeFileSync(DB_FILE, JSON.stringify({ grupos: [], spams: [] }, null, 2));
    }
    return JSON.parse(fs.readFileSync(DB_FILE));
}
function saveDB(data) {
    fs.writeFileSync(DB_FILE, JSON.stringify(data, null, 2));
}

// ====== BOT ======
async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info');
    const sock = makeWASocket({ auth: state });

    sock.ev.on('creds.update', saveCreds);

    sock.ev.on('messages.upsert', async ({ messages }) => {
        const msg = messages[0];
        if (!msg.message || !msg.key.remoteJid) return;
        const from = msg.key.remoteJid;
        const isGroup = from.endsWith('@g.us');
        const sender = msg.key.participant || msg.key.remoteJid;
        const body = msg.message.conversation || msg.message.extendedTextMessage?.text || '';
        const args = body.trim().split(' ');
        const cmd = args[0]?.toLowerCase();

        const db = loadDB();

        // ====== COMANDOS ======

        // Adicionar grupo de divulgação
        if (cmd === '!addgrupo' && isGroup) {
            if (!db.grupos.includes(from)) {
                db.grupos.push(from);
                saveDB(db);
                await sock.sendMessage(from, { text: '✅ Grupo adicionado à lista de divulgação!' });
            } else {
                await sock.sendMessage(from, { text: '⚠️ Este grupo já está na lista!' });
            }
        }

        // Remover grupo de divulgação
        if (cmd === '!removegrupo' && isGroup) {
            db.grupos = db.grupos.filter(g => g !== from);
            saveDB(db);
            await sock.sendMessage(from, { text: '❌ Grupo removido da lista de divulgação!' });
        }

        // Adicionar spam pago
        if (cmd === '!spam' && args.length > 1) {
            const mensagem = body.slice(6);
            const expira = new Date();
            expira.setDate(expira.getDate() + 15);
            const id = db.spams.length ? db.spams[db.spams.length - 1].id + 1 : 1;
            db.spams.push({ id, cliente: sender, mensagem, expira: expira.toISOString() });
            saveDB(db);
            await sock.sendMessage(from, { text: `✅ Seu spam foi adicionado!\n🆔 ID: ${id}\n📅 Expira em: ${expira.toLocaleDateString()}` });
        }

        // Renovar spam
        if (cmd === '!renovar' && args[1]) {
            const id = parseInt(args[1]);
            const spam = db.spams.find(s => s.id === id && s.cliente === sender);
            if (spam) {
                const novaData = new Date();
                novaData.setDate(novaData.getDate() + 15);
                spam.expira = novaData.toISOString();
                saveDB(db);
                await sock.sendMessage(from, { text: `🔄 Seu spam foi renovado até ${novaData.toLocaleDateString()}` });
            } else {
                await sock.sendMessage(from, { text: '❌ Spam não encontrado ou não é seu!' });
            }
        }

        // Remover spam
        if (cmd === '!removerspam' && args[1]) {
            const id = parseInt(args[1]);
            db.spams = db.spams.filter(s => !(s.id === id && s.cliente === sender));
            saveDB(db);
            await sock.sendMessage(from, { text: '🗑️ Spam removido com sucesso!' });
        }
    });

    // ====== DIVULGAÇÃO AUTOMÁTICA ======
    setInterval(() => {
        const db = loadDB();
        const agora = new Date();
        const ativos = db.spams.filter(s => new Date(s.expira) > agora);

        db.grupos.forEach(grupo => {
            ativos.forEach(spam => {
                sock.sendMessage(grupo, { text: spam.mensagem });
            });
        });

        console.log(`📢 Divulgação enviada para ${ativos.length} anúncios ativos.`);
    }, 5 * 60 * 1000); // 5 minutos

    // ====== SPAM FIXO NEJIRE (a cada 3 horas) ======
    setInterval(() => {
        const db = loadDB();
        const spamFixo = "🌸 Oii! Eu sou a Nejire 💙, bot de RPG e divulgação no WhatsApp 📢💌\n✨ Tenho planos de divulgação e também ajudo a organizar seu grupo de RPG 🗒️💫\n💰 Para valores e pacotes: wa.me/5511962757751";
        db.grupos.forEach(grupo => {
            sock.sendMessage(grupo, { text: spamFixo });
        });
        console.log('💙 Spam fixo da Nejire enviado.');
    }, 3 * 60 * 60 * 1000); // 3 horas
}

startBot();
