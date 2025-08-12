# 🛒 Spree Commerce Super App Implementation Guide
## Building on Your Existing Spree + Store Credit Foundation

> **Multi-Show Framework**: This implementation extends the unified talent show architecture defined in [TALENT_SHOWS_FRAMEWORK.md](../../docs/TALENT_SHOWS_FRAMEWORK.md)

> **Show-Specific Guides**:
> - XFactor: See [XFACTOR_COMPLETE_GUIDE.md](../../docs/talent-shows/xfactor/XFACTOR_COMPLETE_GUIDE.md) and [XFACTOR_TECHNICAL_IMPLEMENTATION.md](../../docs/talent-shows/xfactor/XFACTOR_TECHNICAL_IMPLEMENTATION.md)
> - Cambodia's Got Talent: See [CGT_IMPLEMENTATION_GUIDE.md](../../docs/talent-shows/cgt/CGT_IMPLEMENTATION_GUIDE.md)
> - The Voice Cambodia: See [VOICE_IMPLEMENTATION_GUIDE.md](../../docs/talent-shows/voice/VOICE_IMPLEMENTATION_GUIDE.md)

---

## 🎯 Current State Assessment

### ✅ What You Already Have
- **Spree Commerce** core e-commerce functionality
- **Store Credit** system for wallet/gift card features
- **Event management** capabilities
- **Basic product catalog**

### 🚀 What We Need to Add
- **Multi-Talent Show voting** system (XFactor, CGT, Voice)
- **Universal voting architecture** with show-specific features
- **Subscription management** for all shows
- **Advanced loyalty program** with cross-show benefits
- **P2P transfers** and enhanced wallet
- **Mobile API endpoints** with dynamic show support
- **Real-time features** for all show formats

---

## 🏗️ Spree Extension Strategy

### 📦 Required Spree Extensions & Custom Gems

#### **Core Extensions to Install**
```ruby
# Gemfile additions
gem 'spree_api', '~> 4.6'
gem 'spree_auth_devise', '~> 4.5'
gem 'spree_multi_currency', '~> 4.3'
gem 'spree_digital', '~> 4.2'        # For digital products
gem 'spree_subscription', '~> 3.1'   # Subscription management
gem 'spree_wishlist', '~> 4.1'       # User wishlists
gem 'spree_social', '~> 5.0'         # Social auth
gem 'spree_reviews', '~> 4.0'        # Product reviews

# Custom gems we'll create
gem 'spree_hangmeas_talent_shows'    # Multi-show voting system (XFactor, CGT, Voice)
gem 'spree_hangmeas_loyalty'         # Advanced loyalty program with cross-show benefits
gem 'spree_hangmeas_wallet'          # Enhanced wallet features
gem 'spree_hangmeas_events'          # Event-specific features
gem 'spree_hangmeas_mobile'          # Mobile API enhancements
```

