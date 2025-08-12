# 📺 Week 9-10: Subscription System Implementation

## 🎯 Overview
This phase focuses on implementing a comprehensive subscription management system with plan management, billing integration, feature access control, and subscription APIs for the Hangmeas Super App.

---

> **Streaming Service Integration**: For streaming-specific subscription tiers and infrastructure, see [STREAMING_INFRASTRUCTURE.md](../../../STREAMING_INFRASTRUCTURE.md#business-models)

## 📋 Development Checklist

### Week 9: Core Subscription Features

#### Day 1-2: Plan Management System
- [ ] Create subscription plan management interface
- [ ] Implement plan versioning and pricing
- [ ] Set up plan features and limitations
- [ ] Create plan comparison and recommendations
- [ ] Implement plan activation/deactivation

#### Day 3-4: Subscription Lifecycle Management
- [ ] Implement subscription creation flow
- [ ] Set up trial period management
- [ ] Create subscription state transitions
- [ ] Implement upgrade/downgrade logic
- [ ] Add cancellation and pause functionality

#### Day 5: Billing Foundation
- [ ] Set up billing cycle calculations
- [ ] Create invoice generation system
- [ ] Implement proration calculations
- [ ] Add payment method integration
- [ ] Create billing retry logic

### Week 10: Advanced Features & Integration

#### Day 1-2: Feature Access Control
- [ ] Implement subscription-based feature gates
- [ ] Create usage tracking system
- [ ] Add quota management
- [ ] Implement feature analytics
- [ ] Set up access control middleware

#### Day 3-4: Payment Integration & Automation
- [ ] Integrate multiple payment gateways
- [ ] Implement automated billing
- [ ] Create dunning management
- [ ] Add payment failure handling
- [ ] Implement refund processing

#### Day 5: Subscription APIs & Analytics
- [ ] Create comprehensive subscription APIs
- [ ] Implement subscription analytics
- [ ] Add churn prediction
- [ ] Create subscription reports
- [ ] Set up monitoring and alerts

---

## 🔨 Implementation Details

### 1. Enhanced Subscription Models

```ruby
# app/models/subscription_plan.rb
class SubscriptionPlan < ApplicationRecord
  has_many :user_subscriptions, foreign_key: 'plan_id', dependent: :restrict_with_error
  has_many :subscription_invoices, through: :user_subscriptions
  has_many :plan_features, dependent: :destroy
  has_many :plan_versions, dependent: :destroy
  
  validates :name, :slug, :price, :interval, presence: true
  validates :slug, uniqueness: true
  validates :price, numericality: { greater_than: 0 }
  validates :interval, inclusion: { in: %w[day week month year] }
  validates :interval_count, numericality: { greater_than: 0 }
  
  scope :active, -> { where(active: true) }
  scope :by_price, -> { order(:price) }
  scope :recommended, -> { where(recommended: true) }
  
  before_save :ensure_single_recommended, if: :recommended?
  after_create :create_default_features
  
  def monthly_equivalent_price
    case interval
    when 'day'
      price * 30 / interval_count
    when 'week'
      price * 4 / interval_count
    when 'month'
      price / interval_count
    when 'year'
      price / (12 * interval_count)
    end
  end
  
  def features_hash
    @features_hash ||= plan_features.index_by(&:feature_name)
  end
  
  def has_feature?(feature_name)
    features_hash.key?(feature_name.to_s)
  end
  
  def feature_limit(feature_name)
    features_hash[feature_name.to_s]&.limit_value || 0
  end
  
  def feature_enabled?(feature_name)
    feature = features_hash[feature_name.to_s]
    feature&.enabled? || false
  end
  
  def compare_with(other_plan)
    return {} unless other_plan.is_a?(SubscriptionPlan)
    
    all_features = (features_hash.keys + other_plan.features_hash.keys).uniq
    
    all_features.map do |feature_name|
      current_feature = features_hash[feature_name]
      other_feature = other_plan.features_hash[feature_name]
      
      {
        feature: feature_name,
        current: {
          enabled: current_feature&.enabled? || false,
          limit: current_feature&.limit_value || 0
        },
        other: {
          enabled: other_feature&.enabled? || false,
          limit: other_feature&.limit_value || 0
        }
      }
    end
  end
  
  def upgrade_path_to(target_plan)
    return nil if target_plan.monthly_equivalent_price <= monthly_equivalent_price
    
    price_difference = target_plan.monthly_equivalent_price - monthly_equivalent_price
    
    {
      target_plan: target_plan,
      price_increase: price_difference,
      new_features: target_plan.features_hash.keys - features_hash.keys,
      enhanced_features: compare_with(target_plan).select do |comparison|
        comparison[:other][:enabled] && (
          !comparison[:current][:enabled] ||
          comparison[:other][:limit] > comparison[:current][:limit]
        )
      end
    }
  end
  
  private
  
  def ensure_single_recommended
    self.class.where(recommended: true).where.not(id: id).update_all(recommended: false)
  end
  
  def create_default_features
    default_features = {
      'ad_free' => { enabled: true, limit: nil },
      'hd_streaming' => { enabled: price >= 9.99, limit: nil },
      'offline_downloads' => { enabled: price >= 14.99, limit: price >= 14.99 ? nil : 10 },
      'multiple_devices' => { enabled: true, limit: price >= 9.99 ? nil : 2 },
      'exclusive_content' => { enabled: price >= 14.99, limit: nil },
      'live_events' => { enabled: price >= 19.99, limit: nil },
      'early_access' => { enabled: price >= 14.99, limit: nil }
    }
    
    default_features.each do |name, config|
      plan_features.create!(
        feature_name: name,
        enabled: config[:enabled],
        limit_value: config[:limit]
      )
    end
  end
end

# app/models/plan_feature.rb
class PlanFeature < ApplicationRecord
  belongs_to :subscription_plan
  
  validates :feature_name, presence: true, uniqueness: { scope: :subscription_plan_id }
  validates :limit_value, numericality: { greater_than: 0, allow_nil: true }
  
  scope :enabled, -> { where(enabled: true) }
  
  def unlimited?
    enabled? && limit_value.nil?
  end
  
  def limited?
    enabled? && limit_value.present?
  end
  
  def display_value
    if enabled?
      unlimited? ? 'Unlimited' : limit_value.to_s
    else
      'Not included'
    end
  end
end

# app/models/user_subscription.rb
class UserSubscription < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :plan, class_name: 'SubscriptionPlan'
  has_many :subscription_invoices, dependent: :destroy
  has_many :feature_usages, dependent: :destroy
  has_many :subscription_events, dependent: :destroy
  
  validates :status, inclusion: { in: %w[trial active cancelled expired paused past_due] }
  validate :end_date_after_start_date
  validate :trial_end_before_period_end
  
  scope :active_or_trial, -> { where(status: ['active', 'trial']) }
  scope :billable, -> { where(status: ['active', 'trial', 'past_due']) }
  scope :ending_soon, ->(days = 7) { where(current_period_end: ..days.days.from_now) }
  
  state_machine :status, initial: :trial do
    event :activate do
      transition [:trial, :past_due] => :active
    end
    
    event :cancel do
      transition [:trial, :active, :past_due] => :cancelled
    end
    
    event :expire do
      transition [:trial, :active, :past_due, :cancelled] => :expired
    end
    
    event :pause do
      transition [:active, :trial] => :paused
    end
    
    event :resume do
      transition :paused => :active
    end
    
    event :mark_past_due do
      transition :active => :past_due
    end
    
    after_transition any => any do |subscription, transition|
      subscription.log_event(
        event_type: transition.event.to_s,
        from_status: transition.from,
        to_status: transition.to,
        triggered_at: Time.current
      )
    end
    
    after_transition to: :cancelled do |subscription|
      subscription.update!(cancelled_at: Time.current)
      SubscriptionCancellationJob.perform_later(subscription.id)
    end
    
    after_transition to: :expired do |subscription|
      SubscriptionExpiredJob.perform_later(subscription.id)
    end
  end
  
  def active?
    ['active', 'trial'].include?(status) && current_period_end > Time.current
  end
  
  def in_trial?
    status == 'trial' && trial_end.present? && Time.current <= trial_end
  end
  
  def days_remaining
    return 0 unless active?
    ((current_period_end - Time.current) / 1.day).to_i
  end
  
  def trial_days_remaining
    return 0 unless in_trial? && trial_end
    [(trial_end - Time.current) / 1.day, 0].max.to_i
  end
  
  def usage_for_feature(feature_name)
    today = Date.current
    
    feature_usages.where(
      feature_name: feature_name,
      usage_date: current_period_start.to_date..today
    ).sum(:usage_count)
  end
  
  def can_use_feature?(feature_name, requested_usage = 1)
    return false unless active?
    
    feature = plan.features_hash[feature_name.to_s]
    return false unless feature&.enabled?
    return true if feature.unlimited?
    
    current_usage = usage_for_feature(feature_name)
    (current_usage + requested_usage) <= feature.limit_value
  end
  
  def track_feature_usage(feature_name, usage_count = 1, metadata = {})
    return false unless active?
    
    feature_usage = feature_usages.find_or_initialize_by(
      feature_name: feature_name,
      usage_date: Date.current
    )
    
    feature_usage.usage_count = (feature_usage.usage_count || 0) + usage_count
    feature_usage.metadata = (feature_usage.metadata || {}).merge(metadata)
    feature_usage.save!
    
    # Check if approaching limit
    if plan.feature_limit(feature_name) > 0
      usage_percentage = (feature_usage.usage_count.to_f / plan.feature_limit(feature_name) * 100)
      
      if usage_percentage >= 80 && !feature_usage.warning_sent?
        FeatureUsageWarningJob.perform_later(id, feature_name, usage_percentage)
        feature_usage.update!(warning_sent: true)
      end
    end
    
    feature_usage
  end
  
  def next_billing_amount
    return 0 unless billable?
    
    base_amount = plan.price
    
    # Apply any pending changes
    if metadata['pending_plan_change']
      pending_plan = SubscriptionPlan.find(metadata['pending_plan_change']['plan_id'])
      proration = calculate_proration_for_upgrade(pending_plan)
      base_amount = proration[:amount]
    end
    
    # Apply any credits or discounts
    credits = user.store_credits.where(currency: plan.currency).sum(:amount)
    
    [base_amount - credits, 0].max
  end
  
  def calculate_proration_for_upgrade(new_plan)
    return { amount: new_plan.price, proration: 0 } unless active?
    
    # Calculate unused time in current period
    total_period_seconds = current_period_end - current_period_start
    unused_seconds = current_period_end - Time.current
    unused_percentage = unused_seconds / total_period_seconds
    
    # Calculate credit for unused portion of current plan
    current_plan_credit = plan.price * unused_percentage
    
    # Calculate cost of new plan for remaining period
    new_plan_cost = new_plan.price * unused_percentage
    
    proration_amount = new_plan_cost - current_plan_credit
    
    {
      amount: new_plan.price,
      proration: proration_amount,
      credit: current_plan_credit,
      unused_days: (unused_seconds / 86400).round
    }
  end
  
  def renewal_price
    plan.price
  end
  
  def log_event(event_type, metadata = {})
    subscription_events.create!(
      event_type: event_type,
      metadata: metadata.merge(
        plan_name: plan.name,
        plan_price: plan.price,
        user_id: user.id
      )
    )
  end
  
  private
  
  def end_date_after_start_date
    return unless current_period_start && current_period_end
    
    if current_period_end <= current_period_start
      errors.add(:current_period_end, 'must be after start date')
    end
  end
  
  def trial_end_before_period_end
    return unless trial_end && current_period_end
    
    if trial_end > current_period_end
      errors.add(:trial_end, 'must be before period end')
    end
  end
end

# app/models/subscription_invoice.rb
class SubscriptionInvoice < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :subscription, class_name: 'UserSubscription'
  has_many :invoice_line_items, dependent: :destroy
  has_many :payment_attempts, dependent: :destroy
  
  validates :number, :amount, :currency, :status, presence: true
  validates :number, uniqueness: true
  validates :amount, numericality: { greater_than: 0 }
  validates :status, inclusion: { in: %w[draft pending paid failed refunded void] }
  
  before_validation :generate_invoice_number, on: :create
  after_create :create_line_items
  
  scope :unpaid, -> { where(status: ['pending', 'failed']) }
  scope :overdue, -> { where(status: ['pending', 'failed']).where('due_date < ?', Time.current) }
  
  state_machine :status, initial: :draft do
    event :finalize do
      transition :draft => :pending
    end
    
    event :pay do
      transition [:pending, :failed] => :paid
    end
    
    event :fail_payment do
      transition [:pending, :paid] => :failed
    end
    
    event :refund do
      transition :paid => :refunded
    end
    
    event :void do
      transition [:draft, :pending] => :void
    end
    
    after_transition to: :paid do |invoice|
      invoice.update!(paid_at: Time.current)
      SubscriptionPaymentSuccessJob.perform_later(invoice.id)
    end
    
    after_transition to: :failed do |invoice|
      SubscriptionPaymentFailedJob.perform_later(invoice.id)
    end
  end
  
  def overdue?
    unpaid? && due_date < Time.current
  end
  
  def days_overdue
    return 0 unless overdue?
    ((Time.current - due_date) / 1.day).to_i
  end
  
  def unpaid?
    ['pending', 'failed'].include?(status)
  end
  
  def total_with_tax
    amount + (tax_amount || 0)
  end
  
  def attempt_payment(payment_method = nil)
    payment_method ||= user.default_payment_method
    
    return false unless payment_method
    
    attempt = payment_attempts.create!(
      payment_method_id: payment_method.id,
      amount: total_with_tax,
      currency: currency,
      status: 'pending'
    )
    
    SubscriptionPaymentProcessor.perform_later(attempt.id)
    
    attempt
  end
  
  def retry_payment
    return false if paid? || void?
    return false if payment_attempts.count >= 3
    
    attempt_payment
  end
  
  def calculate_late_fees
    return 0 unless overdue?
    
    base_fee = 5.00
    daily_fee = amount * 0.01 # 1% per day
    
    base_fee + (daily_fee * days_overdue)
  end
  
  def add_late_fees!
    return false unless overdue?
    
    late_fee = calculate_late_fees
    return false if late_fee <= 0
    
    invoice_line_items.create!(
      description: "Late fee (#{days_overdue} days overdue)",
      amount: late_fee,
      line_item_type: 'late_fee'
    )
    
    self.amount += late_fee
    save!
  end
  
  private
  
  def generate_invoice_number
    return if number.present?
    
    date_prefix = Time.current.strftime('%Y%m')
    sequence = self.class.where('number LIKE ?', "#{date_prefix}%").count + 1
    
    self.number = "#{date_prefix}#{sequence.to_s.rjust(6, '0')}"
  end
  
  def create_line_items
    # Main subscription line item
    invoice_line_items.create!(
      description: "#{subscription.plan.name} subscription",
      amount: subscription.plan.price,
      line_item_type: 'subscription',
      period_start: subscription.current_period_start,
      period_end: subscription.current_period_end
    )
    
    # Proration adjustments if any
    if metadata['proration_amount'].present?
      invoice_line_items.create!(
        description: 'Proration adjustment',
        amount: metadata['proration_amount'].to_f,
        line_item_type: 'proration'
      )
    end
    
    # Credits if any
    credits_applied = [amount * -0.1, user.store_credits.sum(:amount)].min
    if credits_applied > 0
      invoice_line_items.create!(
        description: 'Account credit applied',
        amount: -credits_applied,
        line_item_type: 'credit'
      )
      
      # Deduct credits from user account
      user.store_credits.create!(
        amount: -credits_applied,
        currency: currency,
        transaction_type: 'invoice_payment',
        reason: "Applied to invoice #{number}"
      )
    end
    
    # Recalculate total
    self.amount = invoice_line_items.sum(:amount)
    save!
  end
end
```

### 2. Subscription Service Layer

```ruby
# app/services/subscription_service.rb
class SubscriptionService
  def initialize(user)
    @user = user
  end
  
  def subscribe(plan_id, options = {})
    plan = SubscriptionPlan.find(plan_id)
    
    ActiveRecord::Base.transaction do
      # Cancel existing subscription if any
      cancel_current_subscription if current_subscription&.active?
      
      # Determine trial eligibility
      trial_days = calculate_trial_days(plan, options)
      
      # Create subscription
      subscription = create_subscription(plan, trial_days, options)
      
      # Process initial payment if not in trial
      if trial_days == 0
        process_initial_payment(subscription, options[:payment_method])
      else
        schedule_trial_end_reminder(subscription)
      end
      
      # Schedule next billing
      schedule_next_billing(subscription)
      
      # Award loyalty points
      award_subscription_points(subscription)
      
      # Send confirmation
      send_subscription_confirmation(subscription)
      
      subscription
    end
  rescue PaymentError => e
    subscription&.destroy
    raise e
  end
  
  def upgrade(new_plan_id, options = {})
    return nil unless current_subscription&.active?
    
    new_plan = SubscriptionPlan.find(new_plan_id)
    old_plan = current_subscription.plan
    
    # Validate upgrade
    unless new_plan.monthly_equivalent_price > old_plan.monthly_equivalent_price
      raise InvalidUpgradeError, 'New plan must be more expensive than current plan'
    end
    
    ActiveRecord::Base.transaction do
      # Calculate proration
      proration = current_subscription.calculate_proration_for_upgrade(new_plan)
      
      # Process upgrade payment
      if proration[:proration] > 0
        process_upgrade_payment(current_subscription, proration, options[:payment_method])
      end
      
      # Update subscription
      current_subscription.update!(
        plan: new_plan,
        metadata: current_subscription.metadata.merge(
          'upgraded_from' => old_plan.id,
          'upgraded_at' => Time.current,
          'proration_amount' => proration[:proration]
        )
      )
      
      # Log upgrade event
      current_subscription.log_event('plan_upgraded', {
        old_plan_id: old_plan.id,
        new_plan_id: new_plan.id,
        proration_amount: proration[:proration]
      })
      
      # Send upgrade confirmation
      SubscriptionMailer.plan_upgraded(current_subscription, old_plan).deliver_later
      
      # Award bonus loyalty points for upgrade
      bonus_points = (new_plan.price - old_plan.price) * 10
      @user.loyalty.award_points(bonus_points, "Subscription upgrade to #{new_plan.name}")
      
      current_subscription
    end
  end
  
  def downgrade(new_plan_id, effective_date = nil)
    return nil unless current_subscription&.active?
    
    new_plan = SubscriptionPlan.find(new_plan_id)
    old_plan = current_subscription.plan
    
    # Validate downgrade
    unless new_plan.monthly_equivalent_price < old_plan.monthly_equivalent_price
      raise InvalidDowngradeError, 'New plan must be less expensive than current plan'
    end
    
    effective_date ||= current_subscription.current_period_end
    
    if effective_date <= Time.current
      # Immediate downgrade
      perform_immediate_downgrade(new_plan)
    else
      # Schedule downgrade for end of billing period
      schedule_downgrade(new_plan, effective_date)
    end
  end
  
  def cancel(reason = nil, immediate = false)
    return false unless current_subscription&.active?
    
    ActiveRecord::Base.transaction do
      if immediate
        current_subscription.cancel!
        current_subscription.update!(cancelled_at: Time.current)
      else
        # Cancel at end of billing period
        current_subscription.metadata = current_subscription.metadata.merge(
          'cancel_at_period_end' => true,
          'cancellation_reason' => reason,
          'cancellation_requested_at' => Time.current
        )
        current_subscription.save!
      end
      
      # Cancel scheduled billing
      cancel_scheduled_billing(current_subscription)
      
      # Process cancellation
      process_cancellation(current_subscription, reason, immediate)
      
      true
    end
  end
  
  def pause(resume_date = nil)
    return false unless current_subscription&.active?
    
    current_subscription.pause!
    
    current_subscription.update!(
      metadata: current_subscription.metadata.merge(
        'paused_at' => Time.current,
        'pause_resume_date' => resume_date,
        'pause_reason' => 'user_requested'
      )
    )
    
    # Cancel scheduled billing
    cancel_scheduled_billing(current_subscription)
    
    # Schedule auto-resume if date provided
    if resume_date
      SubscriptionResumeJob.set(wait_until: resume_date).perform_later(current_subscription.id)
    end
    
    # Send pause confirmation
    SubscriptionMailer.subscription_paused(current_subscription).deliver_later
    
    true
  end
  
  def resume
    return false unless current_subscription&.paused?
    
    # Calculate extension for paused time
    paused_duration = Time.current - Time.parse(current_subscription.metadata['paused_at'])
    new_period_end = current_subscription.current_period_end + paused_duration
    
    current_subscription.resume!
    current_subscription.update!(
      current_period_end: new_period_end,
      metadata: current_subscription.metadata.merge(
        'resumed_at' => Time.current,
        'paused_duration_seconds' => paused_duration.to_i
      )
    )
    
    # Reschedule billing
    schedule_next_billing(current_subscription)
    
    # Send resume confirmation
    SubscriptionMailer.subscription_resumed(current_subscription).deliver_later
    
    true
  end
  
  def current_subscription
    @current_subscription ||= @user.user_subscriptions.active_or_trial.first
  end
  
  def subscription_history
    @user.user_subscriptions.includes(:plan).order(created_at: :desc)
  end
  
  def usage_summary(feature_name = nil)
    return {} unless current_subscription
    
    if feature_name
      feature_usage_summary(feature_name)
    else
      all_features_usage_summary
    end
  end
  
  def can_use_feature?(feature_name, requested_usage = 1)
    current_subscription&.can_use_feature?(feature_name, requested_usage) || false
  end
  
  def track_usage(feature_name, usage_count = 1, metadata = {})
    return false unless current_subscription
    
    current_subscription.track_feature_usage(feature_name, usage_count, metadata)
  end
  
  def upcoming_invoice
    return nil unless current_subscription&.billable?
    
    {
      amount: current_subscription.next_billing_amount,
      currency: current_subscription.plan.currency,
      due_date: current_subscription.current_period_end,
      description: "#{current_subscription.plan.name} subscription renewal",
      period: {
        start: current_subscription.current_period_end,
        end: calculate_next_period_end(current_subscription)
      }
    }
  end
  
  def billing_history(limit = 12)
    @user.subscription_invoices
         .includes(:subscription => :plan)
         .order(created_at: :desc)
         .limit(limit)
  end
  
  def calculate_savings_vs_monthly
    return 0 unless current_subscription
    
    plan = current_subscription.plan
    return 0 if plan.interval == 'month'
    
    monthly_equivalent = plan.monthly_equivalent_price
    
    # Find closest monthly plan for comparison
    monthly_plan = SubscriptionPlan.active
                                  .where(interval: 'month')
                                  .order('ABS(price - ?)', monthly_equivalent)
                                  .first
    
    return 0 unless monthly_plan
    
    annual_cost = monthly_equivalent * 12
    monthly_cost = monthly_plan.price * 12
    
    monthly_cost - annual_cost
  end
  
  private
  
  def calculate_trial_days(plan, options)
    # Check user's trial history
    has_trialed = @user.user_subscriptions.where.not(trial_end: nil).exists?
    return 0 if has_trialed
    
    # Use plan's default trial or option override
    options[:trial_days] || plan.trial_days || 7
  end
  
  def create_subscription(plan, trial_days, options)
    start_time = Time.current
    trial_end = trial_days > 0 ? start_time + trial_days.days : nil
    period_start = trial_end || start_time
    
    @user.user_subscriptions.create!(
      plan: plan,
      status: trial_days > 0 ? 'trial' : 'active',
      current_period_start: period_start,
      current_period_end: calculate_period_end(plan, period_start),
      trial_start: trial_days > 0 ? start_time : nil,
      trial_end: trial_end,
      metadata: {
        'signup_source' => options[:source] || 'web',
        'promo_code' => options[:promo_code],
        'trial_days_granted' => trial_days
      }
    )
  end
  
  def calculate_period_end(plan, start_time)
    case plan.interval
    when 'day'
      start_time + (plan.interval_count || 1).days
    when 'week'
      start_time + (plan.interval_count || 1).weeks
    when 'month'
      start_time + (plan.interval_count || 1).months
    when 'year'
      start_time + (plan.interval_count || 1).years
    end
  end
  
  def process_initial_payment(subscription, payment_method_id)
    payment_method = @user.user_payment_methods.find(payment_method_id) if payment_method_id
    
    # Create invoice
    invoice = subscription.subscription_invoices.create!(
      user: @user,
      amount: subscription.plan.price,
      currency: subscription.plan.currency,
      due_date: Time.current,
      status: 'pending'
    )
    
    # Attempt payment
    payment_result = PaymentProcessor.charge(
      amount: invoice.total_with_tax,
      currency: invoice.currency,
      customer: @user,
      payment_method: payment_method,
      description: "Subscription: #{subscription.plan.name}"
    )
    
    if payment_result.success?
      invoice.pay!
      subscription.activate!
    else
      invoice.fail_payment!
      raise PaymentError, payment_result.error_message
    end
  end
  
  def process_upgrade_payment(subscription, proration, payment_method_id)
    return if proration[:proration] <= 0
    
    payment_method = @user.user_payment_methods.find(payment_method_id) if payment_method_id
    
    payment_result = PaymentProcessor.charge(
      amount: proration[:proration],
      currency: subscription.plan.currency,
      customer: @user,
      payment_method: payment_method,
      description: "Subscription upgrade proration"
    )
    
    unless payment_result.success?
      raise PaymentError, payment_result.error_message
    end
  end
  
  def perform_immediate_downgrade(new_plan)
    old_plan = current_subscription.plan
    
    # Calculate refund for unused time
    refund_amount = calculate_downgrade_refund(new_plan)
    
    # Update subscription
    current_subscription.update!(
      plan: new_plan,
      metadata: current_subscription.metadata.merge(
        'downgraded_from' => old_plan.id,
        'downgraded_at' => Time.current,
        'refund_amount' => refund_amount
      )
    )
    
    # Process refund if applicable
    if refund_amount > 0
      @user.store_credits.create!(
        amount: refund_amount,
        currency: current_subscription.plan.currency,
        transaction_type: 'refund',
        reason: "Subscription downgrade refund"
      )
    end
    
    # Log downgrade event
    current_subscription.log_event('plan_downgraded', {
      old_plan_id: old_plan.id,
      new_plan_id: new_plan.id,
      refund_amount: refund_amount
    })
    
    current_subscription
  end
  
  def schedule_downgrade(new_plan, effective_date)
    current_subscription.metadata = current_subscription.metadata.merge(
      'scheduled_downgrade' => {
        'new_plan_id' => new_plan.id,
        'effective_date' => effective_date,
        'scheduled_at' => Time.current
      }
    )
    current_subscription.save!
    
    # Schedule downgrade job
    SubscriptionDowngradeJob.set(wait_until: effective_date).perform_later(current_subscription.id)
    
    # Send confirmation
    SubscriptionMailer.downgrade_scheduled(current_subscription, new_plan).deliver_later
  end
  
  def calculate_downgrade_refund(new_plan)
    return 0 unless current_subscription.active?
    
    # Calculate unused time
    total_period = current_subscription.current_period_end - current_subscription.current_period_start
    unused_time = current_subscription.current_period_end - Time.current
    unused_percentage = unused_time / total_period
    
    # Calculate refund
    current_plan_unused = current_subscription.plan.price * unused_percentage
    new_plan_cost = new_plan.price * unused_percentage
    
    [current_plan_unused - new_plan_cost, 0].max
  end
  
  def cancel_current_subscription
    current_subscription.cancel!
    current_subscription.update!(cancelled_at: Time.current)
  end
  
  def schedule_next_billing(subscription)
    return unless subscription.billable?
    
    SubscriptionBillingJob.set(wait_until: subscription.current_period_end)
                         .perform_later(subscription.id)
  end
  
  def cancel_scheduled_billing(subscription)
    # This would depend on your background job system
    # For Sidekiq with sidekiq-cron, you might do:
    # Sidekiq::Cron::Job.find("subscription_#{subscription.id}")&.destroy
  end
  
  def award_subscription_points(subscription)
    points = subscription.plan.price * 10 # 10 points per dollar
    @user.loyalty.award_points(points, "Subscription to #{subscription.plan.name}")
  end
  
  def send_subscription_confirmation(subscription)
    SubscriptionMailer.subscription_created(subscription).deliver_later
    
    # Send push notification
    PushNotificationService.send_to_user(
      @user,
      'Subscription Active! 🎉',
      "Welcome to #{subscription.plan.name}! Enjoy your premium features.",
      {
        type: 'subscription_activated',
        subscription_id: subscription.id
      }
    )
  end
  
  def schedule_trial_end_reminder(subscription)
    reminder_time = subscription.trial_end - 3.days
    
    SubscriptionTrialReminderJob.set(wait_until: reminder_time)
                               .perform_later(subscription.id)
  end
  
  def process_cancellation(subscription, reason, immediate)
    # Log cancellation
    subscription.log_event('subscription_cancelled', {
      reason: reason,
      immediate: immediate,
      cancelled_by: 'user'
    })
    
    # Send cancellation email
    SubscriptionMailer.subscription_cancelled(subscription, reason, immediate).deliver_later
    
    # Award retention points (for future win-back campaigns)
    @user.loyalty.award_points(100, 'Subscription cancellation (retention)')
    
    # Schedule win-back campaign
    if !immediate
      WinBackCampaignJob.set(wait: 7.days).perform_later(@user.id)
    end
  end
  
  def feature_usage_summary(feature_name)
    usage = current_subscription.usage_for_feature(feature_name)
    limit = current_subscription.plan.feature_limit(feature_name)
    
    {
      feature: feature_name,
      usage: usage,
      limit: limit,
      unlimited: limit == 0,
      percentage_used: limit > 0 ? (usage.to_f / limit * 100).round(2) : 0,
      remaining: limit > 0 ? [limit - usage, 0].max : nil
    }
  end
  
  def all_features_usage_summary
    return {} unless current_subscription
    
    current_subscription.plan.features_hash.keys.map do |feature_name|
      feature_usage_summary(feature_name)
    end.index_by { |summary| summary[:feature] }
  end
  
  def calculate_next_period_end(subscription)
    calculate_period_end(subscription.plan, subscription.current_period_end)
  end
  
  # Custom exception classes
  class PaymentError < StandardError; end
  class InvalidUpgradeError < StandardError; end
  class InvalidDowngradeError < StandardError; end
end
```

### 3. Feature Access Control Middleware

```ruby
# app/middleware/subscription_feature_gate.rb
class SubscriptionFeatureGate
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = ActionDispatch::Request.new(env)
    
    # Only check API requests
    if request.path.start_with?('/api/v1/') && requires_subscription_check?(request)
      user = extract_user_from_request(request)
      
      if user && !check_feature_access(user, request)
        return feature_restricted_response(request)
      end
    end
    
    @app.call(env)
  end
  
  private
  
  def requires_subscription_check?(request)
    # Define which endpoints require subscription checks
    protected_paths = [
      '/api/v1/premium/',
      '/api/v1/exclusive/',
      '/api/v1/downloads/',
      '/api/v1/streaming/hd'
    ]
    
    protected_paths.any? { |path| request.path.start_with?(path) }
  end
  
  def extract_user_from_request(request)
    auth_header = request.headers['Authorization']
    return nil unless auth_header
    
    token = auth_header.split(' ').last
    begin
      decoded = JsonWebToken.decode(token)
      Spree::User.find(decoded[:user_id])
    rescue
      nil
    end
  end
  
  def check_feature_access(user, request)
    subscription_service = SubscriptionService.new(user)
    
    feature_name = determine_required_feature(request.path)
    return true unless feature_name
    
    subscription_service.can_use_feature?(feature_name)
  end
  
  def determine_required_feature(path)
    case path
    when %r{/api/v1/premium/}
      'premium_content'
    when %r{/api/v1/exclusive/}
      'exclusive_content'
    when %r{/api/v1/downloads/}
      'offline_downloads'
    when %r{/api/v1/streaming/hd}
      'hd_streaming'
    else
      nil
    end
  end
  
  def feature_restricted_response(request)
    [
      402, # Payment Required
      { 'Content-Type' => 'application/json' },
      [JSON.generate({
        success: false,
        error: 'subscription_required',
        message: 'This feature requires an active subscription',
        upgrade_url: '/api/v1/subscriptions/plans'
      })]
    ]
  end
end

# config/application.rb
config.middleware.insert_before ActionDispatch::ShowExceptions, SubscriptionFeatureGate

# app/controllers/concerns/subscription_required.rb
module SubscriptionRequired
  extend ActiveSupport::Concern
  
  included do
    before_action :check_subscription_required, if: :subscription_required?
  end
  
  private
  
  def check_subscription_required
    return if current_user.nil?
    
    subscription_service = SubscriptionService.new(current_user)
    feature_name = required_subscription_feature
    
    unless subscription_service.can_use_feature?(feature_name)
      render json: {
        success: false,
        error: 'subscription_required',
        message: "This feature requires #{feature_name.humanize}",
        current_plan: current_user.subscription&.plan&.name,
        upgrade_options: available_upgrade_plans
      }, status: :payment_required
      return
    end
    
    # Track feature usage
    subscription_service.track_usage(feature_name, 1, {
      controller: controller_name,
      action: action_name,
      timestamp: Time.current
    })
  end
  
  def subscription_required?
    false # Override in controllers that need subscription checks
  end
  
  def required_subscription_feature
    # Override in specific controllers
    'premium_access'
  end
  
  def available_upgrade_plans
    current_plan = current_user.subscription&.plan
    return SubscriptionPlan.active.as_json(only: [:id, :name, :price]) unless current_plan
    
    SubscriptionPlan.active
                   .where('price > ?', current_plan.price)
                   .as_json(only: [:id, :name, :price])
  end
end
```

### 4. Subscription APIs

```ruby
# app/controllers/api/v1/subscriptions_controller.rb
module Api
  module V1
    class SubscriptionsController < BaseController
      # GET /api/v1/subscriptions/plans
      def plans
        plans = SubscriptionPlan.active.includes(:plan_features).order(:price)
        
        render json: {
          success: true,
          data: {
            plans: plans.map { |plan| format_plan(plan) },
            recommended_plan: plans.find(&:recommended?)&.then { |p| format_plan(p) },
            current_subscription: current_subscription_summary
          }
        }
      end
      
      # GET /api/v1/subscriptions/current
      def current
        subscription_service = SubscriptionService.new(current_user)
        subscription = subscription_service.current_subscription
        
        if subscription
          render json: {
            success: true,
            data: {
              subscription: format_subscription(subscription),
              usage: subscription_service.usage_summary,
              upcoming_invoice: subscription_service.upcoming_invoice,
              savings: subscription_service.calculate_savings_vs_monthly
            }
          }
        else
          render json: {
            success: false,
            message: 'No active subscription'
          }, status: :not_found
        end
      end
      
      # POST /api/v1/subscriptions/subscribe
      def subscribe
        plan_id = params[:plan_id]
        options = {
          payment_method: params[:payment_method_id],
          source: 'mobile_api',
          trial_days: params[:trial_days]&.to_i,
          promo_code: params[:promo_code]
        }
        
        subscription_service = SubscriptionService.new(current_user)
        
        begin
          subscription = subscription_service.subscribe(plan_id, options)
          
          success_response(
            {
              subscription: format_subscription(subscription),
              trial_active: subscription.in_trial?,
              trial_ends_at: subscription.trial_end
            },
            'Subscription created successfully'
          )
        rescue SubscriptionService::PaymentError => e
          error_response(e.message, :payment_required)
        rescue ActiveRecord::RecordNotFound
          error_response('Plan not found', :not_found)
        rescue StandardError => e
          error_response('Subscription creation failed', :unprocessable_entity, [e.message])
        end
      end
      
      # PUT /api/v1/subscriptions/upgrade
      def upgrade
        new_plan_id = params[:plan_id]
        options = {
          payment_method: params[:payment_method_id]
        }
        
        subscription_service = SubscriptionService.new(current_user)
        
        begin
          subscription = subscription_service.upgrade(new_plan_id, options)
          
          success_response(
            format_subscription(subscription),
            'Subscription upgraded successfully'
          )
        rescue SubscriptionService::PaymentError => e
          error_response(e.message, :payment_required)
        rescue SubscriptionService::InvalidUpgradeError => e
          error_response(e.message, :unprocessable_entity)
        end
      end
      
      # PUT /api/v1/subscriptions/downgrade
      def downgrade
        new_plan_id = params[:plan_id]
        immediate = params[:immediate] == 'true'
        effective_date = immediate ? Time.current : nil
        
        subscription_service = SubscriptionService.new(current_user)
        
        begin
          result = subscription_service.downgrade(new_plan_id, effective_date)
          
          success_response(
            {
              subscription: format_subscription(result),
              immediate: immediate,
              effective_date: effective_date
            },
            immediate ? 'Subscription downgraded' : 'Downgrade scheduled'
          )
        rescue SubscriptionService::InvalidDowngradeError => e
          error_response(e.message, :unprocessable_entity)
        end
      end
      
      # DELETE /api/v1/subscriptions/cancel
      def cancel
        reason = params[:reason]
        immediate = params[:immediate] == 'true'
        
        subscription_service = SubscriptionService.new(current_user)
        
        if subscription_service.cancel(reason, immediate)
          success_response(
            {
              cancelled: true,
              immediate: immediate,
              access_until: immediate ? Time.current : subscription_service.current_subscription.current_period_end
            },
            immediate ? 'Subscription cancelled' : 'Subscription will cancel at period end'
          )
        else
          error_response('Cancellation failed')
        end
      end
      
      # PUT /api/v1/subscriptions/pause
      def pause
        resume_date = params[:resume_date]&.to_datetime
        
        subscription_service = SubscriptionService.new(current_user)
        
        if subscription_service.pause(resume_date)
          success_response(
            {
              paused: true,
              resume_date: resume_date,
              auto_resume: resume_date.present?
            },
            'Subscription paused'
          )
        else
          error_response('Pause failed')
        end
      end
      
      # PUT /api/v1/subscriptions/resume
      def resume
        subscription_service = SubscriptionService.new(current_user)
        
        if subscription_service.resume
          success_response(
            format_subscription(subscription_service.current_subscription),
            'Subscription resumed'
          )
        else
          error_response('Resume failed')
        end
      end
      
      # GET /api/v1/subscriptions/usage/:feature
      def feature_usage
        feature_name = params[:feature]
        subscription_service = SubscriptionService.new(current_user)
        
        usage_data = subscription_service.usage_summary(feature_name)
        
        if usage_data.present?
          success_response(usage_data)
        else
          error_response('Feature not found', :not_found)
        end
      end
      
      # GET /api/v1/subscriptions/invoices
      def invoices
        page = params[:page]&.to_i || 1
        per_page = [params[:per_page]&.to_i || 10, 50].min
        
        subscription_service = SubscriptionService.new(current_user)
        invoices = subscription_service.billing_history(per_page * 2)
                                     .page(page).per(per_page)
        
        render json: {
          success: true,
          data: {
            invoices: invoices.map { |invoice| format_invoice(invoice) },
            pagination: pagination_meta(invoices)
          }
        }
      end
      
      # GET /api/v1/subscriptions/compare/:plan_id
      def compare_plan
        target_plan = SubscriptionPlan.find(params[:plan_id])
        subscription_service = SubscriptionService.new(current_user)
        current_subscription = subscription_service.current_subscription
        
        if current_subscription
          comparison = current_subscription.plan.compare_with(target_plan)
          upgrade_path = current_subscription.plan.upgrade_path_to(target_plan)
          
          render json: {
            success: true,
            data: {
              current_plan: format_plan(current_subscription.plan),
              target_plan: format_plan(target_plan),
              feature_comparison: comparison,
              upgrade_path: upgrade_path,
              cost_difference: target_plan.monthly_equivalent_price - current_subscription.plan.monthly_equivalent_price
            }
          }
        else
          render json: {
            success: true,
            data: {
              target_plan: format_plan(target_plan),
              features: target_plan.features_hash.map do |name, feature|
                {
                  name: name,
                  enabled: feature.enabled?,
                  limit: feature.limit_value
                }
              end
            }
          }
        end
      end
      
      private
      
      def format_plan(plan)
        {
          id: plan.id,
          name: plan.name,
          slug: plan.slug,
          description: plan.description,
          price: plan.price.to_f,
          currency: plan.currency,
          interval: plan.interval,
          interval_count: plan.interval_count,
          trial_days: plan.trial_days,
          monthly_equivalent: plan.monthly_equivalent_price.to_f,
          recommended: plan.recommended?,
          badge_text: plan.badge_text,
          features: plan.plan_features.map do |feature|
            {
              name: feature.feature_name,
              enabled: feature.enabled?,
              limit: feature.limit_value,
              display_value: feature.display_value,
              unlimited: feature.unlimited?
            }
          end,
          metadata: plan.metadata
        }
      end
      
      def format_subscription(subscription)
        {
          id: subscription.id,
          status: subscription.status,
          plan: format_plan(subscription.plan),
          current_period_start: subscription.current_period_start,
          current_period_end: subscription.current_period_end,
          trial_start: subscription.trial_start,
          trial_end: subscription.trial_end,
          cancelled_at: subscription.cancelled_at,
          days_remaining: subscription.days_remaining,
          trial_days_remaining: subscription.trial_days_remaining,
          in_trial: subscription.in_trial?,
          cancel_at_period_end: subscription.metadata['cancel_at_period_end'],
          next_billing_amount: subscription.next_billing_amount.to_f,
          created_at: subscription.created_at,
          metadata: subscription.metadata
        }
      end
      
      def format_invoice(invoice)
        {
          id: invoice.id,
          number: invoice.number,
          amount: invoice.amount.to_f,
          currency: invoice.currency,
          status: invoice.status,
          due_date: invoice.due_date,
          paid_at: invoice.paid_at,
          description: invoice.description,
          period_start: invoice.period_start,
          period_end: invoice.period_end,
          line_items: invoice.invoice_line_items.map do |item|
            {
              description: item.description,
              amount: item.amount.to_f,
              type: item.line_item_type
            }
          end,
          overdue: invoice.overdue?,
          days_overdue: invoice.days_overdue
        }
      end
      
      def current_subscription_summary
        subscription_service = SubscriptionService.new(current_user)
        subscription = subscription_service.current_subscription
        
        return nil unless subscription
        
        {
          plan_name: subscription.plan.name,
          status: subscription.status,
          expires_at: subscription.current_period_end,
          in_trial: subscription.in_trial?
        }
      end
    end
  end
end
```

### 5. Background Jobs for Subscription Management

```ruby
# app/jobs/subscription_billing_job.rb
class SubscriptionBillingJob < ApplicationJob
  queue_as :critical
  
  def perform(subscription_id)
    subscription = UserSubscription.find(subscription_id)
    return unless subscription.billable?
    
    # Check if already billed for this period
    existing_invoice = subscription.subscription_invoices
                                 .where(period_start: subscription.current_period_start)
                                 .first
    
    return if existing_invoice&.paid?
    
    ActiveRecord::Base.transaction do
      # Create invoice
      invoice = create_invoice_for_subscription(subscription)
      
      # Attempt payment
      payment_success = attempt_invoice_payment(invoice)
      
      if payment_success
        # Extend subscription period
        extend_subscription_period(subscription)
        
        # Award loyalty points
        award_renewal_points(subscription)
        
        # Send success notifications
        send_payment_success_notifications(subscription, invoice)
      else
        # Handle payment failure
        handle_payment_failure(subscription, invoice)
      end
    end
  rescue StandardError => e
    Rails.logger.error "Subscription billing failed for #{subscription_id}: #{e.message}"
    
    # Alert admin
    AdminNotificationMailer.billing_error(subscription, e.message).deliver_later
  end
  
  private
  
  def create_invoice_for_subscription(subscription)
    # Check for pending plan changes
    if subscription.metadata['scheduled_downgrade']
      downgrade_data = subscription.metadata['scheduled_downgrade']
      new_plan = SubscriptionPlan.find(downgrade_data['new_plan_id'])
      
      # Apply downgrade
      subscription.update!(
        plan: new_plan,
        metadata: subscription.metadata.except('scheduled_downgrade')
      )
    end
    
    invoice = subscription.subscription_invoices.create!(
      user: subscription.user,
      amount: subscription.plan.price,
      currency: subscription.plan.currency,
      due_date: subscription.current_period_end,
      status: 'pending',
      period_start: subscription.current_period_end,
      period_end: calculate_next_period_end(subscription)
    )
    
    invoice.finalize!
    invoice
  end
  
  def attempt_invoice_payment(invoice)
    subscription = invoice.subscription
    user = subscription.user
    
    # Try default payment method
    payment_method = user.default_payment_method
    
    unless payment_method
      invoice.fail_payment!
      return false
    end
    
    payment_result = PaymentProcessor.charge(
      amount: invoice.total_with_tax,
      currency: invoice.currency,
      customer: user,
      payment_method: payment_method,
      description: "Subscription renewal: #{subscription.plan.name}",
      metadata: {
        subscription_id: subscription.id,
        invoice_id: invoice.id
      }
    )
    
    if payment_result.success?
      invoice.update!(
        transaction_id: payment_result.transaction_id,
        payment_method: payment_method.payment_type
      )
      invoice.pay!
      true
    else
      invoice.update!(failure_reason: payment_result.error_message)
      invoice.fail_payment!
      false
    end
  end
  
  def extend_subscription_period(subscription)
    new_period_start = subscription.current_period_end
    new_period_end = calculate_next_period_end(subscription)
    
    subscription.update!(
      current_period_start: new_period_start,
      current_period_end: new_period_end,
      status: 'active'
    )
    
    # Schedule next billing
    SubscriptionBillingJob.set(wait_until: new_period_end)
                         .perform_later(subscription.id)
  end
  
  def calculate_next_period_end(subscription)
    plan = subscription.plan
    start_time = subscription.current_period_end
    
    case plan.interval
    when 'day'
      start_time + (plan.interval_count || 1).days
    when 'week'
      start_time + (plan.interval_count || 1).weeks
    when 'month'
      start_time + (plan.interval_count || 1).months
    when 'year'
      start_time + (plan.interval_count || 1).years
    end
  end
  
  def award_renewal_points(subscription)
    user = subscription.user
    points = subscription.plan.price * 5 # 5 points per dollar for renewals
    
    user.loyalty.award_points(points, "Subscription renewal - #{subscription.plan.name}")
  end
  
  def send_payment_success_notifications(subscription, invoice)
    # Email receipt
    SubscriptionMailer.payment_successful(subscription, invoice).deliver_later
    
    # Push notification
    PushNotificationService.send_to_user(
      subscription.user,
      'Payment Successful',
      "Your #{subscription.plan.name} subscription has been renewed",
      {
        type: 'payment_success',
        subscription_id: subscription.id,
        invoice_id: invoice.id
      }
    )
  end
  
  def handle_payment_failure(subscription, invoice)
    user = subscription.user
    
    # Mark subscription as past due
    subscription.mark_past_due!
    
    # Send payment failure notifications
    SubscriptionMailer.payment_failed(subscription, invoice).deliver_later
    
    # Schedule retry attempts
    schedule_payment_retries(invoice)
    
    # Start dunning process
    DunningProcessJob.perform_later(subscription.id)
  end
  
  def schedule_payment_retries(invoice)
    # Retry after 3 days
    RetryPaymentJob.set(wait: 3.days).perform_later(invoice.id)
    
    # Retry after 7 days
    RetryPaymentJob.set(wait: 7.days).perform_later(invoice.id)
    
    # Final retry after 14 days
    RetryPaymentJob.set(wait: 14.days).perform_later(invoice.id)
  end
end

# app/jobs/subscription_trial_reminder_job.rb
class SubscriptionTrialReminderJob < ApplicationJob
  queue_as :default
  
  def perform(subscription_id)
    subscription = UserSubscription.find(subscription_id)
    return unless subscription.in_trial?
    
    days_remaining = subscription.trial_days_remaining
    
    # Send appropriate reminder based on days remaining
    case days_remaining
    when 7
      send_trial_week_reminder(subscription)
    when 3
      send_trial_three_day_reminder(subscription)
    when 1
      send_trial_final_reminder(subscription)
    end
    
    # Schedule next reminder if needed
    if days_remaining > 1
      next_reminder_days = case days_remaining
                          when 7
                            3
                          when 3
                            1
                          end
      
      if next_reminder_days
        SubscriptionTrialReminderJob.set(wait: next_reminder_days.days)
                                   .perform_later(subscription_id)
      end
    end
  end
  
  private
  
  def send_trial_week_reminder(subscription)
    SubscriptionMailer.trial_ending_soon(subscription, 7).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Trial Ending Soon',
      "Your #{subscription.plan.name} trial ends in 7 days",
      {
        type: 'trial_reminder',
        days_remaining: 7,
        subscription_id: subscription.id
      }
    )
  end
  
  def send_trial_three_day_reminder(subscription)
    SubscriptionMailer.trial_ending_soon(subscription, 3).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Trial Ending in 3 Days',
      "Don't lose access to #{subscription.plan.name} features",
      {
        type: 'trial_reminder',
        days_remaining: 3,
        subscription_id: subscription.id
      }
    )
  end
  
  def send_trial_final_reminder(subscription)
    SubscriptionMailer.trial_ending_soon(subscription, 1).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Trial Ends Tomorrow! ⏰',
      'Subscribe now to keep your premium features',
      {
        type: 'trial_final_reminder',
        days_remaining: 1,
        subscription_id: subscription.id
      }
    )
  end
end

# app/jobs/dunning_process_job.rb
class DunningProcessJob < ApplicationJob
  queue_as :default
  
  DUNNING_STAGES = [
    { days: 1, stage: 'gentle_reminder' },
    { days: 7, stage: 'payment_reminder' },
    { days: 14, stage: 'final_warning' },
    { days: 21, stage: 'suspension_notice' },
    { days: 30, stage: 'cancellation' }
  ].freeze
  
  def perform(subscription_id)
    subscription = UserSubscription.find(subscription_id)
    return unless subscription.past_due?
    
    # Find latest failed invoice
    failed_invoice = subscription.subscription_invoices
                               .where(status: 'failed')
                               .order(created_at: :desc)
                               .first
    
    return unless failed_invoice
    
    days_overdue = failed_invoice.days_overdue
    current_stage = determine_dunning_stage(days_overdue)
    
    return unless current_stage
    
    # Check if we've already sent this stage notification
    last_dunning = subscription.subscription_events
                             .where(event_type: 'dunning_email_sent')
                             .where("metadata->>'stage' = ?", current_stage[:stage])
                             .order(created_at: :desc)
                             .first
    
    return if last_dunning && last_dunning.created_at > 1.day.ago
    
    # Execute dunning stage
    execute_dunning_stage(subscription, failed_invoice, current_stage)
    
    # Schedule next stage if not final
    unless current_stage[:stage] == 'cancellation'
      next_stage = DUNNING_STAGES.find { |stage| stage[:days] > days_overdue }
      if next_stage
        wait_days = next_stage[:days] - days_overdue
        DunningProcessJob.set(wait: wait_days.days).perform_later(subscription_id)
      end
    end
  end
  
  private
  
  def determine_dunning_stage(days_overdue)
    DUNNING_STAGES.reverse.find { |stage| days_overdue >= stage[:days] }
  end
  
  def execute_dunning_stage(subscription, invoice, stage)
    case stage[:stage]
    when 'gentle_reminder'
      send_gentle_reminder(subscription, invoice)
    when 'payment_reminder'
      send_payment_reminder(subscription, invoice)
    when 'final_warning'
      send_final_warning(subscription, invoice)
    when 'suspension_notice'
      send_suspension_notice(subscription, invoice)
    when 'cancellation'
      cancel_subscription(subscription, invoice)
    end
    
    # Log dunning event
    subscription.log_event('dunning_email_sent', {
      stage: stage[:stage],
      days_overdue: invoice.days_overdue,
      invoice_id: invoice.id
    })
  end
  
  def send_gentle_reminder(subscription, invoice)
    SubscriptionMailer.payment_reminder_gentle(subscription, invoice).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Payment Reminder',
      'We were unable to process your payment. Please update your payment method.',
      {
        type: 'payment_reminder',
        subscription_id: subscription.id,
        stage: 'gentle'
      }
    )
  end
  
  def send_payment_reminder(subscription, invoice)
    SubscriptionMailer.payment_reminder_urgent(subscription, invoice).deliver_later
    
    # Add late fees
    invoice.add_late_fees!
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Urgent: Payment Required',
      'Your subscription will be suspended soon. Please update your payment method.',
      {
        type: 'payment_urgent',
        subscription_id: subscription.id,
        stage: 'urgent'
      }
    )
  end
  
  def send_final_warning(subscription, invoice)
    SubscriptionMailer.payment_final_warning(subscription, invoice).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Final Warning: Payment Required',
      'Your subscription will be cancelled in 7 days without payment.',
      {
        type: 'payment_final_warning',
        subscription_id: subscription.id,
        stage: 'final'
      }
    )
  end
  
  def send_suspension_notice(subscription, invoice)
    # Suspend access but don't cancel yet
    subscription.update!(
      metadata: subscription.metadata.merge(
        'access_suspended' => true,
        'suspended_at' => Time.current
      )
    )
    
    SubscriptionMailer.access_suspended(subscription, invoice).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Access Suspended',
      'Your subscription has been suspended due to non-payment.',
      {
        type: 'access_suspended',
        subscription_id: subscription.id
      }
    )
  end
  
  def cancel_subscription(subscription, invoice)
    subscription.cancel!
    
    SubscriptionMailer.subscription_cancelled_nonpayment(subscription, invoice).deliver_later
    
    PushNotificationService.send_to_user(
      subscription.user,
      'Subscription Cancelled',
      'Your subscription has been cancelled due to non-payment.',
      {
        type: 'subscription_cancelled',
        subscription_id: subscription.id,
        reason: 'non_payment'
      }
    )
    
    # Schedule win-back campaign
    WinBackCampaignJob.set(wait: 30.days).perform_later(subscription.user.id)
  end
end
```

### 6. Subscription Analytics Service

```ruby
# app/services/subscription_analytics_service.rb
class SubscriptionAnalyticsService
  def self.generate_report(start_date = 30.days.ago, end_date = Time.current)
    new(start_date, end_date).generate_report
  end
  
  def initialize(start_date, end_date)
    @start_date = start_date
    @end_date = end_date
  end
  
  def generate_report
    {
      period: {
        start_date: @start_date,
        end_date: @end_date,
        days: (@end_date - @start_date).to_i / 1.day
      },
      overview: overview_metrics,
      revenue: revenue_metrics,
      subscriptions: subscription_metrics,
      churn: churn_analysis,
      plans: plan_performance,
      cohorts: cohort_analysis,
      predictions: predictions
    }
  end
  
  private
  
  def overview_metrics
    {
      total_subscribers: UserSubscription.active_or_trial.count,
      new_subscribers: UserSubscription.where(created_at: @start_date..@end_date).count,
      churned_subscribers: UserSubscription.where(cancelled_at: @start_date..@end_date).count,
      trial_conversions: trial_conversion_rate,
      mrr: calculate_mrr,
      arr: calculate_arr,
      average_revenue_per_user: calculate_arpu,
      lifetime_value: calculate_ltv
    }
  end
  
  def revenue_metrics
    paid_invoices = SubscriptionInvoice.where(
      paid_at: @start_date..@end_date,
      status: 'paid'
    )
    
    {
      total_revenue: paid_invoices.sum(:amount),
      recurring_revenue: paid_invoices.joins(:subscription)
                                    .where(user_subscriptions: { status: 'active' })
                                    .sum(:amount),
      trial_conversions_revenue: calculate_trial_conversion_revenue,
      upgrades_revenue: calculate_upgrade_revenue,
      refunds: calculate_refunds,
      revenue_by_plan: revenue_by_plan,
      revenue_growth_rate: calculate_revenue_growth_rate
    }
  end
  
  def subscription_metrics
    {
      total_active: UserSubscription.active_or_trial.count,
      total_trials: UserSubscription.where(status: 'trial').count,
      total_cancelled: UserSubscription.where(status: 'cancelled').count,
      total_expired: UserSubscription.where(status: 'expired').count,
      subscription_growth: calculate_subscription_growth,
      churn_rate: calculate_churn_rate,
      retention_rate: calculate_retention_rate,
      average_subscription_length: calculate_average_subscription_length
    }
  end
  
  def churn_analysis
    beginning_subscribers = UserSubscription.active_or_trial
                                          .where(created_at: ..@start_date)
                                          .count
    
    churned_subscribers = UserSubscription.where(cancelled_at: @start_date..@end_date).count
    
    churn_rate = beginning_subscribers > 0 ? (churned_subscribers.to_f / beginning_subscribers * 100) : 0
    
    {
      churn_rate: churn_rate.round(2),
      churned_count: churned_subscribers,
      churn_by_plan: churn_by_plan,
      churn_reasons: churn_reasons,
      time_to_churn: average_time_to_churn,
      voluntary_vs_involuntary: voluntary_vs_involuntary_churn
    }
  end
  
  def plan_performance
    SubscriptionPlan.active.map do |plan|
      subscriptions = plan.user_subscriptions.where(created_at: @start_date..@end_date)
      active_subscriptions = plan.user_subscriptions.active_or_trial
      
      {
        plan_id: plan.id,
        plan_name: plan.name,
        price: plan.price,
        new_subscriptions: subscriptions.count,
        total_active: active_subscriptions.count,
        revenue: active_subscriptions.sum { |s| s.plan.price },
        churn_rate: calculate_plan_churn_rate(plan),
        conversion_rate: calculate_plan_trial_conversion_rate(plan),
        popularity_rank: nil # Will be calculated after sorting
      }
    end.sort_by { |p| -p[:total_active] }
        .each_with_index { |plan, index| plan[:popularity_rank] = index + 1 }
  end
  
  def cohort_analysis
    # Group users by signup month and track retention
    cohorts = {}
    
    (0..11).each do |months_ago|
      cohort_start = months_ago.months.ago.beginning_of_month
      cohort_end = cohort_start.end_of_month
      
      cohort_users = Spree::User.joins(:user_subscriptions)
                               .where(created_at: cohort_start..cohort_end)
                               .distinct
      
      next if cohort_users.empty?
      
      retention_data = (0..months_ago).map do |month|
        retention_period = month.months.ago.beginning_of_month
        retained_users = cohort_users.joins(:user_subscriptions)
                                   .where(
                                     user_subscriptions: {
                                       current_period_start: ..retention_period.end_of_month,
                                       current_period_end: retention_period..
                                     }
                                   ).distinct.count
        
        {
          month: month,
          retained_users: retained_users,
          retention_rate: (retained_users.to_f / cohort_users.count * 100).round(2)
        }
      end
      
      cohorts[cohort_start.strftime('%Y-%m')] = {
        cohort_size: cohort_users.count,
        retention: retention_data
      }
    end
    
    cohorts
  end
  
  def predictions
    {
      next_month_mrr: predict_next_month_mrr,
      churn_risk_users: identify_churn_risk_users,
      expansion_opportunities: identify_expansion_opportunities,
      ltv_prediction: predict_customer_ltv
    }
  end
  
  def trial_conversion_rate
    trials_started = UserSubscription.where(
      trial_start: @start_date..@end_date
    ).where.not(trial_start: nil).count
    
    trials_converted = UserSubscription.where(
      trial_start: @start_date..@end_date,
      status: 'active'
    ).where.not(trial_start: nil).count
    
    return 0 if trials_started == 0
    
    (trials_converted.to_f / trials_started * 100).round(2)
  end
  
  def calculate_mrr
    active_subscriptions = UserSubscription.active_or_trial
    
    active_subscriptions.sum do |subscription|
      subscription.plan.monthly_equivalent_price
    end
  end
  
  def calculate_arr
    calculate_mrr * 12
  end
  
  def calculate_arpu
    total_revenue = SubscriptionInvoice.where(
      paid_at: @start_date..@end_date,
      status: 'paid'
    ).sum(:amount)
    
    unique_subscribers = UserSubscription.where(
      created_at: @start_date..@end_date
    ).distinct.count(:user_id)
    
    return 0 if unique_subscribers == 0
    
    (total_revenue / unique_subscribers).round(2)
  end
  
  def calculate_ltv
    average_monthly_revenue = calculate_arpu
    average_lifespan_months = calculate_average_subscription_length
    
    (average_monthly_revenue * average_lifespan_months).round(2)
  end
  
  def calculate_trial_conversion_revenue
    converted_trials = UserSubscription.joins(:subscription_invoices)
                                     .where(trial_start: @start_date..@end_date)
                                     .where(status: 'active')
                                     .where(subscription_invoices: { 
                                       paid_at: @start_date..@end_date,
                                       status: 'paid'
                                     })
    
    converted_trials.sum('subscription_invoices.amount')
  end
  
  def calculate_upgrade_revenue
    upgrade_events = SubscriptionEvent.where(
      event_type: 'plan_upgraded',
      created_at: @start_date..@end_date
    )
    
    upgrade_events.sum do |event|
      event.metadata['proration_amount']&.to_f || 0
    end
  end
  
  def calculate_refunds
    SubscriptionInvoice.where(
      status: 'refunded',
      updated_at: @start_date..@end_date
    ).sum(:amount)
  end
  
  def revenue_by_plan
    SubscriptionInvoice.joins(subscription: :plan)
                      .where(paid_at: @start_date..@end_date, status: 'paid')
                      .group('subscription_plans.name')
                      .sum(:amount)
  end
  
  def calculate_revenue_growth_rate
    current_period_revenue = SubscriptionInvoice.where(
      paid_at: @start_date..@end_date,
      status: 'paid'
    ).sum(:amount)
    
    previous_period_start = @start_date - (@end_date - @start_date)
    previous_period_revenue = SubscriptionInvoice.where(
      paid_at: previous_period_start..@start_date,
      status: 'paid'
    ).sum(:amount)
    
    return 0 if previous_period_revenue == 0
    
    ((current_period_revenue - previous_period_revenue) / previous_period_revenue * 100).round(2)
  end
  
  def calculate_subscription_growth
    new_subscriptions = UserSubscription.where(created_at: @start_date..@end_date).count
    cancelled_subscriptions = UserSubscription.where(cancelled_at: @start_date..@end_date).count
    
    new_subscriptions - cancelled_subscriptions
  end
  
  def calculate_churn_rate
    beginning_subscribers = UserSubscription.active_or_trial
                                          .where(created_at: ..@start_date)
                                          .count
    
    churned_subscribers = UserSubscription.where(cancelled_at: @start_date..@end_date).count
    
    return 0 if beginning_subscribers == 0
    
    (churned_subscribers.to_f / beginning_subscribers * 100).round(2)
  end
  
  def calculate_retention_rate
    100 - calculate_churn_rate
  end
  
  def calculate_average_subscription_length
    completed_subscriptions = UserSubscription.where(status: ['cancelled', 'expired'])
                                            .where.not(cancelled_at: nil)
    
    return 0 if completed_subscriptions.empty?
    
    total_days = completed_subscriptions.sum do |sub|
      end_date = sub.cancelled_at || sub.current_period_end
      (end_date - sub.created_at) / 1.day
    end
    
    (total_days / completed_subscriptions.count / 30).round(2) # Convert to months
  end
  
  def churn_by_plan
    UserSubscription.joins(:plan)
                   .where(cancelled_at: @start_date..@end_date)
                   .group('subscription_plans.name')
                   .count
  end
  
  def churn_reasons
    SubscriptionEvent.where(
      event_type: 'subscription_cancelled',
      created_at: @start_date..@end_date
    ).group("metadata->>'reason'").count
  end
  
  def average_time_to_churn
    churned_subscriptions = UserSubscription.where(cancelled_at: @start_date..@end_date)
    
    return 0 if churned_subscriptions.empty?
    
    total_days = churned_subscriptions.sum do |sub|
      (sub.cancelled_at - sub.created_at) / 1.day
    end
    
    (total_days / churned_subscriptions.count).round(2)
  end
  
  def voluntary_vs_involuntary_churn
    voluntary = SubscriptionEvent.where(
      event_type: 'subscription_cancelled',
      created_at: @start_date..@end_date
    ).where("metadata->>'cancelled_by' = ?", 'user').count
    
    involuntary = SubscriptionEvent.where(
      event_type: 'subscription_cancelled',
      created_at: @start_date..@end_date
    ).where("metadata->>'reason' = ?", 'non_payment').count
    
    {
      voluntary: voluntary,
      involuntary: involuntary,
      total: voluntary + involuntary
    }
  end
  
  def calculate_plan_churn_rate(plan)
    plan_subscriptions = plan.user_subscriptions
                            .where(created_at: ..@start_date)
                            .count
    
    plan_churned = plan.user_subscriptions
                      .where(cancelled_at: @start_date..@end_date)
                      .count
    
    return 0 if plan_subscriptions == 0
    
    (plan_churned.to_f / plan_subscriptions * 100).round(2)
  end
  
  def calculate_plan_trial_conversion_rate(plan)
    plan_trials = plan.user_subscriptions
                     .where(trial_start: @start_date..@end_date)
                     .where.not(trial_start: nil)
                     .count
    
    plan_conversions = plan.user_subscriptions
                          .where(trial_start: @start_date..@end_date)
                          .where(status: 'active')
                          .count
    
    return 0 if plan_trials == 0
    
    (plan_conversions.to_f / plan_trials * 100).round(2)
  end
  
  def predict_next_month_mrr
    # Simple linear regression based on last 6 months
    monthly_mrr = (0..5).map do |months_ago|
      date = months_ago.months.ago.beginning_of_month
      
      active_subs = UserSubscription.active_or_trial
                                  .where(created_at: ..date.end_of_month)
                                  .where('current_period_end > ?', date.beginning_of_month)
      
      active_subs.sum { |sub| sub.plan.monthly_equivalent_price }
    end.reverse
    
    # Calculate trend
    x_values = (0...monthly_mrr.size).to_a
    mean_x = x_values.sum.to_f / x_values.size
    mean_y = monthly_mrr.sum.to_f / monthly_mrr.size
    
    numerator = x_values.zip(monthly_mrr).sum { |x, y| (x - mean_x) * (y - mean_y) }
    denominator = x_values.sum { |x| (x - mean_x) ** 2 }
    
    slope = denominator != 0 ? numerator / denominator : 0
    intercept = mean_y - slope * mean_x
    
    next_month_prediction = slope * monthly_mrr.size + intercept
    [next_month_prediction, 0].max.round(2)
  end
  
  def identify_churn_risk_users
    # Users with payment failures or decreased usage
    risk_users = UserSubscription.active_or_trial
                                .joins(:user)
                                .where(
                                  'last_active_at < ? OR EXISTS (
                                    SELECT 1 FROM subscription_invoices 
                                    WHERE subscription_invoices.subscription_id = user_subscriptions.id 
                                    AND subscription_invoices.status = ? 
                                    AND subscription_invoices.created_at > ?
                                  )',
                                  7.days.ago,
                                  'failed',
                                  30.days.ago
                                )
    
    risk_users.limit(100).map do |subscription|
      {
        user_id: subscription.user.id,
        subscription_id: subscription.id,
        risk_factors: calculate_risk_factors(subscription),
        risk_score: calculate_risk_score(subscription)
      }
    end.sort_by { |user| -user[:risk_score] }
  end
  
  def identify_expansion_opportunities
    # Users on lower plans with high usage
    opportunities = UserSubscription.active_or_trial
                                  .joins(:plan, :feature_usages)
                                  .where('subscription_plans.price < ?', 19.99)
                                  .group('user_subscriptions.id')
                                  .having('AVG(feature_usages.usage_count) > ?', 80) # High usage
    
    opportunities.limit(50).map do |subscription|
      {
        user_id: subscription.user.id,
        current_plan: subscription.plan.name,
        usage_percentage: calculate_average_usage_percentage(subscription),
        recommended_plan: recommend_upgrade_plan(subscription),
        potential_additional_revenue: calculate_upgrade_revenue_potential(subscription)
      }
    end
  end
  
  def predict_customer_ltv
    # Use historical data to predict LTV
    historical_ltv = UserSubscription.where(status: ['cancelled', 'expired'])
                                   .joins(:subscription_invoices)
                                   .group(:user_id)
                                   .sum('subscription_invoices.amount')
    
    return 0 if historical_ltv.empty?
    
    historical_ltv.values.sum.to_f / historical_ltv.size
  end
  
  def calculate_risk_factors(subscription)
    factors = []
    
    # Payment issues
    if subscription.subscription_invoices.where(status: 'failed').where('created_at > ?', 30.days.ago).exists?
      factors << 'recent_payment_failure'
    end
    
    # Low activity
    if subscription.user.last_active_at < 7.days.ago
      factors << 'low_activity'
    end
    
    # High support tickets
    if subscription.user.support_tickets.where('created_at > ?', 30.days.ago).count > 3
      factors << 'high_support_usage'
    end
    
    # Feature usage decline
    if calculate_usage_trend(subscription) < -20
      factors << 'declining_usage'
    end
    
    factors
  end
  
  def calculate_risk_score(subscription)
    risk_factors = calculate_risk_factors(subscription)
    
    base_score = 0
    base_score += 40 if risk_factors.include?('recent_payment_failure')
    base_score += 20 if risk_factors.include?('low_activity')
    base_score += 15 if risk_factors.include?('high_support_usage')
    base_score += 25 if risk_factors.include?('declining_usage')
    
    [base_score, 100].min
  end
  
  def calculate_usage_trend(subscription)
    # Compare current month usage to previous month
    current_month_usage = subscription.feature_usages
                                    .where(usage_date: Date.current.beginning_of_month..Date.current)
                                    .sum(:usage_count)
    
    previous_month_usage = subscription.feature_usages
                                     .where(usage_date: 1.month.ago.beginning_of_month..1.month.ago.end_of_month)
                                     .sum(:usage_count)
    
    return 0 if previous_month_usage == 0
    
    ((current_month_usage - previous_month_usage).to_f / previous_month_usage * 100).round(2)
  end
  
  def calculate_average_usage_percentage(subscription)
    plan_features = subscription.plan.plan_features.where(enabled: true).where.not(limit_value: nil)
    return 0 if plan_features.empty?
    
    usage_percentages = plan_features.map do |feature|
      usage = subscription.usage_for_feature(feature.feature_name)
      limit = feature.limit_value
      
      limit > 0 ? (usage.to_f / limit * 100) : 0
    end
    
    (usage_percentages.sum / usage_percentages.size).round(2)
  end
  
  def recommend_upgrade_plan(subscription)
    current_price = subscription.plan.price
    
    SubscriptionPlan.active
                   .where('price > ?', current_price)
                   .order(:price)
                   .first
                   &.name
  end
  
  def calculate_upgrade_revenue_potential(subscription)
    recommended_plan_name = recommend_upgrade_plan(subscription)
    return 0 unless recommended_plan_name
    
    recommended_plan = SubscriptionPlan.find_by(name: recommended_plan_name)
    return 0 unless recommended_plan
    
    (recommended_plan.price - subscription.plan.price) * 12 # Annual difference
  end
end
```

### 7. Testing Suite

```ruby
# spec/services/subscription_service_spec.rb
require 'rails_helper'

RSpec.describe SubscriptionService do
  let(:user) { create(:user, :verified) }
  let(:plan) { create(:subscription_plan, price: 9.99, trial_days: 7) }
  let(:service) { described_class.new(user) }
  
  describe '#subscribe' do
    context 'with trial period' do
      it 'creates subscription in trial status' do
        subscription = service.subscribe(plan.id)
        
        expect(subscription.status).to eq('trial')
        expect(subscription.trial_end).to be_present
        expect(subscription.trial_days_remaining).to be > 0
      end
      
      it 'schedules trial end reminder' do
        expect(SubscriptionTrialReminderJob).to receive(:set).and_call_original
        expect(SubscriptionTrialReminderJob).to receive(:perform_later)
        
        service.subscribe(plan.id)
      end
    end
    
    context 'without trial period' do
      let(:plan) { create(:subscription_plan, price: 9.99, trial_days: 0) }
      let(:payment_method) { create(:payment_method, user: user) }
      
      it 'creates active subscription and processes payment' do
        allow(PaymentProcessor).to receive(:charge).and_return(
          double(success?: true, transaction_id: 'txn_123')
        )
        
        subscription = service.subscribe(plan.id, payment_method: payment_method.id)
        
        expect(subscription.status).to eq('active')
        expect(subscription.subscription_invoices.paid).to exist
      end
    end
  end
  
  describe '#upgrade' do
    let!(:subscription) { service.subscribe(plan.id) }
    let(:premium_plan) { create(:subscription_plan, price: 19.99) }
    
    before { subscription.activate! }
    
    it 'upgrades to higher tier plan' do
      allow(PaymentProcessor).to receive(:charge).and_return(
        double(success?: true, transaction_id: 'txn_upgrade')
      )
      
      upgraded = service.upgrade(premium_plan.id)
      
      expect(upgraded.plan).to eq(premium_plan)
      expect(upgraded.metadata['upgraded_from']).to eq(plan.id)
    end
    
    it 'calculates proration correctly' do
      proration = subscription.calculate_proration_for_upgrade(premium_plan)
      
      expect(proration[:amount]).to eq(premium_plan.price)
      expect(proration[:proration]).to be > 0
    end
  end
  
  describe '#cancel' do
    let!(:subscription) { service.subscribe(plan.id) }
    
    before { subscription.activate! }
    
    it 'cancels subscription immediately when requested' do
      result = service.cancel('user_request', true)
      
      expect(result).to be_truthy
      expect(subscription.reload.status).to eq('cancelled')
      expect(subscription.cancelled_at).to be_present
    end
    
    it 'schedules cancellation at period end' do
      result = service.cancel('user_request', false)
      
      expect(result).to be_truthy
      expect(subscription.reload.metadata['cancel_at_period_end']).to be_truthy
    end
  end
end

# spec/jobs/subscription_billing_job_spec.rb
require 'rails_helper'

RSpec.describe SubscriptionBillingJob do
  let(:user) { create(:user, :with_payment_method) }
  let(:plan) { create(:subscription_plan, price: 9.99) }
  let(:subscription) { create(:user_subscription, user: user, plan: plan, status: 'active') }
  
  describe '#perform' do
    it 'creates invoice and processes payment' do
      allow(PaymentProcessor).to receive(:charge).and_return(
        double(success?: true, transaction_id: 'txn_renewal')
      )
      
      expect {
        described_class.new.perform(subscription.id)
      }.to change { subscription.subscription_invoices.count }.by(1)
      
      invoice = subscription.subscription_invoices.last
      expect(invoice.status).to eq('paid')
      expect(subscription.reload.current_period_end).to be > Time.current
    end
    
    it 'handles payment failure gracefully' do
      allow(PaymentProcessor).to receive(:charge).and_return(
        double(success?: false, error_message: 'Card declined')
      )
      
      described_class.new.perform(subscription.id)
      
      invoice = subscription.subscription_invoices.last
      expect(invoice.status).to eq('failed')
      expect(subscription.reload.status).to eq('past_due')
    end
  end
end
```

---

## 🚀 Deployment & Monitoring

### Health Checks & Monitoring

```ruby
# app/controllers/api/v1/subscription_health_controller.rb
module Api
  module V1
    class SubscriptionHealthController < ApplicationController
      include ActionController::API
      
      def show
        health_data = {
          status: 'healthy',
          timestamp: Time.current,
          metrics: {
            active_subscriptions: UserSubscription.active_or_trial.count,
            failed_payments_last_24h: failed_payments_count,
            pending_invoices: pending_invoices_count,
            churn_rate_last_30_days: calculate_monthly_churn_rate
          },
          services: {
            database: check_database,
            payment_gateway: check_payment_gateway,
            background_jobs: check_background_jobs
          }
        }
        
        overall_healthy = health_data[:services].values.all? { |s| s[:status] == 'healthy' }
        status_code = overall_healthy ? 200 : 503
        
        render json: health_data, status: status_code
      end
      
      private
      
      def failed_payments_count
        SubscriptionInvoice.where(
          status: 'failed',
          updated_at: 24.hours.ago..Time.current
        ).count
      end
      
      def pending_invoices_count
        SubscriptionInvoice.where(status: 'pending').count
      end
      
      def calculate_monthly_churn_rate
        start_of_month = 30.days.ago
        beginning_subs = UserSubscription.active_or_trial.where(created_at: ..start_of_month).count
        churned_subs = UserSubscription.where(cancelled_at: start_of_month..Time.current).count
        
        return 0 if beginning_subs == 0
        (churned_subs.to_f / beginning_subs * 100).round(2)
      end
      
      def check_database
        UserSubscription.limit(1).count
        { status: 'healthy', response_time: 0 }
      rescue => e
        { status: 'unhealthy', error: e.message }
      end
      
      def check_payment_gateway
        # This would ping your payment provider's health endpoint
        { status: 'healthy' }
      rescue => e
        { status: 'unhealthy', error: e.message }
      end
      
      def check_background_jobs
        # Check if critical jobs are running
        recent_billing_jobs = Sidekiq::Stats.new.processed > 0
        { status: recent_billing_jobs ? 'healthy' : 'degraded' }
      rescue => e
        { status: 'unhealthy', error: e.message }
      end
    end
  end
end
```

---

## 📈 Next Steps

After completing Week 9-10:

1. **Load Testing**: Test subscription billing under high volume
2. **Payment Gateway Integration**: Complete integration with multiple providers
3. **Advanced Analytics**: Set up subscription cohort analysis
4. **Customer Success Tools**: Implement retention and expansion workflows  
5. **Prepare for Week 11-12**: Loyalty & Gamification implementation

The subscription system now provides a complete foundation for SaaS-style recurring billing with comprehensive plan management, feature access control, and detailed analytics.