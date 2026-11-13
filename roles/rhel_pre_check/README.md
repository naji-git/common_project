# rhel_pre_check ロール
 
パッチ適用やメンテナンス作業の実施前に、対象サーバーの状態確認・エビデンス保存・設定バックアップを一括して行う共通ロールです。
本ロールで取得したエビデンス（`*_before`）は、作業後に実行する `rhel_post_check` ロール で取得する `*_after` との比較・設定復旧の基準として使用されます。
 
## 処理一覧 (`tasks/main.yml`)
本ロールを実行すると、以下の順序で各タスクファイルが処理されます。
 
1. **事前リソース使用状況の確認 (`check_resource_usage.yml`)**
  - エビデンス保存用ディレクトリを準備（既存ディレクトリがある場合は `<rhel_pre_check_patch_evidence_dir>_backup_<日時>` にリネーム退避したうえで新規作成）
  - CPU（`sar -u`）、メモリ（`sar -r`）、ディスク（`df -h`）の使用状況をエビデンス保存
2. **事前システム状態の確認 (`check_system_status.yml`)**
  - プロセス状態（`sk_proc_dsp.sh`）、カーネルバージョン、systemd unit-files、Java情報（/usr/lib/jvm）をエビデンス保存
  - `/usr/lib/jvm` が存在しない場合、Java情報の取得はスキップ
3. **yum.conf / dnf.conf の修正・事前パッケージ情報の取得・リポジトリ設定の無効化 (`check_package_info.yml`)**
  - 対象ファイルは OS メジャーバージョンで切り替え（RHEL7: `/etc/yum.conf`、RHEL8以降: `/etc/dnf/dnf.conf`）し、修正前の内容をエビデンス保存
  - `exclude` に `kernel` を含む行がコメントアウト（`#` または `;`）されている場合のみ、以下を実施
    - `installonly_limit` を `2` に変更
    - `exclude` 行のコメントアウトを解除（`exclude` を有効化）
  - `rpm -qa` の結果をエビデンス保存
  - `/etc/yum.repos.d/*.repo` に `enabled=1` を含むファイルが存在する場合のみ、以下を実施
    - 該当する repo ファイルを `<repoファイル名>_before` としてエビデンス保存
    - `enabled=1` を `enabled=0` に変更（リポジトリを無効化）
    - `yum repolist all` を実行し、有効（enabled）なリポジトリが残っていないことを確認（残っている場合は異常終了）
4. **httpd設定の退避 (`backup_httpd_conf.yml`)**
  - `/etc/httpd/conf/httpd.conf` が存在する場合のみ実行（未導入時はスキップ）
  - `conf.modules.d` / `conf.d` のファイル一覧と `httpd.service` をエビデンス保存
5. **ca-certificates の退避 (`backup_ca_certificates_conf.yml`)**
  - Java の cacerts（`$(dirname $(dirname $(readlink -f $(which java))))/lib/security/cacerts`）を `keytool -list` で確認し、エイリアスに `ninsho`（大文字小文字は区別しない）を含む証明書がある場合のみ実行
    - 対象が無い場合、または java / keytool が無い場合は「ca-certificatesの対象が無いので処理をスキップします」を表示してスキップ
    - 対象のエイリアスが複数ある場合は退避対象を特定できないため異常終了（エイリアスの一覧と確認手順をメッセージで案内）
  - 対象の証明書を `e-ninshoca.crt`（PEM 形式）としてエビデンスディレクトリに書き出し、ファイルが作成されたことを確認
  - 退避したエイリアス名を `ca-certificates_alias_before`、SHA256 フィンガープリントを `ca-certificates_FingerprintSHA256_before` にエビデンス保存（`rhel_post_check` が復旧の判定に使用）
6. **PostgreSQLリンクの確認 (`check_postgresql_link.yml`)**
  - `psql` コマンドが存在する場合のみ実行（未導入時はスキップ）
  - `/bin`、`/usr/bin` 配下の PostgreSQL 関連シンボリックリンクをエビデンス保存
