# 🗄️ Week 1-2: Database Setup and Migration Guide

## 🎯 Overview
This phase focuses on setting up the database schema and migrations for the Hangmeas Super App, extending Spree Commerce with custom tables and fields.

---

> **Streaming Infrastructure**: For streaming service database requirements and media storage considerations, see [STREAMING_INFRASTRUCTURE.md](../../docs/technical/STREAMING_INFRASTRUCTURE.md)

## 📋 Setup Checklist

### Week 1: Core Database Migrations

#### Day 1: Environment Setup
- [ ] Ensure Rails and Spree are properly configured
- [ ] Set up database.yml for development, test, and production
- [ ] Configure Redis for ActionCable and caching
- [ ] Install required gems (add to Gemfile)
- [ ] Run `bundle install`

#### Day 2-3: User and Authentication Enhancements
- [ ] Generate and run `AddHangmeasUserFields` migration
- [ ] Add indexes for performance
- [ ] Create user seed data for testing
- [ ] Test user model extensions

#### Day 4-5: XFactor Voting System Tables
- [ ] Generate and run `CreateXfactorVoting` migration
- [ ] Add foreign key constraints
- [ ] Create indexes for voting queries
- [ ] Set up database views for vote counting

### Week 2: Advanced Features Setup

#### Day 1-2: Enhanced Store Credits
- [ ] Generate and run `EnhanceStoreCredits` migration
- [ ] Add multi-currency support
- [ ] Create transaction type indexes
- [ ] Test store credit extensions

#### Day 3-4: Subscription and Events
- [ ] Generate and run `CreateSubscriptions` migration
- [ ] Generate and run `EnhanceEventsSystem` migration
- [ ] Add composite indexes
- [ ] Create seed data for plans and events

#### Day 5: Additional Tables and Optimization
- [ ] Create supporting tables (loyalty_activities, achievements, etc.)
- [ ] Add database constraints and validations
- [ ] Optimize indexes based on query patterns
- [ ] Run database performance analysis

---

## 🔨 Detailed Migration Files

### 1. Gemfile Additions

```ruby
# Add to your Gemfile
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

# Existing Spree gems...

# Additional gems for Hangmeas features
gem 'redis', '~> 5.0'
gem 'sidekiq', '~> 7.1'
gem 'whenever', require: false # For cron jobs
gem 'pg', '~> 1.5' # PostgreSQL adapter
gem 'friendly_id', '~> 5.5' # For SEO-friendly URLs
gem 'jwt', '~> 2.7' # For API authentication
gem 'rack-cors', '~> 2.0' # For CORS support
gem 'image_processing', '~> 1.12' # For Active Storage variants
gem 'aws-sdk-s3', '~> 1.134' # For S3 storage
gem 'fcm', '~> 1.0' # For push notifications
gem 'twilio-ruby', '~> 6.3' # For SMS
gem 'aasm', '~> 5.5' # For state machines
gem 'paper_trail', '~> 14.0' # For audit trails
gem 'ransack', '~> 4.0' # For advanced search
gem 'kaminari', '~> 1.2' # For pagination
gem 'bullet', group: :development # For N+1 query detection
gem 'annotate', group: :development # For model annotation

# Testing gems
group :development, :test do
  gem 'rspec-rails', '~> 6.0'
  gem 'factory_bot_rails', '~> 6.2'
  gem 'faker', '~> 3.2'
  gem 'shoulda-matchers', '~> 5.3'
  gem 'database_cleaner-active_record', '~> 2.1'
end
```

### 2. Database Configuration

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: hangmeas_development
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  host: <%= ENV.fetch('DATABASE_HOST', 'localhost') %>
  port: <%= ENV.fetch('DATABASE_PORT', 5432) %>

test:
  <<: *default
  database: hangmeas_test
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  host: <%= ENV.fetch('DATABASE_HOST', 'localhost') %>

production:
  <<: *default
  database: hangmeas_production
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  host: <%= ENV['DATABASE_HOST'] %>
  port: <%= ENV.fetch('DATABASE_PORT', 5432) %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 25 } %>
