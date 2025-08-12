# 🗳️ Week 5-6: XFactor Voting System Implementation

## 🎯 Overview
This phase focuses on implementing the complete XFactor voting system, including vote casting functionality, real-time results, WebSocket integration, and vote package purchases.

---

## 📋 Development Checklist

### Week 5: Core Voting Functionality

#### Day 1-2: Vote Casting System
- [ ] Implement vote validation and limits
- [ ] Create vote processing service
- [ ] Set up vote balance management
- [ ] Implement SMS voting integration
- [ ] Create vote transaction logging

#### Day 3-4: Vote Package Integration
- [ ] Create vote package products in Spree
- [ ] Implement package purchase flow
- [ ] Set up automatic vote credit system
- [ ] Create bonus vote calculations
- [ ] Test end-to-end purchase flow

#### Day 5: Real-time Infrastructure
- [ ] Set up Redis for ActionCable
- [ ] Configure WebSocket connections
- [ ] Implement voting channels
- [ ] Create real-time update jobs
- [ ] Test WebSocket performance

### Week 6: Advanced Features & API

#### Day 1-2: Real-time Results
- [ ] Implement live vote counting
- [ ] Create result broadcasting system
- [ ] Set up result caching strategy
- [ ] Implement leaderboard updates
- [ ] Create vote analytics

#### Day 3-4: Mobile API Endpoints
- [ ] Create voting API controllers
- [ ] Implement authentication middleware
- [ ] Add rate limiting for votes
- [ ] Create API documentation
- [ ] Test API endpoints

#### Day 5: Integration & Testing
- [ ] Full system integration test
- [ ] Load testing for concurrent votes
- [ ] SMS gateway integration test
- [ ] Push notification setup
- [ ] Performance optimization

---

## 🔨 Implementation Details

### 1. Vote Processing Service

```ruby
# app/services/xfactor/vote_processor.rb
module Xfactor
  class VoteProcessor
    include ActiveModel::Model
    
    attr_accessor :user, :contestant_id, :episode_id, :vote_type, :vote_count, :metadata
    
    validates :user, :contestant_id, :episode_id, :vote_type, :vote_count, presence: true
    validates :vote_count, numericality: { greater_than: 0 }
    
    def process
      return false unless valid?
      
      ActiveRecord::Base.transaction do
        @contestant = find_contestant
        @episode = find_episode
        
        validate_voting_eligibility!
        validate_vote_limits!
        
        case vote_type
        when 'free'
          process_free_vote
        when 'paid'
          process_paid_vote
        when 'sms'
          process_sms_vote
        when 'bundle'
          process_bundle_vote
        else
          errors.add(:vote_type, 'Invalid vote type')
          raise ActiveRecord::Rollback
        end
        
        create_vote_record
        update_statistics
        broadcast_update
        send_confirmation
        
        true
      end
    rescue StandardError => e
      errors.add(:base, e.message)
      false
    end
    
    private
    
    def find_contestant
      Contestant.find(contestant_id)
    rescue ActiveRecord::RecordNotFound
      errors.add(:contestant_id, 'Contestant not found')
      raise ActiveRecord::Rollback
    end
    
    def find_episode
      Episode.find(episode_id)
    rescue ActiveRecord::RecordNotFound
      errors.add(:episode_id, 'Episode not found')
      raise ActiveRecord::Rollback
    end
    
    def validate_voting_eligibility!
      unless @episode.voting_active?
        raise VotingClosedError, 'Voting is not currently active for this episode'
      end
      
      unless @episode.contestants.include?(@contestant)
        raise InvalidContestantError, 'Contestant is not in this episode'
      end
      
      if @contestant.eliminated?
        raise ContestantEliminatedError, 'Contestant has been eliminated'
      end
    end
    
    def validate_vote_limits!
      case vote_type
      when 'free'
        validate_free_vote_limit!
      when 'paid'
        validate_paid_vote_balance!
      when 'sms'
        validate_sms_vote_eligibility!
      end
    end
    
    def validate_free_vote_limit!
      used_votes = user.votes
                      .where(episode: @episode, vote_type: 'free')
                      .sum(:vote_count)
      
      remaining = Vote::FREE_VOTE_LIMIT - used_votes
      
      if vote_count > remaining
        raise VoteLimitExceededError, "You can only cast #{remaining} more free votes"
      end
    end
    
    def validate_paid_vote_balance!
      balance = user.wallet.balance('VOTES')
      
      if balance < vote_count
        raise InsufficientVoteBalanceError, "Insufficient vote balance. You have #{balance} votes"
      end
    end
    
    def validate_sms_vote_eligibility!
      unless user.phone_verified_at.present?
        raise PhoneNotVerifiedError, 'Phone number must be verified for SMS voting'
      end
      
      # Check daily SMS limit
      daily_sms_votes = user.votes
                           .where(vote_type: 'sms', created_at: Date.current.all_day)
                           .sum(:vote_count)
      
      if daily_sms_votes + vote_count > 50 # Daily limit
        raise SMSLimitExceededError, 'Daily SMS vote limit exceeded'
      end
    end
    
    def process_free_vote
      # Free votes don't require payment
      @transaction_id = "FREE_#{SecureRandom.hex(8)}"
    end
    
    def process_paid_vote
      # Deduct from vote balance
      store_credit = Spree::StoreCredit.create!(
        user: user,
        amount: -vote_count,
        currency: 'VOTES',
        transaction_type: 'vote_cast',
        reason: "Vote for #{@contestant.name} - Episode #{@episode.episode_number}",
        metadata: {
          contestant_id: contestant_id,
          episode_id: episode_id,
          vote_count: vote_count
        }.to_json
      )
      
      @transaction_id = store_credit.reference_number
    end
    
    def process_sms_vote
      # Create SMS vote log
      sms_log = SmsVoteLog.create!(
        user: user,
        phone_number: user.phone_number,
        carrier: detect_carrier(user.phone_number),
        short_code: '1255', # XFactor short code
        message_content: "VOTE #{@contestant.id}",
        charge_amount: 0.25 * vote_count, # $0.25 per SMS vote
        status: 'pending'
      )
      
      # Queue SMS billing job
      SmsBillingJob.perform_later(sms_log.id)
      
      @transaction_id = "SMS_#{sms_log.id}"
    end
    
    def process_bundle_vote
      # Bundle votes are pre-purchased
      bundle = user.active_vote_bundle
      
      if bundle.nil? || bundle.remaining_votes < vote_count
        raise InsufficientBundleVotesError, 'Insufficient bundle votes'
      end
      
      bundle.decrement!(:remaining_votes, vote_count)
      @transaction_id = "BUNDLE_#{bundle.id}"
    end
    
    def create_vote_record
      @vote = Vote.create!(
        user: user,
        contestant: @contestant,
        episode: @episode,
        vote_type: vote_type,
        vote_count: vote_count,
        amount_paid: calculate_amount_paid,
        payment_method: detect_payment_method,
        transaction_id: @transaction_id,
        ip_address: metadata[:ip_address],
        user_agent: metadata[:user_agent],
        metadata: metadata.to_json
      )
    end
    
    def calculate_amount_paid
      case vote_type
      when 'free'
        0
      when 'sms'
        0.25 * vote_count
      when 'paid'
        # Track the original purchase price if available
        metadata[:purchase_price] || 0
      else
        0
      end
    end
    
    def detect_payment_method
      case vote_type
      when 'free'
        'free'
      when 'sms'
        'sms'
      when 'paid'
        'wallet'
      when 'bundle'
        'bundle'
      end
    end
    
    def update_statistics
      # Update contestant stats
      @contestant.increment!(:total_votes_received, vote_count)
      
      # Update episode stats
      @episode.increment!(:total_votes, vote_count)
      
      # Update user stats
      user.increment!(:total_votes_cast, vote_count)
      
      # Award loyalty points
      points = calculate_loyalty_points
      user.loyalty.award_points(points, "XFactor voting - #{vote_count} votes")
    end
    
    def calculate_loyalty_points
      base_points = vote_count * 2
      
      # Bonus points for paid votes
      if vote_type == 'paid'
        base_points * 1.5
      else
        base_points
      end.to_i
    end
    
    def broadcast_update
      # Broadcast to episode channel
      VotingResultsBroadcastJob.perform_later(@episode.id)
      
      # Broadcast to contestant channel
      ContestantVoteUpdateJob.perform_later(@contestant.id, @episode.id)
    end
    
    def send_confirmation
      # Send push notification
      if user.push_token.present?
        PushNotificationJob.perform_later(
          user.id,
          'Vote Confirmed! 🎉',
          "Your #{vote_count} vote(s) for #{@contestant.name} have been counted!",
          {
            type: 'vote_confirmation',
            contestant_id: @contestant.id,
            episode_id: @episode.id,
            vote_count: vote_count
          }
        )
      end
      
      # Log activity
      ActivityLogger.log(
        user: user,
        action: 'vote_cast',
        resource: @vote,
        metadata: {
          contestant_name: @contestant.name,
          episode_number: @episode.episode_number,
          vote_type: vote_type,
          vote_count: vote_count
        }
      )
    end
    
    def detect_carrier(phone_number)
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
  
  # Custom error classes
  class VotingError < StandardError; end
  class VotingClosedError < VotingError; end
  class InvalidContestantError < VotingError; end
  class ContestantEliminatedError < VotingError; end
  class VoteLimitExceededError < VotingError; end
  class InsufficientVoteBalanceError < VotingError; end
  class PhoneNotVerifiedError < VotingError; end
  class SMSLimitExceededError < VotingError; end
  class InsufficientBundleVotesError < VotingError; end
end
```

