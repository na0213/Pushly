# Pushly 実装プロンプト集

通知アプリ「Pushly」を Claude Code / Codex を使って Xcode + Swift + SwiftData で実装するための、段階的なプロンプトと手作業の指示。

要件定義は `Notify_要件定義書_v0.3.docx` を参照。

---

## 全体の進め方

各フェーズはこの順序：

1. **【手作業】** あなたが Xcode や設定で行う作業
2. **【プロンプト】** Claude Code または Codex に投げるプロンプト
3. **【手作業】** ビルド・実機確認・コミット

各フェーズ完了後に **必ず実機 or シミュレータで動作確認**してから次に進むこと。動かないまま次を積むと修正が雪だるま式になる。

---

## 用語の使い分け

- **Claude Code**：このプロジェクトのコード生成と修正の主担当
- **Codex**：補助的なコード生成（必要に応じて）
- **「あなた」**：ユーザー（あなた）が手を動かす作業

---

# Phase 0: プロジェクトのセットアップ

## 0-1. 【手作業】Xcode で新規プロジェクト作成

1. Xcode を起動 → **File → New → Project**
2. **iOS → App** を選択
3. 以下を入力：
   - Product Name: `Pushly`
   - Team: あなたの Apple Developer アカウント
   - Organization Identifier: 任意の逆ドメイン（例：`com.yourname`）
   - Interface: **SwiftUI**
   - Language: **Swift**
   - Storage: **SwiftData** ← 必ずチェック
   - Include Tests: お好み
4. 保存場所を選んで Create

## 0-2. 【手作業】Deployment Target を 17.0 に

1. プロジェクトのルートを選択 → **Pushly** ターゲット → **General** タブ
2. **Minimum Deployments → iOS** を **17.0** に変更

## 0-3. 【手作業】Capabilities の設定（後回しでOK）

**注意：このステップは Apple Developer Program (年12,800円) の契約後に行います。**

開発初期は無料の Personal Team で十分に進められます。シミュレータでも実機でも、ローカル通知やデータ保存は動作します。

iCloud 同期と Push Notifications の有効化は、リリースが見えてきた Phase 9 直前に契約してから行います。それまでは以下の代替方針で進めます：

- **iCloud 同期なし**で開発する（SwiftData はローカル保存のみ）
- ローカル通知は使える（`UNUserNotificationCenter` は無料で動作）

### 契約後に行う作業（Phase 9 直前で実施）

1. https://developer.apple.com/programs/ から Apple Developer Program に登録（年12,800円）
2. ターゲットの **Signing & Capabilities** タブ
3. **+ Capability** で以下を追加：
   - **iCloud**
     - Services の **CloudKit** にチェック
     - Containers の **+** で新規コンテナを作成（`iCloud.com.natomi.Pushly` 形式）
   - **Background Modes**
     - **Remote notifications** にチェック
   - **Push Notifications**
4. PushlyApp.swift の `ModelConfiguration` に `cloudKitDatabase: .private(.default)` を追加（Phase 1 で書いてあるが、契約前はこの行をコメントアウトしておく）

## 0-4. 【手作業】Git の初期化

1. Xcode 内のターミナル、または Terminal.app で：
```bash
cd /path/to/Pushly
git init
git add .
git commit -m "Initial Xcode project setup"
```

## 0-5. 【プロンプト】プロジェクト構造の整理

Claude Code に以下を投げる：

```
Pushly という iOS アプリのプロジェクトを開いています。
SwiftUI + SwiftData + iOS 17 がベースです。

以下のフォルダ構造を作って、空のプレースホルダファイルを配置してください。

Pushly/
├── PushlyApp.swift           (既存、後で編集)
├── ContentView.swift          (既存、後で RootView に置き換え予定)
├── Models/
│   └── (空)
├── Views/
│   ├── Root/
│   ├── Home/
│   ├── Templates/
│   ├── Events/
│   ├── History/
│   ├── Settings/
│   └── Shared/
├── Helpers/
│   ├── DesignSystem.swift     (空ファイル)
│   ├── DateExtensions.swift   (空ファイル)
│   └── NotificationScheduler.swift  (空ファイル)
└── Preview Content/           (既存)

Xcode のグループとファイルシステム両方に反映してください。
ContentView.swift と PushlyApp.swift はまだ編集しないでください。
```

## 0-6. 【手作業】ビルド確認 → コミット

- ⌘B でビルドが通ることを確認
- `git add . && git commit -m "Setup folder structure"`

---

# Phase 1: データモデルの実装

## 1-1. 【プロンプト】SwiftData モデル

Claude Code に以下を投げる：

