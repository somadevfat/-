# GitHub Actionsで実現するAndroidアプリのCI/CDパイプライン構築ガイド

Android開発において、ビルド、テスト、そして配布のプロセスを自動化することは、開発効率を飛躍的に向上させ、品質の高いアプリを迅速にユーザーへ届けるために不可欠です。

この記事では、GitHub Actionsを利用して、AndroidアプリのCI/CDパイプラインをゼロから構築する方法をステップバイステップで解説します。

## この記事で構築するパイプライン

* **CI (継続的インテグレーション)**: `main`ブランチへのpushや、Pull Requestの作成をトリガーに、アプリのビルドとユニットテストを自動で実行します。
* **CD (継続的デプロイ)**: `main`ブランチへのpushをトリガーに、署名済みのReleaseビルドを作成し、Firebase App Distributionへ自動で配布します。

## 前提条件

* GitHub上にAndroidアプリのプロジェクトリポジトリが存在すること。
* Firebaseプロジェクトが作成済みであること（CDでFirebase App Distributionを利用するため）。

---

## ステップ1: CI (ビルドとテストの自動化)

まずは、コードがリポジトリにプッシュされるたびに、ビルドとテストが自動的に実行される環境を構築します。

### ワークフローファイルの作成

1.  リポジトリのルートに `.github/workflows/` というディレクトリを作成します。
2.  その中に `android_ci.yml` という名前でYAMLファイルを作成し、以下の内容を貼り付けます。

```yaml
# .github/workflows/android_ci.yml

name: Android CI

# ワークフローが実行されるトリガーを指定
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    # 実行環境として最新のUbuntuを指定
    runs-on: ubuntu-latest

    steps:
    # (1) ソースコードをチェックアウト
    - name: Checkout
      uses: actions/checkout@v4

    # (2) JDK 17 をセットアップ
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    # (3) Gradleのキャッシュ設定（ビルド時間短縮のため）
    - name: Gradle Cache
      uses: actions/cache@v4
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
        restore-keys: |
          ${{ runner.os }}-gradle-

    # (4) Gradle Wrapperに実行権限を付与
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew

    # (5) Debugビルドを実行
    - name: Build with Gradle
      run: ./gradlew assembleDebug

    # (6) ユニットテストを実行
    - name: Run unit tests
      run: ./gradlew testDebugUnitTest
```

### CIワークフローの解説

* **`on`**: `main`ブランチへの`push`と`pull_request`をトリガーにワークフローを開始します。
* **`steps`**:
    1.  **`actions/checkout@v4`**: リポジトリのソースコードをVM（仮想マシン）にダウンロードします。
    2.  **`actions/setup-java@v4`**: Androidビルドに必要なJava環境をセットアップします。
    3.  **`actions/cache@v4`**: Gradleが利用するライブラリなどをキャッシュし、2回目以降のビルドを高速化します。
    4.  **`chmod +x gradlew`**: ビルドスクリプトである`gradlew`に実行権限を与えます。
    5.  **`./gradlew assembleDebug`**: Debugバージョンのアプリをビルドします。
    6.  **`./gradlew testDebugUnitTest`**: ユニットテストを実行します。

このファイルをリポジトリに追加してpushすると、リポジトリの **[Actions]** タブでCIが実行される様子を確認できます。

---

## ステップ2: CD (Firebaseへの自動配布)

次に、CIが成功したビルドをテスター向けに自動で配布する仕組みを構築します。配布先としては導入が容易な **Firebase App Distribution** を利用します。

### 1. アプリの署名設定

Releaseビルドには署名が必要です。署名キーとパスワードを安全に管理するために **GitHub Secrets** を利用します。

1.  **署名キーのBase64エンコード**:
    ターミナルで、あなたの署名キーファイル (`.jks`) をBase64文字列に変換します。

    ```bash
    base64 -w 0 my-release-key.jks
    ```
    表示された長い文字列をコピーしておきます。

