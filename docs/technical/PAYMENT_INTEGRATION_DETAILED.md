# Payment Integration - Complete Technical Guide

## Overview of Payment Landscape in Cambodia

### Popular Payment Methods
1. **ABA PayWay** - 40% market share
2. **ACLEDA XPay** - 25% market share  
3. **Wing Money** - 20% market share
4. **TrueMoney** - 10% market share
5. **International Cards** - 5% market share

## 1. ABA PayWay Integration

### Complete Integration Flow

#### Step 1: Merchant Setup
```javascript
// Merchant Configuration
{
  "merchant": {
    "id": "HANGMEAS_PROD_001",
    "name": "Hang Meas Entertainment",
    "api_key": "pk_live_xxxxxxxxxxxxxx",
    "secret_key": "sk_live_xxxxxxxxxxxxxx",
    "webhook_url": "https://api.hangmeas.com/webhooks/aba",
    "return_url": "hangmeas://payment/complete",
    "categories": ["entertainment", "tickets"]
  }
}
```

#### Step 2: Payment Request Flow
```
Mobile App                    Backend                      ABA API
    │                            │                            │
    ├─ 1. Select ABA ───────────▶│                            │
    │                            ├─ 2. Create Order ─────────▶│
    │                            │   {                        │
    │                            │     amount: 52.50,         │
    │                            │     items: [...],          │
    │                            │     customer: {...}        │
    │                            │   }                        │
    │                            │                            │
    │                            │◀─ 3. Return QR Data ───────┤
    │◀─ 4. Display QR ───────────┤   {                        │
    │                            │     qr_string: "...",      │
    │                            │     reference: "PAY123",   │
    │                            │     expires_at: "..."      │
    │                            │   }                        │
    │                            │                            │
    │─ 5. Poll Status ─────────▶│─ 6. Check Status ─────────▶│
    │  (every 2 sec)             │                            │
    │                            │◀─ 7. Status Update ────────┤
    │◀─ 8. Update UI ────────────┤                            │
    │                            │                            │
    │                            │◀─ 9. Webhook (success) ────┤
    │                            │                            │
    │◀─ 10. Show Success ────────┤                            │
```

#### Step 3: QR Code Display Implementation
```javascript
// Frontend: QR Code Component
const ABAPaymentQR = ({ orderData }) => {
  const [timeLeft, setTimeLeft] = useState(300); // 5 minutes
  const [status, setStatus] = useState('pending');
  
  useEffect(() => {
    // Countdown timer
    const timer = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 0) {
          setStatus('expired');
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
    
    // Status polling
    const pollStatus = setInterval(async () => {
      const result = await checkPaymentStatus(orderData.reference);
      if (result.status === 'completed') {
        setStatus('success');
        clearInterval(pollStatus);
      }
    }, 2000);
    
    return () => {
      clearInterval(timer);
      clearInterval(pollStatus);
    };
  }, []);
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Scan to Pay with ABA</Text>
      
      {status === 'pending' && (
        <>
          <QRCode 
            value={orderData.qr_string}
            size={250}
            logo={require('./assets/aba-logo.png')}
          />
          
          <View style={styles.details}>
            <Text>Amount: ${orderData.amount}</Text>
            <Text>Order: {orderData.reference}</Text>
            <Text>Expires in: {formatTime(timeLeft)}</Text>
          </View>
          
          <TouchableOpacity style={styles.helpButton}>
            <Text>Having trouble? Pay another way</Text>
          </TouchableOpacity>
        </>
      )}
      
      {status === 'expired' && (
        <View style={styles.expired}>
          <Icon name="clock" size={50} color="red" />
          <Text>Payment time expired</Text>
          <Button title="Try Again" onPress={retry} />
        </View>
      )}
      
      {status === 'success' && (
        <View style={styles.success}>
          <Icon name="check-circle" size={50} color="green" />
          <Text>Payment Successful!</Text>
          <Text>Redirecting to your tickets...</Text>
        </View>
      )}
    </View>
  );
};
```