#### **Database Extensions Needed**
```ruby
# Create these migrations

# 1. Enhanced User Profile
class AddHangmeasUserFields < ActiveRecord::Migration[7.0]
  def change
    add_column :spree_users, :phone_number, :string, index: true
    add_column :spree_users, :khmer_name, :string
    add_column :spree_users, :date_of_birth, :date
    add_column :spree_users, :province, :string
    add_column :spree_users, :loyalty_tier, :string, default: 'bronze'
    add_column :spree_users, :loyalty_points, :integer, default: 0
    add_column :spree_users, :total_spent, :decimal, precision: 10, scale: 2, default: 0
    add_column :spree_users, :referral_code, :string, index: true
    add_column :spree_users, :referred_by_id, :integer
    add_column :spree_users, :language_preference, :string, default: 'en'
    add_column :spree_users, :push_token, :string
    add_column :spree_users, :kyc_status, :string, default: 'pending'
    add_column :spree_users, :verification_level, :string, default: 'basic'
  end
end

# 2. Multi-Talent Show System
class CreateTalentShowSystem < ActiveRecord::Migration[7.0]
  def change
    # Universal talent show structure
    create_table :talent_shows do |t|
      t.string :show_type, null: false # 'xfactor', 'cgt', 'voice'
      t.string :name, null: false
      t.text :description
      t.string :logo_url
      t.text :theme_config # JSON for show-specific UI themes
      t.text :show_config  # JSON for show-specific settings
      t.boolean :active, default: true
      t.timestamps
      
      t.index :show_type
    end
    
    create_table :talent_show_seasons do |t|
      t.references :show, foreign_key: { to_table: :talent_shows }
      t.string :name, null: false
      t.text :description
      t.integer :season_number
      t.date :start_date
      t.date :end_date
      t.string :status, default: 'upcoming'
      t.text :season_config # JSON for season-specific settings
      t.timestamps
    end
    
    create_table :talent_show_episodes do |t|
      t.references :season, foreign_key: { to_table: :talent_show_seasons }
      t.string :title
      t.integer :episode_number
      t.string :episode_type # 'audition', 'battle', 'live_show', 'finale'
      t.datetime :air_date
      t.datetime :voting_start
      t.datetime :voting_end
      t.string :status, default: 'upcoming'
      t.text :episode_config # JSON for episode-specific settings
      t.timestamps
    end
    
    create_table :talent_show_contestants do |t|
      t.references :season, foreign_key: { to_table: :talent_show_seasons }
      t.string :name, null: false
      t.string :khmer_name
      t.text :bio_en
      t.text :bio_km
      t.string :hometown
      t.integer :age
      t.string :photo_url
      t.string :audition_video_url
      t.string :status, default: 'active'
      t.text :contestant_data # JSON for show-specific data
      t.timestamps
    end
    
    # XFactor-specific data
    create_table :xfactor_contestant_data do |t|
      t.references :contestant, foreign_key: { to_table: :talent_show_contestants }
      t.string :category # 'boys', 'girls', 'over_25', 'groups'
      t.string :mentor_judge
      t.integer :six_chair_position
      t.timestamps
    end
    
    # CGT-specific data
    create_table :cgt_contestant_data do |t|
      t.references :contestant, foreign_key: { to_table: :talent_show_contestants }
      t.string :act_type # 'singing', 'dancing', 'magic', etc.
      t.text :act_description
      t.text :props_required
      t.string :golden_buzzer_judge
      t.datetime :golden_buzzer_timestamp
      t.timestamps
    end
    
    # Voice-specific data
    create_table :voice_contestant_data do |t|
      t.references :contestant, foreign_key: { to_table: :talent_show_contestants }
      t.string :coach
      t.text :chair_turned_by # JSON array of coach names
      t.string :battle_opponent
      t.string :stolen_by
      t.string :current_round
      t.timestamps
    end
    
    # Universal voting system
    create_table :talent_show_votes do |t|
      t.references :user, foreign_key: { to_table: :spree_users }
      t.references :contestant, foreign_key: { to_table: :talent_show_contestants }
      t.references :episode, foreign_key: { to_table: :talent_show_episodes }
      t.string :show_type # 'xfactor', 'cgt', 'voice'
      t.string :vote_type # 'free', 'paid', 'sms'
      t.integer :vote_count, default: 1
      t.decimal :multiplier, precision: 3, scale: 2, default: 1.0
      t.string :ip_address
      t.text :metadata # JSON field for device info, vote details, etc.
      t.timestamps
      
      t.index [:show_type, :episode_id]
      t.index [:contestant_id, :episode_id]
    end
    
    # Universal vote packages
    create_table :vote_packages do |t|
      t.string :show_type # null means universal package
      t.string :name
      t.integer :vote_count
      t.integer :bonus_votes, default: 0
      t.decimal :price, precision: 10, scale: 2
      t.string :currency, default: 'USD'
      t.text :features # JSON for package features
      t.boolean :active, default: true
      t.timestamps
      
      t.index :show_type
    end
  end
end

# 3. Enhanced Store Credit for Wallet
class EnhanceStoreCredits < ActiveRecord::Migration[7.0]
  def change
    add_column :spree_store_credits, :transaction_type, :string # 'topup', 'transfer', 'refund', 'purchase'
    add_column :spree_store_credits, :source_type, :string      # 'aba', 'wing', 'card', 'p2p'
    add_column :spree_store_credits, :source_id, :string        # External transaction ID
    add_column :spree_store_credits, :recipient_id, :integer    # For P2P transfers
    add_column :spree_store_credits, :sender_id, :integer       # For P2P transfers
    add_column :spree_store_credits, :reference_number, :string, index: true
    add_column :spree_store_credits, :metadata, :text           # JSON for additional data
    
    add_index :spree_store_credits, :transaction_type
    add_index :spree_store_credits, :source_type
  end
end

# 4. Subscription System
class CreateSubscriptions < ActiveRecord::Migration[7.0]
  def change
    create_table :subscription_plans do |t|
      t.string :name, null: false
      t.text :description
      t.decimal :price, precision: 10, scale: 2
      t.string :currency, default: 'USD'
      t.string :interval # 'month', 'year'
      t.integer :interval_count, default: 1
      t.text :features # JSON
      t.boolean :active, default: true
      t.timestamps
    end
    
    create_table :user_subscriptions do |t|
      t.references :user, foreign_key: { to_table: :spree_users }
      t.references :plan, foreign_key: { to_table: :subscription_plans }
      t.string :status # 'active', 'cancelled', 'expired', 'paused'
      t.datetime :current_period_start
      t.datetime :current_period_end
      t.datetime :cancelled_at
      t.datetime :trial_end
      t.text :metadata
      t.timestamps
    end
  end
end

# 5. Events System Enhancement
class EnhanceEventsSystem < ActiveRecord::Migration[7.0]
  def change
    # Assuming you have events as products, add these fields
    add_column :spree_products, :event_type, :string # 'concert', 'xfactor', 'show'
    add_column :spree_products, :event_date, :datetime
    add_column :spree_products, :venue_name, :string
    add_column :spree_products, :venue_address, :text
    add_column :spree_products, :venue_capacity, :integer
    add_column :spree_products, :sale_start_date, :datetime
    add_column :spree_products, :sale_end_date, :datetime
    add_column :spree_products, :artist_id, :integer
    add_column :spree_products, :seating_chart, :text # JSON
    add_column :spree_products, :is_digital_event, :boolean, default: false
    
    # Seat management
    create_table :event_seats do |t|
      t.references :product, foreign_key: { to_table: :spree_products }
      t.string :section
      t.string :row
      t.string :seat_number
      t.string :seat_type # 'vip', 'standard', 'premium'
      t.decimal :price, precision: 10, scale: 2
      t.string :status, default: 'available' # 'available', 'reserved', 'sold'
      t.references :line_item, foreign_key: { to_table: :spree_line_items }, null: true
      t.timestamps
    end
  end
end
```