2.  **GitHub Secretsの設定**:
    リポジトリの **[Settings]** > **[Secrets and variables]** > **[Actions]** に移動し、以下の4つのSecretを登録します。

    * `KEYSTORE_JKS_BASE64`: 先ほどコピーした署名キーのBase64文字列
    * `KEYSTORE_PASSWORD`: キーストアのパスワード
    * `KEY_ALIAS`: キーのエイリアス
    * `KEY_PASSWORD`: キーのパスワード

### 2. Firebaseへの接続設定

1.  **Firebase CLIのインストールとログイン**:
    ローカルマシンにFirebase CLIがなければインストールし、ログインします。

    ```bash
    npm install -g firebase-tools
    firebase login:ci
    ```
    ブラウザが開いてログインが完了すると、ターミナルに **Refresh Token** が表示されます。これをコピーします。

2.  **Firebase App IDの確認**:
    Firebaseコンソールのプロジェクト設定で、対象となるAndroidアプリの **アプリID** を確認します。 `1:1234567890:android:abcdef1234567890` のような形式です。

3.  **GitHub Secretsへの登録**:
    先ほどと同様に、リポジトリのSecretsに以下を追加します。

    * `FIREBASE_TOKEN`: Firebase CLIで取得したRefresh Token
    * `FIREBASE_APP_ID`: あなたのFirebaseアプリID

### 3. CDワークフローの作成

CI/CDを一つのファイルで管理します。先ほどの `android_ci.yml` を以下のように拡張します。

```yaml
# .github/workflows/android_ci_cd.yml

name: Android CI/CD

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  # CIジョブ (ビルドとテスト)
  ci-build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Gradle Cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      - name: Build with Gradle
        run: ./gradlew assembleDebug
      - name: Run unit tests
        run: ./gradlew testDebugUnitTest

  # CDジョブ (ビルドと配布)
  cd-build-deploy:
    # ci-build-testジョブが成功し、かつmainブランチへのpushの場合のみ実行
    needs: ci-build-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Gradle Cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      
      # (1) 署名キーをSecretsからファイルに復元
      - name: Decode Keystore
        run: echo "${{ secrets.KEYSTORE_JKS_BASE64 }}" | base64 --decode > app/my-release-key.jks
      
      # (2) 署名付きAABを作成
      - name: Build Release AAB
        run: ./gradlew bundleRelease
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      # (3) Firebase App Distributionにアップロード
      - name: Upload to Firebase App Distribution
        uses: wzieba/firebas-app-distribution@v1.4.0
        with:
          appId: ${{ secrets.FIREBASE_APP_ID }}
          token: ${{ secrets.FIREBASE_TOKEN }}
          groups: "testers" # 配布先のグループ名
          file: app/build/outputs/bundle/release/app-release.aab
```
*注: `build.gradle`側で、署名情報を環境変数から読み込む設定が別途必要です。*

### CDワークフローの解説
* **`needs: ci-build-test`**: `ci-build-test`ジョブの成功を待ちます。
* **`if: ...`**: このジョブが`main`ブランチへの`push`の時だけ実行されるように制限します。
* **`Decode Keystore`**: Secretsに保存したBase64文字列を、元の`.jks`ファイルにデコードして復元します。
* **`Build Release AAB`**: 署名情報（環境変数経由で渡す）を使って、リリース用のAAB(Android App Bundle)を作成します。
* **`wzieba/firebas-app-distribution`**: AABをFirebase App Distributionにアップロードするための便利なアクションです。`testers`グループに配布しています。

---

## まとめ

これで、GitHub ActionsによるAndroidアプリのCI/CDパイプラインが完成しました。

* **開発中**: Pull Requestを作成すれば、自動でビルドとテストが走り、コードの品質を担保できます。
* **リリース時**: `main`ブランチにマージ（push）するだけで、署名済みのアプリが自動でテスターに配布されます。

このパイプラインを基盤として、さらに「Google Playへの自動リリース」や「UIテストの追加」など、あなたのプロジェクトに合わせて発展させていくことができます。

## ページ一覧
- [応募予定企業一覧はこちら](./index.html)
- [AndroidアプリのCI/CDガイドはこちら](./test.html)