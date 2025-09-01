# 📱 Week 7-8: Mobile API Implementation

## 🎯 Overview
This phase focuses on creating comprehensive mobile API endpoints for the Hangmeas Super App, including authentication, real-time features, push notifications, and mobile-optimized data delivery.

---

## 📋 Development Checklist

### Week 7: Authentication & Core APIs

#### Day 1-2: Authentication System
- [ ] Implement JWT token authentication
- [ ] Create mobile login/register endpoints
- [ ] Set up phone number verification
- [ ] Implement biometric authentication support
- [ ] Create session management

#### Day 3-4: User Profile & Wallet APIs
- [ ] Create user profile management endpoints
- [ ] Implement wallet APIs (balance, topup, transfer)
- [ ] Create transaction history endpoints
- [ ] Add KYC verification APIs
- [ ] Implement loyalty system APIs

#### Day 5: Core Content APIs
- [ ] Create product catalog APIs
- [ ] Implement event listing endpoints
- [ ] Add XFactor season/episode APIs
- [ ] Create search and filtering APIs
- [ ] Implement pagination and caching

### Week 8: Advanced Features & Push Notifications

#### Day 1-2: Real-time Updates
- [ ] Set up push notification service
- [ ] Implement WebSocket API endpoints
- [ ] Create notification management APIs
- [ ] Add real-time voting updates
- [ ] Implement live event features

#### Day 3-4: Digital Marketplace Integration
- [ ] Create cart management APIs
- [ ] Implement checkout flow APIs
- [ ] Add order management endpoints
- [ ] Create payment integration APIs
- [ ] Implement subscription management APIs

#### Day 5: API Documentation & Testing
- [ ] Generate comprehensive API documentation
- [ ] Create API testing suite
- [ ] Implement rate limiting
- [ ] Add API versioning
- [ ] Performance optimization

---

## 🔨 Implementation Details

### 1. Authentication System

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      include ActionController::API
      include ExceptionHandler
      include Authenticable
      
      before_action :authenticate_request
      before_action :set_current_user
      before_action :update_last_active
      
      protected
      
      def current_user
        @current_user
      end
      
      def authenticate_request
        @current_user = AuthorizeApiRequest.call(request.headers).result
        render json: { error: 'Not Authorized' }, status: 401 unless @current_user
      end
      
      def set_current_user
        Current.user = @current_user if @current_user
      end
      
      def update_last_active
        @current_user&.touch(:last_active_at)
      end
      
      def paginate(collection, per_page: 20)
        page = params[:page]&.to_i || 1
        per_page = [params[:per_page]&.to_i || per_page, 100].min
        
        collection.page(page).per(per_page)
      end
      
      def pagination_meta(collection)
        {
          current_page: collection.current_page,
          total_pages: collection.total_pages,
          total_count: collection.total_count,
          per_page: collection.limit_value
        }
      end
      
      def success_response(data = {}, message = nil, status = :ok)
        response = { success: true }
        response[:message] = message if message
        response[:data] = data if data.present?
        
        render json: response, status: status
      end
      
      def error_response(message, status = :bad_request, errors = nil)
        response = { success: false, message: message }
        response[:errors] = errors if errors
        
        render json: response, status: status
      end
    end
  end
end

# app/services/authorize_api_request.rb
class AuthorizeApiRequest
  def initialize(headers = {})
    @headers = headers
  end
  
  def call
    { result: user }
  end
  
  private
  
  attr_reader :headers
  
  def user
    if decoded_auth_token
      @user ||= Spree::User.find(decoded_auth_token[:user_id])
    end
  rescue JWT::DecodeError => e
    nil
  end
  
  def decoded_auth_token
    @decoded_auth_token ||= JsonWebToken.decode(http_auth_header)
  end
  
  def http_auth_header
    if headers['Authorization'].present?
      return headers['Authorization'].split(' ').last
    end
  end
end

# app/services/json_web_token.rb
class JsonWebToken
  SECRET_KEY = Rails.application.credentials.secret_key_base
  
  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET_KEY)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY)[0]
    HashWithIndifferentAccess.new decoded
  end
end

