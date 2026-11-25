---
name: rails-sidekiq-job
description: Use when adding, changing, or reviewing a Ruby on Rails background job — a Sidekiq worker or ActiveJob, what to enqueue, idempotency, retry and backoff policy, scheduling, or its spec. Triggers on "background job", "Sidekiq worker", "perform_async", "perform_later", "retries", "idempotent job", "enqueue", "move this off the request", or any work on an app/workers/*_worker.rb or app/jobs/*_job.rb.
metadata:
  version: 1.0.0
---

# Rails Sidekiq Job

A playbook for writing a background job that is **safe to run more than once, thin, and observable**. A job is not just "code that runs later" — it runs on a different process, after an unknown delay, possibly twice, possibly after a deploy. Write it for that reality. Work through the steps in order; skip one only when it genuinely does not apply, and say why.

## Principles

- **At-least-once, not exactly-once.** Retries, crashes, and redeliveries mean every job can run again. Correctness comes from **idempotency**, never from hoping it runs once.
- **Enqueue IDs, not objects.** Arguments are serialized to Redis as JSON and may sit in the queue across a deploy. Pass primitives the worker re-loads; never a whole model, a `User`, or anything with state that will be stale.
- **The worker is plumbing.** Dequeue → load → delegate → done. Business logic lives in a model or service object that is testable without Sidekiq.
- **Match the app you're in.** Follow the codebase's existing base worker, queue names, and retry conventions rather than inventing a second pattern.

## 1. Enqueue with small, serializable arguments

```ruby
# Good — a primitive the worker re-loads
OrderConfirmationWorker.perform_async(order.id)

# Bad — serializes a whole object; stale by the time it runs
OrderConfirmationWorker.perform_async(order)
```

Pass ids, strings, numbers, booleans. If you need several, pass several ids — not a hash of attributes that can drift from the record.

## 2. Keep the worker thin

```ruby
# app/workers/order_confirmation_worker.rb
class OrderConfirmationWorker
  include Sidekiq::Job

  sidekiq_options queue: :mailers, retry: 5

  def perform(order_id)
    order = Order.find(order_id)
    return if order.confirmation_sent?        # idempotency guard — see step 3

    OrderConfirmation.new(order).deliver       # real work lives here
  end
end
```

The worker loads, guards, and delegates. If `perform` grows an `if`/`case` over business rules, extract a service object and call it.

## 3. Make it idempotent (it will run more than once)

Before doing the work, check whether it is already done; after, record that it is. Use whichever fits:

- **A state check** — `return if order.confirmation_sent?`, then set the flag in the same transaction as the work.
- **A unique key** — a `processed_events` row with a unique index on the event id; insert it in the transaction, let a duplicate raise and no-op.
- **`find_or_create_by`** on a natural key instead of blind `create`.

```ruby
def perform(order_id)
  order = Order.find(order_id)
  Order.transaction do
    locked = order.lock!                        # SELECT ... FOR UPDATE
    return if locked.confirmation_sent?
    OrderConfirmation.new(locked).deliver
    locked.update!(confirmation_sent_at: Time.current)
  end
end
```

Re-running this job is now a safe no-op. That is the whole point.

## 4. Choose a deliberate retry policy

Don't accept the default blindly; decide:

- **Transient work** (a flaky API, a lock timeout) — let it retry (`retry: 5`) with Sidekiq's exponential backoff.
- **Permanently-doomed work** (bad argument, deleted record) — don't thrash the queue. Rescue the known error, log, and return; or raise `Sidekiq::JobRetry::Skip`.
- **`Model::RecordNotFound`** — usually means the row was deleted after enqueue; catch it and return rather than retrying 25 times.

```ruby
def perform(order_id)
  order = Order.find(order_id)
  # ...
rescue ActiveRecord::RecordNotFound
  Rails.logger.info("OrderConfirmationWorker: order #{order_id} gone, skipping")
end
```

## 5. Guard money and external side-effects

For anything that charges a card, sends a payout, or calls a third party:

- **Be conservative on retries.** An operation that isn't provably idempotent should be `retry: 0` or bounded, so a retry can't double-charge.
- **Lock the row** (`with_lock` / advisory lock) so two workers can't process the same record at once.
- **Record the external reference** (charge id, idempotency key) before and after the call, so a retry can detect "already done" instead of repeating it.
- Never leave money in a half-done state across a `raise`; wrap the state change and the side-effect so they commit together or not at all.

## 6. Keep jobs fast, scoped, and observable

- One job, one unit of work. To act on many records, **fan out** (enqueue one job per id) instead of one giant job that dies at record 4,000 and restarts from zero.
- Put slow/external work on its own queue so it can't starve latency-sensitive jobs.
- Log the ids you're acting on and raise on real failure — a swallowed exception is a silent data bug. Let it surface so Sidekiq retries and your error tracker sees it.

## 7. Test it

Test two things: that the **right job is enqueued with the right args**, and that **`perform` is correct and idempotent**.

```ruby
# spec/workers/order_confirmation_worker_spec.rb
require "sidekiq/testing"

RSpec.describe OrderConfirmationWorker do
  let(:order) { create(:order) }

  it "enqueues with the order id" do
    expect { OrderConfirmationWorker.perform_async(order.id) }
      .to change(OrderConfirmationWorker.jobs, :size).by(1)
    expect(OrderConfirmationWorker.jobs.last["args"]).to eq([order.id])
  end

  it "delivers once and is a no-op on a second run" do
    Sidekiq::Testing.inline! do
      expect { OrderConfirmationWorker.new.perform(order.id) }
        .to change { order.reload.confirmation_sent? }.from(false).to(true)
      expect { OrderConfirmationWorker.new.perform(order.id) }
        .not_to change { ActionMailer::Base.deliveries.size }
    end
  end
end
```

Run it red first if you're adding behaviour, then green.

## Checklist before you call it done

- [ ] Arguments are ids/primitives the worker re-loads — no objects, no stale attributes.
- [ ] Worker only loads, guards, delegates; real logic is in a model/service.
- [ ] Running the job twice is a safe no-op (state check, unique key, or lock).
- [ ] Retry policy is deliberate; doomed work returns instead of thrashing.
- [ ] Money/external calls are locked, bounded, and keyed so a retry can't double-act.
- [ ] Large work fans out into per-record jobs; slow work has its own queue.
- [ ] Spec covers enqueue args **and** that a second `perform` doesn't repeat the effect.
