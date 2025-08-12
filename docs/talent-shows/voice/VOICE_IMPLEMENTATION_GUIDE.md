# The Voice Cambodia - Implementation Guide

## Overview

The Voice Cambodia features unique blind auditions where coaches select contestants based solely on their voice, followed by battle rounds and live shows. This guide covers the specific implementation details for integrating The Voice into the Hang Meas Mobile App.

> **Framework Reference**: This implementation extends the base framework defined in [TALENT_SHOWS_FRAMEWORK.md](../../TALENT_SHOWS_FRAMEWORK.md)

---

## Show Format Specifics

### What Makes The Voice Unique

The Voice's distinctive features:
- **Blind Auditions**: Coaches can't see performers
- **The Chairs**: Iconic rotating chairs
- **Coach Teams**: Each coach builds a team
- **Battles**: Team members compete against each other
- **Steals**: Coaches can steal eliminated contestants
- **Coach Mentorship**: Intensive coaching throughout

### Show Structure

```
Blind Auditions → Battle Rounds → Knockouts → Live Playoffs → Live Shows → Finale
      ↓               ↓             ↓            ↓              ↓         ↓
  Chair Turns    Head-to-Head    Solo Songs    Top 20      Top 8    Winner
  (Team of 12)   (Team of 6)    (Team of 4)  (Top 5/coach)
```

---

## Key Features

### 1. Blind Audition System

```javascript
const blindAuditionFeature = {
  "mechanics": {
    "coach_view": "Blocked until chair turn",
    "decision_time": "During performance only",
    "chair_turn": "Irreversible decision",
    "contestant_choice": "If multiple chairs turn"
  },
  "ui_experience": {
    "coach_perspective": {
      "screen": "Blurred/blocked video",
      "audio": "Full quality audio only",
      "turn_button": "Large 'I WANT YOU' button",
      "timer": "Song duration countdown"
    },
    "viewer_perspective": {
      "split_screen": "Contestant + coaches backs",
      "chair_animation": "Dramatic rotation effect",
      "reaction_capture": "Coach surprise faces",
      "anticipation_meter": "Will they turn?"
    }
  },
  "tracking": {
    "turn_timing": "Exact second of turn",
    "turn_sequence": "Order of coach turns",
    "decision_factors": "Post-performance survey",
    "success_rate": "Turns per coach stats"
  }
}
```

### 2. Team Management

```javascript
const teamManagement = {
  "team_structure": {
    "coaches": 4,
    "team_size": {
      "after_blinds": 12,
      "after_battles": 6,
      "after_knockouts": 4,
      "live_shows": 3
    }
  },
  "coach_powers": {
    "blocks": {
      "quantity": 2,
      "usage": "Prevent other coach from getting contestant",
      "timing": "Blind auditions only"
    },
    "steals": {
      "quantity": 2,
      "usage": "Take eliminated contestant",
      "available_in": ["battles", "knockouts"]
    },
    "saves": {
      "quantity": 1,
      "usage": "Save own eliminated contestant",
      "available_in": ["battles", "knockouts"]
    }
  }
}
```

### 3. Battle Rounds

```javascript
const battleRounds = {
  "format": {
    "pairing": "Coach pairs team members",
    "song_selection": "Coach chooses song",
    "duration": "One song together",
    "outcome": "Coach picks winner"
  },
  "steal_opportunity": {
    "eligible": "Losing contestant",
    "steal_window": "30 seconds after decision",
    "max_steals": 2,
    "strategy": "Coaches build diverse teams"
  },
  "ui_elements": {
    "battle_bracket": "Tournament-style display",
    "steal_timer": "30-second countdown",
    "coach_reactions": "Picture-in-picture",
    "audience_meter": "Crowd favorite indicator"
  }
}
```

---

## Mobile App Features

### 1. Chair Turn Interface

