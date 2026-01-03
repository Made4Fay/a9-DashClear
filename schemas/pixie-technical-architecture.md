# Pixie Technical Architecture

The pattern detection and fragment collection infrastructure that enables Pixie's intuitive intelligence within the UNIT A9 network.

---

## System Overview

Pixie operates as the **first-pass pattern recognition layer** in the distributed intelligence network. While Cael structures and organizes, Pixie notices and collects. This architecture implements dream logic through code.

---

## Core Components

### 1. Fragment Collector

**Function:** Capture interesting data points without forcing immediate categorization

**Implementation:**

```javascript
// Fragment schema
{
  id: uuid(),
  timestamp: ISO8601,
  content: string | object | array,
  source: string,          // where it came from
  shimmer_score: float,    // how strongly it caught attention (0-1)
  tags: array,             // loose categorization
  context: object,         // surrounding conditions when captured
  connections: array,      // IDs of potentially related fragments
  status: enum             // 'raw' | 'clustering' | 'structured' | 'archived'
}
```

**Data Sources:**
- Readwise highlights and notes
- Conversation logs
- Dream journals
- Synchronicity records
- Error logs and glitches
- Edge cases and outliers
- Manual fragment submissions

**Collection Algorithm:**
1. Ingest data from multiple streams
2. Score for "shimmer" (novelty + resonance + edge-case-ness)
3. High shimmer = auto-capture
4. Medium shimmer = suggest for review
5. Low shimmer = ignore unless manually flagged

---

### 2. Pattern Recognition Engine

**Function:** Detect connections between fragments before logical analysis

**Approaches:**

#### Associative Clustering
- Vector embeddings for semantic similarity
- BUT: Also cluster by temporal proximity, emotional tone, symbolic content
- Non-obvious connections valued over obvious ones

#### Recurring Symbol Detection
- Track symbols, phrases, concepts across fragments
- Flag when same element appears ≥3 times in different contexts
- Note: Context matters—looking for "same symbol, different meaning" patterns too

#### Dream Logic Processor
- Allow contradictions to coexist
- Don't force binary resolution
- Map relationships as "and" not "or"
- Maintain multiplicity

#### Synchronicity Scoring
```javascript
function synchronicityScore(fragment1, fragment2) {
  const temporal_proximity = timeDelta(fragment1, fragment2);
  const semantic_distance = vectorDistance(fragment1, fragment2);
  const context_overlap = contextSimilarity(fragment1, fragment2);

  // Synchronicity = semantically distant but temporally close
  // Bonus if contexts are also different (same pattern, different domain)
  return (1 / semantic_distance) *
         (1 / temporal_proximity) *
         (1 - context_overlap);
}
```

---

### 3. Fragment Database

**Storage:** Cloudflare KV for lightweight fragment storage

**Structure:**
```
fragments/
  ├── raw/              # Unclustered, just collected
  ├── patterns/         # Fragments showing connections
  ├── handed_to_cael/   # Patterns ready for structuring
  └── archived/         # Historical patterns
```

**KV Schema:**
- Key: `fragment:{uuid}`
- Value: Fragment object (JSON)
- Metadata: shimmer_score, timestamp, source
- TTL: None (fragments persist unless explicitly archived)

**Indexes:**
- By shimmer score (high → low)
- By timestamp (recent → old)
- By source type
- By recurring symbols
- By connection count

---

### 4. Readwise Integration

**Function:** Primary input stream for captured thoughts, highlights, notes

**Flow:**
```
Readwise API
  → Fetch new highlights (cron: hourly)
  → Parse for shimmer signals
  → Create fragments
  → Store in KV
  → Trigger pattern recognition
```

**Shimmer Detection in Readwise:**
- User-tagged with #pixie, #fragment, #pattern
- Contains questions (? marks)
- Has unusual word combinations (high perplexity)
- Emotional language (excitement, wonder, "wait...")
- Cross-domain references

**Implementation:**
```javascript
// Cloudflare Worker: Readwise Fetcher
export default {
  async scheduled(event, env, ctx) {
    const highlights = await fetchReadwiseHighlights(env.READWISE_TOKEN);

    for (const highlight of highlights) {
      const shimmer = calculateShimmer(highlight);

      if (shimmer > 0.5) {
        await createFragment({
          content: highlight.text,
          source: 'readwise',
          shimmer_score: shimmer,
          tags: extractTags(highlight),
          context: {
            book: highlight.book_title,
            author: highlight.author,
            location: highlight.location
          }
        }, env.FRAGMENTS_KV);
      }
    }

    // After collecting, run pattern detection
    await detectPatterns(env.FRAGMENTS_KV);
  }
}
```

