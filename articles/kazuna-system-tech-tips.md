---
title: "Next.js + NestJS + Docker + さくらVPSで詰まった5つのハマりポイントと解決策"
emoji: "🔧"
type: "tech"
topics: ["nextjs", "nestjs", "docker", "vps", "typescript"]
published: true
---

# Next.js + NestJS + Docker + さくらVPSで詰まった5つのポイントと解決策

個人事業主（食品卸業）から業務委託を受け、業務管理システムを一人で開発・本番デプロイした際に詰まったポイントをまとめました。

システムの全体像・要件定義からアジャイル開発・手順書配布までの全工程は別記事にまとめています。
→ **[個人事業主から業務委託を受け、発酵食品卸の業務管理システムを一人で完遂した話（全体像編）](#)** ※公開予定

技術スタック：`Next.js 15（App Router）` / `NestJS` / `TypeScript` / `PostgreSQL` / `Docker Compose` / `さくらVPS（Ubuntu / 1GB）`

---

## 詰まりポイント① 日本語ファイル名がアップロード後に文字化けする

### 問題

NestJSでMultipartファイルアップロードを実装した際、**PDFのファイル名に日本語が含まれていると文字化け**が発生した。

```
// アップロード後のファイル名
"é–‹ç¤ºæå®šé€šçŸ¥æ›¸.pdf"  // 文字化けした状態
"開示指定通知書.pdf"          // 期待値
```

書類アーカイブ機能では顧客名・年月を含む日本語ファイル名をそのまま保存・検索する仕様だったため、これは致命的なバグだった。

### 原因

MulterがHTTPマルチパートリクエストのヘッダーを **latin1（ISO-8859-1）** として解釈し、UTF-8の日本語バイト列をそのまま文字列化してしまう。

ブラウザ側は正しくUTF-8で送信しているが、Multerがデコード済みのlatin1文字列として渡してくるため、受け取り側でUTF-8に変換し直す必要がある。

### 解決策

`Buffer.from()` でlatin1 → UTF-8に再変換する。

```typescript
// NestJS / Multerのファイルインターセプター内で変換
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';

@Injectable()
export class JapaneseFilenameInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    if (request.file) {
      request.file.originalname = Buffer.from(
        request.file.originalname,
        'latin1'
      ).toString('utf8');
    }
    if (request.files) {
      request.files.forEach((file: Express.Multer.File) => {
        file.originalname = Buffer.from(
          file.originalname,
          'latin1'
        ).toString('utf8');
      });
    }
    return next.handle();
  }
}
```

コントローラーでの使い方：

```typescript
@Post('upload')
@UseInterceptors(
  FileInterceptor('file'),
  JapaneseFilenameInterceptor,
)
async uploadFile(@UploadedFile() file: Express.Multer.File) {
  // この時点で file.originalname は正しいUTF-8文字列
  console.log(file.originalname); // "開示指定通知書.pdf"
}
```

:::message
**ポイント**
`'latin1'` → `'utf8'` の変換はMulter固有の問題。Express + Multerの組み合わせではほぼ必ず発生するため、日本語ファイル名を扱うシステムでは最初から対応しておくことを推奨する。
:::

---

## 詰まりポイント② Docker Compose内部でNextからNestへのAPIリクエストがCONNECTION_REFUSEDになる

### 問題

ローカル開発では `http://localhost:3001` でAPIを叩けていたのに、**Docker Compose環境でSSR（Server Side）からAPIリクエストするとCONNECTION_REFUSEDになる**。

```
// SSR内（Server Component / API Route）で発生
FetchError: request to http://localhost:3001/api/customers failed
reason: connect ECONNREFUSED 127.0.0.1:3001
```

クライアント側（ブラウザからのCSRリクエスト）は問題なく動作していたため、原因の特定に時間がかかった。

### 原因

Docker Composeでは**各サービスは独立したネットワーク空間**を持つ。

`localhost` はコンテナ自身を指すため、Nextコンテナから `localhost:3001` にアクセスしても、そこには何もいない。Nestコンテナには **サービス名（`backend`）** でアクセスする必要がある。

```
[ ブラウザ ]  → http://localhost:3001  → [ Nextコンテナ（ポートフォワード） ]
[ Nextコンテナ(SSR) ] → http://backend:3001 → [ NestJS コンテナ ]
                             ↑ サービス名で通信
```

### 解決策

SSR（サーバーサイド）とCSR（クライアントサイド）でAPIのベースURLを切り替える。

```yaml
# docker-compose.yml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - INTERNAL_API_URL=http://backend:3001   # SSR用（コンテナ内部）
      - NEXT_PUBLIC_API_URL=http://localhost:3001  # CSR用（ブラウザから）
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "3001:3001"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/kazuna

  db:
    image: postgres:15
    ports:
      - "5432:5432"
```

Next.js側でSSR/CSRを判定してURLを切り替える：

```typescript
// lib/api.ts
const isServer = typeof window === 'undefined';

export const API_BASE_URL = isServer
  ? process.env.INTERNAL_API_URL   // SSR: http://backend:3001
  : process.env.NEXT_PUBLIC_API_URL; // CSR: http://localhost:3001

export async function fetchCustomers() {
  const res = await fetch(`${API_BASE_URL}/api/customers`);
  return res.json();
}
```

:::message alert
**よくある間違い**
`NEXT_PUBLIC_` プレフィックスを付けた環境変数はビルド時にクライアントバンドルに埋め込まれる。サーバー側でのみ使う内部URLには `NEXT_PUBLIC_` を付けてはいけない（セキュリティリスク）。
:::

---

## 詰まりポイント③ Next.js App RouterでdynamicParamsのTypeErrorが出る

### 問題

Next.js 15のApp Routerで動的ルートのページコンポーネントを実装した際、**`params.id` にアクセスするとTypeErrorが発生**する。

```typescript
// app/invoices/[id]/page.tsx
export default function InvoicePage({ params }: { params: { id: string } }) {
  console.log(params.id); // TypeError: params.id is not a string
}
```

### 原因

Next.js 15から **`params` が `Promise<{ id: string }>` 型に変更**された。非同期でパラメータを解決する設計になったため、`await` せずに参照するとエラーになる。

### 解決策

`params` を `await` してから使う。コンポーネントを `async` にする必要がある。

```typescript
// ❌ Next.js 14以前の書き方（15では動かない）
export default function InvoicePage({ params }: { params: { id: string } }) {
  const { id } = params;
  // ...
}

// ✅ Next.js 15以降の正しい書き方
export default async function InvoicePage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;  // awaitが必要
  const invoice = await getInvoice(id);
  // ...
}
```

`generateMetadata` でも同様に `await` が必要：

```typescript
export async function generateMetadata({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return { title: `請求書 #${id}` };
}
```

:::message
**バージョン確認コマンド**
```bash
npx next --version
```
Next.js 15以降であれば `params` は必ず `Promise` として扱う。
:::

---

## 詰まりポイント④ さくらVPS 1GBでDocker Composeが起動中にOOM Killerに落とされる

### 問題

さくらVPS（メモリ1GB）でDocker Composeを起動すると、**PostgreSQLまたはNestJSコンテナがランダムに落ちる**。ログを確認するとカーネルのOOM Killer（Out-Of-Memory Killer）に強制終了されていた。

```bash
# dmesgで確認
$ sudo dmesg | grep -i "killed process"
[12345.678] Out of memory: Killed process 1234 (node) total-vm:512000kB ...
```

本番デプロイ直後はサービスが安定していたが、数時間後にコンテナが突然落ちるという現象が断続的に発生した。

### 原因

**メモリ1GBはDocker Compose（Next.js + NestJS + PostgreSQL + nginx）をフル稼働させるには不足している。**

各サービスのメモリ消費の目安：
- Next.js（SSRビルド後）：約200〜300MB
- NestJS：約150〜200MB
- PostgreSQL：約100〜200MB
- nginx：約10〜30MB

合計で余裕で1GBを超える。

### 解決策

**スワップメモリを2GB追加設定**する。スワップはディスクをメモリの代替として使うため速度は落ちるが、1GBプランのVPSで複数コンテナを動かす際の現実的な解決策。

```bash
# スワップファイルを2GB作成
sudo fallocate -l 2G /swapfile