---

## 🎫 Multi-Show Talent Show Integration with Spree

### 🗳️ Universal Voting System Implementation

#### **1. Talent Show Models**
```ruby
# app/models/talent_show.rb
class TalentShow < ApplicationRecord
  has_many :talent_show_seasons, dependent: :destroy
  has_many :episodes, through: :talent_show_seasons
  has_many :contestants, through: :talent_show_seasons
  has_many :votes, through: :contestants
  
  validates :show_type, presence: true, inclusion: { in: %w[xfactor cgt voice] }
  validates :name, presence: true
  
  enum status: { upcoming: 0, active: 1, completed: 2 }
  
  scope :current, -> { where(status: :active) }
  scope :by_type, ->(type) { where(show_type: type) }
  
  # Show-specific configuration
  def show_config
    case show_type
    when 'xfactor'
      { categories: %w[Boys Girls Groups Over25s], elimination_style: 'weekly' }
    when 'cgt'
      { categories: %w[Singing Dancing Comedy Magic Other], golden_buzzer: true }
    when 'voice'
      { categories: %w[Team1 Team2 Team3 Team4], blind_auditions: true }
    end
  end
end

# app/models/talent_show_season.rb
class TalentShowSeason < ApplicationRecord
  belongs_to :talent_show
  has_many :episodes, dependent: :destroy
  has_many :contestants, dependent: :destroy
  has_many :votes, through: :contestants
  
  validates :name, presence: true
  
  enum status: { upcoming: 0, active: 1, completed: 2 }
  
  scope :current, -> { where(status: :active).first }
    
  def total_votes
    votes.sum(:vote_count)
  end
end

# app/models/contestant.rb
class Contestant < ApplicationRecord
  belongs_to :talent_show_season
  has_many :votes, class_name: 'TalentShowVote', dependent: :destroy
  has_many :episodes, through: :talent_show_season
  
  # Polymorphic association for show-specific data
  has_one :xfactor_contestant_data, dependent: :destroy
  has_one :cgt_contestant_data, dependent: :destroy  
  has_one :voice_contestant_data, dependent: :destroy
  
  validates :name, presence: true
  validates :category, presence: true
  
  delegate :show_type, to: :talent_show_season, allow_nil: true
  delegate :talent_show, to: :talent_show_season, allow_nil: true
  
  def vote_count_for_episode(episode)
    votes.where(episode: episode).sum(:vote_count)
  end
  
  def total_votes
    votes.sum(:vote_count)
  end
  
  def percentage_for_episode(episode)
    total = episode.total_votes
    return 0 if total.zero?
    
    (vote_count_for_episode(episode).to_f / total * 100).round(1)
  end
  
  # Show-specific data access
  def show_data
    case show_type
    when 'xfactor'
      xfactor_contestant_data
    when 'cgt'
      cgt_contestant_data
    when 'voice'
      voice_contestant_data
    end
  end
end

# app/models/talent_show_vote.rb
class TalentShowVote < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :contestant
  belongs_to :episode
  belongs_to :talent_show_season
  
  validates :vote_type, inclusion: { in: %w[free paid sms] }
  validates :vote_count, presence: true, numericality: { greater_than: 0 }
  validates :show_type, inclusion: { in: %w[xfactor cgt voice] }
  
  before_create :check_voting_limits
  after_create :deduct_vote_balance, :award_loyalty_points
  
  scope :for_show, ->(show_type) { where(show_type: show_type) }
  scope :for_episode, ->(episode) { where(episode: episode) }
  
  private
  
  def check_voting_limits
    if vote_type == 'free'
      # Show-specific free vote limits
      limit = case show_type
              when 'xfactor' then 5
              when 'cgt' then 3  # Lower limit for CGT
              when 'voice' then 10 # Higher for Voice due to more contestants
              else 5
              end
              
      free_votes_used = user.talent_show_votes
                           .where(episode: episode, vote_type: 'free')
                           .sum(:vote_count)
      
      if free_votes_used + vote_count > limit
        errors.add(:base, "Exceeded free vote limit for #{show_type.upcase}")
        throw :abort
      end
    end
  end
  
  def deduct_vote_balance
    if vote_type == 'paid'
      # Universal vote credits work across all shows
      vote_credit = user.store_credits
                       .where(currency: 'VOTES')
                       .first
      
      if vote_credit && vote_credit.amount >= vote_count
        vote_credit.update(amount: vote_credit.amount - vote_count)
        # Track spend by show type
        vote_credit.store_credit_events.create!(
          action: 'used',
          amount: -vote_count,
          user_total_amount: vote_credit.amount,
          metadata: { show_type: show_type, contestant_id: contestant_id }
        )
      else
        raise 'Insufficient vote balance'
      end
    end
  end
  
  def award_loyalty_points
    # Multi-show bonus points
    base_points = vote_count
    multiplier = case show_type
                 when 'xfactor' then 1.0
                 when 'cgt' then 1.2  # Bonus for CGT
                 when 'voice' then 1.1 # Small bonus for Voice
                 else 1.0
                 end
                 
    points = (base_points * multiplier).to_i
    user.increment(:loyalty_points, points)
    user.save
  end
end
```