```
Pushly のデータモデルを実装します。
CloudKit 同期対応で、以下の制約を厳守してください：

- 全プロパティにデフォルト値を設定
- @Attribute(.unique) は使わない
- リレーションは必ず optional 型
- リレーションには inverse を必ず指定

Models/ ディレクトリに、以下の SwiftData モデルを作成してください。

## Models/Category.swift

@Model final class Category {
  - id: UUID (デフォルト UUID())
  - name: String (デフォルト "")
  - colorHex: String (デフォルト "#6b8fb8")   // hexで色を保存
  - createdAt: Date (デフォルト .now)
  - sortOrder: Int (デフォルト 0)
  - templates: [Template]? (リレーション、inverse: \Template.category)
  - events: [Event]? (リレーション、inverse: \Event.category)
  - init(name: String, colorHex: String)

## Models/Template.swift

@Model final class Template {
  - id: UUID
  - name: String
  - createdAt: Date
  - sortOrder: Int (デフォルト 0)
  - isArchived: Bool (デフォルト false)
  - isOneShot: Bool (デフォルト false)
  - category: Category?  (optional)
  - notificationRules: [NotificationRule]? (cascade削除、inverse: \NotificationRule.template)
  - events: [Event]? (cascade削除、inverse: \Event.template)  // 単発予定削除に対応するためcascade
  - init(name: String, category: Category?, isOneShot: Bool = false)

## Models/NotificationRule.swift

enum NotificationTiming: String, Codable, CaseIterable {
  case sameDay, dayBefore, weekBefore, monthBefore, custom

  var label: String { 当日/1日前/1週間前/1ヶ月前/カスタム }
  func daysBefore(custom: Int?) -> Int  // .custom時はcustom値、それ以外は 0/1/7/30
}

@Model final class NotificationRule {
  - id: UUID
  - timingRaw: String (デフォルト NotificationTiming.dayBefore.rawValue)
  - customDaysBefore: Int? (nil)
  - message: String ("")
  - isEnabled: Bool (true)
  - template: Template?
  - var timing: NotificationTiming  (computed、timingRaw のラッパー)
  - init(timing: NotificationTiming, message: String, customDaysBefore: Int?)

## Models/Event.swift

enum EventStatus: String, Codable {
  case scheduled, completed
}

@Model final class Event {
  - id: UUID
  - name: String  // テンプレ作成時の名前を引き継ぐ(後で変わっても影響しないように)
  - scheduledDate: Date
  - memo: String ("")
  - urlString: String ("")
  - amount: Decimal? (nil)
  - statusRaw: String (EventStatus.scheduled.rawValue)
  - completedAt: Date? (nil)
  - followUpMonths: Int? (nil)
  - followUpFired: Bool (false)
  - category: Category? (optional、最初に作成したテンプレのカテゴリをコピー)
  - template: Template?
  - pendingNotificationIDs: [String] ([])
  - var status: EventStatus (computed)
  - init(name: String, category: Category?, scheduledDate: Date, template: Template?)

すべて Swift 5.9+ / SwiftData の最新構文で書いてください。
import Foundation, import SwiftData を忘れずに。
```

## 1-2. 【プロンプト】PushlyApp.swift の更新

```
Pushly/PushlyApp.swift を以下のように書き換えてください。

- ModelContainer を作る
- Schema に [Category, Template, NotificationRule, Event] を渡す
- ModelConfiguration の cloudKitDatabase オプションについて：
  - 開発初期は Apple Developer Program 未契約なのでコメントアウトしておく
  - Phase 9 直前に契約後、`cloudKitDatabase: .private(.default)` をアンコメントする
  - コードコメントで「// Phase 9: Apple Developer Program 契約後にアンコメント」と書く
- WindowGroup { ContentView() } はそのまま

エラーハンドリングは fatalError で OK(MVPなので)。

最終的なコード例:

```swift
import SwiftUI
import SwiftData

@main
struct PushlyApp: App {
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([
            Category.self,
            Template.self,
            NotificationRule.self,
            Event.self,
        ])
        let config = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false
            // Phase 9: Apple Developer Program 契約後にアンコメント
            // cloudKitDatabase: .private(.default)
        )
        do {
            return try ModelContainer(for: schema, configurations: [config])
        } catch {
            fatalError("ModelContainer init failed: \(error)")
        }
    }()

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(sharedModelContainer)
    }
}
```
```

## 1-3. 【手作業】ビルド確認

- ⌘B でビルド
- エラーが出たら Claude Code に貼り付けて修正してもらう
- 通ったら `git commit -m "Add SwiftData models"`

---

# Phase 2: デザインシステム

## 2-1. 【プロンプト】DesignSystem.swift

```
Pushly のデザインシステムを Helpers/DesignSystem.swift に実装してください。

プロトタイプは pushly-prototype-v6.html を参照。
カラーは「オフホワイト〜ピーチ〜ピンク〜ブルー」の柔らかなグラデーション系。
彩度を抑え、ニュートラルなトーンで統一する。

## カラー
enum Palette として以下を定義(全てColor型):

- bgStart:     Color(red: 0.961, green: 0.929, blue: 0.894)  // #f5ede4 ペールピーチ
- bgMid:       Color(red: 0.941, green: 0.863, blue: 0.878)  // #f0dce0 ペールピンク
- bgEnd:       Color(red: 0.863, green: 0.894, blue: 0.933)  // #dce4ee ペールブルー
- surface:     Color.white
- ink:         Color(red: 0.122, green: 0.122, blue: 0.141)  // #1f1f24
- inkSoft:     Color(red: 0.420, green: 0.420, blue: 0.470)  // #6b6b78
- inkFaint:    Color(red: 0.647, green: 0.647, blue: 0.690)  // #a5a5b0
- line:        Color.black.opacity(0.06)
- lineStrong:  Color.black.opacity(0.12)
- premium:     Color(red: 0.769, green: 0.451, blue: 0.541)  // #c4738a ダスティローズ
- accentWarm:  Color(red: 0.851, green: 0.604, blue: 0.478)  // #d99a7a
- accentCool:  Color(red: 0.541, green: 0.643, blue: 0.769)  // #8aa4c4

## 背景 View
struct AppBackground: View
- ZStack で以下を重ねる:
  - 一番下: LinearGradient (bgStart → bgMid → bgEnd を上から下に、stops: 0/0.45/1.0)
  - 中段: RadialGradient (中央上に bgStart、ぼかし大きめ)
  - 上段: RadialGradient (右中央付近に bgMid)
  - 下段: RadialGradient (左下に bgEnd)
- ignoresSafeArea()

## カードのモディファイア
- .cardStyle(): 白カード(角丸20、影 y:10 radius:30 opacity 0.08 inkベース、line border)
- .darkCardStyle(): ダークカード(背景 ink、文字白、角丸24、内側パディング多め)

## ボタンスタイル
- PrimaryButtonStyle (背景 ink、白文字、角丸14、フォント 15pt semibold)
- SecondaryButtonStyle (背景 .white、ink文字、border lineStrong)

## カテゴリ色のヘルパー
let CATEGORY_COLOR_PALETTE: [String] = [
  "#d96b52", "#e8a23d", "#c4a256", "#8aa66a",
  "#5fa0a8", "#6b8fb8", "#8d7bc4", "#b56fae",
  "#d97a9b", "#8d8d95", "#5a6d75", "#3d4047",
  "#a0856e", "#7a9474", "#c08a5a", "#9b6fa8"
]

func categoryColor(from hex: String) -> Color
- "#RRGGBB" を Color に変換するユーティリティ

## カウントダウン用フォント拡張
extension Font:
- displayLg: 30pt / weight(.bold) / .system
- displayMd: 22pt / weight(.semibold)
- countdown: 72pt / weight(.bold)
- eyebrow:   10pt / weight(.medium) / 字間を広く取る用

## 注意点
- 紫の強い色は使わない。premium は紫ではなくダスティローズ(#c4738a)を使用
- 全体的に彩度を抑え、暖色から寒色へ移ろうトーンを意識
- SwiftUI の Color(red:green:blue:) は0〜1の値を取る点に注意
```