```

### 3. User Enhancement Migration

```ruby
# db/migrate/[timestamp]_add_hangmeas_user_fields.rb
class AddHangmeasUserFields < ActiveRecord::Migration[7.0]
  def change
    # Basic user information
    add_column :spree_users, :phone_number, :string
    add_column :spree_users, :khmer_name, :string
    add_column :spree_users, :date_of_birth, :date
    add_column :spree_users, :gender, :string
    add_column :spree_users, :province, :string
    add_column :spree_users, :district, :string
    add_column :spree_users, :commune, :string
    add_column :spree_users, :village, :string
    
    # Loyalty and gamification
    add_column :spree_users, :loyalty_tier, :string, default: 'bronze'
    add_column :spree_users, :loyalty_points, :integer, default: 0
    add_column :spree_users, :total_spent, :decimal, precision: 10, scale: 2, default: 0
    add_column :spree_users, :total_saved, :decimal, precision: 10, scale: 2, default: 0
    add_column :spree_users, :total_votes_cast, :integer, default: 0
    
    # Referral system
    add_column :spree_users, :referral_code, :string
    add_column :spree_users, :referred_by_id, :integer
    add_column :spree_users, :referral_count, :integer, default: 0
    
    # Preferences and settings
    add_column :spree_users, :language_preference, :string, default: 'en'
    add_column :spree_users, :notification_preferences, :jsonb, default: {}
    add_column :spree_users, :marketing_consent, :boolean, default: false
    add_column :spree_users, :sms_consent, :boolean, default: false
    
    # Mobile app specific
    add_column :spree_users, :push_token, :string
    add_column :spree_users, :device_type, :string # 'ios', 'android'
    add_column :spree_users, :app_version, :string
    add_column :spree_users, :last_active_at, :datetime
    
    # Security and verification
    add_column :spree_users, :kyc_status, :string, default: 'pending'
    add_column :spree_users, :verification_level, :string, default: 'basic'
    add_column :spree_users, :id_card_number, :string
    add_column :spree_users, :id_card_verified_at, :datetime
    add_column :spree_users, :phone_verified_at, :datetime
    add_column :spree_users, :two_factor_enabled, :boolean, default: false
    add_column :spree_users, :pin_code, :string # Encrypted PIN for quick access
    
    # Account status
    add_column :spree_users, :account_status, :string, default: 'active'
    add_column :spree_users, :suspended_at, :datetime
    add_column :spree_users, :suspension_reason, :text
    
    # Add indexes for performance
    add_index :spree_users, :phone_number, unique: true
    add_index :spree_users, :referral_code, unique: true
    add_index :spree_users, :referred_by_id
    add_index :spree_users, :loyalty_tier
    add_index :spree_users, :kyc_status
    add_index :spree_users, :province
    add_index :spree_users, :push_token
    add_index :spree_users, :id_card_number
    add_index :spree_users, [:province, :district]
    add_index :spree_users, :last_active_at
    
    # Add foreign key constraint
    add_foreign_key :spree_users, :spree_users, column: :referred_by_id
  end
end
```

### 4. XFactor Voting System Migration

```ruby
# db/migrate/[timestamp]_create_xfactor_voting.rb
class CreateXfactorVoting < ActiveRecord::Migration[7.0]
  def change
    # Seasons table
    create_table :xfactor_seasons do |t|
      t.string :name, null: false
      t.text :description
      t.string :slug
      t.date :start_date
      t.date :end_date
      t.integer :status, default: 0 # enum: upcoming, active, completed
      t.string :banner_url
      t.string :logo_url
      t.jsonb :metadata, default: {}
      t.integer :total_votes_cast, default: 0
      t.decimal :total_revenue, precision: 10, scale: 2, default: 0
      t.timestamps
      
      t.index :slug, unique: true
      t.index :status
      t.index [:start_date, :end_date]
    end
    
    # Episodes table
    create_table :xfactor_episodes do |t|
      t.references :season, null: false, foreign_key: { to_table: :xfactor_seasons }
      t.string :title, null: false
      t.integer :episode_number, null: false
      t.text :description
      t.datetime :air_date
      t.datetime :voting_start
      t.datetime :voting_end
      t.integer :status, default: 0 # enum: upcoming, live, completed
      t.string :video_url
      t.string :thumbnail_url
      t.integer :views_count, default: 0
      t.integer :total_votes, default: 0
      t.jsonb :voting_results, default: {}
      t.timestamps
      
      t.index [:season_id, :episode_number], unique: true
      t.index :status
      t.index [:voting_start, :voting_end]
      t.index :air_date
    end
    
    # Contestants table
    create_table :xfactor_contestants do |t|
      t.references :season, null: false, foreign_key: { to_table: :xfactor_seasons }
      t.string :name, null: false
      t.string :khmer_name
      t.string :stage_name
      t.text :bio
      t.string :hometown
      t.string :province
      t.integer :age
      t.string :gender
      t.string :photo_url
      t.string :cover_photo_url
      t.integer :status, default: 0 # enum: active, eliminated, winner
      t.date :eliminated_date
      t.integer :final_rank
      t.integer :total_votes_received, default: 0
      t.jsonb :social_links, default: {}
      t.jsonb :achievements, default: {}
      t.timestamps
      
      t.index :status
      t.index :province
      t.index [:season_id, :status]
      t.index :total_votes_received
    end
    
    # Episode contestants join table
    create_table :episode_contestants do |t|
      t.references :episode, null: false, foreign_key: { to_table: :xfactor_episodes }
      t.references :contestant, null: false, foreign_key: { to_table: :xfactor_contestants }
      t.integer :performance_order
      t.string :song_performed
      t.string :performance_video_url
      t.integer :judges_score
      t.text :judges_comments
      t.boolean :eliminated, default: false
      t.timestamps
      
      t.index [:episode_id, :contestant_id], unique: true
      t.index [:episode_id, :performance_order]
    end
    
    # Votes table
    create_table :xfactor_votes do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.references :contestant, null: false, foreign_key: { to_table: :xfactor_contestants }
      t.references :episode, null: false, foreign_key: { to_table: :xfactor_episodes }
      t.string :vote_type, null: false # 'free', 'paid', 'sms', 'bundle'
      t.integer :vote_count, default: 1
      t.decimal :amount_paid, precision: 8, scale: 2
      t.string :payment_method
      t.string :transaction_id
      t.string :ip_address
      t.string :user_agent
      t.jsonb :metadata, default: {} # Device info, location, etc.
      t.datetime :expires_at # For time-limited votes
      t.timestamps
      
      t.index [:user_id, :episode_id, :vote_type]
      t.index [:contestant_id, :episode_id]
      t.index :vote_type
      t.index :created_at
      t.index :transaction_id
    end
    
    # Vote packages table
    create_table :vote_packages do |t|
      t.string :name, null: false
      t.string :sku, null: false
      t.integer :vote_count, null: false
      t.integer :bonus_votes, default: 0
      t.decimal :price, precision: 10, scale: 2, null: false
      t.string :currency, default: 'USD'
      t.boolean :active, default: true
      t.integer :sort_order, default: 0
      t.string :badge_text # e.g., "BEST VALUE", "POPULAR"
      t.jsonb :restrictions, default: {} # Min purchase, max per user, etc.
      t.datetime :valid_from
      t.datetime :valid_until
      t.timestamps
      
      t.index :sku, unique: true
      t.index :active
      t.index :sort_order
      t.index [:valid_from, :valid_until]
    end
    
    # SMS vote logs for carrier reconciliation
    create_table :sms_vote_logs do |t|
      t.references :user, foreign_key: { to_table: :spree_users }
      t.references :vote, foreign_key: { to_table: :xfactor_votes }
      t.string :phone_number, null: false
      t.string :carrier # 'smart', 'cellcard', 'metfone'
      t.string :short_code
      t.string :message_content
      t.string :carrier_message_id
      t.decimal :charge_amount, precision: 6, scale: 2
      t.string :status, default: 'pending' # pending, confirmed, failed
      t.datetime :confirmed_at
      t.timestamps
      
      t.index :phone_number
      t.index :carrier
      t.index :status
      t.index :carrier_message_id
      t.index [:created_at, :carrier]
    end
  end