# パーミッション設定
sudo chmod 600 /swapfile

# スワップとして初期化
sudo mkswap /swapfile

# スワップを有効化
sudo swapon /swapfile

# 有効化を確認
free -h
#               total        used        free
# Mem:          987Mi        ...
# Swap:         2.0Gi        ...

# 再起動後も有効にする（/etc/fstabに追記）
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

スワップ設定後のメモリ状況を確認しながら運用：

```bash
# リアルタイムでメモリ・スワップ使用量を監視
watch -n 5 free -h

# コンテナ別のメモリ使用量確認
docker stats --no-stream
```

:::message
**補足**
スワップはディスクI/Oを多用するため、データベースの書き込みが多い用途では注意が必要。本番トラフィックが増えてきたらVPSプランのアップグレード（2〜4GB）を検討する。さくらVPSは後からプランアップグレードが可能。
:::

---

## 詰まりポイント⑤ インボイス制度対応の税率計算ロジックとExcel出力

### 問題

食品卸業では**同一請求書内に軽減税率8%（食品）と標準税率10%（容器・送料等）の品目が混在する**。さらにインボイス制度（適格請求書等保存方式）に対応した請求書フォーマットが法的に要求される。

単純な消費税計算とは異なり、以下が必要だった：
- 税率ごとに品目を分類
- 税率ごとに小計・消費税額を別計算
- 合計欄に税率別の内訳を明示
- Excel出力時にこのフォーマットを再現

