# user-ebuild

**Bubblewrap (`bwrap`)** を使用して、Gentoo の `ebuild` コマンドを非ルート（一般ユーザー）権限で実行するための軽量なラッパースクリプトです。

## 概要

`user-ebuild` は、Linux の名前空間（namespaces）を利用して隔離されたサンドボックス環境を作成します。これにより、ホストシステムの `sudo` やルート権限を必要とせずに、`unpack`、`compile`、`package`（バイナリパッケージの作成）などの ebuild フェーズを実行できます。

## 特徴

- **安全な隔離**: ホストシステムのルートファイルシステムは読み取り専用でマウントされます。
- **偽装ルート**: サンドボックス内で現在のユーザーを `uid 0` (root) にマッピングし、Portage の権限要件を満たします。
- **書き込み可能なオーバーレイ**: Portage の書き込みが必要なディレクトリ（`PORTAGE_TMPDIR`、`DISTDIR`、`PKGDIR`）を、`/tmp` 内のユーザー所有ディレクトリにリダイレクトします。
- **ユーザー情報の偽装**: サンドボックス内に偽の `/etc/passwd` と `/etc/group` ファイルを自動生成し、`chown` や権限エラーを防止します。
- **Manifest のサポート**: `manifest` コマンドの実行時、対象の ebuild ディレクトリに対して書き込み権限がある場合、自動的に書き込み可能としてバインドマウントします。

## 必須条件

- **Bubblewrap**: `sys-apps/bubblewrap` がインストールされている必要があります。
- **Portage**: `sys-apps/portage` がインストールされている必要があります（`ebuild` コマンドおよび設定ファイルのため）。

## インストール

1. `user-ebuild` スクリプトをローカルのパス（例: `~/bin/` やカレントディレクトリ）にコピーします。
2. 実行権限を付与します:
   ```bash
   chmod +x user-ebuild
   ```

## 使い方

標準の `ebuild` コマンドと全く同じように実行します:

```bash
# ソースコードの展開
./user-ebuild /path/to/category/package/package-version.ebuild unpack

# パッケージのコンパイル
./user-ebuild /path/to/category/package/package-version.ebuild compile

# バイナリパッケージ (.xpak または .gpkg.tar) の作成
./user-ebuild /path/to/category/package/package-version.ebuild package

# Manifest の更新（ローカルオーバーレイ用）
./user-ebuild /path/to/category/package/package-version.ebuild manifest
```

## 仕組み

1. **サンドボックスのセットアップ**: `/tmp/gentoo-user-ebuild-${USER}` にサンドボックスのルートを作成します。
2. **名前空間の隔離**: `bwrap --unshare-all` を使用して、IPC、PID、およびマウント名前空間を隔離します。
3. **バインドマウント**:
   - `/` (ホスト) -> `/` (サンドボックス, 読み取り専用)
   - `/tmp/gentoo-user-ebuild-${USER}/var/tmp/portage` -> `/var/tmp/portage` (書き込み可能)
   - `/tmp/gentoo-user-ebuild-${USER}/var/cache/distfiles` -> `/var/cache/distfiles` (書き込み可能)
4. **環境設定**: `FEATURES="-userpriv -usersandbox -sandbox -userfetch"` を設定し、Portage 内部の権限変更やサンドボックス機能を無効化します（これらは `bwrap` 環境と重複するか、互換性がないため）。

## サンドボックスのディレクトリ構造

スクリプトは `/tmp/gentoo-user-ebuild-${USER}/` をベースとして使用します。内部には以下の構造が作成されます:
- `var/tmp/portage/`: ビルド作業ディレクトリ。
- `var/cache/distfiles/`: ダウンロードされたソースアーカイブ。
- `var/cache/binpkgs/`: 生成されたバイナリパッケージ。
- `etc/portage/`: システムの Portage 設定のコピー。

## クリーンアップ

サンドボックスは `/tmp` に配置されているため、通常は再起動時に消去されます。手動でクリーンアップするには以下を実行します:
```bash
rm -rf /tmp/gentoo-user-ebuild-${USER}
```

## 制限事項

- **システムへのインストール**: このスクリプトはビルドとテスト用です。ルートファイルシステムが読み取り専用でマウントされているため、ホストシステムの `/usr` や `/lib` にパッケージをインストールすることはできません。
- **依存関係**: ビルド依存関係を満たすために、ホストシステムにインストールされているライブラリやヘッダーに依存します。ホストに存在しない依存パッケージが必要な場合、ビルドは失敗します。