#### Step 4: Backend Payment Processing
```javascript
// Backend: ABA Payment Service
class ABAPaymentService {
  constructor() {
    this.apiKey = process.env.ABA_API_KEY;
    this.secretKey = process.env.ABA_SECRET_KEY;
    this.baseUrl = process.env.ABA_API_URL;
  }
  
  async createPaymentRequest(orderData) {
    try {
      // Prepare request
      const payload = {
        merchant_id: this.merchantId,
        amount: orderData.total,
        currency: 'USD',
        order_id: orderData.id,
        order_desc: `Tickets for ${orderData.eventName}`,
        customer: {
          name: orderData.customerName,
          phone: orderData.customerPhone,
          email: orderData.customerEmail
        },
        items: orderData.tickets.map(ticket => ({
          name: `${ticket.type} - Seat ${ticket.seat}`,
          quantity: 1,
          price: ticket.price
        })),
        return_url: `${this.baseUrl}/payment/aba/return`,
        callback_url: `${this.baseUrl}/webhooks/aba`,
        expires_in: 300 // 5 minutes
      };
      
      // Sign request
      const signature = this.generateSignature(payload);
      payload.signature = signature;
      
      // Make API call
      const response = await axios.post(
        `${this.baseUrl}/api/v1/payment/create`,
        payload,
        {
          headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      // Store payment reference
      await this.storePaymentReference({
        orderId: orderData.id,
        reference: response.data.reference,
        expiresAt: new Date(Date.now() + 300000)
      });
      
      return {
        success: true,
        qr_string: response.data.qr_code,
        reference: response.data.reference,
        expires_at: response.data.expires_at
      };
      
    } catch (error) {
      logger.error('ABA payment creation failed:', error);
      throw new PaymentError('Failed to create payment request');
    }
  }
  
  generateSignature(payload) {
    const sortedKeys = Object.keys(payload).sort();
    const signString = sortedKeys
      .map(key => `${key}=${payload[key]}`)
      .join('&');
    
    return crypto
      .createHmac('sha256', this.secretKey)
      .update(signString)
      .digest('hex');
  }
  
  async handleWebhook(webhookData) {
    // Verify webhook signature
    const expectedSignature = this.generateSignature(webhookData);
    if (webhookData.signature !== expectedSignature) {
      throw new Error('Invalid webhook signature');
    }
    
    // Process based on status
    switch (webhookData.status) {
      case 'completed':
        await this.handleSuccessfulPayment(webhookData);
        break;
      case 'failed':
        await this.handleFailedPayment(webhookData);
        break;
      case 'cancelled':
        await this.handleCancelledPayment(webhookData);
        break;
    }
  }
  
  async handleSuccessfulPayment(data) {
    const order = await Order.findOne({ 
      reference: data.reference 
    });
    
    if (!order) {
      throw new Error('Order not found');
    }
    
    // Update order status
    order.status = 'paid';
    order.paymentDetails = {
      method: 'aba_payway',
      transactionId: data.transaction_id,
      paidAt: new Date()
    };
    await order.save();
    
    // Issue tickets
    await this.ticketService.issueTickets(order);
    
    // Send confirmations
    await this.notificationService.sendPaymentSuccess(order);
  }
}
```

## 2. Wing Money Integration

### Wing Payment Flow Diagram
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Mobile App    │     │  Hang Meas API  │     │   Wing API      │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                         │
         │ 1. Select Wing        │                         │
         ├──────────────────────▶│                         │
         │                       │ 2. Init Transaction     │
         │                       ├────────────────────────▶│
         │                       │                         │
         │                       │ 3. Transaction ID       │
         │ 4. Enter Phone #      │◀────────────────────────┤
         │◀──────────────────────┤                         │
         │                       │                         │
         │ 5. Submit Phone       │                         │
         ├──────────────────────▶│ 6. Send Payment Req    │
         │                       ├────────────────────────▶│
         │                       │                         │
         │                       │ 7. SMS to User          │
         │                       │◀────────────────────────┤
         │                       │                         │
         │    User receives SMS and replies with PIN       │
         │                       │                         │
         │                       │ 8. Payment Complete     │
         │                       │◀────────────────────────┤
         │ 9. Success            │                         │
         │◀──────────────────────┤                         │