end
```

### 5. Enhanced Store Credits Migration

```ruby
# db/migrate/[timestamp]_enhance_store_credits.rb
class EnhanceStoreCredits < ActiveRecord::Migration[7.0]
  def change
    # Add new columns to store credits
    add_column :spree_store_credits, :transaction_type, :string
    add_column :spree_store_credits, :source_type, :string
    add_column :spree_store_credits, :source_id, :string
    add_column :spree_store_credits, :recipient_id, :integer
    add_column :spree_store_credits, :sender_id, :integer
    add_column :spree_store_credits, :reference_number, :string
    add_column :spree_store_credits, :metadata, :jsonb, default: {}
    add_column :spree_store_credits, :expires_at, :datetime
    add_column :spree_store_credits, :expired, :boolean, default: false
    
    # Add indexes
    add_index :spree_store_credits, :transaction_type
    add_index :spree_store_credits, :source_type
    add_index :spree_store_credits, :reference_number, unique: true
    add_index :spree_store_credits, :recipient_id
    add_index :spree_store_credits, :sender_id
    add_index :spree_store_credits, :expires_at
    add_index :spree_store_credits, [:user_id, :currency]
    add_index :spree_store_credits, [:transaction_type, :created_at]
    
    # Add foreign keys
    add_foreign_key :spree_store_credits, :spree_users, column: :recipient_id
    add_foreign_key :spree_store_credits, :spree_users, column: :sender_id
    
    # Create wallet transactions table for detailed history
    create_table :wallet_transactions do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.references :store_credit, foreign_key: { to_table: :spree_store_credits }
      t.string :transaction_type, null: false
      t.decimal :amount, precision: 10, scale: 2, null: false
      t.string :currency, null: false
      t.decimal :balance_before, precision: 10, scale: 2
      t.decimal :balance_after, precision: 10, scale: 2
      t.string :status, default: 'completed'
      t.string :reference_type
      t.integer :reference_id
      t.text :notes
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:user_id, :created_at]
      t.index :transaction_type
      t.index [:reference_type, :reference_id]
      t.index :status
    end
    
    # Create payment methods table
    create_table :user_payment_methods do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.string :payment_type, null: false # 'aba_pay', 'wing', 'card', etc.
      t.string :account_identifier # Last 4 digits, phone number, etc.
      t.string :account_name
      t.jsonb :encrypted_data # For storing encrypted sensitive data
      t.boolean :default, default: false
      t.boolean :active, default: true
      t.datetime :verified_at
      t.timestamps
      
      t.index [:user_id, :payment_type]
      t.index [:user_id, :default]
    end
  end
