# rhel_pre_reboot_recovery ロール
 
パッチ適用の後、OS 再起動の前に、パッチ適用結果の確認・適用後のエビデンス取得・事前取得結果との比較・設定復旧を行う共通ロールです。
事前確認は `rhel_pre_check` ロール で取得したエビデンス（`*_before`）を前提とし、本ロールで `*_after` を取得して差分を確認します。OS 再起動の後の確認は `rhel_post_check` ロールで行います。
 
## 処理一覧 (`tasks/main.yml`)
本ロールを実行すると、以下の順序で各タスクファイルが処理されます。
 
1. **パッチ適用結果の確認 (`check_patch_apply_result.yml`)**
  - パッチ適用ログの確認
    - `{{ rhel_pre_reboot_recovery_patch_evidence_dir }}/security_update.log` の存在を確認。存在しない場合は異常終了（`rhel_update_patch` の実行有無とログの出力先を確認するようメッセージで案内）
    - ログの内容に `Complete!` が含まれることを確認。含まれない場合は異常終了（パッチ適用が正常に完了していない可能性があるため、ログを確認して再実行するようメッセージで案内）
  - SAM 監視ログの確認：当日の `/usr/local/sam/kanshi/log/filtered/msg_filtered_<YYYYMMDD>.log` の内容を全て表示（ファイルが無い場合は WARNING 表示。内容の確認および判定は作業者が実施）
2. **事後パッケージ情報の取得 (`check_package_info.yml`)**
  - `rpm -qa` の結果（`sort -n` 済み）を `rpm_qa_after` としてエビデンス取得
  - 事前エビデンス（`rpm_qa_before`）が存在する場合のみ、事前・事後の差分（更新されたパッケージ）を `rpm_qa_diff` としてエビデンス取得・表示（差分が無ければ「差分なし」）
    - `rpm_qa_before` が存在しない場合は WARNING を表示して差分比較をスキップ（事前確認の実施有無を確認）
    - `diff` の終了コードが 0（差分なし）・1（差分あり）以外の場合は異常終了
  - yumリポジトリ参照設定の戻し
    - エビデンスディレクトリ配下の `*.repo_before`（事前確認で退避した repo ファイル）を、`_before` を除いたファイル名で `/etc/yum.repos.d/` へ復旧（`become` で実行）
    - `*.repo_before` が存在しない場合は何もしない
3. **httpd設定の復旧 (`restore_httpd_conf.yml`)**
  - `/etc/httpd/conf/httpd.conf` が存在する場合のみ実行（未導入時は「httpd未導入のため処理をスキップします。」を表示してスキップ）
  - `conf.modules.d` / `conf.d` の事後ファイル一覧（`conf.modules.d_after` / `conf.d_after`）と `httpd.service`（`httpd.service_after`）を退避
  - 事前エビデンス（`conf.modules.d_before` / `conf.d_before`）が存在する場合のみ、差分（パッチで追加された不要ファイル）をエビデンスディレクトリへ移動
    - 事前エビデンスが存在しない場合は WARNING を表示して比較をスキップ
    - 差分があった場合は、移動後のファイル一覧（`conf.modules.d_after_mv` / `conf.d_after_mv`）が事前状態と一致することを検証（不一致時は異常終了）
  - 事前エビデンス（`httpd.service_before`）が存在する場合のみ、`httpd.service` を復旧し、daemon-reload のうえ httpd を起動・状態確認
    - `httpd.service_before` が存在しない場合は WARNING を表示して復旧と起動をスキップ（`httpd.service` の内容確認後、手動で起動）
  - ここでの httpd 起動は、事前確認時の起動状態の復元とは別の処理。開発環境では、OS 再起動後の `rhel_post_check`（エージェント状態の確認）で、事前確認時の起動状態に合わせて再度起動または停止される
 
---
## 必要な変数
 
1. **作業ディレクトリ（`defaults/main.yml`）**
  - rhel_pre_reboot_recovery_base_dir: "/home/ansible" （パッチ作業のベースディレクトリ）
  - rhel_pre_reboot_recovery_patch_evidence_dir: "{{ rhel_pre_reboot_recovery_base_dir }}/rhel_patch" （エビデンス格納ディレクトリ。パッチ適用ログ `security_update.log` の確認に使用）
  - いずれも個別プロジェクトの `group_vars/all.yml` でシステムごとに上書きできる。`rhel_pre_check_patch_evidence_dir` と同じパスにすること（異なると事前エビデンスやパッチ適用ログを読めない）。
 
2. **`check_package_info.yml` / `restore_httpd_conf.yml` が参照する変数**
  - 現状、この2つのタスクファイルは `rhel_pre_reboot_recovery_patch_evidence_dir` ではなく `rhel_post_check_patch_evidence_dir` を参照している
  - `rhel_post_check_patch_evidence_dir` は本ロールの `defaults` に定義がないため、`group_vars/all.yml` などで定義するか、`rhel_post_check` の `defaults` が読み込み済みである必要がある。未定義の場合は変数未定義エラーになる
  - `rhel_pre_check_patch_evidence_dir` / `rhel_post_check_patch_evidence_dir` / `rhel_pre_reboot_recovery_patch_evidence_dir` は同じパスにすること
 
---
## 前提条件
 
- パッチ適用（`rhel_update_patch`）が実施済みで、パッチ適用ログ `security_update.log` がエビデンスディレクトリに出力されていること
- 事前確認（`rhel_pre_check`）が実施済みで、エビデンスディレクトリ配下に以下の事前エビデンスが存在すること
  - `rpm_qa_before`
  - `conf.modules.d_before`, `conf.d_before`, `httpd.service_before`（httpd導入時）
  - `*.repo_before`（事前確認で `/etc/yum.repos.d/` 配下の repo ファイルを退避している場合。パッチ適用後に元へ戻す対象）
- Ansible の fact（`gather_facts`）が取得済みであること（当日の日付の判定に使用）
- 実行ユーザーが、`/etc/yum.repos.d/` 配下の更新、`/etc/httpd` 配下のファイル移動、`httpd.service` の復旧と httpd の起動、SAM 監視ログの参照を行える権限（`become`）を持つこと
 
---
## 注意事項
 
- パッチ適用ログに `Complete!` が無い場合は異常終了し、以降の処理（SAM 監視ログの確認、パッケージ情報の取得、httpd 設定の復旧）は実行されません。
- 本ロールでエビデンスディレクトリ配下に出力したファイルは、OS 再起動後に実行する `rhel_post_check` の最後（作業ディレクトリの整理）で、`<YYYYMMDD>` ディレクトリへ移動されます。
 
---
## 出力されるエビデンス (エビデンスディレクトリ配下)
 
| ファイル | 内容 |
| --- | --- |
| `rpm_qa_after` / `rpm_qa_diff` | 事後のパッケージ一覧 / 事前エビデンスとの差分 |
| `conf.modules.d_after` / `conf.d_after` / `httpd.service_after` | 事後の httpd 設定情報 |
| `conf.modules.d_after_mv` / `conf.d_after_mv` | 不要ファイル退避後の httpd 設定一覧 |
 
---
## 実行方法
 
Playbook または呼び出し元ロールから include して実行します。
 
```yaml
- name: パッチ適用後・再起動前の確認処理を実行する
  ansible.builtin.include_role:
    name: rhel_pre_reboot_recovery
```