#### **2. Multi-Show Vote Package Products**
```ruby
# Create universal vote packages as Spree products
class CreateUniversalVotePackageProducts < ActiveRecord::Migration[7.0]
  def up
    # Create a taxonomy for vote packages
    taxonomy = Spree::Taxonomy.create!(name: 'Vote Packages')
    vote_taxon = Spree::Taxon.create!(
      name: 'Universal Show Votes',
      taxonomy: taxonomy,
      parent: taxonomy.root
    )
    
    # Create universal vote package products
    vote_packages = [
      { name: 'Starter Pack', votes: 20, price: 1.99, description: 'Works on all shows' },
      { name: 'Power Pack', votes: 50, price: 4.99, bonus: 10, description: 'Most popular - All shows' },
      { name: 'Super Pack', votes: 200, price: 14.99, bonus: 50, description: 'XFactor + CGT + Voice' },
      { name: 'Ultimate Pack', votes: 1000, price: 49.99, bonus: 200, vip: true, description: 'All shows + VIP perks' }
    ]
    
    vote_packages.each do |package|
      total_votes = package[:votes] + (package[:bonus] || 0)
      product = Spree::Product.create!(
        name: "#{package[:name]} - #{total_votes} Universal Votes",
        description: package[:description],
        price: package[:price],
        available_on: Time.current,
        shipping_category: Spree::ShippingCategory.first,
        tax_category: Spree::TaxCategory.first
      )
      
      # Add custom attributes
      product.product_properties.create!(
        property: Spree::Property.find_or_create_by(name: 'vote_count'),
        value: package[:votes].to_s
      )
      
      product.product_properties.create!(
        property: Spree::Property.find_or_create_by(name: 'package_type'),
        value: 'universal_votes'
      )
      
      product.product_properties.create!(
        property: Spree::Property.find_or_create_by(name: 'compatible_shows'),
        value: 'xfactor,cgt,voice'
      )
      
      if package[:bonus]
        product.product_properties.create!(
          property: Spree::Property.find_or_create_by(name: 'bonus_votes'),
          value: package[:bonus].to_s
        )
      end
      
      if package[:vip]
        product.product_properties.create!(
          property: Spree::Property.find_or_create_by(name: 'vip_benefits'),
          value: 'true'
        )
      end
      
      # Categorize
      product.taxons << vote_taxon
    end
  end
end
```