## 2-2. 【プロンプト】DateExtensions.swift

```
Helpers/DateExtensions.swift に Date のヘルパーを追加してください。

extension Date {
  func formattedYMD() -> String  // "2026.05.19"
  func formattedYM() -> String   // "2026.05"
  var daysFromNow: Int           // 今日からの差(過去はマイナス)
  func startOfDay() -> Date
}
```

## 2-3. 【手作業】ビルド確認 → コミット

`git commit -m "Add design system and date helpers"`

---

# Phase 3: ルート画面とタブナビゲーション

## 3-1. 【プロンプト】RootView と空のタブ画面

```
Pushly のルート画面とタブナビゲーションを作ります。

## Views/Root/RootView.swift
TabView ベースで3タブ:
- HomeView (予定タブ、アイコン: "bell")
- TemplatesView (テンプレタブ、アイコン: "square.stack")
- HistoryView (履歴タブ、アイコン: "archivebox")

タブのアクセントカラーは Palette.ink。

## Views/Home/HomeView.swift (空のプレースホルダ)
NavigationStack + AppBackground を背景に、
"予定" というタイトルだけ表示する仮実装。

## Views/Templates/TemplatesView.swift (同上、タイトル: テンプレート)
## Views/History/HistoryView.swift (同上、タイトル: 履歴)

## ContentView.swift を削除して、RootView を PushlyApp の WindowGroup に渡すよう変更。

すべて iOS 17 / SwiftUI で実装してください。
```

## 3-2. 【手作業】シミュレータで起動確認

- ⌘R でシミュレータ起動
- 3タブが切り替えられること、ラベンダー背景が表示されることを確認
- `git commit -m "Add root view with 3 tabs"`

---

# Phase 4: テンプレート機能の実装

これが Pushly のコア機能の起点。テンプレートが作れないと予定も作れない。

## 4-1. 【プロンプト】デフォルトカテゴリの seed

```
Pushly に初回起動時のデフォルトカテゴリを用意します。

Helpers/SeedData.swift を作成してください。

func seedDefaultCategoriesIfNeeded(context: ModelContext)
- 既存の Category が0件なら以下を挿入:
  - 免許    #6b8fb8
  - サブスク #8d7bc4
  - 健康    #8aa66a
  - 車      #3d4047
  - その他  #a0856e
- sortOrder を 0,1,2,3,4 で設定

PushlyApp.swift の中で、ModelContainer 作成直後に
container.mainContext を使って seedDefaultCategoriesIfNeeded を呼んでください。
```

## 4-2. 【プロンプト】カテゴリ管理シート

```
Views/Templates/CategoryManagerView.swift を作成。
sheet として表示する想定。

機能:
- 既存カテゴリの一覧表示(カラードット + 名前 + 使用件数)
- 使用件数0のカテゴリは「削除」ボタンを表示
- 新規カテゴリ追加:
  - 名前のテキスト入力
  - 16色のパレットから1つ選択(円形のスウォッチ8x2グリッド)
  - 「追加」ボタンで Category を作成

16色のパレット定数を Helpers/DesignSystem.swift に追加:
let CATEGORY_COLOR_PALETTE: [String] = [
  "#d96b52", "#e8a23d", "#c4a256", "#8aa66a",
  "#5fa0a8", "#6b8fb8", "#8d7bc4", "#b56fae",
  "#d97a9b", "#8d8d95", "#5a6d75", "#3d4047",
  "#a0856e", "#7a9474", "#c08a5a", "#9b6fa8"
]

カテゴリの使用件数は Template と Event 両方の categoryId で集計。
SwiftData の @Query を使う。
```

## 4-3. 【プロンプト】テンプレート一覧（並び替え・アーカイブ対応）

