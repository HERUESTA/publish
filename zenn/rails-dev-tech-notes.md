# Rails開発で押さえておきたい技術メモ

## Dockerfileとは

Dockerfileは、Dockerイメージを作成するための設計図（テキストファイル）

アプリケーションの実行環境を「コード」として定義できるので、誰でも同じ環境を再現することができる

```dockerfile
# 例: Ruby on Rails用のDockerfile
FROM ruby:3.2

WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```

**主要な命令:**

| 命令 | 役割 |
|------|------|
| `FROM` | ベースイメージを指定（土台となるOS・言語環境） |
| `WORKDIR` | コンテナ内の作業ディレクトリを設定 |
| `COPY` | ホストのファイルをコンテナにコピー |
| `RUN` | イメージビルド時にコマンドを実行 |
| `CMD` | コンテナ起動時に実行するコマンド |
| `EXPOSE` | 公開するポート番号を宣言 |
| `ENV` | 環境変数を設定 |

---

## DockerfileとDockerfile.devの違い（本番環境と開発環境の違い）

同じアプリでも、開発時と本番運用では求めるものが異なります

### Dockerfile（本番環境向け）

- **軽量・最適化** を重視
- 不要なファイル（テストコード、開発用gemなど）を含めない
- マルチステージビルド（1つのDockerfile内に複数のFROM命令を使って、ビルドを段階的に分ける手法 => 最終イメージのサイズを小さくする）でイメージサイズを削減
- `RAILS_ENV=production` で動作

```dockerfile
# 本番用: マルチステージビルドの例
FROM ruby:3.2-slim AS build
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

FROM ruby:3.2-slim
WORKDIR /app
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY . .
RUN bundle exec rails assets:precompile
CMD ["rails", "server", "-b", "0.0.0.0"]
```

### Dockerfile.dev（開発環境向け）

- **開発のしやすさ** を重視
- デバッグツールやdev用gemを含む
- ソースコードはボリュームマウントでホストと共有（コード変更が即反映）
- `RAILS_ENV=development` で動作

```dockerfile
# 開発用
FROM ruby:3.2
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install  # dev/test用gemも含む
COPY . .
CMD ["rails", "server", "-b", "0.0.0.0"]
```

### 比較まとめ

| 項目 | 本番（Dockerfile） | 開発（Dockerfile.dev） |
|------|---------------------|------------------------|
| イメージサイズ | 小さい（slim使用） | 大きくてもOK |
| デバッグツール | なし | あり（byebug等） |
| ソースコード | COPY でイメージに含める | ボリュームマウント |
| アセット | プリコンパイル済み | オンデマンド |
| 最適化 | する | しない |

---

## ActiveStorageとS3の関係性

### Active Storageとは

Active StorageはRailsに組み込まれたファイルアップロード機能です。モデルにファイルを簡単に添付できます。

```ruby
class User < ApplicationRecord
  has_one_attached :avatar       # 1つのファイル
  has_many_attached :photos      # 複数のファイル
end
```

### S3（Amazon S3）とは

Amazon S3はAWSが提供するクラウドストレージサービスです。大量のファイルを安全・安価に保存できます。

### 両者の関係

Active Storageは **「ファイルをどう扱うか」のインターフェース** で、S3は **「ファイルをどこに保存するか」の保存先** です。

```
ユーザー → Rails(Active Storage) → S3（保存先）
                                  → ローカルディスク（開発時）
                                  → GCS（Google Cloud Storage）
```

Active Storageの設定で保存先を切り替えられます:

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: ap-northeast-1
  bucket: my-app-bucket
```

```ruby
# config/environments/development.rb
config.active_storage.service = :local   # 開発時はローカルに保存

# config/environments/production.rb
config.active_storage.service = :amazon  # 本番はS3に保存
```

**なぜ本番でS3を使うのか:**

- サーバーのディスク容量を気にしなくていい
- 複数サーバー構成でもファイルを共有できる
- 高い耐久性（99.999999999%）でデータを守れる
- CDN（CloudFront）と組み合わせて高速配信できる

---

## Bootstrapと生のCSSの違い

### 生のCSS

HTMLの見た目を一からすべて自分で書くスタイルです。

```css
.btn {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.btn:hover {
  background-color: #0056b3;
}
```

**メリット:**
- 完全に自由なデザインが可能
- 不要なCSSが含まれないので軽量
- フレームワークの学習コストがない

**デメリット:**
- レスポンシブ対応を自分で実装する必要がある
- ブラウザ間の差異を自分で吸収する必要がある
- 開発に時間がかかる

### Bootstrap

Twitter社が開発したCSSフレームワークです。あらかじめ用意されたクラスを付けるだけでデザインできます。

```html
<button class="btn btn-primary">ボタン</button>
<div class="container">
  <div class="row">
    <div class="col-md-6">左カラム</div>
    <div class="col-md-6">右カラム</div>
  </div>
</div>
```

**メリット:**
- 素早くそれなりの見た目が作れる
- レスポンシブ対応が簡単（グリッドシステム）
- コンポーネント（ナビバー、モーダル等）が豊富

**デメリット:**
- 「Bootstrap感」のある見た目になりがち
- 不要なCSSも読み込まれるのでファイルサイズが大きい
- カスタマイズに手間がかかることがある

### 比較まとめ

| 項目 | 生のCSS | Bootstrap |
|------|---------|-----------|
| 開発速度 | 遅い | 速い |
| デザインの自由度 | 高い | やや制限あり |
| ファイルサイズ | 小さい | 大きい |
| レスポンシブ対応 | 自分で実装 | 標準搭載 |
| 学習コスト | CSS知識のみ | Bootstrap独自のクラス名を覚える必要あり |

---

### 後からBootstrapを使うのはありか？

**結論: ありだけど、注意が必要。**

#### うまくいくケース

- まだCSSがあまり書かれていない初期段階
- 既存のクラス名がBootstrapと衝突しない
- 管理画面など、デザインの統一性をそこまで気にしない部分への導入

#### 注意が必要なケース

- **クラス名の衝突**: 既存の `.container` や `.row` などがBootstrapのクラスと競合する可能性がある
- **スタイルの上書き合戦**: Bootstrapのデフォルトスタイルと既存CSSが干渉して、意図しない表示崩れが起きる
- **二重管理**: 生CSSとBootstrapが混在すると、どちらを修正すればいいか分かりにくくなる

#### 導入するなら

1. **部分的に導入する**: グリッドシステムだけ、コンポーネントだけなど範囲を限定する
2. **既存CSSとの衝突を確認**: 同名クラスがないかチェックする
3. **チームで方針を統一する**: 「新規ページはBootstrap、既存は触らない」などルールを決める

できれば **プロジェクトの初期段階で使うかどうかを決めておく** のがベストです。
