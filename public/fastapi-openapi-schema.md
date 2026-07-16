---
title: FastAPIのレスポンス定義を見直してOpenAPIスキーマとのズレを減らした話
tags:
  - Python
  - OpenAPI
  - FastAPI
  - API設計
  - pydantic
private: false
updated_at: '2026-07-16T22:18:35+09:00'
id: 14dbaa95c5ee734703a7
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# FastAPIのレスポンス定義を見直してOpenAPIスキーマとのズレを減らした話

## はじめに

既存のFastAPIプロジェクトに参加すると、APIの実装だけでなく、OpenAPIスキーマがフロントエンドや型生成にどう使われているかを確認する場面があります。

FastAPIでは、Pydanticモデルや `response_model` をもとにOpenAPIスキーマが生成されます。  
そのため、APIレスポンスの型定義が曖昧だと、フロントエンド側の型生成や実装にも影響します。

この記事では、既存FastAPIプロジェクトでOpenAPIスキーマを確認するときに見るポイントをまとめます。

## 1. response_modelが定義されているか

まず確認するのは、エンドポイントに `response_model` が定義されているかです。

```python
@app.get("/articles/{article_id}", response_model=ArticleResponse)
def get_article(article_id: int):
    ...
```

`response_model` があると、APIのレスポンス構造が明示され、OpenAPIスキーマにも反映されます。

逆に、`response_model` がない場合、実際にどのようなレスポンスが返るのかを実装から読み解く必要があります。

## 2. Pydanticモデルでレスポンスの形を表現する

次に確認するのは、Pydanticモデルでレスポンスの形がどこまで表現されているかです。

### 必須フィールド

```python
from pydantic import BaseModel

class ArticleResponse(BaseModel):
    id: int
    title: str
```

この場合、`id` と `title` は必須フィールドとしてOpenAPIスキーマに反映されます。

### nullableなフィールド

値が存在しない可能性がある場合は、`None` を許容する型として表現します。

```python
class ArticleResponse(BaseModel):
    id: int
    title: str
    category_name: str | None = None
```

このように書くと、`category_name` は `string` または `null` として扱われます。

フロントエンド側でOpenAPIスキーマからTypeScript型を生成している場合、以下のような型になります。

```ts
type ArticleResponse = {
  id: number;
  title: string;
  category_name: string | null;
};
```

### 任意フィールドとnullableの違い

注意したいのは、「フィールドが存在しない可能性がある」と「値がnullになる可能性がある」は別物という点です。

```python
class ArticleResponse(BaseModel):
    id: int
    title: str
    summary: str | None = None
```

この場合、`summary` は省略可能かつ `None` も許容されます。

一方で、レスポンスに必ず `summary` キーを含めたい場合は、実装側で明示的に `None` を返すようにします。

```python
return ArticleResponse(
    id=1,
    title="FastAPI example",
    summary=None,
)
```

フロントエンド側では、キーが存在しないのか、値が `null` なのかで扱いが変わることがあります。  
APIの契約としてどちらを許容するのかを決めておくことが重要です。

### ネストしたオブジェクト

レスポンスにネストしたオブジェクトを含める場合は、モデルを分けて定義します。

```python
class AuthorResponse(BaseModel):
    id: int
    name: str

class ArticleResponse(BaseModel):
    id: int
    title: str
    author: AuthorResponse
```

OpenAPIスキーマ上でも `author` がオブジェクトとして表現されます。

### 配列

一覧APIなどでは、配列をレスポンスに含めることがあります。

```python
class ArticleResponse(BaseModel):
    id: int
    title: str

class ArticleListResponse(BaseModel):
    items: list[ArticleResponse]
    total: int
```

```python
@app.get("/articles", response_model=ArticleListResponse)
def list_articles():
    return ArticleListResponse(
        items=[
            ArticleResponse(id=1, title="First article"),
            ArticleResponse(id=2, title="Second article"),
        ],
        total=2,
    )
```

### enum

決まった値だけを返す場合は、`Enum` を使うとスキーマにも反映されます。

```python
from enum import Enum

class ArticleStatus(str, Enum):
    draft = "draft"
    published = "published"
    archived = "archived"

class ArticleResponse(BaseModel):
    id: int
    title: str
    status: ArticleStatus
```

フロントエンド側では、文字列リテラル型やenum相当の型として扱いやすくなります。

### Fieldで説明や例を追加する

`Field` を使うと、OpenAPIスキーマに説明や例を追加できます。

```python
from pydantic import BaseModel, Field

class ArticleResponse(BaseModel):
    id: int = Field(..., description="記事ID", examples=[1])
    title: str = Field(..., description="記事タイトル", examples=["FastAPI example"])
    category_name: str | None = Field(
        None,
        description="カテゴリ名。未分類の場合はnull",
        examples=["Python"],
    )
```

OpenAPIスキーマを読む人にとって、説明や例があるとAPIの意図が分かりやすくなります。

## 3. 実際の返却値とモデルが一致しているか

`response_model` が定義されていても、実装側の返却値がモデルの意図とずれていることがあります。

確認するポイントは以下です。