```
Views/Templates/TemplatesView.swift を本実装してください。

機能:
- @Query で Template を以下の条件で取得:
  - isOneShot == false かつ isArchived == false のみ
  - sortOrder 昇順
- 右上の + ボタンで TemplateEditorView をシートで開く(新規作成)
- 「カテゴリを管理」ボタンで CategoryManagerView をシートで開く
- "アーカイブ済み" ボタンで ArchivedTemplatesView を NavigationLink で開く
- テンプレート一覧は List で表示し、TemplateRowView でカード表示:
  - 左端にカテゴリ色のマーカー(縦バー、4x40pt)
  - カテゴリ名(eyebrow)
  - 予定名(displayMd)
  - 有効な通知ルールの件数 "N 件の通知"
  - 右端に chevron

並び替え機能:
- List + .onMove modifier で実装
- 並び替えモードに入るための EditButton を navigation bar に配置
- 並び替え完了時に各 Template の sortOrder を 0 から振り直して保存

スワイプアクション:
- 左スワイプで "アーカイブ" ボタン (灰色)
  - タップで template.isArchived = true、save
- 左スワイプを更に深くスワイプ または "削除" ボタン(赤)
  - 確認アラート: "完全に削除しますか？このテンプレートに紐づく全ての履歴も削除されます。"
  - OK で context.delete(template)、save
  - (cascade で events と rules も削除される)

行タップ:
- TemplateEditorView (編集モード) を NavigationLink で開く

空状態のメッセージ:
  "テンプレートを作成しましょう"
  "免許更新、健康診断など、繰り返し発生する予定の通知設定を作っておけます"
```

## 4-3b. 【プロンプト】アーカイブ済みテンプレート画面

```
Views/Templates/ArchivedTemplatesView.swift を実装してください。

機能:
- @Query で Template を isOneShot == false かつ isArchived == true で取得
- sortOrder 順
- ヘッダ: "アーカイブ済み" タイトル + "アーカイブされたテンプレートは新規予定作成時には表示されません。" のサブテキスト
- 各行は TemplateRowView (淡くグレーアウトしたスタイル)
- スワイプアクション:
  - "復元" (緑): template.isArchived = false
  - "削除" (赤): 確認アラート → context.delete(template)
- 空状態: "アーカイブされたテンプレートはありません"

アーカイブされたテンプレートの行は、通常のテンプレ行よりも薄く表示する(opacity 0.7など)。
```

## 4-4. 【プロンプト】テンプレート編集画面

```
Views/Templates/TemplateEditorView.swift を実装してください。

引数:
- template: Template? (nilなら新規、それ以外は編集)

機能:
- 予定名のテキスト入力(必須)
- カテゴリ選択 chip(横スクロール、選択中はinkに反転)、末尾に「+ 追加」chipで CategoryManagerView を開く
- 通知タイミング5種類(NotificationTiming.allCases):
  - チェックボックスでオン/オフ
  - オン時のみメッセージ入力欄(複数行)を展開
  - .custom の場合は日数(1〜365)のステッパー表示
- デフォルト値:
  - 新規: dayBefore と weekBefore のみオン
  - 編集: 既存の NotificationRule から復元
- 保存ボタンで:
  - 編集時は既存 NotificationRule を全削除して作り直し
  - 新規時は Template + NotificationRule を挿入
  - 新規作成時の sortOrder は、現在の Templates の最大 sortOrder + 1 を割り当てる(@Query で取得)
  - 予定名 or カテゴリが未入力なら保存ボタンを disabled に
- 上部に「キャンセル」ボタン(navigation bar leading)

NotificationTiming のデフォルトメッセージ:
- sameDay: "今日が予定日です"
- dayBefore: "明日が予定日です"
- weekBefore: "1週間後に予定があります"
- monthBefore: "1ヶ月後に予定があります"
- custom: ""

ModelContext は @Environment(\.modelContext) で取得。
```

## 4-5. 【手作業】テスト

- シミュレータで起動
- テンプレートタブ → +ボタン → テンプレ作成 → 一覧に出る
- カテゴリ管理 → 新規追加 → 編集画面で選べる
- テンプレタップ → 編集 → 保存できる
- EditButton をタップして並び替えモード → ドラッグで順序変更 → 抜けて再度開いても順序が保持される
- 左スワイプで「アーカイブ」 → 一覧から消える
- 「アーカイブ済み」リンクから ArchivedTemplatesView を開き、復元できる
- 「アーカイブ済み」から削除 → 確認アラート後に消える
- `git commit -m "Implement template management with reorder/archive/delete"`

---

# Phase 5: 予定(Event)の作成と表示

## 5-1. 【プロンプト】予定編集画面

```
Views/Events/EventEditorView.swift を実装してください。

引数:
- template: Template
- suggestedDate: Date? (nilなら30日後をデフォルト)
- onSaved: (() -> Void)? (保存後のコールバック)

機能:
- ヘッダにテンプレのカテゴリ(eyebrow) + 予定名(displayLg) を表示(編集不可)
- カレンダー(DatePicker .graphical)で予定日選択
  - 初期値: suggestedDate ?? 30日後
- メモ(TextField axis: .vertical)
- URL(TextField、autocapitalization off)
- 金額(数字入力、左に "¥")
- 通知プレビュー(テンプレの enabled rules を読み取り専用で並べる)
- 「予定を作成」ボタンで Event を作成して保存

通知のスケジューリングは Phase 6 で実装するので、ここでは Event の保存だけ。
保存後に onSaved を呼ぶ。

カテゴリは template.category をそのまま使う(コピーではなく参照)。
```

## 5-2. 【プロンプト】予定の作成方法選択シート

```
Views/Events/CreateMethodPickerView.swift を実装してください。
ホームの + ボタンから sheet で開く想定。

機能:
シートの最上段に大きめのタイトル「予定を作成」を表示。
その下に2つの選択カード(OptionCard、CompletionSheet の3択カードと同じデザイン)を縦並びで配置:

1. テンプレートから作成
   "繰り返し発生する予定（免許更新、健康診断、サブスクなど）"
   タップで TemplatePickerView をシート遷移(またはNavigationLink)

2. 単発で作成
   "ライブのチケット、予約日など、一度きりの予定"
   タップで OneShotEventEditorView を遷移

両方とも、その先の編集画面で「予定を作成」したら、このCreateMethodPickerViewごとdismissする。
```

## 5-2b. 【プロンプト】テンプレートピッカー

