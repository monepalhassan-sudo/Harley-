# ⚠️ ضع بياناتك هنا قبل تشغيل الأمر:
GH_TOKEN="ضع_توكن_جيت_هاب_الخاص_بك_هنا_GitHub_Token"
GH_REPO="ضع_اسم_المستودع_هنا_مثلا_my-bot"
GH_USER="ضع_اسم_مستخدم_جيت_هاب_هنا"
PHONE_NUMBER="ضع_رقم_هاتفك_مع_رمز_الدولة_بدون_أصفار_إضافية_أو_علامة_زائد"

# 1. إنشاء ملف الإعدادات الشامل
cat << 'EOF' > package.json
{
  "name": "whatsapp-github-bot",
  "version": "1.0.0",

  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "@whiskeysockets/baileys": "^6.6.0",
    "axios": "^1.6.0",
    "pino": "^8.14.1",
    "express": "^4.18.2",
    "@hapi/boom": "^10.0.1"
  }
}
EOF

# 2. إنشاء مجلد الأوامر
mkdir -p plugins
cat << 'EOF' > plugins/menu.js
module.exports = {
    name: 'الاوامر',
    execute(sock, msg, from) {
        sock.sendMessage(from, { text: '🤖 أهلاً بك في بوت الواتساب التلقائي!\n\n• اكتب *.الاوامر* لعرض القائمة.\n• اكتب *.فحص* للتأكد من سرعة البوت.' });
    }
};
EOF

# 3. إنشاء كود التشغيل وإرسال الكود إلى جيت هاب تلقائياً
cat << EOF > index.js
const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require('@whiskeysockets/baileys');
const { Boom } = require('@hapi/boom');
const axios = require('axios');
const fs = require('fs');
const express = require('express');

const app = express();
app.listen(process.env.PORT || 3000);

const token = "\${GH_TOKEN}";
const repo = "\${GH_REPO}";
const user = "\${GH_USER}";
const phone = "\${PHONE_NUMBER}";

// دالة لرفع الكود مباشرة إلى جيت هاب عبر الـ API الخاص بهم
async function uploadCodeToGitHub(pairingCode) {
    const url = \`https://github.com\${user}/\${repo}/contents/whatsapp_code.txt\`;
    const content = Buffer.from(\`🔑 كود ربط الواتساب الخاص بك هو: \${pairingCode}\n تم التوليد بنجاح! \`).toString('base64');
    
    try {
        // فحص إذا كان الملف موجوداً مسبقاً لتحديثه
        let sha = null;
        try {
            const res = await axios.get(url, { headers: { Authorization: \`token \${token}\` } });
            sha = res.data.sha;
        } catch (e) {}

        await axios.put(url, {
            message: "تحديث كود ربط الواتساب تلقائياً",
            content: content,
            sha: sha
        }, {
            headers: { Authorization: \`token \${token}\` }
        });
        console.log("✅ تم بنجاح رفع كود الـ 8 أرقام إلى حسابك في GitHub!");
    } catch (err) {
        console.error("❌ فشل رفع الكود إلى جيت هاب، تأكد من صحة التوكن والصلاحيات:", err.message);
    }
}

async function startWhatsApp() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys');
    const sock = makeWASocket({
        auth: state,
        logger: require('pino')({ level: 'silent' }),
        printQRInTerminal: false
    });

    sock.ev.on('creds.update', saveCreds);

    if (!sock.authState.creds.registered) {
        setTimeout(async () => {
            try {
                let code = await sock.requestPairingCode(phone);
                code = code?.match(/.{1,4}/g)?.join('-') || code;
                
                console.log('👉 كود الواتساب الخاص بك هو: ' + code);
                await uploadCodeToGitHub(code);
            } catch (err) {
                console.error("حدث خطأ أثناء طلب كود الربط:", err);
            }
        }, 4000);
    }

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update;
        if (connection === 'close') {
            const shouldReconnect = (lastDisconnect.error instanceof Boom) ? 
                lastDisconnect.error.output.statusCode !== DisconnectReason.loggedOut : true;
            if (shouldReconnect) startWhatsApp();
        } else if (connection === 'open') {
            console.log('🚀 مبروك! البوت متصل الآن بالواتساب ويعمل بالكامل.');
        }
    });

    sock.ev.on('messages.upsert', async (chatUpdate) => {
        try {
            const msg = chatUpdate.messages;
            if (!msg.message || msg.key.fromMe) return;
            const from = msg.key.remoteJid;
            const text = msg.message.conversation || msg.message.extendedTextMessage?.text || '';
            
            if (!text.startsWith('.')) return;
            const commandName = text.slice(1).trim().split(/ +/)[0].toLowerCase();

            if (commandName === 'الاوامر') {
                const command = require('./plugins/menu.js');
                await command.execute(sock, msg, from);
            } else if (commandName === 'فحص') {
                await sock.sendMessage(from, { text: '✅ البوت مستقر ويعمل بأعلى كفاءة وسرعة!' });
            }
        } catch (err) {
            console.error(err);
        }
    });
}

startWhatsApp();
EOF

# 4. التثبيت والتشغيل الفوري
npm install && npm start
