const { Telegraf, Markup } = require('telegraf');
const fs = require('fs');
const path = require('path');
require('dotenv').config();
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Bot Running');
});

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on ${PORT}`);
});
// Ensure Environment Variables Exist
if (!process.env.BOT_TOKEN || !process.env.ADMIN_ID) {
  console.error('ERROR: BOT_TOKEN and ADMIN_ID are required in environment variables (.env)');
  process.exit(1);
}

const BOT_TOKEN = process.env.BOT_TOKEN;
const ADMIN_ID = parseInt(process.env.ADMIN_ID, 10);

// --- FILE PATHS FOR STORAGE ---
const USERS_FILE = path.join(__dirname, 'users.json');
const SETTINGS_FILE = path.join(__dirname, 'settings.json');
const BUTTONS_FILE = path.join(__dirname, 'buttons.json');
const PAYMENTS_FILE = path.join(__dirname, 'payments.json');
const BANNED_FILE = path.join(__dirname, 'banned.json');

// --- DATABASE HELPERS ---
function readJSON(filePath, defaultValue = []) {
  try {
    if (!fs.existsSync(filePath)) {
      fs.writeFileSync(filePath, JSON.stringify(defaultValue, null, 2), 'utf8');
      return defaultValue;
    }
    const data = fs.readFileSync(filePath, 'utf8');
    return JSON.parse(data);
  } catch (err) {
    console.error(`Error reading ${filePath}:`, err);
    return defaultValue;
  }
}

function writeJSON(filePath, data) {
  try {
    fs.writeFileSync(filePath, JSON.stringify(data, null, 2), 'utf8');
    return true;
  } catch (err) {
    console.error(`Error writing ${filePath}:`, err);
    return false;
  }
}

// Initialize default assets inside files if missing
function initStorage() {
  readJSON(USERS_FILE, []);
  readJSON(BANNED_FILE, []);
  readJSON(PAYMENTS_FILE, []);
  readJSON(SETTINGS_FILE, {
    startImage: '',
    startCaption: '👋 <b>Welcome to our Premium Bot!</b>\n\nUse the menu buttons below to interact, or register for premium services.',
    upiId: 'example@upi',
    qrImage: '',
premiumFiles: []
  });
  readJSON(BUTTONS_FILE, [
    { id: 'btn_def_1', text: '📢 Channel', type: 'url', value: 'https://t.me/Telegram' },
    { id: 'btn_def_2', text: '💬 Support', type: 'url', value: 'https://t.me/Telegram' }
  ]);
}

initStorage();

// --- IN-MEMORY STATE MACHINE FOR SCENARIO PROCESSING ---
const states = new Map();

// --- KEYBOARD GENERATORS ---
function getStartKeyboard(userId) {
  const buttons = readJSON(BUTTONS_FILE, []);
  const keyboard = [];

  let row = [];
  buttons.forEach((btn) => {
    const item = btn.type === 'url'
      ? Markup.button.url(btn.text, btn.value)
      : Markup.button.callback(btn.text, btn.value);
    row.push(item);
    if (row.length === 2) {
      keyboard.push(row);
      row = [];
    }
  });
  if (row.length > 0) {
    keyboard.push(row);
  }

  // Persistent Get Premium Button
  keyboard.push([Markup.button.callback('💎 Get Premium', 'get_premium')]);

  // Append Admin Console Shortcut if Admin
  if (userId === ADMIN_ID) {
    keyboard.push([Markup.button.callback('⚙️ Admin Panel', 'admin_panel')]);
  }

  return Markup.inlineKeyboard(keyboard);
}

function getAdminKeyboard() {
  return Markup.inlineKeyboard([
    [Markup.button.callback('🖼️ Change Start Image', 'adm_change_img'), Markup.button.callback('✍️ Change Start Caption', 'adm_change_cap')],
   [Markup.button.callback('💳 Payment Settings', 'adm_payment_settings'),
Markup.button.callback('📂 Manage Files', 'adm_manage_files')],
    [Markup.button.callback('🔘 Manage Buttons', 'adm_manage_btns'), Markup.button.callback('📈 View Payments', 'adm_view_payments')],
    [Markup.button.callback('📣 Broadcast Message', 'adm_broadcast_menu'), Markup.button.callback('📊 View Stats', 'adm_view_stats')],
    [Markup.button.callback('🚫 Ban User', 'adm_ban_user'), Markup.button.callback('✅ Unban User', 'adm_unban_user')],
    [Markup.button.callback('❌ Close Panel', 'adm_close')]
  ]);
}

function getBroadcastTypeKeyboard() {
  return Markup.inlineKeyboard([
    [Markup.button.callback('📝 Text Broadcast', 'bc_text'), Markup.button.callback('🖼️ Photo Broadcast', 'bc_photo')],
    [Markup.button.callback('🔗 Text + Button', 'bc_text_btn'), Markup.button.callback('🖼️🔗 Photo + Button', 'bc_photo_btn')],
    [Markup.button.callback('⬅️ Back to Control Panel', 'adm_back')]
  ]);
}

// --- INITIALIZE BOT ---
const bot = new Telegraf(BOT_TOKEN);

// --- BAN CHECK & SAVE USER MIDDLEWARE ---
bot.use(async (ctx, next) => {
  if (!ctx.from) return next();

  const userId = ctx.from.id;
  const banned = readJSON(BANNED_FILE, []);

  if (banned.includes(userId)) {
    if (ctx.callbackQuery) {
      return ctx.answerCbQuery('🚫 You are banned from using this bot.', { show_alert: true });
    }
    return ctx.reply('🚫 You are banned from using this bot.');
  }

  // Register New Users automatically
  const users = readJSON(USERS_FILE, []);
  const exists = users.some(u => u.id === userId);
  if (!exists) {
    users.push({
      id: userId,
      username: ctx.from.username || 'N/A',
      firstName: ctx.from.first_name || 'N/A',
      joinedAt: new Date().toISOString()
    });
    writeJSON(USERS_FILE, users);
  }

  return next();
});

// --- USER HANDLERS ---

// /start Command
bot.start(async (ctx) => {
  const userId = ctx.from.id;
  states.delete(userId); // Clear states

  const settings = readJSON(SETTINGS_FILE);
  const caption = settings.startCaption || 'Welcome!';
  const startImage = settings.startImage;

  try {
    if (startImage) {
      await ctx.replyWithPhoto(startImage, {
        caption: caption,
        parse_mode: 'HTML',
        ...getStartKeyboard(userId)
      });
    } else {
      await ctx.reply(caption, {
        parse_mode: 'HTML',
        ...getStartKeyboard(userId)
      });
    }
  } catch (err) {
    console.error('Error starting bot:', err.message);
    await ctx.reply(caption, {
      parse_mode: 'HTML',
      ...getStartKeyboard(userId)
    });
  }
});

// "Get Premium" Click Handler
bot.action('verify_payment', async (ctx) => {
  await ctx.answerCbQuery();

  states.set(ctx.from.id, {
    step: 'awaiting_screenshot'
  });

  await ctx.reply(
    '📸 Please send your payment screenshot as a photo.\n\nAfter sending the screenshot, your request will be forwarded to the admin for verification.'
  );
});
bot.action('get_premium', async (ctx) => {
  await ctx.answerCbQuery();
  const userId = ctx.from.id;

  const settings = readJSON(SETTINGS_FILE);
  const upiId = settings.upiId || 'Not Set';
  const qrImage = settings.qrImage;
const verifyKeyboard = Markup.inlineKeyboard([
  [Markup.button.callback('✅ Verify Payment', 'verify_payment')]
]);
  const text = `💎 <b>GET PREMIUM SPECIAL ACCESS</b>\n\n` +
    `Complete your payment using the credentials below:\n\n` +
    `💵 <b>UPI ID:</b> <code>${upiId}</code>\n\n` +
    `⚠️ <b>Important Guidelines:</b>\n` +
    `1. Complete the payment using GPay, Paytm, or PhonePe.\n` +
    `2. Take a clear screenshot of the payment receipt.\n` +
    `3. <b>Upload/Send the screenshot directly as a photo to this chat.</b>`;

  states.set(userId, { step: 'awaiting_screenshot' });

  try {
    if (qrImage) {
        caption: text,await ctx.replyWithPhoto(qrImage, {
  caption: text,
  parse_mode: 'HTML',
  ...verifyKeyboard
});
   } else {
  await ctx.reply(text, {
    parse_mode: 'HTML',
    ...verifyKeyboard
  });

    }
  } catch (err) {
    console.error('Error opening premium view:', err.message);
    await ctx.reply(text, { parse_mode: 'HTML' });
  }
});

// --- ADMIN CONTROL CONSOLE (Command & Button) ---

function openAdminPanel(ctx, isCallback = false) {
  const msgText = '⚙️ <b>Welcome to the Admin Control Panel</b>\nManage settings, buttons, users, payouts, and system broadcasts here.';
  if (isCallback) {
    return ctx.editMessageText(msgText, {
      parse_mode: 'HTML',
      ...getAdminKeyboard()
    });
  } else {
    return ctx.reply(msgText, {
      parse_mode: 'HTML',
      ...getAdminKeyboard()
    });
  }
}

bot.command('admin', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.reply('❌ Access Denied.');
  states.delete(ADMIN_ID);
  await openAdminPanel(ctx, false);
});

bot.action('admin_panel', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery('❌ Access Denied.');
  await ctx.answerCbQuery();
  states.delete(ADMIN_ID);
  await openAdminPanel(ctx, false);
});

// Close Panel Callback
bot.action('adm_close', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.delete(ADMIN_ID);
  try {
    await ctx.deleteMessage();
  } catch (e) {}
});

// Back to Admin Home Callback
bot.action('adm_back', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.delete(ADMIN_ID);
  await openAdminPanel(ctx, true);
});

// Change Start Image Callback
bot.action('adm_change_img', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'set_start_img' });
  await ctx.reply('🖼️ <b>Upload a new Welcoming Image file</b>, or send <code>none</code> to remove start images completely.', { parse_mode: 'HTML' });
});

// Change Start Caption Callback
bot.action('adm_change_cap', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'set_start_caption' });
  await ctx.reply('✍️ <b>Provide your new Start Caption.</b> You can use HTML tags (<code>&lt;b&gt;</code>, <code>&lt;i&gt;</code>, etc.):', { parse_mode: 'HTML' });
});

// Change UPI Target ID Callback
bot.action('adm_set_upi', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'set_upi_id' });
  await ctx.reply('💳 <b>Provide your new Destination UPI Address:</b>\n(e.g., <code>username@bank</code>)', { parse_mode: 'HTML' });
});

// Change UPI QR Graphic Code Callback
bot.action('adm_set_qr', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'set_qr_code' });
  await ctx.reply('📷 <b>Upload a photo of your new UPI QR code</b>, or send <code>none</code> to disable the QR image:', { parse_mode: 'HTML' });
});
bot.action('adm_payment_settings', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();

  await ctx.answerCbQuery();

  states.set(ADMIN_ID, {
    step: 'payment_upi'
  });

  await ctx.reply('💳 Send your UPI ID:');
});
bot.action('adm_manage_files', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();

  await ctx.answerCbQuery();

  const keyboard = Markup.inlineKeyboard([
    [Markup.button.callback('➕ Add File', 'adm_add_file')],
    [Markup.button.callback('📋 View Files', 'adm_view_files')],
    [Markup.button.callback('❌ Delete File', 'adm_delete_files')]
  ]);

  await ctx.reply('📂 Premium File Manager', keyboard);
});

bot.action('adm_premium_file', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();

  await ctx.answerCbQuery();

  states.set(ADMIN_ID, {
    step: 'premium_file'
  });

  await ctx.reply(
    '📁 Upload the premium file.\n\nThis file will be sent automatically after payment approval.'
  );
});
bot.action('adm_manage_files', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();

  await ctx.answerCbQuery();

  const keyboard = Markup.inlineKeyboard([
    [Markup.button.callback('➕ Add File', 'adm_add_file')],
    [Markup.button.callback('📋 View Files', 'adm_view_files')],
    [Markup.button.callback('❌ Delete File', 'adm_delete_files')]
  ]);

  await ctx.reply(
    '📂 Premium File Manager',
    keyboard
  );
});
// View Stats Callback
bot.action('adm_add_file', async (ctx) => {
  await ctx.answerCbQuery();

  states.set(ADMIN_ID, {
    step: 'premium_file'
  });

  await ctx.reply('📁 Upload file.');
});

bot.action('adm_view_files', async (ctx) => {

  await ctx.answerCbQuery();

  const settings = readJSON(SETTINGS_FILE);

  if (!settings.premiumFiles ||
      settings.premiumFiles.length === 0) {

    return ctx.reply('❌ No files found.');
  }

  let msg = '📂 Premium Files\n\n';

  settings.premiumFiles.forEach((f, i) => {
    msg += `${i + 1}. ${f.name}\n`;
  });

  await ctx.reply(msg);
});

bot.action('adm_delete_files', async (ctx) => {

  await ctx.answerCbQuery();

  const settings = readJSON(SETTINGS_FILE);

  if (!settings.premiumFiles ||
      settings.premiumFiles.length === 0) {

    return ctx.reply('❌ No files found.');
  }

  const rows = settings.premiumFiles.map(
    (f, i) => [
      Markup.button.callback(
        `❌ ${f.name}`,
        `del_file_${i}`
      )
    ]
  );

  await ctx.reply(
    'Select file to delete:',
    Markup.inlineKeyboard(rows)
  );
});

bot.action(/^del_file_/, async (ctx) => {

  await ctx.answerCbQuery();

  const index = parseInt(
    ctx.callbackQuery.data.replace(
      'del_file_',
      ''
    )
  );

  const settings = readJSON(SETTINGS_FILE);

  settings.premiumFiles.splice(index, 1);

  writeJSON(
    SETTINGS_FILE,
    settings
  );

  await ctx.reply(
    '✅ File deleted successfully.'
  );
});

bot.action('adm_view_stats', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  const users = readJSON(USERS_FILE, []);
  const banned = readJSON(BANNED_FILE, []);
  await ctx.reply(
    `📊 <b>Real-Time Bot Statistics</b>\n\n` +
    `👥 <b>Total Users Joined:</b> <code>${users.length}</code>\n` +
    `🚫 <b>Total Banned Users:</b> <code>${banned.length}</code>`,
    { parse_mode: 'HTML' }
  );
});

// Ban User State Activation
bot.action('adm_ban_user', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'ban_user_id' });
  await ctx.reply('🚫 <b>Provide the Numeric User ID</b> of the user you want to ban from accessing the bot:', { parse_mode: 'HTML' });
});

// Unban User State Activation
bot.action('adm_unban_user', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'unban_user_id' });
  await ctx.reply('✅ <b>Provide the Numeric User ID</b> of the user you want to unban:', { parse_mode: 'HTML' });
});

// View Payments Callback
bot.action('adm_view_payments', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const payments = readJSON(PAYMENTS_FILE, []);
  const pending = payments.filter(p => p.status === 'pending');

  if (pending.length === 0) {
    return ctx.reply('📝 No pending payment verification requests in queue.');
  }

  await ctx.reply(`📊 <b>Found ${pending.length} Pending Payment Requests:</b>`);
  for (const r of pending) {
    const adminActionMarkup = Markup.inlineKeyboard([
      [
        Markup.button.callback('✅ Approve', `pay_app_${r.id}`),
        Markup.button.callback('❌ Reject', `pay_rej_${r.id}`)
      ]
    ]);
    try {
      await ctx.replyWithPhoto(r.fileId, {
        caption: `🔑 <b>Request ID:</b> <code>${r.id}</code>\n👤 <b>User:</b> @${r.username} (ID: <code>${r.userId}</code>)\n📅 <b>Time:</b> ${new Date(r.timestamp).toLocaleString()}`,
        parse_mode: 'HTML',
        ...adminActionMarkup
      });
    } catch (err) {
      console.error('Error forwarding payment detail to admin:', err.message);
    }
  }
});

// --- PAYMENT VALIDATION HANDLERS (CALLBACKS) ---
bot.action(/^(pay_app_|pay_rej_)/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
try {
  await ctx.answerCbQuery();
} catch (e) {}

  const isApprove = ctx.callbackQuery.data.startsWith('pay_app_');
  const reqId = ctx.callbackQuery.data.replace(isApprove ? 'pay_app_' : 'pay_rej_', '');

  const payments = readJSON(PAYMENTS_FILE, []);
  const index = payments.findIndex(p => p.id === reqId);

  if (index === -1) {
    return ctx.reply('❌ No record matching this request identifier found.');
  }

  const record = payments[index];
  if (record.status !== 'pending') {
    return ctx.reply(`⚠️ Request already updated: <b>${record.status}</b>`, { parse_mode: 'HTML' });
  }

  if (isApprove) {
    payments[index].status = 'approved';
    writeJSON(PAYMENTS_FILE, payments);
try {
  await ctx.editMessageReplyMarkup({ inline_keyboard: [] });
} catch (e) {}

    try {
      await ctx.telegram.sendMessage(record.userId, '🎉 <b>Congratulations!</b> Your payment screenshot has been verified. <b>Premium Unlocked Successfully!</b>', { parse_mode: 'HTML' });
const settings = readJSON(SETTINGS_FILE);
if (settings.premiumFiles) {

  for (const file of settings.premiumFiles) {

    await ctx.telegram.sendDocument(
      record.userId,
      file.fileId
    );

  }

}

    } catch (e) {
      console.error('Failed to notify approved user:', e.message);
    }

    await ctx.editMessageCaption(`✅ <b>Approved Request</b> <code>${reqId}</code>\n👤 User: @${record.username} (ID: <code>${record.userId}</code>)`, { parse_mode: 'HTML' });
  } else {
    payments[index].status = 'rejected';
    writeJSON(PAYMENTS_FILE, payments);
try {
  await ctx.editMessageReplyMarkup({ inline_keyboard: [] });
} catch (e) {}

    try {
      await ctx.telegram.sendMessage(record.userId, '❌ <b>Payment Denied!</b>\nAdmin was unable to verify your receipt details. Please contact support or resend valid proof.', { parse_mode: 'HTML' });
    } catch (e) {
      console.error('Failed to notify rejected user:', e.message);
    }

    await ctx.editMessageCaption(`❌ <b>Rejected Request</b> <code>${reqId}</code>\n👤 User: @${record.username} (ID: <code>${record.userId}</code>)`, { parse_mode: 'HTML' });
  }
});

// --- DYNAMIC BUTTONS ENGINE SUB-MENU ---
bot.action('adm_manage_btns', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const keyboard = Markup.inlineKeyboard([
    [Markup.button.callback('➕ Add Button', 'adm_btn_add'), Markup.button.callback('✏️ Edit Button', 'adm_btn_edit_list')],
    [Markup.button.callback('❌ Delete Button', 'adm_btn_del_list')],
    [Markup.button.callback('⬅️ Back to Admin Panel', 'adm_back')]
  ]);

  await ctx.editMessageText('🔘 <b>Manage Welcome Interface Dynamic Buttons:</b>\nAdd, edit, or clear keyboard layout nodes.', {
    parse_mode: 'HTML',
    ...keyboard
  });
});

// Add Button Clicked
bot.action('adm_btn_add', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'add_btn_text' });
  await ctx.reply('👉 <b>Please enter the Text Label</b> for your new button:', { parse_mode: 'HTML' });
});

// Add Button Type Selection (from state engine)
bot.action(/^(adm_btn_type_url|adm_btn_type_cb)/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const type = ctx.callbackQuery.data === 'adm_btn_type_url' ? 'url' : 'callback';
  const currentState = states.get(ADMIN_ID);

  if (!currentState || currentState.step !== 'add_btn_type') {
    return ctx.reply('❌ Flow disrupted. Return and start button process again.');
  }

  states.set(ADMIN_ID, {
    step: 'add_btn_val',
    tempText: currentState.tempText,
    tempType: type
  });

  if (type === 'url') {
    await ctx.reply('🔗 <b>Send the destination URL:</b>\n(e.g., <code>https://t.me/example</code>)', { parse_mode: 'HTML' });
  } else {
    await ctx.reply('⚡ <b>Send the Callback Payload/Payload Action identifier string:</b>\n(e.g., <code>help_action</code>)', { parse_mode: 'HTML' });
  }
});

// Edit Button List Selection
bot.action('adm_btn_edit_list', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const buttons = readJSON(BUTTONS_FILE, []);
  if (buttons.length === 0) {
    return ctx.reply('⚠️ No custom buttons registered to edit.');
  }

  const rows = buttons.map(b => [Markup.button.callback(b.text, `adm_btn_edit_node_${b.id}`)]);
  rows.push([Markup.button.callback('⬅️ Back to Dynamic Buttons', 'adm_manage_btns')]);

  await ctx.editMessageText('✏️ <b>Select a button configuration to update:</b>', {
    parse_mode: 'HTML',
    ...Markup.inlineKeyboard(rows)
  });
});

// Individual Edit Button Action Submenu
bot.action(/^adm_btn_edit_node_/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const btnId = ctx.callbackQuery.data.replace('adm_btn_edit_node_', '');
  const buttons = readJSON(BUTTONS_FILE, []);
  const btn = buttons.find(b => b.id === btnId);

  if (!btn) return ctx.reply('❌ Button metadata not found.');

  const keyboard = Markup.inlineKeyboard([
    [Markup.button.callback('✏️ Edit Text Label', `adm_ed_txt_${btnId}`), Markup.button.callback('🔗 Edit URL/Action Value', `adm_ed_val_${btnId}`)],
    [Markup.button.callback('⬅️ Back to List', 'adm_btn_edit_list')]
  ]);

  await ctx.editMessageText(
    `⚙️ <b>Edit Button Node Configuration:</b>\n\n` +
    `🏷️ <b>Label:</b> <code>${btn.text}</code>\n` +
    `⚡ <b>Type:</b> <code>${btn.type}</code>\n` +
    `🎯 <b>Target:</b> <code>${btn.value}</code>`,
    { parse_mode: 'HTML', ...keyboard }
  );
});

// Trigger change button text state
bot.action(/^adm_ed_txt_/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  const btnId = ctx.callbackQuery.data.replace('adm_ed_txt_', '');
  states.set(ADMIN_ID, { step: 'edit_btn_text', btnId });
  await ctx.reply('✍️ <b>Please send the new Text Label</b> for this button:', { parse_mode: 'HTML' });
});

// Trigger change button target value state
bot.action(/^adm_ed_val_/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  const btnId = ctx.callbackQuery.data.replace('adm_ed_val_', '');
  states.set(ADMIN_ID, { step: 'edit_btn_val', btnId });
  await ctx.reply('🎯 <b>Please send the new destination URL or Callback Action target value:</b>', { parse_mode: 'HTML' });
});

// Delete Button List Selection
bot.action('adm_btn_del_list', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const buttons = readJSON(BUTTONS_FILE, []);
  if (buttons.length === 0) {
    return ctx.reply('⚠️ No custom buttons registered to delete.');
  }

  const rows = buttons.map(b => [Markup.button.callback(`❌ ${b.text}`, `adm_btn_del_node_${b.id}`)]);
  rows.push([Markup.button.callback('⬅️ Back to Dynamic Buttons', 'adm_manage_btns')]);

  await ctx.editMessageText('❌ <b>Select the button you want to delete permanently:</b>', {
    parse_mode: 'HTML',
    ...Markup.inlineKeyboard(rows)
  });
});

// Handle deletion callback execution
bot.action(/^adm_btn_del_node_/, async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();

  const btnId = ctx.callbackQuery.data.replace('adm_btn_del_node_', '');
  let buttons = readJSON(BUTTONS_FILE, []);
  const btn = buttons.find(b => b.id === btnId);

  if (!btn) return ctx.reply('❌ Button metadata not found.');

  buttons = buttons.filter(b => b.id !== btnId);
  writeJSON(BUTTONS_FILE, buttons);

  await ctx.reply(`✅ <b>"${btn.text}"</b> button removed from startup menus successfully!`);
  await openAdminPanel(ctx, false);
});

// --- BROADCAST MANAGER SELECTION SUB-MENU ---
bot.action('adm_broadcast_menu', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  await ctx.editMessageText('📣 <b>Select Broadcast Type:</b>\nChoose preferred delivery configurations:', {
    parse_mode: 'HTML',
    ...getBroadcastTypeKeyboard()
  });
});

// Broadcast action trigger states
bot.action('bc_text', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'bc_text' });
  await ctx.reply('📝 <b>Send the HTML format Text Broadcast</b> to push to all users:', { parse_mode: 'HTML' });
});

bot.action('bc_photo', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'bc_photo' });
  await ctx.reply('🖼️ <b>Upload the Broadcast Photo</b> with optional caption:', { parse_mode: 'HTML' });
});

bot.action('bc_text_btn', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'bc_text_btn' });
  await ctx.reply(
    `🔗 <b>Text + Inline URL Button Broadcast</b>\n\n` +
    `Send details using this format structure:\n` +
    `<code>Message Body Text === Button Text === Button URL</code>\n\n` +
    `<b>Example:</b>\n` +
    `<code>Check out our group chat! === Join Group === https://t.me/Telegram</code>`,
    { parse_mode: 'HTML' }
  );
});

