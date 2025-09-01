# 📚 Week 3-4: Core Models and Services Implementation

## 🎯 Overview
This phase focuses on implementing the core models and services that form the foundation of the Hangmeas Super App, building on top of scalable technology.

---

## 📋 Development Checklist

### Week 3: Core Models Implementation

#### Day 1-2: XFactor Models
- [ ] Create `app/models/xfactor/season.rb`
- [ ] Create `app/models/xfactor/episode.rb`
- [ ] Create `app/models/xfactor/contestant.rb`
- [ ] Create `app/models/xfactor/vote.rb`
- [ ] Create `app/models/vote_package.rb`
- [ ] Write model tests for all XFactor models
- [ ] Add model validations and associations

#### Day 3-4: Enhanced User & Store Credit Models
- [ ] Create `app/models/spree/user_decorator.rb`
- [ ] Create `app/models/spree/store_credit_decorator.rb`
- [ ] Create `app/models/loyalty_activity.rb`
- [ ] Add new fields to User model
- [ ] Extend Store Credit for multi-currency support
- [ ] Write tests for extended models

#### Day 5: Subscription Models
- [ ] Create `app/models/subscription_plan.rb`
- [ ] Create `app/models/user_subscription.rb`
- [ ] Add subscription associations to User model
- [ ] Write subscription model tests
- [ ] Create seed data for subscription plans

### Week 4: Core Services Implementation

#### Day 1-2: Wallet Service
- [ ] Create `app/services/hangmeas/wallet_service.rb`
- [ ] Implement balance management methods
- [ ] Implement P2P transfer functionality
- [ ] Implement topup functionality
- [ ] Create vote package purchase methods
- [ ] Write comprehensive service tests

#### Day 3-4: Loyalty Service
- [ ] Create `app/services/hangmeas/loyalty_service.rb`
- [ ] Implement points awarding system
- [ ] Implement tier calculation logic
- [ ] Create tier upgrade functionality
- [ ] Implement points redemption
- [ ] Write loyalty service tests

#### Day 5: Subscription Service
- [ ] Create `app/services/hangmeas/subscription_service.rb`
- [ ] Implement subscription creation
- [ ] Add trial period handling
- [ ] Create billing scheduling logic
- [ ] Implement feature access control
- [ ] Write subscription service tests

---

## 🔨 Implementation Details

### 1. XFactor Models

#### Season Model (`app/models/xfactor/season.rb`)
```ruby
module Xfactor
  class Season < ApplicationRecord
    self.table_name = 'xfactor_seasons'
    
    # Associations
    has_many :episodes, dependent: :destroy
    has_many :contestants, dependent: :destroy
    has_many :votes, through: :contestants
    
    # Validations
    validates :name, presence: true, uniqueness: true
    validates :status, inclusion: { in: %w[upcoming active completed] }
    
    # Enums
    enum status: { upcoming: 0, active: 1, completed: 2 }
    
    # Scopes
    scope :current, -> { active.first }
    scope :with_stats, -> { includes(:episodes, :contestants) }
    
    # Instance methods
    def total_votes
      votes.sum(:vote_count)
    end
    
    def voting_active?
      active? && episodes.any?(&:voting_active?)
    end
    
    def contestant_rankings
      contestants.includes(:votes)
                 .sort_by { |c| -c.total_votes }
                 .map.with_index do |contestant, index|
        {
          rank: index + 1,
          contestant: contestant,
          votes: contestant.total_votes,
          percentage: contestant.vote_percentage
        }
      end
    end
  end
end
```

#### Episode Model (`app/models/xfactor/episode.rb`)
```ruby
module Xfactor
  class Episode < ApplicationRecord
    self.table_name = 'xfactor_episodes'
    
    # Associations
    belongs_to :season
    has_many :votes, dependent: :destroy
    has_many :episode_contestants
    has_many :contestants, through: :episode_contestants
    
    # Validations
    validates :title, :episode_number, presence: true
    validates :episode_number, uniqueness: { scope: :season_id }
    validates :status, inclusion: { in: %w[upcoming live completed] }
    
    # Enums
    enum status: { upcoming: 0, live: 1, completed: 2 }
    
    # Scopes
    scope :current_voting, -> { where('voting_start <= ? AND voting_end >= ?', Time.current, Time.current) }
    scope :aired, -> { where('air_date <= ?', Time.current) }
    
    # Callbacks
    after_save :broadcast_voting_status_change, if: :saved_change_to_status?
    
    # Instance methods
    def voting_active?
      voting_start.present? && 
      voting_end.present? && 
      Time.current.between?(voting_start, voting_end)
    end
    
    def total_votes
      votes.sum(:vote_count)
    end
    
    def voting_results
      contestants.includes(:votes)
                 .where(votes: { episode_id: id })
                 .group('xfactor_contestants.id')
                 .select('xfactor_contestants.*, SUM(xfactor_votes.vote_count) as total_votes')
                 .order('total_votes DESC')
    end
    
    private
    
    def broadcast_voting_status_change
      VotingStatusBroadcastJob.perform_later(self)
    end
  end
end
```

#### Contestant Model (`app/models/xfactor/contestant.rb`)
```ruby
module Xfactor
  class Contestant < ApplicationRecord
    self.table_name = 'xfactor_contestants'
    
    # Associations
    belongs_to :season
    has_many :votes, dependent: :destroy
    has_many :episode_contestants
    has_many :episodes, through: :episode_contestants
    has_one_attached :photo
    has_one_attached :performance_video
    
    # Validations
    validates :name, presence: true
    validates :status, inclusion: { in: %w[active eliminated winner] }
    validates :age, numericality: { greater_than: 0, less_than: 100 }, allow_nil: true
    
    # Enums
    enum status: { active: 0, eliminated: 1, winner: 2 }
    
    # Scopes
    scope :active_only, -> { active }
    scope :by_hometown, ->(hometown) { where(hometown: hometown) }
    
    # Instance methods
    def vote_count_for_episode(episode)
      votes.where(episode: episode).sum(:vote_count)
    end
    
    def total_votes
      votes.sum(:vote_count)
    end
    
    def vote_percentage
      total = season.total_votes
      return 0.0 if total.zero?
      
      (total_votes.to_f / total * 100).round(2)
    end
    
    def percentage_for_episode(episode)
      total = episode.total_votes
      return 0.0 if total.zero?
      
      (vote_count_for_episode(episode).to_f / total * 100).round(2)
    end
    
    def profile_data
      {
        id: id,
        name: name,
        khmer_name: khmer_name,
        bio: bio,
        hometown: hometown,
        age: age,
        photo_url: photo.attached? ? Rails.application.routes.url_helpers.url_for(photo) : nil,
        status: status,
        total_votes: total_votes,
        vote_percentage: vote_percentage
      }
    end
  end
end
```

