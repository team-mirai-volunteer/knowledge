# データ統合パターン実装ガイド

このドキュメントは、team-miraiの`get_supporter_info`と`get_stripe_info`privateプロジェクトから抽出した、データ統合の4つの主要パターンをまとめたものです。それぞれの使用場面、メリット、実装方法を具体的なサンプルコードと共に解説します。

## 目次

1. [Google Spreadsheetからデータを読む](#1-google-spreadsheetからデータを読む)
2. [集計データをGistに書く](#2-集計データをgistに書く)
3. [集計データをSpreadsheetに書く](#3-集計データをspreadsheetに書く)
4. [集計データをGitHubに書く](#4-集計データをgithubに書く)
5. [統合パターンの組み合わせ例](#5-統合パターンの組み合わせ例)
6. [セットアップ・設定方法](#6-セットアップ設定方法)

---

## 1. Google Spreadsheetからデータを読む

### 使用場面
- **手動入力データの取得**: ボランティアや担当者が手動で入力した寄付金情報、イベント参加者データなど
- **設定値の管理**: アプリケーションの設定値をスプレッドシートで管理し、非エンジニアでも更新可能にする
- **マスターデータの管理**: 商品リスト、価格表、カテゴリ情報などの参照データ

### メリット
- **非エンジニアでも編集可能**: プログラミング知識なしでデータを更新できる
- **リアルタイム反映**: スプレッドシートの変更が即座にシステムに反映される
- **共同編集**: 複数人で同時にデータを編集・管理できる
- **データ検証**: スプレッドシートの機能でデータの妥当性をチェック可能

### 実装サンプル（get_stripe_infoから抽出）

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build
from collections import defaultdict

def read_manual_data_from_sheets(credentials_path, spreadsheet_id, sheet_name):
    """
    Google Spreadsheetsから手動入力データを読み込む
    get_stripe_info/combine_data_sources.py から抽出
    """
    daily_totals = defaultdict(lambda: {"amount": 0, "count": 0})
    
    try:
        credentials = service_account.Credentials.from_service_account_file(
            credentials_path, scopes=["https://www.googleapis.com/auth/spreadsheets"]
        )
        
        service = build("sheets", "v4", credentials=credentials)
        
        result = (
            service.spreadsheets()
            .values()
            .get(spreadsheetId=spreadsheet_id, range=f"{sheet_name}!A:C")
            .execute()
        )
        
        values = result.get("values", [])
        
        if not values:
            print(f"スプレッドシート '{sheet_name}' にデータがありません")
            return daily_totals
        
        # ヘッダー行をスキップして処理
        for row in values[1:]:
            if not row:
                continue
            
            try:
                date = row[0] if len(row) > 0 else ""
                amount_str = row[1] if len(row) > 1 else "0"
                
                if date and amount_str:
                    cleaned_amount = amount_str.strip().replace(',', '')
                    if cleaned_amount.isdigit():
                        amount = int(cleaned_amount)
                        if amount > 0:
                            daily_totals[date]["amount"] += amount
                            daily_totals[date]["count"] += 1
                            
            except (ValueError, IndexError) as e:
                print(f"データ処理エラー (行: {row}): {e}")
                continue
        
        print(f"✅ スプレッドシートから {len(daily_totals)} 日分のデータを読み込みました")
        return daily_totals
        
    except Exception as e:
        print(f"❌ スプレッドシート読み込みエラー: {e}")
        return daily_totals
```

---

## 2. 集計データをGistに書く

### 使用場面
- **簡易API提供**: 外部アプリケーションからJSONデータを取得するエンドポイント
- **データ共有**: チーム内でのリアルタイムデータ共有
- **外部システム連携**: 他のサービスからのデータ参照, 例えばLooker StudioはJSONデータを直接読み込むことが可能

### メリット
- **無料で利用可能**: GitHub Gistは無料で使用できる
- **Raw URLでアクセス**: 直接JSONデータにHTTPアクセス可能
- **バージョン管理**: データの変更履歴が自動で保存される

### 実装サンプル（get_stripe_infoから抽出）

```python
import requests
import json
from datetime import datetime, timezone

def upload_to_gist(json_data, github_token, gist_id=None):
    """
    JSONデータをGitHub Gistにアップロード
    get_stripe_info/combine_data_sources.py から抽出
    """
    if not github_token:
        print("GitHub token not provided, skipping Gist upload")
        return None
    
    headers = {
        "Authorization": f"token {github_token}",
        "Accept": "application/vnd.github.v3+json"
    }
    
    timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")
    filename = "latest_stripe_data.json"
    
    gist_data = {
        "description": f"Stripe Analysis Data - {timestamp}",
        "public": False,
        "files": {
            filename: {
                "content": json.dumps(json_data, ensure_ascii=False, indent=2)
            }
        }
    }
    
    try:
        if gist_id:
            # 既存Gistの更新
            url = f"https://api.github.com/gists/{gist_id}"
            response = requests.patch(url, headers=headers, json=gist_data)
        else:
            # 新規Gist作成
            url = "https://api.github.com/gists"
            response = requests.post(url, headers=headers, json=gist_data)
        
        if response.status_code in [200, 201]:
            gist_info = response.json()
            print(f"Gist uploaded successfully: {gist_info['html_url']}")
            return gist_info
        else:
            print(f"Failed to upload Gist: {response.status_code} - {response.text}")
            return None
    except Exception as e:
        print(f"Error uploading to Gist: {e}")
        return None
```

---

## 3. 集計データをSpreadsheetに書く

### 使用場面
- **レポート作成**: 定期的な売上レポート、分析レポートの自動生成
- **ダッシュボード**: リアルタイムデータの可視化
- **データ共有**: 非エンジニアチームとのデータ共有

### メリット
- **視覚的**: グラフやチャートで直感的にデータを理解
- **計算機能**: スプレッドシートの関数で追加分析が可能
- **共有しやすい**: URLで簡単にチーム内共有
- **印刷対応**: レポートとして印刷・PDF化が容易

### 実装サンプル（get_stripe_infoから抽出）

```python
def update_spreadsheet(data, spreadsheet_id, sheet_name, credentials_path):
    """
    スプレッドシートにデータを書き込む
    get_stripe_info/combine_data_sources.py から抽出
    """
    try:
        credentials = service_account.Credentials.from_service_account_file(
            credentials_path, 
            scopes=["https://www.googleapis.com/auth/spreadsheets"]
        )
        
        service = build("sheets", "v4", credentials=credentials)
        
        # シートの存在確認と作成
        sheet_metadata = service.spreadsheets().get(spreadsheetId=spreadsheet_id).execute()
        sheet_exists = False
        
        for sheet in sheet_metadata["sheets"]:
            if sheet["properties"]["title"] == sheet_name:
                sheet_exists = True
                break
        
        if not sheet_exists:
            body = {
                "requests": [{
                    "addSheet": {
                        "properties": {"title": sheet_name}
                    }
                }]
            }
            service.spreadsheets().batchUpdate(
                spreadsheetId=spreadsheet_id, body=body
            ).execute()
            print(f"新しいシート '{sheet_name}' を作成しました")
        
        # データ書き込み
        body = {"values": data}
        
        result = (
            service.spreadsheets()
            .values()
            .update(
                spreadsheetId=spreadsheet_id,
                range=f"{sheet_name}!A1",
                valueInputOption="RAW",
                body=body,
            )
            .execute()
        )
        
        print(f"{result.get('updatedCells')} セルを更新しました")
        return result
        
    except Exception as e:
        print(f"スプレッドシート更新エラー: {e}")
        return None
```

---

## 4. 集計データをGitHubに書く

### 使用場面
- **データの永続化**: 重要なデータをGitリポジトリで管理・バックアップ
- **バージョン管理**: データの変更履歴を詳細に追跡
- **自動化ワークフロー**: GitHub Actionsと連携した定期的なデータ更新
- **透明性の確保**: オープンデータとしての公開・監査

### メリット
- **完全なバージョン履歴**: すべての変更が記録される
- **ブランチ機能**: 実験的なデータ処理を安全に実行
- **Issue・PR連携**: データ変更の議論・レビュープロセス

### 実装サンプル（GitHub Actionsワークフローから抽出）

```python
import subprocess
import json
from datetime import datetime

def commit_data_to_github(data, filename, commit_message=None):
    """
    データをGitHubリポジトリにコミット
    .github/workflows/hourly_stripe_update.yml のパターンを参考
    """
    try:
        # データをJSONファイルに書き込み
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        
        # Gitの設定
        subprocess.run(['git', 'config', '--local', 'user.email', 'actions@github.com'], check=True)
        subprocess.run(['git', 'config', '--local', 'user.name', 'GitHub Actions Bot'], check=True)
        
        # ファイルをステージング
        subprocess.run(['git', 'add', filename], check=True)
        
        # 変更があるかチェック
        result = subprocess.run(['git', 'diff', '--staged', '--quiet'], capture_output=True)
        
        if result.returncode == 0:
            print("変更がないため、コミットをスキップします")
            return True
        
        # コミット
        if not commit_message:
            timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            commit_message = f"Update {filename} - {timestamp}"
        
        subprocess.run(['git', 'commit', '-m', commit_message], check=True)
        subprocess.run(['git', 'push'], check=True)
        
        print(f"{filename} をGitHubにコミット・プッシュしました")
        return True
        
    except subprocess.CalledProcessError as e:
        print(f"Git操作エラー: {e}")
        return False
    except Exception as e:
        print(f"ファイル操作エラー: {e}")
        return False
```

### GitHub Actions設定例（実際のワークフローから抽出）

```yaml
name: Hourly Stripe Data Update

on:
  schedule:
    - cron: '0 * * * *'  # 毎時実行
  workflow_dispatch:

jobs:
  update-stripe-data:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          token: ${{ secrets.GH_TOKEN }}

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install stripe python-dotenv google-api-python-client google-auth requests

      - name: Combine all data sources and update summary sheet
        env:
          GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_CREDENTIALS }}
          SPREADSHEET_ID: ${{ secrets.SPREADSHEET_ID }}
          GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
          GIST_ID: ${{ secrets.GIST_ID }}
        run: |
          echo "$GOOGLE_CREDENTIALS" | base64 -d > google-credentials.json
          export GOOGLE_CREDENTIALS_PATH="google-credentials.json"
          
          python combine_data_sources.py
          
          rm google-credentials.json

      - name: Commit and push changes
        env:
          GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
        run: |
          git config --local user.email "actions@github.com"
          git config --local user.name "GitHub Actions Bot"
          git add stripe_charges*.csv stripe_analysis*.txt stripe_data*.json
          
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            timestamp=$(date +'%Y-%m-%d %H:%M:%S')
            git commit -m "Update Stripe data - ${timestamp}"
            git push
            echo "Changes pushed to repository"
          fi
```

---

## 5. 統合パターンの組み合わせ例

### 完全なデータパイプライン（combine_data_sources.pyから抽出）

```python
def complete_data_pipeline():
    """4つの統合パターンを組み合わせた完全なデータパイプライン"""
    
    # 1. Google Spreadsheetsからデータを読み込み
    manual_data = read_manual_data_from_sheets(
        CREDENTIALS_PATH, SPREADSHEET_ID, MANUAL_SHEET_NAME
    )
    
    # 2. 他のデータソースと統合
    account1_data = get_stripe_data_from_csv(account1_csv)
    account2_data = get_stripe_data_from_csv(account2_csv)
    
    # 3. データを統合
    combined_data = combine_data_sources(account1_data, account2_data, manual_data)
    
    # 4. データ分析
    analysis_results = analyze_combined_data(combined_data)
    
    # 5. 複数の出力先に配信
    # 5a. JSONファイルに書き込み
    json_data = write_json_file(analysis_results)
    
    # 5b. GitHub Gistに書き込み（API用）
    upload_to_gist(json_data, GIST_TOKEN, GIST_ID)
    
    # 5c. Google Spreadsheetsに書き込み（レポート用）
    spreadsheet_data = format_data_for_spreadsheet(combined_data)
    update_spreadsheet(spreadsheet_data, SPREADSHEET_ID, COMBINED_SHEET_NAME, CREDENTIALS_PATH)
    
    print("データパイプライン完了")
```

### 用途別の組み合わせパターン

#### パターンA: リアルタイムダッシュボード
```python
# Spreadsheet読み込み → Gist書き込み（API用）
manual_data = read_manual_data_from_sheets(...)
api_data = create_api_data(analyze_combined_data(manual_data))
upload_to_gist(api_data, ...)
```

#### パターンB: 定期レポート生成
```python
# 複数ソース統合 → Spreadsheet書き込み → GitHub履歴保存
combined_data = combine_multiple_sources(...)
update_spreadsheet(format_data_for_spreadsheet(combined_data), ...)
commit_data_to_github(combined_data, "monthly_report.json")
```

---

## 6. セットアップ・設定方法

### 必要な依存関係

```bash
pip install google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client requests python-dotenv
```

### 環境変数設定

```bash
# .env
GOOGLE_CREDENTIALS_PATH=google-credentials.json
SPREADSHEET_ID=your_spreadsheet_id
GIST_TOKEN=ghp_your_github_token
GIST_ID=your_gist_id
```

### 使い分けの指針

| パターン | 適用場面 | メリット | 注意点 |
|---------|---------|---------|--------|
| **Spreadsheet読み込み** | 手動データ入力 | 非エンジニア対応 | データ品質管理 |
| **Gist書き込み** | API提供 | 簡単・無料 | 更新頻度制限 |
| **Spreadsheet書き込み** | レポート・ダッシュボード | 視覚化・共有 | 容量制限 |
| **GitHub書き込み** | 履歴管理・監査 | 完全な履歴 | ストレージ使用量 |

これらのパターンを参考に、プロジェクトの要件に応じてカスタマイズして使用してください。
