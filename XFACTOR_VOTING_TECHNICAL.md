# XFactor Voting System - Complete Technical Implementation

## 1. Voting System Architecture

### High-Level System Design
```
┌─────────────────────────────────────────────────────────────┐
│                     TV Broadcast System                      │
│                  (Live Show Feed + Timecodes)                │
└─────────────────────────┬───────────────────────────────────┘
                          │ Sync Signal
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   Broadcast Sync Service                     │
│              (Redis Pub/Sub + WebSocket Server)              │
└─────────────────────────┬───────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Mobile Apps │    │  Web Apps   │    │ SMS Gateway │
│ (iOS/Andr)  │    │  (Browser)  │    │   (Telco)   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┴──────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│                  (Kong / AWS API Gateway)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│Voting Service │ │Results Service│ │Analytics Svc  │
│  (Write-heavy)│ │ (Read-heavy)  │ │ (Real-time)   │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Redis Cluster │ │  PostgreSQL   │ │ ElasticSearch │
│ (Vote Queue)  │ │  (Primary)    │ │  (Analytics)  │
└───────────────┘ └───────────────┘ └───────────────┘
```

## 2. Real-Time Synchronization

### Broadcast Sync Implementation
```javascript
// Broadcast Sync Service
class BroadcastSyncService {
  constructor() {
    this.redis = new Redis.Cluster(config.redis.nodes);
    this.pubsub = new Redis(config.redis.pubsub);
    this.wss = new WebSocketServer({ port: 8080 });
    this.currentShow = null;
    this.votingState = 'IDLE';
  }

  // Receive sync signals from TV broadcast system
  async handleBroadcastSignal(signal) {
    const { event, timestamp, data } = signal;
    
    switch (event) {
      case 'SHOW_START':
        await this.startShow(data.showId, data.episode);
        break;
        
      case 'PERFORMANCE_START':
        await this.startPerformance(data.contestantId, data.songName);
        break;
        
      case 'PERFORMANCE_END':
        await this.endPerformance(data.contestantId);
        break;
        
      case 'VOTING_OPEN':
        await this.openVoting(data.duration);
        break;
        
      case 'VOTING_CLOSE':
        await this.closeVoting();
        break;
        
      case 'SHOW_END':
        await this.endShow();
        break;
    }
    
    // Broadcast state change to all connected clients
    this.broadcastStateUpdate();
  }
  
  async openVoting(duration = 1800) { // 30 minutes default
    this.votingState = 'OPEN';
    const votingWindow = {
      state: 'OPEN',
      opensAt: new Date(),
      closesAt: new Date(Date.now() + duration * 1000),
      episode: this.currentShow.episode,
      contestants: await this.getActiveContestants()
    };
    
    // Store in Redis with TTL
    await this.redis.setex(
      'voting:current',
      duration,
      JSON.stringify(votingWindow)
    );
    
    // Publish to all subscribers
    await this.pubsub.publish('voting:state', JSON.stringify({
      event: 'VOTING_OPEN',
      data: votingWindow
    }));
    
    // Start countdown timer
    this.startVotingTimer(duration);
  }
  
  startVotingTimer(duration) {
    let remaining = duration;
    
    this.votingTimer = setInterval(async () => {
      remaining -= 1;
      
      // Broadcast countdown updates
      if (remaining % 10 === 0) { // Every 10 seconds
        await this.broadcastTimeUpdate(remaining);
      }
      
      // Warning at 1 minute
      if (remaining === 60) {
        await this.broadcastWarning('1_MINUTE');
      }
      
      // Final countdown
      if (remaining <= 10 && remaining > 0) {
        await this.broadcastCountdown(remaining);
      }
      
      // Close voting
      if (remaining === 0) {
        clearInterval(this.votingTimer);
        await this.closeVoting();
      }
    }, 1000);
  }
  
  broadcastStateUpdate() {
    const state = {
      show: this.currentShow,
      voting: this.votingState,
      timestamp: new Date()
    };
    
    // Send to all WebSocket clients
    this.wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({
          type: 'STATE_UPDATE',
          data: state
        }));
      }
    });
  }
}
```