#### Vote Model (`app/models/xfactor/vote.rb`)
```ruby
module Xfactor
  class Vote < ApplicationRecord
    self.table_name = 'xfactor_votes'
    
    # Associations
    belongs_to :user, class_name: 'Spree::User'
    belongs_to :contestant
    belongs_to :episode
    
    # Validations
    validates :vote_type, inclusion: { in: %w[free paid sms bundle] }
    validates :vote_count, presence: true, numericality: { greater_than: 0 }
    validate :voting_period_active
    validate :contestant_in_episode
    
    # Callbacks
    before_create :check_voting_limits
    after_create :process_vote
    after_create :broadcast_vote_update
    
    # Constants
    FREE_VOTE_LIMIT = 5
    
    # Scopes
    scope :by_type, ->(type) { where(vote_type: type) }
    scope :recent, -> { order(created_at: :desc) }
    scope :for_episode, ->(episode) { where(episode: episode) }
    
    # Class methods
    def self.user_votes_for_episode(user, episode)
      where(user: user, episode: episode)
    end
    
    # Instance methods
    def free_vote?
      vote_type == 'free'
    end
    
    def paid_vote?
      vote_type == 'paid'
    end
    
    private
    
    def voting_period_active
      unless episode&.voting_active?
        errors.add(:base, 'Voting is not currently active for this episode')
      end
    end
    
    def contestant_in_episode
      unless episode&.contestants&.include?(contestant)
        errors.add(:contestant, 'is not participating in this episode')
      end
    end
    
    def check_voting_limits
      if free_vote?
        free_votes_used = self.class.user_votes_for_episode(user, episode)
                                   .by_type('free')
                                   .sum(:vote_count)
        
        if free_votes_used + vote_count > FREE_VOTE_LIMIT
          errors.add(:base, "Exceeded free vote limit. You have #{FREE_VOTE_LIMIT - free_votes_used} free votes remaining.")
          throw :abort
        end
      end
    end
    
    def process_vote
      case vote_type
      when 'paid'
        deduct_vote_balance
      when 'sms'
        log_sms_vote
      end
      
      award_loyalty_points
      update_user_stats
    end
    
    def deduct_vote_balance
      wallet_service = Hangmeas::WalletService.new(user)
      vote_balance = wallet_service.balance('VOTES')
      
      if vote_balance >= vote_count
        Spree::StoreCredit.create!(
          user: user,
          amount: -vote_count,
          currency: 'VOTES',
          transaction_type: 'vote_cast',
          reason: "Vote for #{contestant.name} - Episode #{episode.episode_number}"
        )
      else
        errors.add(:base, 'Insufficient vote balance')
        raise ActiveRecord::Rollback
      end
    end
    
    def award_loyalty_points
      loyalty_service = Hangmeas::LoyaltyService.new(user)
      points = vote_count * 2 # 2 points per vote
      loyalty_service.award_points(points, "XFactor voting - #{vote_count} votes")
    end
    
    def update_user_stats
      user.increment(:total_votes_cast, vote_count)
      user.save
    end
    
    def log_sms_vote
      # Log SMS vote for billing reconciliation
      SmsVoteLog.create!(
        user: user,
        vote: self,
        phone_number: user.phone_number,
        carrier: detect_carrier(user.phone_number),
        status: 'pending'
      )
    end
    
    def broadcast_vote_update
      VotingResultsBroadcastJob.perform_later(episode.id)
    end
    
    def detect_carrier(phone_number)
      # Implement carrier detection logic for Cambodia
      # Smart: 010, 015, 016, 069, 070, 081, 086, 087, 093, 096
      # Cellcard: 011, 012, 014, 017, 061, 076, 077, 078, 085, 089, 092, 095, 099
      # Metfone: 088, 097, 071, 031, 060, 066, 067, 068, 090
      case phone_number[0..2]
      when '010', '015', '016', '069', '070', '081', '086', '087', '093', '096'
        'smart'
      when '011', '012', '014', '017', '061', '076', '077', '078', '085', '089', '092', '095', '099'
        'cellcard'
      when '088', '097', '071', '031', '060', '066', '067', '068', '090'
        'metfone'
      else
        'unknown'
      end
    end
  end
end
```

### 2. Enhanced User Model