---

## 💳 Enhanced Wallet System

### 🏦 Extending Spree Store Credits

#### **1. Enhanced Store Credit Model**
```ruby
# app/models/spree/store_credit_decorator.rb
Spree::StoreCredit.class_eval do
  # Add new currencies
  WALLET_CURRENCIES = %w[USD KHR VOTES POINTS].freeze
  
  validates :currency, inclusion: { in: WALLET_CURRENCIES }
  
  belongs_to :recipient, class_name: 'Spree::User', foreign_key: 'recipient_id', optional: true
  belongs_to :sender, class_name: 'Spree::User', foreign_key: 'sender_id', optional: true
  
  scope :wallet_balance, ->(currency) { where(currency: currency) }
  scope :p2p_transfers, -> { where(transaction_type: 'p2p_transfer') }
  scope :topups, -> { where(transaction_type: 'topup') }
  
  def self.currencies_for_user(user)
    where(user: user)
      .group(:currency)
      .sum(:amount)
  end
  
  def self.create_p2p_transfer(sender, recipient, amount, currency = 'USD')
    transaction do
      # Deduct from sender
      sender_credit = create!(
        user: sender,
        sender: sender,
        recipient: recipient,
        amount: -amount,
        currency: currency,
        transaction_type: 'p2p_transfer',
        reference_number: generate_reference_number,
        reason: "Transfer to #{recipient.display_name}"
      )
      
      # Credit to recipient
      recipient_credit = create!(
        user: recipient,
        sender: sender,
        recipient: recipient,
        amount: amount,
        currency: currency,
        transaction_type: 'p2p_transfer',
        reference_number: sender_credit.reference_number,
        reason: "Transfer from #{sender.display_name}"
      )
      
      [sender_credit, recipient_credit]
    end
  end
  
  private
  
  def self.generate_reference_number
    "HM#{Time.current.to_i}#{rand(1000..9999)}"
  end
end
```

#### **2. Wallet Service**
```ruby
# app/services/hangmeas/wallet_service.rb
module Hangmeas
  class WalletService
    def initialize(user)
      @user = user
    end
    
    def balance(currency = 'USD')
      @user.store_credits
           .where(currency: currency)
           .sum(:amount)
    end
    
    def all_balances
      @user.store_credits
           .group(:currency)
           .sum(:amount)
    end
    
    def topup(amount, currency, payment_method, source_id = nil)
      Spree::StoreCredit.create!(
        user: @user,
        amount: amount,
        currency: currency,
        transaction_type: 'topup',
        source_type: payment_method,
        source_id: source_id,
        reference_number: Spree::StoreCredit.generate_reference_number,
        reason: "Wallet topup via #{payment_method}"
      )
    end
    
    def transfer_to(recipient, amount, currency = 'USD')
      return false if balance(currency) < amount
      
      Spree::StoreCredit.create_p2p_transfer(@user, recipient, amount, currency)
    end
    
    def purchase_vote_package(vote_count, bonus_votes = 0)
      total_votes = vote_count + bonus_votes
      
      Spree::StoreCredit.create!(
        user: @user,
        amount: total_votes,
        currency: 'VOTES',
        transaction_type: 'purchase',
        reason: "Vote package: #{vote_count} votes + #{bonus_votes} bonus"
      )
    end
    
    def transaction_history(limit = 50)
      @user.store_credits
           .order(created_at: :desc)
           .limit(limit)
           .includes(:recipient, :sender)
    end
  end
end
```

---

## 📱 Mobile API Extensions

### 🔌 Spree API Enhancements