# app/controllers/api/v1/auth_controller.rb
module Api
  module V1
    class AuthController < ApplicationController
      include ActionController::API
      
      # POST /api/v1/auth/login
      def login
        user = authenticate_user
        
        if user
          token = JsonWebToken.encode(user_id: user.id)
          
          # Update device info
          update_device_info(user)
          
          render json: {
            success: true,
            token: token,
            user: user_profile(user),
            expires_at: 24.hours.from_now
          }
        else
          render json: {
            success: false,
            message: 'Invalid credentials'
          }, status: :unauthorized
        end
      end
      
      # POST /api/v1/auth/register
      def register
        user = Spree::User.new(user_params)
        user.password = params[:password]
        
        if user.save
          # Send verification SMS
          send_verification_sms(user) if user.phone_number.present?
          
          # Generate referral code
          user.update!(referral_code: generate_referral_code)
          
          # Apply referral if provided
          apply_referral(user, params[:referral_code]) if params[:referral_code].present?
          
          token = JsonWebToken.encode(user_id: user.id)
          
          render json: {
            success: true,
            message: 'Registration successful',
            token: token,
            user: user_profile(user),
            verification_required: user.phone_number.present?
          }, status: :created
        else
          render json: {
            success: false,
            message: 'Registration failed',
            errors: user.errors.full_messages
          }, status: :unprocessable_entity
        end
      end
      
      # POST /api/v1/auth/verify_phone
      def verify_phone
        user = Spree::User.find_by(phone_number: params[:phone_number])
        
        unless user
          return render json: {
            success: false,
            message: 'User not found'
          }, status: :not_found
        end
        
        verification = PhoneVerification.find_by(
          phone_number: params[:phone_number],
          code: params[:verification_code],
          expires_at: Time.current..
        )
        
        if verification
          user.update!(
            phone_verified_at: Time.current,
            verification_level: 'phone_verified'
          )
          
          verification.update!(verified_at: Time.current)
          
          # Award welcome bonus
          user.loyalty.award_points(100, 'Phone verification bonus')
          
          render json: {
            success: true,
            message: 'Phone verified successfully',
            user: user_profile(user)
          }
        else
          render json: {
            success: false,
            message: 'Invalid or expired verification code'
          }, status: :unprocessable_entity
        end
      end
      
      # POST /api/v1/auth/resend_verification
      def resend_verification
        user = Spree::User.find_by(phone_number: params[:phone_number])
        
        unless user
          return render json: {
            success: false,
            message: 'User not found'
          }, status: :not_found
        end
        
        if user.phone_verified_at.present?
          return render json: {
            success: false,
            message: 'Phone already verified'
          }, status: :bad_request
        end
        
        send_verification_sms(user)
        
        render json: {
          success: true,
          message: 'Verification code sent'
        }
      end
      
      # POST /api/v1/auth/refresh
      def refresh
        auth_header = request.headers['Authorization']
        token = auth_header.split(' ').last if auth_header
        
        begin
          decoded = JsonWebToken.decode(token)
          user = Spree::User.find(decoded[:user_id])
          
          new_token = JsonWebToken.encode(user_id: user.id)
          
          render json: {
            success: true,
            token: new_token,
            expires_at: 24.hours.from_now
          }
        rescue JWT::ExpiredSignature
          render json: {
            success: false,
            message: 'Token expired'
          }, status: :unauthorized
        rescue JWT::DecodeError
          render json: {
            success: false,
            message: 'Invalid token'
          }, status: :unauthorized
        end
      end
      
      # POST /api/v1/auth/logout
      def logout
        # Invalidate push token
        if current_user&.push_token == params[:push_token]
          current_user.update!(push_token: nil)
        end
        
        render json: {
          success: true,
          message: 'Logged out successfully'
        }
      end
      
      # POST /api/v1/auth/forgot_password
      def forgot_password
        user = Spree::User.find_by(email: params[:email]) || 
               Spree::User.find_by(phone_number: params[:phone_number])
        
        if user
          # Generate reset token
          reset_token = SecureRandom.hex(3) # 6 digit code
          user.update!(
            reset_password_token: Devise.token_generator.digest(self, :reset_password_token, reset_token),
            reset_password_sent_at: Time.current
          )
          
          if params[:phone_number].present?
            send_password_reset_sms(user, reset_token)
          else
            UserMailer.password_reset(user, reset_token).deliver_later
          end
        end
        
        # Always return success to prevent enumeration
        render json: {
          success: true,
          message: 'Password reset instructions sent'
        }
      end
      
      # POST /api/v1/auth/reset_password
      def reset_password
        user = Spree::User.find_by(
          reset_password_token: Devise.token_generator.digest(self, :reset_password_token, params[:reset_token])
        )
        
        unless user && user.reset_password_sent_at > 15.minutes.ago
          return render json: {
            success: false,
            message: 'Invalid or expired reset token'
          }, status: :unprocessable_entity
        end
        
        if user.update(password: params[:password], reset_password_token: nil, reset_password_sent_at: nil)
          render json: {
            success: true,
            message: 'Password reset successfully'
          }
        else
          render json: {
            success: false,
            message: 'Password reset failed',
            errors: user.errors.full_messages
          }, status: :unprocessable_entity
        end
      end
      
      private
      
      def authenticate_user
        if params[:email].present?
          user = Spree::User.find_by(email: params[:email])
        elsif params[:phone_number].present?
          user = Spree::User.find_by(phone_number: params[:phone_number])
        end
        
        user&.authenticate(params[:password])
      end
      
      def user_params
        params.permit(
          :email, :phone_number, :first_name, :last_name, :khmer_name,
          :date_of_birth, :gender, :province, :language_preference
        )
      end
      
      def user_profile(user)
        {
          id: user.id,
          email: user.email,
          phone_number: user.phone_number,
          first_name: user.first_name,
          last_name: user.last_name,
          khmer_name: user.khmer_name,
          display_name: user.display_name,
          loyalty_tier: user.loyalty_tier,
          loyalty_points: user.loyalty_points,
          phone_verified: user.phone_verified_at.present?,
          kyc_status: user.kyc_status,
          profile_completion: user.profile_completion_percentage,
          referral_code: user.referral_code,
          created_at: user.created_at
        }
      end
      
      def update_device_info(user)
        user.update!(
          push_token: params[:push_token],
          device_type: params[:device_type],
          app_version: params[:app_version],
          last_active_at: Time.current
        )
      end
      
      def send_verification_sms(user)
        code = sprintf('%06d', rand(100000..999999))
        
        PhoneVerification.create!(
          phone_number: user.phone_number,
          code: code,
          expires_at: 10.minutes.from_now
        )
        
        SmsGateway.send_sms(
          to: user.phone_number,
          message: "Your Hangmeas verification code is: #{code}. Valid for 10 minutes.",
          sender_id: 'HANGMEAS'
        )
      end
      
      def send_password_reset_sms(user, token)
        SmsGateway.send_sms(
          to: user.phone_number,
          message: "Your Hangmeas password reset code is: #{token}. Valid for 15 minutes.",
          sender_id: 'HANGMEAS'
        )
      end
      
      def generate_referral_code
        loop do
          code = "HM#{SecureRandom.alphanumeric(6).upcase}"
          break code unless Spree::User.exists?(referral_code: code)
        end
      end
      
      def apply_referral(user, referral_code)
        referrer = Spree::User.find_by(referral_code: referral_code)
        return unless referrer
        
        user.update!(referred_by_id: referrer.id)
        
        # Award referral bonuses
        ReferralBonusJob.perform_later(referrer.id, user.id)
      end
    end
  end