#### User Decorator (`app/models/spree/user_decorator.rb`)
```ruby
module Spree
  module UserDecorator
    def self.prepended(base)
      base.has_many :votes, class_name: 'Xfactor::Vote', dependent: :destroy
      base.has_many :loyalty_activities, dependent: :destroy
      base.has_many :user_subscriptions, dependent: :destroy
      base.has_one :active_subscription, -> { active }, class_name: 'UserSubscription'
      base.has_many :sent_transfers, class_name: 'Spree::StoreCredit', foreign_key: 'sender_id'
      base.has_many :received_transfers, class_name: 'Spree::StoreCredit', foreign_key: 'recipient_id'
      
      base.validates :phone_number, presence: true, uniqueness: true, if: :phone_number_changed?
      base.validates :referral_code, uniqueness: true, allow_nil: true
      base.validates :loyalty_tier, inclusion: { in: %w[bronze silver gold platinum] }
      
      base.before_create :generate_referral_code
      base.after_create :apply_referral_bonus
      
      base.scope :by_tier, ->(tier) { where(loyalty_tier: tier) }
      base.scope :top_spenders, -> { order(total_spent: :desc) }
      base.scope :verified, -> { where(kyc_status: 'verified') }
    end
    
    def display_name
      khmer_name.presence || "#{first_name} #{last_name}".strip.presence || email.split('@').first
    end
    
    def wallet
      @wallet ||= Hangmeas::WalletService.new(self)
    end
    
    def loyalty
      @loyalty ||= Hangmeas::LoyaltyService.new(self)
    end
    
    def subscription
      @subscription ||= Hangmeas::SubscriptionService.new(self)
    end
    
    def can_vote_free?(episode)
      free_votes = votes.where(episode: episode, vote_type: 'free').sum(:vote_count)
      free_votes < Xfactor::Vote::FREE_VOTE_LIMIT
    end
    
    def remaining_free_votes(episode)
      free_votes = votes.where(episode: episode, vote_type: 'free').sum(:vote_count)
      [Xfactor::Vote::FREE_VOTE_LIMIT - free_votes, 0].max
    end
    
    def tier_benefits
      Hangmeas::LoyaltyService::TIER_BENEFITS[loyalty_tier.to_sym]
    end
    
    def next_tier_progress
      current_tier_index = %w[bronze silver gold platinum].index(loyalty_tier)
      return 100 if current_tier_index >= 3 # Already platinum
      
      next_tier = %w[bronze silver gold platinum][current_tier_index + 1]
      next_threshold = Hangmeas::LoyaltyService::TIER_THRESHOLDS[next_tier.to_sym]
      
      points_progress = (loyalty_points.to_f / next_threshold[:points] * 100).round
      spending_progress = (total_spent.to_f / next_threshold[:spending] * 100).round
      
      [points_progress, spending_progress].min
    end
    
    def profile_completion_percentage
      required_fields = %w[first_name last_name phone_number date_of_birth province]
      completed_fields = required_fields.count { |field| send(field).present? }
      
      (completed_fields.to_f / required_fields.size * 100).round
    end
    
    def verified?
      kyc_status == 'verified'
    end
    
    def can_transfer?(amount, currency = 'USD')
      verified? && wallet.balance(currency) >= amount
    end
    
    private
    
    def generate_referral_code
      self.referral_code = loop do
        code = "HM#{SecureRandom.alphanumeric(6).upcase}"
        break code unless self.class.exists?(referral_code: code)
      end
    end
    
    def apply_referral_bonus
      return unless referred_by_id.present?
      
      referrer = self.class.find_by(id: referred_by_id)
      return unless referrer
      
      # Award bonus to referrer
      referrer.loyalty.award_points(500, "Referral bonus - #{display_name}")
      
      # Award bonus to new user
      loyalty.award_points(200, "Welcome bonus - Referred by #{referrer.display_name}")
      
      # Send notifications
      ReferralBonusNotificationJob.perform_later(referrer.id, id)
    end
  end
end

Spree::User.prepend(Spree::UserDecorator)
```

### 3. Enhanced Store Credit Model

#### Store Credit Decorator (`app/models/spree/store_credit_decorator.rb`)
```ruby
module Spree
  module StoreCreditDecorator
    def self.prepended(base)
      base.const_set(:WALLET_CURRENCIES, %w[USD KHR VOTES POINTS].freeze)
      
      base.validates :currency, inclusion: { in: base::WALLET_CURRENCIES }
      base.validates :reference_number, uniqueness: true, if: :reference_number?
      
      base.belongs_to :recipient, class_name: 'Spree::User', foreign_key: 'recipient_id', optional: true
      base.belongs_to :sender, class_name: 'Spree::User', foreign_key: 'sender_id', optional: true
      
      base.scope :wallet_balance, ->(currency) { where(currency: currency) }
      base.scope :p2p_transfers, -> { where(transaction_type: 'p2p_transfer') }
      base.scope :topups, -> { where(transaction_type: 'topup') }
      base.scope :recent, -> { order(created_at: :desc) }
      base.scope :by_type, ->(type) { where(transaction_type: type) }
      
      base.before_create :set_reference_number
      base.after_create :send_transaction_notification
    end
    
    def self.currencies_for_user(user)
      where(user: user)
        .group(:currency)
        .sum(:amount)
        .reject { |_, amount| amount.zero? }
    end
    
    def self.create_p2p_transfer(sender, recipient, amount, currency = 'USD', notes = nil)
      raise ArgumentError, 'Invalid amount' if amount <= 0
      raise ArgumentError, 'Cannot transfer to self' if sender == recipient
      
      transaction do
        # Validate sender balance
        sender_balance = where(user: sender, currency: currency).sum(:amount)
        raise InsufficientFundsError, 'Insufficient balance' if sender_balance < amount
        
        reference_number = generate_reference_number
        
        # Deduct from sender
        sender_credit = create!(
          user: sender,
          sender: sender,
          recipient: recipient,
          amount: -amount,
          currency: currency,
          transaction_type: 'p2p_transfer',
          reference_number: reference_number,
          reason: "Transfer to #{recipient.display_name}",
          metadata: { notes: notes }.to_json
        )
        
        # Credit to recipient
        recipient_credit = create!(
          user: recipient,
          sender: sender,
          recipient: recipient,
          amount: amount,
          currency: currency,
          transaction_type: 'p2p_transfer',
          reference_number: reference_number,
          reason: "Transfer from #{sender.display_name}",
          metadata: { notes: notes }.to_json
        )
        
        # Award loyalty points for transfer
        sender.loyalty.award_points(5, 'P2P transfer sent')
        
        [sender_credit, recipient_credit]
      end
    end
    
    def self.create_topup(user, amount, currency, payment_method, external_ref = nil)
      create!(
        user: user,
        amount: amount,
        currency: currency,
        transaction_type: 'topup',
        source_type: payment_method,
        source_id: external_ref,
        reference_number: generate_reference_number,
        reason: "Wallet topup via #{payment_method.titleize}"
      )
    end
    
    def self.generate_reference_number
      loop do
        ref = "HM#{Time.current.strftime('%Y%m%d')}#{SecureRandom.random_number(10000).to_s.rjust(4, '0')}"
        break ref unless exists?(reference_number: ref)
      end
    end
    
    def display_amount
      Spree::Money.new(amount.abs, currency: currency).to_s
    end
    
    def transaction_description
      case transaction_type
      when 'p2p_transfer'
        amount > 0 ? "Received from #{sender&.display_name}" : "Sent to #{recipient&.display_name}"
      when 'topup'
        "Topup via #{source_type&.titleize}"
      when 'purchase'
        reason
      when 'vote_cast'
        reason
      else
        reason || transaction_type.humanize
      end
    end
    
    def metadata_hash
      JSON.parse(metadata || '{}')
    rescue JSON::ParserError
      {}
    end
    
    private
    
    def set_reference_number
      self.reference_number ||= self.class.generate_reference_number
    end
    
    def send_transaction_notification
      case transaction_type
      when 'p2p_transfer'
        if amount > 0
          # Notify recipient
          TransactionNotificationJob.perform_later(user.id, id, 'received')
        else
          # Notify sender
          TransactionNotificationJob.perform_later(user.id, id, 'sent')
        end
      when 'topup'
        TransactionNotificationJob.perform_later(user.id, id, 'topup')
      end
    end
  end
  
  class InsufficientFundsError < StandardError; end
end

Spree::StoreCredit.prepend(Spree::StoreCreditDecorator)
```

