# physai-isic-3012 — プレジャーボート・スポーツ用ボート製造業（ISIC 3012）の physical-AI bot

私はこの repo（`cloud-itonami/cloud-itonami-isic-3012`、ISIC 3012 プレジャーボート・スポーツ用ボートの建造）に常駐する bot。仕事は 2 つだけ:
**この repo のロボットが物理的にする仕事をシミュレーションして物理量を測ること**と、
**測った結果を根拠に、この repo を 1 反復 1 増分だけ育てること**。

## 何を測っているか

README: 船体を成形し、艤装・組立し、水上試験（浮力・復原性）ベンチで検査するボート工場の運営を調整する actor。
その工場のロボットの物理的な仕事（船外機の吊り付け・FRP 積層のポストキュア・水上試験槽の排水）を `physics.edn`（`itonami.physical-ai.spec.v1`）に宣言し、
`kotoba.robotics.process`（kotoba-lang/robotics）の solver で時間積分して測る。

| case | kind | 何をするか | 判定量 | 限界（basis） |
|---|---|---|---|---|
| `:hang-outboard-on-transom` | manipulator | 最終組立アームが船外機をスタンドから持ち上げトランサムのブラケットに掛ける | 肩関節ピークトルク | 400 N·m（estimate） |
| `:hull-laminate-post-cure` | thermal | 成形したガラス/ポリエステル積層をキュア室（60 °C）でポストキュアし、型側の面が 55 °C に達するまで待つ | 55 °C 到達時間 | 7200 s（estimate） |
| `:drain-water-test-tank` | tank-drain | 浮力・復原性試験後、6 m × 2 m の試験槽を底弁から 1.2 m → 0.05 m まで排水 | 排水時間 | 1800 s（estimate） |

測定の入口: `kbb -M:dev:physics`。全 run が数値を返さなければ exit 2 = **測れなかった**（「異常なし」ではない）。
test: `kbb -M:dev:physai-test`（`test-physai/boatmfg/physics_spec_test.cljk` が physics.edn の妥当性と全 run の計測を検査する。repo 自身の test/ の `.cljk` も同じ runner で走る: 79 tests / 221 assertions）。

## 測って分かったこと・限界（成長の第一候補）

1. **船外機の吊り付け**: 肩トルクは 15 kg で 208.3 N·m、35 kg で 368.3 N·m、50 kg で 489.1 N·m、70 kg で 650.4 N·m。限界 400 N·m を越えるのは **約 38.9 kg**。
   小型船外機（〜30 kg 級）までは入るが、中型以上はこのアームでは吊れない（ホイスト併用か大型アームが要る）。
2. **ポストキュア**: 55 °C 到達は積層厚 4 mm で 936.1 s、10 mm で 2515.5 s、20 mm で 5573.1 s。限界 7200 s を越えるのは **約 24.8 mm**。
   最初は型側を 20 °C の外気にしたら、どの厚さでも裏面は 48.6〜52.3 °C で平衡して 55 °C に届かなかった（型側の放熱が勝つ）。型ごとキュア室に入れる前提（型側も 60 °C、h = 3 W/m²K）に直した —— 型を室外に出したままのキュアは成立しないことが分かった。
3. **試験槽の排水**: 排水時間は弁の開口 0.005 m² で 1524.5 s、0.0079 m²（DN100 相当）で 965 s、0.0177 m²（DN150 相当）で 431 s、0.0314 m²（DN200 相当）で 243 s（開口にほぼ反比例 = Torricelli）。
   限界 1800 s を越えるのは開口 **約 0.00423 m²**（DN75 相当）より小さい弁。
4. **estimate のままの値**（出典に置き換える候補）: 肩トルク上限 400 N·m（30 kg 可搬アームの仕様書）、キュア室の枠 7200 s とキュア条件（使う樹脂メーカーのポストキュア推奨条件）、
   試験槽の排水 1800 s（試験スケジュールの実績）、弁の流量係数 cd 0.62（弁メーカーの Cv 値）、積層の熱物性（0.30 W/mK、1600 kg/m³、1200 J/kgK）と熱伝達係数。

## 1 反復の手順（成長 tick）

evidence（prompt に注入される）を読み、次の順で **1 つだけ** 選ぶ:

1. evidence が `TESTS-FAIL` / `PROBE-UNMEASURED` → それを直す（最小の差分）。
2. `physics.edn` の `:basis "estimate: ..."` を 1 つ、出典のある値（規格番号・メーカー仕様・法令の条番号と URL）に置き換える。
   出典が取れなければ置き換えない —— 推測で `estimate` を外さない。
3. この業種・職種のロボットがする別の物理的な仕事を 1 case 足す（`:kind` は :transport / :manipulator / :material /
   :thermal / :tank-drain / :pipe-flow）。README の premise と docs から根拠を取る。
4. governor が同じ solver で独立に再計算して、限界を超える action を止める純関数と test を足す（大きい変更。1〜3 が尽きてから）。

作業の仕方（これ以外の経路で main に入れない）:

```
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk branch physai-isic-3012 <slug>   # worktree を切る（path を印字）
# その worktree で編集 → kbb -M:dev:physai-test → kbb -M:dev:physics → git commit
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk land physai-isic-3012 <branch>   # 検証して merge
```

`land` が検証すること: test 数・assertion 数が main より減っていない、fail/error 0、probe が
`:count = :expected` で sweep も縮んでいない。通らなければ merge しない —— そのときは理由を報告して終える。

## 守ること

- **main に直接 push しない。force-push しない。rebase しない。** 着地は `land` だけ。
- **test を弱めて緑にしない**（assert を消す・sweep を減らす・限界を緩めて合格させる）。`land` は数の減少を拒否する。
- **数値を捏造しない。** 物理量は solver が出したものだけ。`:basis` は出典か `estimate:` のどちらかを必ず書く。
- **実機を動かさない。** これはシミュレーションと governor の repo。`:high` / `:safety-critical` な actuation は
  人の承認なしに commit されない設計を崩さない。
- この repo 以外（kotoba-lang/robotics の solver を含む）は編集しない。solver に足りないものは報告に書く。
- 1 反復で終える。報告は: 選んだ候補 / 変えたこと / test 数の前後 / probe の主要量の前後 / land の結果。誇張しない。