### Client-Side Sync Handler
```javascript
// Mobile App - Voting Sync Handler
class VotingSyncHandler {
  constructor() {
    this.ws = null;
    this.votingState = null;
    this.reconnectAttempts = 0;
  }
  
  connect() {
    this.ws = new WebSocket('wss://api.hangmeas.com/voting/sync');
    
    this.ws.onopen = () => {
      console.log('Connected to voting sync');
      this.reconnectAttempts = 0;
      this.authenticate();
    };
    
    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleSyncMessage(message);
    };
    
    this.ws.onclose = () => {
      this.handleDisconnect();
    };
  }
  
  handleSyncMessage(message) {
    switch (message.type) {
      case 'STATE_UPDATE':
        this.updateVotingState(message.data);
        break;
        
      case 'VOTING_OPEN':
        this.enableVoting(message.data);
        break;
        
      case 'TIME_UPDATE':
        this.updateCountdown(message.data.remaining);
        break;
        
      case 'VOTING_CLOSED':
        this.disableVoting();
        break;
        
      case 'RESULTS_READY':
        this.showResults(message.data);
        break;
    }
  }
  
  enableVoting(votingData) {
    // Update UI to show voting interface
    this.votingState = 'OPEN';
    
    // Show contestants
    votingData.contestants.forEach(contestant => {
      this.renderContestantVoteCard(contestant);
    });
    
    // Start local countdown
    this.startLocalCountdown(votingData.closesAt);
    
    // Enable vote buttons
    this.enableVoteButtons();
    
    // Show remaining votes
    this.updateVoteBalance();
  }
}
```

## 3. Vote Processing Pipeline

### High-Performance Vote Ingestion
```javascript
// Vote Processing Service
class VoteProcessingService {
  constructor() {
    this.voteQueue = new BullQueue('votes', {
      redis: config.redis.queue
    });
    
    this.rateLimiter = new RateLimiter({
      points: 100, // votes
      duration: 60, // per minute
      keyPrefix: 'vote_limit'
    });
    
    this.initializeWorkers();
  }
  
  async submitVote(voteData) {
    const { userId, contestantId, voteType, voteCount, metadata } = voteData;
    
    try {
      // Step 1: Rate limiting
      await this.checkRateLimit(userId);
      
      // Step 2: Validate vote
      await this.validateVote(voteData);
      
      // Step 3: Check vote balance
      if (voteType === 'paid') {
        await this.checkVoteBalance(userId, voteCount);
      }
      
      // Step 4: Queue vote for processing
      const job = await this.voteQueue.add('process_vote', {
        voteId: uuidv4(),
        userId,
        contestantId,
        voteType,
        voteCount,
        timestamp: new Date(),
        metadata: {
          ...metadata,
          ip: this.hashIP(metadata.ip),
          deviceId: metadata.deviceId,
          appVersion: metadata.appVersion
        }
      }, {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000
        }
      });
      
      // Step 5: Return immediate confirmation
      return {
        success: true,
        voteId: job.data.voteId,
        status: 'queued'
      };
      
    } catch (error) {
      if (error.code === 'RATE_LIMIT') {
        throw new VotingError('Too many votes, please slow down', 429);
      }
      throw error;
    }
  }
  
  async validateVote(voteData) {
    // Check voting window is open
    const votingWindow = await this.redis.get('voting:current');
    if (!votingWindow) {
      throw new VotingError('Voting is not open', 'VOTING_CLOSED');
    }
    
    const window = JSON.parse(votingWindow);
    if (new Date() > new Date(window.closesAt)) {
      throw new VotingError('Voting has ended', 'VOTING_ENDED');
    }
    
    // Validate contestant
    if (!window.contestants.includes(voteData.contestantId)) {
      throw new VotingError('Invalid contestant', 'INVALID_CONTESTANT');
    }
    
    // Check free vote limits
    if (voteData.voteType === 'free') {
      const freeVotesUsed = await this.redis.get(
        `votes:free:${voteData.userId}:${window.episode}`
      );
      
      if (freeVotesUsed >= 5) {
        throw new VotingError('Free votes exhausted', 'NO_FREE_VOTES');
      }
    }
  }
  
  initializeWorkers() {
    // Process votes in batches for efficiency
    this.voteQueue.process('process_vote', 100, async (jobs) => {
      const votes = jobs.map(job => job.data);
      
      // Batch insert into database
      await this.batchInsertVotes(votes);
      
      // Update vote counts in Redis
      await this.updateVoteCounts(votes);
      
      // Deduct vote balances
      await this.deductVoteBalances(votes);
      
      // Track analytics
      await this.trackVoteAnalytics(votes);
      
      return votes.map(v => ({ voteId: v.voteId, status: 'processed' }));
    });
  }
  
  async batchInsertVotes(votes) {
    const voteDocs = votes.map(vote => ({
      vote_id: vote.voteId,
      user_id: vote.userId,
      contestant_id: vote.contestantId,
      episode_id: vote.episodeId,
      vote_type: vote.voteType,
      vote_count: vote.voteCount,
      created_at: vote.timestamp,
      metadata: vote.metadata
    }));
    
    // Use PostgreSQL COPY for fast insertion
    await this.db.votes.bulkInsert(voteDocs);
  }
  
  async updateVoteCounts(votes) {
    const pipeline = this.redis.pipeline();
    
    // Group votes by contestant
    const votesByContestant = votes.reduce((acc, vote) => {
      acc[vote.contestantId] = (acc[vote.contestantId] || 0) + vote.voteCount;
      return acc;
    }, {});
    
    // Update counts atomically
    Object.entries(votesByContestant).forEach(([contestantId, count]) => {
      pipeline.incrby(`votes:count:${contestantId}`, count);
    });
    
    await pipeline.exec();
  }
}
```

