# 🏆 Week 11-12: Loyalty & Gamification System Implementation

## 🎯 Overview
This phase focuses on implementing an advanced loyalty program with gamification elements, including points system, tier management, achievement tracking, and rewards redemption for the Hangmeas Super App.

---

## 📋 Development Checklist

### Week 11: Core Loyalty System

#### Day 1-2: Enhanced Points & Tier System
- [ ] Implement advanced loyalty point calculations
- [ ] Create tier progression with benefits
- [ ] Set up tier upgrade/downgrade logic
- [ ] Add tier-based multipliers and bonuses
- [ ] Create tier anniversary rewards

#### Day 3-4: Achievement & Badge System
- [ ] Create achievement engine with rules
- [ ] Implement badge collection system
- [ ] Add milestone achievements
- [ ] Create social sharing for achievements
- [ ] Set up achievement notifications

#### Day 5: Rewards & Redemption
- [ ] Create rewards catalog
- [ ] Implement redemption system
- [ ] Add digital rewards (discount codes, credits)
- [ ] Create physical rewards fulfillment
- [ ] Set up rewards inventory management

### Week 12: Gamification & Advanced Features

#### Day 1-2: Gamification Elements
- [ ] Implement daily/weekly challenges
- [ ] Create leaderboards and competitions
- [ ] Add streak tracking and bonuses
- [ ] Implement social features (friends, sharing)
- [ ] Create seasonal campaigns

#### Day 3-4: Advanced Analytics & Personalization
- [ ] Build loyalty analytics dashboard
- [ ] Implement personalized recommendations
- [ ] Create predictive engagement models
- [ ] Add A/B testing for campaigns
- [ ] Set up retention analysis

#### Day 5: Integration & API
- [ ] Create comprehensive loyalty APIs
- [ ] Integrate with mobile app features
- [ ] Add real-time notifications
- [ ] Create admin management interface
- [ ] Performance testing and optimization

---

## 🔨 Implementation Details

### 1. Enhanced Loyalty Models

```ruby
# app/models/loyalty_tier.rb
class LoyaltyTier < ApplicationRecord
  has_many :users, class_name: 'Spree::User', foreign_key: 'loyalty_tier_id'
  has_many :tier_benefits, dependent: :destroy
  has_many :tier_challenges, dependent: :destroy
  
  validates :name, :slug, presence: true, uniqueness: true
  validates :points_threshold, :spending_threshold, presence: true, numericality: { greater_than_or_equal_to: 0 }
  validates :level, presence: true, numericality: { greater_than: 0 }, uniqueness: true
  
  scope :ordered, -> { order(:level) }
  scope :active, -> { where(active: true) }
  
  before_validation :generate_slug
  
  def next_tier
    self.class.where('level > ?', level).order(:level).first
  end
  
  def previous_tier
    self.class.where('level < ?', level).order(level: :desc).first
  end
  
  def requirements_met?(user)
    user.loyalty_points >= points_threshold && user.total_spent >= spending_threshold
  end
  
  def benefits_summary
    tier_benefits.active.includes(:benefit_type).map do |benefit|
      {
        type: benefit.benefit_type.name,
        value: benefit.value,
        description: benefit.description,
        active: benefit.active?
      }
    end
  end
  
  def upgrade_bonus_points
    case level
    when 2 then 500   # Silver
    when 3 then 1500  # Gold
    when 4 then 3000  # Platinum
    when 5 then 5000  # Diamond
    else 0
    end
  end
  
  private
  
  def generate_slug
    self.slug = name.parameterize if name.present? && slug.blank?
  end
end

# app/models/tier_benefit.rb
class TierBenefit < ApplicationRecord
  belongs_to :loyalty_tier
  belongs_to :benefit_type
  
  validates :value, presence: true
  validates :benefit_type_id, uniqueness: { scope: :loyalty_tier_id }
  
  scope :active, -> { where(active: true) }
  
  def formatted_value
    case benefit_type.value_type
    when 'percentage'
      "#{value}%"
    when 'multiplier'
      "#{value}x"
    when 'amount'
      "$#{value}"
    when 'boolean'
      value ? 'Yes' : 'No'
    else
      value.to_s
    end
  end
end

# app/models/benefit_type.rb
class BenefitType < ApplicationRecord
  has_many :tier_benefits, dependent: :destroy
  
  validates :name, :key, :value_type, presence: true
  validates :key, uniqueness: true
  validates :value_type, inclusion: { in: %w[percentage multiplier amount boolean text] }
  
  # Standard benefit types
  STANDARD_BENEFITS = {
    'points_multiplier' => { name: 'Points Multiplier', value_type: 'multiplier' },
    'birthday_bonus' => { name: 'Birthday Bonus Points', value_type: 'amount' },
    'free_shipping_threshold' => { name: 'Free Shipping Threshold', value_type: 'amount' },
    'early_access' => { name: 'Early Access to Sales', value_type: 'boolean' },
    'exclusive_events' => { name: 'Exclusive Events Access', value_type: 'boolean' },
    'priority_support' => { name: 'Priority Customer Support', value_type: 'boolean' },
    'special_discounts' => { name: 'Special Discounts', value_type: 'percentage' },
    'bonus_votes' => { name: 'Bonus Votes per Episode', value_type: 'amount' },
    'vip_content' => { name: 'VIP Content Access', value_type: 'boolean' },
    'concierge_service' => { name: 'Personal Concierge', value_type: 'boolean' }
  }.freeze
  
  def self.seed_standard_benefits
    STANDARD_BENEFITS.each do |key, attributes|
      find_or_create_by(key: key) do |benefit|
        benefit.name = attributes[:name]
        benefit.value_type = attributes[:value_type]
        benefit.description = "Standard #{attributes[:name].downcase} benefit"
      end
    end
  end
end

# app/models/achievement.rb
class Achievement < ApplicationRecord
  belongs_to :achievement_category
  has_many :user_achievements, dependent: :destroy
  has_many :users, through: :user_achievements, class_name: 'Spree::User'
  has_one_attached :badge_image
  
  validates :name, :key, :points_reward, presence: true
  validates :key, uniqueness: true
  validates :points_reward, numericality: { greater_than: 0 }
  validates :difficulty, inclusion: { in: %w[easy medium hard legendary] }
  
  scope :active, -> { where(active: true) }
  scope :by_difficulty, ->(level) { where(difficulty: level) }
  scope :by_category, ->(category) { joins(:achievement_category).where(achievement_categories: { slug: category }) }
  
  def completion_rate
    total_users = Spree::User.count
    return 0 if total_users == 0
    
    (user_achievements.count.to_f / total_users * 100).round(2)
  end
  
  def rarity_level
    rate = completion_rate
    case rate
    when 0..1 then 'legendary'
    when 1..5 then 'epic'
    when 5..15 then 'rare'
    when 15..40 then 'uncommon'
    else 'common'
    end
  end
  
  def requirements_met?(user)
    return false unless active?
    
    case achievement_type
    when 'spending'
      user.total_spent >= target_value
    when 'points'
      user.loyalty_points >= target_value
    when 'orders'
      user.orders.complete.count >= target_value
    when 'voting'
      user.total_votes_cast >= target_value
    when 'referrals'
      user.referral_count >= target_value
    when 'streak'
      calculate_user_streak(user) >= target_value
    when 'social'
      user.social_shares_count >= target_value
    else
      false
    end
  end
  
  private
  
  def calculate_user_streak(user)
    # Calculate current streak based on achievement type
    case key
    when 'daily_login_streak'
      user.calculate_daily_login_streak
    when 'voting_streak'
      user.calculate_voting_streak
    when 'purchase_streak'
      user.calculate_purchase_streak
    else
      0
    end
  end
end

# app/models/user_achievement.rb
class UserAchievement < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :achievement
  
  validates :user_id, uniqueness: { scope: :achievement_id }
  
  scope :recent, -> { order(earned_at: :desc) }
  scope :by_category, ->(category) { joins(achievement: :achievement_category).where(achievement_categories: { slug: category }) }
  
  after_create :award_points, :send_achievement_notification, :check_meta_achievements
  
  def days_since_earned
    return 0 unless earned_at
    ((Time.current - earned_at) / 1.day).to_i
  end
  
  def share_text
    "I just earned the '#{achievement.name}' achievement on Hangmeas! 🏆 #HangmeasAchievement"
  end
  
  private
  
  def award_points
    user.loyalty.award_points(
      achievement.points_reward,
      "Achievement: #{achievement.name}",
      { achievement_id: achievement.id }
    )
  end
  
  def send_achievement_notification
    AchievementNotificationJob.perform_later(id)
  end
  
  def check_meta_achievements
    # Check for achievements about earning achievements
    MetaAchievementCheckJob.perform_later(user.id)
  end
end

# app/models/loyalty_challenge.rb
class LoyaltyChallenge < ApplicationRecord
  belongs_to :challenge_type
  has_many :user_challenge_progresses, dependent: :destroy
  has_many :participating_users, through: :user_challenge_progresses, source: :user, class_name: 'Spree::User'
  
  validates :name, :start_date, :end_date, :target_value, :points_reward, presence: true
  validates :end_date, comparison: { greater_than: :start_date }
  validates :target_value, :points_reward, numericality: { greater_than: 0 }
  validates :challenge_scope, inclusion: { in: %w[daily weekly monthly seasonal] }
  
  scope :active, -> { where('start_date <= ? AND end_date >= ?', Time.current, Time.current) }
  scope :upcoming, -> { where('start_date > ?', Time.current) }
  scope :completed, -> { where('end_date < ?', Time.current) }
  
  enum status: { draft: 0, active: 1, completed: 2, cancelled: 3 }
  
  def active?
    status == 'active' && Time.current.between?(start_date, end_date)
  end
  
  def days_remaining
    return 0 unless active?
    ((end_date - Time.current) / 1.day).to_i
  end
  
  def completion_rate
    return 0 if participating_users.count == 0
    
    completed_users = user_challenge_progresses.where('progress >= ?', target_value).count
    (completed_users.to_f / participating_users.count * 100).round(2)
  end
  
  def user_progress(user)
    user_challenge_progresses.find_by(user: user) || 
    user_challenge_progresses.build(user: user, progress: 0)
  end
  
  def eligible_for_user?(user)
    return false unless active?
    return false if tier_requirement.present? && user.loyalty_tier_level < tier_requirement
    return false if max_participants.present? && participating_users.count >= max_participants
    
    true
  end
end

# app/models/user_challenge_progress.rb
class UserChallengeProgress < ApplicationRecord
  belongs_to :user, class_name: 'Spree::User'
  belongs_to :loyalty_challenge
  
  validates :user_id, uniqueness: { scope: :loyalty_challenge_id }
  validates :progress, numericality: { greater_than_or_equal_to: 0 }
  
  after_update :check_completion, if: :saved_change_to_progress?
  
  def completion_percentage
    return 0 if loyalty_challenge.target_value == 0
    [(progress.to_f / loyalty_challenge.target_value * 100), 100].min.round(2)
  end
  
  def completed?
    progress >= loyalty_challenge.target_value
  end
  
  def remaining_progress
    [loyalty_challenge.target_value - progress, 0].max
  end
  
  private
  
  def check_completion
    return if completed_at.present? # Already completed
    return unless completed?
    
    self.completed_at = Time.current
    save!
    
    # Award challenge completion rewards
    ChallengeCompletionJob.perform_later(id)
  end
end
```

