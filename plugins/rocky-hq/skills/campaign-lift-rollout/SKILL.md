---
name: campaign-lift-rollout
description: 本部が「この前のクーポン効いた?他店に展開していい?」と聞いたときの判断フロー。単に効果値を出すのでなく、因果手法を振り分けて有意性を判定し、勝ち筋の裏取り→展開先選定→店別ドライラン→本適用→事後の対照検証まで導く。owner/hq_admin向け、rocky-regi(ロッキーレジ)本部の攻めフロー。
---

# 施策の効き検証→横展開/撤退判断

## これは何か
本部が「この前の施策、効いたの?」「効いたなら他店にも展開していいか?」と聞いたときに、単一店の効果値だけを出して終わりにしない。施策の性質(holdoutあり/なし)で因果手法を振り分け、有意性を判定し、勝ち筋を商品・時間帯・オファーのどれに帰属させ、似た構造の未実施店を選び、必ず1店ずつドライラン→本適用で反映し、最後に「展開したこと自体のlift」を対照で検証するところまでを1本に通す。横展開の意思決定と、撤退の意思決定を、同じ土俵で扱う。

## 使うMCPツール(rocky-regi)
- get_campaign_roi — holdoutありクーポンのlift/ROI/posterior_prob。主軸1。
- get_intervention_effect — holdoutなし値上げ・プロモのCausalImpact rel_effect/posterior_prob。主軸2。
- get_menu_engineering / get_business_summary — 勝ち筋の裏取り(商品か・時間帯か・全体地力か)。
- get_menu_abc_analysis — 勝ち商品の位置づけ(主力/準主力)確認。
- list_stores / get_store_context / get_store_insights — 展開先候補の絞り込みと「まだ打っていない店」判定。
- get_menu_and_courses / query_records — 勝ち商品が展開先の在庫・メニューに存在するか機械照合。
- simulate_price_change — 値上げ系施策の展開先での余地事前試算(読取専用・価格は変えない)。
- get_goal_progress — 展開後の対照検証で使う。
- create_store_promo / update_store_promo — プロモの店別本適用(要人間承認)。
- create_visibility_schedule / update_visibility_schedule — 露出スケジュールの店別本適用(要人間承認)。
- create_coupon_distribution_scenario / set_coupon_distribution_scenario_enabled / update_coupon_distribution_scenario / cancel_coupon_distribution — クーポン施策の展開・停止(holdout必須、要人間承認)。

Meta広告側のlift評価は Meta公式Ads MCP が想定連携だが、契約前は未接続。接続済みでない環境では触れない。#1674ポータル巡回はBOT規約リスクで保留のため、本スキルから起動しない。

## 手順(この順で多段に呼ぶ)

### 1. 介入棚卸し(何を効果検証にかけるか決める)
直近の施策を、holdoutあり/なしで2群に分ける。ADR-0069の二重計上回避に従い、同じ施策を両方の指標で語らない。
- holdoutあり(クーポン配布・LINEセグメント配信など) → get_campaign_roi
- holdoutなし(価格改定・全店プロモ・露出変更など) → get_intervention_effect

施策が1件もない期間なら「検証対象なし」と明示して終わる。作った施策を無理に効いたことにしない。

### 2. 有意性判定(当たり/効果なし/外れに仕分け)
両ツールが返す posterior_prob(≥0.95で有意)、lift または rel_effect、信頼区間を見て、次の3ラベルに仕分ける。
- 当たり:posterior_prob ≥ 0.95 かつ効果符号が想定方向
- 判断保留:posterior_prob < 0.95、または INSUFFICIENT_SAMPLE 相当
- 外れ/撤退候補:posterior_prob ≥ 0.95 かつ効果符号が逆(売上/客数を下げている)

NOT_SIGNIFICANT は必ず「判断保留」とラベルする。「効果があった気がする」で横展開に進めない。外れ側は step 6 で撤退フローに回す。

### 3. 勝ち筋の裏取り(何が効いたのかを言語化)
当たり施策について、勝因が「商品」「時間帯」「オファー(価格/クーポン)」「地力そのもの」のどれに宿っているかを分解する。
- get_menu_engineering で対象商品の象限変化(dog→plow horse など)を確認
- get_business_summary で同期間の全体売上・客数の伸びと突き合わせ、店全体が伸びただけの可能性を除外
- get_menu_abc_analysis で勝ち商品がA/B/Cのどこにいるかを添える

「全体地力が伸びていただけ」なら横展開は打ち止め。地力側の要因を探る別スキルに渡す。

### 4. 展開先の選定(まだ打っていない、似た店)
list_stores + get_store_context で業態・規模・客層が近い店をラフに束ね、その中から「まだ同種の施策を打っていない店」を候補に絞る。get_store_insights で「まだ勝ち筋を打っていない店」(target=abc/engineering/visit-source 系のwarning/info)が出ている店を優先。

owner/hq_admin の権限で見える店だけを候補にする。権限外の店は候補に含めない(422で弾かれる)。