### 2. Vote Package Management

```ruby
# app/services/xfactor/vote_package_service.rb
module Xfactor
  class VotePackageService
    def self.create_spree_products
      taxonomy = ensure_vote_taxonomy
      
      VotePackage.active.each do |package|
        create_or_update_product(package, taxonomy)
      end
    end
    
    private
    
    def self.ensure_vote_taxonomy
      taxonomy = Spree::Taxonomy.find_or_create_by(name: 'Digital Products')
      taxon = taxonomy.taxons.find_or_create_by(
        name: 'XFactor Votes',
        parent_id: taxonomy.root.id,
        permalink: 'digital-products/xfactor-votes'
      )
      taxon
    end
    
    def self.create_or_update_product(package, taxon)
      product = Spree::Product.find_or_initialize_by(sku: package.sku)
      
      total_votes = package.vote_count + package.bonus_votes
      
      product.assign_attributes(
        name: "#{package.name} - #{total_votes} Votes",
        description: build_description(package),
        price: package.price,
        currency: package.currency,
        available_on: Time.current,
        shipping_category: digital_shipping_category,
        tax_category: digital_tax_category,
        product_type: 'digital'
      )
      
      product.save!
      
      # Add to taxonomy
      product.taxons << taxon unless product.taxons.include?(taxon)
      
      # Add properties
      add_product_properties(product, package)
      
      # Add images if needed
      add_product_image(product, package) if product.images.empty?
      
      product
    end
    
    def self.build_description(package)
      desc = "Get #{package.vote_count} votes"
      desc += " + #{package.bonus_votes} BONUS votes" if package.bonus_votes > 0
      desc += " to support your favorite contestant!"
      desc += "\n\n#{package.badge_text}" if package.badge_text.present?
      desc
    end
    
    def self.digital_shipping_category
      Spree::ShippingCategory.find_or_create_by(name: 'Digital')
    end
    
    def self.digital_tax_category
      Spree::TaxCategory.find_or_create_by(name: 'Digital Goods')
    end
    
    def self.add_product_properties(product, package)
      properties = {
        'vote_count' => package.vote_count.to_s,
        'bonus_votes' => package.bonus_votes.to_s,
        'total_votes' => (package.vote_count + package.bonus_votes).to_s,
        'package_type' => 'vote_package'
      }
      
      properties.each do |name, value|
        property = Spree::Property.find_or_create_by(name: name)
        product_property = product.product_properties.find_or_initialize_by(property: property)
        product_property.update!(value: value)
      end
    end
    
    def self.add_product_image(product, package)
      # Add vote package image based on package size
      image_path = case package.vote_count
                   when 0..50 then 'vote_package_small.png'
                   when 51..200 then 'vote_package_medium.png'
                   when 201..1000 then 'vote_package_large.png'
                   else 'vote_package_ultimate.png'
                   end
      
      if File.exist?(Rails.root.join('app/assets/images', image_path))
        product.images.create!(
          attachment: File.open(Rails.root.join('app/assets/images', image_path)),
          alt: package.name
        )
      end
    end
  end
end

# app/models/spree/order_decorator.rb
module Spree
  module OrderDecorator
    def self.prepended(base)
      base.after_transition to: :complete do |order|
        order.process_vote_purchases
      end
    end
    
    def process_vote_purchases
      line_items.includes(:product).each do |line_item|
        next unless line_item.product.product_properties.exists?(
          property: Spree::Property.find_by(name: 'package_type'),
          value: 'vote_package'
        )
        
        process_vote_package_purchase(line_item)
      end
    end
    
    private
    
    def process_vote_package_purchase(line_item)
      product = line_item.product
      quantity = line_item.quantity
      
      # Get vote counts from product properties
      vote_count = product.property('vote_count').to_i * quantity
      bonus_votes = product.property('bonus_votes').to_i * quantity
      total_votes = vote_count + bonus_votes
      
      # Credit votes to user wallet
      wallet_service = Hangmeas::WalletService.new(user)
      wallet_service.purchase_vote_package(
        vote_count,
        bonus_votes,
        line_item.price * quantity
      )
      
      # Send confirmation
      VotePackagePurchaseMailer.confirmation(user, line_item, total_votes).deliver_later
      
      # Track analytics
      Analytics.track(
        user_id: user.id,
        event: 'Vote Package Purchased',
        properties: {
          package_name: product.name,
          vote_count: vote_count,
          bonus_votes: bonus_votes,
          total_votes: total_votes,
          amount: line_item.price * quantity,
          order_number: number
        }
      )
    end
  end
end

Spree::Order.prepend(Spree::OrderDecorator)
```