### 2. Advanced Loyalty Service

```ruby
# app/services/loyalty/engagement_service.rb
module Loyalty
  class EngagementService
    def initialize(user)
      @user = user
    end
    
    def calculate_engagement_score
      scores = {
        recency: calculate_recency_score,
        frequency: calculate_frequency_score,
        monetary: calculate_monetary_score,
        activity: calculate_activity_score,
        social: calculate_social_score
      }
      
      # Weighted average
      weights = { recency: 0.25, frequency: 0.25, monetary: 0.25, activity: 0.15, social: 0.10 }
      
      total_score = scores.sum { |metric, score| score * weights[metric] }
      
      {
        total_score: total_score.round(2),
        scores: scores,
        segment: determine_engagement_segment(total_score),
        recommendations: generate_recommendations(scores)
      }
    end
    
    def track_engagement_activity(activity_type, metadata = {})
      EngagementActivity.create!(
        user: @user,
        activity_type: activity_type,
        metadata: metadata,
        engagement_score: calculate_activity_score_impact(activity_type),
        created_at: Time.current
      )
      
      # Update user's engagement metrics
      update_user_engagement_metrics
      
      # Check for engagement-based achievements
      check_engagement_achievements(activity_type)
    end
    
    def get_personalized_offers
      engagement_data = calculate_engagement_score
      tier_level = @user.loyalty_tier_level
      
      offers = []
      
      # Points multiplier offers for low-activity users
      if engagement_data[:scores][:activity] < 50
        offers << {
          type: 'points_multiplier',
          title: 'Get Back in the Game!',
          description: 'Earn 2x points on your next purchase',
          multiplier: 2.0,
          expires_at: 7.days.from_now
        }
      end
      
      # Exclusive rewards for high-value users
      if engagement_data[:scores][:monetary] > 80
        offers << {
          type: 'exclusive_reward',
          title: 'VIP Exclusive',
          description: 'Access to limited edition merchandise',
          reward_id: find_exclusive_reward_id,
          expires_at: 30.days.from_now
        }
      end
      
      # Social sharing incentives
      if engagement_data[:scores][:social] < 30
        offers << {
          type: 'social_challenge',
          title: 'Share & Earn',
          description: 'Share your achievements and earn 100 bonus points',
          points_reward: 100,
          expires_at: 14.days.from_now
        }
      end
      
      offers
    end
    
    def predict_churn_risk
      engagement_score = calculate_engagement_score[:total_score]
      days_since_last_activity = (@user.last_active_at ? (Time.current - @user.last_active_at) / 1.day : 999).to_i
      
      risk_factors = []
      risk_score = 0
      
      # Low engagement
      if engagement_score < 30
        risk_factors << 'low_engagement'
        risk_score += 40
      end
      
      # Inactive for extended period
      if days_since_last_activity > 14
        risk_factors << 'inactive'
        risk_score += 30
      end
      
      # Declining spending
      if calculate_spending_trend < -50
        risk_factors << 'declining_spending'
        risk_score += 20
      end
      
      # No recent achievements
      if @user.user_achievements.where('earned_at > ?', 30.days.ago).empty?
        risk_factors << 'no_recent_achievements'
        risk_score += 10
      end
      
      {
        risk_score: [risk_score, 100].min,
        risk_level: categorize_risk_level(risk_score),
        risk_factors: risk_factors,
        recommended_actions: generate_retention_actions(risk_factors)
      }
    end
    
    def generate_milestone_rewards
      milestones = []
      
      # Points milestones
      next_points_milestone = calculate_next_milestone(@user.loyalty_points, [1000, 5000, 10000, 25000, 50000])
      if next_points_milestone
        milestones << {
          type: 'points',
          current: @user.loyalty_points,
          target: next_points_milestone,
          progress: (@user.loyalty_points.to_f / next_points_milestone * 100).round,
          reward: "#{next_points_milestone / 100} bonus points + exclusive badge"
        }
      end
      
      # Spending milestones
      next_spending_milestone = calculate_next_milestone(@user.total_spent.to_i, [100, 500, 1000, 2500, 5000])
      if next_spending_milestone
        milestones << {
          type: 'spending',
          current: @user.total_spent.to_i,
          target: next_spending_milestone,
          progress: (@user.total_spent / next_spending_milestone * 100).round,
          reward: "$#{next_spending_milestone / 10} store credit"
        }
      end
      
      # Voting milestones
      next_voting_milestone = calculate_next_milestone(@user.total_votes_cast, [100, 500, 1000, 5000, 10000])
      if next_voting_milestone
        milestones << {
          type: 'voting',
          current: @user.total_votes_cast,
          target: next_voting_milestone,
          progress: (@user.total_votes_cast.to_f / next_voting_milestone * 100).round,
          reward: "Exclusive voter badge + #{next_voting_milestone / 10} bonus votes"
        }
      end
      
      milestones
    end
    
    private
    
    def calculate_recency_score
      return 0 unless @user.last_active_at
      
      days_since_last_activity = (Time.current - @user.last_active_at) / 1.day
      
      case days_since_last_activity
      when 0..1 then 100
      when 1..3 then 90
      when 3..7 then 75
      when 7..14 then 50
      when 14..30 then 25
      else 10
      end
    end
    
    def calculate_frequency_score
      # Activity frequency in last 30 days
      activities = @user.engagement_activities.where('created_at > ?', 30.days.ago).count
      
      case activities
      when 0 then 0
      when 1..5 then 25
      when 6..15 then 50
      when 16..30 then 75
      else 100
      end
    end
    
    def calculate_monetary_score
      # Spending in last 90 days
      recent_spending = @user.orders.complete
                            .where('completed_at > ?', 90.days.ago)
                            .sum(:total)
      
      case recent_spending
      when 0 then 0
      when 0..50 then 25
      when 50..200 then 50
      when 200..500 then 75
      else 100
      end
    end
    
    def calculate_activity_score
      # Diverse activity engagement
      activity_types = @user.engagement_activities
                           .where('created_at > ?', 30.days.ago)
                           .distinct
                           .pluck(:activity_type)
                           .count
      
      case activity_types
      when 0 then 0
      when 1..2 then 25
      when 3..5 then 50
      when 6..8 then 75
      else 100
      end
    end
    
    def calculate_social_score
      # Social engagement (shares, referrals, etc.)
      social_activities = @user.engagement_activities
                              .where(activity_type: ['social_share', 'referral', 'review'])
                              .where('created_at > ?', 30.days.ago)
                              .count
      
      case social_activities
      when 0 then 0
      when 1..2 then 30
      when 3..5 then 60
      else 100
      end
    end
    
    def calculate_activity_score_impact(activity_type)
      case activity_type
      when 'login' then 1
      when 'purchase' then 10
      when 'vote' then 3
      when 'social_share' then 5
      when 'review' then 8
      when 'referral' then 15
      else 1
      end
    end
    
    def determine_engagement_segment(score)
      case score
      when 0..25 then 'at_risk'
      when 25..50 then 'developing'
      when 50..75 then 'engaged'
      else 'champion'
      end
    end
    
    def generate_recommendations(scores)
      recommendations = []
      
      recommendations << 'Increase app usage frequency' if scores[:frequency] < 50
      recommendations << 'Try making a purchase to boost your status' if scores[:monetary] < 30
      recommendations << 'Share your achievements on social media' if scores[:social] < 40
      recommendations << 'Log in daily to maintain your streak' if scores[:recency] < 70
      
      recommendations
    end
    
    def update_user_engagement_metrics
      engagement_data = calculate_engagement_score
      
      @user.update!(
        engagement_score: engagement_data[:total_score],
        engagement_segment: engagement_data[:segment],
        last_engagement_calculated_at: Time.current
      )
    end
    
    def check_engagement_achievements(activity_type)
      # Check for streak achievements
      if activity_type == 'login'
        streak = @user.calculate_daily_login_streak
        check_streak_achievement('daily_login', streak)
      elsif activity_type == 'vote'
        streak = @user.calculate_voting_streak
        check_streak_achievement('voting', streak)
      end
    end
    
    def check_streak_achievement(streak_type, streak_days)
      milestone_days = [7, 14, 30, 60, 100]
      
      milestone_days.each do |days|
        next unless streak_days >= days
        
        achievement_key = "#{streak_type}_streak_#{days}"
        achievement = Achievement.find_by(key: achievement_key)
        
        next unless achievement
        next if @user.user_achievements.exists?(achievement: achievement)
        
        @user.user_achievements.create!(
          achievement: achievement,
          earned_at: Time.current,
          metadata: { streak_days: streak_days }
        )
      end
    end
    
    def calculate_spending_trend
      current_month_spending = @user.orders.complete
                                   .where('completed_at > ?', 30.days.ago)
                                   .sum(:total)
      
      previous_month_spending = @user.orders.complete
                                    .where('completed_at > ? AND completed_at <= ?', 60.days.ago, 30.days.ago)
                                    .sum(:total)
      
      return 0 if previous_month_spending == 0
      
      ((current_month_spending - previous_month_spending) / previous_month_spending * 100).round(2)
    end
    
    def categorize_risk_level(score)
      case score
      when 0..25 then 'low'
      when 25..50 then 'medium'
      when 50..75 then 'high'
      else 'critical'
      end
    end
    
    def generate_retention_actions(risk_factors)
      actions = []
      
      actions << 'Send re-engagement campaign' if risk_factors.include?('low_engagement')
      actions << 'Offer comeback discount' if risk_factors.include?('inactive')
      actions << 'Provide exclusive early access' if risk_factors.include?('declining_spending')
      actions << 'Suggest easy achievements' if risk_factors.include?('no_recent_achievements')
      
      actions
    end
    
    def calculate_next_milestone(current_value, milestones)
      milestones.find { |milestone| current_value < milestone }
    end
    
    def find_exclusive_reward_id
      # Find available exclusive rewards for high-value customers
      Reward.exclusive.available.first&.id
    end
  end
end

# app/services/loyalty/gamification_service.rb
module Loyalty
  class GamificationService
    def initialize(user)
      @user = user
    end
    
    def create_daily_challenges
      return if @user.user_challenge_progresses.joins(:loyalty_challenge)
                    .where(loyalty_challenges: { challenge_scope: 'daily' })
                    .where('loyalty_challenges.start_date = ?', Date.current)
                    .exists?
      
      # Generate 3 daily challenges based on user behavior
      challenges = generate_personalized_challenges
      
      challenges.each do |challenge_data|
        challenge = LoyaltyChallenge.create!(
          name: challenge_data[:name],
          description: challenge_data[:description],
          challenge_scope: 'daily',
          challenge_type: ChallengeType.find_by(key: challenge_data[:type]),
          target_value: challenge_data[:target],
          points_reward: challenge_data[:points],
          start_date: Time.current.beginning_of_day,
          end_date: Time.current.end_of_day,
          status: 'active'
        )
        
        # Auto-enroll user
        challenge.user_challenge_progresses.create!(user: @user, progress: 0)
      end
    end
    
    def update_challenge_progress(activity_type, value = 1, metadata = {})
      active_challenges = @user.user_challenge_progresses
                              .joins(:loyalty_challenge)
                              .where(loyalty_challenges: { status: 'active' })
                              .where('loyalty_challenges.start_date <= ? AND loyalty_challenges.end_date >= ?', 
                                    Time.current, Time.current)
      
      active_challenges.each do |progress|
        challenge = progress.loyalty_challenge
        next unless challenge_applies_to_activity?(challenge, activity_type)
        
        old_progress = progress.progress
        progress.increment(:progress, value)
        progress.save!
        
        # Send progress update notification
        if progress.progress != old_progress
          ChallengeProgressNotificationJob.perform_later(progress.id, old_progress)
        end
      end
    end
    
    def generate_leaderboard(timeframe = 'weekly', category = 'overall')
      case timeframe
      when 'daily'
        start_time = Time.current.beginning_of_day
        end_time = Time.current.end_of_day
      when 'weekly'
        start_time = Time.current.beginning_of_week
        end_time = Time.current.end_of_week
      when 'monthly'
        start_time = Time.current.beginning_of_month
        end_time = Time.current.end_of_month
      else
        start_time = 30.days.ago
        end_time = Time.current
      end
      
      base_query = Spree::User.joins(:loyalty_activities)
                              .where(loyalty_activities: { created_at: start_time..end_time })
      
      case category
      when 'points'
        rankings = base_query.group(:id)
                            .sum('loyalty_activities.points')
                            .sort_by { |_, points| -points }
                            .first(100)
      when 'voting'
        rankings = base_query.joins(:votes)
                            .where(xfactor_votes: { created_at: start_time..end_time })
                            .group(:id)
                            .sum('xfactor_votes.vote_count')
                            .sort_by { |_, votes| -votes }
                            .first(100)
      when 'spending'
        rankings = base_query.joins(:orders)
                            .where(spree_orders: { completed_at: start_time..end_time, state: 'complete' })
                            .group(:id)
                            .sum('spree_orders.total')
                            .sort_by { |_, total| -total }
                            .first(100)
      else # overall engagement
        rankings = base_query.group(:id)
                            .average(:engagement_score)
                            .sort_by { |_, score| -(score || 0) }
                            .first(100)
      end
      
      leaderboard = rankings.map.with_index do |(user_id, value), index|
        user = Spree::User.find(user_id)
        {
          rank: index + 1,
          user: {
            id: user.id,
            name: user.display_name,
            avatar_url: user.avatar.attached? ? Rails.application.routes.url_helpers.url_for(user.avatar) : nil,
            loyalty_tier: user.loyalty_tier
          },
          value: value.round(2),
          badge: determine_rank_badge(index + 1)
        }
      end
      
      # Add current user's position if not in top 100
      user_position = find_user_position_in_category(@user, category, start_time, end_time)
      if user_position > 100
        user_value = calculate_user_value_for_category(@user, category, start_time, end_time)
        leaderboard << {
          rank: user_position,
          user: {
            id: @user.id,
            name: @user.display_name,
            avatar_url: @user.avatar.attached? ? Rails.application.routes.url_helpers.url_for(@user.avatar) : nil,
            loyalty_tier: @user.loyalty_tier
          },
          value: user_value,
          badge: nil,
          current_user: true
        }
      end
      
      {
        timeframe: timeframe,
        category: category,
        period: { start: start_time, end: end_time },
        leaderboard: leaderboard,
        total_participants: count_participants_in_category(category, start_time, end_time)
      }
    end
    
    def create_seasonal_campaign(season_name, duration_days = 30)
      campaign = SeasonalCampaign.create!(
        name: season_name,
        description: "Special #{season_name} campaign with exclusive rewards",
        start_date: Time.current,
        end_date: duration_days.days.from_now,
        status: 'active',
        theme_config: generate_seasonal_theme(season_name)
      )
      
      # Create seasonal challenges
      seasonal_challenges = generate_seasonal_challenges(season_name, campaign)
      
      # Create seasonal rewards
      seasonal_rewards = generate_seasonal_rewards(season_name, campaign)
      
      {
        campaign: campaign,
        challenges: seasonal_challenges,
        rewards: seasonal_rewards
      }
    end
    
    def calculate_friend_rankings
      # Get user's friends (this would depend on your social features implementation)
      friend_ids = @user.friendships.accepted.pluck(:friend_id)
      return [] if friend_ids.empty?
      
      friends = Spree::User.where(id: friend_ids)
      
      rankings = friends.map do |friend|
        {
          user: {
            id: friend.id,
            name: friend.display_name,
            avatar_url: friend.avatar.attached? ? Rails.application.routes.url_helpers.url_for(friend.avatar) : nil
          },
          loyalty_points: friend.loyalty_points,
          loyalty_tier: friend.loyalty_tier,
          total_achievements: friend.user_achievements.count,
          recent_activity: friend.engagement_activities.where('created_at > ?', 7.days.ago).count
        }
      end
      
      # Sort by loyalty points
      rankings.sort_by! { |friend| -friend[:loyalty_points] }
      
      # Add ranks
      rankings.each_with_index { |friend, index| friend[:rank] = index + 1 }
      
      # Find current user's position
      user_points = @user.loyalty_points
      user_rank = rankings.count { |friend| friend[:loyalty_points] > user_points } + 1
      
      {
        friends: rankings,
        user_rank: user_rank,
        total_friends: rankings.count
      }
    end
    
    private
    
    def generate_personalized_challenges
      user_behavior = analyze_user_behavior
      challenges = []
      
      # Challenge 1: Based on primary activity
      case user_behavior[:primary_activity]
      when 'voting'
        challenges << {
          name: 'Vote Champion',
          description: 'Cast 10 votes today',
          type: 'voting',
          target: 10,
          points: 50
        }
      when 'shopping'
        challenges << {
          name: 'Shopping Spree',
          description: 'Make a purchase today',
          type: 'purchase',
          target: 1,
          points: 100
        }
      else
        challenges << {
          name: 'Daily Explorer',
          description: 'Browse 5 different product categories',
          type: 'browsing',
          target: 5,
          points: 30
        }
      end
      
      # Challenge 2: Social engagement
      challenges << {
        name: 'Social Butterfly',
        description: 'Share an achievement or product',
        type: 'social',
        target: 1,
        points: 25
      }
      
      # Challenge 3: Based on tier level
      if @user.loyalty_tier_level < 3
        challenges << {
          name: 'Points Collector',
          description: 'Earn 100 loyalty points',
          type: 'points',
          target: 100,
          points: 20
        }
      else
        challenges << {
          name: 'Loyalty Elite',
          description: 'Help a friend by referring them',
          type: 'referral',
          target: 1,
          points: 200
        }
      end
      
      challenges
    end
    
    def analyze_user_behavior
      recent_activities = @user.engagement_activities
                              .where('created_at > ?', 7.days.ago)
                              .group(:activity_type)
                              .count
      
      primary_activity = recent_activities.max_by { |_, count| count }&.first || 'browsing'
      
      {
        primary_activity: primary_activity,
        activity_frequency: recent_activities.values.sum,
        diversity_score: recent_activities.keys.count
      }
    end
    
    def challenge_applies_to_activity?(challenge, activity_type)
      case challenge.challenge_type.key
      when 'voting' then activity_type == 'vote'
      when 'purchase' then activity_type == 'purchase'
      when 'social' then ['social_share', 'referral', 'review'].include?(activity_type)
      when 'browsing' then activity_type == 'browse'
      when 'points' then true # Points can come from any activity
      when 'referral' then activity_type == 'referral'
      else false
      end
    end
    
    def determine_rank_badge(rank)
      case rank
      when 1 then '🥇'
      when 2 then '🥈'
      when 3 then '🥉'
      when 4..10 then '⭐'
      else nil
      end
    end
    
    def find_user_position_in_category(user, category, start_time, end_time)
      # This would involve a complex query to find the user's actual rank
      # Simplified implementation
      case category
      when 'points'
        user_points = user.loyalty_activities
                         .where(created_at: start_time..end_time)
                         .sum(:points)
        
        better_users = Spree::User.joins(:loyalty_activities)
                                 .where(loyalty_activities: { created_at: start_time..end_time })
                                 .group(:id)
                                 .having('SUM(loyalty_activities.points) > ?', user_points)
                                 .count
        
        better_users + 1
      else
        999 # Placeholder
      end
    end
    
    def calculate_user_value_for_category(user, category, start_time, end_time)
      case category
      when 'points'
        user.loyalty_activities
            .where(created_at: start_time..end_time)
            .sum(:points)
      when 'voting'
        user.votes
            .where(created_at: start_time..end_time)
            .sum(:vote_count)
      when 'spending'
        user.orders
            .where(completed_at: start_time..end_time, state: 'complete')
            .sum(:total)
      else
        user.engagement_score || 0
      end
    end
    
    def count_participants_in_category(category, start_time, end_time)
      case category
      when 'points'
        Spree::User.joins(:loyalty_activities)
                   .where(loyalty_activities: { created_at: start_time..end_time })
                   .distinct
                   .count
      when 'voting'
        Spree::User.joins(:votes)
                   .where(xfactor_votes: { created_at: start_time..end_time })
                   .distinct
                   .count
      when 'spending'
        Spree::User.joins(:orders)
                   .where(spree_orders: { completed_at: start_time..end_time, state: 'complete' })
                   .distinct
                   .count
      else
        Spree::User.where.not(engagement_score: nil).count
      end
    end
    
    def generate_seasonal_theme(season_name)
      themes = {
        'Spring Festival' => {
          primary_color: '#FF69B4',
          secondary_color: '#98FB98',
          background_image: 'spring_bg.jpg',
          icon_style: 'flowers'
        },
        'Summer Splash' => {
          primary_color: '#FFD700',
          secondary_color: '#00BFFF',
          background_image: 'summer_bg.jpg',
          icon_style: 'beach'
        },
        'Autumn Harvest' => {
          primary_color: '#FF8C00',
          secondary_color: '#8B4513',
          background_image: 'autumn_bg.jpg',
          icon_style: 'leaves'
        },
        'Winter Wonder' => {
          primary_color: '#87CEEB',
          secondary_color: '#4169E1',
          background_image: 'winter_bg.jpg',
          icon_style: 'snowflakes'
        }
      }
      
      themes[season_name] || themes.values.first
    end
    
    def generate_seasonal_challenges(season_name, campaign)
      base_challenges = [
        {
          name: "#{season_name} Champion",
          description: "Complete 5 daily challenges during #{season_name}",
          target: 5,
          points: 500
        },
        {
          name: "#{season_name} Socialite",
          description: "Share 3 achievements during #{season_name}",
          target: 3,
          points: 300
        },
        {
          name: "#{season_name} Collector",
          description: "Earn 1000 points during #{season_name}",
          target: 1000,
          points: 200
        }
      ]
      
      base_challenges.map do |challenge_data|
        LoyaltyChallenge.create!(
          name: challenge_data[:name],
          description: challenge_data[:description],
          challenge_scope: 'seasonal',
          challenge_type: ChallengeType.find_by(key: 'points'),
          target_value: challenge_data[:target],
          points_reward: challenge_data[:points],
          start_date: campaign.start_date,
          end_date: campaign.end_date,
          seasonal_campaign: campaign,
          status: 'active'
        )
      end
    end
    
    def generate_seasonal_rewards(season_name, campaign)
      rewards = [
        {
          name: "#{season_name} Exclusive Badge",
          description: "Limited edition #{season_name} achievement badge",
          reward_type: 'digital',
          points_cost: 0, # Free for participation
          quantity_available: nil # Unlimited
        },
        {
          name: "#{season_name} Discount Pack",
          description: "20% discount on all purchases during #{season_name}",
          reward_type: 'discount',
          points_cost: 500,
          discount_percentage: 20,
          quantity_available: 1000
        },
        {
          name: "#{season_name} Premium Avatar",
          description: "Exclusive seasonal avatar frame",
          reward_type: 'cosmetic',
          points_cost: 300,
          quantity_available: nil
        }
      ]
      
      rewards.map do |reward_data|
        Reward.create!(
          name: reward_data[:name],
          description: reward_data[:description],
          reward_type: reward_data[:reward_type],
          points_cost: reward_data[:points_cost],
          quantity_available: reward_data[:quantity_available],
          seasonal_campaign: campaign,
          available_from: campaign.start_date,
          available_until: campaign.end_date,
          active: true
        )
      end
    end
  end
end
```

