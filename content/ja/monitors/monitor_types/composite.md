---
title: 複合条件モニター
kind: documentation
aliases:
  - /ja/guides/composite_monitors
description: 複数のモニターを組み合わせた式に対してアラートする
further_reading:
  - link: monitors/notifications
    tag: Documentation
    text: モニター通知の設定
  - link: monitors/downtimes
    tag: Documentation
    text: モニターをミュートするダウンタイムのスケジュール
  - link: monitors/monitor_status
    tag: Documentation
    text: モニターステータスを確認
---
## 概要

複合条件モニターは、個別モニターを組み合わせて 1 つのモニターにし、さらに詳細なアラート条件を定義します。

既存のモニターを選択して、複合条件モニターを作成します。たとえば、モニター `A` とモニター `B` とします。次に、`A && B` などのブール演算子を使用してトリガー条件を設定します。複合条件モニターは、個別モニターが同時に複合条件モニターのトリガー条件を真にする値を持つときにトリガーします。

構成の目的上、複合条件モニターは構成モニターに依存しません。複合条件モニターの通知ポリシーは、構成モニターのポリシーに影響することなく変更できます。その逆も同様です。さらに、複合条件モニターを削除しても、構成モニターは削除されません。複合条件モニターは他のモニターを所有せずに、その結果を使用するだけです。複数の複合条件モニターが同じ個別モニターを参照することもあります。

**注**: `個別モニター`、`構成モニター`、`非複合条件モニター` という用語はすべて、複合条件モニターがそのステータスを計算するために使用するモニターを指します。

## モニターの作成

Datadog で[複合条件モニター][1]を作成するには、メインナビゲーションを使用して次のように移動します: *Monitors --> New Monitor --> Composite*。

### モニターの選択とトリガー条件の設定

#### モニターの選択

複合条件モニターで使用する個別モニターを最大 10 個まで選択できます。異なるアラートタイプのモニターでもかまいません（シンプルアラートのみ、マルチアラートのみ、または両方の組み合わせ）。複合条件モニターを個別モニターとして使用することはできません。最初のモニターを選択すると、UI にそのアラートタイプと現在のステータスが表示されます。

マルチアラートモニターを選択すると、UI にはモニターの group-by 節と、現在レポートされている一意のソースの数が表示されます（例: `Returns 5 host groups`）。マルチアラートモニターを組み合わせる場合、この情報は自然にペアになるモニターを選択するのに役立ちます。

同じグループを持つモニターを選択する必要があります。そうしなかった場合、UI には、このような複合条件モニターは決してトリガーしない旨の警告が表示されます。

```text
Group Matching Error
The selected monitors in the expression may not lead to a
composite result because they have not evaluated any
common groupings or have less than 2 selected monitors.
```