7. **Apache・JBoss・JP1 の停止 (`stop_process.yml`)**
  - サービス停止の前に、SAM監視ログの確認を行う
    - 当日の `/usr/local/sam/kanshi/log/filtered/msg_filtered_<YYYYMMDD>.log` が存在する場合は、内容を表示する（内容の確認は作業者が実施）
    - 存在しない場合のみ、`root:root`・権限 `0644` で空ファイルを作成する（`ansible.builtin.file` の `state: touch`）。既存のファイルの権限は変更しない
  - パッチ適用前に、Apache（httpd）、JBoss、JP1 を次の順序で停止する
    1. Apache（httpd）
    2. JBoss
    3. JP1（`jp1_ajs3` → `jp1_base` の順）
  - Apache・JBoss の停止は環境によって処理が異なる
    - 開発環境（`is_develop_env: true`）
      - `rhel_pre_check_httpd_services` / `rhel_pre_check_jboss_services` のうち、サーバー上に存在して起動中（`running`）のサービスだけを対象にする
      - 停止前に、起動中だったサービス名の一覧を JSON でエビデンス保存する（起動中のサービスがない場合は空のリスト `[]` を保存する）
      - 起動中だったサービスだけを停止する（停止済み・存在しないサービスは何もしない）
      - 保存した JSON は、パッチ適用後に実行する `rhel_post_check` ロールが再起動前の状態に戻すために使用する
    - プレ本・本番環境（`is_develop_env` が未定義または `false`）
      - `rhel_pre_check_stop_services` に定義された全サービスを、リストの並び順どおりに停止する
      - 起動状態の保存はしない（パッチ適用後に `rhel_post_check_stop_services` が起動していることを確認する）
      - サーバー上に存在しないサービス名を指定するとタスクが失敗するため、実在するサービス名だけを指定すること
  - JP1 の停止は全環境共通
    - `rhel_pre_check_jp1_services`（既定: `jp1_ajs3`, `jp1_base`）のうち、サーバー上に存在して起動中のサービスだけを対象にする。JP1 が未導入のサーバーでは何も停止しない
    - 停止前に、起動中だったサービス名の一覧を JSON でエビデンス保存する
    - リストの並び順（JP1/AJS3 → JP1/Base）で停止する
    - 停止後にサービス状態を再取得し、対象のサービスが停止していることを `assert` で確認する（停止できていない場合は異常終了し、確認すべきコマンド `systemctl status <service>` / `journalctl -u <service>` をメッセージで案内する）
  - `check_resource_usage.yml` が既存のエビデンスディレクトリを退避してから作り直すため、本処理はその後に実行する（先に実行すると保存した JSON が退避される）
8. **/boot 空き容量の確保 (`cleanup_boot_space.yml`)**
  - 旧 `rhel_cleanup_boot_space` ロールの処理を移行したもの。仮想化基盤（`rhel_system_platform` が `kasou`）かつ RHEL 7 で、パッチ格納ディレクトリ（`rhel_patch_rpm_directory`）直下に kernel-debug のパッチ（`rhel_pre_check_boot_space_patch_file_pattern`、既定 `kernel-debug-[0-9]*.rpm`）がある場合のみ実行（それ以外はスキップ。パッチが無い場合は、その旨と確認事項をメッセージで表示）
  - パッチ適用前の最新世代の `kernel-debug` パッケージを削除して `/boot` の空き容量を確保（通常カーネル `kernel` は削除しない）
  - 削除対象の世代はインストール済み `kernel` のうちビルド時刻が最新のものを自動判定（`rhel_pre_check_boot_space_target_kernel_version` で明示も可）
  - 削除対象が未インストールの場合は削除をスキップして正常終了（冪等）
  - 以下を `assert` で確認し、1つでも満たさない場合は異常終了（パッチ適用に進まない）
    - 通常カーネルで起動していること
    - `yum remove --assumeno` の結果、対象パッケージ以外が削除されないこと
    - 対象パッケージが `/boot` 配下にファイルを持つこと
    - 削除後に `/boot` の空き容量が増え、対象ファイルが `/boot` から消えていること
    - 削除前後のパッケージ差分が対象パッケージ1つだけであること
  - 削除前後の `rpm -qa`、依存解決結果、結果サマリをエビデンス保存
  - 他の事前確認の後に実行するため、`rpm_qa_before` は kernel-debug 削除前の状態を記録する（削除前後の差分は `boot_space_rpm_qa_before` / `boot_space_rpm_qa_after` で確認できる）
  - パッチ適用（`rhel_update_patch`）の前に実行すること。パッチ適用後に実行してはならない（残すべき最新世代の debug カーネルを削除してしまう）
---
## 必要な変数
 