### 3. Real-time Voting with ActionCable

```ruby
# app/channels/voting_channel.rb
class VotingChannel < ApplicationCable::Channel
  def subscribed
    if params[:episode_id].present?
      episode = Xfactor::Episode.find(params[:episode_id])
      stream_from "voting_episode_#{episode.id}"
      
      # Send current results on subscribe
      transmit(type: 'initial_results', data: current_results(episode))
    end
    
    if params[:contestant_id].present?
      contestant = Xfactor::Contestant.find(params[:contestant_id])
      stream_from "voting_contestant_#{contestant.id}"
    end
  end
  
  def unsubscribed
    # Cleanup when channel is unsubscribed
  end
  
  def request_update(data)
    episode = Xfactor::Episode.find(data['episode_id'])
    transmit(type: 'results_update', data: current_results(episode))
  end
  
  private
  
  def current_results(episode)
    Rails.cache.fetch("episode_#{episode.id}_results", expires_in: 10.seconds) do
      {
        episode_id: episode.id,
        total_votes: episode.total_votes,
        voting_active: episode.voting_active?,
        contestants: episode.contestants.map do |contestant|
          votes = contestant.vote_count_for_episode(episode)
          {
            id: contestant.id,
            name: contestant.name,
            photo_url: contestant.photo_url,
            votes: votes,
            percentage: episode.total_votes > 0 ? (votes.to_f / episode.total_votes * 100).round(2) : 0,
            rank: nil # Will be calculated client-side
          }
        end,
        last_updated: Time.current
      }
    end
  end
end

# app/jobs/voting_results_broadcast_job.rb
class VotingResultsBroadcastJob < ApplicationJob
  queue_as :high_priority
  
  def perform(episode_id)
    episode = Xfactor::Episode.find(episode_id)
    
    # Broadcast to episode channel
    results = compile_results(episode)
    ActionCable.server.broadcast(
      "voting_episode_#{episode_id}",
      type: 'results_update',
      data: results
    )
    
    # Update cache
    Rails.cache.write(
      "episode_#{episode_id}_results",
      results,
      expires_in: 10.seconds
    )
    
    # Broadcast individual contestant updates
    results[:contestants].each do |contestant_data|
      ActionCable.server.broadcast(
        "voting_contestant_#{contestant_data[:id]}",
        type: 'vote_update',
        data: {
          episode_id: episode_id,
          votes: contestant_data[:votes],
          percentage: contestant_data[:percentage],
          rank: contestant_data[:rank]
        }
      )
    end
  end
  
  private
  
  def compile_results(episode)
    contestants_data = episode.contestants.includes(:votes).map do |contestant|
      votes = contestant.votes.where(episode: episode).sum(:vote_count)
      {
        id: contestant.id,
        name: contestant.name,
        khmer_name: contestant.khmer_name,
        photo_url: contestant.photo_url,
        hometown: contestant.hometown,
        votes: votes,
        percentage: 0, # Will be calculated after
        rank: 0 # Will be assigned after sorting
      }
    end
    
    # Sort by votes and assign ranks
    contestants_data.sort_by! { |c| -c[:votes] }
    total_votes = contestants_data.sum { |c| c[:votes] }
    
    contestants_data.each_with_index do |contestant, index|
      contestant[:rank] = index + 1
      contestant[:percentage] = total_votes > 0 ? (contestant[:votes].to_f / total_votes * 100).round(2) : 0
    end
    
    {
      episode_id: episode.id,
      episode_number: episode.episode_number,
      total_votes: total_votes,
      unique_voters: episode.votes.distinct.count(:user_id),
      voting_active: episode.voting_active?,
      voting_end: episode.voting_end,
      contestants: contestants_data,
      vote_breakdown: {
        free_votes: episode.votes.where(vote_type: 'free').sum(:vote_count),
        paid_votes: episode.votes.where(vote_type: 'paid').sum(:vote_count),
        sms_votes: episode.votes.where(vote_type: 'sms').sum(:vote_count)
      },
      last_updated: Time.current
    }
  end
end

# app/jobs/contestant_vote_update_job.rb
class ContestantVoteUpdateJob < ApplicationJob
  queue_as :high_priority
  
  def perform(contestant_id, episode_id)
    contestant = Xfactor::Contestant.find(contestant_id)
    episode = Xfactor::Episode.find(episode_id)
    
    vote_data = {
      contestant_id: contestant_id,
      episode_id: episode_id,
      total_votes: contestant.vote_count_for_episode(episode),
      percentage: contestant.percentage_for_episode(episode),
      recent_votes: recent_votes(contestant, episode),
      trending: calculate_trending(contestant, episode)
    }
    
    ActionCable.server.broadcast(
      "voting_contestant_#{contestant_id}",
      type: 'detailed_update',
      data: vote_data
    )
  end
  
  private
  
  def recent_votes(contestant, episode)
    contestant.votes
              .where(episode: episode)
              .order(created_at: :desc)
              .limit(10)
              .includes(:user)
              .map do |vote|
      {
        user_name: vote.user.display_name,
        vote_count: vote.vote_count,
        vote_type: vote.vote_type,
        time_ago: time_ago_in_words(vote.created_at)
      }
    end
  end
  
  def calculate_trending(contestant, episode)
    # Compare votes in last hour vs previous hour
    current_hour_votes = contestant.votes
                                  .where(episode: episode)
                                  .where('created_at >= ?', 1.hour.ago)
                                  .sum(:vote_count)
    
    previous_hour_votes = contestant.votes
                                   .where(episode: episode)
                                   .where('created_at >= ? AND created_at < ?', 2.hours.ago, 1.hour.ago)
                                   .sum(:vote_count)
    
    return 'stable' if previous_hour_votes == 0
    
    change_percentage = ((current_hour_votes - previous_hour_votes).to_f / previous_hour_votes * 100).round
    
    if change_percentage > 20
      'hot'
    elsif change_percentage > 0
      'rising'
    elsif change_percentage < -20
      'falling'
    else
      'stable'
    end
  end
end
```