```

### Wing Integration Code
```javascript
// Wing Payment Service
class WingPaymentService {
  async initiatePayment(orderData) {
    // Step 1: Create Wing transaction
    const wingTransaction = await this.createWingTransaction({
      amount: orderData.total,
      currency: 'USD',
      merchant_ref: orderData.id,
      description: `Hang Meas Tickets - ${orderData.eventName}`
    });
    
    return {
      transactionId: wingTransaction.id,
      status: 'pending_phone'
    };
  }
  
  async sendPaymentRequest(transactionId, phoneNumber) {
    // Validate phone number
    if (!this.isValidWingNumber(phoneNumber)) {
      throw new Error('Invalid Wing account number');
    }
    
    // Step 2: Send payment request to user's phone
    const request = await this.wingAPI.sendPaymentRequest({
      transaction_id: transactionId,
      phone_number: this.formatPhoneNumber(phoneNumber),
      notification_type: 'SMS',
      language: 'km' // Khmer
    });
    
    // Step 3: Start monitoring for payment
    this.startPaymentMonitoring(transactionId);
    
    return {
      status: 'pending_pin',
      message: 'Please check your SMS and reply with your PIN'
    };
  }
  
  isValidWingNumber(phone) {
    // Wing numbers format validation
    const wingFormat = /^(0|855)?(11|12|13|14|15|16|17|18|19|31|60|61|66|67|68|69|70|71|76|77|78|85|86|87|88|89|90|92|93|95|96|97|98|99)[0-9]{6,7}$/;
    return wingFormat.test(phone.replace(/\s+/g, ''));
  }
  
  formatPhoneNumber(phone) {
    // Convert to Wing format
    let cleaned = phone.replace(/\s+/g, '');
    if (cleaned.startsWith('0')) {
      cleaned = '855' + cleaned.substring(1);
    }
    if (!cleaned.startsWith('855')) {
      cleaned = '855' + cleaned;
    }
    return cleaned;
  }
  
  async handleWingWebhook(data) {
    const { transaction_id, status, paid_at } = data;
    
    const order = await Order.findOne({ 
      'payment.wingTransactionId': transaction_id 
    });
    
    if (status === 'SUCCESS') {
      order.status = 'paid';
      order.payment.completedAt = paid_at;
      await order.save();
      
      // Issue tickets
      await this.ticketService.issueTickets(order);
      
      // Send SMS confirmation via Wing
      await this.sendWingConfirmation(order);
    }
  }
}
```

### Wing SMS Templates
```javascript
// SMS sent to user for payment request
const paymentRequestSMS = {
  km: `ទូទាត់ $${amount} ទៅ Hang Meas សម្រាប់សំបុត្រ ${eventName}។ ឆ្លើយតបជាមួយលេខសម្ងាត់ Wing របស់អ្នក។`,
  en: `Pay $${amount} to Hang Meas for ${eventName} tickets. Reply with your Wing PIN.`
};