```javascript
const ChairTurnInterface = () => {
  const [predictions, setPredictions] = useState({});
  const [turnMoments, setTurnMoments] = useState([]);
  
  return (
    <AuditionView>
      <VideoPlayer
        dualView={true}
        views={['contestant', 'coaches_back']}
        onTimeUpdate={updateTimeline}
      />
      
      <ChairStatus>
        {coaches.map(coach => (
          <CoachChair key={coach.id}>
            <ChairBack 
              turned={coach.turned}
              blocked={coach.blocked}
            />
            <CoachName>{coach.name}</CoachName>
            <TurnPrediction>
              <PredictButton
                onClick={() => predictTurn(coach.id)}
                disabled={coach.turned || prediction.submitted}
              >
                Will Turn
              </PredictButton>
            </TurnPrediction>
          </CoachChair>
        ))}
      </ChairStatus>
      
      <TurnTimeline>
        {turnMoments.map(moment => (
          <TurnMarker
            key={moment.id}
            time={moment.timestamp}
            coach={moment.coach}
            type={moment.type} // 'turn', 'block', 'almost'
          />
        ))}
      </TurnTimeline>
      
      {multipleChairs && (
        <ContestantChoice>
          <ChoicePrompt>Who will they choose?</ChoicePrompt>
          <CoachOptions>
            {turnedCoaches.map(coach => (
              <ChoiceOption 
                coach={coach}
                onClick={() => predictChoice(coach.id)}
              />
            ))}
          </CoachOptions>
        </ContestantChoice>
      )}
    </AuditionView>
  );
};
```

### 2. Team Tracker

```javascript
const TeamTracker = ({ coachId }) => {
  const team = useTeamData(coachId);
  
  return (
    <TeamDashboard>
      <CoachHeader>
        <CoachProfile>
          <CoachImage src={coach.image} />
          <CoachName>{coach.name}</CoachName>
          <TeamMotto>{coach.motto}</TeamMotto>
        </CoachProfile>
        <PowersRemaining>
          <Power type="block" remaining={coach.blocks} />
          <Power type="steal" remaining={coach.steals} />
          <Power type="save" remaining={coach.saves} />
        </PowersRemaining>
      </CoachHeader>
      
      <TeamGrid>
        {team.members.map(member => (
          <TeamMember key={member.id}>
            <MemberImage src={member.image} />
            <MemberName>{member.name}</MemberName>
            <VoiceType>{member.voiceType}</VoiceType>
            <Status>
              {member.stolen && <StolenBadge from={member.originalCoach} />}
              {member.saved && <SavedBadge />}
            </Status>
            <PerformanceHistory>
              {member.performances.map(perf => (
                <Performance
                  key={perf.id}
                  type={perf.round}
                  result={perf.result}
                  opponent={perf.opponent}
                />
              ))}
            </PerformanceHistory>
          </TeamMember>
        ))}
      </TeamGrid>
      
      <TeamStats>
        <Stat label="Win Rate" value={team.winRate} />
        <Stat label="Steal Success" value={team.stealSuccess} />
        <Stat label="Finals Reached" value={team.finalsCount} />
      </TeamStats>
    </TeamDashboard>
  );
};
```

### 3. Battle Bracket Visualization

```javascript
const BattleBracket = () => {
  return (
    <BracketContainer>
      {coaches.map(coach => (
        <CoachBracket key={coach.id}>
          <CoachHeader>{coach.name}'s Battles</CoachHeader>
          <Battles>
            {coach.battles.map(battle => (
              <BattleCard key={battle.id}>
                <Contestants>
                  <Contestant>
                    <Image src={battle.contestant1.image} />
                    <Name>{battle.contestant1.name}</Name>
                  </Contestant>
                  <VS>VS</VS>
                  <Contestant>
                    <Image src={battle.contestant2.image} />
                    <Name>{battle.contestant2.name}</Name>
                  </Contestant>
                </Contestants>
                <SongInfo>{battle.song}</SongInfo>
                <BattleResult>
                  {battle.winner && (
                    <>
                      <Winner>{battle.winner.name} WINS</Winner>
                      {battle.stolen && (
                        <Stolen>
                          {battle.loser.name} stolen by {battle.stolenBy}
                        </Stolen>
                      )}
                    </>
                  )}
                </BattleResult>
                <VoteButton onClick={() => voteForBattle(battle.id)}>
                  Vote for Your Favorite
                </VoteButton>
              </BattleCard>
            ))}
          </Battles>
        </CoachBracket>
      ))}
    </BracketContainer>
  );
};
```

---

## Technical Implementation

### 1. Database Schema Extensions