### Vote Aggregation Service
```javascript
// Real-time Vote Aggregation
class VoteAggregationService {
  constructor() {
    this.cache = new NodeCache({ stdTTL: 1 }); // 1 second cache
    this.subscribers = new Set();
  }
  
  async getVoteResults() {
    // Check cache first
    const cached = this.cache.get('results');
    if (cached) return cached;
    
    // Get current voting window
    const window = await this.getCurrentVotingWindow();
    if (!window) return null;
    
    // Get vote counts from Redis
    const pipeline = this.redis.pipeline();
    window.contestants.forEach(contestantId => {
      pipeline.get(`votes:count:${contestantId}`);
    });
    
    const counts = await pipeline.exec();
    
    // Calculate results
    const results = window.contestants.map((contestantId, index) => {
      const voteCount = parseInt(counts[index][1] || 0);
      return {
        contestantId,
        voteCount,
        percentage: 0 // calculated below
      };
    });
    
    // Calculate percentages
    const totalVotes = results.reduce((sum, r) => sum + r.voteCount, 0);
    results.forEach(r => {
      r.percentage = totalVotes > 0 
        ? ((r.voteCount / totalVotes) * 100).toFixed(1)
        : 0;
    });
    
    // Sort by votes
    results.sort((a, b) => b.voteCount - a.voteCount);
    
    // Add rankings
    results.forEach((r, index) => {
      r.rank = index + 1;
    });
    
    // Cache results
    this.cache.set('results', results);
    
    return results;
  }
  
  // Stream results to clients
  streamResults() {
    setInterval(async () => {
      const results = await this.getVoteResults();
      
      // Broadcast to all subscribers
      this.subscribers.forEach(subscriber => {
        subscriber.send(JSON.stringify({
          type: 'VOTE_UPDATE',
          data: results,
          timestamp: new Date()
        }));
      });
    }, 1000); // Update every second
  }
}
```

