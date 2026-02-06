# OSRS ML Bot Guide

A machine-learning driven bot for Old School RuneScape that uses Q-learning to autonomously train skills, switch tasks intelligently, and operate for extended periods without user intervention.

## Architecture

```
ml_bot.simba          Main entry point + GUI form
    |
    +-- session_manager.simba    Autonomous operation (error recovery, breaks, health)
         |
         +-- task_switching.simba    ML-driven task selection + divergent gameplay
              |
              +-- activities.simba   Skill training implementations (WC, fishing, mining, etc.)
              +-- ml_engine.simba    Q-learning core (state/action/reward, Q-table persistence)
```

## How It Works

### Q-Learning Decision Engine

The bot uses **tabular Q-learning**, a reinforcement learning algorithm, to learn which skills are most productive to train at any given time.

**State Space** (what the bot observes):
- Average combat skill level (discretized into 5 buckets)
- Average skilling level (5 buckets)
- Skill imbalance / standard deviation (5 buckets)
- Session fatigue level (5 buckets)
- Time of day (5 buckets)

This gives 5^5 = 3,125 possible states.

**Action Space** (what the bot can do):
- 10 trainable skills (woodcutting, fishing, mining, cooking, firemaking, thieving, agility, melee, ranged, magic)
- 4 meta-actions (explore, bank management, social/idle, take break)

**Reward Signal** (how the bot learns):
- XP gained per time unit (primary reward)
- Level-up bonus (+50 reward)
- Death penalty (-25)
- Idle/stuck penalty (-5)

The Q-table persists to disk (`configs/mlbot/qtable.json`), so the bot **improves across sessions**. Early sessions explore randomly; later sessions exploit the best-known strategies.

### Task Switching

The Task Switcher manages activity transitions:

1. **Activity Duration**: Each activity runs for 5-45 minutes (Gaussian random), adjusted by fatigue and goal proximity.
2. **Momentum**: Staying on the same task builds momentum, making the bot less likely to switch mid-streak.
3. **Goal Tracking**: The bot sets milestone goals (next 10-level increment per skill) and spends more time on skills close to a goal.
4. **Forced Switches**: After the maximum activity time, the bot always switches.

### Divergent Gameplay

To avoid detection and mimic real players, the bot periodically performs non-productive actions:

- **Explore**: Walk around, rotate camera
- **Check Stats**: Open stats tab, hover over skills
- **Emotes**: Open emote tab
- **Examine Objects**: Right-click random things
- **Read Chat**: Pause and "read" public chat
- **Wander**: Walk through town areas doing nothing

Divergent actions occur ~12% of the time, increasing with fatigue and time since last divergence.

### Autonomous Session Management

The Session Manager keeps the bot running unattended:

- **Error Recovery**: Automatically re-logs, closes stuck interfaces, recovers from disconnects (up to 5 attempts)
- **Health Monitoring**: Tracks deaths/hour, stuck events/hour, login failures. Shuts down safely if thresholds are exceeded.
- **Break Scheduling**: Generates realistic play sessions (45 min to 4 hours) followed by long breaks (15-90 min)
- **Sleep Schedule**: Configurable overnight sleep break (default: 7 hours starting at ~2 AM)
- **State Persistence**: Saves Q-table and session state to disk every 10 minutes and on shutdown
- **Fatigue System**: Tracks virtual fatigue that increases over time and decreases during breaks, affecting activity duration and break frequency

## Quick Start

1. Open `mlbot/ml_bot.simba` in Simba
2. Click Run - the configuration form appears
3. Check the skills you want to train
4. Enable "Divergent Gameplay" for human-like behavior
5. (Optional) Set a max runtime in minutes
6. Click Start

The bot will:
- Log in automatically
- Start training the ML-selected skill
- Switch tasks every 5-45 minutes based on Q-learning decisions
- Take breaks on a realistic schedule
- Recover from errors automatically
- Save its learning progress to disk

## Configuration

### Skills Tab

Toggle which skills the bot can train. Disabled skills are never selected by the ML engine. Enable at least 3-4 skills for meaningful task switching.

### ML Settings Tab