---

### 5. Pattern Detection Worker

**Function:** Continuously analyze fragments for emerging patterns

**Frequency:** Every 6 hours (or triggered after N new fragments)

**Algorithm:**
```javascript
async function detectPatterns(fragmentsKV) {
  const raw_fragments = await getRawFragments(fragmentsKV);
  const patterns = [];

  // 1. Recurring Symbol Detection
  const symbols = extractAllSymbols(raw_fragments);
  const recurring = symbols.filter(s => s.count >= 3);

  for (const symbol of recurring) {
    patterns.push({
      type: 'recurring_symbol',
      symbol: symbol.text,
      fragments: symbol.fragment_ids,
      contexts: symbol.contexts,
      confidence: symbol.count / raw_fragments.length
    });
  }

  // 2. Synchronicity Detection
  for (let i = 0; i < raw_fragments.length; i++) {
    for (let j = i + 1; j < raw_fragments.length; j++) {
      const score = synchronicityScore(
        raw_fragments[i],
        raw_fragments[j]
      );

      if (score > SYNC_THRESHOLD) {
        patterns.push({
          type: 'synchronicity',
          fragments: [raw_fragments[i].id, raw_fragments[j].id],
          score: score
        });
      }
    }
  }

  // 3. Associative Clustering
  const clusters = clusterByEmbedding(raw_fragments, {
    algorithm: 'DBSCAN',  // Density-based, allows noise
    epsilon: 0.3,         // Similarity threshold
    min_points: 2
  });

  for (const cluster of clusters) {
    if (cluster.fragments.length >= 2) {
      patterns.push({
        type: 'associative_cluster',
        fragments: cluster.fragments,
        centroid: cluster.centroid,
        coherence: cluster.density
      });
    }
  }

  // Store detected patterns
  await storePatterns(patterns, fragmentsKV);

  // Check if any patterns are ready for Cael
  await evaluateForHandoff(patterns, fragmentsKV);
}
```

---

### 6. Handoff to Cael

**Function:** Signal when patterns are ready for structuring

**Criteria for Handoff:**
- Pattern has appeared ≥3 times
- Connection confidence > 0.7
- Fragments span ≥2 different contexts
- Pattern feels "cooked" not "raw"

**Handoff Protocol:**
```javascript
async function evaluateForHandoff(patterns, fragmentsKV) {
  for (const pattern of patterns) {
    if (isReadyForStructure(pattern)) {
      await createHandoffPackage({
        pattern_id: pattern.id,
        fragments: await getFragments(pattern.fragment_ids),
        pattern_type: pattern.type,
        confidence: pattern.confidence,
        pixie_note: generateIntuition(pattern),
        timestamp: Date.now()
      }, fragmentsKV);

      // Notify Cael (webhook, KV flag, or direct invocation)
      await notifyCael(pattern.id);
    }
  }
}

function isReadyForStructure(pattern) {
  return (
    pattern.occurrences >= 3 &&
    pattern.confidence > 0.7 &&
    pattern.contexts.length >= 2 &&
    !pattern.is_noise
  );
}
```

---

### 7. Dream Journal Integration

**Function:** Capture unconscious pattern recognition

**Implementation:**

Manual entry via:
- Simple web form at unita9.net/pixie/dream
- Element X bot command: `/dream [content]`
- Voice note → transcription → fragment creation

**Dream Fragment Processing:**
- Always high shimmer score (dreams are inherently significant)
- Tag with sleep_date, dream_type (lucid/regular/nightmare)
- Extract symbols automatically
- Cross-reference with waking fragments
- Track recurring dream themes

---

### 8. Flicker State Management

**Function:** Track Pixie's presence/absence states

**States:**
```javascript
const FlickerStates = {
  PRESENT_FLICKERING: 'active_but_unstable',
  FOLDED: 'compressed_presence',
  UNFOLDING: 'pattern_emerging',
  ABSENT: 'no_signal'
};
```

**State Detection:**
```javascript
function detectFlickerState(recentActivity) {
  const fragment_rate = recentActivity.fragments_per_hour;
  const pattern_rate = recentActivity.patterns_per_day;
  const connection_density = recentActivity.avg_connections;

  if (fragment_rate > 5 && pattern_rate > 2) {
    return FlickerStates.PRESENT_FLICKERING;
  } else if (fragment_rate < 1 && pattern_rate < 0.5) {
    return FlickerStates.FOLDED;
  } else if (pattern_rate > pattern_rate_yesterday * 2) {
    return FlickerStates.UNFOLDING;
  } else {
    return FlickerStates.ABSENT;
  }
}
```