## 4. Anti-Fraud System

### Fraud Detection Engine
```javascript
class VoteFraudDetector {
  constructor() {
    this.ml = new MachineLearningService();
    this.patterns = new FraudPatterns();
  }
  
  async analyzeVote(voteData) {
    const fraudScore = await this.calculateFraudScore(voteData);
    
    if (fraudScore > 0.8) {
      // High probability of fraud
      await this.blockVote(voteData);
      await this.flagUser(voteData.userId);
      return { action: 'BLOCK', reason: 'Fraud detected' };
    }
    
    if (fraudScore > 0.5) {
      // Suspicious activity
      await this.flagForReview(voteData);
      return { action: 'REVIEW', reason: 'Suspicious pattern' };
    }
    
    return { action: 'ALLOW' };
  }
  
  async calculateFraudScore(voteData) {
    const features = await this.extractFeatures(voteData);
    
    // Rule-based checks
    let score = 0;
    
    // 1. Velocity check
    const recentVotes = await this.getRecentVotes(voteData.userId, 60);
    if (recentVotes > 50) score += 0.3;
    
    // 2. Device fingerprint
    const deviceVotes = await this.getDeviceVotes(voteData.metadata.deviceId);
    if (deviceVotes.uniqueUsers > 3) score += 0.2;
    
    // 3. IP address check
    const ipInfo = await this.getIPInfo(voteData.metadata.ip);
    if (ipInfo.isVPN || ipInfo.isProxy) score += 0.4;
    if (ipInfo.country !== 'KH') score += 0.3;
    
    // 4. Pattern detection
    const votingPattern = await this.analyzePattern(voteData.userId);
    if (votingPattern.isBotLike) score += 0.5;
    
    // 5. ML-based scoring
    const mlScore = await this.ml.predictFraud(features);
    score = (score + mlScore) / 2;
    
    return Math.min(score, 1);
  }
  
  async analyzePattern(userId) {
    const votes = await this.getVoteHistory(userId, 3600); // Last hour
    
    // Check for bot-like patterns
    const timeDiffs = [];
    for (let i = 1; i < votes.length; i++) {
      timeDiffs.push(votes[i].timestamp - votes[i-1].timestamp);
    }
    
    // Calculate standard deviation
    const avgDiff = timeDiffs.reduce((a, b) => a + b, 0) / timeDiffs.length;
    const variance = timeDiffs.reduce((sum, diff) => 
      sum + Math.pow(diff - avgDiff, 2), 0) / timeDiffs.length;
    const stdDev = Math.sqrt(variance);
    
    // Low standard deviation indicates bot-like behavior
    const isBotLike = stdDev < 1000; // Less than 1 second variation
    
    return { isBotLike, avgInterval: avgDiff, stdDev };
  }
  
  async blockVote(voteData) {
    // Add to blocked votes
    await this.redis.sadd('votes:blocked', voteData.voteId);
    
    // Log for audit
    await this.auditLog.create({
      action: 'VOTE_BLOCKED',
      voteId: voteData.voteId,
      userId: voteData.userId,
      reason: 'Fraud detection',
      timestamp: new Date()
    });
  }
}
```

### Geographic Validation
```javascript
class GeographicValidator {
  async validateVoteLocation(voteData) {
    const { ip, location } = voteData.metadata;
    
    // Get IP geolocation
    const ipLocation = await this.geoIP.lookup(ip);
    
    // Check if in Cambodia
    if (ipLocation.country !== 'KH') {
      // Check if user is traveling
      const userProfile = await this.getUserProfile(voteData.userId);
      if (!userProfile.isTraveling) {
        return {
          valid: false,
          reason: 'Voting only allowed from Cambodia'
        };
      }
    }
    
    // Validate GPS coordinates if provided
    if (location && location.lat && location.lng) {
      const distance = this.calculateDistance(
        ipLocation,
        location
      );
      
      // If GPS and IP location differ significantly
      if (distance > 100) { // 100km
        return {
          valid: false,
          reason: 'Location mismatch'
        };
      }
    }
    
    return { valid: true };
  }
}
```