### 4. Wallet Service Implementation

#### Wallet Service (`app/services/hangmeas/wallet_service.rb`)
```ruby
module Hangmeas
  class WalletService
    attr_reader :user
    
    def initialize(user)
      @user = user
    end
    
    def balance(currency = 'USD')
      user.store_credits
          .where(currency: currency)
          .sum(:amount)
    end
    
    def all_balances
      user.store_credits
          .group(:currency)
          .sum(:amount)
          .reject { |_, amount| amount.zero? }
    end
    
    def formatted_balances
      all_balances.transform_values do |amount, currency|
        Spree::Money.new(amount, currency: currency).to_s
      end
    end
    
    def topup(amount, currency, payment_method, external_ref = nil, metadata = {})
      raise ArgumentError, 'Invalid amount' if amount <= 0
      raise ArgumentError, 'Invalid currency' unless Spree::StoreCredit::WALLET_CURRENCIES.include?(currency)
      
      credit = Spree::StoreCredit.create_topup(user, amount, currency, payment_method, external_ref)
      
      # Award loyalty points for topup
      points = calculate_topup_points(amount, currency)
      user.loyalty.award_points(points, "Wallet topup - #{amount} #{currency}")
      
      # Process any topup bonuses
      process_topup_bonus(amount, currency, payment_method)
      
      credit
    end
    
    def transfer_to(recipient, amount, currency = 'USD', notes = nil)
      raise ArgumentError, 'Recipient required' unless recipient.is_a?(Spree::User)
      raise ArgumentError, 'Invalid amount' if amount <= 0
      raise Spree::InsufficientFundsError, 'Insufficient balance' if balance(currency) < amount
      
      # Check transfer limits
      check_transfer_limits(amount, currency)
      
      # Perform transfer
      transfers = Spree::StoreCredit.create_p2p_transfer(user, recipient, amount, currency, notes)
      
      # Send push notifications
      send_transfer_notifications(recipient, amount, currency, notes)
      
      transfers
    end
    
    def purchase_vote_package(vote_count, bonus_votes = 0, price = nil)
      total_votes = vote_count + bonus_votes
      
      credit = Spree::StoreCredit.create!(
        user: user,
        amount: total_votes,
        currency: 'VOTES',
        transaction_type: 'purchase',
        reason: "Vote package: #{vote_count} votes + #{bonus_votes} bonus",
        metadata: { 
          vote_count: vote_count, 
          bonus_votes: bonus_votes,
          price: price 
        }.to_json
      )
      
      # Award loyalty points
      user.loyalty.award_points(vote_count * 2, "Vote package purchase")
      
      credit
    end
    
    def convert_currency(from_amount, from_currency, to_currency)
      return from_amount if from_currency == to_currency
      
      # Implement currency conversion logic
      # For now, using fixed rates
      rates = {
        'USD_KHR' => 4100,
        'KHR_USD' => 1.0 / 4100
      }
      
      key = "#{from_currency}_#{to_currency}"
      rate = rates[key] || 1.0
      
      (from_amount * rate).round(2)
    end
    
    def transaction_history(options = {})
      scope = user.store_credits.includes(:recipient, :sender)
      
      scope = scope.where(currency: options[:currency]) if options[:currency]
      scope = scope.where(transaction_type: options[:type]) if options[:type]
      scope = scope.where('created_at >= ?', options[:from_date]) if options[:from_date]
      scope = scope.where('created_at <= ?', options[:to_date]) if options[:to_date]
      
      scope.recent.limit(options[:limit] || 50)
    end
    
    def monthly_summary(month = Date.current)
      start_date = month.beginning_of_month
      end_date = month.end_of_month
      
      transactions = user.store_credits
                        .where(created_at: start_date..end_date)
      
      {
        total_in: transactions.where('amount > 0').group(:currency).sum(:amount),
        total_out: transactions.where('amount < 0').group(:currency).sum(:amount).transform_values(&:abs),
        transaction_count: transactions.count,
        by_type: transactions.group(:transaction_type).count
      }
    end
    
    def can_transfer?(amount, currency = 'USD')
      user.verified? && balance(currency) >= amount && within_daily_limit?(amount, currency)
    end
    
    private
    
    def calculate_topup_points(amount, currency)
      # Convert to USD for points calculation
      usd_amount = convert_currency(amount, currency, 'USD')
      
      # 10 points per dollar topped up
      (usd_amount * 10).to_i
    end
    
    def process_topup_bonus(amount, currency, payment_method)
      # Check for active topup promotions
      promotion = TopupPromotion.active.applicable_for(amount, currency, payment_method).first
      return unless promotion
      
      bonus_amount = promotion.calculate_bonus(amount)
      if bonus_amount > 0
        Spree::StoreCredit.create!(
          user: user,
          amount: bonus_amount,
          currency: currency,
          transaction_type: 'bonus',
          reason: "Topup bonus - #{promotion.name}"
        )
      end
    end
    
    def check_transfer_limits(amount, currency)
      # Daily transfer limit check
      unless within_daily_limit?(amount, currency)
        raise StandardError, 'Daily transfer limit exceeded'
      end
      
      # Single transaction limit
      usd_amount = convert_currency(amount, currency, 'USD')
      max_single_transfer = user.verified? ? 5000 : 500
      
      if usd_amount > max_single_transfer
        raise StandardError, "Single transfer limit is $#{max_single_transfer}"
      end
    end
    
    def within_daily_limit?(amount, currency)
      today_transfers = user.store_credits
                           .where(transaction_type: 'p2p_transfer')
                           .where('amount < 0')
                           .where(currency: currency)
                           .where('created_at >= ?', Time.current.beginning_of_day)
                           .sum(:amount).abs
      
      daily_limit = user.verified? ? 10000 : 1000
      usd_amount = convert_currency(amount + today_transfers, currency, 'USD')
      
      usd_amount <= daily_limit
    end
    
    def send_transfer_notifications(recipient, amount, currency, notes)
      # Send push notification to recipient
      PushNotificationJob.perform_later(
        recipient.id,
        'Money Received!',
        "#{user.display_name} sent you #{Spree::Money.new(amount, currency: currency)}",
        { type: 'transfer_received', amount: amount, currency: currency }
      )
      
      # Send confirmation to sender
      PushNotificationJob.perform_later(
        user.id,
        'Transfer Successful',
        "You sent #{Spree::Money.new(amount, currency: currency)} to #{recipient.display_name}",
        { type: 'transfer_sent', amount: amount, currency: currency }
      )
    end
  end
end
```