```
Views/Events/TemplatePickerView.swift を実装してください。

機能:
- @Query で Template を取得(isOneShot==false かつ isArchived==false のみ、sortOrder 昇順)
- "どのテンプレートを使いますか？" のコピー
- TemplateRowView(再利用)を縦に並べる
- 行タップで EventEditorView を NavigationLink で push
- EventEditorView の onSaved でルートのシートを dismiss する仕組み(クロージャを上位から渡す)
- テンプレが0件なら "先にテンプレートを作成してください" と表示

NavigationStack の中で動作する想定。
```

## 5-2c. 【プロンプト】単発予定作成画面

```
Views/Events/OneShotEventEditorView.swift を実装してください。
1画面で「テンプレート相当の入力」と「予定の入力」を両方できるようにします。

機能(セクション順):

1. ヘッダ:
   eyebrow "one-shot"
   h1 "単発の予定"

2. 予定名(必須、テキスト入力)

3. カテゴリ選択(chips、TemplateEditorView と同じ。+追加で CategoryManagerView)

4. 通知タイミング(TemplateEditorView と同じ、5種類のチェックボックス+メッセージ)

5. 予定日(DatePicker .graphical)
   初期値: 7日後

6. メモ(複数行)、URL、金額(EventEditorView と同じUI)

7. 「予定を作成」ボタン:
   保存処理は以下の順:
   a. 内部用に Template を新規作成: isOneShot = true, isArchived = false, name=入力値, category=選択値
   b. その template に NotificationRule を有効なものだけ作成
   c. その template の Event を作成、scheduledDate と詳細を保存
   d. context.save()
   e. await NotificationScheduler.shared.scheduleNotifications(for: event) (Phase 6 で組み込む)
   f. onSaved 呼び出し → ルートのシートを dismiss

上部に「キャンセル」ボタン。
予定名 or カテゴリが未入力 or 通知タイミングが0個なら保存ボタン disabled。
```

## 5-3. 【プロンプト】ホーム画面の本実装

```
Views/Home/HomeView.swift を本実装してください。

機能:
- @Query で status=scheduled の Event を scheduledDate 昇順で取得
- 上部: カテゴリフィルタータブ(横スクロール chip)
  - 先頭は「すべて」、以下 Category を sortOrder 順
  - アクティブな chip は inkBg + 白文字
  - chip にはカテゴリ色のドット
- 右上の + ボタンで CreateMethodPickerView を sheet 表示
- 予定カード EventCardView:
  - 一番上(最近) は darkCard(背景 #14141a、文字白)、それ以外は白カード
  - darkCard には右上にカテゴリ色のブロブ(blur radius 60、opacity 0.5)
  - 内容:
    - カテゴリピル(カラードット + カテゴリ名 大文字)
    - 予定名 22pt semibold
    - 予定日 12pt inkSoft
    - カウントダウン: ラベル "DAYS LEFT/TODAY/DAYS PAST" + 数字 72pt + "days"
- カードタップで EventDetailView へ NavigationLink

空状態: "予定はありません" "右上の + から、新しい予定を作成できます。"
```

## 5-4. 【プロンプト】予定詳細画面(進行中)

```
Views/Events/EventDetailView.swift を実装してください。

引数: event: Event (@Bindable)

機能:
- ヒーローカード(ダーク、角丸24):
  - 右上にカテゴリ色のブロブ
  - カテゴリピル
  - 予定名 24pt
  - 予定日
  - カウントダウン (大きい数字 80pt)
- 詳細カード(白カード、横並びの label/value):
  - メモ(あれば)
  - URL(あれば、Linkとして表示)
  - 金額(あれば、24pt bold で表示)
- 「予定を終了する」ボタン (primary)
  - タップで CompletionSheet を sheet として表示

予定終了の3択フローは次の 5-5 で実装。
```

## 5-5. 【プロンプト】予定終了の3択シート

```
Views/Events/CompletionSheet.swift を実装してください。

引数:
- event: Event
- onClose: () -> Void

機能:
シートの中身(presentationDetents([.medium, .large])):

タイトル:
  eyebrow "complete"
  h1 "この予定をどうしますか？"

3つの OptionCard を縦に並べる。各カード:
- 左に番号(01, 02, 03、premium色 #6b5fb8)
- タイトル
- 説明
- 右に chevron

01. 次回の予定を作成する
   "同じテンプレートで、新しい予定日を設定します"
   タップ動作:
     event.status = .completed
     event.completedAt = .now
     try? context.save()
     onClose()
     親側で EventEditorView を push する(下記参照)

【重要】event.template?.isOneShot == true の場合、この選択肢「01」は非表示にする。
単発予定はそもそも繰り返しを想定していないため、「次回作成」フローを提供しない。
表示するのは「02 あとでリマインドする」「03 何もせず終了」のみ。

02. あとでリマインドする
   "「前回から〇ヶ月たちました」と通知します"
   この下に StepperRow:
     "N ヶ月後"(状態 followupMonths、初期12、範囲1〜36、+/-ボタン)
   タップ動作:
     event.followUpMonths = followupMonths
     event.status = .completed
     event.completedAt = .now
     try? context.save()
     (通知スケジュールは Phase 6 で実装、今は TODO コメントで)
     onClose()

03. 何もせず終了
   "履歴に保存して通知はしません"
   タップ動作:
     event.status = .completed
     event.completedAt = .now
     try? context.save()
     onClose()

【重要】01 を選んだ場合、シートを閉じた後にユーザーが日付を選べる
予定編集画面に遷移する。日付の初期値は前回日付の1年後。
このため CompletionSheet では「どの選択肢を選んだか」を外に伝える仕組みが必要。

CompletionChoice という enum を作って:
  case createNext  // 次回作成
  case followUp(months: Int)
  case justFinish

onClose の引数として CompletionChoice を渡すように設計。
EventDetailView 側で choice を受け取り、createNext なら EventEditorView を push する。
```