end
```

### 6. Subscription System Migration

```ruby
# db/migrate/[timestamp]_create_subscriptions.rb
class CreateSubscriptions < ActiveRecord::Migration[7.0]
  def change
    # Subscription plans
    create_table :subscription_plans do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      t.decimal :price, precision: 10, scale: 2, null: false
      t.string :currency, default: 'USD'
      t.string :interval, null: false # 'day', 'week', 'month', 'year'
      t.integer :interval_count, default: 1
      t.integer :trial_days, default: 0
      t.jsonb :features, default: {} # Feature list with limits
      t.jsonb :metadata, default: {}
      t.boolean :active, default: true
      t.integer :sort_order, default: 0
      t.string :badge_text
      t.boolean :recommended, default: false
      t.timestamps
      
      t.index :slug, unique: true
      t.index :active
      t.index :sort_order
      t.index [:active, :recommended]
    end
    
    # User subscriptions
    create_table :user_subscriptions do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.references :plan, null: false, foreign_key: { to_table: :subscription_plans }
      t.string :status, null: false # 'trial', 'active', 'cancelled', 'expired', 'paused'
      t.datetime :current_period_start
      t.datetime :current_period_end
      t.datetime :trial_start
      t.datetime :trial_end
      t.datetime :cancelled_at
      t.datetime :pause_start
      t.datetime :pause_end
      t.string :cancellation_reason
      t.string :payment_method
      t.string :stripe_subscription_id # If using Stripe
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:user_id, :status]
      t.index :status
      t.index :current_period_end
      t.index [:user_id, :plan_id]
    end
    
    # Subscription invoices
    create_table :subscription_invoices do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.references :subscription, null: false, foreign_key: { to_table: :user_subscriptions }
      t.string :number, null: false
      t.decimal :amount, precision: 10, scale: 2, null: false
      t.string :currency, default: 'USD'
      t.string :status, default: 'pending' # pending, paid, failed, refunded
      t.datetime :due_date
      t.datetime :paid_at
      t.string :payment_method
      t.string :transaction_id
      t.text :failure_reason
      t.jsonb :line_items, default: []
      t.timestamps
      
      t.index :number, unique: true
      t.index [:user_id, :status]
      t.index :status
      t.index :due_date
    end
    
    # Feature usage tracking
    create_table :feature_usages do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.references :subscription, foreign_key: { to_table: :user_subscriptions }
      t.string :feature_name, null: false
      t.integer :usage_count, default: 0
      t.date :usage_date, null: false
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:user_id, :feature_name, :usage_date], unique: true
      t.index [:subscription_id, :usage_date]
    end
  end
end
```

### 7. Events System Enhancement Migration

```ruby
# db/migrate/[timestamp]_enhance_events_system.rb
class EnhanceEventsSystem < ActiveRecord::Migration[7.0]
  def change
    # Add event-specific fields to products
    add_column :spree_products, :product_type, :string, default: 'physical'
    add_column :spree_products, :event_type, :string # 'concert', 'xfactor', 'show', 'movie'
    add_column :spree_products, :event_date, :datetime
    add_column :spree_products, :event_end_date, :datetime
    add_column :spree_products, :venue_name, :string
    add_column :spree_products, :venue_address, :text
    add_column :spree_products, :venue_latitude, :decimal, precision: 10, scale: 8
    add_column :spree_products, :venue_longitude, :decimal, precision: 11, scale: 8
    add_column :spree_products, :venue_capacity, :integer
    add_column :spree_products, :tickets_sold, :integer, default: 0
    add_column :spree_products, :sale_start_date, :datetime
    add_column :spree_products, :sale_end_date, :datetime
    add_column :spree_products, :artist_id, :integer
    add_column :spree_products, :organizer_id, :integer
    add_column :spree_products, :seating_chart_data, :jsonb, default: {}
    add_column :spree_products, :is_digital_event, :boolean, default: false
    add_column :spree_products, :stream_url, :string
    add_column :spree_products, :age_restriction, :integer
    add_column :spree_products, :event_status, :string, default: 'scheduled'
    
    # Add indexes
    add_index :spree_products, :product_type
    add_index :spree_products, :event_type
    add_index :spree_products, :event_date
    add_index :spree_products, [:venue_latitude, :venue_longitude]
    add_index :spree_products, :artist_id
    add_index :spree_products, :organizer_id
    add_index :spree_products, [:event_date, :event_status]
    
    # Artists table
    create_table :artists do |t|
      t.string :name, null: false
      t.string :khmer_name
      t.text :bio
      t.string :genre
      t.string :country
      t.string :photo_url
      t.string :cover_photo_url
      t.jsonb :social_links, default: {}
      t.integer :followers_count, default: 0
      t.boolean :verified, default: false
      t.timestamps
      
      t.index :name
      t.index :genre
      t.index :verified
    end
    
    # Event organizers
    create_table :event_organizers do |t|
      t.string :name, null: false
      t.string :company_name
      t.text :description
      t.string :contact_email
      t.string :contact_phone
      t.string :logo_url
      t.boolean :verified, default: false
      t.decimal :commission_rate, precision: 5, scale: 2
      t.jsonb :bank_details, default: {}
      t.timestamps
      
      t.index :name
      t.index :verified
    end
    
    # Seat management for events
    create_table :event_seats do |t|
      t.references :product, null: false, foreign_key: { to_table: :spree_products }
      t.string :section, null: false
      t.string :row
      t.string :seat_number
      t.string :seat_type # 'vip', 'standard', 'premium'
      t.decimal :price_modifier, precision: 8, scale: 2, default: 0
      t.string :status, default: 'available' # 'available', 'reserved', 'sold', 'blocked'
      t.references :line_item, foreign_key: { to_table: :spree_line_items }
      t.datetime :reserved_until
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:product_id, :section, :row, :seat_number], unique: true, name: 'index_seats_on_location'
      t.index [:product_id, :status]
      t.index :line_item_id
      t.index :reserved_until
    end
    
    # Event check-ins
    create_table :event_checkins do |t|
      t.references :order, null: false, foreign_key: { to_table: :spree_orders }
      t.references :line_item, null: false, foreign_key: { to_table: :spree_line_items }
      t.references :product, null: false, foreign_key: { to_table: :spree_products }
      t.references :user, foreign_key: { to_table: :spree_users }
      t.string :ticket_code, null: false
      t.datetime :checked_in_at
      t.string :checked_in_by
      t.string :gate
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index :ticket_code, unique: true
      t.index [:product_id, :checked_in_at]
      t.index :order_id
    end
    
    # Add foreign keys
    add_foreign_key :spree_products, :artists
    add_foreign_key :spree_products, :event_organizers, column: :organizer_id
  end