#### **1. Mobile API Controller**
```ruby
# app/controllers/api/v1/hangmeas/mobile_controller.rb
module Api::V1::Hangmeas
  class MobileController < Spree::Api::BaseController
    before_action :authenticate_spree_user!
    
    def dashboard
      render json: {
        user: user_summary,
        wallet: wallet_summary,
        loyalty: loyalty_summary,
        recent_activity: recent_activity,
        recommendations: recommendations
      }
    end
    
    private
    
    def user_summary
      {
        id: current_spree_user.id,
        name: current_spree_user.display_name,
        khmer_name: current_spree_user.khmer_name,
        loyalty_tier: current_spree_user.loyalty_tier,
        loyalty_points: current_spree_user.loyalty_points,
        total_spent: current_spree_user.total_spent,
        member_since: current_spree_user.created_at
      }
    end
    
    def wallet_summary
      wallet_service = Hangmeas::WalletService.new(current_spree_user)
      wallet_service.all_balances
    end
    
    def loyalty_summary
      {
        current_tier: current_spree_user.loyalty_tier,
        points: current_spree_user.loyalty_points,
        next_tier_progress: calculate_tier_progress,
        benefits: tier_benefits
      }
    end
  end
end

# app/controllers/api/v1/hangmeas/voting_controller.rb
module Api::V1::Hangmeas
  class VotingController < Spree::Api::BaseController
    before_action :authenticate_spree_user!
    
    def current_episode
      episode = Xfactor::Episode.joins(:season)
                                .where(xfactor_seasons: { status: :active })
                                .where('voting_start <= ? AND voting_end >= ?', 
                                      Time.current, Time.current)
                                .first
      
      if episode
        render json: {
          episode: episode_json(episode),
          contestants: contestants_json(episode),
          user_votes: user_votes_json(episode),
          voting_status: voting_status(episode)
        }
      else
        render json: { error: 'No active voting' }, status: 404
      end
    end
    
    def cast_vote
      vote = Xfactor::Vote.new(vote_params.merge(
        user: current_spree_user,
        ip_address: request.remote_ip
      ))
      
      if vote.save
        render json: { 
          success: true,
          vote_id: vote.id,
          remaining_votes: remaining_vote_balance
        }
      else
        render json: { errors: vote.errors }, status: 422
      end
    end
    
    def results
      episode = Xfactor::Episode.find(params[:episode_id])
      
      render json: {
        episode_id: episode.id,
        total_votes: episode.total_votes,
        results: episode.contestants.map do |contestant|
          {
            contestant_id: contestant.id,
            name: contestant.name,
            vote_count: contestant.vote_count_for_episode(episode),
            percentage: contestant.percentage_for_episode(episode)
          }
        end,
        last_updated: Time.current
      }
    end
    
    private
    
    def vote_params
      params.require(:vote).permit(:contestant_id, :episode_id, :vote_type, :vote_count)
    end
  end
end
```

#### **2. WebSocket for Real-time Updates**
```ruby
# app/channels/voting_channel.rb
class VotingChannel < ApplicationCable::Channel
  def subscribed
    episode = Xfactor::Episode.find(params[:episode_id])
    stream_from "voting_#{episode.id}"
  end
  
  def unsubscribed
    # Cleanup
  end
end

# app/jobs/voting_results_broadcast_job.rb
class VotingResultsBroadcastJob < ApplicationJob
  def perform(episode_id)
    episode = Xfactor::Episode.find(episode_id)
    results = calculate_results(episode)
    
    ActionCable.server.broadcast("voting_#{episode_id}", {
      type: 'results_update',
      data: results,
      timestamp: Time.current
    })
  end
  
  private
  
  def calculate_results(episode)
    # Implementation similar to API results method
  end
end

# Schedule regular broadcasts during voting
# config/schedule.rb (using whenever gem)
every 10.seconds do
  runner "VotingResultsBroadcastJob.perform_later(Xfactor::Episode.current_voting&.id)"
end
```

---

## 🎮 Loyalty & Gamification

### 🏆 Advanced Loyalty System