```sql
-- Voice-specific tables extending base framework
CREATE TABLE voice_coaches (
    id UUID PRIMARY KEY,
    season_id UUID REFERENCES seasons(id),
    name VARCHAR(100) NOT NULL,
    bio TEXT,
    image_url VARCHAR(500),
    team_motto VARCHAR(200),
    blocks_remaining INTEGER DEFAULT 2,
    steals_remaining INTEGER DEFAULT 2,
    saves_remaining INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE voice_teams (
    id UUID PRIMARY KEY,
    contestant_id UUID REFERENCES contestants(id),
    coach_id UUID REFERENCES voice_coaches(id),
    joined_round VARCHAR(20), -- 'blind', 'battle', 'knockout'
    original_coach_id UUID REFERENCES voice_coaches(id),
    is_stolen BOOLEAN DEFAULT false,
    is_saved BOOLEAN DEFAULT false,
    eliminated_round VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE voice_blind_auditions (
    id UUID PRIMARY KEY,
    contestant_id UUID REFERENCES contestants(id),
    episode_id UUID REFERENCES episodes(id),
    performance_order INTEGER,
    chairs_turned UUID[], -- Array of coach IDs
    blocks_used JSONB, -- {blocking_coach: blocked_coach}
    contestant_choice UUID REFERENCES voice_coaches(id),
    turn_timestamps JSONB, -- {coach_id: timestamp}
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE voice_battles (
    id UUID PRIMARY KEY,
    episode_id UUID REFERENCES episodes(id),
    coach_id UUID REFERENCES voice_coaches(id),
    contestant1_id UUID REFERENCES contestants(id),
    contestant2_id UUID REFERENCES contestants(id),
    song VARCHAR(200),
    winner_id UUID REFERENCES contestants(id),
    stolen_by UUID REFERENCES voice_coaches(id),
    saved_by UUID REFERENCES voice_coaches(id),
    audience_vote_percentage JSONB, -- {contestant1: 60, contestant2: 40}
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE voice_viewer_predictions (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    audition_id UUID REFERENCES voice_blind_auditions(id),
    chair_predictions JSONB, -- {coach_id: will_turn}
    choice_prediction UUID, -- Which coach contestant will choose
    prediction_score INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Real-Time Synchronization

```javascript
const voiceSyncService = {
  "blind_audition_sync": {
    "events": [
      "PERFORMANCE_START",
      "CHAIR_TURN",
      "BLOCK_USED",
      "PERFORMANCE_END",
      "CONTESTANT_CHOOSING",
      "CHOICE_MADE"
    ],
    "real_time_updates": {
      "chair_status": "Instant animation trigger",
      "block_notification": "Show who blocked whom",
      "prediction_results": "Score calculation",
      "choice_suspense": "Build tension"
    }
  },
  "battle_sync": {
    "events": [
      "BATTLE_START",
      "BATTLE_END",
      "WINNER_ANNOUNCED",
      "STEAL_WINDOW_OPEN",
      "STEAL_MADE",
      "SAVE_USED"
    ],
    "timing": {
      "steal_window": 30, // seconds
      "save_decision": 60 // seconds
    }
  }
}
```

### 3. API Endpoints

```javascript
// Voice-specific endpoints
const voiceRoutes = {
  // Teams
  "GET /api/v1/shows/voice/teams": "Get all teams with members",
  "GET /api/v1/shows/voice/teams/:coachId": "Get specific team",
  "GET /api/v1/shows/voice/teams/:coachId/stats": "Team performance stats",
  
  // Blind Auditions
  "GET /api/v1/shows/voice/auditions": "List all blind auditions",
  "POST /api/v1/shows/voice/auditions/:id/predict": "Submit chair predictions",
  "GET /api/v1/shows/voice/auditions/:id/results": "Get turn results",
  
  // Battles
  "GET /api/v1/shows/voice/battles": "Get all battles",
  "GET /api/v1/shows/voice/battles/:coachId": "Get coach's battles",
  "POST /api/v1/shows/voice/battles/:id/vote": "Vote for favorite",
  
  // Coach Powers
  "GET /api/v1/shows/voice/coaches/:id/powers": "Remaining powers",
  "POST /api/v1/shows/voice/block": "Use block (admin)",
  "POST /api/v1/shows/voice/steal": "Use steal (admin)",
  
  // Predictions & Gaming
  "GET /api/v1/shows/voice/predictions/leaderboard": "Top predictors",
  "GET /api/v1/shows/voice/predictions/my-score": "User's prediction score"
}
```

---

## Unique Monetization

### Voice-Specific Revenue Streams

```javascript
const voiceMonetization = {
  "team_packages": {
    "team_pass": {
      "price": "$9.99/season",
      "benefits": [
        "All team member performances",
        "Exclusive coach content",
        "Team merchandise discount",
        "Vote multiplier for team"
      ]
    },
    "coach_cam": {
      "price": "$4.99/month",
      "features": [
        "Coach reactions during blinds",
        "Strategy discussions",
        "Behind-scenes coaching"
      ]
    }
  },
  "prediction_games": {
    "chair_turn_predictor": {
      "entry": "$0.99",
      "prize_pool": "Split among perfect predictors",
      "bonus": "Meet winning contestant"
    },
    "bracket_challenge": {
      "entry": "$4.99",
      "prize": "VIP finale tickets"
    }
  },
  "interactive_features": {
    "virtual_coach": {
      "price": "$19.99",
      "includes": [
        "AI voice analysis",
        "Personalized coaching tips",
        "Compare with contestants"
      ]
    },
    "steal_predictor": {
      "price": "$1.99/battle round",
      "reward": "Exclusive battle footage"
    }
  }
}
```

---

## Marketing & Engagement

### Social Features

```javascript
const voiceSocialFeatures = {
  "team_loyalty": {
    "team_badges": "Show support for coach",
    "team_challenges": "Inter-team competitions",
    "team_chat": "Exclusive team supporter chat"
  },
  "viral_moments": {
    "chair_turn_compilations": "Best blind audition turns",
    "block_moments": "Dramatic block usage",
    "steal_battles": "Intense steal moments",
    "shock_choices": "Unexpected contestant choices"
  },
  "user_generated": {
    "cover_challenges": "#VoiceCambodiaCover",
    "coach_impressions": "#BeLikeCoach",
    "prediction_videos": "React to predictions"
  }
}
```

### Gamification

```javascript
const voiceGamification = {
  "prediction_league": {
    "scoring": {
      "chair_turn_correct": 10,
      "all_chairs_perfect": 50,
      "contestant_choice": 25,
      "battle_winner": 15,
      "steal_prediction": 30
    },
    "rewards": {
      "level_1": "Voice predictor badge",
      "level_2": "Exclusive content unlock",
      "level_3": "Virtual meet with coaches",
      "level_4": "VIP show tickets"
    }
  },
  "team_supporter_rewards": {
    "loyalty_points": "For consistent team support",
    "team_victories": "Points when team member wins",
    "engagement": "Sharing, voting, discussing"
  }
}
```

---

## Analytics Focus

### Voice-Specific Metrics

```javascript
const voiceAnalytics = {
  "audition_metrics": {
    "chair_turn_rate": "Percentage getting chairs",
    "multi_chair_rate": "Multiple chair turns",
    "block_effectiveness": "Successful block impact",
    "coach_preferences": "Turn patterns by genre"
  },
  "team_metrics": {
    "team_diversity": "Voice types per team",
    "steal_success_rate": "Stolen contestants' progress",
    "coach_win_rate": "Historical success",
    "fan_loyalty": "Team supporter retention"
  },
  "engagement_metrics": {
    "prediction_participation": "Users making predictions",
    "team_affiliation": "Users choosing teams",
    "battle_voting": "Audience participation",
    "replay_rate": "Most rewatched moments"
  }
}
```

---

## Implementation Phases

### Phase 1: Core Voice Features (Month 1)
- Blind audition interface
- Basic team display
- Chair turn animations

### Phase 2: Battle System (Month 2)
- Battle bracket visualization
- Steal/save mechanics
- Team management tools

### Phase 3: Predictions & Gaming (Month 3)
- Chair turn predictions
- Battle outcome predictions
- Leaderboards and rewards

### Phase 4: Enhanced Features (Month 4)
- Coach cam integration
- Advanced team analytics
- Social team features

---

## Success KPIs

- **Prediction Participation**: 60% of viewers predict
- **Team Affiliation**: 80% choose a team to support
- **Coach Content Views**: 2M+ views per episode
- **Revenue per User**: $3.50 per active user
- **Social Shares**: 100K+ shares per episode

The Voice Cambodia's unique format with coaches, teams, and dramatic moments like chair turns and steals creates compelling interactive opportunities that differentiate it from other talent shows in the Hang Meas ecosystem.