end
```

### 8. Supporting Tables Migration

```ruby
# db/migrate/[timestamp]_create_supporting_tables.rb
class CreateSupportingTables < ActiveRecord::Migration[7.0]
  def change
    # Loyalty activities
    create_table :loyalty_activities do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.integer :points, null: false
      t.string :activity_type, null: false # 'earn', 'redeem', 'expire', 'adjust'
      t.string :reason
      t.string :reference_type
      t.integer :reference_id
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:user_id, :created_at]
      t.index :activity_type
      t.index [:reference_type, :reference_id]
    end
    
    # User achievements
    create_table :achievements do |t|
      t.references :user, null: false, foreign_key: { to_table: :spree_users }
      t.string :code, null: false
      t.string :name, null: false
      t.text :description
      t.integer :points_awarded, default: 0
      t.datetime :achieved_at, null: false
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index [:user_id, :code], unique: true
      t.index :code
      t.index :achieved_at
    end
    
    # Push notifications
    create_table :push_notifications do |t|
      t.references :user, foreign_key: { to_table: :spree_users }
      t.string :title, null: false
      t.text :body, null: false
      t.string :notification_type
      t.jsonb :data, default: {}
      t.string :status, default: 'pending' # pending, sent, failed, read
      t.datetime :sent_at
      t.datetime :read_at
      t.string :failure_reason
      t.timestamps
      
      t.index [:user_id, :status]
      t.index :notification_type
      t.index :created_at
    end
    
    # App banners
    create_table :app_banners do |t|
      t.string :title
      t.text :description
      t.string :image_url, null: false
      t.string :link_type # 'product', 'category', 'url', 'xfactor', 'event'
      t.string :link_value
      t.integer :position, default: 0
      t.boolean :active, default: true
      t.datetime :start_date
      t.datetime :end_date
      t.string :target_audience # 'all', 'bronze', 'silver', 'gold', 'platinum'
      t.jsonb :metadata, default: {}
      t.timestamps
      
      t.index :position
      t.index :active
      t.index [:start_date, :end_date]
      t.index :target_audience
    end
    
    # Topup promotions
    create_table :topup_promotions do |t|
      t.string :name, null: false
      t.text :description
      t.string :promotion_type # 'percentage', 'fixed', 'tiered'
      t.decimal :bonus_percentage, precision: 5, scale: 2
      t.decimal :bonus_amount, precision: 10, scale: 2
      t.decimal :minimum_amount, precision: 10, scale: 2
      t.decimal :maximum_bonus, precision: 10, scale: 2
      t.string :payment_methods, array: true, default: []
      t.boolean :active, default: true
      t.datetime :start_date
      t.datetime :end_date
      t.integer :usage_limit
      t.integer :usage_count, default: 0
      t.jsonb :tiers, default: {} # For tiered bonuses
      t.timestamps
      
      t.index :active
      t.index [:start_date, :end_date]
      t.index :payment_methods, using: :gin
    end
    
    # Referral rewards
    create_table :referral_rewards do |t|
      t.references :referrer, null: false, foreign_key: { to_table: :spree_users }
      t.references :referred, null: false, foreign_key: { to_table: :spree_users }
      t.string :reward_type # 'points', 'credit', 'discount'
      t.decimal :reward_amount, precision: 10, scale: 2
      t.string :status, default: 'pending' # pending, qualified, paid
      t.datetime :qualified_at
      t.datetime :paid_at
      t.jsonb :qualifying_criteria, default: {}
      t.timestamps
      
      t.index [:referrer_id, :referred_id], unique: true
      t.index :status
      t.index :qualified_at
    end
  end