end
```

### 2. User Profile & Wallet APIs

```ruby
# app/controllers/api/v1/profile_controller.rb
module Api
  module V1
    class ProfileController < BaseController
      # GET /api/v1/profile
      def show
        render json: {
          success: true,
          data: detailed_user_profile
        }
      end
      
      # PUT /api/v1/profile
      def update
        if current_user.update(profile_params)
          # Award completion bonus if profile just completed
          if current_user.profile_completion_percentage == 100 && 
             current_user.loyalty_activities.where(reason: 'Profile completion').empty?
            current_user.loyalty.award_points(200, 'Profile completion')
          end
          
          render json: {
            success: true,
            message: 'Profile updated successfully',
            data: detailed_user_profile
          }
        else
          error_response('Profile update failed', :unprocessable_entity, current_user.errors.full_messages)
        end
      end
      
      # POST /api/v1/profile/avatar
      def upload_avatar
        if current_user.avatar.attach(params[:avatar])
          render json: {
            success: true,
            message: 'Avatar uploaded successfully',
            avatar_url: rails_blob_url(current_user.avatar)
          }
        else
          error_response('Avatar upload failed')
        end
      end
      
      # PUT /api/v1/profile/push_token
      def update_push_token
        current_user.update!(
          push_token: params[:push_token],
          device_type: params[:device_type]
        )
        
        success_response(message: 'Push token updated')
      end
      
      # PUT /api/v1/profile/preferences
      def update_preferences
        preferences = current_user.notification_preferences.merge(
          notification_params.to_h
        )
        
        current_user.update!(notification_preferences: preferences)
        
        success_response(
          { preferences: current_user.notification_preferences },
          'Preferences updated'
        )
      end
      
      # POST /api/v1/profile/kyc
      def submit_kyc
        kyc_submission = current_user.kyc_submissions.create!(
          id_card_number: params[:id_card_number],
          id_card_front_image: params[:id_card_front],
          id_card_back_image: params[:id_card_back],
          selfie_image: params[:selfie],
          status: 'pending'
        )
        
        # Queue KYC processing
        KycProcessingJob.perform_later(kyc_submission.id)
        
        success_response(
          { submission_id: kyc_submission.id },
          'KYC documents submitted for review'
        )
      end
      
      # GET /api/v1/profile/referrals
      def referrals
        referred_users = Spree::User.where(referred_by_id: current_user.id)
                                   .select(:id, :first_name, :last_name, :khmer_name, :created_at, :loyalty_tier)
        
        rewards = current_user.referral_rewards
                             .includes(:referred)
                             .order(created_at: :desc)
        
        render json: {
          success: true,
          data: {
            referral_code: current_user.referral_code,
            total_referrals: referred_users.count,
            total_rewards: rewards.where(status: 'paid').sum(:reward_amount),
            referred_users: referred_users.map do |user|
              {
                id: user.id,
                name: user.display_name,
                joined_at: user.created_at,
                tier: user.loyalty_tier
              }
            end,
            recent_rewards: rewards.limit(10).map do |reward|
              {
                referred_user: reward.referred.display_name,
                reward_type: reward.reward_type,
                amount: reward.reward_amount,
                status: reward.status,
                earned_at: reward.paid_at || reward.qualified_at
              }
            end
          }
        }
      end
      
      private
      
      def detailed_user_profile
        {
          id: current_user.id,
          email: current_user.email,
          phone_number: current_user.phone_number,
          first_name: current_user.first_name,
          last_name: current_user.last_name,
          khmer_name: current_user.khmer_name,
          display_name: current_user.display_name,
          date_of_birth: current_user.date_of_birth,
          gender: current_user.gender,
          province: current_user.province,
          district: current_user.district,
          language_preference: current_user.language_preference,
          loyalty_tier: current_user.loyalty_tier,
          loyalty_points: current_user.loyalty_points,
          total_spent: current_user.total_spent,
          phone_verified: current_user.phone_verified_at.present?,
          kyc_status: current_user.kyc_status,
          verification_level: current_user.verification_level,
          profile_completion: current_user.profile_completion_percentage,
          referral_code: current_user.referral_code,
          referral_count: current_user.referral_count,
          tier_benefits: current_user.tier_benefits,
          next_tier_progress: current_user.next_tier_progress,
          avatar_url: current_user.avatar.attached? ? rails_blob_url(current_user.avatar) : nil,
          notification_preferences: current_user.notification_preferences,
          created_at: current_user.created_at,
          last_active_at: current_user.last_active_at
        }
      end
      
      def profile_params
        params.permit(
          :first_name, :last_name, :khmer_name, :date_of_birth, :gender,
          :province, :district, :commune, :village, :language_preference
        )
      end
      
      def notification_params
        params.permit(
          :push_enabled, :email_enabled, :sms_enabled,
          :marketing_enabled, :voting_updates, :order_updates,
          :event_reminders, :loyalty_updates
        )
      end
    end
  end
end