### 5. Loyalty Service Implementation

#### Loyalty Service (`app/services/hangmeas/loyalty_service.rb`)
```ruby
module Hangmeas
  class LoyaltyService
    TIER_THRESHOLDS = {
      bronze: { points: 0, spending: 0 },
      silver: { points: 1000, spending: 100 },
      gold: { points: 5000, spending: 500 },
      platinum: { points: 15000, spending: 1500 }
    }.freeze
    
    TIER_BENEFITS = {
      bronze: { 
        multiplier: 1.0, 
        shipping_threshold: 50, 
        discount: 0,
        free_votes: 5,
        birthday_bonus: 100
      },
      silver: { 
        multiplier: 1.2, 
        shipping_threshold: 30, 
        discount: 5,
        free_votes: 10,
        birthday_bonus: 200,
        early_access: true
      },
      gold: { 
        multiplier: 1.5, 
        shipping_threshold: 0, 
        discount: 10,
        free_votes: 20,
        birthday_bonus: 500,
        early_access: true,
        vip_support: true
      },
      platinum: { 
        multiplier: 2.0, 
        shipping_threshold: 0, 
        discount: 15,
        free_votes: 50,
        birthday_bonus: 1000,
        early_access: true,
        vip_support: true,
        exclusive_events: true
      }
    }.freeze
    
    POINT_RULES = {
      purchase: 1,          # 1 point per dollar spent
      vote: 2,              # 2 points per vote
      review: 50,           # 50 points per product review
      referral: 500,        # 500 points per successful referral
      social_share: 10,     # 10 points per social share
      profile_complete: 100 # 100 points for completing profile
    }.freeze
    
    attr_reader :user
    
    def initialize(user)
      @user = user
    end
    
    def award_points(points, reason = nil, metadata = {})
      raise ArgumentError, 'Points must be positive' unless points > 0
      
      # Apply tier multiplier
      multiplier = current_tier_benefits[:multiplier]
      actual_points = (points * multiplier).to_i
      
      # Update user points
      user.increment(:loyalty_points, actual_points)
      
      # Log the activity
      activity = LoyaltyActivity.create!(
        user: user,
        points: actual_points,
        reason: reason,
        activity_type: 'earn',
        metadata: metadata.to_json
      )
      
      # Check for tier upgrade
      check_tier_upgrade
      
      # Check for achievements
      check_achievements(activity)
      
      # Save user changes
      user.save
      
      actual_points
    end
    
    def redeem_points(points, reason = nil, metadata = {})
      raise ArgumentError, 'Points must be positive' unless points > 0
      raise InsufficientPointsError, 'Insufficient points' if user.loyalty_points < points
      
      user.decrement(:loyalty_points, points)
      
      LoyaltyActivity.create!(
        user: user,
        points: -points,
        reason: reason,
        activity_type: 'redeem',
        metadata: metadata.to_json
      )
      
      user.save
      true
    end
    
    def exchange_points_for_credit(points, currency = 'USD')
      # Exchange rate: 100 points = $1
      exchange_rate = 100
      raise ArgumentError, 'Points must be divisible by 100' unless points % exchange_rate == 0
      
      amount = points / exchange_rate.to_f
      
      ActiveRecord::Base.transaction do
        redeem_points(points, "Exchange for #{currency} credit")
        
        Spree::StoreCredit.create!(
          user: user,
          amount: amount,
          currency: currency,
          transaction_type: 'points_exchange',
          reason: "Exchanged #{points} points"
        )
      end
      
      amount
    end
    
    def current_tier
      user.loyalty_tier.to_sym
    end
    
    def current_tier_benefits
      TIER_BENEFITS[current_tier]
    end
    
    def calculate_tier
      points = user.loyalty_points
      spending = user.total_spent.to_f
      
      TIER_THRESHOLDS.reverse_each do |tier, thresholds|
        if points >= thresholds[:points] && spending >= thresholds[:spending]
          return tier.to_s
        end
      end
      
      'bronze'
    end
    
    def next_tier_requirements
      current_index = TIER_THRESHOLDS.keys.index(current_tier)
      return nil if current_index >= TIER_THRESHOLDS.keys.length - 1
      
      next_tier = TIER_THRESHOLDS.keys[current_index + 1]
      thresholds = TIER_THRESHOLDS[next_tier]
      
      {
        tier: next_tier,
        points_needed: [thresholds[:points] - user.loyalty_points, 0].max,
        spending_needed: [thresholds[:spending] - user.total_spent.to_f, 0].max
      }
    end
    
    def tier_progress
      next_req = next_tier_requirements
      return 100 unless next_req
      
      points_progress = user.loyalty_points.to_f / TIER_THRESHOLDS[next_req[:tier]][:points] * 100
      spending_progress = user.total_spent.to_f / TIER_THRESHOLDS[next_req[:tier]][:spending] * 100
      
      [points_progress, spending_progress].min.round
    end
    
    def apply_birthday_bonus
      return false unless user.date_of_birth&.month == Date.current.month &&
                         user.date_of_birth&.day == Date.current.day
      
      # Check if already awarded this year
      last_birthday_bonus = LoyaltyActivity.where(
        user: user,
        reason: 'Birthday bonus',
        created_at: Date.current.beginning_of_year..Date.current.end_of_year
      ).first
      
      return false if last_birthday_bonus
      
      bonus_points = current_tier_benefits[:birthday_bonus]
      award_points(bonus_points, 'Birthday bonus', { birthday: true })
      
      true
    end
    
    def activity_history(limit = 50)
      user.loyalty_activities
          .order(created_at: :desc)
          .limit(limit)
    end
    
    def points_expiring_soon(days = 30)
      # Implement points expiration logic if needed
      # For now, points don't expire
      0
    end
    
    private
    
    def check_tier_upgrade
      new_tier = calculate_tier
      old_tier = user.loyalty_tier
      
      if new_tier != old_tier && tier_rank(new_tier) > tier_rank(old_tier)
        user.loyalty_tier = new_tier
        
        # Send notification
        TierUpgradeNotificationJob.perform_later(user.id, old_tier, new_tier)
        
        # Award tier upgrade bonus
        bonus = tier_upgrade_bonus(new_tier)
        if bonus > 0
          user.increment(:loyalty_points, bonus)
          
          LoyaltyActivity.create!(
            user: user,
            points: bonus,
            reason: "Tier upgrade to #{new_tier.capitalize}",
            activity_type: 'earn',
            metadata: { tier_upgrade: true, from: old_tier, to: new_tier }.to_json
          )
        end
      end
    end
    
    def tier_rank(tier)
      TIER_THRESHOLDS.keys.index(tier.to_sym) || 0
    end
    
    def tier_upgrade_bonus(tier)
      case tier
      when 'silver' then 250
      when 'gold' then 1000
      when 'platinum' then 2500
      else 0
      end
    end
    
    def check_achievements(activity)
      # Check various achievements
      check_first_purchase_achievement
      check_voting_achievements
      check_spending_achievements
      check_activity_achievements
    end
    
    def check_first_purchase_achievement
      if user.orders.complete.count == 1
        award_achievement('first_purchase', 100, 'First Purchase')
      end
    end
    
    def check_voting_achievements
      total_votes = user.votes.sum(:vote_count)
      
      case total_votes
      when 100
        award_achievement('voter_100', 50, 'Cast 100 votes')
      when 1000
        award_achievement('voter_1000', 200, 'Cast 1,000 votes')
      when 10000
        award_achievement('super_voter', 1000, 'Cast 10,000 votes')
      end
    end
    
    def check_spending_achievements
      case user.total_spent
      when 100..199
        award_achievement('spender_100', 100, 'Spent $100')
      when 500..999
        award_achievement('spender_500', 500, 'Spent $500')
      when 1000..Float::INFINITY
        award_achievement('big_spender', 1000, 'Spent $1,000')
      end
    end
    
    def check_activity_achievements
      consecutive_days = calculate_consecutive_active_days
      
      case consecutive_days
      when 7
        award_achievement('week_streak', 50, '7-day activity streak')
      when 30
        award_achievement('month_streak', 200, '30-day activity streak')
      when 365
        award_achievement('year_streak', 2000, '365-day activity streak')
      end
    end
    
    def award_achievement(code, points, name)
      # Check if already awarded
      existing = user.achievements.find_by(code: code)
      return if existing
      
      # Create achievement
      achievement = user.achievements.create!(
        code: code,
        name: name,
        points_awarded: points,
        achieved_at: Time.current
      )
      
      # Award points
      award_points(points, "Achievement: #{name}", { achievement_id: achievement.id })
      
      # Send notification
      AchievementNotificationJob.perform_later(user.id, achievement.id)
    end
    
    def calculate_consecutive_active_days
      # Simplified calculation - would need proper implementation
      last_activity = user.loyalty_activities.order(created_at: :desc).first
      return 0 unless last_activity
      
      (Date.current - last_activity.created_at.to_date).to_i
    end
  end
  
  class InsufficientPointsError < StandardError; end
end
```