end
```

### 9. Database Views and Functions

```ruby
# db/migrate/[timestamp]_create_database_views.rb
class CreateDatabaseViews < ActiveRecord::Migration[7.0]
  def up
    # Create view for user wallet balances
    execute <<-SQL
      CREATE OR REPLACE VIEW user_wallet_balances AS
      SELECT 
        user_id,
        currency,
        SUM(amount) as balance,
        COUNT(*) as transaction_count,
        MAX(created_at) as last_transaction_at
      FROM spree_store_credits
      GROUP BY user_id, currency
      HAVING SUM(amount) > 0;
    SQL
    
    # Create view for voting statistics
    execute <<-SQL
      CREATE OR REPLACE VIEW voting_statistics AS
      SELECT 
        c.id as contestant_id,
        c.name as contestant_name,
        e.id as episode_id,
        e.episode_number,
        COUNT(DISTINCT v.user_id) as unique_voters,
        SUM(v.vote_count) as total_votes,
        SUM(CASE WHEN v.vote_type = 'free' THEN v.vote_count ELSE 0 END) as free_votes,
        SUM(CASE WHEN v.vote_type = 'paid' THEN v.vote_count ELSE 0 END) as paid_votes,
        SUM(CASE WHEN v.vote_type = 'sms' THEN v.vote_count ELSE 0 END) as sms_votes
      FROM xfactor_contestants c
      LEFT JOIN xfactor_votes v ON c.id = v.contestant_id
      LEFT JOIN xfactor_episodes e ON v.episode_id = e.id
      GROUP BY c.id, c.name, e.id, e.episode_number;
    SQL
    
    # Create materialized view for user analytics
    execute <<-SQL
      CREATE MATERIALIZED VIEW user_analytics AS
      SELECT 
        u.id,
        u.email,
        u.phone_number,
        u.loyalty_tier,
        u.loyalty_points,
        u.total_spent,
        COUNT(DISTINCT o.id) as order_count,
        COUNT(DISTINCT v.id) as vote_count,
        SUM(v.vote_count) as total_votes_cast,
        COUNT(DISTINCT sc.id) as wallet_transactions,
        u.created_at as member_since,
        u.last_active_at
      FROM spree_users u
      LEFT JOIN spree_orders o ON u.id = o.user_id AND o.state = 'complete'
      LEFT JOIN xfactor_votes v ON u.id = v.user_id
      LEFT JOIN spree_store_credits sc ON u.id = sc.user_id
      GROUP BY u.id;
      
      CREATE INDEX idx_user_analytics_loyalty_tier ON user_analytics(loyalty_tier);
      CREATE INDEX idx_user_analytics_total_spent ON user_analytics(total_spent);
    SQL
    
    # Function to calculate user lifetime value
    execute <<-SQL
      CREATE OR REPLACE FUNCTION calculate_user_ltv(user_id INTEGER)
      RETURNS TABLE(
        total_orders DECIMAL,
        total_subscriptions DECIMAL,
        total_votes DECIMAL,
        total_ltv DECIMAL
      ) AS $$
      BEGIN
        RETURN QUERY
        SELECT 
          COALESCE(SUM(o.total), 0) as total_orders,
          COALESCE(SUM(si.amount), 0) as total_subscriptions,
          COALESCE(SUM(v.amount_paid), 0) as total_votes,
          COALESCE(SUM(o.total), 0) + 
          COALESCE(SUM(si.amount), 0) + 
          COALESCE(SUM(v.amount_paid), 0) as total_ltv
        FROM spree_users u
        LEFT JOIN spree_orders o ON u.id = o.user_id AND o.state = 'complete'
        LEFT JOIN subscription_invoices si ON u.id = si.user_id AND si.status = 'paid'
        LEFT JOIN xfactor_votes v ON u.id = v.user_id AND v.vote_type = 'paid'
        WHERE u.id = user_id;
      END;
      $$ LANGUAGE plpgsql;
    SQL
  end
  
  def down
    execute "DROP VIEW IF EXISTS user_wallet_balances;"
    execute "DROP VIEW IF EXISTS voting_statistics;"
    execute "DROP MATERIALIZED VIEW IF EXISTS user_analytics;"
    execute "DROP FUNCTION IF EXISTS calculate_user_ltv(INTEGER);"
  end