# app/controllers/api/v1/wallet_controller.rb
module Api
  module V1
    class WalletController < BaseController
      # GET /api/v1/wallet/balance
      def balance
        wallet = current_user.wallet
        
        render json: {
          success: true,
          data: {
            balances: wallet.all_balances,
            formatted_balances: wallet.formatted_balances,
            primary_currency: 'USD',
            last_updated: Time.current
          }
        }
      end
      
      # POST /api/v1/wallet/topup
      def topup
        unless params[:amount].present? && params[:currency].present? && params[:payment_method].present?
          return error_response('Missing required parameters')
        end
        
        amount = params[:amount].to_f
        currency = params[:currency]
        payment_method = params[:payment_method]
        
        # Validate amount
        if amount <= 0
          return error_response('Invalid amount')
        end
        
        # Validate currency
        unless Spree::StoreCredit::WALLET_CURRENCIES.include?(currency)
          return error_response('Unsupported currency')
        end
        
        begin
          case payment_method
          when 'aba_pay'
            result = process_aba_payment(amount, currency)
          when 'wing'
            result = process_wing_payment(amount, currency)
          when 'card'
            result = process_card_payment(amount, currency)
          else
            return error_response('Unsupported payment method')
          end
          
          if result[:success]
            # Create topup record
            credit = current_user.wallet.topup(
              amount,
              currency,
              payment_method,
              result[:transaction_id],
              { gateway_response: result[:response] }
            )
            
            success_response(
              {
                transaction_id: credit.reference_number,
                amount: amount,
                currency: currency,
                new_balance: current_user.wallet.balance(currency)
              },
              'Topup successful'
            )
          else
            error_response(result[:message] || 'Payment failed')
          end
        rescue StandardError => e
          error_response('Topup failed', :unprocessable_entity, [e.message])
        end
      end
      
      # POST /api/v1/wallet/transfer
      def transfer
        recipient = Spree::User.find_by(
          phone_number: params[:recipient_phone]
        ) || Spree::User.find_by(
          referral_code: params[:recipient_code]
        )
        
        unless recipient
          return error_response('Recipient not found')
        end
        
        if recipient == current_user
          return error_response('Cannot transfer to yourself')
        end
        
        amount = params[:amount].to_f
        currency = params[:currency] || 'USD'
        notes = params[:notes]
        
        begin
          transfers = current_user.wallet.transfer_to(recipient, amount, currency, notes)
          
          success_response(
            {
              transaction_id: transfers.first.reference_number,
              recipient: {
                name: recipient.display_name,
                phone: recipient.phone_number
              },
              amount: amount,
              currency: currency,
              new_balance: current_user.wallet.balance(currency)
            },
            'Transfer successful'
          )
        rescue StandardError => e
          error_response(e.message, :unprocessable_entity)
        end
      end
      
      # GET /api/v1/wallet/transactions
      def transactions
        page = params[:page]&.to_i || 1
        per_page = [params[:per_page]&.to_i || 20, 100].min
        
        transactions = current_user.wallet.transaction_history(
          currency: params[:currency],
          type: params[:type],
          from_date: params[:from_date],
          to_date: params[:to_date],
          limit: per_page * 2 # Get more for pagination
        ).page(page).per(per_page)
        
        render json: {
          success: true,
          data: {
            transactions: transactions.map { |tx| format_transaction(tx) },
            pagination: pagination_meta(transactions)
          }
        }
      end
      
      # GET /api/v1/wallet/exchange_rates
      def exchange_rates
        rates = CurrencyService.current_rates
        
        render json: {
          success: true,
          data: {
            rates: rates,
            base_currency: 'USD',
            last_updated: rates[:updated_at]
          }
        }
      end
      
      # POST /api/v1/wallet/convert
      def convert
        from_amount = params[:from_amount].to_f
        from_currency = params[:from_currency]
        to_currency = params[:to_currency]
        
        if current_user.wallet.balance(from_currency) < from_amount
          return error_response('Insufficient balance')
        end
        
        begin
          converted_amount = current_user.wallet.convert_currency(
            from_amount, from_currency, to_currency
          )
          
          # Perform the conversion
          ActiveRecord::Base.transaction do
            # Deduct from source currency
            Spree::StoreCredit.create!(
              user: current_user,
              amount: -from_amount,
              currency: from_currency,
              transaction_type: 'conversion',
              reason: "Convert to #{to_currency}"
            )
            
            # Add to target currency
            Spree::StoreCredit.create!(
              user: current_user,
              amount: converted_amount,
              currency: to_currency,
              transaction_type: 'conversion',
              reason: "Convert from #{from_currency}"
            )
          end
          
          success_response(
            {
              from_amount: from_amount,
              from_currency: from_currency,
              to_amount: converted_amount,
              to_currency: to_currency,
              exchange_rate: converted_amount / from_amount
            },
            'Currency conversion successful'
          )
        rescue StandardError => e
          error_response('Conversion failed', :unprocessable_entity, [e.message])
        end
      end
      
      private
      
      def process_aba_payment(amount, currency)
        # Implement ABA payment processing
        AbaPaymentGateway.charge(
          amount: amount,
          currency: currency,
          customer_id: current_user.id,
          description: 'Wallet topup'
        )
      end
      
      def process_wing_payment(amount, currency)
        # Implement Wing payment processing
        WingPaymentGateway.charge(
          amount: amount,
          currency: currency,
          customer_phone: current_user.phone_number,
          description: 'Wallet topup'
        )
      end
      
      def process_card_payment(amount, currency)
        # Implement card payment processing
        CardPaymentGateway.charge(
          amount: amount,
          currency: currency,
          customer_id: current_user.id,
          card_token: params[:card_token],
          description: 'Wallet topup'
        )
      end
      
      def format_transaction(transaction)
        {
          id: transaction.id,
          reference_number: transaction.reference_number,
          amount: transaction.amount,
          currency: transaction.currency,
          transaction_type: transaction.transaction_type,
          description: transaction.transaction_description,
          status: 'completed',
          created_at: transaction.created_at,
          recipient: transaction.recipient&.display_name,
          sender: transaction.sender&.display_name,
          metadata: transaction.metadata_hash
        }
      end
    end
  end
end
```

### 3. Push Notification System

```ruby
# app/services/push_notification_service.rb
class PushNotificationService
  def self.send_notification(user_ids, title, body, data = {})
    users = Spree::User.where(id: user_ids).where.not(push_token: nil)
    
    users.find_each do |user|
      send_to_user(user, title, body, data)
    end
  end
  
  def self.send_to_user(user, title, body, data = {})
    return unless user.push_token.present?
    
    notification = PushNotification.create!(
      user: user,
      title: title,
      body: body,
      data: data,
      notification_type: data[:type]
    )
    
    PushNotificationJob.perform_later(notification.id)
    
    notification
  end
  
  def self.send_bulk_notification(title, body, data = {}, filters = {})
    users = Spree::User.where.not(push_token: nil)
    
    # Apply filters
    users = users.where(loyalty_tier: filters[:tier]) if filters[:tier].present?
    users = users.where(province: filters[:province]) if filters[:province].present?
    users = users.where('created_at >= ?', filters[:registered_after]) if filters[:registered_after].present?
    
    users.find_each(batch_size: 100) do |user|
      send_to_user(user, title, body, data)
    end
  end
end

# app/jobs/push_notification_job.rb
class PushNotificationJob < ApplicationJob
  queue_as :default
  
  def perform(notification_id)
    notification = PushNotification.find(notification_id)
    user = notification.user
    
    return unless user.push_token.present?
    
    begin
      case user.device_type
      when 'ios'
        send_ios_notification(notification)
      when 'android'
        send_android_notification(notification)
      else
        send_fcm_notification(notification)
      end
      
      notification.update!(
        status: 'sent',
        sent_at: Time.current
      )
    rescue StandardError => e
      notification.update!(
        status: 'failed',
        failure_reason: e.message
      )
      
      # Remove invalid tokens
      if e.message.include?('InvalidRegistration') || e.message.include?('NotRegistered')
        user.update!(push_token: nil)
      end
    end
  end
  
  private
  
  def send_ios_notification(notification)
    client = Apnotic::Client.new(
      connection: Apnotic::Connection.development(
        cert_path: Rails.application.credentials.apn_cert_path
      )
    )
    
    apn_notification = Apnotic::Notification.new(notification.user.push_token)
    apn_notification.alert = {
      title: notification.title,
      body: notification.body
    }
    apn_notification.custom_payload = notification.data
    apn_notification.badge = calculate_badge_count(notification.user)
    
    response = client.push(apn_notification)
    
    unless response.ok?
      raise "APN Error: #{response.body}"
    end
  ensure
    client&.close
  end
  
  def send_android_notification(notification)
    send_fcm_notification(notification)
  end
  
  def send_fcm_notification(notification)
    fcm = FCM.new(Rails.application.credentials.fcm_server_key)
    
    response = fcm.send(
      [notification.user.push_token],
      {
        notification: {
          title: notification.title,
          body: notification.body
        },
        data: notification.data.transform_values(&:to_s)
      }
    )
    
    if response[:failure] > 0
      raise "FCM Error: #{response[:results].first}"
    end
  end
  
  def calculate_badge_count(user)
    # Count unread notifications
    user.push_notifications.where(read_at: nil).count
  end