// SMS sent after successful payment
const confirmationSMS = {
  km: `ការទូទាត់ជោគជ័យ! សំបុត្ររបស់អ្នកសម្រាប់ ${eventName} ត្រូវបានផ្ញើទៅកាន់ Hang Meas app។`,
  en: `Payment successful! Your tickets for ${eventName} have been sent to your Hang Meas app.`
};
```

## 3. ACLEDA XPay Integration

### ACLEDA Payment Flow
```
User Interface                Backend               ACLEDA XPay
      │                          │                      │
      │ Select ACLEDA XPay       │                      │
      ├─────────────────────────▶│                      │
      │                          │ Create Session       │
      │                          ├─────────────────────▶│
      │                          │                      │
      │                          │ Session Token + URL  │
      │ Redirect to ACLEDA       │◀─────────────────────┤
      │◀─────────────────────────┤                      │
      │                          │                      │
      │════════════════════════════════════════════════│
      │         User logs into ACLEDA app              │
      │         Confirms payment details               │
      │         Enters transaction PIN                 │
      │════════════════════════════════════════════════│
      │                          │                      │
      │                          │ Payment notification │
      │                          │◀─────────────────────┤
      │ Return to app            │                      │
      │◀─────────────────────────┤                      │
      │                          │                      │
```

### ACLEDA Integration Implementation
```javascript
class ACLEDAPaymentService {
  async createPaymentSession(orderData) {
    const session = await this.acledaAPI.createSession({
      merchant_id: process.env.ACLEDA_MERCHANT_ID,
      amount: orderData.total,
      currency: 'USD',
      reference: orderData.id,
      return_url: 'hangmeas://payment/acleda/return',
      cancel_url: 'hangmeas://payment/acleda/cancel',
      notify_url: `${process.env.API_URL}/webhooks/acleda`,
      customer: {
        name: orderData.customerName,
        phone: orderData.customerPhone
      },
      items: this.formatOrderItems(orderData.tickets),
      expires_in: 600 // 10 minutes
    });
    
    // Store session for verification
    await this.redis.setex(
      `acleda:session:${session.token}`,
      600,
      JSON.stringify({
        orderId: orderData.id,
        amount: orderData.total
      })
    );
    
    return {
      sessionToken: session.token,
      paymentUrl: session.payment_url,
      expiresAt: session.expires_at
    };
  }
  
  async handleReturn(sessionToken, status) {
    const sessionData = await this.redis.get(`acleda:session:${sessionToken}`);
    if (!sessionData) {
      throw new Error('Invalid or expired session');
    }
    
    const { orderId } = JSON.parse(sessionData);
    
    if (status === 'success') {
      // Verify with ACLEDA
      const verification = await this.acledaAPI.verifyPayment(sessionToken);
      
      if (verification.status === 'PAID') {
        await this.completeOrder(orderId, {
          method: 'acleda_xpay',
          transactionId: verification.transaction_id
        });
      }
    }
  }
}
```

## 4. International Card Processing (Visa/Mastercard)

### 3D Secure Flow
```
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│   App      │     │   Backend  │     │  Payment   │     │   Bank     │
│            │     │            │     │  Gateway   │     │            │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                  │                   │                   │
      │ Card Details     │                   │                   │
      ├─────────────────▶│                   │                   │
      │                  │ Tokenize          │                   │
      │                  ├──────────────────▶│                   │
      │                  │                   │ Check 3DS        │
      │                  │ Token + 3DS URL   │──────────────────▶
      │ 3DS Challenge    │◀──────────────────┤                   │
      │◀─────────────────┤                   │                   │
      │                  │                   │                   │
      │ [User completes 3DS authentication with bank]            │
      │                  │                   │                   │
      │ 3DS Complete     │                   │                   │
      ├─────────────────▶│ Process Payment   │                   │
      │                  ├──────────────────▶│ Authorize        │
      │                  │                   ├──────────────────▶│
      │                  │                   │                   │
      │                  │ Success           │ Approved         │
      │ Payment Complete │◀──────────────────┤◀──────────────────┤
      │◀─────────────────┤                   │                   │