1. **作業ディレクトリ（`defaults/main.yml`）**
  - rhel_pre_check_patch_evidence_dir: "/home/ansible/rhel_patch" （エビデンス格納ディレクトリ。パッチ作業のベースディレクトリ）
  - 個別プロジェクトの `group_vars/all.yml` でシステムごとに上書きできる。`rhel_post_check` の `rhel_post_check_patch_evidence_dir` と同じパスにすること（異なると `rhel_post_check` が事前エビデンスを読めない）
 
2. **Apache / JBoss / JP1 停止の設定（各システム専用プロジェクト側のインベントリ変数等で定義。ロール内には定義なし）**
 
| 変数名 | 未定義時 | 説明 |
| --- | --- | --- |
| `is_develop_env` | `false` | `true` の場合は開発環境として扱い、起動状態を記録して停止する。それ以外はプレ本・本番環境として扱う |
| `rhel_pre_check_httpd_services` | `[]` | 開発環境で停止する Apache のサービス名のリスト（`.service` を除いた名前。例: `httpd`） |
| `rhel_pre_check_jboss_services` | `[]` | 開発環境で停止する JBoss のサービス名のリスト（`.service` を除いた名前） |
| `rhel_pre_check_stop_services` | `[]` | プレ本・本番環境で停止するサービス名のリスト（`.service` を除いた名前。停止順はリストの並び順） |
| `rhel_pre_check_jp1_services` | `["jp1_ajs3", "jp1_base"]` | 停止する JP1 のサービス名のリスト（`.service` を除いた名前）。リストの並び順が停止順 |
 
  - 未定義または空のリストの場合は、該当する停止処理は何も行わない。
  - 開発環境で `rhel_pre_check_httpd_services` / `rhel_pre_check_jboss_services` に指定したサービスが、サーバーに存在しない場合や停止していた場合は、停止の対象外となる。
  - プレ本・本番環境で `rhel_pre_check_stop_services` に存在しないサービス名を指定すると、停止に失敗して異常終了する。
  - `rhel_post_check` 側の `rhel_post_check_httpd_services` / `rhel_post_check_jboss_services` / `rhel_post_check_stop_services` は、ここで指定したリストと同じ内容にする。
 
3. **ca-certificates 退避の設定（`defaults/main.yml`）**
 
| 変数名 | デフォルト値 | 説明 |
| --- | --- | --- |
| `rhel_pre_check_ca_cert_alias_pattern` | `ninsho` | cacerts から退避対象を探すエイリアスの検索パターン（大文字小文字を区別しない） |
| `rhel_pre_check_ca_cert_keystore_password` | `changeit` | cacerts のパスワード（Java 標準の既定値） |
 
4. **/boot 空き容量確保の設定（`defaults/main.yml`）**
 
| 変数名 | デフォルト値 | 説明 |
| --- | --- | --- |
| `rhel_pre_check_boot_space_platform` | `{{ rhel_system_platform }}` | プラットフォーム。`kasou` の場合のみ kernel-debug 削除を実行。共通変数 `rhel_system_platform` から引き継ぐため通常は直接指定しない |
| `rhel_patch_rpm_directory` | `/home/ansible/patch` | パッチ（RPM）格納ディレクトリ。`rhel_post_check` と共通の変数で、`group_vars/all.yml` でシステムごとに上書きできる。直下に kernel-debug のパッチがある場合のみ kernel-debug 削除を実行 |
| `rhel_pre_check_boot_space_patch_file_pattern` | `kernel-debug-[0-9]*.rpm` | kernel-debug のパッチ RPM のファイル名パターン。`kernel-debug-devel` など別パッケージを除くため、バージョン部の数字で始める |
| `rhel_pre_check_boot_space_target_package_name` | `kernel-debug` | 削除対象のパッケージ名 |
| `rhel_pre_check_boot_space_target_kernel_version` | `""`（自動判定） | 削除対象の世代（`VERSION-RELEASE.ARCH` 形式） |
| `rhel_pre_check_boot_space_mount_point` | `/boot` | 空き容量を確認するマウントポイント。`/boot` を独立させていない構成では `/` などを指定 |
| `rhel_pre_check_boot_space_supported_major_versions` | `["7"]` | kernel-debug 削除に対応する RHEL メジャーバージョン |
 
