# Cambodia's Got Talent - Implementation Guide

## Overview

Cambodia's Got Talent (CGT) showcases diverse talents including singing, dancing, magic, comedy, and unique acts. This guide covers the specific implementation details for integrating CGT into the Hang Meas Mobile App.

> **Framework Reference**: This implementation extends the base framework defined in [TALENT_SHOWS_FRAMEWORK.md](../../TALENT_SHOWS_FRAMEWORK.md)

---

## Show Format Specifics

### What Makes CGT Different

Unlike singing-only competitions, CGT features:
- **Variety Acts**: From magic to comedy to danger acts
- **All Ages**: 5-year-old dancers to 80-year-old singers
- **Golden Buzzer**: Sends acts straight to semi-finals
- **No Categories**: All acts compete together
- **Visual Performances**: Many acts require full stage visibility

### Show Structure

```
Auditions → Judge Cuts → Semi-Finals → Grand Final
   ↓             ↓            ↓            ↓
Open Call    Top 100      Top 20       Winner
```

---

## Key Features

### 1. Golden Buzzer System

```javascript
const goldenBuzzerFeature = {
  "rules": {
    "uses_per_judge": 1,
    "uses_per_season": 4, // 4 judges
    "instant_advancement": "semi_finals",
    "immunity": "Cannot be eliminated until semi-finals"
  },
  "ui_experience": {
    "button_animation": "Confetti explosion",
    "sound_effect": "Dramatic buzzer sound",
    "screen_takeover": "Full screen golden effect",
    "social_sharing": "Auto-generate shareable moment"
  },
  "tracking": {
    "timestamp": "Exact moment of press",
    "judge": "Which judge used it",
    "audience_reaction": "Measure excitement spike",
    "viral_potential": "Social media tracking"
  }
}
```

### 2. Act Categorization

```javascript
const actCategories = {
  "singing": {
    "icon": "🎤",
    "subcategories": ["solo", "group", "choir", "opera", "traditional"]
  },
  "dancing": {
    "icon": "💃",
    "subcategories": ["contemporary", "traditional", "street", "group", "solo"]
  },
  "magic_illusion": {
    "icon": "🎩",
    "subcategories": ["close_up", "stage", "mentalism", "escape"]
  },
  "comedy": {
    "icon": "😂",
    "subcategories": ["stand_up", "physical", "musical", "sketch"]
  },
  "acrobatics": {
    "icon": "🤸",
    "subcategories": ["aerial", "ground", "group", "extreme"]
  },
  "music_instrumental": {
    "icon": "🎸",
    "subcategories": ["traditional", "modern", "fusion", "orchestra"]
  },
  "unique_acts": {
    "icon": "🌟",
    "subcategories": ["danger", "animal", "technology", "other"]
  }
}
```

### 3. Visual Performance Requirements

Since many acts are visual (magic, dance, acrobatics), the app needs:

```javascript
const visualPerformanceFeatures = {
  "camera_angles": {
    "wide_shot": "Full stage view for dance groups",
    "close_up": "Magic trick details",
    "aerial_view": "Acrobatics formations",
    "audience_reaction": "Shock and awe moments"
  },
  "special_requirements": {
    "aspect_ratio": "16:9 mandatory",
    "minimum_quality": "720p for trick visibility",
    "frame_rate": "60fps for fast movements",
    "multi_camera": "Essential for magic acts"
  },
  "replay_features": {
    "slow_motion": "Acrobatics and magic reveals",
    "instant_replay": "Golden buzzer moments",
    "highlight_clips": "Auto-generated best moments"
  }
}
```

---

## Mobile App Features

### 1. Act Discovery & Filtering

```javascript
// UI Component: Act Filter
const ActFilterComponent = () => {
  return (
    <FilterSection>
      <CategoryFilter>
        {Object.entries(actCategories).map(([key, category]) => (
          <FilterChip
            key={key}
            icon={category.icon}
            label={key.replace('_', ' ')}
            onSelect={() => filterByCategory(key)}
          />
        ))}
      </CategoryFilter>
      
      <AgeRangeFilter>
        <Slider min={5} max={80} label="Performer Age" />
      </AgeRangeFilter>
      
      <SpecialFilters>
        <Chip label="Golden Buzzer Acts" special={true} />
        <Chip label="Group Acts" />
        <Chip label="Solo Acts" />
      </SpecialFilters>
    </FilterSection>
  );
};
```