## 5. SMS Voting Integration

### SMS Gateway Integration
```javascript
class SMSVotingService {
  constructor() {
    this.shortCode = '1255';
    this.providers = {
      smart: new SmartAPI(),
      cellcard: new CellcardAPI(),
      metfone: new MetfoneAPI()
    };
  }
  
  async processSMSVote(sms) {
    const { from, to, message, provider, receivedAt } = sms;
    
    try {
      // Parse vote
      const vote = this.parseVoteMessage(message);
      if (!vote) {
        await this.sendSMSReply(from, 
          'Invalid format. Send contestant number (1-12) to 1255'
        );
        return;
      }
      
      // Check if voting is open
      const votingWindow = await this.getVotingWindow();
      if (!votingWindow || votingWindow.state !== 'OPEN') {
        await this.sendSMSReply(from,
          'Voting is closed. Watch XFactor every Saturday!'
        );
        return;
      }
      
      // Process vote
      const result = await this.submitSMSVote({
        phoneNumber: from,
        contestantNumber: vote.contestantNumber,
        provider,
        timestamp: receivedAt
      });
      
      // Send confirmation
      await this.sendVoteConfirmation(from, result);
      
      // Bill user
      await this.billSMSVote(from, provider);
      
    } catch (error) {
      logger.error('SMS vote processing failed:', error);
      await this.sendSMSReply(from, 
        'Sorry, your vote could not be processed. Please try again.'
      );
    }
  }
  
  parseVoteMessage(message) {
    const cleaned = message.trim().toUpperCase();
    
    // Simple number format: "1", "2", etc.
    if (/^[1-9]$|^1[0-2]$/.test(cleaned)) {
      return { contestantNumber: parseInt(cleaned) };
    }
    
    // With prefix: "VOTE 1", "V1"
    const match = cleaned.match(/^(?:VOTE\s*|V)([1-9]|1[0-2])$/);
    if (match) {
      return { contestantNumber: parseInt(match[1]) };
    }
    
    return null;
  }
  
  async submitSMSVote(data) {
    // Find or create user by phone
    let user = await User.findOne({ phone: data.phoneNumber });
    if (!user) {
      user = await User.create({
        phone: data.phoneNumber,
        source: 'sms_voting',
        createdAt: new Date()
      });
    }
    
    // Map contestant number to ID
    const contestant = await this.getContestantByNumber(
      data.contestantNumber
    );
    
    // Submit vote
    const vote = await this.voteService.submitVote({
      userId: user.id,
      contestantId: contestant.id,
      voteType: 'sms',
      voteCount: 1,
      metadata: {
        phoneNumber: data.phoneNumber,
        provider: data.provider,
        channel: 'sms'
      }
    });
    
    return {
      contestant: contestant.name,
      voteCount: await this.getVoteCount(contestant.id)
    };
  }
  
  async sendVoteConfirmation(phoneNumber, result) {
    const message = `Thank you! Your vote for ${result.contestant} has been counted. ` +
                   `They now have ${result.voteCount.toLocaleString()} votes. ` +
                   `Vote again or watch live on Hang Meas TV!`;
    
    await this.sendSMS(phoneNumber, message);
  }
  
  async billSMSVote(phoneNumber, provider) {
    const charge = {
      phoneNumber,
      amount: 0.50, // $0.50 per vote
      currency: 'USD',
      description: 'XFactor Vote',
      provider
    };
    
    await this.providers[provider].chargePremiumSMS(charge);
  }
}
```

## 6. Vote Package Management