### 4. Mobile API Controllers

```ruby
# app/controllers/api/v1/xfactor/voting_controller.rb
module Api
  module V1
    module Xfactor
      class VotingController < Api::V1::BaseController
        before_action :authenticate_user!
        before_action :set_episode, only: [:current_episode, :cast_vote, :results]
        before_action :set_contestant, only: [:cast_vote]
        
        # GET /api/v1/xfactor/voting/current_episode
        def current_episode
          if @episode
            render json: {
              success: true,
              data: {
                episode: episode_json(@episode),
                contestants: contestants_json(@episode),
                user_voting_status: user_voting_status(@episode),
                vote_packages: available_vote_packages
              }
            }
          else
            render json: {
              success: false,
              error: 'No active voting episode',
              next_episode: next_episode_info
            }, status: 404
          end
        end
        
        # POST /api/v1/xfactor/voting/cast_vote
        def cast_vote
          processor = VoteProcessor.new(
            user: current_user,
            contestant_id: params[:contestant_id],
            episode_id: @episode.id,
            vote_type: params[:vote_type],
            vote_count: params[:vote_count].to_i,
            metadata: {
              ip_address: request.remote_ip,
              user_agent: request.user_agent,
              app_version: request.headers['X-App-Version'],
              device_id: request.headers['X-Device-ID']
            }
          )
          
          if processor.process
            render json: {
              success: true,
              data: {
                vote_id: processor.vote.id,
                contestant: {
                  id: @contestant.id,
                  name: @contestant.name,
                  current_votes: @contestant.vote_count_for_episode(@episode)
                },
                user_status: {
                  votes_remaining: user_votes_remaining(@episode),
                  vote_balance: current_user.wallet.balance('VOTES'),
                  loyalty_points_earned: processor.loyalty_points_earned
                }
              }
            }
          else
            render json: {
              success: false,
              errors: processor.errors.full_messages
            }, status: 422
          end
        end
        
        # GET /api/v1/xfactor/voting/results/:episode_id
        def results
          render json: {
            success: true,
            data: {
              episode: {
                id: @episode.id,
                title: @episode.title,
                episode_number: @episode.episode_number,
                voting_active: @episode.voting_active?,
                total_votes: @episode.total_votes
              },
              results: voting_results(@episode),
              vote_breakdown: vote_breakdown(@episode),
              last_updated: Time.current
            }
          }
        end
        
        # GET /api/v1/xfactor/voting/history
        def history
          votes = current_user.votes
                             .includes(:contestant, :episode)
                             .order(created_at: :desc)
                             .page(params[:page])
                             .per(20)
          
          render json: {
            success: true,
            data: {
              votes: votes.map { |vote| vote_json(vote) },
              pagination: pagination_meta(votes)
            }
          }
        end
        
        # GET /api/v1/xfactor/voting/vote_balance
        def vote_balance
          render json: {
            success: true,
            data: {
              balance: current_user.wallet.balance('VOTES'),
              pending_votes: pending_sms_votes,
              recent_purchases: recent_vote_purchases,
              available_packages: available_vote_packages
            }
          }
        end
        
        private
        
        def set_episode
          @episode = if params[:episode_id]
                      Episode.find(params[:episode_id])
                    else
                      Episode.joins(:season)
                            .where(xfactor_seasons: { status: 'active' })
                            .where('voting_start <= ? AND voting_end >= ?', Time.current, Time.current)
                            .first
                    end
        end
        
        def set_contestant
          @contestant = Contestant.find(params[:contestant_id])
          
          unless @episode.contestants.include?(@contestant)
            render json: {
              success: false,
              error: 'Contestant not in this episode'
            }, status: 422
          end
        end
        
        def episode_json(episode)
          {
            id: episode.id,
            title: episode.title,
            episode_number: episode.episode_number,
            description: episode.description,
            air_date: episode.air_date,
            voting_start: episode.voting_start,
            voting_end: episode.voting_end,
            voting_active: episode.voting_active?,
            time_remaining: episode.voting_active? ? time_remaining(episode.voting_end) : nil,
            total_votes: episode.total_votes,
            thumbnail_url: episode.thumbnail_url,
            video_url: episode.video_url
          }
        end
        
        def contestants_json(episode)
          episode.contestants.map do |contestant|
            {
              id: contestant.id,
              name: contestant.name,
              khmer_name: contestant.khmer_name,
              bio: contestant.bio,
              hometown: contestant.hometown,
              age: contestant.age,
              photo_url: contestant.photo_url,
              status: contestant.status,
              votes: contestant.vote_count_for_episode(episode),
              percentage: contestant.percentage_for_episode(episode),
              performance: episode_contestant_performance(episode, contestant)
            }
          end
        end
        
        def episode_contestant_performance(episode, contestant)
          ec = EpisodeContestant.find_by(episode: episode, contestant: contestant)
          return nil unless ec
          
          {
            performance_order: ec.performance_order,
            song_performed: ec.song_performed,
            judges_score: ec.judges_score,
            performance_video_url: ec.performance_video_url
          }
        end
        
        def user_voting_status(episode)
          {
            has_voted: current_user.votes.where(episode: episode).exists?,
            free_votes_used: current_user.votes.where(episode: episode, vote_type: 'free').sum(:vote_count),
            free_votes_remaining: current_user.remaining_free_votes(episode),
            paid_votes_cast: current_user.votes.where(episode: episode, vote_type: 'paid').sum(:vote_count),
            total_votes_cast: current_user.votes.where(episode: episode).sum(:vote_count),
            vote_balance: current_user.wallet.balance('VOTES')
          }
        end
        
        def user_votes_remaining(episode)
          {
            free_votes: current_user.remaining_free_votes(episode),
            vote_balance: current_user.wallet.balance('VOTES'),
            sms_votes_available: true # Can always vote via SMS
          }
        end
        
        def voting_results(episode)
          contestants = episode.contestants.includes(:votes)
          
          results = contestants.map do |contestant|
            votes = contestant.votes.where(episode: episode).sum(:vote_count)
            {
              contestant_id: contestant.id,
              name: contestant.name,
              khmer_name: contestant.khmer_name,
              photo_url: contestant.photo_url,
              votes: votes,
              percentage: episode.total_votes > 0 ? (votes.to_f / episode.total_votes * 100).round(2) : 0
            }
          end
          
          results.sort_by { |r| -r[:votes] }
                .each_with_index { |r, i| r[:rank] = i + 1 }
        end
        
        def vote_breakdown(episode)
          {
            total_votes: episode.total_votes,
            unique_voters: episode.votes.distinct.count(:user_id),
            by_type: {
              free: episode.votes.where(vote_type: 'free').sum(:vote_count),
              paid: episode.votes.where(vote_type: 'paid').sum(:vote_count),
              sms: episode.votes.where(vote_type: 'sms').sum(:vote_count)
            },
            by_hour: votes_by_hour(episode)
          }
        end
        
        def votes_by_hour(episode)
          episode.votes
                 .where('created_at >= ?', 24.hours.ago)
                 .group_by_hour(:created_at)
                 .sum(:vote_count)
                 .map { |k, v| { hour: k, votes: v } }
        end
        
        def pending_sms_votes
          current_user.sms_vote_logs
                     .where(status: 'pending')
                     .count
        end
        
        def recent_vote_purchases
          current_user.orders
                     .complete
                     .joins(:line_items => :product)
                     .where('spree_product_properties.property_id = ? AND spree_product_properties.value = ?',
                           Spree::Property.find_by(name: 'package_type').id,
                           'vote_package')
                     .order(completed_at: :desc)
                     .limit(5)
                     .map do |order|
            {
              order_number: order.number,
              completed_at: order.completed_at,
              total_votes: order.line_items.sum do |li|
                li.product.property('total_votes').to_i * li.quantity
              end,
              amount: order.total
            }
          end
        end
        
        def available_vote_packages
          VotePackage.active.order(:sort_order).map do |package|
            {
              id: package.id,
              name: package.name,
              vote_count: package.vote_count,
              bonus_votes: package.bonus_votes,
              total_votes: package.vote_count + package.bonus_votes,
              price: package.price,
              currency: package.currency,
              badge_text: package.badge_text,
              spree_product_id: Spree::Product.find_by(sku: package.sku)&.id
            }
          end
        end
        
        def vote_json(vote)
          {
            id: vote.id,
            contestant: {
              id: vote.contestant.id,
              name: vote.contestant.name,
              photo_url: vote.contestant.photo_url
            },
            episode: {
              id: vote.episode.id,
              title: vote.episode.title,
              episode_number: vote.episode.episode_number
            },
            vote_type: vote.vote_type,
            vote_count: vote.vote_count,
            created_at: vote.created_at
          }
        end
        
        def next_episode_info
          next_episode = Episode.joins(:season)
                               .where(xfactor_seasons: { status: 'active' })
                               .where('voting_start > ?', Time.current)
                               .order(:voting_start)
                               .first
          
          return nil unless next_episode
          
          {
            id: next_episode.id,
            title: next_episode.title,
            air_date: next_episode.air_date,
            voting_start: next_episode.voting_start
          }
        end
        
        def time_remaining(end_time)
          seconds = (end_time - Time.current).to_i
          return 0 if seconds < 0
          
          {
            total_seconds: seconds,
            days: seconds / 86400,
            hours: (seconds % 86400) / 3600,
            minutes: (seconds % 3600) / 60,
            seconds: seconds % 60
          }
        end
        
        def pagination_meta(collection)
          {
            current_page: collection.current_page,
            total_pages: collection.total_pages,
            total_count: collection.total_count,
            per_page: collection.limit_value
          }
        end
      end
    end
  end
end
```