end

# app/controllers/api/v1/notifications_controller.rb
module Api
  module V1
    class NotificationsController < BaseController
      # GET /api/v1/notifications
      def index
        notifications = current_user.push_notifications
                                   .order(created_at: :desc)
        
        notifications = paginate(notifications)
        
        render json: {
          success: true,
          data: {
            notifications: notifications.map { |n| format_notification(n) },
            unread_count: current_user.push_notifications.where(read_at: nil).count,
            pagination: pagination_meta(notifications)
          }
        }
      end
      
      # PUT /api/v1/notifications/:id/read
      def mark_as_read
        notification = current_user.push_notifications.find(params[:id])
        notification.update!(read_at: Time.current)
        
        success_response(message: 'Notification marked as read')
      end
      
      # PUT /api/v1/notifications/mark_all_read
      def mark_all_read
        current_user.push_notifications
                   .where(read_at: nil)
                   .update_all(read_at: Time.current)
        
        success_response(message: 'All notifications marked as read')
      end
      
      # DELETE /api/v1/notifications/:id
      def destroy
        notification = current_user.push_notifications.find(params[:id])
        notification.destroy!
        
        success_response(message: 'Notification deleted')
      end
      
      # GET /api/v1/notifications/preferences
      def preferences
        render json: {
          success: true,
          data: current_user.notification_preferences
        }
      end
      
      # PUT /api/v1/notifications/preferences
      def update_preferences
        current_user.update!(notification_preferences: notification_params)
        
        success_response(
          current_user.notification_preferences,
          'Notification preferences updated'
        )
      end
      
      private
      
      def format_notification(notification)
        {
          id: notification.id,
          title: notification.title,
          body: notification.body,
          type: notification.notification_type,
          data: notification.data,
          read: notification.read_at.present?,
          created_at: notification.created_at,
          time_ago: time_ago_in_words(notification.created_at)
        }
      end
      
      def notification_params
        params.permit(
          :push_enabled, :email_enabled, :sms_enabled,
          :marketing_enabled, :voting_updates, :order_updates,
          :event_reminders, :loyalty_updates, :security_alerts
        )
      end
    end
  end
end
```

### 4. Real-time WebSocket API

```ruby
# app/controllers/api/v1/websocket_controller.rb
module Api
  module V1
    class WebsocketController < BaseController
      # POST /api/v1/websocket/subscribe
      def subscribe
        channel = params[:channel]
        channel_params = params[:channel_params] || {}
        
        case channel
        when 'voting'
          episode_id = channel_params[:episode_id]
          episode = Xfactor::Episode.find(episode_id)
          
          # Generate WebSocket connection token
          token = generate_ws_token(current_user, 'voting', episode_id)
          
          render json: {
            success: true,
            data: {
              ws_url: "#{websocket_url}/cable",
              token: token,
              channel: 'VotingChannel',
              params: { episode_id: episode_id }
            }
          }
        when 'user'
          # Personal user channel for notifications
          token = generate_ws_token(current_user, 'user', current_user.id)
          
          render json: {
            success: true,
            data: {
              ws_url: "#{websocket_url}/cable",
              token: token,
              channel: 'UserChannel',
              params: { user_id: current_user.id }
            }
          }
        else
          error_response('Invalid channel')
        end
      end
      
      # GET /api/v1/websocket/health
      def health
        render json: {
          success: true,
          data: {
            status: 'connected',
            server_time: Time.current,
            active_connections: ActionCable.server.connections.count
          }
        }
      end
      
      private
      
      def generate_ws_token(user, channel, resource_id)
        JsonWebToken.encode(
          user_id: user.id,
          channel: channel,
          resource_id: resource_id,
          exp: 2.hours.from_now
        )
      end
      
      def websocket_url
        if Rails.env.production?
          'wss://api.hangmeas.com'
        else
          'ws://localhost:3000'
        end
      end
    end
  end
end

# app/channels/user_channel.rb
class UserChannel < ApplicationCable::Channel
  def subscribed
    user = find_verified_user
    return reject unless user
    
    stream_from "user_#{user.id}"
    
    # Send connection confirmation
    transmit(type: 'connected', data: { user_id: user.id })
  end
  
  def unsubscribed
    # Cleanup when channel is unsubscribed
  end
  
  def ping(data)
    transmit(type: 'pong', timestamp: Time.current)
  end
  
  def request_notifications(data)
    user = find_verified_user
    return unless user
    
    notifications = user.push_notifications
                       .where(read_at: nil)
                       .order(created_at: :desc)
                       .limit(10)
    
    transmit(
      type: 'notifications',
      data: {
        notifications: notifications.map do |n|
          {
            id: n.id,
            title: n.title,
            body: n.body,
            type: n.notification_type,
            data: n.data,
            created_at: n.created_at
          }
        end,
        unread_count: user.push_notifications.where(read_at: nil).count
      }
    )
  end
  
  private
  
  def find_verified_user
    token = params[:token]
    return nil unless token
    
    begin
      decoded = JsonWebToken.decode(token)
      return nil unless decoded[:channel] == 'user'
      
      Spree::User.find(decoded[:user_id])
    rescue JWT::DecodeError
      nil
    end
  end
end

# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    
    def connect
      self.current_user = find_verified_user
    end
    
    private
    
    def find_verified_user
      token = request.params[:token]
      return reject_unauthorized_connection unless token
      
      begin
        decoded = JsonWebToken.decode(token)
        user = Spree::User.find(decoded[:user_id])
        user
      rescue JWT::DecodeError
        reject_unauthorized_connection
      end
    end
  end
