# コーディングエージェント実装比較調査レポート

このレポートでは、3つのコーディングエージェント実装（OpenCode、Gemini CLI、cat-code）について詳細な技術調査を行った結果をまとめています。

## 調査対象

1. **OpenCode** - Go言語実装のターミナルベースAIアシスタント
2. **Gemini CLI** - Google Gemini APIを使用したNode.js/TypeScript実装
3. **cat-code** - 猫をテーマにしたシンプルなTypeScript/React実装

## 1. CLI使用技術と工夫

### OpenCode (Go実装)
- **フレームワーク**: Cobra (コマンドライン)、Bubble Tea (TUI)
- **主要ライブラリ**:
  - `charmbracelet/bubbletea`: ターミナルUI構築
  - `charmbracelet/lipgloss`: スタイリング
  - `spf13/cobra`: CLIコマンド管理
  - `spf13/viper`: 設定管理
- **特徴**:
  - フルスクリーンのインタラクティブTUI
  - 非対話モード対応 (`-p`フラグ)
  - 複数の出力フォーマット (text/json)
  - デバッグモード内蔵

### Gemini CLI (Node.js/TypeScript実装)
- **フレームワーク**: React + Ink (ターミナルUI)
- **主要技術**:
  - モノレポ構造 (packages/cli, packages/core)
  - ESBuildによる高速ビルド
  - React DevTools統合
  - TypeScript完全対応
- **特徴**:
  - Reactコンポーネントベースの設計
  - ストリーミング応答対応
  - テーマシステム
  - サンドボックス環境（Docker/Podman）統合

### cat-code (TypeScript/React実装)
- **フレームワーク**: React + Ink
- **主要ライブラリ**:
  - `commander`: CLIパラメータ処理
  - `ink`: React for CLI
  - `simple-git`: Git統合
  - `glob`: ファイル検索
- **特徴**:
  - シンプルで軽量な実装
  - 猫の鳴き声でコミュニケーション
  - セーフモード（--safe）対応
  - Bunランタイム使用

## 2. プロンプトエンジニアリング

### OpenCode
**階層的プロンプト管理**:
- エージェント別プロンプト（Coder、Title、Task、Summarizer）
- プロバイダー別最適化（OpenAI/Anthropic）
- 動的コンテキスト注入（環境情報、LSP情報、プロジェクト固有情報）

**特徴的な実装**:
```go
// プロバイダー別分岐
func CoderPrompt(provider models.ModelProvider) string {
    basePrompt := baseAnthropicCoderPrompt
    switch provider {
    case models.ProviderOpenAI:
        basePrompt = baseOpenAICoderPrompt
    }
    return fmt.Sprintf("%s\n\n%s\n%s", basePrompt, envInfo, lspInformation())
}
```

### Gemini CLI
**動的適応型プロンプト**:
- 環境変数によるカスタムプロンプト読み込み
- Git状態、サンドボックス環境に応じた動的調整
- 段階的ワークフロー（理解→計画→実装→検証）
- 履歴圧縮システムによる長期対話対応

**XML構造化履歴圧縮**:
```xml
<compressed_chat_history>
    <overall_goal>...</overall_goal>
    <key_knowledge>...</key_knowledge>
    <file_system_state>...</file_system_state>
    <recent_actions>...</recent_actions>
    <current_plan>...</current_plan>
</compressed_chat_history>
```

### cat-code
- 猫語のボキャブラリーベース応答
- 感情検出によるコンテキスト認識
- ultrathinkキーワードによる思考時間調整

## 3. ガードレール機能

### OpenCode
**包括的許可システム**:
- 操作別の許可要求（編集、実行、フェッチ）
- セッション単位の許可管理
- 危険コマンドのブラックリスト
- 安全な読み取り専用コマンドのホワイトリスト

**実装例**:
```go
var bannedCommands = []string{
    "curl", "wget", "nc", "telnet", "chrome", "firefox"
}

var safeReadOnlyCommands = []string{
    "ls", "echo", "pwd", "date", "git status", "go version"
}
```