### 2. Performance Showcase

```javascript
const PerformanceShowcase = ({ act }) => {
  return (
    <ShowcaseCard>
      <VideoPlayer
        src={act.performanceVideo}
        poster={act.thumbnail}
        controls={['play', 'fullscreen', 'quality']}
      />
      
      <ActInfo>
        <ActName>{act.name}</ActName>
        <ActType icon={actCategories[act.type].icon}>
          {act.type}
        </ActType>
        {act.goldenBuzzer && (
          <GoldenBuzzerBadge judge={act.goldenBuzzerJudge} />
        )}
      </ActInfo>
      
      <ActStats>
        <ViewCount>{act.views} views</ViewCount>
        <VoteCount>{act.votes} votes</VoteCount>
        <ShareButton onClick={() => shareAct(act)} />
      </ActStats>
      
      {act.type === 'magic_illusion' && (
        <SpoilerWarning>
          This video contains magic reveals
        </SpoilerWarning>
      )}
    </ShowcaseCard>
  );
};
```

### 3. Voting Interface

```javascript
const CGTVotingInterface = () => {
  const [selectedActs, setSelectedActs] = useState([]);
  const maxVotes = 10; // Different from XFactor's 5
  
  return (
    <VotingContainer>
      <VotingHeader>
        <TimeRemaining />
        <VotesRemaining count={maxVotes - selectedActs.length} />
      </VotingHeader>
      
      <ActGrid>
        {acts.map(act => (
          <ActVoteCard key={act.id}>
            <ActThumbnail src={act.image} />
            <ActCategory icon={actCategories[act.type].icon} />
            <ActName>{act.name}</ActName>
            <VoteButton
              voted={selectedActs.includes(act.id)}
              onClick={() => toggleVote(act.id)}
            />
          </ActVoteCard>
        ))}
      </ActGrid>
      
      <VoteSubmit
        enabled={selectedActs.length > 0}
        onClick={submitVotes}
      />
    </VotingContainer>
  );
};
```

---

## Technical Considerations

### 1. Video Storage & Streaming

Due to the visual nature of acts:

```javascript
const videoRequirements = {
  "storage": {
    "full_performance": "5-10 minutes per act",
    "highlights": "30-60 second clips",
    "golden_buzzer_moments": "Full uncut reaction",
    "audition_packages": "Include judge comments"
  },
  "streaming_quality": {
    "minimum": "480p for mobile data saving",
    "standard": "720p for clear trick visibility", 
    "premium": "1080p for large group performances",
    "adaptive": "Auto-switch based on connection"
  },
  "special_features": {
    "360_video": "For immersive acts (future)",
    "multi_angle": "Essential for magic acts",
    "slow_motion": "Built-in for replays"
  }
}
```

### 2. Database Schema Extensions