#### **1. Loyalty Service**
```ruby
# app/services/hangmeas/loyalty_service.rb
module Hangmeas
  class LoyaltyService
    TIER_THRESHOLDS = {
      bronze: { points: 0, spending: 0 },
      silver: { points: 1000, spending: 100 },
      gold: { points: 5000, spending: 500 },
      platinum: { points: 15000, spending: 1500 }
    }.freeze
    
    TIER_BENEFITS = {
      bronze: { multiplier: 1.0, shipping_threshold: 50, discount: 0 },
      silver: { multiplier: 1.2, shipping_threshold: 30, discount: 5 },
      gold: { multiplier: 1.5, shipping_threshold: 0, discount: 10 },
      platinum: { multiplier: 2.0, shipping_threshold: 0, discount: 15 }
    }.freeze
    
    def initialize(user)
      @user = user
    end
    
    def award_points(points, reason = nil)
      multiplier = TIER_BENEFITS[@user.loyalty_tier.to_sym][:multiplier]
      actual_points = (points * multiplier).to_i
      
      @user.increment(:loyalty_points, actual_points)
      
      # Log the activity
      LoyaltyActivity.create!(
        user: @user,
        points: actual_points,
        reason: reason,
        activity_type: 'earn'
      )
      
      # Check for tier upgrade
      check_tier_upgrade
      
      @user.save
      actual_points
    end
    
    def redeem_points(points, reason = nil)
      return false if @user.loyalty_points < points
      
      @user.decrement(:loyalty_points, points)
      
      LoyaltyActivity.create!(
        user: @user,
        points: -points,
        reason: reason,
        activity_type: 'redeem'
      )
      
      @user.save
      true
    end
    
    def calculate_tier
      points = @user.loyalty_points
      spending = @user.total_spent.to_f
      
      TIER_THRESHOLDS.reverse_each do |tier, thresholds|
        if points >= thresholds[:points] && spending >= thresholds[:spending]
          return tier.to_s
        end
      end
      
      'bronze'
    end
    
    private
    
    def check_tier_upgrade
      new_tier = calculate_tier
      old_tier = @user.loyalty_tier
      
      if new_tier != old_tier
        @user.loyalty_tier = new_tier
        
        # Send congratulations notification
        TierUpgradeNotificationJob.perform_later(@user.id, old_tier, new_tier)
        
        # Award tier upgrade bonus
        tier_bonus = tier_upgrade_bonus(new_tier)
        @user.increment(:loyalty_points, tier_bonus) if tier_bonus > 0
      end
    end
    
    def tier_upgrade_bonus(tier)
      case tier
      when 'silver' then 250
      when 'gold' then 1000
      when 'platinum' then 2500
      else 0
      end
    end
  end
end

# app/models/loyalty_activity.rb
class LoyaltyActivity < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  
  validates :points, :activity_type, presence: true
  validates :activity_type, inclusion: { in: %w[earn redeem] }
end
```

#### **2. Points Integration with Orders**
```ruby
# app/models/spree/order_decorator.rb
Spree::Order.class_eval do
  after_transition to: :complete do |order|
    HangmeasOrderCompleteJob.perform_later(order.id)
  end
end

# app/jobs/hangmeas_order_complete_job.rb
class HangmeasOrderCompleteJob < ApplicationJob
  def perform(order_id)
    order = Spree::Order.find(order_id)
    return unless order.user
    
    loyalty_service = Hangmeas::LoyaltyService.new(order.user)
    
    # Award points (1 point per dollar spent)
    points = (order.total * 1).to_i
    loyalty_service.award_points(points, "Order ##{order.number}")
    
    # Update user's total spent
    order.user.increment(:total_spent, order.total)
    order.user.save
    
    # Check for achievements
    check_achievements(order)
  end
  
  private
  
  def check_achievements(order)
    # Implement achievement checking logic
    # e.g., first purchase, big spender, etc.
  end
end
```

---

## 📺 Subscription Management

### 🔄 Subscription Integration

#### **1. Subscription Models**
```ruby
# app/models/subscription_plan.rb
class SubscriptionPlan < ApplicationRecord
  has_many :user_subscriptions, foreign_key: 'plan_id'
  has_many :users, through: :user_subscriptions, source: :user
  
  validates :name, :price, :interval, presence: true
  
  def features_list
    JSON.parse(features || '[]')
  end
  
  def monthly_price
    case interval
    when 'year'
      price / 12
    else
      price
    end
  end
end

# app/models/user_subscription.rb
class UserSubscription < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :plan, class_name: 'SubscriptionPlan'
  
  validates :status, inclusion: { 
    in: %w[trial active cancelled expired paused] 
  }
  
  scope :active, -> { where(status: ['trial', 'active']) }
  
  def active?
    ['trial', 'active'].include?(status) && 
    current_period_end > Time.current
  end
  
  def days_remaining
    return 0 unless active?
    ((current_period_end - Time.current) / 1.day).to_i
  end
end
```