同じグループのマルチアラートモニターを選択した場合でも、モニターに共通のレポートソース（共通のグループ化とも呼ばれる）がない場合、`Group Matching Error` が表示されることがあります。共通のレポートソースがない場合、Datadog は複合条件モニターのステータスを計算できず、トリガーしません。 ただし、警告を無視して、とにかくモニターを作成することは_可能_です。詳細については、[複合条件モニターが共通の報告元ソースを選択する方法](#select-monitors-and-set-triggering-conditions)を参照してください。

警告が表示されないように 2 つ目のモニターを選択すると、**Trigger when** フィールドにデフォルトのトリガー条件 `a && b` が挿入され、提案された複合条件モニターのステータスが表示されます。

#### トリガー条件を設定する

**Trigger when** フィールドに、ブール演算子を使用して目的のトリガー条件を記述します。個別モニターは、フォーム内のラベル (`a`、`b`、`c` など) で識別されます。括弧を使用して演算子の優先度を制御すると、複雑な条件を作成できます。

以下はどれも有効なトリガー条件です。

```text
!(a && b)
a || b && !c
(a || b) && (c || d)
```

#### データなし

複合条件モニターがデータなしの状態にあることを `Do not notify`（通知しない）か `Notify`（通知する）かを選択します。ここでどちらを選択しても、個別モニターの `Notify no data` 設定には影響しませんが、複合条件がデータなしでアラートを出すためには、個別モニターと複合モニターの両方をデータが欠落していることを `Notify`（通知する）に設定する必要があります。

### 通知

**Say what's happening** と **Notify your team** のセクションに関する詳しい説明は、[通知][2]のページを参照してください。

## API

API を使用する場合、非複合条件モニターのクエリは、メトリクス、タグ、`avg` などの集計関数、group-by 節などをカプセル化できます。複合条件モニターのクエリは、モニター ID を使用する構成モニターに関して定義されます。

たとえば、2 つの非複合条件モニターに次のクエリと ID があるとします。
```text
"avg(last_1m):avg:system.mem.free{role:database} < 2147483648" # Monitor ID: 1234
"avg(last_1m):avg:system.cpu.system{role:database} > 50" # Monitor ID: 5678
```

上記のモニターを組み合わせた複合条件モニターのクエリは、`"1234 && 5678"` や `"!1234 || 5678"` などになります。

## 複合条件モニターの仕組み

このセクションでは、いくつかのサンプルを使用してトリガー条件が計算される方法と、さまざまなシナリオで受け取るアラートの数について説明します。

### トリガー条件の計算

Datadog は `A && B && C` を予想どおりに計算しますが、どのモニターステータスがアラート対象と見なされるのでしょうか？複合条件モニターが考慮するステータスには、以下の 6 つがあります。

| ステータス    | アラート対象                             |
|-----------|------------------------------------------|
| `Ok`      | 偽                                    |
| `Warn`    | 真                                     |
| `Alert`   | 真                                     |
| `Skipped` | 偽                                    |
| `No Data` | 偽（`notify_no_data` が真の場合は真） |
| `Unknown` | 真                                     |

複合条件モニターがアラート対象と評価する場合は、個別モニターの中で重大度が最高のステータスを継承し、アラートをトリガーします。複合条件モニターが非アラート対象と評価する場合は、重大度が**最低**のステータスを継承します。

not (!) 演算子は、個別または複合条件モニターステータスを `Ok`、`Alert`、または `No Data` のいずれかに変更します。たとえば、モニター `A` がいずれかのアラート対象ステータスである場合、`!A` は `OK` になります。モニター `A` がいずれかの**非**アラート対象ステータスである場合、`!A` は `Alert` になります。モニター `A` が `No Data` ステータスである場合は、`!A` も `No Data` になります。

3 つの個別モニター（`A`、`B`、`C`）とトリガー条件 `A && B && C` を使用する複合条件モニターを考えます。次の表は、個別モニターにさまざまなステータスが与えられた場合の複合条件モニターの結果ステータスです。アラート対象かどうかは、T または F で示されます。

| モニター A   | モニター B   | モニター C   | 複合条件ステータス | アラートがトリガーされるか？ |
|-------------|-------------|-------------|------------------|------------------|
| Unknown (T) | Warn (T)    | Unknown (T) | Warn (T)         | {{< X >}}        |
| Skipped (F) | Ok (F)      | Unknown (T) | Ok (F)           |                  |
| Alert (T)   | Warn (T)    | Unknown (T) | Alert (T)        | {{< X >}}        |
| Skipped (F) | No Data (F) | Unknown (T) | Skipped (F)      |                  |

この 4 つのシナリオのうち 2 つでは、すべての個別モニターが最高の重大度ステータス `Alert` を持っているわけではありませんが、アラートがトリガーします。

### コンポーネントモニターステータス

複合条件モニターは、各コンポーネントモニターのモニター結果のスライディングウィンドウを使用して評価されます。具体的には、**各コンポーネントモニターに対して過去 5 分間で最も重大なステータスを使用します**。たとえば、`A && B` として定義された複合条件モニターに次のコンポーネント結果がある場合（タイムスタンプは 1 分間隔）、`A` が技術的に `OK` 状態であっても、複合条件モニターは T1 でトリガーします。

| モニター | T0    | T1    | T2    |
|---------|-------|-------|-------|
| A       | Alert | OK    | OK    |
| B       | OK    | Alert | Alert |

この動作の正当性は、アラートシステムにとって同時性を定義することが非常に難しい問題であることにあります。モニターはさまざまなスケジュールに従って評価されますが、メトリクスのレイテンシーにより、おそらく同時に発生した 2 つのイベントであっても、モニターが最終的に評価される際には異なる時間に起こる可能性があります。複合条件モニターのアラートは、単に現在の状態をサンプリングするだけでは見逃されてしまう可能性があります。

この動作の結果、コンポーネントモニターが解決してから複合条件モニターが解決するまでに数分かかることがあります。

### アラートの数

受け取るアラートの数は、個別モニターのアラートタイプによって異なります。すべての個別モニターがシンプルアラートなら、複合条件モニターもシンプルアラートタイプになります。`A`、`B`、`C` のクエリがすべて同時に `true` の場合に、複合条件モニターは 1 つの通知をトリガーします。

個別モニターが 1 つでもマルチアラートなら、複合条件モニターもマルチアラートです。同時にいくつのアラートが送信されるかは、複合条件モニターが使用しているマルチアラートモニターが 1 つか複数かに依存します。

#### マルチアラートモニターが 1 つの場合

モニター `A` が `host` でグループ化されたマルチアラートモニターというシナリオを考えます。このモニターに報告元ソースが 4 つある場合は (ホスト `web01` ～ `web04`)、Datadog がこの複合条件モニターを評価するたびに、最大 4 つのアラートを受け取る可能性があります。言い換えると、1 回の評価サイクルで Datadog は 4 つのケースを考慮します。ケースごとに、モニター `A` のステータスはソースによって変わる可能性がありますが、モニター `B` と `C` (シンプルアラート) のステータスは変化しません。

次の表は、複合条件モニター `A && B && C` のある時点での各マルチアラートケースのステータスを示しています。

| ソース | モニター A | モニター B | モニター C | 複合条件ステータス | アラートがトリガーされるか？ |
|--------|-----------|-----------|-----------|------------------|------------------|
| web01  | Alert     | Warn      | Alert     | Alert            | {{< X >}}        |
| web02  | Ok        | Warn      | Alert     | Ok               |                  |
| web03  | Warn      | Warn      | Alert     | Alert            | {{< X >}}        |
| web04  | Skipped   | Warn      | Alert     | Skipped          |                  |

#### 複数のマルチアラートモニター

今度は、モニター`A` と `B` はマルチアラートで、ホストでグループ化されているというシナリオを考えます。1 サイクルあたりのアラート数は、最大でもモニター `A` とモニター `B` に共通する報告元ソースの数になります。`web01` ～ `web05` がモニター `A` の報告元で、`web04` ～ `web09` がモニター `B` の報告元の場合、複合条件モニターは、共通するソース（`web04` と `web05`）のみを考慮します。1 回の評価サイクルで、最大 2 つのアラートを受け取る可能性があります。

複合条件モニター `A && B && C` のサイクルの例を以下に示します。

| ソース | モニター A | モニター B | モニター C | 複合条件ステータス | アラートがトリガーされるか？ |
|--------|-----------|-----------|-----------|------------------|------------------|
| web04  | Unknown   | Warn      | Alert     | Alert            | {{< X >}}        |
| web05  | Ok        | Ok        | Alert     | Ok               |                  |

### 共通の報告元ソース

多くのマルチアラートモニターを使用する複合条件モニターでは、個別モニターの*共通の報告元ソース*のみが考慮されます。上記の例では、共通のソースは `host:web04` と `host:web05` です。

複合条件モニターはタグ*値*（`web04`）のみを参照し、タグ*キー*（`host`）は参照しません。上記の例に、単一の報告元ソース `environment:web04` を持つ `environment` によってグループ化されたマルチアラートモニター `D` が含まれている場合、複合条件モニターは `web04` を `A`、`B`、`D` 間の単一の共通の報告元ソースと見なします。

共通のタグ値を持たずに同じタグ名でグループ化されているマルチアラートモニターを使用して、複合条件モニターを作成できます。これは、共有タグ名が共通の報告元ソースになり得るからです。これらの値は将来一致する可能性があります。そのため、上の例では、`host:web04` と `host:web05` が共通の報告元ソースと見なさます。

たとえば、モニター `A` の `web04` と `web05`、およびモニター `D` の `dev` と `prod` のように、異なるタグでグループ化された 2 つのモニターが重複する値を持つことはめったにありません。それらが重複する場合は、これらのモニターで構成される複合条件モニターがアラートをトリガーできるようになります。

2 つ以上のタグで分割されたマルチアラートモニターの場合、モニターグループはタグの組み合わせ全体に対応します。たとえば、モニター `1` が `device,host` に基づくマルチアラートであり、モニター `2` が `host` に基づくマルチアラートである場合、複合条件モニターはモニター `1` とモニター `2` を組み合わせることができます。

{{< img src="monitors/monitor_types/composite/multi-alert-1.png" alt="書き込み通知"  style="width:80%;">}}

ただし、`host,url` に基づくマルチアラート、モニター `3` を考えると、モニター `1` とモニター `3` の組み合わせは、グルーピングが大きく異なるために、複合条件の結果にならない可能性があります。
{{< img src="monitors/monitor_types/composite/multi-alert-2.png" alt="書き込み通知" style="width:80%;">}}

最善の判断で意味のあるマルチアラートモニターを選択してください。

## その他の参考資料

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://app.datadoghq.com/monitors#create/composite
[2]: /ja/monitors/notifications