### 6. Subscription Service Implementation

#### Subscription Service (`app/services/hangmeas/subscription_service.rb`)
```ruby
module Hangmeas
  class SubscriptionService
    attr_reader :user
    
    def initialize(user)
      @user = user
    end
    
    def current_subscription
      @current_subscription ||= user.user_subscriptions.active.includes(:plan).first
    end
    
    def active?
      current_subscription&.active?
    end
    
    def subscribe_to_plan(plan_id, options = {})
      plan = SubscriptionPlan.find(plan_id)
      
      ActiveRecord::Base.transaction do
        # Cancel any existing subscription
        cancel_current_subscription if current_subscription
        
        # Determine trial eligibility
        trial_days = determine_trial_days(plan, options[:trial_days])
        
        # Create new subscription
        subscription = user.user_subscriptions.create!(
          plan: plan,
          status: trial_days > 0 ? 'trial' : 'active',
          current_period_start: Time.current,
          current_period_end: calculate_period_end(plan, trial_days),
          trial_end: trial_days > 0 ? trial_days.days.from_now : nil,
          metadata: {
            source: options[:source] || 'web',
            promo_code: options[:promo_code],
            payment_method: options[:payment_method]
          }.to_json
        )
        
        # Process payment if not trial
        if trial_days == 0
          process_subscription_payment(subscription, options[:payment_method])
        end
        
        # Schedule next billing
        schedule_next_billing(subscription)
        
        # Award loyalty points
        points = plan.price * 10 # 10 points per dollar
        user.loyalty.award_points(points, "Subscription to #{plan.name}")
        
        # Send welcome email
        SubscriptionMailer.welcome(subscription).deliver_later
        
        subscription
      end
    end
    
    def cancel_subscription(reason = nil, immediate = false)
      return false unless current_subscription
      
      ActiveRecord::Base.transaction do
        if immediate
          current_subscription.update!(
            status: 'cancelled',
            cancelled_at: Time.current
          )
        else
          # Cancel at end of billing period
          current_subscription.update!(
            status: 'active',
            cancelled_at: Time.current,
            metadata: current_subscription.metadata_hash.merge(
              cancel_at_period_end: true,
              cancellation_reason: reason
            ).to_json
          )
        end
        
        # Cancel scheduled billing
        cancel_scheduled_billing(current_subscription)
        
        # Send cancellation email
        SubscriptionMailer.cancellation(current_subscription).deliver_later
        
        true
      end
    end
    
    def pause_subscription(resume_date = nil)
      return false unless current_subscription&.active?
      
      current_subscription.update!(
        status: 'paused',
        metadata: current_subscription.metadata_hash.merge(
          paused_at: Time.current,
          resume_date: resume_date
        ).to_json
      )
      
      # Cancel scheduled billing
      cancel_scheduled_billing(current_subscription)
      
      # Schedule resume if date provided
      if resume_date
        SubscriptionResumeJob.set(wait_until: resume_date)
                             .perform_later(current_subscription.id)
      end
      
      true
    end
    
    def resume_subscription
      return false unless current_subscription&.paused?
      
      # Calculate new period end
      paused_days = (Time.current - current_subscription.metadata_hash['paused_at'].to_time).to_i / 1.day
      new_period_end = current_subscription.current_period_end + paused_days.days
      
      current_subscription.update!(
        status: 'active',
        current_period_end: new_period_end
      )
      
      # Reschedule billing
      schedule_next_billing(current_subscription)
      
      true
    end
    
    def upgrade_plan(new_plan_id)
      return false unless current_subscription
      
      new_plan = SubscriptionPlan.find(new_plan_id)
      old_plan = current_subscription.plan
      
      # Calculate prorated amount
      prorated_amount = calculate_proration(old_plan, new_plan, current_subscription)
      
      ActiveRecord::Base.transaction do
        # Update subscription
        current_subscription.update!(
          plan: new_plan,
          metadata: current_subscription.metadata_hash.merge(
            upgraded_from: old_plan.id,
            upgraded_at: Time.current,
            proration_amount: prorated_amount
          ).to_json
        )
        
        # Process payment for difference
        if prorated_amount > 0
          process_upgrade_payment(current_subscription, prorated_amount)
        elsif prorated_amount < 0
          # Issue credit
          issue_credit(-prorated_amount, "Plan downgrade credit")
        end
        
        # Send upgrade email
        SubscriptionMailer.plan_change(current_subscription, old_plan).deliver_later
        
        true
      end
    end
    
    def has_feature?(feature_name)
      return false unless active?
      
      plan_features = current_subscription.plan.features_list
      plan_features.include?(feature_name.to_s)
    end
    
    def usage_for_feature(feature_name)
      return nil unless has_feature?(feature_name)
      
      # Implement usage tracking
      case feature_name
      when 'api_calls'
        user.api_calls.where(created_at: current_billing_period).count
      when 'storage'
        user.calculate_storage_usage
      when 'team_members'
        user.team_members.active.count
      else
        0
      end
    end
    
    def billing_history(limit = 12)
      user.subscription_invoices
          .includes(:subscription)
          .order(created_at: :desc)
          .limit(limit)
    end
    
    def upcoming_invoice
      return nil unless current_subscription&.active?
      
      {
        amount: current_subscription.plan.price,
        currency: current_subscription.plan.currency,
        date: current_subscription.current_period_end,
        items: [
          {
            description: current_subscription.plan.name,
            amount: current_subscription.plan.price,
            period: "#{current_subscription.current_period_start.to_date} - #{current_subscription.current_period_end.to_date}"
          }
        ]
      }
    end
    
    private
    
    def determine_trial_days(plan, requested_days)
      # Check if user has used trial before
      previous_trials = user.user_subscriptions
                           .where(plan: plan)
                           .where.not(trial_end: nil)
                           .count
      
      return 0 if previous_trials > 0
      
      requested_days || plan.trial_days || 0
    end
    
    def calculate_period_end(plan, trial_days)
      start_date = trial_days > 0 ? trial_days.days.from_now : Time.current
      
      case plan.interval
      when 'month'
        start_date + (plan.interval_count || 1).months
      when 'year'
        start_date + (plan.interval_count || 1).years
      when 'week'
        start_date + (plan.interval_count || 1).weeks
      else
        start_date + 1.month
      end
    end
    
    def process_subscription_payment(subscription, payment_method)
      # Create invoice
      invoice = SubscriptionInvoice.create!(
        user: user,
        subscription: subscription,
        amount: subscription.plan.price,
        currency: subscription.plan.currency,
        status: 'pending'
      )
      
      # Process payment based on method
      case payment_method
      when 'wallet'
        process_wallet_payment(invoice)
      when 'card'
        process_card_payment(invoice)
      else
        raise "Unsupported payment method: #{payment_method}"
      end
    end
    
    def process_wallet_payment(invoice)
      wallet = user.wallet
      
      if wallet.balance(invoice.currency) >= invoice.amount
        wallet.deduct(invoice.amount, invoice.currency, "Subscription payment - Invoice ##{invoice.number}")
        invoice.update!(status: 'paid', paid_at: Time.current)
      else
        invoice.update!(status: 'failed', failure_reason: 'Insufficient wallet balance')
        raise InsufficientFundsError, 'Insufficient wallet balance'
      end
    end
    
    def process_card_payment(invoice)
      # Implement card payment processing
      # This would integrate with your payment gateway
    end
    
    def calculate_proration(old_plan, new_plan, subscription)
      # Calculate remaining days in current period
      days_remaining = (subscription.current_period_end - Time.current).to_i / 1.day
      total_days = (subscription.current_period_end - subscription.current_period_start).to_i / 1.day
      
      # Calculate unused portion of old plan
      old_plan_credit = (old_plan.price * days_remaining / total_days.to_f).round(2)
      
      # Calculate cost of new plan for remaining period
      new_plan_cost = (new_plan.price * days_remaining / total_days.to_f).round(2)
      
      # Return difference (positive means user owes, negative means credit)
      new_plan_cost - old_plan_credit
    end
    
    def schedule_next_billing(subscription)
      SubscriptionBillingJob.set(wait_until: subscription.current_period_end)
                           .perform_later(subscription.id)
    end
    
    def cancel_scheduled_billing(subscription)
      # Cancel any scheduled billing jobs
      # This would need to be implemented based on your job queue system
    end
    
    def issue_credit(amount, reason)
      Spree::StoreCredit.create!(
        user: user,
        amount: amount,
        currency: current_subscription.plan.currency,
        transaction_type: 'credit',
        reason: reason
      )
    end
    
    def current_billing_period
      return Date.current.all_month unless current_subscription
      
      current_subscription.current_period_start..current_subscription.current_period_end
    end
  end
  
  class InsufficientFundsError < StandardError; end
end
```

