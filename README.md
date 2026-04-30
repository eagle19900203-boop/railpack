/**
 * GUBON LUCID OS - 核心點火引擎 (正式環境版)
 * 校準：藍新正式網址、SHA256 加密、自動化 Webhook 解鎖
 */
const express = require('express');
const crypto = require('crypto');
const { PrismaClient } = require('@prisma/client');
const app = express();
const prisma = new PrismaClient();

app.use(express.json());
app.use(require('cors')());

const CONFIG = {
    MerchantID: process.env.NEWEBPAY_MERCHANT_ID,
    HashKey: process.env.NEWEBPAY_HASH_KEY,
    HashIV: process.env.NEWEBPAY_HASH_IV,
    PayURL: 'https://core.newebpay.com/MPG/mpg_gateway', // 正式收款網址
    Version: '2.0'
};

// 1. 支付發動：產出加密 TradeInfo
app.post('/api/payment/init', async (req, res) => {
    const { reportId, amount, email } = req.body;
    
    const tradeData = {
        MerchantID: CONFIG.MerchantID,
        RespondType: 'JSON',
        TimeStamp: Math.floor(Date.now() / 1000),
        Version: CONFIG.Version,
        MerchantOrderNo: `GB_${reportId}_${Date.now()}`,
        Amt: amount,
        ItemDesc: 'GUBON 核心決策指令',
        NotifyURL: `${process.env.BACKEND_URL}/api/payment/webhook`,
        ReturnURL: `${process.env.FRONTEND_URL}/success`,
        Email: email,
        LoginType: 0
    };

    // AES 加密
    const chain = Object.entries(tradeData).map(([k, v]) => `${k}=${v}`).join('&');
    const cipher = crypto.createCipheriv('aes-256-cbc', CONFIG.HashKey, CONFIG.HashIV);
    let aesEnc = cipher.update(chain, 'utf8', 'hex') + cipher.final('hex');

    // SHA256 簽章
    const sha = crypto.createHash('sha256')
        .update(`HashKey=${CONFIG.HashKey}&${aesEnc}&HashIV=${CONFIG.HashIV}`)
        .digest('hex').toUpperCase();

    res.json({ PayURL: CONFIG.PayURL, MerchantID: CONFIG.MerchantID, TradeInfo: aesEnc, TradeSha: sha });
});

// 2. 自動化 Webhook：藍新回傳成功後自動解鎖資料庫
app.post('/api/payment/webhook', async (req, res) => {
    const { Status, TradeInfo } = req.body;
    if (Status === 'SUCCESS') {
        const decipher = crypto.createDecipheriv('aes-256-cbc', CONFIG.HashKey, CONFIG.HashIV);
        let decoded = decipher.update(TradeInfo, 'hex', 'utf8') + decipher.final('utf8');
        const result = JSON.parse(decoded);
        const reportId = result.Result.MerchantOrderNo.split('_')[1];

        await prisma.report.update({ where: { id: reportId }, data: { isPaid: true } });
        console.log(`🔥 報告 ${reportId} 已自動解鎖！`);
    }
    res.send('1|OK');
});

app.get('/health', (req, res) => res.status(200).json({ status: 'IGNITED' }));
app.listen(process.env.PORT || 8080);
GUBON 最終點火執行手冊
​統帥，請按照此順序完成最後的操作。
​第一步：GitHub 變數配置 (Secrets)
​請前往您的 GitHub Repo > Settings > Secrets，填入：
​NEWEBPAY_MERCHANT_ID: 您的藍新正式 ID
​NEWEBPAY_HASH_KEY: 您的藍新正式 Key
​NEWEBPAY_HASH_IV: 您的藍新正式 IV
​DATABASE_URL: PostgreSQL 線上連線字串
​RENDER_DEPLOY_HOOK: Render 的部署連結
​VERCEL_TOKEN: Vercel 權杖
​第二步：資料庫點火
​在本地執行一次同步，確保線上資料表存在：
name: GUBON_FINAL_IGNITION

on:
  push:
    branches: [ "main" ] # 監控主分支推送

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 📥 複製代碼庫
        uses: actions/checkout@v4

      - name: 🛠️ 建立 Node.js 環境 (20.x)
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: 📦 安裝依賴與 Prisma 生成
        run: |
          npm install
          npx prisma generate

      - name: 🐳 容器化與推送 (Railpack 邏輯)
        # 此步驟模擬 Railpack 構建並推送至 Docker Hub
        run: |
          echo "正在構建 GUBON 核心鏡像..."
          # docker build -t ${{ secrets.DOCKER_USERNAME }}/gubon-engine:latest .
          # docker push ${{ secrets.DOCKER_USERNAME }}/gubon-engine:latest

      - name: 🚀 觸發雲端部署 (Render & Vercel)
        run: |
          # 觸發後端熱更新
          curl -X GET "${{ secrets.RENDER_DEPLOY_HOOK }}"
          echo "後端點火指令已下達"

      - name: 🎨 前端自動部署
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
{
  "$schema": "[https://railway.app/railway.schema.json](https://railway.app/railway.schema.json)",
  "build": {
    "builder": "NIXPACKS"
  },
  "deploy": {
    "startCommand": "node server.js",
    "restartPolicyType": "ON_FAILURE"
  }
}
const express = require('express');
const crypto = require('crypto');
const cors = require('cors');
const app = express();

app.use(cors());
app.use(express.json());

const CONFIG = {
    MerchantID: process.env.NEWEBPAY_MERCHANT_ID,
    HashKey: process.env.NEWEBPAY_HASH_KEY,
    HashIV: process.env.NEWEBPAY_HASH_IV,
    PayURL: '[https://core.newebpay.com/MPG/mpg_gateway](https://core.newebpay.com/MPG/mpg_gateway)', // 正式機
    Version: '2.0'
};

app.post('/api/pay/init', (req, res) => {
    const { orderId, amount } = req.body;
    const tradeData = {
        MerchantID: CONFIG.MerchantID,
        RespondType: 'JSON',
        TimeStamp: Math.floor(Date.now() / 1000),
        Version: CONFIG.Version,
        MerchantOrderNo: orderId,
        Amt: amount,
        ItemDesc: 'GUBON 指令解鎖',
        LoginType: 0
    };

    const chain = Object.entries(tradeData).map(([k,v]) => `${k}=${v}`).join('&');
    const cipher = crypto.createCipheriv('aes-256-cbc', CONFIG.HashKey, CONFIG.HashIV);
    let encrypted = cipher.update(chain, 'utf8', 'hex') + cipher.final('hex');
    const sha = crypto.createHash('sha256')
        .update(`HashKey=${CONFIG.HashKey}&${encrypted}&HashIV=${CONFIG.HashIV}`)
        .digest('hex').toUpperCase();

    res.json({ PayURL: CONFIG.PayURL, MerchantID: CONFIG.MerchantID, TradeInfo: encrypted, TradeSha: sha });
});

app.get('/health', (req, res) => res.send('IGNITED'));
app.listen(process.env.PORT || 8080);
name: GUBON_DEPLOY
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trigger Render
        run: curl -X GET "${{ secrets.RENDER_DEPLOY_HOOK }}"
      - name: Vercel Deploy
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-args: '--prod'