### 3. Loyalty APIs

```ruby
# app/controllers/api/v1/loyalty_controller.rb
module Api
  module V1
    class LoyaltyController < BaseController
      # GET /api/v1/loyalty/dashboard
      def dashboard
        engagement_service = Loyalty::EngagementService.new(current_user)
        gamification_service = Loyalty::GamificationService.new(current_user)
        
        render json: {
          success: true,
          data: {
            user_summary: user_loyalty_summary,
            tier_info: current_tier_info,
            points_breakdown: points_breakdown,
            achievements: recent_achievements,
            challenges: active_challenges,
            leaderboard: gamification_service.generate_leaderboard('weekly', 'overall')[:leaderboard].first(10),
            milestones: engagement_service.generate_milestone_rewards,
            personalized_offers: engagement_service.get_personalized_offers
          }
        }
      end
      
      # GET /api/v1/loyalty/points
      def points
        activities = current_user.loyalty_activities
                                .order(created_at: :desc)
                                .includes(:user)
                                .limit(50)
        
        render json: {
          success: true,
          data: {
            current_points: current_user.loyalty_points,
            lifetime_points: current_user.loyalty_activities.where('points > 0').sum(:points),
            points_spent: current_user.loyalty_activities.where('points < 0').sum(:points).abs,
            recent_activities: activities.map { |activity| format_loyalty_activity(activity) },
            tier_progress: calculate_tier_progress
          }
        }
      end
      
      # GET /api/v1/loyalty/achievements
      def achievements
        page = params[:page]&.to_i || 1
        per_page = [params[:per_page]&.to_i || 20, 50].min
        category = params[:category]
        
        user_achievements = current_user.user_achievements
                                       .includes(:achievement => :achievement_category)
                                       .order(earned_at: :desc)
        
        user_achievements = user_achievements.by_category(category) if category.present?
        user_achievements = user_achievements.page(page).per(per_page)
        
        # Get available achievements
        available_achievements = Achievement.active.includes(:achievement_category)
        available_achievements = available_achievements.by_category(category) if category.present?
        
        earned_achievement_ids = current_user.user_achievements.pluck(:achievement_id)
        unearned_achievements = available_achievements.where.not(id: earned_achievement_ids)
        
        render json: {
          success: true,
          data: {
            earned: user_achievements.map { |ua| format_user_achievement(ua) },
            available: unearned_achievements.limit(20).map { |a| format_achievement(a, current_user) },
            categories: AchievementCategory.active.map { |cat| format_category(cat) },
            stats: {
              total_earned: current_user.user_achievements.count,
              total_available: Achievement.active.count,
              completion_percentage: (current_user.user_achievements.count.to_f / Achievement.active.count * 100).round(2)
            },
            pagination: pagination_meta(user_achievements)
          }
        }
      end
      
      # GET /api/v1/loyalty/challenges
      def challenges
        gamification_service = Loyalty::GamificationService.new(current_user)
        
        # Get active challenges for user
        active_challenges = current_user.user_challenge_progresses
                                       .joins(:loyalty_challenge)
                                       .where(loyalty_challenges: { status: 'active' })
                                       .where('loyalty_challenges.end_date >= ?', Time.current)
                                       .includes(:loyalty_challenge)
        
        # Get available challenges user can join
        available_challenges = LoyaltyChallenge.active
                                             .where.not(
                                               id: current_user.user_challenge_progresses
                                                              .joins(:loyalty_challenge)
                                                              .pluck('loyalty_challenges.id')
                                             )
                                             .limit(10)
        
        render json: {
          success: true,
          data: {
            active: active_challenges.map { |progress| format_challenge_progress(progress) },
            available: available_challenges.map { |challenge| format_challenge(challenge) },
            completed_today: completed_challenges_today,
            daily_streak: calculate_challenge_streak,
            next_daily_reset: Time.current.end_of_day
          }
        }
      end
      
      # POST /api/v1/loyalty/challenges/:id/join
      def join_challenge
        challenge = LoyaltyChallenge.find(params[:id])
        
        unless challenge.eligible_for_user?(current_user)
          return error_response('Not eligible for this challenge', :forbidden)
        end
        
        if current_user.user_challenge_progresses.exists?(loyalty_challenge: challenge)
          return error_response('Already participating in this challenge', :conflict)
        end
        
        progress = current_user.user_challenge_progresses.create!(
          loyalty_challenge: challenge,
          progress: 0
        )
        
        success_response(
          format_challenge_progress(progress),
          'Successfully joined challenge'
        )
      end
      
      # GET /api/v1/loyalty/leaderboard
      def leaderboard
        timeframe = params[:timeframe] || 'weekly'
        category = params[:category] || 'overall'
        
        gamification_service = Loyalty::GamificationService.new(current_user)
        leaderboard_data = gamification_service.generate_leaderboard(timeframe, category)
        
        render json: {
          success: true,
          data: leaderboard_data
        }
      end
      
      # GET /api/v1/loyalty/rewards
      def rewards
        page = params[:page]&.to_i || 1
        per_page = [params[:per_page]&.to_i || 20, 50].min
        category = params[:category]
        
        available_rewards = Reward.available.active
        available_rewards = available_rewards.where(reward_category: category) if category.present?
        available_rewards = available_rewards.page(page).per(per_page)
        
        user_rewards = current_user.user_rewards.includes(:reward).recent.limit(10)
        
        render json: {
          success: true,
          data: {
            available: available_rewards.map { |reward| format_reward(reward) },
            user_rewards: user_rewards.map { |ur| format_user_reward(ur) },
            user_points: current_user.loyalty_points,
            categories: reward_categories,
            pagination: pagination_meta(available_rewards)
          }
        }
      end
      
      # POST /api/v1/loyalty/rewards/:id/redeem
      def redeem_reward
        reward = Reward.find(params[:id])
        
        unless reward.available? && reward.active?
          return error_response('Reward not available', :unprocessable_entity)
        end
        
        if current_user.loyalty_points < reward.points_cost
          return error_response('Insufficient points', :payment_required)
        end
        
        if reward.quantity_available.present? && reward.quantity_redeemed >= reward.quantity_available
          return error_response('Reward out of stock', :unprocessable_entity)
        end
        
        ActiveRecord::Base.transaction do
          # Deduct points
          current_user.loyalty.redeem_points(reward.points_cost, "Redeemed: #{reward.name}")
          
          # Create user reward
          user_reward = current_user.user_rewards.create!(
            reward: reward,
            points_cost: reward.points_cost,
            status: 'pending',
            redeemed_at: Time.current
          )
          
          # Update reward quantity
          if reward.quantity_available.present?
            reward.increment!(:quantity_redeemed)
          end
          
          # Process reward fulfillment
          RewardFulfillmentJob.perform_later(user_reward.id)
          
          success_response(
            format_user_reward(user_reward),
            'Reward redeemed successfully'
          )
        end
      rescue ActiveRecord::RecordInvalid => e
        error_response('Redemption failed', :unprocessable_entity, e.record.errors.full_messages)
      end
      
      # GET /api/v1/loyalty/tiers
      def tiers
        tiers = LoyaltyTier.active.ordered.includes(:tier_benefits => :benefit_type)
        current_tier = current_user.loyalty_tier_record
        
        render json: {
          success: true,
          data: {
            current_tier: current_tier ? format_tier(current_tier) : nil,
            all_tiers: tiers.map { |tier| format_tier(tier) },
            progress_to_next: calculate_tier_progress,
            tier_history: current_user.tier_histories.recent.limit(5).map { |th| format_tier_history(th) }
          }
        }
      end
      
      # POST /api/v1/loyalty/share_achievement
      def share_achievement
        achievement_id = params[:achievement_id]
        platform = params[:platform] # 'facebook', 'twitter', 'instagram'
        
        user_achievement = current_user.user_achievements.find(achievement_id)
        
        # Track social sharing activity
        engagement_service = Loyalty::EngagementService.new(current_user)
        engagement_service.track_engagement_activity('social_share', {
          achievement_id: achievement_id,
          platform: platform
        })
        
        # Award sharing points
        current_user.loyalty.award_points(25, "Shared achievement: #{user_achievement.achievement.name}")
        
        # Generate share content
        share_content = generate_share_content(user_achievement, platform)
        
        success_response(
          {
            share_url: share_content[:url],
            share_text: share_content[:text],
            share_image: share_content[:image],
            points_earned: 25
          },
          'Achievement shared successfully'
        )
      end
      
      private
      
      def user_loyalty_summary
        {
          id: current_user.id,
          name: current_user.display_name,
          loyalty_points: current_user.loyalty_points,
          loyalty_tier: current_user.loyalty_tier,
          tier_level: current_user.loyalty_tier_level,
          total_spent: current_user.total_spent.to_f,
          member_since: current_user.created_at,
          achievements_count: current_user.user_achievements.count,
          engagement_score: current_user.engagement_score || 0,
          engagement_segment: current_user.engagement_segment || 'developing'
        }
      end
      
      def current_tier_info
        tier = current_user.loyalty_tier_record
        return nil unless tier
        
        {
          name: tier.name,
          level: tier.level,
          benefits: tier.benefits_summary,
          requirements_met: tier.requirements_met?(current_user),
          next_tier: tier.next_tier ? format_tier(tier.next_tier) : nil
        }
      end
      
      def points_breakdown
        {
          earned_this_month: current_user.loyalty_activities
                                        .where('created_at >= ? AND points > 0', Time.current.beginning_of_month)
                                        .sum(:points),
          spent_this_month: current_user.loyalty_activities
                                       .where('created_at >= ? AND points < 0', Time.current.beginning_of_month)
                                       .sum(:points).abs,
          top_earning_activities: current_user.loyalty_activities
                                             .where('created_at >= ? AND points > 0', 30.days.ago)
                                             .group(:reason)
                                             .sum(:points)
                                             .sort_by { |_, points| -points }
                                             .first(5)
                                             .to_h
        }
      end
      
      def recent_achievements
        current_user.user_achievements
                   .includes(:achievement)
                   .order(earned_at: :desc)
                   .limit(5)
                   .map { |ua| format_user_achievement(ua) }
      end
      
      def active_challenges
        current_user.user_challenge_progresses
                   .joins(:loyalty_challenge)
                   .where(loyalty_challenges: { status: 'active' })
                   .where('loyalty_challenges.end_date >= ?', Time.current)
                   .limit(5)
                   .map { |progress| format_challenge_progress(progress) }
      end
      
      def calculate_tier_progress
        current_tier = current_user.loyalty_tier_record
        next_tier = current_tier&.next_tier
        
        return nil unless next_tier
        
        points_progress = [(current_user.loyalty_points.to_f / next_tier.points_threshold * 100), 100].min
        spending_progress = [(current_user.total_spent.to_f / next_tier.spending_threshold * 100), 100].min
        
        {
          points: {
            current: current_user.loyalty_points,
            required: next_tier.points_threshold,
            remaining: [next_tier.points_threshold - current_user.loyalty_points, 0].max,
            percentage: points_progress.round(2)
          },
          spending: {
            current: current_user.total_spent.to_f,
            required: next_tier.spending_threshold.to_f,
            remaining: [next_tier.spending_threshold - current_user.total_spent, 0].max.to_f,
            percentage: spending_progress.round(2)
          },
          overall_progress: [points_progress, spending_progress].min.round(2),
          next_tier_name: next_tier.name
        }
      end
      
      def format_loyalty_activity(activity)
        {
          id: activity.id,
          points: activity.points,
          reason: activity.reason,
          activity_type: activity.activity_type,
          created_at: activity.created_at,
          metadata: activity.metadata || {}
        }
      end
      
      def format_user_achievement(user_achievement)
        achievement = user_achievement.achievement
        
        {
          id: user_achievement.id,
          achievement: format_achievement(achievement, current_user),
          earned_at: user_achievement.earned_at,
          days_since_earned: user_achievement.days_since_earned,
          metadata: user_achievement.metadata || {}
        }
      end
      
      def format_achievement(achievement, user)
        {
          id: achievement.id,
          name: achievement.name,
          description: achievement.description,
          points_reward: achievement.points_reward,
          difficulty: achievement.difficulty,
          rarity_level: achievement.rarity_level,
          completion_rate: achievement.completion_rate,
          requirements_met: achievement.requirements_met?(user),
          progress: calculate_achievement_progress(achievement, user),
          category: {
            id: achievement.achievement_category.id,
            name: achievement.achievement_category.name
          },
          badge_image_url: achievement.badge_image.attached? ? rails_blob_url(achievement.badge_image) : nil
        }
      end
      
      def calculate_achievement_progress(achievement, user)
        current_value = case achievement.achievement_type
                       when 'spending' then user.total_spent
                       when 'points' then user.loyalty_points
                       when 'orders' then user.orders.complete.count
                       when 'voting' then user.total_votes_cast
                       when 'referrals' then user.referral_count
                       else 0
                       end
        
        progress_percentage = achievement.target_value > 0 ? 
          [(current_value.to_f / achievement.target_value * 100), 100].min : 0
        
        {
          current: current_value,
          target: achievement.target_value,
          percentage: progress_percentage.round(2),
          remaining: [achievement.target_value - current_value, 0].max
        }
      end
      
      def format_challenge_progress(progress)
        challenge = progress.loyalty_challenge
        
        {
          id: progress.id,
          challenge: format_challenge(challenge),
          progress: progress.progress,
          target: challenge.target_value,
          completion_percentage: progress.completion_percentage,
          completed: progress.completed?,
          completed_at: progress.completed_at,
          remaining_progress: progress.remaining_progress
        }
      end
      
      def format_challenge(challenge)
        {
          id: challenge.id,
          name: challenge.name,
          description: challenge.description,
          challenge_scope: challenge.challenge_scope,
          target_value: challenge.target_value,
          points_reward: challenge.points_reward,
          start_date: challenge.start_date,
          end_date: challenge.end_date,
          days_remaining: challenge.days_remaining,
          participants_count: challenge.participating_users.count,
          completion_rate: challenge.completion_rate,
          eligible: challenge.eligible_for_user?(current_user)
        }
      end
      
      def format_reward(reward)
        {
          id: reward.id,
          name: reward.name,
          description: reward.description,
          reward_type: reward.reward_type,
          points_cost: reward.points_cost,
          quantity_available: reward.quantity_available,
          quantity_remaining: reward.quantity_available ? reward.quantity_available - reward.quantity_redeemed : nil,
          available_from: reward.available_from,
          available_until: reward.available_until,
          image_url: reward.image.attached? ? rails_blob_url(reward.image) : nil,
          can_afford: current_user.loyalty_points >= reward.points_cost,
          estimated_delivery: reward.estimated_delivery_days
        }
      end
      
      def format_user_reward(user_reward)
        {
          id: user_reward.id,
          reward: format_reward(user_reward.reward),
          points_cost: user_reward.points_cost,
          status: user_reward.status,
          redeemed_at: user_reward.redeemed_at,
          fulfilled_at: user_reward.fulfilled_at,
          tracking_info: user_reward.tracking_info,
          digital_code: user_reward.digital_code
        }
      end
      
      def format_tier(tier)
        {
          id: tier.id,
          name: tier.name,
          level: tier.level,
          points_threshold: tier.points_threshold,
          spending_threshold: tier.spending_threshold.to_f,
          benefits: tier.benefits_summary,
          requirements_met: tier.requirements_met?(current_user),
          upgrade_bonus_points: tier.upgrade_bonus_points
        }
      end
      
      def format_tier_history(tier_history)
        {
          id: tier_history.id,
          from_tier: tier_history.from_tier,
          to_tier: tier_history.to_tier,
          changed_at: tier_history.created_at,
          reason: tier_history.reason,
          bonus_points_awarded: tier_history.bonus_points_awarded
        }
      end
      
      def format_category(category)
        {
          id: category.id,
          name: category.name,
          slug: category.slug,
          description: category.description,
          achievements_count: category.achievements.active.count
        }
      end
      
      def completed_challenges_today
        current_user.user_challenge_progresses
                   .where('completed_at >= ?', Time.current.beginning_of_day)
                   .count
      end
      
      def calculate_challenge_streak
        # Calculate consecutive days with completed challenges
        streak_days = 0
        check_date = Date.current
        
        while check_date > 30.days.ago.to_date
          daily_completions = current_user.user_challenge_progresses
                                         .where(completed_at: check_date.beginning_of_day..check_date.end_of_day)
                                         .count
          
          if daily_completions > 0
            streak_days += 1
            check_date -= 1.day
          else
            break
          end
        end
        
        streak_days
      end
      
      def reward_categories
        Reward.distinct.pluck(:reward_category).compact.map do |category|
          {
            name: category.titleize,
            slug: category,
            count: Reward.available.where(reward_category: category).count
          }
        end
      end
      
      def generate_share_content(user_achievement, platform)
        achievement = user_achievement.achievement
        
        base_text = "I just earned the '#{achievement.name}' achievement on Hangmeas! 🏆"
        
        case platform
        when 'facebook'
          {
            url: "https://hangmeas.com/achievements/#{achievement.id}",
            text: "#{base_text} Join me and start earning rewards! #HangmeasAchievement",
            image: achievement.badge_image.attached? ? rails_blob_url(achievement.badge_image) : nil
          }
        when 'twitter'
          {
            url: "https://hangmeas.com/achievements/#{achievement.id}",
            text: "#{base_text} #HangmeasAchievement #Loyalty #Rewards",
            image: achievement.badge_image.attached? ? rails_blob_url(achievement.badge_image) : nil
          }
        when 'instagram'
          {
            url: "https://hangmeas.com/achievements/#{achievement.id}",
            text: "#{base_text}\n\n#hangmeas #achievement #loyalty #rewards #cambodia",
            image: achievement.badge_image.attached? ? rails_blob_url(achievement.badge_image) : nil
          }
        else
          {
            url: "https://hangmeas.com/achievements/#{achievement.id}",
            text: base_text,
            image: achievement.badge_image.attached? ? rails_blob_url(achievement.badge_image) : nil
          }
        end
      end
    end
  end
end
```