### 5. SMS Voting Integration

```ruby
# app/services/sms/voting_service.rb
module Sms
  class VotingService
    VOTE_PATTERNS = {
      simple: /^VOTE\s+(\d+)$/i,                    # VOTE 123
      multiple: /^VOTE\s+(\d+)\s+(\d+)$/i,          # VOTE 123 5 (5 votes)
      name: /^VOTE\s+(.+)$/i                        # VOTE John Doe
    }.freeze
    
    def self.process_incoming_sms(from_number, to_number, message, carrier_data = {})
      user = find_or_create_user(from_number)
      
      # Parse vote message
      vote_data = parse_vote_message(message)
      
      unless vote_data
        send_error_sms(from_number, "Invalid format. Send 'VOTE [contestant number]' to vote.")
        return false
      end
      
      # Find active episode
      episode = Episode.current_voting.first
      
      unless episode
        send_error_sms(from_number, "No active voting at this time.")
        return false
      end
      
      # Find contestant
      contestant = find_contestant(vote_data[:identifier], episode)
      
      unless contestant
        send_error_sms(from_number, "Contestant not found. Check the number and try again.")
        return false
      end
      
      # Process the vote
      processor = VoteProcessor.new(
        user: user,
        contestant_id: contestant.id,
        episode_id: episode.id,
        vote_type: 'sms',
        vote_count: vote_data[:count],
        metadata: {
          sms_from: from_number,
          sms_to: to_number,
          carrier: carrier_data[:carrier],
          message_id: carrier_data[:message_id]
        }
      )
      
      if processor.process
        send_confirmation_sms(
          from_number,
          "Thanks! Your #{vote_data[:count]} vote(s) for #{contestant.name} have been counted. " \
          "Charges apply. Reply STOP to unsubscribe."
        )
        
        # Update SMS log
        if carrier_data[:message_id]
          SmsVoteLog.where(carrier_message_id: carrier_data[:message_id])
                   .update(status: 'confirmed', confirmed_at: Time.current)
        end
        
        true
      else
        send_error_sms(from_number, processor.errors.full_messages.first)
        false
      end
    end
    
    private
    
    def self.find_or_create_user(phone_number)
      user = Spree::User.find_by(phone_number: normalize_phone(phone_number))
      
      unless user
        # Create basic user account
        user = Spree::User.create!(
          phone_number: normalize_phone(phone_number),
          email: "#{normalize_phone(phone_number)}@sms.hangmeas.com",
          password: SecureRandom.hex(16),
          phone_verified_at: Time.current
        )
        
        # Send welcome SMS
        send_welcome_sms(phone_number)
      end
      
      user
    end
    
    def self.normalize_phone(phone_number)
      # Remove country code if present
      phone_number.gsub(/^\+?855/, '0').gsub(/\D/, '')
    end
    
    def self.parse_vote_message(message)
      message = message.strip
      
      # Try each pattern
      VOTE_PATTERNS.each do |type, pattern|
        if match = message.match(pattern)
          case type
          when :simple
            return { identifier: match[1], count: 1 }
          when :multiple
            count = [match[2].to_i, 10].min # Max 10 votes per SMS
            return { identifier: match[1], count: count }
          when :name
            return { identifier: match[1], count: 1 }
          end
        end
      end
      
      nil
    end
    
    def self.find_contestant(identifier, episode)
      # Try by ID first
      if identifier =~ /^\d+$/
        contestant = episode.contestants.find_by(id: identifier)
        return contestant if contestant
      end
      
      # Try by name (fuzzy match)
      episode.contestants.find_by(
        "LOWER(name) LIKE ? OR LOWER(khmer_name) LIKE ?",
        "%#{identifier.downcase}%",
        "%#{identifier.downcase}%"
      )
    end
    
    def self.send_confirmation_sms(to_number, message)
      SmsGateway.send_sms(
        to: to_number,
        message: message,
        sender_id: 'XFACTOR'
      )
    end
    
    def self.send_error_sms(to_number, message)
      SmsGateway.send_sms(
        to: to_number,
        message: "XFactor: #{message}",
        sender_id: 'XFACTOR'
      )
    end
    
    def self.send_welcome_sms(phone_number)
      SmsGateway.send_sms(
        to: phone_number,
        message: "Welcome to XFactor voting! Your account has been created. " \
                "Visit hangmeas.com to complete your profile and get bonus votes!",
        sender_id: 'HANGMEAS'
      )
    end
  end
end

# app/controllers/webhooks/sms_controller.rb
module Webhooks
  class SmsController < ApplicationController
    skip_before_action :verify_authenticity_token
    
    # POST /webhooks/sms/smart
    def smart
      verify_smart_signature!
      
      result = Sms::VotingService.process_incoming_sms(
        params[:from],
        params[:to],
        params[:message],
        {
          carrier: 'smart',
          message_id: params[:message_id],
          timestamp: params[:timestamp]
        }
      )
      
      render json: { status: result ? 'processed' : 'failed' }
    end
    
    # POST /webhooks/sms/cellcard
    def cellcard
      verify_cellcard_token!
      
      result = Sms::VotingService.process_incoming_sms(
        params[:msisdn],
        params[:short_code],
        params[:text],
        {
          carrier: 'cellcard',
          message_id: params[:msg_id],
          timestamp: params[:received_at]
        }
      )
      
      render json: { success: result }
    end
    
    # POST /webhooks/sms/metfone
    def metfone
      verify_metfone_auth!
      
      result = Sms::VotingService.process_incoming_sms(
        params[:sender],
        params[:receiver],
        params[:content],
        {
          carrier: 'metfone',
          message_id: params[:sms_id],
          timestamp: params[:recv_time]
        }
      )
      
      render xml: { status: result ? 'OK' : 'ERROR' }
    end
    
    private
    
    def verify_smart_signature!
      # Implement Smart signature verification
      signature = request.headers['X-Smart-Signature']
      expected = OpenSSL::HMAC.hexdigest(
        'SHA256',
        Rails.application.credentials.smart_api_secret,
        request.raw_post
      )
      
      unless ActiveSupport::SecurityUtils.secure_compare(signature, expected)
        render json: { error: 'Invalid signature' }, status: :unauthorized
      end
    end
    
    def verify_cellcard_token!
      # Implement Cellcard token verification
      unless params[:api_token] == Rails.application.credentials.cellcard_api_token
        render json: { error: 'Invalid token' }, status: :unauthorized
      end
    end
    
    def verify_metfone_auth!
      # Implement Metfone auth verification
      auth_header = request.headers['Authorization']
      expected = "Bearer #{Rails.application.credentials.metfone_api_key}"
      
      unless auth_header == expected
        render xml: { error: 'Unauthorized' }, status: :unauthorized
      end
    end
  end
end
```