5. **OS バージョン別の判定値（`vars/RedHat_7.yml`）**
  - `yum remove --assumeno` の出力から「依存関係で他のパッケージが削除されないこと」を判定するための文字列（`__rhel_pre_check_boot_space_*`）
  - RHEL 8 以降に対応する場合は `vars/RedHat_8.yml` を追加し、`rhel_pre_check_boot_space_supported_major_versions` に追記する。あわせて削除タスクを `ansible.builtin.dnf` に切り替える必要がある
 
---
## 前提条件
 
- 対象サーバーに `/usr/local/sam/kanshi/shell/sk_proc_dsp.sh` が配置されていること
- kernel-debug 削除を行う場合、共通変数 `rhel_system_platform` が `kasou` で定義されていること（未定義・`aws` の場合は当該処理のみスキップ）
- RHEL 7 の `ansible.builtin.yum` は Python 2.7 の yum/rpm バインディングを使用する。`ansible_python_interpreter` に Python 3 を指定している場合は削除タスクが失敗するため、指定を確認すること
- ca-certificates の退避を行う場合、対象サーバーに `java` と `keytool` が PATH 上に存在すること（どちらかが無い場合は当該処理のみスキップ）
- Ansible の fact（`gather_facts`）が取得済みであること（OSバージョン、ホスト名、日時の判定に使用）
- kernel-debug 削除を行う場合、パッチ（RPM）を `rhel_patch_rpm_directory` 直下に、`rhel_pre_check` の実行前に配置しておくこと（配置前に実行すると kernel-debug 削除はスキップされる）
- 実行ユーザーが `/etc/yum.conf`（または `/etc/dnf/dnf.conf`）および `/etc/yum.repos.d/*.repo` の変更、パッケージ削除、`rhel_pre_check_patch_evidence_dir` の作成を行える権限を持つこと
 
---
## 出力されるエビデンス (`rhel_pre_check_patch_evidence_dir` 配下)
 
| ファイル | 内容 |
| --- | --- |
| `cpu_usage_before` / `mem_usage_before` / `disk_usage_before` | 事前のCPU・メモリ・ディスク使用状況 |
| `process_status_before` | 事前のプロセス状態 |
| `kernel_version_before` | 事前のカーネルバージョン |
| `unit-files_before` | 事前の systemd unit-files 一覧 |
| `java_info_before` | 事前の Java インストール情報 |
| `yum.conf_before` / `dnf.conf_before` | 修正前の yum.conf / dnf.conf（OSにより一方） |
| `rpm_qa_before` | 事前のパッケージ一覧 |
| `<repoファイル名>_before`（例: `rhel.repo_before`） | 無効化前の repo ファイル（`enabled=1` を含むファイルのみ） |
| `conf.modules.d_before` / `conf.d_before` / `httpd.service_before` | 事前の httpd 設定情報 |
| `pg_link_bin_before` / `pg_link_usr_before` | 事前の PostgreSQL シンボリックリンク情報 |
| `e-ninshoca.crt` | 退避した ca-certificates（PEM 形式。対象がある場合のみ。`rhel_post_check` が再インポートに使用） |
| `ca-certificates_alias_before` | 退避した ca-certificates のエイリアス名（対象がある場合のみ） |
| `ca-certificates_FingerprintSHA256_before` | 退避した ca-certificates の SHA256 フィンガープリント（対象がある場合のみ） |
| `httpd_running_services_pre_check.json` / `jboss_running_services_pre_check.json` | 停止前に起動していた Apache / JBoss のサービス名の一覧（JSON。開発環境のみ。`rhel_post_check` が起動状態の復元に使用） |
| `jp1_running_services_pre_check.json` | 停止前に起動していた JP1 のサービス名の一覧（JSON。全環境。停止順で保存。`rhel_post_check` が起動対象と起動順（逆順）の決定に使用） |
| `boot_space_rpm_qa_before` / `boot_space_rpm_qa_after` | kernel-debug 削除前後のパッケージ一覧（ソート済み） |
| `boot_space_remove_dry_run` | `yum remove --assumeno` の出力（依存解決結果） |
| `boot_space_cleanup_result` | `/boot` 空き容量の削除前後比較、削除したファイル、パッケージ差分 |
 
---
## 実行方法
 
Playbook または呼び出し元ロールから include して実行します。
 
```yaml
- name: 事前確認処理を実行する
  ansible.builtin.include_role:
    name: rhel_pre_check
```