end
```

### 5. Digital Marketplace Integration APIs

```ruby
# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < BaseController
      before_action :authenticate_request, except: [:index, :show]
      
      # GET /api/v1/products
      def index
        products = Spree::Product.active.includes(:images, :variants, :taxons)
        
        # Apply filters
        products = apply_product_filters(products)
        
        # Apply search
        if params[:q].present?
          products = products.ransack(
            name_or_description_cont: params[:q],
            variants_sku_cont: params[:q],
            m: 'or'
          ).result
        end
        
        # Sort
        products = apply_sorting(products)
        
        # Paginate
        products = paginate(products, per_page: 20)
        
        render json: {
          success: true,
          data: {
            products: products.map { |p| format_product(p) },
            pagination: pagination_meta(products),
            filters: available_filters
          }
        }
      end
      
      # GET /api/v1/products/:id
      def show
        product = Spree::Product.active.find(params[:id])
        
        render json: {
          success: true,
          data: {
            product: detailed_product(product),
            related_products: related_products(product),
            reviews: product_reviews(product)
          }
        }
      end
      
      # GET /api/v1/products/featured
      def featured
        featured_products = Spree::Product.active
                                         .joins(:taxons)
                                         .where(spree_taxons: { name: 'Featured' })
                                         .limit(10)
        
        render json: {
          success: true,
          data: {
            products: featured_products.map { |p| format_product(p) }
          }
        }
      end
      
      # GET /api/v1/products/categories
      def categories
        taxons = Spree::Taxon.includes(:children)
                            .where(parent_id: Spree::Taxonomy.find_by(name: 'Categories')&.root&.id)
        
        render json: {
          success: true,
          data: {
            categories: taxons.map { |t| format_category(t) }
          }
        }
      end
      
      private
      
      def apply_product_filters(products)
        # Category filter
        if params[:category_id].present?
          products = products.joins(:taxons).where(spree_taxons: { id: params[:category_id] })
        end
        
        # Price range filter
        if params[:min_price].present? || params[:max_price].present?
          min_price = params[:min_price]&.to_f || 0
          max_price = params[:max_price]&.to_f || Float::INFINITY
          products = products.where(price: min_price..max_price)
        end
        
        # Event type filter
        if params[:event_type].present?
          products = products.where(event_type: params[:event_type])
        end
        
        # Availability filter
        if params[:available].present?
          products = products.where(available_on: ..Time.current)
        end
        
        products
      end
      
      def apply_sorting(products)
        case params[:sort]
        when 'price_asc'
          products.order(price: :asc)
        when 'price_desc'
          products.order(price: :desc)
        when 'name'
          products.order(:name)
        when 'newest'
          products.order(created_at: :desc)
        when 'popular'
          products.joins(:orders).group(:id).order('COUNT(spree_orders.id) DESC')
        else
          products.order(:name)
        end
      end
      
      def format_product(product)
        {
          id: product.id,
          name: product.name,
          description: product.description&.truncate(200),
          price: product.price.to_f,
          display_price: product.display_price.to_s,
          currency: product.currency,
          sku: product.sku,
          slug: product.slug,
          available: product.available?,
          in_stock: product.in_stock?,
          images: product.images.map do |image|
            {
              id: image.id,
              url: rails_blob_url(image.attachment),
              alt: image.alt,
              position: image.position
            }
          end,
          categories: product.taxons.map(&:name),
          rating: product.avg_rating&.round(1),
          review_count: product.reviews_count || 0,
          event_date: product.event_date,
          venue_name: product.venue_name,
          badges: product_badges(product)
        }
      end
      
      def detailed_product(product)
        base_product = format_product(product)
        
        base_product.merge(
          full_description: product.description,
          properties: product.product_properties.includes(:property).map do |pp|
            {
              name: pp.property.presentation,
              value: pp.value
            }
          end,
          variants: product.variants.map do |variant|
            {
              id: variant.id,
              sku: variant.sku,
              price: variant.price.to_f,
              display_price: variant.display_price.to_s,
              options_text: variant.options_text,
              in_stock: variant.in_stock?,
              images: variant.images.map do |image|
                {
                  url: rails_blob_url(image.attachment),
                  alt: image.alt
                }
              end
            }
          end,
          shipping_info: {
            weight: product.weight,
            dimensions: {
              height: product.height,
              width: product.width,
              depth: product.depth
            },
            shipping_category: product.shipping_category&.name
          },
          seo: {
            meta_title: product.meta_title,
            meta_description: product.meta_description,
            meta_keywords: product.meta_keywords
          }
        )
      end
      
      def related_products(product)
        # Get products from same categories
        related = Spree::Product.active
                                .joins(:taxons)
                                .where(spree_taxons: { id: product.taxon_ids })
                                .where.not(id: product.id)
                                .limit(6)
        
        related.map { |p| format_product(p) }
      end
      
      def product_reviews(product)
        reviews = product.reviews.approved.includes(:user).order(created_at: :desc).limit(5)
        
        reviews.map do |review|
          {
            id: review.id,
            rating: review.rating,
            title: review.title,
            review: review.review,
            user_name: review.user.display_name,
            created_at: review.created_at,
            helpful_count: review.helpful_count || 0
          }
        end
      end
      
      def product_badges(product)
        badges = []
        
        badges << 'NEW' if product.created_at > 30.days.ago
        badges << 'SALE' if product.on_sale?
        badges << 'LIMITED' if product.event_type.present? && product.event_date&.< 7.days.from_now
        badges << 'BESTSELLER' if product.orders_count && product.orders_count > 100
        
        badges
      end
      
      def format_category(taxon)
        {
          id: taxon.id,
          name: taxon.name,
          slug: taxon.permalink,
          description: taxon.description,
          icon: taxon.icon.attached? ? rails_blob_url(taxon.icon) : nil,
          product_count: taxon.products.active.count,
          children: taxon.children.map do |child|
            {
              id: child.id,
              name: child.name,
              slug: child.permalink,
              product_count: child.products.active.count
            }
          end
        }
      end
      
      def available_filters
        {
          categories: Spree::Taxon.joins(:products).distinct.pluck(:id, :name).map { |id, name| { id: id, name: name } },
          price_ranges: [
            { label: 'Under $10', min: 0, max: 10 },
            { label: '$10 - $25', min: 10, max: 25 },
            { label: '$25 - $50', min: 25, max: 50 },
            { label: '$50 - $100', min: 50, max: 100 },
            { label: 'Over $100', min: 100, max: nil }
          ],
          event_types: Spree::Product.active.distinct.pluck(:event_type).compact,
          sort_options: [
            { value: 'name', label: 'Name A-Z' },
            { value: 'price_asc', label: 'Price: Low to High' },
            { value: 'price_desc', label: 'Price: High to Low' },
            { value: 'newest', label: 'Newest First' },
            { value: 'popular', label: 'Most Popular' }
          ]
        }
      end
    end
  end
end