### 6. Vote Analytics Service

```ruby
# app/services/xfactor/vote_analytics_service.rb
module Xfactor
  class VoteAnalyticsService
    def initialize(episode)
      @episode = episode
    end
    
    def generate_report
      {
        summary: voting_summary,
        timeline: voting_timeline,
        demographics: voter_demographics,
        geographic: geographic_distribution,
        payment_methods: payment_method_breakdown,
        contestant_analysis: contestant_performance,
        predictions: voting_predictions
      }
    end
    
    private
    
    def voting_summary
      votes = @episode.votes
      
      {
        total_votes: votes.sum(:vote_count),
        unique_voters: votes.distinct.count(:user_id),
        average_votes_per_user: votes.sum(:vote_count).to_f / votes.distinct.count(:user_id),
        revenue: {
          sms: votes.where(vote_type: 'sms').sum(:amount_paid),
          paid_packages: calculate_package_revenue,
          total: calculate_total_revenue
        },
        engagement_rate: calculate_engagement_rate
      }
    end
    
    def voting_timeline
      # Hourly breakdown
      hourly = @episode.votes
                      .group_by_hour(:created_at)
                      .sum(:vote_count)
      
      # Key moments
      peak_hour = hourly.max_by { |_, v| v }
      
      {
        hourly_votes: hourly,
        peak_hour: peak_hour[0],
        peak_votes: peak_hour[1],
        voting_velocity: calculate_voting_velocity
      }
    end
    
    def voter_demographics
      users = Spree::User.joins(:votes)
                         .where(xfactor_votes: { episode_id: @episode.id })
                         .distinct
      
      {
        by_tier: users.group(:loyalty_tier).count,
        by_age_group: group_by_age(users),
        by_gender: users.group(:gender).count,
        new_vs_returning: {
          new: users.where('spree_users.created_at >= ?', @episode.voting_start).count,
          returning: users.where('spree_users.created_at < ?', @episode.voting_start).count
        },
        average_loyalty_points: users.average(:loyalty_points).to_i
      }
    end
    
    def geographic_distribution
      votes_by_province = Spree::User.joins(:votes)
                                    .where(xfactor_votes: { episode_id: @episode.id })
                                    .group(:province)
                                    .sum('xfactor_votes.vote_count')
      
      {
        by_province: votes_by_province,
        top_provinces: votes_by_province.sort_by { |_, v| -v }.first(5).to_h,
        urban_vs_rural: calculate_urban_rural_split(votes_by_province)
      }
    end
    
    def payment_method_breakdown
      {
        by_type: @episode.votes.group(:vote_type).sum(:vote_count),
        by_amount: @episode.votes.group(:vote_type).sum(:amount_paid),
        conversion_rate: calculate_paid_conversion_rate,
        average_purchase: calculate_average_purchase_value
      }
    end
    
    def contestant_performance
      @episode.contestants.map do |contestant|
        votes = contestant.votes.where(episode: @episode)
        
        {
          contestant_id: contestant.id,
          name: contestant.name,
          total_votes: votes.sum(:vote_count),
          unique_voters: votes.distinct.count(:user_id),
          vote_sources: votes.group(:vote_type).sum(:vote_count),
          hourly_trend: votes.group_by_hour(:created_at).sum(:vote_count),
          momentum_score: calculate_momentum(contestant),
          predicted_rank: predict_final_rank(contestant)
        }
      end
    end
    
    def voting_predictions
      return nil unless @episode.voting_active?
      
      remaining_hours = (@episode.voting_end - Time.current) / 3600.0
      current_velocity = calculate_voting_velocity
      
      {
        estimated_total_votes: @episode.total_votes + (current_velocity * remaining_hours).to_i,
        projected_winner: project_winner,
        confidence_level: calculate_prediction_confidence,
        estimated_revenue: project_revenue(remaining_hours)
      }
    end
    
    def calculate_package_revenue
      # Calculate revenue from vote package purchases during episode
      Spree::LineItem.joins(:order, :product)
                    .where(spree_orders: { state: 'complete' })
                    .where('spree_orders.completed_at BETWEEN ? AND ?', 
                          @episode.voting_start, 
                          @episode.voting_end || Time.current)
                    .where('spree_products.sku LIKE ?', 'VOTE_%')
                    .sum('spree_line_items.price * spree_line_items.quantity')
    end
    
    def calculate_total_revenue
      sms_revenue = @episode.votes.where(vote_type: 'sms').sum(:amount_paid)
      package_revenue = calculate_package_revenue
      
      sms_revenue + package_revenue
    end
    
    def calculate_engagement_rate
      total_users = Spree::User.where('created_at <= ?', @episode.voting_start).count
      voting_users = @episode.votes.distinct.count(:user_id)
      
      return 0 if total_users.zero?
      
      (voting_users.to_f / total_users * 100).round(2)
    end
    
    def calculate_voting_velocity
      # Votes per hour in the last 3 hours
      recent_votes = @episode.votes
                            .where('created_at >= ?', 3.hours.ago)
                            .sum(:vote_count)
      
      recent_votes / 3.0
    end
    
    def group_by_age(users)
      users.select('
        CASE
          WHEN EXTRACT(YEAR FROM AGE(date_of_birth)) < 18 THEN "Under 18"
          WHEN EXTRACT(YEAR FROM AGE(date_of_birth)) BETWEEN 18 AND 24 THEN "18-24"
          WHEN EXTRACT(YEAR FROM AGE(date_of_birth)) BETWEEN 25 AND 34 THEN "25-34"
          WHEN EXTRACT(YEAR FROM AGE(date_of_birth)) BETWEEN 35 AND 44 THEN "35-44"
          ELSE "45+"
        END as age_group,
        COUNT(*) as count
      ').group('age_group')
    end
    
    def calculate_urban_rural_split(votes_by_province)
      urban_provinces = ['Phnom Penh', 'Siem Reap', 'Battambang', 'Sihanoukville']
      
      urban_votes = votes_by_province.select { |p, _| urban_provinces.include?(p) }.values.sum
      rural_votes = votes_by_province.values.sum - urban_votes
      
      {
        urban: urban_votes,
        rural: rural_votes,
        urban_percentage: (urban_votes.to_f / votes_by_province.values.sum * 100).round(2)
      }
    end
    
    def calculate_paid_conversion_rate
      total_users = @episode.votes.distinct.count(:user_id)
      paid_users = @episode.votes.where(vote_type: ['paid', 'sms']).distinct.count(:user_id)
      
      return 0 if total_users.zero?
      
      (paid_users.to_f / total_users * 100).round(2)
    end
    
    def calculate_average_purchase_value
      paid_votes = @episode.votes.where(vote_type: ['paid', 'sms'])
      
      return 0 if paid_votes.empty?
      
      paid_votes.sum(:amount_paid) / paid_votes.distinct.count(:user_id)
    end
    
    def calculate_momentum(contestant)
      # Compare last hour vs previous hour
      last_hour = contestant.votes
                           .where(episode: @episode)
                           .where('created_at >= ?', 1.hour.ago)
                           .sum(:vote_count)
      
      previous_hour = contestant.votes
                               .where(episode: @episode)
                               .where('created_at >= ? AND created_at < ?', 2.hours.ago, 1.hour.ago)
                               .sum(:vote_count)
      
      return 0 if previous_hour.zero?
      
      ((last_hour - previous_hour).to_f / previous_hour * 100).round(2)
    end
    
    def predict_final_rank(contestant)
      # Simple prediction based on current votes and momentum
      current_rank = @episode.contestants
                            .sort_by { |c| -c.vote_count_for_episode(@episode) }
                            .index(contestant) + 1
      
      momentum = calculate_momentum(contestant)
      
      if momentum > 50
        [current_rank - 1, 1].max
      elsif momentum < -50
        [current_rank + 1, @episode.contestants.count].min
      else
        current_rank
      end
    end
    
    def project_winner
      rankings = @episode.contestants.map do |contestant|
        {
          contestant_id: contestant.id,
          name: contestant.name,
          current_votes: contestant.vote_count_for_episode(@episode),
          momentum: calculate_momentum(contestant),
          projected_votes: project_final_votes(contestant)
        }
      end
      
      rankings.max_by { |r| r[:projected_votes] }
    end
    
    def project_final_votes(contestant)
      current_votes = contestant.vote_count_for_episode(@episode)
      momentum = calculate_momentum(contestant) / 100.0
      remaining_hours = (@episode.voting_end - Time.current) / 3600.0
      
      hourly_rate = contestant.votes
                             .where(episode: @episode)
                             .where('created_at >= ?', 1.hour.ago)
                             .sum(:vote_count)
      
      projected_additional = (hourly_rate * (1 + momentum) * remaining_hours).to_i
      
      current_votes + projected_additional
    end
    
    def calculate_prediction_confidence
      # Based on time remaining and vote volatility
      time_elapsed = (Time.current - @episode.voting_start) / (@episode.voting_end - @episode.voting_start)
      volatility = calculate_vote_volatility
      
      base_confidence = time_elapsed * 100
      confidence_penalty = volatility * 10
      
      [base_confidence - confidence_penalty, 0].max.round
    end
    
    def calculate_vote_volatility
      # Standard deviation of hourly votes
      hourly_votes = @episode.votes
                            .group_by_hour(:created_at)
                            .sum(:vote_count)
                            .values
      
      return 0 if hourly_votes.empty?
      
      mean = hourly_votes.sum.to_f / hourly_votes.size
      variance = hourly_votes.sum { |v| (v - mean) ** 2 } / hourly_votes.size
      
      Math.sqrt(variance) / mean
    end
    
    def project_revenue(remaining_hours)
      current_revenue = calculate_total_revenue
      hourly_revenue = current_revenue / ((Time.current - @episode.voting_start) / 3600.0)
      
      current_revenue + (hourly_revenue * remaining_hours * 0.8) # 0.8 factor for declining rate
    end
  end
end
```