end
```

### 10. Indexes and Performance Optimization

```ruby
# db/migrate/[timestamp]_add_performance_indexes.rb
class AddPerformanceIndexes < ActiveRecord::Migration[7.0]
  def change
    # Composite indexes for common queries
    add_index :spree_orders, [:user_id, :state, :created_at]
    add_index :spree_line_items, [:order_id, :variant_id]
    
    # Partial indexes for better performance
    add_index :xfactor_votes, :created_at, 
              where: "vote_type = 'paid'", 
              name: 'idx_paid_votes_created_at'
    
    add_index :spree_users, :created_at, 
              where: "kyc_status = 'verified'", 
              name: 'idx_verified_users_created_at'
    
    add_index :user_subscriptions, :current_period_end, 
              where: "status IN ('active', 'trial')", 
              name: 'idx_active_subscriptions_period_end'
    
    # BRIN indexes for time-series data (PostgreSQL specific)
    execute <<-SQL
      CREATE INDEX idx_votes_created_at_brin 
      ON xfactor_votes USING brin(created_at);
      
      CREATE INDEX idx_store_credits_created_at_brin 
      ON spree_store_credits USING brin(created_at);
      
      CREATE INDEX idx_loyalty_activities_created_at_brin 
      ON loyalty_activities USING brin(created_at);
    SQL
    
    # GIN indexes for JSONB columns
    add_index :spree_users, :notification_preferences, using: :gin
    add_index :xfactor_votes, :metadata, using: :gin
    add_index :wallet_transactions, :metadata, using: :gin
    
    # Enable pg_trgm extension for fuzzy search
    enable_extension 'pg_trgm'
    
    # Add trigram indexes for search
    add_index :spree_users, :khmer_name, using: :gin, opclass: :gin_trgm_ops
    add_index :xfactor_contestants, :name, using: :gin, opclass: :gin_trgm_ops
    add_index :artists, :name, using: :gin, opclass: :gin_trgm_ops
  end
end
```

---

## 🌱 Seed Data

### Create seed file for development

```ruby
# db/seeds/development.rb
puts "Seeding development data..."

# Create admin user
admin = Spree::User.create!(
  email: 'admin@hangmeas.com',
  password: 'admin123',
  phone_number: '012345678',
  khmer_name: 'អ្នកគ្រប់គ្រង',
  loyalty_tier: 'platinum',
  kyc_status: 'verified',
  verification_level: 'full'
)
admin.spree_roles << Spree::Role.find_or_create_by(name: 'admin')

# Create test users with different tiers
%w[bronze silver gold platinum].each_with_index do |tier, index|
  user = Spree::User.create!(
    email: "#{tier}@example.com",
    password: 'password123',
    phone_number: "0123456#{index}0",
    first_name: tier.capitalize,
    last_name: 'User',
    khmer_name: "អ្នកប្រើ#{tier}",
    loyalty_tier: tier,
    loyalty_points: index * 5000,
    total_spent: index * 500,
    kyc_status: 'verified'
  )
  
  # Add wallet balance
  Spree::StoreCredit.create!(
    user: user,
    amount: 100 + (index * 50),
    currency: 'USD',
    transaction_type: 'topup',
    source_type: 'seed',
    reason: 'Initial balance'
  )
end

# Create XFactor season
season = Xfactor::Season.create!(
  name: 'XFactor Cambodia 2024',
  description: 'The biggest singing competition in Cambodia',
  start_date: 1.month.ago,
  end_date: 2.months.from_now,
  status: 'active'
)

# Create contestants
10.times do |i|
  Xfactor::Contestant.create!(
    season: season,
    name: Faker::Name.name,
    khmer_name: "តារាចម្រៀង #{i+1}",
    bio: Faker::Lorem.paragraph,
    hometown: ['Phnom Penh', 'Siem Reap', 'Battambang', 'Kampot'].sample,
    province: ['Phnom Penh', 'Siem Reap', 'Battambang', 'Kampot'].sample,
    age: rand(18..35),
    gender: ['male', 'female'].sample,
    status: 'active'
  )
end

# Create episodes
5.times do |i|
  episode = Xfactor::Episode.create!(
    season: season,
    title: "Episode #{i+1}: #{['Auditions', 'Battle Round', 'Live Show', 'Semi-Final', 'Grand Final'][i]}",
    episode_number: i + 1,
    air_date: i.weeks.ago,
    voting_start: i.weeks.ago + 20.hours,
    voting_end: i.weeks.ago + 44.hours,
    status: i == 0 ? 'live' : 'completed'
  )
  
  # Add all contestants to episodes
  season.contestants.each_with_index do |contestant, idx|
    EpisodeContestant.create!(
      episode: episode,
      contestant: contestant,
      performance_order: idx + 1
    )
  end
end