| Setting | Description |
|---------|-------------|
| Divergent Gameplay | Enables human-like non-productive actions |
| Safe Mode | More breaks, slower pace, extra antiban |
| Max Runtime | Total runtime in minutes (0 = unlimited) |

### Antiban Tab

Standard WaspLib antiban configuration. The ML bot adds its own antiban on top:
- Learned break timing (the bot learns when breaks are beneficial)
- Fatigue-aware scheduling
- Divergent gameplay actions

## File Reference

| File | Lines | Purpose |
|------|-------|---------|
| `ml_engine.simba` | ~350 | Q-learning algorithm, Q-table management, reward computation |
| `activities.simba` | ~380 | Skill training methods with level-based progression |
| `task_switching.simba` | ~420 | ML-driven task selection, divergent gameplay, goal tracking |
| `session_manager.simba` | ~400 | Autonomous operation, health monitoring, break scheduling |
| `ml_bot.simba` | ~300 | Main script, GUI form, initialization |

## How the Bot Improves Over Time

### Session 1 (Fresh Q-Table)
- Exploration rate: 40%
- Makes mostly random skill choices
- Learns basic XP rates for each skill at your level

### Sessions 2-5
- Exploration rate: ~25-15%
- Starts favoring skills with higher XP/hour
- Learns which activities are safe (no deaths)
- Begins timing breaks more efficiently

### Sessions 10+
- Exploration rate: ~5% (minimum)
- Reliably selects the best activity for current state
- Has learned time-of-day patterns
- Break timing optimized for your play pattern

### Resetting the ML Model

Delete `configs/mlbot/qtable.json` to start learning from scratch.

## Customization

### Adding a New Skill

1. Add the action to `EMLAction` in `ml_engine.simba`
2. Create a `BuildXxxMethods()` function in `activities.simba`
3. Add it to `BuildAllActivities()`
4. Add a form checkbox in `ml_bot.simba`
5. Handle the action in `SessionManager.RunMainLoop()`

### Adjusting ML Parameters

Edit the constants at the top of `ml_engine.simba`:

```pascal
ML_LEARNING_RATE    = 0.15;    // Higher = learn faster, less stable
ML_DISCOUNT_FACTOR  = 0.90;    // Higher = value future rewards more
ML_EXPLORE_RATE_MAX = 0.40;    // Starting exploration probability
ML_EXPLORE_RATE_MIN = 0.05;    // Minimum exploration floor
ML_EXPLORE_DECAY    = 0.9985;  // Per-decision decay rate
```

### Adjusting Break Schedule

Edit the constants at the top of `session_manager.simba`:

```pascal
SM_PLAY_SESSION_MIN = 45 * ONE_MINUTE;   // Shortest play session
SM_PLAY_SESSION_MAX = 4 * ONE_HOUR;      // Longest play session
SM_LONG_BREAK_MIN   = 15 * ONE_MINUTE;   // Shortest long break
SM_LONG_BREAK_MAX   = 90 * ONE_MINUTE;   // Longest long break
```

## Safety Features

- **Death Limit**: Shuts down after 3 deaths in one hour
- **Stuck Limit**: Shuts down after 5 stuck events in one hour
- **Login Failure Limit**: Shuts down after 5 consecutive login failures
- **Idle Detection**: Attempts recovery if no XP gain for 9+ minutes
- **Hourly Health Reset**: Error counters reset every hour
- **Persistent State**: Q-table and session data survive script restarts

## Troubleshooting

### Bot keeps selecting the same skill
- This means the Q-table has learned that skill is most rewarding. Enable more skills to give it options.
- Delete `qtable.json` to reset learning.

### Bot shuts down too quickly
- Check the health metrics in the log. Common causes: too many deaths (combat too hard), too many stuck events (pathfinding issues).
- Enable Safe Mode for more conservative behavior.

### Bot doesn't seem to learn
- The Q-table needs ~50+ decisions to start showing patterns.
- Check that `qtable.json` is being saved (look for save messages in log).

### Break timing feels unnatural
- Adjust `SM_PLAY_SESSION_MIN/MAX` and `SM_LONG_BREAK_MIN/MAX`.
- The ML engine also learns to request breaks via `ACT_TAKE_BREAK`.