# app/controllers/api/v1/cart_controller.rb
module Api
  module V1
    class CartController < BaseController
      before_action :ensure_cart
      
      # GET /api/v1/cart
      def show
        render json: {
          success: true,
          data: format_cart(@cart)
        }
      end
      
      # POST /api/v1/cart/add
      def add_item
        variant = Spree::Variant.find(params[:variant_id])
        quantity = params[:quantity]&.to_i || 1
        
        line_item = @cart.contents.add(variant, quantity)
        
        if line_item.persisted?
          @cart.reload
          success_response(
            format_cart(@cart),
            'Item added to cart'
          )
        else
          error_response(
            'Failed to add item to cart',
            :unprocessable_entity,
            line_item.errors.full_messages
          )
        end
      end
      
      # PUT /api/v1/cart/update/:line_item_id
      def update_item
        line_item = @cart.line_items.find(params[:line_item_id])
        quantity = params[:quantity].to_i
        
        if quantity > 0
          line_item.update!(quantity: quantity)
        else
          line_item.destroy!
        end
        
        @cart.reload
        success_response(
          format_cart(@cart),
          'Cart updated'
        )
      end
      
      # DELETE /api/v1/cart/remove/:line_item_id
      def remove_item
        line_item = @cart.line_items.find(params[:line_item_id])
        line_item.destroy!
        
        @cart.reload
        success_response(
          format_cart(@cart),
          'Item removed from cart'
        )
      end
      
      # DELETE /api/v1/cart/clear
      def clear
        @cart.empty!
        
        success_response(
          format_cart(@cart),
          'Cart cleared'
        )
      end
      
      # POST /api/v1/cart/apply_coupon
      def apply_coupon
        promotion = Spree::Promotion.active.find_by(code: params[:coupon_code])
        
        unless promotion
          return error_response('Invalid coupon code')
        end
        
        if @cart.promotions.include?(promotion)
          return error_response('Coupon already applied')
        end
        
        @cart.promotions << promotion
        @cart.recalculate
        
        success_response(
          format_cart(@cart),
          'Coupon applied successfully'
        )
      end
      
      # DELETE /api/v1/cart/remove_coupon/:promotion_id
      def remove_coupon
        promotion = @cart.promotions.find(params[:promotion_id])
        @cart.promotions.delete(promotion)
        @cart.recalculate
        
        success_response(
          format_cart(@cart),
          'Coupon removed'
        )
      end
      
      private
      
      def ensure_cart
        @cart = current_user.orders.incomplete.first || current_user.orders.create!
      end
      
      def format_cart(cart)
        {
          id: cart.id,
          number: cart.number,
          item_count: cart.item_count,
          item_total: cart.item_total.to_f,
          total: cart.total.to_f,
          display_item_total: cart.display_item_total.to_s,
          display_total: cart.display_total.to_s,
          currency: cart.currency,
          line_items: cart.line_items.map do |item|
            {
              id: item.id,
              variant_id: item.variant_id,
              product_name: item.product.name,
              variant_name: item.variant.options_text,
              quantity: item.quantity,
              price: item.price.to_f,
              total: item.total.to_f,
              display_price: item.display_price.to_s,
              display_total: item.display_total.to_s,
              image_url: item.variant.images.first&.attachment&.then { |img| rails_blob_url(img) },
              in_stock: item.variant.in_stock?
            }
          end,
          adjustments: cart.adjustments.map do |adj|
            {
              id: adj.id,
              label: adj.label,
              amount: adj.amount.to_f,
              display_amount: adj.display_amount.to_s,
              eligible: adj.eligible?
            }
          end,
          promotions: cart.promotions.map do |promo|
            {
              id: promo.id,
              name: promo.name,
              code: promo.code,
              description: promo.description
            }
          end,
          shipping_required: cart.shipments.any?,
          taxes_included: Spree::Config.prices_inc_tax
        }
      end
    end
  end
end
```

### 6. API Rate Limiting & Security

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle requests by IP (60rpm)
  throttle('req/ip', limit: 60, period: 1.minute) do |req|
    req.ip if req.path.start_with?('/api/')
  end
  
  # Throttle login attempts by IP (5 attempts per 20 minutes)
  throttle('logins/ip', limit: 5, period: 20.minutes) do |req|
    req.ip if req.path == '/api/v1/auth/login' && req.post?
  end
  
  # Throttle login attempts by email
  throttle('logins/email', limit: 5, period: 20.minutes) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.params['email']&.presence || req.params['phone_number']&.presence
    end
  end
  
  # Throttle voting by user (100 votes per minute)
  throttle('voting/user', limit: 100, period: 1.minute) do |req|
    if req.path.include?('/voting/cast_vote') && req.post?
      # Extract user ID from JWT token
      auth_header = req.get_header('HTTP_AUTHORIZATION')
      if auth_header && (token = auth_header.split(' ').last)
        begin
          decoded = JsonWebToken.decode(token)
          decoded[:user_id]
        rescue
          nil
        end
      end
    end
  end
  
  # Throttle SMS verification requests
  throttle('sms/phone', limit: 3, period: 1.hour) do |req|
    if req.path.include?('/auth/verify_phone') || req.path.include?('/auth/resend_verification')
      req.params['phone_number']&.presence
    end
  end
end

# app/controllers/concerns/api_rate_limiting.rb
module ApiRateLimiting
  extend ActiveSupport::Concern
  
  included do
    rescue_from Rack::Attack::Throttle do |_exception|
      render json: {
        success: false,
        error: 'Rate limit exceeded',
        message: 'Too many requests. Please try again later.'
      }, status: :too_many_requests
    end
  end
end

# Add to BaseController
class Api::V1::BaseController
  include ApiRateLimiting
  # ... rest of controller
end
```

### 7. API Documentation Generation

```ruby
# spec/requests/api_documentation_spec.rb
require 'rails_helper'
require 'rspec_api_documentation/dsl'

resource "Authentication" do
  explanation "API endpoints for user authentication"
  
  header "Content-Type", "application/json"
  
  post "/api/v1/auth/login" do
    parameter :email, "User email address", required: false
    parameter :phone_number, "User phone number", required: false
    parameter :password, "User password", required: true
    parameter :device_type, "Device type (ios/android)", required: false
    parameter :push_token, "Push notification token", required: false
    
    example "Login with email" do
      user = create(:user, email: 'test@example.com', password: 'password123')
      
      do_request(
        email: 'test@example.com',
        password: 'password123',
        device_type: 'ios'
      )
      
      expect(status).to eq(200)
      expect(response_body).to match(/token/)
    end
    
    example "Login with phone number" do
      user = create(:user, phone_number: '012345678', password: 'password123')
      
      do_request(
        phone_number: '012345678',
        password: 'password123',
        device_type: 'android'
      )
      
      expect(status).to eq(200)
      expect(response_body).to match(/token/)
    end
  end
  
  post "/api/v1/auth/register" do
    parameter :email, "User email address", required: true
    parameter :phone_number, "User phone number", required: true
    parameter :password, "User password", required: true
    parameter :first_name, "User first name", required: true
    parameter :last_name, "User last name", required: true
    
    example "Register new user" do
      do_request(
        email: 'newuser@example.com',
        phone_number: '012345679',
        password: 'password123',
        first_name: 'John',
        last_name: 'Doe'
      )
      
      expect(status).to eq(201)
      expect(response_body).to match(/token/)
    end
  end
end

# Generate docs with: bundle exec rake docs:generate
```