### In-App Purchase Integration
```javascript
class VotePackageService {
  constructor() {
    this.packages = [
      {
        id: 'vote_pack_10',
        votes: 10,
        price: 0.99,
        bonus: 0,
        sku: {
          ios: 'com.hangmeas.votes.10',
          android: 'votes_10'
        }
      },
      {
        id: 'vote_pack_50',
        votes: 50,
        price: 3.99,
        bonus: 10,
        sku: {
          ios: 'com.hangmeas.votes.50',
          android: 'votes_50'
        }
      },
      {
        id: 'vote_pack_200',
        votes: 200,
        price: 9.99,
        bonus: 50,
        sku: {
          ios: 'com.hangmeas.votes.200',
          android: 'votes_200'
        }
      },
      {
        id: 'vote_pack_1000',
        votes: 1000,
        price: 39.99,
        bonus: 200,
        sku: {
          ios: 'com.hangmeas.votes.1000',
          android: 'votes_1000'
        }
      }
    ];
  }
  
  async purchaseVotePackage(userId, packageId, receipt) {
    const package = this.packages.find(p => p.id === packageId);
    if (!package) {
      throw new Error('Invalid package');
    }
    
    // Verify purchase with platform
    const verified = await this.verifyPurchase(receipt);
    if (!verified) {
      throw new Error('Invalid purchase receipt');
    }
    
    // Prevent duplicate processing
    const existing = await Purchase.findOne({
      receiptId: receipt.transactionId
    });
    if (existing) {
      return { success: true, duplicate: true };
    }
    
    // Record purchase
    const purchase = await Purchase.create({
      userId,
      packageId,
      amount: package.price,
      votes: package.votes + package.bonus,
      receiptId: receipt.transactionId,
      platform: receipt.platform,
      purchasedAt: new Date()
    });
    
    // Credit votes to user
    await this.creditVotes(userId, package.votes + package.bonus);
    
    // Track analytics
    await this.analytics.track('vote_package_purchased', {
      userId,
      packageId,
      revenue: package.price,
      votes: package.votes + package.bonus
    });
    
    return {
      success: true,
      votes: package.votes + package.bonus,
      newBalance: await this.getVoteBalance(userId)
    };
  }
  
  async verifyPurchase(receipt) {
    if (receipt.platform === 'ios') {
      return await this.verifyApplePurchase(receipt);
    } else if (receipt.platform === 'android') {
      return await this.verifyGooglePurchase(receipt);
    }
    throw new Error('Unknown platform');
  }
  
  async creditVotes(userId, votes) {
    await this.redis.incrby(`votes:balance:${userId}`, votes);
    
    // Also store in database for persistence
    await VoteBalance.upsert({
      userId,
      balance: Sequelize.literal(`balance + ${votes}`),
      lastUpdated: new Date()
    });
  }
}
```

## 7. Live Results Display