---

## 🎮 Gamification Background Jobs

```ruby
# app/jobs/daily_challenge_generation_job.rb
class DailyChallengeGenerationJob < ApplicationJob
  queue_as :default
  
  def perform
    # Generate daily challenges for all active users
    active_users = Spree::User.where('last_active_at > ?', 7.days.ago)
    
    active_users.find_each do |user|
      gamification_service = Loyalty::GamificationService.new(user)
      gamification_service.create_daily_challenges
    end
  end
end

# app/jobs/achievement_check_job.rb
class AchievementCheckJob < ApplicationJob
  queue_as :default
  
  def perform(user_id, activity_type = nil)
    user = Spree::User.find(user_id)
    
    # Get achievements that might be triggered by this activity
    achievements_to_check = if activity_type
                           Achievement.active.where(achievement_type: activity_type)
                         else
                           Achievement.active
                         end
    
    # Filter out already earned achievements
    earned_achievement_ids = user.user_achievements.pluck(:achievement_id)
    achievements_to_check = achievements_to_check.where.not(id: earned_achievement_ids)
    
    achievements_to_check.each do |achievement|
      next unless achievement.requirements_met?(user)
      
      user_achievement = user.user_achievements.create!(
        achievement: achievement,
        earned_at: Time.current,
        metadata: calculate_achievement_metadata(user, achievement)
      )
      
      Rails.logger.info "Achievement earned: User #{user.id} earned #{achievement.name}"
    end
  end
  
  private
  
  def calculate_achievement_metadata(user, achievement)
    {
      user_level_at_earning: user.loyalty_tier,
      points_at_earning: user.loyalty_points,
      spending_at_earning: user.total_spent,
      calculated_at: Time.current
    }
  end
end

# app/jobs/tier_evaluation_job.rb
class TierEvaluationJob < ApplicationJob
  queue_as :default
  
  def perform(user_id)
    user = Spree::User.find(user_id)
    current_tier = user.loyalty_tier_record
    
    # Find the appropriate tier based on current points and spending
    eligible_tiers = LoyaltyTier.active.where(
      'points_threshold <= ? AND spending_threshold <= ?',
      user.loyalty_points,
      user.total_spent
    ).order(level: :desc)
    
    new_tier = eligible_tiers.first
    
    return unless new_tier && new_tier != current_tier
    
    # Check if it's an upgrade or downgrade
    if new_tier.level > (current_tier&.level || 0)
      process_tier_upgrade(user, current_tier, new_tier)
    elsif new_tier.level < current_tier.level
      process_tier_downgrade(user, current_tier, new_tier)
    end
  end
  
  private
  
  def process_tier_upgrade(user, old_tier, new_tier)
    Rails.logger.info "Tier upgrade: User #{user.id} upgraded from #{old_tier&.name} to #{new_tier.name}"
    
    ActiveRecord::Base.transaction do
      # Update user tier
      user.update!(loyalty_tier: new_tier.name)
      
      # Award upgrade bonus points
      bonus_points = new_tier.upgrade_bonus_points
      if bonus_points > 0
        user.loyalty.award_points(bonus_points, "Tier upgrade bonus - #{new_tier.name}")
      end
      
      # Create tier history record
      user.tier_histories.create!(
        from_tier: old_tier&.name,
        to_tier: new_tier.name,
        reason: 'automatic_upgrade',
        bonus_points_awarded: bonus_points
      )
      
      # Send congratulations
      TierUpgradeNotificationJob.perform_later(user.id, old_tier&.name, new_tier.name)
    end
  end
  
  def process_tier_downgrade(user, old_tier, new_tier)
    Rails.logger.info "Tier downgrade: User #{user.id} downgraded from #{old_tier.name} to #{new_tier.name}"
    
    # Update user tier
    user.update!(loyalty_tier: new_tier.name)
    
    # Create tier history record
    user.tier_histories.create!(
      from_tier: old_tier.name,
      to_tier: new_tier.name,
      reason: 'automatic_downgrade'
    )
    
    # Send notification
    TierChangeNotificationJob.perform_later(user.id, old_tier.name, new_tier.name, 'downgrade')
  end
end

# app/jobs/engagement_analytics_job.rb
class EngagementAnalyticsJob < ApplicationJob
  queue_as :low_priority
  
  def perform
    # Calculate engagement scores for all users
    Spree::User.where('last_active_at > ?', 90.days.ago).find_each do |user|
      engagement_service = Loyalty::EngagementService.new(user)
      engagement_data = engagement_service.calculate_engagement_score
      
      user.update!(
        engagement_score: engagement_data[:total_score],
        engagement_segment: engagement_data[:segment],
        last_engagement_calculated_at: Time.current
      )
      
      # Check for at-risk users and create retention tasks
      if engagement_data[:segment] == 'at_risk'
        RetentionCampaignJob.perform_later(user.id)
      end
    end
  end
end
```