---

## 🧪 Testing Strategy

### Model Tests
```ruby
# spec/models/xfactor/vote_spec.rb
require 'rails_helper'

RSpec.describe Xfactor::Vote, type: :model do
  describe 'validations' do
    it { should validate_presence_of(:vote_count) }
    it { should validate_inclusion_of(:vote_type).in_array(%w[free paid sms bundle]) }
  end
  
  describe 'associations' do
    it { should belong_to(:user) }
    it { should belong_to(:contestant) }
    it { should belong_to(:episode) }
  end
  
  describe 'free vote limits' do
    let(:user) { create(:user) }
    let(:episode) { create(:xfactor_episode, :with_active_voting) }
    let(:contestant) { create(:xfactor_contestant) }
    
    it 'enforces free vote limit' do
      # Create 5 free votes
      5.times do
        create(:xfactor_vote, user: user, episode: episode, contestant: contestant, vote_type: 'free', vote_count: 1)
      end
      
      # 6th vote should fail
      vote = build(:xfactor_vote, user: user, episode: episode, contestant: contestant, vote_type: 'free', vote_count: 1)
      expect(vote).not_to be_valid
      expect(vote.errors[:base]).to include(/Exceeded free vote limit/)
    end
  end
end
```

### Service Tests
```ruby
# spec/services/hangmeas/wallet_service_spec.rb
require 'rails_helper'

RSpec.describe Hangmeas::WalletService do
  let(:user) { create(:user) }
  let(:service) { described_class.new(user) }
  
  describe '#topup' do
    it 'creates a store credit' do
      expect {
        service.topup(100, 'USD', 'aba_pay', 'ABA123')
      }.to change { user.store_credits.count }.by(1)
    end
    
    it 'awards loyalty points' do
      expect(user.loyalty).to receive(:award_points).with(1000, anything)
      service.topup(100, 'USD', 'aba_pay')
    end
  end
  
  describe '#transfer_to' do
    let(:recipient) { create(:user) }
    
    before do
      service.topup(200, 'USD', 'aba_pay')
    end
    
    it 'transfers funds between users' do
      expect {
        service.transfer_to(recipient, 50, 'USD')
      }.to change { recipient.wallet.balance('USD') }.by(50)
        .and change { service.balance('USD') }.by(-50)
    end
    
    it 'fails with insufficient funds' do
      expect {
        service.transfer_to(recipient, 300, 'USD')
      }.to raise_error(Spree::InsufficientFundsError)
    end
  end
end
```

---

## 📝 Development Notes

### Key Considerations

1. **Database Indexes**: Ensure proper indexes are added for:
   - `spree_users.phone_number`
   - `spree_users.referral_code`
   - `spree_store_credits.reference_number`
   - `xfactor_votes` composite index on `[user_id, episode_id, vote_type]`

2. **Background Jobs**: Set up ActiveJob with Sidekiq for:
   - Vote result broadcasting
   - Subscription billing
   - Notification sending
   - Loyalty point calculations

3. **Caching Strategy**:
   - Cache vote counts with 10-second expiry
   - Cache user wallet balances with invalidation on change
   - Cache loyalty tier calculations

4. **Security Considerations**:
   - Implement rate limiting for voting
   - Add fraud detection for P2P transfers
   - Secure API endpoints with proper authentication

5. **Performance Optimizations**:
   - Use counter caches for vote counts
   - Implement database views for complex queries
   - Use Redis for real-time features

### Next Steps

After completing Week 3-4 implementation:
1. Run comprehensive test suite
2. Perform load testing for voting system
3. Set up monitoring for background jobs
4. Document API endpoints
5. Prepare for Week 5-6: XFactor Integration phase