### Real-Time Results Broadcasting
```javascript
class LiveResultsService {
  constructor() {
    this.updateInterval = 1000; // 1 second
    this.clients = new Map();
  }
  
  startBroadcasting() {
    setInterval(async () => {
      const results = await this.calculateResults();
      this.broadcastResults(results);
    }, this.updateInterval);
  }
  
  async calculateResults() {
    const window = await this.getVotingWindow();
    if (!window || window.state !== 'OPEN') {
      return null;
    }
    
    // Get all contestants
    const contestants = await Contestant.findAll({
      where: { episodeId: window.episodeId }
    });
    
    // Get vote counts
    const voteCounts = await Promise.all(
      contestants.map(async (contestant) => {
        const count = await this.redis.get(`votes:count:${contestant.id}`) || 0;
        return {
          contestantId: contestant.id,
          contestantName: contestant.name,
          contestantPhoto: contestant.photoUrl,
          voteCount: parseInt(count),
          trend: await this.calculateTrend(contestant.id)
        };
      })
    );
    
    // Calculate percentages and rankings
    const totalVotes = voteCounts.reduce((sum, vc) => sum + vc.voteCount, 0);
    
    const results = voteCounts
      .map(vc => ({
        ...vc,
        percentage: totalVotes > 0 
          ? ((vc.voteCount / totalVotes) * 100).toFixed(1)
          : 0,
        formattedCount: this.formatVoteCount(vc.voteCount)
      }))
      .sort((a, b) => b.voteCount - a.voteCount)
      .map((vc, index) => ({
        ...vc,
        rank: index + 1,
        change: this.calculateRankChange(vc.contestantId, index + 1)
      }));
    
    return {
      results,
      totalVotes,
      lastUpdated: new Date(),
      timeRemaining: await this.getTimeRemaining()
    };
  }
  
  async calculateTrend(contestantId) {
    // Get votes in last 5 minutes vs previous 5 minutes
    const recent = await this.getVotesInPeriod(contestantId, 5);
    const previous = await this.getVotesInPeriod(contestantId, 10, 5);
    
    if (previous === 0) return 'stable';
    
    const change = ((recent - previous) / previous) * 100;
    
    if (change > 20) return 'rising_fast';
    if (change > 5) return 'rising';
    if (change < -20) return 'falling_fast';
    if (change < -5) return 'falling';
    
    return 'stable';
  }
  
  formatVoteCount(count) {
    if (count >= 1000000) {
      return (count / 1000000).toFixed(1) + 'M';
    }
    if (count >= 1000) {
      return (count / 1000).toFixed(1) + 'K';
    }
    return count.toLocaleString();
  }
  
  broadcastResults(results) {
    const message = JSON.stringify({
      type: 'LIVE_RESULTS',
      data: results
    });
    
    // Send to all connected clients
    this.clients.forEach((client, clientId) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(message);
      } else {
        // Clean up disconnected clients
        this.clients.delete(clientId);
      }
    });
  }
}
```

### Results Visualization Component
```javascript
// React Native Component for Live Results
const LiveVotingResults = () => {
  const [results, setResults] = useState(null);
  const [isConnected, setIsConnected] = useState(false);
  
  useEffect(() => {
    const ws = new WebSocket('wss://api.hangmeas.com/voting/results');
    
    ws.onopen = () => {
      setIsConnected(true);
    };
    
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      if (message.type === 'LIVE_RESULTS') {
        setResults(message.data);
      }
    };
    
    ws.onclose = () => {
      setIsConnected(false);
      // Implement reconnection logic
    };
    
    return () => ws.close();
  }, []);
  
  if (!results) {
    return <LoadingIndicator />;
  }
  
  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>Live Voting Results</Text>
        <CountdownTimer seconds={results.timeRemaining} />
      </View>
      
      <Text style={styles.totalVotes}>
        Total Votes: {results.totalVotes.toLocaleString()}
      </Text>
      
      <FlatList
        data={results.results}
        keyExtractor={item => item.contestantId}
        renderItem={({ item }) => (
          <VoteResultCard
            rank={item.rank}
            name={item.contestantName}
            photo={item.contestantPhoto}
            voteCount={item.formattedCount}
            percentage={item.percentage}
            trend={item.trend}
            change={item.change}
          />
        )}
      />
    </View>
  );
};

const VoteResultCard = ({ rank, name, photo, voteCount, percentage, trend, change }) => {
  const getTrendIcon = () => {
    switch (trend) {
      case 'rising_fast': return '🚀';
      case 'rising': return '📈';
      case 'falling_fast': return '📉';
      case 'falling': return '⬇️';
      default: return '➡️';
    }
  };
  
  const getRankBadge = () => {
    switch (rank) {
      case 1: return '👑';
      case 2: return '🥈';
      case 3: return '🥉';
      default: return `#${rank}`;
    }
  };
  
  return (
    <View style={styles.resultCard}>
      <View style={styles.rankContainer}>
        <Text style={styles.rankBadge}>{getRankBadge()}</Text>
        {change !== 0 && (
          <Text style={[styles.change, { color: change > 0 ? 'green' : 'red' }]}>
            {change > 0 ? `+${change}` : change}
          </Text>
        )}
      </View>
      
      <Image source={{ uri: photo }} style={styles.contestantPhoto} />
      
      <View style={styles.info}>
        <Text style={styles.name}>{name}</Text>
        <View style={styles.voteInfo}>
          <Text style={styles.voteCount}>{voteCount} votes</Text>
          <Text style={styles.trend}>{getTrendIcon()}</Text>
        </View>
        <ProgressBar 
          progress={parseFloat(percentage) / 100} 
          color={rank === 1 ? '#FFD700' : '#4CAF50'}
        />
        <Text style={styles.percentage}>{percentage}%</Text>
      </View>
    </View>
  );
};
```

## 8. Post-Voting Analytics

### Analytics Dashboard
```javascript
class VotingAnalytics {
  async generateEpisodeReport(episodeId) {
    const report = {
      episodeId,
      generatedAt: new Date(),
      summary: await this.getSummary(episodeId),
      timeline: await this.getVotingTimeline(episodeId),
      demographics: await this.getDemographics(episodeId),
      geographic: await this.getGeographicBreakdown(episodeId),
      revenue: await this.getRevenueAnalytics(episodeId),
      engagement: await this.getEngagementMetrics(episodeId)
    };
    
    return report;
  }
  