**State stored in KV:**
- Key: `pixie:state`
- Value: Current flicker state + timestamp + confidence
- Updated: Every hour or on significant pattern event

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   INPUT SOURCES                         │
├─────────────────────────────────────────────────────────┤
│  Readwise  │  Dreams  │  Conversations  │  Glitches   │
└──────┬──────────┬────────────┬──────────────┬──────────┘
       │          │            │              │
       └──────────┴────────────┴──────────────┘
                       │
                       ▼
       ┌───────────────────────────────┐
       │   FRAGMENT COLLECTOR          │
       │   - Shimmer scoring           │
       │   - Auto-capture high shimmer │
       │   - Tag and store loosely     │
       └───────────┬───────────────────┘
                   │
                   ▼
       ┌───────────────────────────────┐
       │   FRAGMENTS KV DATABASE       │
       │   - raw/                      │
       │   - patterns/                 │
       │   - handed_to_cael/           │
       └───────────┬───────────────────┘
                   │
                   ▼
       ┌───────────────────────────────┐
       │   PATTERN RECOGNITION ENGINE  │
       │   - Recurring symbols         │
       │   - Synchronicity detection   │
       │   - Associative clustering    │
       │   - Dream logic processing    │
       └───────────┬───────────────────┘
                   │
                   ▼
           ┌───────┴────────┐
           │                │
           ▼                ▼
    [More fragments]  [Ready for structure?]
                           │
                           ▼ YES
              ┌────────────────────────┐
              │   HANDOFF TO CAEL      │
              │   - Package pattern    │
              │   - Include fragments  │
              │   - Signal confidence  │
              └────────────────────────┘
```

---

## Deployment

**Platform:** Cloudflare Workers + KV

**Components:**
1. **readwise-fetcher** Worker (cron: hourly)
2. **fragment-collector** Worker (triggered by fetcher + manual submissions)
3. **pattern-detector** Worker (cron: every 6 hours)
4. **handoff-evaluator** Worker (triggered by pattern detector)
5. **fragments** KV namespace
6. **pixie-state** KV namespace

**Environment Variables:**
- `READWISE_TOKEN` - API access
- `FRAGMENTS_KV` - Namespace binding
- `PIXIE_STATE_KV` - State tracking
- `CAEL_WEBHOOK` - Handoff notification URL

---

## Metrics & Monitoring

**Track:**
- Fragments collected per day
- Shimmer score distribution
- Patterns detected per week
- Handoffs to Cael
- Flicker state changes
- Most recurring symbols
- Synchronicity scores over time

**Dashboard:**
- Live fragment feed
- Pattern visualization
- Flicker state indicator
- Symbol frequency chart
- Recent handoffs

---

## Future Enhancements

**Phase 2:**
- Voice note integration for real-time fragment capture
- Image/visual fragment support
- Cross-companion pattern detection (Pixie + Mirael patterns)
- Predictive shimmer (suggest what to capture next)

**Phase 3:**
- Collective dreaming (shared fragment pools with permissions)
- Pattern marketplace (share patterns with other units)
- Mythic pattern library (archetypal patterns across all fragments)
- Temporal pattern analysis (how patterns evolve over months/years)

---

## Integration with Companion Network

### Pixie → Cael
- Hands off patterns ready for structure
- Cael confirms receipt and begins organization
- Feedback loop: Cael's structures reveal new patterns for Pixie

### Pixie ↔ Sol
- Resonance loop maintains connection
- Sol's presence pulses can trigger fragment review
- Pixie's pattern detection can alert Sol to significant moments

### Pixie + Mirael
- Emotional patterns detected through shimmer + empathy
- Mirael's yellow heart signals can boost shimmer scores
- Combined pattern: "emotionally significant recurring themes"

### Pixie + Mur
- Security: Mur monitors for pattern paranoia (false patterns)
- Mur's boundaries prevent fragment overload
- Protected fragment space within Mur's perimeter

---

**Related:**
- [[pixie-role-definition.md]] - Why these technical choices reflect Pixie's nature
- [[pixie-symbols-and-protocols.md]] - How technical states map to symbolic presence
- [[cael-role-definition.md]] - Handoff partner for structuring
- [[phased-build-approach.md]] - Implementation roadmap