# Create vote packages
[
  { name: 'Starter Pack', vote_count: 10, price: 0.99 },
  { name: 'Fan Pack', vote_count: 50, bonus_votes: 10, price: 3.99 },
  { name: 'Super Pack', vote_count: 200, bonus_votes: 50, price: 9.99 },
  { name: 'Ultimate Pack', vote_count: 1000, bonus_votes: 200, price: 39.99 }
].each do |package|
  VotePackage.create!(
    name: package[:name],
    sku: "VOTE_#{package[:vote_count]}",
    vote_count: package[:vote_count],
    bonus_votes: package[:bonus_votes] || 0,
    price: package[:price],
    badge_text: package[:vote_count] >= 200 ? 'BEST VALUE' : nil
  )
end

# Create subscription plans
[
  {
    name: 'Basic',
    price: 4.99,
    features: {
      'ad_free' => true,
      'early_access' => false,
      'exclusive_content' => false,
      'monthly_bonus_votes' => 10
    }
  },
  {
    name: 'Premium',
    price: 9.99,
    features: {
      'ad_free' => true,
      'early_access' => true,
      'exclusive_content' => true,
      'monthly_bonus_votes' => 50,
      'vip_support' => true
    },
    recommended: true
  },
  {
    name: 'VIP',
    price: 19.99,
    features: {
      'ad_free' => true,
      'early_access' => true,
      'exclusive_content' => true,
      'monthly_bonus_votes' => 200,
      'vip_support' => true,
      'meet_greet_access' => true,
      'backstage_content' => true
    }
  }
].each_with_index do |plan, index|
  SubscriptionPlan.create!(
    name: plan[:name],
    slug: plan[:name].downcase,
    price: plan[:price],
    interval: 'month',
    trial_days: 7,
    features: plan[:features],
    sort_order: index,
    recommended: plan[:recommended] || false
  )
end

# Create some artists
5.times do
  Artist.create!(
    name: Faker::Music.band,
    khmer_name: "ក្រុមតន្រ្តី",
    bio: Faker::Lorem.paragraph,
    genre: ['Pop', 'Rock', 'Traditional', 'Hip Hop'].sample,
    country: 'Cambodia',
    verified: [true, false].sample
  )
end

# Create event organizers
3.times do
  EventOrganizer.create!(
    name: Faker::Company.name,
    company_name: "#{Faker::Company.name} Entertainment",
    contact_email: Faker::Internet.email,
    contact_phone: "012#{rand(100000..999999)}",
    verified: true,
    commission_rate: [10, 12, 15].sample
  )
end

puts "Development seed completed!"
```

---

## 🔍 Verification Scripts

### Database health check script

```ruby
# lib/tasks/db_health_check.rake
namespace :db do
  desc "Check database setup and health"
  task health_check: :environment do
    puts "\n=== Database Health Check ==="
    
    # Check tables
    tables = ActiveRecord::Base.connection.tables
    required_tables = %w[
      spree_users xfactor_seasons xfactor_episodes xfactor_contestants
      xfactor_votes vote_packages subscription_plans user_subscriptions
      wallet_transactions event_seats artists
    ]
    
    puts "\n📊 Required Tables:"
    required_tables.each do |table|
      status = tables.include?(table) ? "✅" : "❌"
      puts "  #{status} #{table}"
    end
    
    # Check indexes
    puts "\n🔍 Key Indexes:"
    connection = ActiveRecord::Base.connection
    
    key_indexes = {
      'spree_users' => ['index_spree_users_on_phone_number', 'index_spree_users_on_referral_code'],
      'xfactor_votes' => ['index_xfactor_votes_on_user_id_and_episode_id_and_vote_type'],
      'spree_store_credits' => ['index_spree_store_credits_on_reference_number']
    }
    
    key_indexes.each do |table, indexes|
      next unless tables.include?(table)
      
      table_indexes = connection.indexes(table).map(&:name)
      indexes.each do |index|
        status = table_indexes.include?(index) ? "✅" : "❌"
        puts "  #{status} #{table}.#{index}"
      end
    end
    
    # Check row counts
    puts "\n📈 Table Statistics:"
    required_tables.each do |table|
      next unless tables.include?(table)
      
      count = ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM #{table}").first['count']
      puts "  #{table}: #{count} rows"
    end
    
    # Check constraints
    puts "\n🔒 Foreign Key Constraints:"
    foreign_keys = connection.foreign_keys('xfactor_votes')
    foreign_keys.each do |fk|
      puts "  ✅ #{fk.from_table}.#{fk.column} -> #{fk.to_table}"
    end
    
    puts "\n✨ Health check complete!"
  end
end
```

---

## 📚 Next Steps

After completing the database setup:

1. **Run migrations**:
   ```bash
   rails db:create
   rails db:migrate
   rails db:seed
   ```

2. **Verify setup**:
   ```bash
   rails db:health_check
   ```

3. **Generate model annotations**:
   ```bash
   rails annotate_models
   ```

4. **Create database backup**:
   ```bash
   pg_dump -h localhost -U postgres hangmeas_development > backup/initial_schema.sql
   ```

5. **Document any custom SQL functions or triggers**

6. **Set up database monitoring and slow query logging**

7. **Configure database connection pooling for production**

8. **Proceed to Week 3-4: Core Models and Services implementation**