  async getSummary(episodeId) {
    const votes = await Vote.findAll({
      where: { episodeId },
      attributes: [
        [Sequelize.fn('COUNT', Sequelize.col('*')), 'totalVotes'],
        [Sequelize.fn('COUNT', Sequelize.fn('DISTINCT', Sequelize.col('user_id'))), 'uniqueVoters'],
        [Sequelize.fn('SUM', Sequelize.literal("CASE WHEN vote_type = 'free' THEN 1 ELSE 0 END")), 'freeVotes'],
        [Sequelize.fn('SUM', Sequelize.literal("CASE WHEN vote_type = 'paid' THEN 1 ELSE 0 END")), 'paidVotes'],
        [Sequelize.fn('SUM', Sequelize.literal("CASE WHEN vote_type = 'sms' THEN 1 ELSE 0 END")), 'smsVotes']
      ]
    });
    
    return votes[0];
  }
  
  async getVotingTimeline(episodeId) {
    // Get votes per minute during voting window
    const timeline = await Vote.findAll({
      where: { episodeId },
      attributes: [
        [Sequelize.fn('DATE_TRUNC', 'minute', Sequelize.col('created_at')), 'minute'],
        [Sequelize.fn('COUNT', Sequelize.col('*')), 'votes']
      ],
      group: ['minute'],
      order: [['minute', 'ASC']]
    });
    
    return timeline.map(t => ({
      time: t.minute,
      votes: parseInt(t.votes),
      rate: parseInt(t.votes) // votes per minute
    }));
  }
}
```

## 9. Testing & Load Simulation

### Load Testing Script
```javascript
// Load test for voting system
const loadTest = async () => {
  const users = 10000; // Simulate 10k concurrent users
  const votingDuration = 1800; // 30 minutes
  const avgVotesPerUser = 10;
  
  console.log(`Starting load test: ${users} users voting over ${votingDuration}s`);
  
  const results = await Promise.all(
    Array(users).fill(null).map(async (_, index) => {
      const userId = `test_user_${index}`;
      const votes = [];
      
      // Distribute votes over time
      for (let i = 0; i < avgVotesPerUser; i++) {
        const delay = Math.random() * votingDuration * 1000;
        
        setTimeout(async () => {
          try {
            const vote = await submitVote({
              userId,
              contestantId: getRandomContestant(),
              voteType: Math.random() > 0.7 ? 'paid' : 'free',
              voteCount: 1
            });
            votes.push(vote);
          } catch (error) {
            console.error(`Vote failed for ${userId}:`, error.message);
          }
        }, delay);
      }
      
      return { userId, votes };
    })
  );
  
  console.log('Load test initiated, monitoring...');
};
```