### Gemini CLI
**多層防御アプローチ**:
- ツール実行前の確認システム
- 承認モード（DEFAULT、AUTO_EDIT、YOLO）
- サンドボックス統合（Docker/Podman/macOS Seatbelt）
- ファイルシステムアクセス制限
- コマンドホワイトリスト/ブラックリスト

**macOS Seatbeltプロファイル**:
- permissive-open/closed/proxied
- restrictive-open/closed/proxied

### cat-code
- セーフモード（ファイル変更なし）
- Git管理ファイルのみ編集対象
- ランダムな単語置換に限定

## 4. エラー修正機能

### OpenCode
**LSP統合による高度な診断**:
- マルチ言語LSPサーバー対応（gopls、tsserver、rust-analyzer、pyright）
- リアルタイム診断情報収集
- CodeAction による自動修正提案
- WorkspaceEdit適用による安全な修正

**診断情報フォーマット**:
```
Error: /path/to/file.go:15:10 [gopls][undefined] undefined: variableName
Warn: /path/to/file.go:20:5 [gopls][unused] variable declared but not used
```

### Gemini CLI
**EditCorrector システム**:
- LLM出力の自動修正
- エスケープシーケンス正規化
- old_string/new_string の自動調整
- LRUキャッシュによる修正結果保存

**高度なリトライメカニズム**:
- 指数バックオフ + ジッター
- インテリジェントエラー分類
- 認証フォールバック（Flash モデルへの切り替え）

### cat-code
- 基本的なファイル編集のみ
- エラー修正機能は実装されていない

## 5. 実装されているツール

### OpenCode (11ツール)
**ファイル操作**: view、edit、write、patch
**検索**: ls、glob、grep
**実行**: bash
**診断**: diagnostics
**外部**: fetch、sourcegraph

### Gemini CLI (14ツール)
**コアツール**: edit、read_file、write_file、run_shell_command、glob、search_file_content、list_directory、google_web_search、web_fetch、save_memory
**拡張**: Discovery Tools、MCP Tools

### cat-code
- ファイル編集機能のみ（FileEditor クラス）

## 6. 特徴的な機能比較

| 機能 | OpenCode | Gemini CLI | cat-code |
|------|----------|------------|----------|
| 言語 | Go | TypeScript/Node.js | TypeScript |
| UI | Bubble Tea TUI | React/Ink | React/Ink |
| LSP統合 | ✓ | ✗ | ✗ |
| サンドボックス | ✗ | ✓ | ✗ |
| MCP対応 | ✓ | ✓ | ✗ |
| 履歴圧縮 | ✗ | ✓ | ✗ |
| マルチプロバイダー | ✓ (11種) | ✗ (Geminiのみ) | ✗ |
| エラー自動修正 | LSPベース | EditCorrector | ✗ |
| テーマ | ✓ (10種) | ✓ | ✗ |
| 非対話モード | ✓ | ✗ | ✗ |

## 7. アーキテクチャの特徴

### OpenCode
- **クリーンアーキテクチャ**: internal パッケージによる明確な責任分離
- **依存性注入**: インターフェースベースの設計
- **並行処理**: goroutineによる効率的な非同期処理
- **イベント駆動**: PubSubパターンによるコンポーネント間通信

### Gemini CLI
- **モノレポ構造**: coreとcliの明確な分離
- **Reactパラダイム**: コンポーネントベースのUI設計
- **ストリーミング対応**: リアルタイム応答処理
- **プラグイン可能**: MCPによる拡張性

### cat-code
- **シンプル設計**: 最小限の依存関係
- **単一責任**: ファイル編集に特化
- **軽量実装**: 高速起動と低メモリ使用

## まとめ

各実装は異なるアプローチと強みを持っています：

- **OpenCode**: エンタープライズグレードの本格的な開発支援ツール。LSP統合と多言語対応が特徴。
- **Gemini CLI**: Google AIとの深い統合、優れたエラー修正機能、サンドボックス環境での安全な実行。
- **cat-code**: 教育的でシンプルな実装。コーディングエージェントの基本概念を理解するのに最適。

選択は用途と要件によって異なりますが、本格的な開発にはOpenCodeまたはGemini CLI、学習や実験にはcat-codeが適しています。