---

## 📊 Performance Optimization

### 1. Database Optimization

```ruby
# config/initializers/database_optimizations.rb
Rails.application.configure do
  # Connection pooling
  config.database_configuration[Rails.env]['pool'] = ENV.fetch('DB_POOL', 25).to_i
  config.database_configuration[Rails.env]['checkout_timeout'] = 5
  
  # Query optimization
  config.active_record.strict_loading_by_default = true if Rails.env.development?
end

# Add indexes for API queries
# db/migrate/add_api_indexes.rb
class AddApiIndexes < ActiveRecord::Migration[7.0]
  def change
    # User queries
    add_index :spree_users, [:last_active_at, :id]
    add_index :spree_users, [:push_token, :device_type]
    
    # Product queries
    add_index :spree_products, [:available_on, :id]
    add_index :spree_products, [:created_at, :id]
    add_index :spree_products, [:price, :id]
    
    # Order queries
    add_index :spree_orders, [:user_id, :state, :created_at]
    add_index :spree_line_items, [:order_id, :variant_id]
    
    # Notification queries
    add_index :push_notifications, [:user_id, :read_at, :created_at]
    add_index :push_notifications, [:created_at, :notification_type]
  end
end
```

### 2. Caching Strategy

```ruby
# config/initializers/caching.rb
Rails.application.configure do
  config.cache_store = :redis_cache_store, {
    url: ENV['REDIS_URL'],
    namespace: 'hangmeas_api',
    expires_in: 1.hour,
    race_condition_ttl: 5.seconds,
    error_handler: -> (method:, returning:, exception:) {
      Rails.logger.error("Cache error: #{exception.message}")
    }
  }
end

# app/controllers/concerns/api_caching.rb
module ApiCaching
  extend ActiveSupport::Concern
  
  private
  
  def cache_json(key, expires_in: 5.minutes, &block)
    Rails.cache.fetch(key, expires_in: expires_in) do
      yield.to_json
    end
  end
  
  def cache_key_for(object, suffix = nil)
    timestamp = object.respond_to?(:updated_at) ? object.updated_at.to_i : Time.current.to_i
    key = "#{object.class.name.downcase}_#{object.id}_#{timestamp}"
    key += "_#{suffix}" if suffix
    key
  end
end
```

---

## 🧪 Testing Suite

```ruby
# spec/support/api_helpers.rb
module ApiHelpers
  def json_response
    JSON.parse(response.body)
  end
  
  def auth_headers_for(user)
    token = JsonWebToken.encode(user_id: user.id)
    { 'Authorization' => "Bearer #{token}" }
  end
  
  def expect_success_response
    expect(response).to have_http_status(:success)
    expect(json_response['success']).to be_truthy
  end
  
  def expect_error_response(status = :bad_request)
    expect(response).to have_http_status(status)
    expect(json_response['success']).to be_falsey
  end
end

RSpec.configure do |config|
  config.include ApiHelpers, type: :request
end

# spec/requests/api/v1/auth_spec.rb
require 'rails_helper'

RSpec.describe 'Authentication API', type: :request do
  describe 'POST /api/v1/auth/login' do
    let(:user) { create(:user, email: 'test@example.com', password: 'password123') }
    
    it 'authenticates user with valid credentials' do
      post '/api/v1/auth/login', params: {
        email: user.email,
        password: 'password123'
      }
      
      expect_success_response
      expect(json_response['token']).to be_present
      expect(json_response['user']['id']).to eq(user.id)
    end
    
    it 'rejects invalid credentials' do
      post '/api/v1/auth/login', params: {
        email: user.email,
        password: 'wrongpassword'
      }
      
      expect_error_response(:unauthorized)
      expect(json_response['message']).to eq('Invalid credentials')
    end
  end
end
```

---

## 📚 Deployment & Monitoring

### 1. API Monitoring

```ruby
# config/initializers/instrumentation.rb
ActiveSupport::Notifications.subscribe('process_action.action_controller') do |name, started, finished, unique_id, data|
  if data[:controller].start_with?('Api::')
    duration = finished - started
    
    # Log slow API requests
    if duration > 1.0
      Rails.logger.warn "Slow API request: #{data[:controller]}##{data[:action]} took #{duration}s"
    end
    
    # Track metrics
    ApiMetrics.track(
      controller: data[:controller],
      action: data[:action],
      status: data[:status],
      duration: duration,
      user_id: Current.user&.id
    )
  end
end
```

### 2. Health Check Endpoint

```ruby
# app/controllers/api/health_controller.rb
module Api
  class HealthController < ApplicationController
    include ActionController::API
    
    def show
      health_data = {
        status: 'healthy',
        timestamp: Time.current,
        version: Rails.application.version,
        environment: Rails.env,
        services: {
          database: check_database,
          redis: check_redis,
          storage: check_storage
        }
      }
      
      overall_status = health_data[:services].values.all? { |s| s[:status] == 'healthy' } ? 200 : 503
      
      render json: health_data, status: overall_status
    end
    
    private
    
    def check_database
      ActiveRecord::Base.connection.execute('SELECT 1')
      { status: 'healthy', response_time: 0 }
    rescue => e
      { status: 'unhealthy', error: e.message }
    end
    
    def check_redis
      Redis.current.ping
      { status: 'healthy' }
    rescue => e
      { status: 'unhealthy', error: e.message }
    end
    
    def check_storage
      ActiveStorage::Blob.service.exist?('health_check')
      { status: 'healthy' }
    rescue => e
      { status: 'unhealthy', error: e.message }
    end
  end
end
```

---

## 🚀 Next Steps

After completing Week 7-8:

1. **Load Testing**: Use tools like Apache Bench or Artillery to test API performance
2. **Security Audit**: Review authentication, authorization, and data validation
3. **Documentation Review**: Ensure all endpoints are properly documented
4. **Mobile Integration**: Work with mobile developers to integrate APIs
5. **Monitoring Setup**: Configure alerting for API errors and performance issues
6. **Prepare for Week 9-10**: Subscription system implementation