### 設計・実装

税率ごとに品目を分類する型定義：

```typescript
// types/invoice.ts
export type TaxRate = 8 | 10;

export interface InvoiceItem {
  id: string;
  productName: string;
  quantity: number;
  unitPrice: number;
  taxRate: TaxRate;  // 8 or 10
}

export interface TaxSummary {
  taxRate: TaxRate;
  subtotal: number;     // 税抜き小計
  taxAmount: number;    // 消費税額（円以下切捨て）
  total: number;        // 税込み合計
}
```

税率別に集計する計算ロジック：

```typescript
// utils/invoice-calculator.ts
export function calculateTaxSummary(items: InvoiceItem[]): TaxSummary[] {
  const taxRates: TaxRate[] = [8, 10];

  return taxRates
    .map((rate) => {
      const targetItems = items.filter((item) => item.taxRate === rate);
      if (targetItems.length === 0) return null;

      const subtotal = targetItems.reduce(
        (sum, item) => sum + item.quantity * item.unitPrice,
        0
      );
      const taxAmount = Math.floor(subtotal * (rate / 100)); // 切り捨て
      return {
        taxRate: rate,
        subtotal,
        taxAmount,
        total: subtotal + taxAmount,
      };
    })
    .filter((summary): summary is TaxSummary => summary !== null);
}

export function calculateGrandTotal(summaries: TaxSummary[]) {
  return {
    subtotalAll: summaries.reduce((sum, s) => sum + s.subtotal, 0),
    taxAmountAll: summaries.reduce((sum, s) => sum + s.taxAmount, 0),
    grandTotal: summaries.reduce((sum, s) => sum + s.total, 0),
  };
}
```

Excel出力（`xlsx` ライブラリ使用）のインボイス形式セクション：

```typescript
// services/invoice-excel.service.ts
import * as XLSX from 'xlsx';

function addTaxSummarySection(ws: XLSX.WorkSheet, summaries: TaxSummary[], startRow: number) {
  summaries.forEach((summary, index) => {
    const row = startRow + index;
    // 税率ごとの内訳行
    XLSX.utils.sheet_add_aoa(ws, [[
      `※${summary.taxRate}%対象`,
      summary.subtotal.toLocaleString(),
      summary.taxAmount.toLocaleString(),
      summary.total.toLocaleString(),
    ]], { origin: { r: row, c: 0 } });
  });
}
```

:::message
**インボイス制度の要件ポイント**
2023年10月以降、適格請求書（インボイス）には「税率ごとの消費税額」の明記が義務付けられている。税率を混在させる場合は必ず税率別に分けて計算・表示する必要がある。
:::

---

## まとめ

| # | 詰まりポイント | 原因 | 解決のキーワード |
|---|---|---|---|
| ① | 日本語ファイル名の文字化け | MulterがLatin1で解釈 | `Buffer.from('latin1').toString('utf8')` |
| ② | Docker内部通信のCONNECTION_REFUSED | localhostはコンテナ自身を指す | サービス名 `http://backend:3001` を使う |
| ③ | `params.id` のTypeError | Next.js 15でparamsがPromise型に | `async` + `await params` |
| ④ | OOM KillerにコンテナがKillされる | VPS 1GBにはDocker Composeが重い | スワップメモリ2GBを追加 |
| ⑤ | 税率混在請求書の計算・Excel出力 | インボイス制度対応の複雑な要件 | 税率別に分類してからreduce集計 |

これらは「調べても情報が少ない」か「エラーメッセージと原因が直感的につながりにくい」問題ばかりで、それぞれ数時間〜半日詰まりました。

同じスタックで開発する方の参考になれば幸いです。

---

システム全体の設計・要件定義・アジャイル開発・手順書配布の話は別記事にまとめています。
→ **[個人事業主から業務委託を受け、発酵食品卸の業務管理システムを一人で完遂した話（全体像編）](#)** ※公開予定

GitHubリポジトリ：[ken-personal/kazuna-system](https://github.com/ken-personal/kazuna-system)
スキルシート：[Notion ポートフォリオ](https://tide-bank-7b4.notion.site/AWS-31b9ee43f66680eeadb1c5f2db6323ee)