### 5. 移植可否の機械照合(商品差・価格差を店ごとに確認)
候補店ごとに、勝ち施策をそのまま持ち込めるかを機械的に照合する。
- get_menu_and_courses / query_records で、勝ち商品(または相当メニュー)が候補店のメニューに存在するかチェック
- 値上げ系なら simulate_price_change で候補店の値上げ余地を事前試算(読取専用、実価格は動かさない)

出力は候補店ごとに次の3ラベル。
- そのまま移植可
- 代替商品で移植(代替候補を1つ提示)
- 移植不可(勝ち商品が存在しない/在庫扱いが違う)

### 6. 店別ドライラン→本適用(1店ずつ、必ず2段)
展開対象を「そのまま移植可」+「代替で移植」に絞ったうえで、必ず次の順で反映する。全店一括反映はしない。
1. まず対象N店・作成M件・各操作のSQL相当・可逆性(取り消せるか)を一覧で提示
2. オーナー/本部の明示承認を待つ
3. 承認後、1店目だけドライラン(create/update系は create_store_promo / create_visibility_schedule / create_coupon_distribution_scenario を対象1店に限定して実行)
4. 1店目の見え方(店側UI・配信状態・クーポン holdout 分割)を確認してから2店目以降を1店ずつ本適用

外れ施策(step 2で撤退候補)は、cancel_coupon_distribution または set_coupon_distribution_scenario_enabled=false / update_store_promo で停止に回す。停止も同じ2段(提示→承認→1店ずつ)を踏む。

store_manager / store_staff の権限しか無いユーザーで本スキルを走らせようとした場合、書込ツールは422で弾かれる。スキル側で「本部権限が必要」と明示して止める。

### 7. 事後の対照検証(横展開そのもののliftを測る)
展開から一定期間(店の営業日で最低2週)経ったら、次を並べて比較する。
- 展開店群 vs 展開しなかった同型店群(step 4の候補から未展開に残った店)
- get_goal_progress の到達率差
- 展開したクーポン施策側で get_campaign_roi の店間比較

ここで初めて「横展開そのものが効いたか」を語る。展開店の地力が元から強かっただけ、という交絡を切り分ける工程を省かない。

## 出力の型(表現ゆれを防ぐため固定)

期間・対象施策
- 期間:YYYY-MM-DD 〜 YYYY-MM-DD
- 検証対象:N件(holdoutあり X件 / holdoutなし Y件)

効果判定
- 当たり:施策名 / lift or rel_effect / posterior_prob / 実施店
- 判断保留:施策名 / 理由(サンプル不足 or posterior_prob < 0.95)
- 外れ・撤退候補:施策名 / 逆効果の大きさ / 実施店

勝ち筋の帰属(当たり施策ごと)
- 主要因:商品 / 時間帯 / オファー / 地力
- 裏付け:menu_engineering の象限変化、business_summary の全体差

展開先候補
- 候補店:N店(業態・規模で類似)
- そのまま移植可:M店 / 代替で移植:K店 / 移植不可:L店

実行プラン(店別・可逆性つき)
- 店A:create_store_promo(可逆) / create_coupon_distribution_scenario(holdout付)
- 店B:代替商品Xで update_visibility_schedule(可逆)
- …

ドライラン→本適用の状況
- 1店目ドライラン結果:OK / NG(理由)
- 本適用済み:N/M店

次に見るべきこと
- 展開から2週後に get_goal_progress と get_campaign_roi の対照比較
- 外れ施策の撤退店リスト

UIウィジェットが有効な場面では、当たり施策のダッシュボードに show_sales_dashboard、勝ち筋の商品裏取りに show_menu_engineering、クーポン系の状態確認に show_approval_queue を添える。SSRで生成する店長向け文面は参考シナリオ止まりで、断定・予言はしない。

## やってはいけないこと(ガード)
- 全店一括反映をしない。必ず1店ずつドライラン→本適用の2段を踏む。
- posterior_prob < 0.95 を「効いた」と言い換えない。「判断保留」で止める。
- holdoutあり施策とholdoutなし施策を同じ指標で並べない(ADR-0069の二重計上回避)。
- 全体売上が伸びていただけの店を「施策が効いた」と扱わない。地力と切り分ける工程を省略しない。
- 権限不足の店を候補に混ぜない。書込は必ず capability で自動判定し、権限外店はスキップ表示。
- 書込系(create_store_promo / create_coupon_distribution_scenario / create_visibility_schedule 等)は、対象店・件数・SQL相当・可逆性を提示し、人間承認を得てから実行する。
- レート制限を超える反復呼び出しをしない。候補店が多いときは step 4 で先に絞ってから step 5 の照合に入る。
- Meta公式Ads MCPは未接続環境では触れない。#1674ポータル巡回(BOT規約リスクで保留)は本スキルから起動しない。
- 読み取り(get_*)と操作(create_/update_/cancel_/set_*_enabled)を混在させて一気に走らせない。判定と反映は必ず分ける。
- SSRで生成する店長向け文面を予言・断定にしない(「必ず伸びます」ではなく「同型店ではliftが観測されました」)。
- 「レジのAIが自動で横展開した」と表現しない。判断と承認は人間、レジは供給と受け皿。