#### **2. Subscription Service**
```ruby
# app/services/hangmeas/subscription_service.rb
module Hangmeas
  class SubscriptionService
    def initialize(user)
      @user = user
    end
    
    def subscribe_to_plan(plan_id, trial_days = 7)
      plan = SubscriptionPlan.find(plan_id)
      
      # Cancel any existing subscription
      cancel_current_subscription if current_subscription
      
      # Create new subscription
      subscription = UserSubscription.create!(
        user: @user,
        plan: plan,
        status: trial_days > 0 ? 'trial' : 'active',
        current_period_start: Time.current,
        current_period_end: calculate_period_end(plan, trial_days),
        trial_end: trial_days > 0 ? trial_days.days.from_now : nil
      )
      
      # Schedule billing
      schedule_billing(subscription) if trial_days == 0
      
      subscription
    end
    
    def current_subscription
      @user.user_subscriptions.active.first
    end
    
    def cancel_subscription(reason = nil)
      subscription = current_subscription
      return false unless subscription
      
      subscription.update!(
        status: 'cancelled',
        cancelled_at: Time.current,
        metadata: { cancellation_reason: reason }
      )
      
      true
    end
    
    def has_feature?(feature_name)
      subscription = current_subscription
      return false unless subscription&.active?
      
      plan_features = subscription.plan.features_list
      plan_features.include?(feature_name.to_s)
    end
    
    private
    
    def calculate_period_end(plan, trial_days)
      start_date = trial_days > 0 ? trial_days.days.from_now : Time.current
      
      case plan.interval
      when 'month'
        start_date + (plan.interval_count || 1).months
      when 'year'
        start_date + (plan.interval_count || 1).years
      else
        start_date + 1.month
      end
    end
    
    def schedule_billing(subscription)
      SubscriptionBillingJob.set(wait_until: subscription.current_period_end)
                           .perform_later(subscription.id)
    end
  end
end
```

---

## 🚀 Implementation Roadmap

### 📅 Development Phases

#### **Phase 1: Foundation (Weeks 1-4)**
```ruby
# Week 1-2: Database setup
rails generate migration AddHangmeasUserFields
rails generate migration CreateXfactorVoting  
rails generate migration EnhanceStoreCredits
rails generate migration CreateSubscriptions
rails generate migration EnhanceEventsSystem
rails db:migrate

# Week 3-4: Core models and services
# - Implement XFactor models
# - Create Wallet service
# - Set up Loyalty service
# - Basic API endpoints
```

#### **Phase 2: XFactor Integration (Weeks 5-8)**
```ruby
# Week 5-6: Voting system
# - Vote casting functionality
# - Real-time results
# - WebSocket integration
# - Vote package purchases

# Week 7-8: Mobile API
# - Authentication endpoints
# - Voting API endpoints
# - Real-time updates
# - Push notifications
```

#### **Phase 3: Enhanced Features (Weeks 9-12)**
```ruby
# Week 9-10: Subscription system
# - Plan management
# - Billing integration
# - Feature access control
# - Subscription API

# Week 11-12: Loyalty & Gamification
# - Points system
# - Tier management
# - Achievement tracking
# - Rewards redemption
```

### 🔧 Configuration Examples

#### **Spree Configuration**
```ruby
# config/initializers/spree.rb
Spree.config do |config|
  # Enable store credits
  config.credit_to_new_allocation = true
  
  # Multi-currency support
  config.currency = 'USD,KHR'
  
  # Custom user registration fields
  config.user_class = 'Spree::User'
  
  # API configuration
  config.api_v2_per_page_limit = 100
end

# Enable CORS for mobile apps
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*' # Configure for production
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

#### **ActionCable Configuration**
```ruby
# config/cable.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: hangmeas_production

# config/routes.rb
Rails.application.routes.draw do
  mount Spree::Core::Engine, at: '/'
  mount ActionCable.server => '/cable'
  
  namespace :api do
    namespace :v1 do
      namespace :hangmeas do
        resources :voting, only: [] do
          collection do
            get :current_episode
            post :cast_vote
            get :results
          end
        end
        
        resources :wallet, only: [] do
          collection do
            get :balance
            post :transfer
            get :history
          end
        end
        
        get :dashboard, to: 'mobile#dashboard'
      end
    end
  end
end
```

This implementation guide provides a solid foundation for extending your existing Spree Commerce setup into a comprehensive super app that includes XFactor voting, enhanced wallet functionality, subscription management, and loyalty programs while leveraging your current store credit system.