bot.action('bc_photo_btn', async (ctx) => {
  if (ctx.from.id !== ADMIN_ID) return ctx.answerCbQuery();
  await ctx.answerCbQuery();
  states.set(ADMIN_ID, { step: 'bc_photo_btn' });
  await ctx.reply(
    `🖼️🔗 <b>Photo + Inline URL Button Broadcast</b>\n\n` +
    `Upload a photo, and set its <b>caption content</b> to the exact layout format below:\n` +
    `<code>Message Caption === Button Text === Button URL</code>\n\n` +
    `<b>Example Caption:</b>\n` +
    `<code>Join our update hub! === Open Hub === https://t.me/Telegram</code>`,
    { parse_mode: 'HTML' }
  );
});

// Broadcast Worker Core Processor Function
async function runSystemBroadcast(ctx, payload) {
  const users = readJSON(USERS_FILE, []);
  if (users.length === 0) return ctx.reply('❌ No user entries registered on database.');

  await ctx.reply(`📣 <b>Transmitting broadcast to ${users.length} users...</b>`, { parse_mode: 'HTML' });
  let successCount = 0;
  let failureCount = 0;

  for (const user of users) {
    try {
      if (payload.type === 'text') {
        await ctx.telegram.sendMessage(user.id, payload.text, { parse_mode: 'HTML' });
      } else if (payload.type === 'photo') {
        await ctx.telegram.sendPhoto(user.id, payload.photoId, { caption: payload.caption, parse_mode: 'HTML' });
      } else if (payload.type === 'text_btn') {
        await ctx.telegram.sendMessage(user.id, payload.text, {
          parse_mode: 'HTML',
          ...Markup.inlineKeyboard([[Markup.button.url(payload.btnText, payload.btnUrl)]])
        });
      } else if (payload.type === 'photo_btn') {
        await ctx.telegram.sendPhoto(user.id, payload.photoId, {
          caption: payload.caption,
          parse_mode: 'HTML',
          ...Markup.inlineKeyboard([[Markup.button.url(payload.btnText, payload.btnUrl)]])
        });
      }
      successCount++;
    } catch (err) {
      console.error(`Skipping broadcast dispatch to user ${user.id}:`, err.message);
      failureCount++;
    }
    // Prevent telegram rate limit blocks
    await new Promise(resolve => setTimeout(resolve, 50));
  }

  states.delete(ADMIN_ID);
  await ctx.reply(
    `📢 <b>Broadcast Sequence Completed!</b>\n\n` +
    `✅ <b>Successful Deliveries:</b> <code>${successCount}</code>\n` +
    `❌ <b>Failed / Inactive Blocks:</b> <code>${failureCount}</code>`,
    { parse_mode: 'HTML' }
  );
  await openAdminPanel(ctx, false);
}

