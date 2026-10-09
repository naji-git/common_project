# pre_check role

パッチ適用やメンテナンス作業の実施前に、対象サーバーの状態確認・エビデンス取得・設定バックアップを一括して行う共通ロールです。

## 処理一覧 (`tasks/main.yml`)

本roleを実行すると、以下の順序で各タスクファイルが処理されます。

1. **事前リソース使用状況の確認 (`check_resource_usage.yml`)**
   - CPU、メモリ、ディスク使用率（`/`）を取得・エビデンス保存
   - 閾値判定（超過時はジョブを異常終了）
2. **事前システム状態の確認 (`check_system_status.yml`)**
   - OS・カーネル情報、起動中プロセス、稼働状態等の確認とエビデンス保存
3. **事前PostgreSQLリンク確認 (`check_postgresql_link.yml`)**
   - DB関連のシンボリックリンクや状態の事前確認
4. **事前httpd設定バックアップ (`backup_httpd_conf.yml`)**
   - 作業前の Web サーバー（Apache/httpd）設定ファイルのバックアップ取得
5. **yum.conf修正と事前パッケージ情報の取得 (`check_package_info.yml`)**
   - パッケージマネージャ設定の調整および適用前パッケージ一覧の記録

---

## 必要な変数（`vars/main.yml`）

1. **設定ファイルの保存先を指定:/home/infra/rhel_patch**
2. **CPU、メモリ、ディスク使用率で使用する閾値については各システム専用プロジェクト側（`roles/rhel_pre_check/vars/main.yaml` 等）で定義してください**
   - threshold_cpu_usage: xx   
   - threshold_mem_usage: xx   
   - threshold_disk_usage: xx   

---

## 実行方法

Playbook または呼び出し元roleから include して実行します。

```yaml
- name: 事前確認処理の実行
  ansible.builtin.include_role:
    name: pre_check