---

## 📊 Loyalty Analytics Dashboard

```ruby
# app/services/loyalty/analytics_service.rb
module Loyalty
  class AnalyticsService
    def self.generate_dashboard_data(date_range = 30.days.ago..Time.current)
      new(date_range).generate_dashboard_data
    end
    
    def initialize(date_range)
      @date_range = date_range
    end
    
    def generate_dashboard_data
      {
        period: {
          start_date: @date_range.begin,
          end_date: @date_range.end,
          days: (@date_range.end - @date_range.begin).to_i / 1.day
        },
        engagement_overview: engagement_metrics,
        points_economy: points_economy_metrics,
        achievement_stats: achievement_statistics,
        tier_distribution: tier_distribution_data,
        challenge_performance: challenge_performance_data,
        retention_analysis: retention_analysis_data,
        top_performers: top_performers_data,
        predictions: predictive_analytics
      }
    end
    
    private
    
    def engagement_metrics
      total_users = Spree::User.count
      active_users = Spree::User.where(last_active_at: @date_range).count
      
      {
        total_users: total_users,
        active_users: active_users,
        engagement_rate: total_users > 0 ? (active_users.to_f / total_users * 100).round(2) : 0,
        segments: Spree::User.group(:engagement_segment).count,
        average_engagement_score: Spree::User.where.not(engagement_score: nil).average(:engagement_score)&.round(2),
        daily_active_users: calculate_daily_active_users,
        session_frequency: calculate_session_frequency
      }
    end
    
    def points_economy_metrics
      points_awarded = LoyaltyActivity.where(created_at: @date_range, activity_type: 'earn').sum(:points)
      points_redeemed = LoyaltyActivity.where(created_at: @date_range, activity_type: 'redeem').sum(:points).abs
      
      {
        points_awarded: points_awarded,
        points_redeemed: points_redeemed,
        net_points_change: points_awarded - points_redeemed,
        redemption_rate: points_awarded > 0 ? (points_redeemed.to_f / points_awarded * 100).round(2) : 0,
        average_points_per_user: points_awarded > 0 ? (points_awarded.to_f / Spree::User.joins(:loyalty_activities).where(loyalty_activities: { created_at: @date_range }).distinct.count).round : 0,
        top_point_sources: LoyaltyActivity.where(created_at: @date_range, activity_type: 'earn')
                                         .group(:reason)
                                         .sum(:points)
                                         .sort_by { |_, points| -points }
                                         .first(10)
                                         .to_h
      }
    end
    
    def achievement_statistics
      achievements_earned = UserAchievement.where(earned_at: @date_range).count
      unique_earners = UserAchievement.where(earned_at: @date_range).distinct.count(:user_id)
      
      {
        achievements_earned: achievements_earned,
        unique_earners: unique_earners,
        average_per_user: unique_earners > 0 ? (achievements_earned.to_f / unique_earners).round(2) : 0,
        most_earned: UserAchievement.joins(:achievement)
                                   .where(earned_at: @date_range)
                                   .group('achievements.name')
                                   .count
                                   .sort_by { |_, count| -count }
                                   .first(10)
                                   .to_h,
        rarest_earned: find_rarest_achievements_earned,
        completion_rates: Achievement.active.map do |achievement|
          {
            name: achievement.name,
            completion_rate: achievement.completion_rate,
            difficulty: achievement.difficulty
          }
        end.sort_by { |a| a[:completion_rate] }
      }
    end
    
    def tier_distribution_data
      tier_counts = Spree::User.group(:loyalty_tier).count
      tier_movements = calculate_tier_movements
      
      {
        current_distribution: tier_counts,
        tier_movements: tier_movements,
        upgrade_conversion_rates: calculate_tier_conversion_rates,
        average_time_in_tier: calculate_average_time_in_tier,
        tier_engagement: calculate_tier_engagement_scores
      }
    end
    
    def challenge_performance_data
      completed_challenges = UserChallengeProgress.where(completed_at: @date_range).count
      total_participants = UserChallengeProgress.joins(:loyalty_challenge)
                                              .where(loyalty_challenges: { start_date: @date_range })
                                              .distinct
                                              .count(:user_id)
      
      {
        completed_challenges: completed_challenges,
        total_participants: total_participants,
        completion_rate: total_participants > 0 ? (completed_challenges.to_f / total_participants * 100).round(2) : 0,
        most_popular_challenges: UserChallengeProgress.joins(:loyalty_challenge)
                                                    .where(created_at: @date_range)
                                                    .group('loyalty_challenges.name')
                                                    .count
                                                    .sort_by { |_, count| -count }
                                                    .first(10)
                                                    .to_h,
        challenge_difficulty_performance: analyze_challenge_difficulty_performance,
        daily_vs_weekly_performance: analyze_challenge_scope_performance
      }
    end
    
    def retention_analysis_data
      # Cohort-based retention analysis
      cohort_data = {}
      
      # Analyze last 12 months of cohorts
      (0..11).each do |months_ago|
        cohort_start = months_ago.months.ago.beginning_of_month
        cohort_end = cohort_start.end_of_month
        
        cohort_users = Spree::User.where(created_at: cohort_start..cohort_end)
        next if cohort_users.empty?
        
        retention_rates = (0..months_ago).map do |month_offset|
          retention_month = month_offset.months.ago.beginning_of_month
          
          retained_users = cohort_users.joins(:loyalty_activities)
                                     .where(loyalty_activities: { 
                                       created_at: retention_month..retention_month.end_of_month 
                                     })
                                     .distinct
                                     .count
          
          {
            month: month_offset,
            retained_count: retained_users,
            retention_rate: (retained_users.to_f / cohort_users.count * 100).round(2)
          }
        end
        
        cohort_data[cohort_start.strftime('%Y-%m')] = {
          cohort_size: cohort_users.count,
          retention: retention_rates
        }
      end
      
      {
        cohort_analysis: cohort_data,
        churn_risk_users: identify_churn_risk_users.count,
        reactivation_success_rate: calculate_reactivation_success_rate,
        loyalty_program_impact: calculate_loyalty_program_impact
      }
    end
    
    def top_performers_data
      {
        top_point_earners: Spree::User.joins(:loyalty_activities)
                                    .where(loyalty_activities: { created_at: @date_range, activity_type: 'earn' })
                                    .group('spree_users.id', 'spree_users.first_name', 'spree_users.last_name', 'spree_users.loyalty_tier')
                                    .sum('loyalty_activities.points')
                                    .sort_by { |_, points| -points }
                                    .first(10)
                                    .map { |(id, fname, lname, tier), points| { 
                                      user_id: id, 
                                      name: "#{fname} #{lname}", 
                                      tier: tier,
                                      points: points 
                                    }},
        
        most_achievements: Spree::User.joins(:user_achievements)
                                    .where(user_achievements: { earned_at: @date_range })
                                    .group('spree_users.id', 'spree_users.first_name', 'spree_users.last_name')
                                    .count
                                    .sort_by { |_, count| -count }
                                    .first(10)
                                    .map { |(id, fname, lname), count| { 
                                      user_id: id, 
                                      name: "#{fname} #{lname}", 
                                      achievements: count 
                                    }},
        
        challenge_champions: UserChallengeProgress.joins(:user, :loyalty_challenge)
                                                .where(completed_at: @date_range)
                                                .group('spree_users.id', 'spree_users.first_name', 'spree_users.last_name')
                                                .count
                                                .sort_by { |_, count| -count }
                                                .first(10)
                                                .map { |(id, fname, lname), count| { 
                                                  user_id: id, 
                                                  name: "#{fname} #{lname}", 
                                                  challenges_completed: count 
                                                }}
      }
    end
    
    def predictive_analytics
      {
        churn_predictions: predict_monthly_churn,
        engagement_trends: calculate_engagement_trends,
        tier_upgrade_predictions: predict_tier_upgrades,
        points_economy_forecast: forecast_points_economy
      }
    end
    
    def calculate_daily_active_users
      (@date_range.begin.to_date..@date_range.end.to_date).map do |date|
        {
          date: date,
          active_users: Spree::User.where(last_active_at: date.beginning_of_day..date.end_of_day).count
        }
      end
    end
    
    def calculate_session_frequency
      users_with_sessions = Spree::User.where(last_active_at: @date_range)
      return 0 if users_with_sessions.empty?
      
      total_sessions = users_with_sessions.sum do |user|
        user.engagement_activities.where(activity_type: 'login', created_at: @date_range).count
      end
      
      (total_sessions.to_f / users_with_sessions.count).round(2)
    end
    
    def find_rarest_achievements_earned
      UserAchievement.joins(:achievement)
                    .where(earned_at: @date_range)
                    .group('achievements.name', 'achievements.id')
                    .count
                    .map { |(name, id), count| 
                      achievement = Achievement.find(id)
                      { 
                        name: name, 
                        earned_count: count, 
                        completion_rate: achievement.completion_rate 
                      }
                    }
                    .sort_by { |a| a[:completion_rate] }
                    .first(5)
    end
    
    def calculate_tier_movements
      tier_changes = TierHistory.where(created_at: @date_range)
      
      {
        upgrades: tier_changes.where(reason: 'automatic_upgrade').count,
        downgrades: tier_changes.where(reason: 'automatic_downgrade').count,
        upgrade_paths: tier_changes.where(reason: 'automatic_upgrade')
                                 .group(:from_tier, :to_tier)
                                 .count,
        average_upgrade_time: calculate_average_upgrade_time
      }
    end
    
    def calculate_tier_conversion_rates
      LoyaltyTier.active.map do |tier|
        next_tier = tier.next_tier
        next unless next_tier
        
        eligible_users = Spree::User.where(loyalty_tier: tier.name).count
        upgraded_users = TierHistory.where(
          from_tier: tier.name,
          to_tier: next_tier.name,
          created_at: @date_range
        ).count
        
        {
          from_tier: tier.name,
          to_tier: next_tier.name,
          eligible_users: eligible_users,
          upgraded_users: upgraded_users,
          conversion_rate: eligible_users > 0 ? (upgraded_users.to_f / eligible_users * 100).round(2) : 0
        }
      end.compact
    end
    
    def calculate_average_time_in_tier
      # Simplified calculation - would need more sophisticated logic for accurate results
      LoyaltyTier.active.map do |tier|
        users_in_tier = Spree::User.where(loyalty_tier: tier.name)
        
        if users_in_tier.any?
          avg_days = users_in_tier.average('EXTRACT(DAYS FROM AGE(CURRENT_DATE, created_at))')&.round(2) || 0
          
          {
            tier: tier.name,
            average_days: avg_days,
            current_users: users_in_tier.count
          }
        end
      end.compact
    end
    
    def calculate_tier_engagement_scores
      Spree::User.group(:loyalty_tier)
                 .average(:engagement_score)
                 .transform_values { |score| score&.round(2) || 0 }
    end
    
    def analyze_challenge_difficulty_performance
      UserChallengeProgress.joins(loyalty_challenge: :challenge_type)
                         .where(completed_at: @date_range)
                         .group('challenge_types.difficulty')
                         .count
    end
    
    def analyze_challenge_scope_performance
      UserChallengeProgress.joins(:loyalty_challenge)
                         .where(completed_at: @date_range)
                         .group('loyalty_challenges.challenge_scope')
                         .count
    end
    
    def identify_churn_risk_users
      Spree::User.where(
        'last_active_at < ? AND engagement_score < ?',
        14.days.ago,
        30
      )
    end
    
    def calculate_reactivation_success_rate
      # Users who were inactive but became active again
      reactivated_users = Spree::User.where(
        'last_active_at BETWEEN ? AND ? AND EXISTS (
          SELECT 1 FROM engagement_activities 
          WHERE engagement_activities.user_id = spree_users.id 
          AND activity_type = ? 
          AND created_at > last_active_at - INTERVAL ? DAY
        )',
        30.days.ago, @date_range.end,
        'reactivation_campaign',
        14
      ).count
      
      total_reactivation_campaigns = EngagementActivity.where(
        activity_type: 'reactivation_campaign',
        created_at: 30.days.ago..@date_range.end
      ).distinct.count(:user_id)
      
      return 0 if total_reactivation_campaigns == 0
      
      (reactivated_users.to_f / total_reactivation_campaigns * 100).round(2)
    end
    
    def calculate_loyalty_program_impact
      # Compare metrics before and after loyalty program participation
      program_participants = Spree::User.joins(:loyalty_activities)
                                       .where(loyalty_activities: { created_at: @date_range })
                                       .distinct
      
      return {} if program_participants.empty?
      
      {
        participants_count: program_participants.count,
        average_order_value_increase: calculate_aov_increase(program_participants),
        purchase_frequency_increase: calculate_frequency_increase(program_participants),
        retention_improvement: calculate_retention_improvement(program_participants)
      }
    end
    
    def predict_monthly_churn
      # Simple prediction based on current trends
      current_churn_rate = calculate_current_churn_rate
      
      {
        current_rate: current_churn_rate,
        predicted_rate: current_churn_rate * 1.1, # Simplified prediction
        at_risk_users: identify_churn_risk_users.count,
        recommended_actions: generate_churn_prevention_actions
      }
    end
    
    def calculate_engagement_trends
      # Calculate engagement trend over the date range
      daily_scores = (@date_range.begin.to_date..@date_range.end.to_date).map do |date|
        avg_score = Spree::User.where(last_engagement_calculated_at: date.beginning_of_day..date.end_of_day)
                              .average(:engagement_score) || 0
        
        { date: date, average_engagement: avg_score.round(2) }
      end
      
      trend_direction = calculate_trend_direction(daily_scores.map { |d| d[:average_engagement] })
      
      {
        daily_scores: daily_scores,
        trend: trend_direction,
        overall_change: calculate_overall_engagement_change
      }
    end
    
    def predict_tier_upgrades
      # Users close to tier upgrades
      upcoming_upgrades = []
      
      LoyaltyTier.active.each do |tier|
        next_tier = tier.next_tier
        next unless next_tier
        
        close_to_upgrade = Spree::User.where(loyalty_tier: tier.name)
                                     .where('loyalty_points >= ? OR total_spent >= ?',
                                           next_tier.points_threshold * 0.8,
                                           next_tier.spending_threshold * 0.8)
        
        upcoming_upgrades << {
          from_tier: tier.name,
          to_tier: next_tier.name,
          candidates: close_to_upgrade.count,
          estimated_upgrades_30_days: (close_to_upgrade.count * 0.3).round
        }
      end
      
      upcoming_upgrades
    end
    
    def forecast_points_economy
      # Forecast based on current trends
      current_daily_award = LoyaltyActivity.where(
        activity_type: 'earn',
        created_at: 7.days.ago..Time.current
      ).sum(:points) / 7.0
      
      current_daily_redemption = LoyaltyActivity.where(
        activity_type: 'redeem',
        created_at: 7.days.ago..Time.current
      ).sum(:points).abs / 7.0
      
      {
        daily_points_awarded: current_daily_award.round,
        daily_points_redeemed: current_daily_redemption.round,
        net_daily_change: (current_daily_award - current_daily_redemption).round,
        30_day_forecast: {
          points_awarded: (current_daily_award * 30).round,
          points_redeemed: (current_daily_redemption * 30).round,
          net_change: ((current_daily_award - current_daily_redemption) * 30).round
        },
        inflation_risk: current_daily_award > current_daily_redemption * 2 ? 'high' : 'normal'
      }
    end
    
    # Helper methods for complex calculations
    
    def calculate_average_upgrade_time
      upgrade_histories = TierHistory.where(reason: 'automatic_upgrade')
                                   .joins(:user)
                                   .group(:user_id)
                                   .order('MIN(tier_histories.created_at)')
      
      return 0 if upgrade_histories.empty?
      
      total_days = upgrade_histories.sum do |history|
        user_signup = Spree::User.find(history.user_id).created_at
        (history.created_at - user_signup) / 1.day
      end
      
      (total_days / upgrade_histories.count).round(2)
    end
    
    def calculate_aov_increase(participants)
      # Simplified calculation - compare before/after loyalty program participation
      before_aov = participants.joins(:orders)
                              .where('spree_orders.completed_at < spree_users.created_at + INTERVAL ? DAY', 30)
                              .where(spree_orders: { state: 'complete' })
                              .average('spree_orders.total') || 0
      
      after_aov = participants.joins(:orders)
                             .where('spree_orders.completed_at > spree_users.created_at + INTERVAL ? DAY', 30)
                             .where(spree_orders: { state: 'complete' })
                             .average('spree_orders.total') || 0
      
      return 0 if before_aov == 0
      
      ((after_aov - before_aov) / before_aov * 100).round(2)
    end
    
    def calculate_frequency_increase(participants)
      # Compare order frequency before and after loyalty program
      0 # Placeholder - would need more sophisticated calculation
    end
    
    def calculate_retention_improvement(participants)
      # Compare retention rates of loyalty program participants vs non-participants
      0 # Placeholder - would need more sophisticated calculation
    end
    
    def calculate_current_churn_rate
      active_30_days_ago = Spree::User.where('last_active_at <= ?', 30.days.ago).count
      still_active = Spree::User.where(
        'last_active_at <= ? AND last_active_at >= ?', 
        30.days.ago, 60.days.ago
      ).count
      
      return 0 if active_30_days_ago == 0
      
      churned = active_30_days_ago - still_active
      (churned.to_f / active_30_days_ago * 100).round(2)
    end
    
    def generate_churn_prevention_actions
      [
        'Send re-engagement emails to inactive users',
        'Offer personalized rewards to at-risk users',
        'Create easy achievement opportunities',
        'Launch win-back campaigns with special offers'
      ]
    end
    
    def calculate_trend_direction(values)
      return 'stable' if values.length < 2
      
      first_half = values[0...values.length/2].sum / (values.length/2).to_f
      second_half = values[values.length/2..-1].sum / (values.length - values.length/2).to_f
      
      difference = second_half - first_half
      
      case difference
      when -Float::INFINITY..-1 then 'declining'
      when -1..1 then 'stable'
      else 'improving'
      end
    end
    
    def calculate_overall_engagement_change
      start_score = Spree::User.where(last_engagement_calculated_at: @date_range.begin.beginning_of_day..@date_range.begin.end_of_day)
                              .average(:engagement_score) || 0
      
      end_score = Spree::User.where(last_engagement_calculated_at: @date_range.end.beginning_of_day..@date_range.end.end_of_day)
                            .average(:engagement_score) || 0
      
      return 0 if start_score == 0
      
      ((end_score - start_score) / start_score * 100).round(2)
    end
  end
end
```