### 7. Testing Suite

```ruby
# spec/services/xfactor/vote_processor_spec.rb
require 'rails_helper'

RSpec.describe Xfactor::VoteProcessor do
  let(:user) { create(:user, :verified) }
  let(:episode) { create(:xfactor_episode, :with_active_voting) }
  let(:contestant) { create(:xfactor_contestant) }
  
  before do
    create(:episode_contestant, episode: episode, contestant: contestant)
  end
  
  describe '#process' do
    context 'with free votes' do
      let(:processor) do
        described_class.new(
          user: user,
          contestant_id: contestant.id,
          episode_id: episode.id,
          vote_type: 'free',
          vote_count: 3
        )
      end
      
      it 'creates vote record' do
        expect { processor.process }.to change { Xfactor::Vote.count }.by(1)
      end
      
      it 'respects free vote limit' do
        # Use up free votes
        create(:xfactor_vote, user: user, episode: episode, vote_type: 'free', vote_count: 4)
        
        processor.vote_count = 2
        expect(processor.process).to be_falsey
        expect(processor.errors[:base]).to include(/can only cast 1 more free vote/)
      end
      
      it 'awards loyalty points' do
        expect(user.loyalty).to receive(:award_points).with(6, anything)
        processor.process
      end
    end
    
    context 'with paid votes' do
      let(:processor) do
        described_class.new(
          user: user,
          contestant_id: contestant.id,
          episode_id: episode.id,
          vote_type: 'paid',
          vote_count: 10
        )
      end
      
      before do
        # Add vote balance
        create(:spree_store_credit, user: user, amount: 20, currency: 'VOTES')
      end
      
      it 'deducts from vote balance' do
        expect { processor.process }.to change { user.wallet.balance('VOTES') }.by(-10)
      end
      
      it 'fails with insufficient balance' do
        processor.vote_count = 25
        expect(processor.process).to be_falsey
        expect(processor.errors[:base]).to include(/Insufficient vote balance/)
      end
    end
    
    context 'with SMS votes' do
      let(:processor) do
        described_class.new(
          user: user,
          contestant_id: contestant.id,
          episode_id: episode.id,
          vote_type: 'sms',
          vote_count: 1
        )
      end
      
      it 'creates SMS log' do
        expect { processor.process }.to change { SmsVoteLog.count }.by(1)
      end
      
      it 'requires phone verification' do
        user.update!(phone_verified_at: nil)
        expect(processor.process).to be_falsey
        expect(processor.errors[:base]).to include(/Phone number must be verified/)
      end
    end
    
    context 'voting validation' do
      it 'prevents voting after deadline' do
        episode.update!(voting_end: 1.hour.ago)
        
        processor = described_class.new(
          user: user,
          contestant_id: contestant.id,
          episode_id: episode.id,
          vote_type: 'free',
          vote_count: 1
        )
        
        expect(processor.process).to be_falsey
        expect(processor.errors[:base]).to include(/Voting is not currently active/)
      end
    end
  end
end

# spec/requests/api/v1/xfactor/voting_spec.rb
require 'rails_helper'

RSpec.describe 'XFactor Voting API', type: :request do
  let(:user) { create(:user, :verified) }
  let(:episode) { create(:xfactor_episode, :with_active_voting) }
  let(:contestant) { create(:xfactor_contestant) }
  let(:auth_headers) { { 'Authorization' => "Bearer #{user.generate_jwt}" } }
  
  before do
    create(:episode_contestant, episode: episode, contestant: contestant)
  end
  
  describe 'GET /api/v1/xfactor/voting/current_episode' do
    it 'returns current voting episode' do
      get '/api/v1/xfactor/voting/current_episode', headers: auth_headers
      
      expect(response).to have_http_status(:success)
      expect(json_response['data']['episode']['id']).to eq(episode.id)
      expect(json_response['data']['contestants']).to be_present
    end
  end
  
  describe 'POST /api/v1/xfactor/voting/cast_vote' do
    it 'casts a free vote' do
      post '/api/v1/xfactor/voting/cast_vote', params: {
        contestant_id: contestant.id,
        vote_type: 'free',
        vote_count: 1
      }, headers: auth_headers
      
      expect(response).to have_http_status(:success)
      expect(json_response['success']).to be_truthy
      expect(contestant.reload.total_votes_received).to eq(1)
    end
    
    it 'handles voting errors' do
      post '/api/v1/xfactor/voting/cast_vote', params: {
        contestant_id: contestant.id,
        vote_type: 'free',
        vote_count: 10 # Exceeds limit
      }, headers: auth_headers
      
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json_response['errors']).to be_present
    end
  end
end
```