## 5-6. 【手作業】テスト

- ホームの + ボタン → 「テンプレートから作成」/「単発で作成」の2択シートが表示される
- 「テンプレートから作成」を選ぶ → テンプレ選択 → 予定編集 → 保存 → ホームに出る
- 「単発で作成」を選ぶ → 全項目を入力 → 保存 → ホームに出る
- テンプレートタブを開き、単発予定で作ったテンプレが表示されていないことを確認(isOneShot=trueなので)
- 予定タップ → 詳細表示 → 終了ボタン
- 「①次回の予定を作成」を選ぶと、編集画面が開き、1年後がデフォルト表示されることを確認
- 「②」「③」も動作確認
- 注意: 単発予定で終了ボタンを押した場合、「①次回作成」を選ぶとどう振る舞うべきか後で検討(現状はテンプレが内部に存在するので動作はするが、ユーザー体験的に違和感がある可能性あり)
- `git commit -m "Implement event creation with template/one-shot modes"`

---

# Phase 6: 通知のスケジューリング

## 6-1. 【プロンプト】NotificationScheduler 実装

```
Helpers/NotificationScheduler.swift を実装してください。

import UserNotifications

@MainActor
final class NotificationScheduler {
  static let shared = NotificationScheduler()

  // 通知許可リクエスト
  func requestAuthorization() async throws -> Bool

  // 予定作成/編集時に呼ぶ
  // 既存の通知IDをキャンセルしてから、テンプレの enabled rules で再スケジュール
  func scheduleNotifications(for event: Event) async

  // 予定終了/削除時に呼ぶ
  func cancelNotifications(for event: Event)

  // ② "前回から〇ヶ月後"通知
  func scheduleFollowUp(for event: Event, months: Int) async

  // 未送信通知の最大数チェック(64件制限)
  func currentPendingCount() async -> Int
}

仕様:
- NotificationRule の timing と customDaysBefore から発火日を計算
- 発火時刻は予定日の朝9時(scheduledDate の時間を 9:00 に置き換え)
- 通知IDは "event-{event.id}-rule-{rule.id}" 形式
- 過去の日付になる通知はスケジュールしない
- event.pendingNotificationIDs に IDの配列を保存

scheduleFollowUp:
- 通知ID: "followup-{event.id}"
- 発火日: event.scheduledDate から months ヶ月後の朝9時
- 本文: "前回の{event.name}から{months}ヶ月たちました。予定をたてますか？"
- event.pendingNotificationIDs にも追加
```

## 6-2. 【プロンプト】通知許可とスケジュール呼び出しの組み込み

```
以下の呼び出しを既存コードに追加してください。

1. PushlyApp.swift の初回起動時:
   - ContentView() の .task で
     NotificationScheduler.shared.requestAuthorization() を呼ぶ
   - 失敗してもアプリは動かす(通知だけ使えなくなる)

2. EventEditorView の保存処理の末尾:
   try? context.save() の後に
   await NotificationScheduler.shared.scheduleNotifications(for: newEvent)

3. CompletionSheet の各選択肢:
   - すべての選択肢で NotificationScheduler.shared.cancelNotifications(for: event)
   - 02 (followUp) では追加で
     await NotificationScheduler.shared.scheduleFollowUp(for: event, months: months)

4. 予定の削除処理を Phase X で作る場合も cancelNotifications を必ず呼ぶ
```

## 6-3. 【手作業】実機テスト(必須)

シミュレータでは通知のスケジューリング動作が不安定なので、**実機で確認**するのが望ましい。

1. iPhone を Mac に接続
2. Xcode で実機を選択して ⌘R
3. アプリ起動時に通知許可ダイアログが出ること
4. 1分後の日付で予定を作って「カスタム 0日前」で通知設定、本当に届くか確認
5. 終了ボタンで通知がキャンセルされること(設定 → 通知 → Pushly で確認)
6. `git commit -m "Implement notification scheduling"`

---

# Phase 7: 履歴機能(2モード)

## 7-1. 【プロンプト】履歴のタイムライン表示

```
Views/History/HistoryView.swift を本実装してください。
2モード切り替え(タイムライン / 予定別)の構造を入れます。

機能:
- 上部にモードスイッチャー(SegmentedControl 風の自作):
  - "タイムライン" / "予定別" の2ボタン、アクティブは inkBg
- その下にカテゴリフィルタータブ(HomeViewと同じ)
- @State で currentMode を保持
- モードによって表示する子ビューを切り替え

子ビュー1:
Views/History/HistoryTimelineView.swift
- @Query で status=completed を completedAt 降順で取得
- カテゴリでフィルター
- HistoryItemView でカード表示:
  - カテゴリドット + カテゴリ名 + 予定名 + 日付 + 金額(あれば)
  - followUpMonths があれば "Nヶ月後" タグ表示
  - タップで SeriesDetailView へ NavigationLink

子ビュー2: 7-2 で実装
```

## 7-2. 【プロンプト】予定別グループ表示

```
Views/History/HistoryGroupedView.swift を実装してください。

機能:
- @Query で status=completed を取得し、templateId で groupBy
- カテゴリフィルター(historyFilter) で絞り込み
- 各グループは GroupedCardView:
  - 上段: カテゴリマーカー + カテゴリ名 + 予定名
  - 下段(border-topで区切る): 横並びで以下を表示
    - 回数 (NUMBER 回)
    - 前回 (日付)
    - 累計 (金額、累計>0 のときだけ)
  - タップで SeriesDetailView へ NavigationLink

並び順: 各グループの最新の完了日で降順ソート。
表示対象: 1件のみのグループ(単発予定または1回しか実施していない通常テンプレ)も表示する。
特別扱い: template.isOneShot == true のグループには右上に小さなラベル "ONE-SHOT" を薄く表示(あくまでUIヒント、削除はしない)。
template が nil(削除済み) のグループは表示しない。
```