---

## 📈 Performance & Monitoring

```ruby
# lib/tasks/loyalty.rake
namespace :loyalty do
  desc "Generate daily challenges for all active users"
  task generate_daily_challenges: :environment do
    DailyChallengeGenerationJob.perform_now
    puts "Daily challenges generated"
  end
  
  desc "Check achievements for all users"
  task check_achievements: :environment do
    Spree::User.where('last_active_at > ?', 7.days.ago).find_each do |user|
      AchievementCheckJob.perform_later(user.id)
    end
    puts "Achievement checks queued"
  end
  
  desc "Evaluate tier upgrades/downgrades"
  task evaluate_tiers: :environment do
    Spree::User.where('loyalty_points_updated_at > ?', 1.day.ago).find_each do |user|
      TierEvaluationJob.perform_later(user.id)
    end
    puts "Tier evaluations queued"
  end
  
  desc "Calculate engagement scores"
  task calculate_engagement: :environment do
    EngagementAnalyticsJob.perform_now
    puts "Engagement scores calculated"
  end
  
  desc "Generate loyalty analytics report"
  task generate_report: :environment do
    report = Loyalty::AnalyticsService.generate_dashboard_data
    puts "Analytics report generated with #{report[:engagement_overview][:total_users]} total users"
  end
  
  desc "Seed standard benefit types"
  task seed_benefits: :environment do
    BenefitType.seed_standard_benefits
    puts "Standard benefit types seeded"
  end
end

# Schedule these tasks with whenever gem or cron
# 
# # config/schedule.rb (if using whenever gem)
# every 1.day, at: '6:00 am' do
#   rake 'loyalty:generate_daily_challenges'
# end
# 
# every 1.hour do
#   rake 'loyalty:check_achievements'
# end
# 
# every 1.day, at: '2:00 am' do
#   rake 'loyalty:evaluate_tiers'
#   rake 'loyalty:calculate_engagement'
# end
```

---

## 🚀 Next Steps & Integration

After completing Week 11-12:

1. **Mobile App Integration**: Ensure all loyalty features work seamlessly in mobile app
2. **Performance Testing**: Test gamification features under load
3. **A/B Testing Setup**: Create framework for testing different reward strategies
4. **Advanced Analytics**: Set up real-time dashboards for loyalty metrics
5. **Customer Support Tools**: Create admin interface for loyalty management
6. **Marketing Integration**: Connect loyalty data with email/SMS campaigns

The loyalty and gamification system now provides a comprehensive engagement platform that drives user retention and increases lifetime value through points, achievements, challenges, and personalized rewards.