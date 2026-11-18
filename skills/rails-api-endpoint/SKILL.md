---
name: rails-api-endpoint
description: Use when adding, changing, or reviewing a Ruby on Rails JSON API endpoint — a new controller action, route wiring, strong params, serialization, authorization, error handling, pagination, or its request spec. Triggers on "add an endpoint", "new Rails API action", "render json", "serializer", "strong params", "Pundit policy", "request spec", or any work on an app/controllers/*_controller.rb that answers JSON.
metadata:
  version: 1.0.0
---

# Rails API Endpoint

A playbook for building a Rails JSON endpoint that is **thin, authorized, consistently shaped, and tested**. Work through the steps in order; skip one only when it genuinely does not apply, and say why.

## Principles

- **Controllers are thin**: authenticate → authorize → load → delegate → render. No business logic and no SQL beyond scoped finders. Push real work into models or service objects.
- **One success shape, one error shape**, used by *every* endpoint. A client should never have to special-case a response.
- **Answer the unhappy paths too**: unauthenticated, unauthorized, not found, and invalid input each get a correct status code and the standard error body.
- **Match the app you're in.** If the codebase already has a serializer, an error convention, or a base controller, follow it rather than inventing a second pattern.

## 1. Route it RESTfully

Prefer `resources` over hand-written paths. Reach for the seven standard actions first; use `member`/`collection` only for genuine sub-resources.

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :articles, only: %i[index show create update destroy]
  end
end
```

Run `bin/rails routes | grep articles` and confirm the path, verb, and controller match your intent.

## 2. Keep the controller thin

```ruby
# app/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < Api::V1::BaseController
      before_action :set_article, only: %i[show update destroy]

      def index
        articles = policy_scope(Article).includes(:author).page(params[:page])
        render json: ArticleSerializer.new(articles).serializable_hash
      end

      def create
        article = current_user.articles.build(article_params)
        authorize article
        if article.save
          render json: ArticleSerializer.new(article).serializable_hash, status: :created
        else
          render_errors(article)
        end
      end

      private

      def set_article
        @article = Article.find(params[:id])
        authorize @article
      end

      def article_params
        params.require(:article).permit(:title, :body, :published)
      end
    end
  end
end
```

Everything here is loading, authorizing, rendering. If a method grows an `if`/`case` over business rules, extract a service object.

## 3. Whitelist input with strong params

Never pass `params` straight to a model. `require` the wrapper key and `permit` only the attributes this action may set. Permit nested attributes explicitly (`permit(tags: [])`). Do **not** permit `role`, `user_id`, or other authorization-relevant fields from the client.

## 4. Authorize every action

Authorization is not optional and not the same as authentication. Use a policy object (Pundit shown; adapt to the app's library):

- `authorize @record` on show/update/destroy.
- `policy_scope(Model)` on index — so users only ever see rows they may see.
- Return **403** on a policy failure, **404** when hiding existence is safer.

A missing `authorize` call is an IDOR waiting to happen. If the app uses Pundit, enable `after_action :verify_authorized` in the base controller so a forgotten check fails loudly.

## 5. Serialize an explicit success shape

Do not `render json: @record` and leak every column. Use a serializer that names its fields:

```ruby
class ArticleSerializer
  include JSONAPI::Serializer # or ActiveModel::Serializer, or a plain PORO
  attributes :id, :title, :body, :published, :created_at
end
```

Decide once whether the body is the bare object or wrapped (`{ "data": ... }`) and keep it uniform.

## 6. Use one error shape

Pick a single structure and funnel all failures through a helper in the base controller:

```ruby
# app/controllers/api/v1/base_controller.rb
rescue_from ActiveRecord::RecordNotFound do
  render json: { errors: [{ detail: "Not found" }] }, status: :not_found
end

rescue_from Pundit::NotAuthorizedError do
  render json: { errors: [{ detail: "Forbidden" }] }, status: :forbidden
end

def render_errors(record, status: :unprocessable_entity)
  errors = record.errors.map { |e| { field: e.attribute, detail: e.full_message } }
  render json: { errors: errors }, status: status
end
```

Status codes: `201` create, `200` read/update, `204` destroy, `401` unauthenticated, `403` forbidden, `404` not found, `422` validation, `500` only for genuine bugs. Never return `200` with an error body.

## 7. Avoid N+1 and unbounded results

- Preload associations the serializer touches: `.includes(:author)`.
- Paginate every `index` (`kaminari`/`pagy`); never return an unbounded collection.
- Scope reads through the current user, not `Model.all`.

## 8. Write the request spec

A request spec is the contract. Cover the happy path **and** the unhappy paths:

```ruby
# spec/requests/api/v1/articles_spec.rb
RSpec.describe "Api::V1::Articles", type: :request do
  let(:user) { create(:user) }

  describe "POST /api/v1/articles" do
    it "creates and returns 201 for a signed-in user" do
      sign_in user
      post "/api/v1/articles", params: { article: { title: "Hi", body: "..." } }
      expect(response).to have_http_status(:created)
      expect(response.parsed_body.dig("data", "attributes", "title")).to eq("Hi")
    end

    it "returns 401 when unauthenticated" do
      post "/api/v1/articles", params: { article: { title: "Hi" } }
      expect(response).to have_http_status(:unauthorized)
    end

    it "returns 422 with the error shape on invalid input" do
      sign_in user
      post "/api/v1/articles", params: { article: { title: "" } }
      expect(response).to have_http_status(:unprocessable_entity)
      expect(response.parsed_body["errors"]).to be_present
    end
  end
end
```

Run it red first if you're adding behaviour, then green.

## Checklist before you call it done

- [ ] Route is RESTful and `bin/rails routes` confirms verb + path.
- [ ] Controller only authenticates, authorizes, loads, delegates, renders.
- [ ] Strong params `require` + `permit`; no authorization fields permitted.
- [ ] `authorize` on member actions, `policy_scope` on index.
- [ ] Success shape comes from a named serializer, not the raw record.
- [ ] Every failure returns the standard error body with the right status.
- [ ] Associations preloaded; `index` paginated.
- [ ] Request spec covers created / unauthenticated / unauthorized / invalid.