## 7-3. 【プロンプト】SeriesDetailView (予定別の詳細)

```
Views/History/SeriesDetailView.swift を実装してください。

引数: template: Template

機能:
- 対象 template の events をすべて取得(完了+進行中)
- 統計を計算:
  - 累計実施回数(completed のみ)
  - 平均間隔(2件以上 completed のとき、月単位の小数)
  - 累計金額、平均金額(金額入力ありのみ)
  - 次回予測(progressing がなく、completed が2件以上ある場合):
    最後の completed の date + 平均間隔 月

UI:
1. SeriesHeroCard(白カード、ブロブ装飾):
   - カテゴリピル
   - 予定名 28pt bold
   - 2x2 のステータスグリッド: 累計回数、平均間隔、累計金額、平均金額
     (金額がない場合は 2列のみ)

2. PredictionCard(progressing がない場合):
   - 紫グラデの背景(linear-gradient #6b5fb8 → #d97a9b 透明度10%)
   - "次回予測" "YYYY.MM 頃" "過去の平均間隔から算出"
   または進行中がある場合は:
   - "次回予定" "YYYY.MM.DD" "あと N 日"

3. AmountTrendChart(累計>0 かつ 2件以上 のとき):
   - ミニ折れ線(SVGではなく Swift Charts を使う)
   - X軸: scheduledDate、Y軸: amount
   - エリア塗り(premium色 → 透明)
   - 軸ラベル: 最初と最後の年月

4. Timeline(縦並び):
   - 全 event を scheduledDate 降順で表示
   - 各行: 円(dot) + 年 + 月日 + メモ + 金額 + "予定"バッジ(進行中なら)

5. NextActionButton(進行中の event が0件 かつ template.isOneShot == false のとき):
   - "次回の予定を作成" primary button
   - タップで EventEditorView を push、suggestedDate に予測日 or last+1年
   - isOneShot の場合はこのボタンを表示しない

6. 履歴アイテムの個別削除:
   - 各タイムラインエントリ(完了済み)に長押しメニューまたはスワイプアクションで「削除」
   - 削除時:
     - NotificationScheduler.shared.cancelNotifications(for: event) を呼ぶ
     - context.delete(event)
     - try? context.save()
     - もし event.template?.isOneShot == true で、そのテンプレに紐づく Event が0件になったら、テンプレも削除する
     - SeriesDetailView を表示している状態で全 event が削除されたら画面を戻す

iOS 17 の Swift Charts を import すること。
```

## 7-4. 【手作業】テスト

- 履歴データを増やすため、テスト用に複数の予定を作って終了させる
- タイムライン/予定別の切り替えが効くこと
- 予定別から SeriesDetailView へ遷移できること
- 統計が正しく計算されていること
- `git commit -m "Implement history with two view modes and series detail"`

---

# Phase 8: 細部の磨き

## 8-1. 【プロンプト】予定の編集と削除

```
EventDetailView に「編集」「削除」のメニューを追加してください。

navigation bar trailing に Menu(systemImage: "ellipsis") を配置し、

- 編集: EventEditorView を編集モードで開く(既存の event を渡す)
  EventEditorView を引数 event: Event? を受け取れるよう拡張
- 削除: 確認アラート表示 → 削除時は
  NotificationScheduler.shared.cancelNotifications(for: event)
  context.delete(event)
  もし event.template?.isOneShot == true で、そのテンプレに紐づく event が0件になった場合は、
  テンプレも一緒に削除する
  try? context.save()
  画面を戻す

(注: テンプレートのアーカイブと削除は Phase 4-3 で既に実装済み)
```

## 8-2. 【プロンプト】設定画面

```
Views/Settings/SettingsView.swift を作成してください。
タブバーには追加せず、ホーム画面の左上に歯車アイコンを置いて sheet で開く。

機能(MVPでは表示のみ):
- iCloud 同期の状態表示(現状の同期可能性は CKContainer の accountStatus を読む)
- 通知許可の状態表示(タップで設定アプリを開く)
- アプリのバージョン表示
- "プライバシーポリシー" "利用規約" のリンク(プレースホルダ)
- "お問い合わせ" メールリンク(プレースホルダ)

将来 Premium 課金が来たら、このページに「Premiumを購入」セクションを追加する。
```

## 8-3. 【手作業】アプリアイコン

1. SF Symbols や Figma で 1024x1024 のアイコンを作る(後で本格デザインに差し替える前提)
2. Assets.xcassets → AppIcon にドラッグ
3. ⌘R で確認

## 8-4. 【手作業】コミット

`git commit -m "Add settings, edit/delete actions, app icon"`

---

# Phase 9: 動作検証とリリース準備

## 9-0. 【手作業】Apple Developer Program の契約と iCloud 設定

ここまで Personal Team (無料) で開発してきましたが、リリースには Apple Developer Program (年12,800円) の契約が必要です。

### 9-0a. Apple Developer Program に登録

1. https://developer.apple.com/programs/ にアクセス
2. Apple ID でサインインして「Enroll」
3. 個人 (Individual) で登録、支払いを完了
4. 承認まで通常24〜48時間

### 9-0b. Xcode で Team を切り替え

1. Xcode → Settings → Accounts で Apple ID を確認
2. プロジェクトのターゲット → **Signing & Capabilities** タブ
3. **Team** を「Personal Team」から有料契約したチームに切り替え

### 9-0c. iCloud と通知の Capabilities を追加