- 必須フィールドが常に返っているか
- nullableなフィールドに `None` が入り得るか
- DBや外部APIの値をそのまま返していないか
- レスポンス用に整形する層があるか
- モデル定義とテストの期待値が一致しているか

特に既存APIでは、後からフィールドが追加されたり、データ取得元が変わったりして、モデルと実態が少しずつズレることがあります。

## 4. OpenAPIスキーマ上でどう見えるか

FastAPIでは、通常 `/openapi.json` や `/docs` からスキーマを確認できます。

確認する項目は以下です。

- `required` に入っているフィールド
- `null` が許容されているか
- 配列やネストしたオブジェクトの型
- enumの値
- descriptionやexample
- エラーレスポンスの定義

OpenAPIスキーマは、バックエンド実装者だけでなく、フロントエンド実装者にとっても重要な契約になります。

## 5. エラーレスポンスを定義する

正常系のレスポンスだけでなく、エラー時のレスポンスもOpenAPIスキーマに定義しておくと、フロントエンド側で扱いやすくなります。

### 共通エラーモデルを定義する

```python
class ErrorResponse(BaseModel):
    code: str
    message: str
```

### responsesでエラーレスポンスを指定する

FastAPIでは、デコレータの `responses` にステータスコードごとのレスポンスを定義できます。

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class ArticleResponse(BaseModel):
    id: int
    title: str

class ErrorResponse(BaseModel):
    code: str
    message: str

@app.get(
    "/articles/{article_id}",
    response_model=ArticleResponse,
    responses={
        404: {
            "model": ErrorResponse,
            "description": "記事が見つからない場合",
        },
    },
)
def get_article(article_id: int):
    if article_id == 404:
        raise HTTPException(
            status_code=404,
            detail={
                "code": "ARTICLE_NOT_FOUND",
                "message": "Article not found",
            },
        )

    return ArticleResponse(
        id=article_id,
        title="FastAPI example",
    )
```

このように定義すると、OpenAPIスキーマ上に404レスポンスの形式が反映されます。

### HTTPExceptionのdetailとのズレに注意する

注意点として、FastAPIの `HTTPException` は標準では `detail` に値を入れます。

つまり、上記のように `detail` にオブジェクトを入れた場合、実際のレスポンスは次のようになります。

```json
{
  "detail": {
    "code": "ARTICLE_NOT_FOUND",
    "message": "Article not found"
  }
}
```

一方で、`responses` に指定した `ErrorResponse` は以下の形を表します。

```json
{
  "code": "ARTICLE_NOT_FOUND",
  "message": "Article not found"
}
```

この2つは形が違います。

そのため、エラーレスポンスを厳密に共通化したい場合は、例外ハンドラを定義してレスポンス形式を揃える必要があります。

### 例外ハンドラでエラー形式を揃える

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()

class ErrorResponse(BaseModel):
    code: str
    message: str

class AppException(Exception):
    def __init__(self, status_code: int, code: str, message: str):
        self.status_code = status_code
        self.code = code
        self.message = message

@app.exception_handler(AppException)
async def app_exception_handler(
    request: Request,
    exc: AppException,
):
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            code=exc.code,
            message=exc.message,
        ).model_dump(),
    )
```

```python
@app.get(
    "/articles/{article_id}",
    response_model=ArticleResponse,
    responses={
        404: {
            "model": ErrorResponse,
            "description": "記事が見つからない場合",
        },
    },
)
def get_article(article_id: int):
    if article_id == 404:
        raise AppException(
            status_code=404,
            code="ARTICLE_NOT_FOUND",
            message="Article not found",
        )

    return ArticleResponse(
        id=article_id,
        title="FastAPI example",
    )
```

このようにすると、OpenAPIスキーマ上の定義と実際のエラーレスポンスの形を揃えやすくなります。

## 6. フロントエンドの型生成にどう影響するか

OpenAPIスキーマからTypeScriptの型を生成している場合、バックエンド側の定義がそのままフロントエンドの型に影響します。

例えば、バックエンドで `str | None` と定義していれば、フロントエンドでは以下のような型になります。

```ts
type ArticleResponse = {
  id: number;
  title: string;
  category_name: string | null;
};
```

この型があることで、フロントエンドではnullチェック漏れに気づきやすくなります。

また、エラーレスポンスもスキーマに定義しておくと、フロントエンド側でエラー処理の型を扱いやすくなります。

## まとめ

FastAPIでは、`response_model` とPydanticモデルを確認することで、APIレスポンスの契約を読み解けます。

既存プロジェクトでAPIを修正するときは、実装だけでなく、以下を確認すると安全です。

- `response_model` が定義されているか
- Pydanticモデルでnullableやenumが表現されているか
- 実際の返却値とモデルが一致しているか
- OpenAPIスキーマ上でどう見えるか
- エラーレスポンスの形式が定義されているか
- フロントエンドの型生成にどう影響するか

OpenAPIスキーマは、バックエンドとフロントエンドの契約です。  
FastAPIではその契約をコードから生成できるため、既存APIを安全に修正するうえで重要な確認ポイントになります。