---

## 📊 Performance Considerations

### 1. Caching Strategy
```ruby
# config/initializers/voting_cache.rb
Rails.application.configure do
  config.cache_store = :redis_cache_store, {
    url: ENV['REDIS_URL'],
    namespace: 'voting',
    expires_in: 1.hour,
    race_condition_ttl: 5.seconds
  }
end
```

### 2. Database Indexes
```sql
-- Optimize vote queries
CREATE INDEX idx_votes_episode_contestant ON xfactor_votes(episode_id, contestant_id);
CREATE INDEX idx_votes_user_episode_type ON xfactor_votes(user_id, episode_id, vote_type);
CREATE INDEX idx_votes_created_at ON xfactor_votes(created_at);

-- Optimize real-time queries
CREATE INDEX idx_votes_episode_created ON xfactor_votes(episode_id, created_at DESC);
```

### 3. Background Job Priorities
```ruby
# config/sidekiq.yml
:queues:
  - [critical, 10]      # Payment processing
  - [high_priority, 5]   # Vote broadcasting
  - [default, 3]         # General processing
  - [low, 1]            # Analytics, reports
```

---

## 🚀 Deployment Checklist

- [ ] Redis configured for ActionCable
- [ ] WebSocket endpoints configured in nginx/Apache
- [ ] SSL certificates for WebSocket connections
- [ ] SMS gateway credentials configured
- [ ] Background job workers running
- [ ] Database indexes created
- [ ] Caching layer configured
- [ ] Rate limiting configured
- [ ] Monitoring and alerting set up
- [ ] Load testing completed

---

## 📈 Monitoring & Analytics

### Key Metrics to Track
1. **Voting Performance**
   - Votes per second
   - WebSocket connection count
   - Vote processing latency
   - Failed vote attempts

2. **Revenue Metrics**
   - Vote package conversion rate
   - Average revenue per user
   - SMS vote revenue
   - Payment failure rate

3. **System Health**
   - API response times
   - Background job queue depth
   - Database query performance
   - Cache hit rate