```

### Card Payment Implementation
```javascript
class CardPaymentService {
  async processCardPayment(paymentData) {
    try {
      // Step 1: Tokenize card
      const token = await this.tokenizeCard({
        number: paymentData.cardNumber,
        exp_month: paymentData.expMonth,
        exp_year: paymentData.expYear,
        cvc: paymentData.cvc,
        name: paymentData.cardholderName
      });
      
      // Step 2: Create payment intent
      const intent = await this.gateway.createPaymentIntent({
        amount: paymentData.amount * 100, // Convert to cents
        currency: 'usd',
        payment_method: token.id,
        confirmation_method: 'manual',
        confirm: true,
        metadata: {
          order_id: paymentData.orderId,
          customer_id: paymentData.customerId
        }
      });
      
      // Step 3: Handle 3D Secure if required
      if (intent.status === 'requires_action' && 
          intent.next_action.type === 'redirect_to_url') {
        return {
          requires3DS: true,
          redirectUrl: intent.next_action.redirect_to_url.url,
          returnUrl: `hangmeas://payment/3ds/return?intent=${intent.id}`
        };
      }
      
      // Step 4: Payment successful
      if (intent.status === 'succeeded') {
        await this.completeCardPayment(paymentData.orderId, intent);
        return {
          success: true,
          transactionId: intent.id
        };
      }
      
    } catch (error) {
      // Handle specific card errors
      if (error.code === 'card_declined') {
        throw new PaymentError('Your card was declined', 'CARD_DECLINED');
      }
      throw error;
    }
  }
  
  // PCI compliance - never store card details
  async tokenizeCard(cardDetails) {
    // This happens on frontend using payment gateway SDK
    // Never send raw card details to your backend
    return await stripe.createToken('card', cardDetails);
  }
}
```

## 5. Payment Security & Compliance

### Security Measures Implementation
```javascript
class PaymentSecurityService {
  // Fraud detection
  async checkFraudRisk(paymentData) {
    const riskScore = await this.calculateRiskScore({
      amount: paymentData.amount,
      userId: paymentData.userId,
      deviceFingerprint: paymentData.deviceId,
      ipAddress: paymentData.ip,
      paymentMethod: paymentData.method
    });
    
    if (riskScore > 0.7) {
      // High risk - require additional verification
      return {
        action: 'CHALLENGE',
        reason: 'High risk transaction'
      };
    }
    
    if (riskScore > 0.5) {
      // Medium risk - monitor closely
      await this.flagForReview(paymentData);
    }
    
    return { action: 'PROCEED' };
  }
  
  calculateRiskScore(data) {
    let score = 0;
    
    // Check velocity
    const recentTransactions = await this.getRecentTransactions(data.userId);
    if (recentTransactions.length > 5) {
      score += 0.2;
    }
    
    // Check amount
    if (data.amount > 500) {
      score += 0.3;
    }
    
    // Check device
    const knownDevice = await this.isKnownDevice(data.deviceFingerprint);
    if (!knownDevice) {
      score += 0.2;
    }
    
    // Check location
    const suspiciousLocation = await this.checkLocation(data.ipAddress);
    if (suspiciousLocation) {
      score += 0.3;
    }
    
    return Math.min(score, 1);
  }
  
  // PCI DSS compliance
  async logPaymentAttempt(data) {
    // Never log sensitive data
    const sanitized = {
      timestamp: new Date(),
      userId: data.userId,
      amount: data.amount,
      method: data.method,
      last4: data.cardNumber ? data.cardNumber.slice(-4) : null,
      result: data.result,
      ip: this.hashIP(data.ip)
    };
    
    await this.auditLog.create(sanitized);
  }
}
```

## 6. Payment Reconciliation

### Daily Reconciliation Process
```javascript
class PaymentReconciliation {
  async runDailyReconciliation() {
    const date = new Date();
    date.setDate(date.getDate() - 1); // Previous day
    
    // Get all payment providers
    const providers = ['aba', 'wing', 'acleda', 'cards'];
    
    for (const provider of providers) {
      await this.reconcileProvider(provider, date);
    }
  }
  
