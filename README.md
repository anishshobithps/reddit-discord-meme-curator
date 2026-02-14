# reddit-discord-meme-curator

🤖 Automated meme curation bot that fetches top posts from Reddit and shares them to Discord with intelligent subreddit rotation and quality scoring.

## Features

- 🎡 **Smart Rotation** - Ensures even distribution across subreddits, no single-sub spam
- 🏆 **Quality Scoring** - Ranks posts by upvotes, engagement, freshness, and ratio
- 🔄 **Duplicate Prevention** - Tracks posted memes and filters crossposts
- 📊 **Usage Analytics** - Monitors 24-hour subreddit distribution
- 🧹 **Auto Cleanup** - Removes old database entries automatically
- ⚡ **Fast & Reliable** - Built with Bun and libSQL for speed
- 🎯 **Customizable** - Weighted subreddits, scoring parameters, rotation strictness

## Prerequisites

- [Bun](https://bun.sh) v1.3.8 or higher
- A [Discord Webhook URL](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)
- A [Turso](https://turso.tech/) database (or compatible libSQL database)

## Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/anishshobithps/reddit-discord-meme-curator.git
   cd reddit-discord-meme-curator
   ```

2. **Install dependencies**
   ```bash
   bun install
   ```

3. **Set up environment variables**

   Create a `.env` file in the root directory:
   ```env
   DB_URL=libsql://your-database-url.turso.io
   DB_TOKEN=your-turso-auth-token
   DISCORD_WEB_URL=https://discord.com/api/webhooks/your/webhook/url
   ```
## Usage

### Run Once
```bash
bun run index.ts
```

## Configuration

Edit the `CONFIG` object in `index.ts`:

```typescript
const CONFIG = {
  SUBREDDITS: ['memes', 'dankmemes', 'wholesomememes', ...],
  FETCH_LIMIT: 50,              // Posts to fetch per subreddit
  MIN_UPVOTES: 100,             // Minimum upvotes required
  MIN_UPVOTE_RATIO: 0.7,        // Minimum ratio (0.0 - 1.0)
  MAX_TITLE_LENGTH: 200,        // Max title length in Discord
  CRON_SCHEDULE: "*/15 * * * *", // Every 15 minute (adjust as needed, we suggest to keep minimum of 15 minutes to avoid spam and not hit API limits)
  ROTATION_ENABLED: true,       // Enable subreddit rotation
  ROTATION_LOOKBACK_POSTS: 3,   // Avoid last N subreddits
  CLEANUP_DAYS: 30,             // Delete DB entries older than N days
};
```

### Subreddit Weights

Adjust quality multipliers for each subreddit:

```typescript
const SUBREDDIT_WEIGHTS = {
  memes: 1.1,           // 10% boost
  dankmemes: 1.2,       // 20% boost
  wholesomememes: 1.0,  // No change
  desimemes: 1.5,       // 50% boost
  indiameme: 1.5,       // 50% boost
  funny: 0.9,           // 10% penalty
  // Add moore to remove
};
```

### Cron Schedule Examples

```typescript
"*/30 * * * *"  // Every 30 minutes
"0 * * * *"     // Every hour
"0 */2 * * *"   // Every 2 hours
"0 9-17 * * *"  // Every hour from 9 AM to 5 PM
"0 12 * * *"    // Once a day at noon
```

[Crontab Guru](https://crontab.guru/) for help with cron expressions.

## How It Works

### 1. Fetch Phase
- Queries top posts from configured subreddits (past 24 hours)
- Fetches up to 50 posts per subreddit concurrently

### 2. Filter Phase
- Removes NSFW content, videos, and non-images
- Checks against posted history (prevents duplicates)
- Filters crossposts if original was already posted
- Validates minimum upvotes and upvote ratio

### 3. Score Phase
Each post is scored based on:
- **Upvotes** (logarithmic scaling, 0-300 pts)
- **Upvote Ratio** (0-50 pts)
- **Engagement** (comment/upvote ratio, 0-30 pts)
- **Freshness** (bonus for <2hr old, penalty for older)
- **Subreddit Weight** (multiplier from config)
- **Rotation Penalty** (recent subreddits get 17-50% reduction)
- **24h Usage Penalty** (10% per post from same sub in 24h)

### 4. Selection Phase
- Top 5 candidates are identified
- Random selection from top 5 for variety
- Rotation penalties ensure diversity

### 5. Post Phase
- Sends embed to Discord webhook
- Stores post ID and subreddit in database
- Logs success/failure

## Database Schema

```sql
-- Posted memes tracker
CREATE TABLE posted_memes (
  id TEXT PRIMARY KEY,           -- Reddit post ID
  subreddit TEXT NOT NULL,       -- Source subreddit
  posted_at INTEGER NOT NULL     -- Timestamp
);

-- Indexes for performance
CREATE INDEX idx_posted_at ON posted_memes(posted_at);
CREATE INDEX idx_subreddit_posted_at ON posted_memes(subreddit, posted_at DESC);
```

## Monitoring & Logs

The bot outputs detailed logs:

```
🚀 Run started at 2024-02-14T12:00:00.000Z
📦 Loaded 1247 posted IDs
📊 Recent subreddits (last 3): memes, funny, dankmemes
📥 Fetched 300 posts from Reddit
✅ Found 156 valid candidates
📂 Candidates from 6 different subreddits

🏆 Top candidates:
  1. [245.3] r/wholesomememes - Wholesome cat meme...
  2. [198.7] r/indiameme - Relatable desi humor...
  ⚖️  Rotation penalty for r/memes: 50%
  📊 Usage penalty for r/memes (2 posts in 24h): 19%

📤 Posting: "Wholesome cat meme"
   From: r/wholesomememes by u/username (12.5K upvotes)
✅ Posted to Discord successfully!
💚 Health: healthy (last post 0.0h ago)
```

### Health Status
- **Healthy**: Database connected, posted within 2 hours
- **Degraded**: Posted within 2-6 hours
- **Unhealthy**: No post in 6+ hours or DB connection failed

## Troubleshooting

### Bot not posting anything
1. Check Discord webhook URL is correct
2. Verify database connection (check `DB_URL` and `DB_TOKEN`)
3. Lower `MIN_UPVOTES` threshold temporarily
4. Check logs for "No valid candidates found"

### Only posting from 1-2 subreddits
1. Increase `ROTATION_LOOKBACK_POSTS` (try 5-6)
2. Check if other subreddits have valid posts that meet criteria
3. Adjust subreddit weights to boost underrepresented ones
4. Lower `MIN_UPVOTES` if subreddits have lower activity

### Posts repeating
1. Verify database is persisting (check Turso dashboard)
2. Ensure `posted_memes` table is being populated
3. Check for duplicate post IDs in database

### Rate limiting errors
1. Increase `REQUEST_TIMEOUT_MS` (default: 10,000)
2. Add delays between requests if needed
3. Reddit generally allows reasonable scraping of public data

### Cron job not running
1. Verify cron syntax with [Crontab Guru](https://crontab.guru/)
2. Check timezone is correct (default: Asia/Kolkata)

## Development

### Project Structure
```
reddit-discord-meme-curator/
├── index.ts              # Main bot code
├── package.json          # Dependencies
├── bun.lockb            # Bun lock file
├── .env                 # Environment variables (create this)
├── .env.example         # Example env file
└── README.md            # This file
```

### Running Tests
```bash
# Dry run without posting
# Comment out sendToDiscord() and run:
bun run index.ts
```

### Adding New Subreddits
1. Add to `CONFIG.SUBREDDITS` array
2. Add weight to `SUBREDDIT_WEIGHTS` object
3. Restart bot

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT License - feel free to use this for your own meme curation needs!

## Acknowledgments

- Reddit API for public post data
- [Turso](https://turso.tech/) for fast edge database
- [Bun](https://bun.sh) for blazing-fast runtime
- The meme communities keeping the internet fun

## Support

For issues or questions:
- Open an issue on GitHub
- Check existing issues for solutions
- Review logs for error messages

---

Made with ☕ and memes
