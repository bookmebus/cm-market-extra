# Talent Shows Framework - Multi-Show Architecture

## Overview

This framework provides a unified architecture for integrating multiple talent shows (XFactor, Cambodia's Got Talent, The Voice Cambodia) into the Hang Meas Mobile App, maximizing code reuse while allowing show-specific customizations.

> **Show-Specific Guides**: 
> - XFactor: See [XFACTOR_COMPLETE_GUIDE.md](./talent-shows/xfactor/XFACTOR_COMPLETE_GUIDE.md) and [XFACTOR_TECHNICAL_IMPLEMENTATION.md](./talent-shows/xfactor/XFACTOR_TECHNICAL_IMPLEMENTATION.md)
> - Cambodia's Got Talent: See [CGT_IMPLEMENTATION_GUIDE.md](./talent-shows/cgt/CGT_IMPLEMENTATION_GUIDE.md) (to be created)
> - The Voice Cambodia: See [VOICE_IMPLEMENTATION_GUIDE.md](./talent-shows/voice/VOICE_IMPLEMENTATION_GUIDE.md) (to be created)

---

## Table of Contents

1. [Show Comparison Matrix](#show-comparison-matrix)
2. [Unified Architecture](#unified-architecture)
3. [Common Components](#common-components)
4. [Show-Specific Configurations](#show-specific-configurations)
5. [Database Design](#database-design)
6. [API Structure](#api-structure)
7. [UI/UX Patterns](#uiux-patterns)
8. [Business Logic](#business-logic)

---

## Show Comparison Matrix

### Core Differences

| Feature | XFactor | Cambodia's Got Talent | The Voice Cambodia |
|---------|---------|----------------------|-------------------|
| **Format** | Singing only | All talents (magic, dance, etc.) | Singing only |
| **Age Groups** | Categories (Boys/Girls/Over 25/Groups) | All ages mixed | All ages mixed |
| **Judges** | 4 judges with categories | 3-4 judges, no categories | 4 coaches with teams |
| **Unique Feature** | Judge mentorship | Golden Buzzer | Blind auditions & chair turns |
| **Team Formation** | By category | No teams | Coach teams |
| **Voting Window** | Weekly live shows | Live shows only | Battle rounds + Live shows |
| **Winner Prize** | Recording contract | Cash prize + show contract | Recording contract + mentorship |

### Common Features

All shows share these core features:
- Auditions → Eliminations → Live Shows → Finale
- Public voting during live shows
- Judge/coach involvement
- Contestant profiles and journey tracking
- Behind-the-scenes content
- Ticket sales for live shows

---

## Unified Architecture

### Component Structure

```javascript
const talentShowArchitecture = {
  "core_modules": {
    "user_management": "Shared across all shows",
    "payment_processing": "Shared across all shows",
    "voting_engine": "Configurable per show",
    "streaming_service": "Shared infrastructure",
    "analytics": "Shared with show-specific metrics"
  },
  "show_specific_modules": {
    "show_configuration": "Per-show settings",
    "voting_rules": "Custom voting logic",
    "ui_themes": "Branded interfaces",
    "content_types": "Show-specific content"
  }
}
```

### Inheritance Model

```javascript
// Base Talent Show Class
class TalentShow {
  constructor(config) {
    this.showType = config.showType;
    this.seasons = [];
    this.votingEnabled = true;
    this.judges = [];
  }
  
  // Common methods
  createSeason() {}
  manageContestants() {}
  handleVoting() {}
  streamContent() {}
}

// XFactor Implementation
class XFactor extends TalentShow {
  constructor() {
    super({ showType: 'XFACTOR' });
    this.categories = ['boys', 'girls', 'over_25', 'groups'];
    this.judgeMentorship = true;
  }
  
  assignJudgeCategories() {}
  handleSixChairChallenge() {}
}

// Cambodia's Got Talent Implementation
class CambodiaGotTalent extends TalentShow {
  constructor() {
    super({ showType: 'CGT' });
    this.talentTypes = ['singing', 'dancing', 'magic', 'comedy', 'other'];
    this.goldenBuzzer = true;
  }
  
  handleGoldenBuzzer() {}
  categorizeActs() {}
}

// The Voice Implementation
class TheVoiceCambodia extends TalentShow {
  constructor() {
    super({ showType: 'VOICE' });
    this.blindAuditions = true;
    this.coachTeams = true;
    this.battles = true;
  }
  
  handleChairTurn() {}
  manageBattleRounds() {}
  handleSteals() {}
}
```

---

## Common Components

### 1. Voting System (Configurable)

```javascript
const votingSystemConfig = {
  "base_features": {
    "real_time_voting": true,
    "multiple_voting_methods": ["app", "sms", "web"],
    "fraud_prevention": true,
    "geographic_restrictions": true
  },
  "show_configurations": {
    "XFACTOR": {
      "voting_window": "30 minutes after performances",
      "free_votes": 5,
      "vote_packages": true,
      "categories": true
    },
    "CGT": {
      "voting_window": "During and after show",
      "free_votes": 10,
      "golden_buzzer_immunity": true,
      "act_types": true
    },
    "VOICE": {
      "voting_phases": ["battles", "knockouts", "live_shows"],
      "team_based_voting": true,
      "coach_save": true,
      "instant_save": true
    }
  }
}
```

### 2. Contestant Management

```javascript
const contestantSchema = {
  "common_fields": {
    "id": "UUID",
    "name": "String",
    "age": "Integer",
    "hometown": "String",
    "bio": "Text",
    "profile_image": "URL",
    "audition_video": "URL",
    "status": "active|eliminated",
    "social_media": "Object"
  },
  "show_specific_fields": {
    "XFACTOR": {
      "category": "boys|girls|over_25|groups",
      "mentor_judge": "Judge ID"
    },
    "CGT": {
      "talent_type": "Enum",
      "act_description": "Text",
      "props_required": "Array",
      "golden_buzzer_used": "Boolean"
    },
    "VOICE": {
      "coach": "Coach ID",
      "chair_turned_by": "Array<Coach ID>",
      "battle_opponent": "Contestant ID",
      "stolen_by": "Coach ID"
    }
  }
}
```

### 3. Show Management Interface

```javascript
const showManagementAPI = {
  // Common endpoints for all shows
  "GET /api/v1/shows/:showType/current": "Get current season",
  "GET /api/v1/shows/:showType/contestants": "List contestants",
  "GET /api/v1/shows/:showType/episodes": "List episodes",
  "POST /api/v1/shows/:showType/vote": "Submit vote",
  
  // Dynamic configuration based on show type
  "GET /api/v1/shows/:showType/config": {
    "response": {
      "show_type": "XFACTOR|CGT|VOICE",
      "voting_rules": "Object",
      "ui_theme": "Object",
      "features_enabled": "Array"
    }
  }
}
```

---

## Show-Specific Configurations

### XFactor Configuration

```javascript
const xfactorConfig = {
  "show_format": {
    "stages": [
      "auditions",
      "bootcamp",
      "six_chair_challenge",
      "judges_houses",
      "live_shows",
      "finale"
    ],
    "unique_features": {
      "category_system": true,
      "judge_mentorship": true,
      "six_chair_challenge": true
    }
  },
  "voting_config": {
    "live_show_voting": true,
    "vote_window_minutes": 30,
    "multiple_voting": true
  }
}
```

### Cambodia's Got Talent Configuration

```javascript
const cgtConfig = {
  "show_format": {
    "stages": [
      "open_auditions",
      "judge_auditions",
      "semi_finals",
      "finals"
    ],
    "unique_features": {
      "golden_buzzer": true,
      "variety_acts": true,
      "all_ages": true
    }
  },
  "act_categories": [
    "singing",
    "dancing",
    "magic_illusion",
    "comedy",
    "acrobatics",
    "instruments",
    "unique_acts"
  ],
  "voting_config": {
    "semi_final_voting": true,
    "final_voting": true,
    "vote_window_minutes": 60
  }
}
```

### The Voice Configuration

```javascript
const voiceConfig = {
  "show_format": {
    "stages": [
      "blind_auditions",
      "battle_rounds",
      "knockouts",
      "live_playoffs",
      "live_shows",
      "finale"
    ],
    "unique_features": {
      "blind_auditions": true,
      "chair_turns": true,
      "coach_teams": true,
      "steals": true,
      "coach_save": true
    }
  },
  "team_config": {
    "coaches": 4,
    "team_size": 12,
    "battle_pairs": true,
    "steal_limit": 2
  },
  "voting_config": {
    "public_voting_start": "live_playoffs",
    "instant_save": true,
    "coach_points": true
  }
}
```

---

## Database Design

### Unified Schema with Show-Specific Extensions

```sql
-- Base tables for all shows
CREATE TABLE shows (
    id UUID PRIMARY KEY,
    show_type VARCHAR(20) NOT NULL, -- XFACTOR, CGT, VOICE
    name VARCHAR(100) NOT NULL,
    logo_url VARCHAR(500),
    theme_config JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE seasons (
    id UUID PRIMARY KEY,
    show_id UUID REFERENCES shows(id),
    season_number INTEGER NOT NULL,
    start_date DATE,
    end_date DATE,
    status VARCHAR(20),
    config JSONB, -- Show-specific configuration
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE contestants (
    id UUID PRIMARY KEY,
    season_id UUID REFERENCES seasons(id),
    name VARCHAR(100) NOT NULL,
    age INTEGER,
    hometown VARCHAR(100),
    bio TEXT,
    profile_image VARCHAR(500),
    status VARCHAR(20),
    metadata JSONB, -- Show-specific data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- XFactor specific
CREATE TABLE xfactor_categories (
    contestant_id UUID REFERENCES contestants(id),
    category VARCHAR(20) NOT NULL,
    mentor_judge_id UUID,
    six_chair_position INTEGER
);

-- CGT specific
CREATE TABLE cgt_acts (
    contestant_id UUID REFERENCES contestants(id),
    act_type VARCHAR(50) NOT NULL,
    act_description TEXT,
    props_required TEXT[],
    golden_buzzer_by VARCHAR(100),
    golden_buzzer_date TIMESTAMP
);

-- Voice specific
CREATE TABLE voice_teams (
    contestant_id UUID REFERENCES contestants(id),
    coach_id UUID NOT NULL,
    chair_turned_by UUID[],
    battle_round_opponent UUID,
    stolen_from_coach UUID,
    steal_round VARCHAR(20)
);

-- Shared voting table with show-specific rules
CREATE TABLE votes (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    contestant_id UUID NOT NULL,
    season_id UUID NOT NULL,
    show_type VARCHAR(20) NOT NULL,
    vote_type VARCHAR(20),
    vote_count INTEGER DEFAULT 1,
    metadata JSONB, -- Show-specific voting data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## API Structure

### Base API Pattern

```javascript
// Base route: /api/v1/shows/:showType

// Common endpoints
router.get('/:showType/seasons/current', getCurrentSeason);
router.get('/:showType/contestants', getContestants);
router.get('/:showType/episodes', getEpisodes);
router.post('/:showType/vote', submitVote);

// Show-specific endpoints loaded dynamically
const showRoutes = {
  'xfactor': require('./routes/xfactor'),
  'cgt': require('./routes/cgt'),
  'voice': require('./routes/voice')
};

// XFactor specific routes
router.get('/xfactor/categories', getCategories);
router.get('/xfactor/six-chair-status', getSixChairStatus);

// CGT specific routes
router.get('/cgt/acts/types', getActTypes);
router.post('/cgt/golden-buzzer', useGoldenBuzzer);

// Voice specific routes
router.post('/voice/chair-turn', recordChairTurn);
router.get('/voice/teams', getTeams);
router.post('/voice/steal', executeSteal);
```

### Dynamic Show Loading

```javascript
class ShowManager {
  constructor() {
    this.shows = new Map();
  }
  
  async loadShow(showType) {
    if (this.shows.has(showType)) {
      return this.shows.get(showType);
    }
    
    const config = await this.getShowConfig(showType);
    let show;
    
    switch (showType) {
      case 'XFACTOR':
        show = new XFactor(config);
        break;
      case 'CGT':
        show = new CambodiaGotTalent(config);
        break;
      case 'VOICE':
        show = new TheVoiceCambodia(config);
        break;
      default:
        throw new Error('Unknown show type');
    }
    
    this.shows.set(showType, show);
    return show;
  }
  
  async getShowConfig(showType) {
    return await db.query(
      'SELECT * FROM shows WHERE show_type = $1',
      [showType]
    );
  }
}
```

---

## UI/UX Patterns

### Dynamic Theming System

```javascript
const showThemes = {
  "xfactor": {
    "primary_color": "#FF0000",
    "secondary_color": "#000000",
    "logo": "xfactor-logo.png",
    "fonts": {
      "heading": "Bold Sans",
      "body": "Regular Sans"
    },
    "ui_elements": {
      "voting_card": "xfactor-vote-card",
      "contestant_profile": "xfactor-profile"
    }
  },
  "cgt": {
    "primary_color": "#FFD700",
    "secondary_color": "#000000",
    "logo": "cgt-logo.png",
    "special_elements": {
      "golden_buzzer": true,
      "act_type_icons": true
    }
  },
  "voice": {
    "primary_color": "#00A8E8",
    "secondary_color": "#FFFFFF",
    "logo": "voice-logo.png",
    "special_elements": {
      "chair_animation": true,
      "team_colors": ["#FF6B6B", "#4ECDC4", "#45B7D1", "#96E6B3"]
    }
  }
}
```

### Component Library Structure

```
/components
  /common
    - VotingCard.js
    - ContestantProfile.js
    - LiveIndicator.js
    - CountdownTimer.js
  /shows
    /xfactor
      - CategorySelector.js
      - SixChairTracker.js
      - JudgeMentor.js
    /cgt
      - ActTypeFilter.js
      - GoldenBuzzerAnimation.js
      - TalentShowcase.js
    /voice
      - ChairTurnAnimation.js
      - TeamDisplay.js
      - BattleBracket.js
```

---

## Business Logic

### Revenue Model Adaptation

```javascript
const revenueModels = {
  "common_streams": {
    "ticket_sales": "All shows",
    "voting_packages": "All shows",
    "premium_content": "All shows",
    "merchandise": "All shows"
  },
  "show_specific": {
    "xfactor": {
      "category_voting": "Vote for entire category",
      "mentor_content": "Exclusive judge content"
    },
    "cgt": {
      "act_sponsorship": "Sponsor specific acts",
      "golden_buzzer_prediction": "Betting on GB usage"
    },
    "voice": {
      "team_packages": "Support entire team",
      "coach_cam": "Exclusive coach reactions"
    }
  }
}
```

### Analytics Framework

```javascript
const analyticsConfig = {
  "common_metrics": [
    "total_votes",
    "unique_voters", 
    "revenue_per_episode",
    "viewer_retention",
    "social_engagement"
  ],
  "show_specific_metrics": {
    "xfactor": [
      "votes_by_category",
      "judge_influence_score",
      "category_popularity"
    ],
    "cgt": [
      "votes_by_act_type",
      "golden_buzzer_impact",
      "act_diversity_index"
    ],
    "voice": [
      "chair_turn_rate",
      "team_loyalty_score",
      "battle_engagement",
      "steal_impact"
    ]
  }
}
```

---

## Implementation Strategy

### Phase 1: Core Framework (Month 1-2)
- Build base TalentShow class and common components
- Implement shared services (voting, streaming, user management)
- Create dynamic show loading system

### Phase 2: XFactor Implementation (Month 3-4)
- Already partially complete
- Adapt existing code to new framework
- Test framework with real show

### Phase 3: CGT Implementation (Month 5-6)
- Implement golden buzzer features
- Add act categorization
- Support variety performance types

### Phase 4: Voice Implementation (Month 7-8)
- Build blind audition mechanics
- Implement team management
- Add battle round features

### Benefits of This Approach

1. **Code Reusability**: 70% shared code across shows
2. **Faster Development**: New shows in 2 months vs 6 months
3. **Consistent UX**: Users familiar with one show can easily use others
4. **Centralized Maintenance**: Fix once, deploy everywhere
5. **Scalability**: Easy to add new shows (The Mask Singer, etc.)
6. **Cost Efficiency**: Reduced development and maintenance costs

---

## Future Expansion

This framework can easily accommodate:
- **The Mask Singer**: Add costume/identity features
- **Cambodia Idol**: Simpler version of XFactor
- **Dance Competition Shows**: Leverage CGT's variety act support
- **Comedy Shows**: Use CGT's framework with comedy focus
- **Kids Versions**: Add age-specific features and parental controls

The modular architecture ensures that adding new show types requires minimal core changes while allowing full customization of show-specific features.