  async reconcileProvider(provider, date) {
    // Step 1: Get our records
    const ourTransactions = await Transaction.find({
      provider,
      createdAt: {
        $gte: startOfDay(date),
        $lte: endOfDay(date)
      }
    });
    
    // Step 2: Get provider records
    const providerTransactions = await this.fetchProviderTransactions(
      provider, 
      date
    );
    
    // Step 3: Match transactions
    const matches = [];
    const mismatches = [];
    const missing = [];
    
    for (const our of ourTransactions) {
      const theirs = providerTransactions.find(
        t => t.reference === our.reference
      );
      
      if (!theirs) {
        missing.push(our);
      } else if (theirs.amount !== our.amount) {
        mismatches.push({ our, theirs });
      } else {
        matches.push({ our, theirs });
      }
    }
    
    // Step 4: Report discrepancies
    if (mismatches.length > 0 || missing.length > 0) {
      await this.alertFinanceTeam({
        provider,
        date,
        mismatches,
        missing
      });
    }
    
    // Step 5: Update reconciliation status
    await this.updateReconciliationStatus({
      provider,
      date,
      total: ourTransactions.length,
      matched: matches.length,
      issues: mismatches.length + missing.length
    });
  }
}
```

## 7. Refund Processing

### Refund Flow
```javascript
class RefundService {
  async processRefund(orderId, reason, items = null) {
    const order = await Order.findById(orderId);
    
    if (!order || order.status !== 'completed') {
      throw new Error('Order not eligible for refund');
    }
    
    // Calculate refund amount
    const refundAmount = items 
      ? this.calculatePartialRefund(order, items)
      : order.total;
    
    // Process refund based on payment method
    let refundResult;
    switch (order.paymentMethod) {
      case 'aba_payway':
        refundResult = await this.refundABA(order, refundAmount);
        break;
      case 'wing':
        refundResult = await this.refundWing(order, refundAmount);
        break;
      case 'card':
        refundResult = await this.refundCard(order, refundAmount);
        break;
    }
    
    // Update order status
    order.refunds.push({
      amount: refundAmount,
      reason,
      processedAt: new Date(),
      transactionId: refundResult.transactionId,
      items: items || order.items
    });
    
    order.status = items ? 'partially_refunded' : 'refunded';
    await order.save();
    
    // Notify customer
    await this.sendRefundNotification(order, refundAmount);
    
    return refundResult;
  }
  
  async refundABA(order, amount) {
    return await this.abaAPI.refund({
      original_transaction: order.paymentTransactionId,
      amount,
      reason: 'Customer refund'
    });
  }
}
```

## 8. Testing Payment Integrations

### Test Card Numbers
```javascript
const testCards = {
  // Successful payments
  success: {
    visa: '4242424242424242',
    mastercard: '5555555555554444',
    localCard: '6200000000000005'
  },
  
  // Failure scenarios
  failures: {
    declined: '4000000000000002',
    insufficientFunds: '4000000000009995',
    expired: '4000000000000069',
    incorrectCvc: '4000000000000127',
    processingError: '4000000000000119'
  },
  
  // 3D Secure
  threeDSecure: {
    required: '4000002500003155',
    optional: '4000002760003184'
  }
};

// Test payment amounts for different scenarios
const testAmounts = {
  success: 50.00,
  declined: 0.02,
  error: 0.05,
  timeout: 0.10
};
```

### Integration Testing
```javascript
describe('Payment Integration Tests', () => {
  describe('ABA PayWay', () => {
    it('should create QR code for payment', async () => {
      const order = await createTestOrder();
      const result = await abaService.createPayment(order);
      
      expect(result).toHaveProperty('qr_string');
      expect(result).toHaveProperty('reference');
      expect(result.expires_at).toBeInFuture();
    });
    
    it('should handle webhook correctly', async () => {
      const webhook = {
        reference: 'TEST123',
        status: 'completed',
        amount: 50.00,
        transaction_id: 'TXN123'
      };
      
      await abaService.handleWebhook(webhook);
      
      const order = await Order.findOne({ reference: 'TEST123' });
      expect(order.status).toBe('paid');
    });
  });
});
```