// --- DYNAMIC INBOUND MEDIA & STRING CAPTURE CONTROLLERS ---
bot.on(['text', 'photo', 'document'], async (ctx, next) => {
  const userId = ctx.from.id;
  const state = states.get(userId);

  if (!state) return next();

  const textInput = ctx.message.text;

  // --- USER LEVEL HANDLER: UPI Screenshots Receipt Upload ---
  if (userId !== ADMIN_ID) {
    if (state.step === 'awaiting_screenshot') {
      let fileId;
      if (ctx.message.photo) {
        fileId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
      } else {
        return ctx.reply('❌ Please upload an image file of your payment screenshot.');
      }

      const reqId = `REQ_${Date.now()}`;
      const payments = readJSON(PAYMENTS_FILE, []);

      payments.push({
        id: reqId,
        userId: userId,
        username: ctx.from.username || 'N/A',
        firstName: ctx.from.first_name || 'N/A',
        fileId: fileId,
        status: 'pending',
        timestamp: new Date().toISOString()
      });

      writeJSON(PAYMENTS_FILE, payments);
      states.delete(userId); // Clear active session state

      await ctx.reply('✅ <b>Payment receipt received successfully!</b>\nAdmin verification is currently under review. You will receive an automated alert here.', { parse_mode: 'HTML' });

      // Notify administrator immediately
      const adminKeyboard = Markup.inlineKeyboard([
        [
          Markup.button.callback('✅ Approve', `pay_app_${reqId}`),
          Markup.button.callback('❌ Reject', `pay_rej_${reqId}`)
        ]
      ]);

      try {
        await ctx.telegram.sendPhoto(ADMIN_ID, fileId, {
          caption: `🚨 <b>New Premium Request</b>\n\n` +
            `👤 <b>User:</b> ${ctx.from.first_name || 'N/A'} (@${ctx.from.username || 'N/A'})\n` +
            `🆔 <b>User ID:</b> <code>${userId}</code>\n` +
            `🔑 <b>Request ID:</b> <code>${reqId}</code>\n` +
            `📅 <b>Date:</b> ${new Date().toLocaleString()}`,
          parse_mode: 'HTML',
          ...adminKeyboard
        });
      } catch (err) {
        console.error('Failed to dispatch alert copy to Admin ID:', err.message);
      }
      return;
    }
    return next();
  }

  // --- ADMIN LEVEL SYSTEM CONFIGURATION INPUT ROUTERS ---

  // Start Welcome Screen Image Config
  if (state.step === 'set_start_img') {
    const settings = readJSON(SETTINGS_FILE);
    if (textInput && textInput.toLowerCase() === 'none') {
      settings.startImage = '';
      writeJSON(SETTINGS_FILE, settings);
      states.delete(ADMIN_ID);
      await ctx.reply('✅ Welcome asset image cleared. Starting screen set to text-only mode.');
      return openAdminPanel(ctx, false);
    }

    if (!ctx.message.photo) {
      return ctx.reply('❌ Invalid file. Please upload an image file, or type <code>none</code>.', { parse_mode: 'HTML' });
    }

    const photoId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
    settings.startImage = photoId;
    writeJSON(SETTINGS_FILE, settings);
    states.delete(ADMIN_ID);

    await ctx.reply('✅ Startup welcome screen image updated successfully!');
    return openAdminPanel(ctx, false);
  }

  // Start Welcome Screen Caption Config
  if (state.step === 'set_start_caption') {
    if (!textInput) return ctx.reply('❌ Text value not found.');
    const settings = readJSON(SETTINGS_FILE);
    settings.startCaption = textInput;
    writeJSON(SETTINGS_FILE, settings);
    states.delete(ADMIN_ID);

    await ctx.reply('✅ Startup welcome text caption set successfully!');
    return openAdminPanel(ctx, false);
  }
if (state.step === 'payment_upi') {

  const settings = readJSON(SETTINGS_FILE);

  settings.upiId = textInput;

  writeJSON(SETTINGS_FILE, settings);

  states.set(ADMIN_ID, {
    step: 'payment_qr'
  });

  return ctx.reply('📷 Now send your QR Code photo:');
}

if (state.step === 'payment_qr') {

  if (!ctx.message.photo) {
    return ctx.reply('❌ Please send QR code photo.');
  }

  const settings = readJSON(SETTINGS_FILE);

  settings.qrImage =
    ctx.message.photo[ctx.message.photo.length - 1].file_id;

  writeJSON(SETTINGS_FILE, settings);

  states.delete(ADMIN_ID);

  await ctx.reply('✅ Payment settings updated successfully.');

  return openAdminPanel(ctx, false);
}
  // Set UPI Config
  if (state.step === 'set_upi_id') {
    if (!textInput) return ctx.reply('❌ Text value not found.');
    const settings = readJSON(SETTINGS_FILE);
    settings.upiId = textInput;
    writeJSON(SETTINGS_FILE, settings);
    states.delete(ADMIN_ID);

    await ctx.reply(`✅ System transaction UPI address updated to: <code>${textInput}</code>`, { parse_mode: 'HTML' });
    return openAdminPanel(ctx, false);
  }

  // Set UPI QR Code Config
  if (state.step === 'set_qr_code') {
    const settings = readJSON(SETTINGS_FILE);
    if (textInput && textInput.toLowerCase() === 'none') {
      settings.qrImage = '';
      writeJSON(SETTINGS_FILE, settings);
      states.delete(ADMIN_ID);
      await ctx.reply('✅ UPI QR graphics deleted. QR display disabled.');
      return openAdminPanel(ctx, false);
    }

    if (!ctx.message.photo) {
      return ctx.reply('❌ Invalid file. Please upload an image file, or type <code>none</code>.', { parse_mode: 'HTML' });
    }

    const photoId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
    settings.qrImage = photoId;
    writeJSON(SETTINGS_FILE, settings);
    states.delete(ADMIN_ID);

    await ctx.reply('✅ System transactional QR graphic image updated successfully!');
    return openAdminPanel(ctx, false);
  }
if (state.step === 'premium_file') {

  if (!ctx.message.document) {
    return ctx.reply('❌ Please upload a document/file.');
  }
const settings = readJSON(SETTINGS_FILE);

if (!settings.premiumFiles) {
  settings.premiumFiles = [];
}

settings.premiumFiles.push({
  name: ctx.message.document.file_name,
  fileId: ctx.message.document.file_id
});

writeJSON(SETTINGS_FILE, settings);

  states.delete(ADMIN_ID);

  await ctx.reply('✅ Premium file saved successfully.');

  return openAdminPanel(ctx, false);
}
  // Ban Target User Process
  if (state.step === 'ban_user_id') {
    const banId = parseInt(textInput, 10);
    if (isNaN(banId)) return ctx.reply('❌ Invalid ID! Input numerical values only.');

    const banned = readJSON(BANNED_FILE, []);
    if (banned.includes(banId)) {
      states.delete(ADMIN_ID);
      await ctx.reply('⚠️ User already restricted from accessing this bot.');
      return openAdminPanel(ctx, false);
    }

    banned.push(banId);
    writeJSON(BANNED_FILE, banned);
    states.delete(ADMIN_ID);

    await ctx.reply(`✅ <b>Banned user ID successfully:</b> <code>${banId}</code>`, { parse_mode: 'HTML' });
    return openAdminPanel(ctx, false);
  }

  // Unban Target User Process
  if (state.step === 'unban_user_id') {
    const unbanId = parseInt(textInput, 10);
    if (isNaN(unbanId)) return ctx.reply('❌ Invalid ID! Input numerical values only.');

    let banned = readJSON(BANNED_FILE, []);
    if (!banned.includes(unbanId)) {
      states.delete(ADMIN_ID);
      await ctx.reply('⚠️ User does not exist inside system ban lists.');
      return openAdminPanel(ctx, false);
    }

    banned = banned.filter(id => id !== unbanId);
    writeJSON(BANNED_FILE, banned);
    states.delete(ADMIN_ID);

    await ctx.reply(`✅ <b>Unbanned user ID successfully:</b> <code>${unbanId}</code>`, { parse_mode: 'HTML' });
    return openAdminPanel(ctx, false);
  }

  // Add Dynamic Menu Button: Capturing Label
  if (state.step === 'add_btn_text') {
    if (!textInput) return ctx.reply('❌ Button label cannot be blank.');
    states.set(ADMIN_ID, { step: 'add_btn_type', tempText: textInput });

    const btnTypeKeyboard = Markup.inlineKeyboard([
      [Markup.button.callback('🔗 Destination URL (Link)', 'adm_btn_type_url')],
      [Markup.button.callback('⚡ Callback Value (Action)', 'adm_btn_type_cb')]
    ]);

    await ctx.reply(`🏷️ Button Label set as: <b>"${textInput}"</b>\n\nSelect button destination type below:`, {
      parse_mode: 'HTML',
      ...btnTypeKeyboard
    });
    return;
  }

  // Add Dynamic Menu Button: Saving values
  if (state.step === 'add_btn_val') {
    if (!textInput) return ctx.reply('❌ Button target value cannot be empty.');
    const buttons = readJSON(BUTTONS_FILE, []);

    const shortId = `btn_${Math.random().toString(36).substring(2, 8)}`;
    buttons.push({
      id: shortId,
      text: state.tempText,
      type: state.tempType,
      value: textInput
    });

    writeJSON(BUTTONS_FILE, buttons);
    states.delete(ADMIN_ID);

    await ctx.reply(`✅ <b>Button Added!</b>\n\n🏷️ Label: <b>${state.tempText}</b>\n⚡ Type: <b>${state.tempType}</b>\n🎯 Target: <code>${textInput}</code>`, { parse_mode: 'HTML' });
    return openAdminPanel(ctx, false);
  }

  // Edit Dynamic Menu Button: Edit Text Label
  if (state.step === 'edit_btn_text') {
    if (!textInput) return ctx.reply('❌ Text value not found.');
    const buttons = readJSON(BUTTONS_FILE, []);
    const idx = buttons.findIndex(b => b.id === state.btnId);

    if (idx !== -1) {
      buttons[idx].text = textInput;
      writeJSON(BUTTONS_FILE, buttons);
      await ctx.reply(`✅ Text label updated successfully: <b>"${textInput}"</b>`, { parse_mode: 'HTML' });
    } else {
      await ctx.reply('❌ Update error. Button matching ID details not found.');
    }

    states.delete(ADMIN_ID);
    return openAdminPanel(ctx, false);
  }

  // Edit Dynamic Menu Button: Edit Value target
  if (state.step === 'edit_btn_val') {
    if (!textInput) return ctx.reply('❌ Target value not found.');
    const buttons = readJSON(BUTTONS_FILE, []);
    const idx = buttons.findIndex(b => b.id === state.btnId);

    if (idx !== -1) {
      buttons[idx].value = textInput;
      // Recalculate type just in case URL/Callback character mappings changed
      buttons[idx].type = textInput.startsWith('http://') || textInput.startsWith('https://') ? 'url' : 'callback';
      writeJSON(BUTTONS_FILE, buttons);
      await ctx.reply(`✅ Destination target updated to: <code>${textInput}</code>`, { parse_mode: 'HTML' });
    } else {
      await ctx.reply('❌ Update error. Button matching ID details not found.');
    }

    states.delete(ADMIN_ID);
    return openAdminPanel(ctx, false);
  }

  // --- BROADCAST STATE CONSOLE ROUTERS ---

  // Plain Text Broadcaster
  if (state.step === 'bc_text') {
    if (!textInput) return ctx.reply('❌ Message text cannot be empty.');
    return runSystemBroadcast(ctx, { type: 'text', text: textInput });
  }

  // Photo Broadcaster
  if (state.step === 'bc_photo') {
    if (!ctx.message.photo) return ctx.reply('❌ File structure error. Please upload a photo.');
    const photoId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
    const caption = ctx.message.caption || '';
    return runSystemBroadcast(ctx, { type: 'photo', photoId, caption });
  }

  // Text with Inline Button URL Broadcaster
  if (state.step === 'bc_text_btn') {
    if (!textInput) return ctx.reply('❌ Broadcast data empty.');
    const parts = textInput.split('===').map(x => x.trim());

    if (parts.length < 3) {
      return ctx.reply('❌ Parsing issue. Format must exactly match:\n<code>Message Text === Button Text === Button URL</code>', { parse_mode: 'HTML' });
    }

    const [body, text, url] = parts;
    return runSystemBroadcast(ctx, { type: 'text_btn', text: body, btnText: text, btnUrl: url });
  }

  // Photo with Inline Button URL Broadcaster
  if (state.step === 'bc_photo_btn') {
    if (!ctx.message.photo) return ctx.reply('❌ Image structure issue. Please upload an image.');
    const photoId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
    const caption = ctx.message.caption || '';
    const parts = caption.split('===').map(x => x.trim());

    if (parts.length < 3) {
      return ctx.reply('❌ Parsing issue in caption. Format must exactly match:\n<code>Message Caption === Button Text === Button URL</code>', { parse_mode: 'HTML' });
    }

    const [body, text, url] = parts;
    return runSystemBroadcast(ctx, { type: 'photo_btn', photoId, caption: body, btnText: text, btnUrl: url });
  }
});

// Generic dynamic callback feedback receiver
bot.on('callback_query', async (ctx, next) => {
  const data = ctx.callbackQuery.data;

  // Let core system commands pass to higher registers
  if (data === 'get_premium' || data === 'admin_panel' || data.startsWith('adm_') || data.startsWith('bc_') || data.startsWith('pay_')) {
    return next();
  }

  // Acknowledge dynamic callback custom queries created by administrative edits
  await ctx.answerCbQuery(`⚡ Callback Node: "${data}" triggered.`, { show_alert: true });
});

// --- GLOBAL LAUNCH SEQUENCING ---
console.log("🚀 Starting Bot...");

bot.launch();

console.log("✅ Bot Started Successfully!");

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM')); 
