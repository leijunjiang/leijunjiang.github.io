---
layout: post
title:  "Rails: Best Practices"
date:   2024-12-24 14:31:20 +0200
categories: jekyll update
tags: software best practices
---

## 【Database】
#  Don't add indexes on low-cardinality fields (like status, enums) < 100 distinct values

- Even when the index is used, it's not efficient because".
   - For a field like `status: ['pending', 'completed']`, each value appears in ~50% of rows
  - The DB still needs to scan half the table, making the index practically useless
  - You're paying the cost of index maintenance for no real benefit
- When querying with status + other fields (like id), the query planner might wrongly choose the status index over more selective indexes

#  Always review migrations before deployment, handle large tables manually

- tables with millions of records can lock for significant periods during migrations
  - Lock duration 
    - 1M rows: seconds of lock
    - 10M rows: minutes of lock
    - 100M+ rows: potential hours of lock
- Use zero-downtime migration strategies
  - 1: Add column without default
  - 2: Backfill data in batches
  - 3: Add index concurrently (PostgreSQL)

#  In Rails, beware of before_action at the top of the controller as transactions start at the first query.

- Problem:

- ```ruby
class OrdersController < ApplicationController
  before_action :authenticate_user! # Starts transaction here with user query
  before_action :fetch_external_data # Long HTTP call while transaction is open
  
  def create
    @order = Order.create!(params) # DB operation inside transaction
  end
end
```

- Better Approach:

- ```ruby
class OrdersController < ApplicationController
  skip_before_action :authenticate_user!
  def create
    # Do HTTP calls first
    external_data = fetch_external_data
    # Then start transaction with auth
    authenticate_user!
    # DB operations last, keeping transaction short
    @order = Order.create!(params.merge(external_data: external_data))
  end
end
```

# Missing unique constraint in database will cause a data integrity issue

- database-level unique constraint
  - ```ruby
    add_index :users, :email, unique: true
    ```
- Model-level unique Validation is not enough
  - ```ruby
    validates :email, presence: true, uniqueness: true # This is not enough
    ```
  - model validation flow:
    - ```ruby
      def validate_uniqueness
        # 2 processes checking the same thing 
        # if email exists in the database
        existing_record = User.where(email: email).exists?

        # 2 processes both get the same result, there is no email
        # 2 processes both create user with the same email
        if existing_record.nil?
          # create
          User.create!(email: email)
        end
      end
    ```
- conclusion:
  - Race condition in high concurrency
  - Duplication of data
  - Fixation of data is complex and risky

## 【Async Jobs】
# Sidekiq
- Idempotency! At least ensure database operations are idempotent
  - external API calls are not idempotent, should be placedat the end of the process
  - when facing OOMkilled, k8s allow sidekiq 10 seconds to finish the job, make sure that Sidekiq can finish the job within 10 seconds
- Make sure retry does not send too much error emails
- Use defensive code to avoid errors
  - data can be corrupted in the middle of the process
  - the data is already corrupted (e.g. user already deleted softly by gem paranoid). I get fired for this.

## 【Business】
# Never trust IDs from frontend, always verify
-  ownership/authorization
    - never use Order.find(params[:id]), instead use current_user.orders.find(params[:id])
- security
  - Use UUID for IDs, so that hackers can't guess the next ID

## 【Security】
# Never hardcode secrets in codebase (from SMS template_ids to JWT secrets) 
- prefer environment variables, database
- use Rails.application.credentials

## 【Cache】

- Don't abuse Rails.cache methods. For Redis keys over 1KB, implement two-level caching
- ```ruby
class DashboardCache
    def self.fetch_for_user(user)
      # Level 1: Store only IDs/metadata in Redis
      cache_keys = Rails.cache.fetch("dashboard_keys_#{user.id}", expires_in: 1.hour) do
        {
        stats_key: "stats_#{user.id}#{Time.current.to_i}",
        reports_key: "reports_#{user.id}#{Time.current.to_i}"
        }
      end

      # Level 2: Store actual data in separate keys
      stats = Rails.cache.fetch(cache_keys[:stats_key], expires_in: 1.hour) do
        DashboardService.generate_stats(user)
      end

      reports = Rails.cache.fetch(cache_keys[:reports_key], expires_in: 1.hour) do
        DashboardService.generate_reports(user)
      end
      
      { stats: stats, reports: reports }
    end
end
```

## 【Code】
Before development: create branch, write design doc, get review, then code

## 【Performance】
- Most performance issues come from database or external dependencies 
  - check N+1 queries first
  - implement monitoring

- Memory
  - QRCode, Image generation, Excel, PDF generation 
  - heavy dependencies in Dockerfile, ImageMagick, wkhtmltopdf etc
    - Docker image size 
    - memory consumption at runtime
      - PDF >= 300MB
      - Image >= 100MB
      - Excel >= 100MB
    
  - use serverless AWS Lambdato handle
    - ```ruby
    response = HTTP.post(
ENV['PDF_GENERATOR_URL'],
json: { template: 'reports/show', data: @report.data }
)
```