```sql
-- CGT specific tables extending base framework
CREATE TABLE cgt_acts (
    id UUID PRIMARY KEY,
    contestant_id UUID REFERENCES contestants(id),
    act_type VARCHAR(50) NOT NULL,
    act_subtype VARCHAR(50),
    group_size INTEGER DEFAULT 1,
    age_range VARCHAR(20), -- 'kids', 'teens', 'adults', 'seniors', 'mixed'
    props_required TEXT[],
    stage_requirements TEXT,
    safety_requirements TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cgt_golden_buzzers (
    id UUID PRIMARY KEY,
    act_id UUID REFERENCES cgt_acts(id),
    judge_id UUID NOT NULL,
    episode_id UUID NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    video_clip_url VARCHAR(500),
    viral_view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cgt_performance_metrics (
    id UUID PRIMARY KEY,
    act_id UUID REFERENCES cgt_acts(id),
    performance_date TIMESTAMP NOT NULL,
    audience_reaction_score DECIMAL(3,2), -- 0-10 scale
    judge_scores JSONB, -- Individual judge scores
    technical_difficulty DECIMAL(3,2),
    entertainment_value DECIMAL(3,2),
    uniqueness_score DECIMAL(3,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3. API Endpoints

```javascript
// CGT-specific endpoints
const cgtRoutes = {
  // Act browsing
  "GET /api/v1/shows/cgt/acts": "List all acts with filters",
  "GET /api/v1/shows/cgt/acts/:category": "Get acts by category",
  "GET /api/v1/shows/cgt/acts/:id": "Get specific act details",
  
  // Golden Buzzer
  "GET /api/v1/shows/cgt/golden-buzzers": "All golden buzzer moments",
  "POST /api/v1/shows/cgt/golden-buzzer/predict": "User predictions",
  
  // Voting
  "POST /api/v1/shows/cgt/vote": "Submit votes (up to 10)",
  "GET /api/v1/shows/cgt/results": "Get current results",
  
  // Special features
  "GET /api/v1/shows/cgt/acts/:id/similar": "Find similar acts",
  "GET /api/v1/shows/cgt/trending": "Trending performances"
}
```

---

## Monetization Opportunities

### CGT-Specific Revenue Streams

```javascript
const cgtMonetization = {
  "golden_buzzer_packages": {
    "prediction_contest": {
      "price": "$1.99",
      "description": "Predict which act gets golden buzzer",
      "prize": "Meet the golden buzzer act"
    },
    "instant_replay": {
      "price": "$0.99", 
      "description": "Rewatch golden buzzer moment unlimited"
    }
  },
  "act_sponsorship": {
    "sponsor_an_act": {
      "price": "$99-999",
      "benefits": [
        "Logo display during performance",
        "Mentioned by hosts",
        "Meet and greet with act"
      ]
    }
  },
  "premium_content": {
    "backstage_pass": {
      "price": "$4.99/month",
      "includes": [
        "All rehearsal footage",
        "Judge discussions",
        "Failed tricks and bloopers"
      ]
    }
  },
  "virtual_tickets": {
    "live_stream_access": "$9.99 per show",
    "season_pass": "$49.99",
    "vip_experience": "$99.99"
  }
}
```

---

## Marketing Features

### Social Media Integration

```javascript
const socialFeatures = {
  "shareable_moments": {
    "golden_buzzer_clips": "Auto-edited with music",
    "wow_moments": "Incredible tricks or fails", 
    "judge_reactions": "Shocked faces compilation",
    "before_after": "Contestant transformation"
  },
  "hashtag_campaigns": {
    "act_specific": "#TeamMagicMan #CGTDanceGroup",
    "weekly_themes": "#CGTGoldenBuzzer #CGTFinale",
    "challenges": "#TryThisTrickAtHome #DanceWithCGT"
  },
  "user_generated": {
    "talent_submissions": "Submit your talent video",
    "reaction_videos": "Fan reaction compilations",
    "prediction_contests": "Guess the winner challenges"
  }
}
```

---

## Analytics & Insights

### CGT-Specific Metrics

```javascript
const cgtAnalytics = {
  "performance_metrics": {
    "act_type_popularity": "Which talents get most votes",
    "age_group_analysis": "Voting patterns by performer age",
    "golden_buzzer_impact": "View/vote spike after GB"
  },
  "engagement_metrics": {
    "replay_rate": "Which acts get rewatched",
    "share_rate": "Viral potential scoring",
    "completion_rate": "Full performance views"
  },
  "business_metrics": {
    "sponsor_roi": "Sponsorship value delivered",
    "premium_conversion": "Free to paid users",
    "merchandise_sales": "Act-specific merchandise"
  }
}
```

---

## Implementation Timeline

### Phase 1: Core Features (Month 1)
- Act categorization system
- Basic voting for variety acts
- Video showcase interface

### Phase 2: Golden Buzzer (Month 2)
- Golden buzzer mechanics
- Special effects and animations
- Prediction contests

### Phase 3: Enhanced Features (Month 3)
- Multi-angle video support
- Act comparison tools
- Social sharing features

### Phase 4: Monetization (Month 4)
- Sponsorship platform
- Premium content gates
- Virtual ticket sales

---

## Success Metrics

- **User Engagement**: Average 3 acts viewed per session
- **Voting Participation**: 40% of viewers vote
- **Golden Buzzer Moments**: 1M+ views within 24 hours
- **Revenue Per User**: $2.50 per active user per season
- **Sponsor Satisfaction**: 80% renewal rate

CGT's variety format offers unique opportunities for engagement and monetization through its diverse acts and visual spectacle, making it a perfect addition to the Hang Meas entertainment ecosystem.