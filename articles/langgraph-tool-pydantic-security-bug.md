---
title: "LangChain の @tool デコレータ、権限チェック引数を無言で消していた話"
emoji: "🔐"
type: "tech"
topics: ["langchain", "langgraph", "python", "security", "llm"]
published: true
---

## はじめに

LangGraph で ReAct Agent を構築する際、ツール実行時にサーバー側から権限情報を注入する設計を取っていました。

「LLM が生成した引数に権限情報を追加してツールを呼ぶ」——シンプルに見えるこの実装が、**テストを書くまで気づかなかったセキュリティバグ**を内包していました。

本記事では、精度テストを追加する過程でバグを発見し、修正するまでの流れを解説します。

## 背景：ツールへの権限注入設計

LangGraph の ReAct Agent では、ユーザーのロール（Admin / Manager / Member）に応じてアクセスできるプロジェクトを制限しています。

```
ユーザー → [agent_node] → ツール呼び出し決定
                         ↓
                      [tool_node] ← ここで権限情報を注入
                         ↓
                    各ツール関数（権限チェックを内部で実施）
```

各ツール関数は `_allowed_project_ids` という引数でアクセス可能なプロジェクト ID のリストを受け取り、スコープ外ならエラーを返す設計です。

```python
@tool("get_documents", args_schema=GetDocumentsInput)
def get_documents(
    project_id: str,
    doc_type: Optional[str] = None,
    _allowed_project_ids: Optional[list[str]] = None,  # サーバー側注入
) -> str:
    if _allowed_project_ids is not None and project_id not in _allowed_project_ids:
        return json.dumps({"error": "アクセス権限がありません"})
    # ...
```

`tool_node` では LLM が生成した引数に権限情報を付け足してからツールを実行します。

```python
def tool_node(state: AgentState) -> dict:
    ctx = state.get("user_context", {})

    for tool_call in last_message.tool_calls:
        tool_args = tool_call["args"]

        # 権限情報を注入
        tool_args["_allowed_project_ids"] = ctx.get("assigned_project_ids")
        tool_args["_user_id"] = ctx.get("user_id")

        matched = next((t for t in ALL_TOOLS if t.name == tool_call["name"]), None)
        output = matched.invoke(tool_args)  # ← ここに問題があった
```

一見正しそうに見えます。しかし、この `matched.invoke(tool_args)` が問題でした。

## バグの原因：pydantic が引数を無言で除去する

LangChain の `@tool` デコレータは内部で `StructuredTool` を生成します。`StructuredTool.invoke()` を呼ぶと、**pydantic による入力バリデーションを経由**します。

ここが落とし穴です。`GetDocumentsInput` スキーマには `_allowed_project_ids` を定義していません。

```python
class GetDocumentsInput(BaseModel):
    project_id: str
    doc_type: Optional[str] = None
    # _allowed_project_ids は意図的にスキーマに含めない
    # （LLM から直接指定されるべきでないため）
```

pydantic はスキーマ外のフィールドをデフォルトで無視するため、`invoke()` を通ると `_allowed_project_ids` は関数に届く前に消えてしまいます。

```
tool_args = {
    "project_id": "p-secret",
    "_allowed_project_ids": ["p1", "p2"],  # ← invoke() の中で除去される
}

↓ matched.invoke(tool_args) の内部

関数が受け取る引数:
    project_id = "p-secret"
    _allowed_project_ids = None  # デフォルト値が使われる
```

そして関数内の権限チェックはこうなっています。

```python
if _allowed_project_ids is not None and project_id not in _allowed_project_ids:
    return json.dumps({"error": "アクセス権限がありません"})
```

`None` は「制限なし（= 全件アクセス可）」として扱われているため、**Member ロールのユーザーでも全プロジェクトのデータを取得できる状態**でした。

## テストで発見

精度テストを追加する中で `test_member_cannot_access_unauthorized_project` を書いたことで発覚しました。

```python
def test_member_cannot_access_unauthorized_project(self):
    result_str = t.get_documents.func(
        project_id="p-secret",
        _allowed_project_ids=["p1", "p2"],  # p-secret は含まない
    )
    result = json.loads(result_str)
    assert "error" in result  # 権限エラーが返るはず
```

テストが失敗し、返ってきたのはエラーではなくドキュメントデータ。原因を追うと `.invoke()` 経由で呼ぶと `_allowed_project_ids` が届いていないことがわかりました。

## 修正：pydantic を介さず直接呼び出す

`StructuredTool` は元の関数を `.func` 属性として持っています。これを直接呼ぶことで pydantic のバリデーションをバイパスできます。

合わせて `inspect.signature` で各ツール関数が受け付ける引数だけを渡すようにしました（ツールによって受け付ける引数が異なるため）。

```python
import inspect

def tool_node(state: AgentState) -> dict:
    ctx = state.get("user_context", {})

    for tool_call in last_message.tool_calls:
        tool_args = tool_call["args"]
        tool_args["_allowed_project_ids"] = ctx.get("assigned_project_ids")
        tool_args["_user_id"] = ctx.get("user_id")

        matched = next((t for t in ALL_TOOLS if t.name == tool_call["name"]), None)
        try:
            # pydantic を介さず .func() で直接呼び出す
            params = inspect.signature(matched.func).parameters
            has_var_kw = any(
                p.kind == inspect.Parameter.VAR_KEYWORD for p in params.values()
            )
            filtered_args = tool_args if has_var_kw else {
                k: v for k, v in tool_args.items() if k in params
            }
            output = matched.func(**filtered_args)
        except Exception as e:
            output = json.dumps({"error": str(e)})
```

`inspect.signature` で絞り込む理由は、ツールによって受け付ける引数が異なるためです。たとえば `get_user_context(user_id: str)` に `_allowed_project_ids` を渡すと `TypeError` になります。

## 修正後のテスト結果

```
tests/test_agent_accuracy.py::TestSecurityEnforcement::test_member_cannot_access_unauthorized_project PASSED
tests/test_agent_accuracy.py  14 passed
```

## まとめ

| | 修正前 | 修正後 |
|---|---|---|
| 呼び出し方 | `matched.invoke(tool_args)` | `matched.func(**filtered_args)` |
| pydantic バリデーション | 通る（引数が除去される） | バイパス |
| 権限チェック | 常に `None` = 全件アクセス可 | 正しく動作 |

LangChain の `@tool` + pydantic の組み合わせは、**スキーマに定義していないフィールドを黙って除去する**という挙動を持っています。セキュリティ目的でサーバー側から引数を注入する設計を取る場合は、`.invoke()` を使わず `.func()` で直接呼び出すことを検討してください。

テストを書かなければ本番リリースまで気づかなかったバグでした。AIを使った実装ほど、ちゃんとテストを書くことの重要性を改めて感じました。