1. **+ Capability** で以下を追加：
   - **iCloud**
     - Services の **CloudKit** にチェック
     - Containers の **+** で新規コンテナを作成（`iCloud.com.natomi.Pushly` のような形式）
   - **Background Modes**
     - **Remote notifications** にチェック
   - **Push Notifications**

### 9-0d. 【プロンプト】PushlyApp.swift の cloudKitDatabase をアンコメント

Claude Code に以下を投げる：

```
PushlyApp.swift の ModelConfiguration で、
コメントアウトしてある cloudKitDatabase: .private(.default) を
アンコメントしてください。
コメント "// Phase 9: ..." も削除して大丈夫です。
```

### 9-0e. 【手作業】ビルドとデータマイグレーション確認

- ⌘R で実機ビルド
- 既存のローカルデータが CloudKit へ初回同期されるのを待つ（数分〜数十分）
- 別の iPhone / iPad に同じ Apple ID でサインインしているなら、アプリインストール後にデータが同期されてくるか確認

---

## 9-1. 【手作業】動作検証チェックリスト

実機で以下をすべて確認:

- [ ] 初回起動でデフォルトカテゴリが作成される
- [ ] 通知許可ダイアログが出る
- [ ] テンプレ作成 → 予定作成(テンプレートから) → 通知が指定時刻に届く
- [ ] 単発予定の作成フローが動く(ライブのチケット等で試す)
- [ ] 単発予定で作ったテンプレが、テンプレート一覧に表示されないことを確認
- [ ] 予定を終了 → 3択それぞれが期待通り動く
- [ ] ①を選ぶと編集画面が開く(自動作成されない)
- [ ] 単発予定の終了時には「①次回作成」が表示されないことを確認
- [ ] ②を選ぶと N ヶ月後に通知が届く
- [ ] テンプレートの並び替えが効く(編集モード→ドラッグ→保持)
- [ ] テンプレートのアーカイブと復元が動く
- [ ] テンプレートの完全削除で履歴も消える(警告ダイアログ表示)
- [ ] 履歴のタイムライン/予定別の切り替えが効く
- [ ] 履歴から個別アイテムを削除できる
- [ ] 単発予定の履歴を削除すると、テンプレも自動削除される
- [ ] 予定別から詳細ページに遷移できる
- [ ] iCloud 同期: 別端末でログインすると同じデータが見える(時間がかかるので待つ)
- [ ] アプリを kill して再起動してもデータが保持される

## 9-2. 【手作業】App Store Connect の準備

- App Store Connect で新規アプリ登録
- バンドルID は Xcode で設定したものと合わせる
- スクリーンショット撮影(プロトタイプv4が参考になる)
- App Store 説明文の下書き
- プライバシーポリシー / 利用規約のサイトを用意

## 9-3. 【手作業】TestFlight 配信

- Xcode → Product → Archive
- Distribute App → App Store Connect → Upload
- App Store Connect で TestFlight に登録
- 自分や数人で1〜2週間ベータテスト

## 9-4. 【手作業】審査提出

- 不具合が落ち着いたら審査提出
- 初回審査は1〜3日かかる

---

# 将来の拡張 (v1.1 以降)

要件定義書のセクション9.2 に記載の機能:

- Premium 課金 (StoreKit 2)
- カスタムカテゴリ追加
- 家計タブ
- 予算アラート
- アプリアイコン変更
- ウィジェット

各機能は別ブランチで開発し、リリースごとに main にマージ。
このプロンプト集とは別に、v1.1 用のプロンプト集を作る。

---

# トラブルシューティング

## CloudKit のエラーが出る場合

- Capabilities の iCloud Container が正しく作成されているか確認
- SwiftData モデルの制約(全 default、optional リレーション、no unique)を再確認
- 開発中はデータをリセットしたいことが多い: Xcode → Window → Devices → アプリを削除 で初期化

## 通知が届かない

- 設定アプリ → 通知 → Pushly で許可状態確認
- スケジュール先の日付が過去になっていないか
- UNUserNotificationCenter.current().getPendingNotificationRequests で実際に登録されているか確認

## ビルドエラーが解決できない

エラー全文をコピーして Claude Code に貼り付ける。
SwiftData の API は iOS バージョンで微妙に違うので、エラーメッセージを正確に伝えるのが重要。

---

# 変更履歴

## v0.4 (このプロンプト)

- v0.3 からの変更:
  - Phase 2-1 のカラーパレットを刷新（紫基調 → ピーチ〜ピンク〜ブルーの柔らかなグラデーション）
  - プレミアムアクセントカラーを紫 (#6b5fb8) からダスティローズ (#c4738a) に変更
  - 暖色アクセント・寒色アクセントの2色を追加
  - 参照プロトタイプを v6 (pushly-prototype-v6.html) に更新

## v0.3

- v0.2 からの追加:
  - Apple Developer Program 契約は Phase 9 まで遅らせる方針に変更
  - Phase 0-3 を「後回しでOK」に書き換え
  - Phase 1-2 の PushlyApp.swift に「cloudKitDatabase は契約後にアンコメント」の注記
  - Phase 9-0 を新設: Apple Developer Program 登録、Capabilities 追加、iCloud 有効化の手順

## v0.2

- v0.1 からの追加:
  - Template に sortOrder / isArchived / isOneShot フィールド
  - テンプレート並び替え機能 (Phase 4-3)
  - テンプレートのアーカイブと復元、完全削除 (Phase 4-3, 4-3b)
  - 単発予定の作成フロー (Phase 5-2 / 5-2b / 5-2c)
  - 単発予定の終了時は「①次回作成」を非表示 (Phase 5-5)
  - 履歴のグループ表示で単発予定の特別扱い (Phase 7-2)
  - 履歴アイテムの個別削除 (Phase 7-3, 8-1)
  - 単発予定の履歴削除時、テンプレも自動削除

End of prompt.md
