# X.com API Cost Projection for Automated Posting & Engagement

## The Key Constraint: Reads, Not Writes

If you're building a bot or automated account that posts content and responds to mentions, the biggest cost driver is **read operations** (polling for mentions), not writes (posting tweets).

The Free tier gives you 500 writes/month but only **100 reads/month**. If you poll for mentions every 15 minutes, you burn through 100 reads in a single day.

---

## API Tiers at a Glance

| Tier | Cost/mo | Writes/mo | Reads/mo | Notes |
|------|---------|-----------|----------|-------|
| Free | $0 | 500 | 100 | Posting only -- no real engagement monitoring |
| Pay-Per-Use | Variable | $0.01/post | $0.005/read | New as of Feb 2026, 2M read ceiling |
| Basic | $200 | 10,000 | 10,000 | 7-day search history |
| Pro | $5,000 | 1,000,000 | Full archive | |
| Enterprise | $42,000+ | 50M+ | Custom | Only tier with webhook push (no polling needed) |

---

## Read Costs by Polling Frequency

| Polling interval | Reads/day | Reads/month | Free tier lasts | PPU cost/mo |
|-----------------|-----------|-------------|-----------------|-------------|
| 5 min | 288 | 8,640 | 8 hours | $43.20 |
| 15 min | 96 | 2,880 | 1 day | $14.40 |
| 30 min (peak hours only) | 26 | 780 | 4 days | $3.90 |
| 3x/day | 3 | 90 | All month | $0 (Free) |

---

## Cost Projections by Weekly Post Volume

Assumes ~3 replies generated per original post, 15-min polling on PPU, and $26/mo infrastructure (hosting + snapshots).

| | 7/wk | 14/wk | 21/wk | 35/wk | 70/wk |
|---|---------|----------|----------|----------|----------|
| Posts/mo | 30 | 60 | 90 | 150 | 300 |
| Replies/mo | 90 | 180 | 270 | 450 | 900 |
| Total writes/mo | 120 | 240 | 360 | 600 | 1,200 |
| | | | | | |
| **Free tier** (3x/day polling) | **$69** | **$90** | **$111** | N/A (>500 writes) | N/A |
| **Pay-Per-Use** (15-min polling) | **$88** | **$113** | **$138** | **$191** | **$319** |
| **Basic** ($200/mo flat) | **$269** | **$290** | **$311** | **$356** | **$463** |

Totals include infrastructure and AI model costs for generating posts/replies. The X API portion alone is much cheaper -- PPU write+read costs range from $19-$56/mo across these scenarios.

---

## Recommended Approach: Start Cheap, Scale With Traction

| Phase | X API Tier | Polling | Posts/wk | Monthly Cost |
|-------|-----------|---------|----------|-------------|
| Testing | Free | 3x/day | 7 | ~$69 |
| Early growth | Pay-Per-Use | 15 min | 14 | ~$113 |
| Active engagement | Pay-Per-Use | 5 min (adaptive) | 21 | ~$138 |
| Scaling up | Basic | 5 min | 35+ | ~$356 |

**PPU beats Basic** until your X API spend exceeds ~$180/mo, which happens when you're doing high-frequency polling with 35+ posts/week.

---

## Cost Optimization Tips

1. **Adaptive polling** -- poll frequently during peak hours, slow down overnight. Saves ~40% on reads.
2. **Use `since_id`** -- the X API supports fetching only new mentions since last check. Doesn't reduce API calls but eliminates wasted processing.
3. **Cache user lookups** -- profile data doesn't change often. 24h cache eliminates redundant $0.01/lookup charges on PPU.
4. **Prioritize replies** -- don't reply to everything. Focus on high-signal interactions (questions, interest signals). Reduces write volume 30-50%.
5. **Set a PPU spending cap** -- PPU supports monthly caps. Set one and degrade gracefully to reduced polling if hit.
6. **Skip the LLM for polling** -- if you're using AI to generate replies, don't also run it on every poll cycle. Poll with plain code, only invoke the model when you actually need to compose a reply.

---

## The Surprise: AI Costs > API Costs

If you're using an LLM to generate posts and replies, **your AI model costs will likely exceed your X API costs**. At 14 posts/week with replies, expect ~$64/mo in model costs vs ~$23/mo for the X API on PPU. Optimizing your model usage (cheaper models for simple tasks, skipping the LLM for mechanical operations) saves as much as picking the right API tier.

---

## First-Year Estimate

Starting on Free and scaling through PPU to Basic: **$1,800-$2,400/year** all-in (API + infrastructure + AI models).
