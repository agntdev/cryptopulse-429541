# CryptoPulse — Bot specification

**Archetype:** finance

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

CryptoPulse is a personal Telegram bot that lets users track cryptocurrency prices with customizable alerts for price thresholds and percent changes. Users can manage watchlists with pre-set or custom tickers, set quiet hours, receive morning summaries, and check prices instantly. The bot owner receives aggregated usage statistics and alert triggers without access to user-specific data.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- cryptocurrency traders and investors
- Telegram users interested in crypto price tracking
- bot owner seeking analytics on user behavior and alert patterns

## Success criteria

- users can add/remove cryptocurrencies to their watchlist
- users receive accurate price alerts based on their set rules
- bot owner receives aggregated metrics without accessing user-specific data
- notifications are delivered according to user preferences including quiet hours and morning summaries

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu with quick actions for adding pre-set cryptocurrencies, configuring settings, and viewing the watchlist
- **/price** (command, actor: user, command: /price) — Check the current price of a specific cryptocurrency or the entire watchlist
- **Add Bitcoin** (button, actor: user, callback: add_ticker:BTC) — Add Bitcoin to the user's watchlist
  - inputs: BTC ticker
  - outputs: updated watchlist entry
- **Add Ethereum** (button, actor: user, callback: add_ticker:ETH) — Add Ethereum to the user's watchlist
  - inputs: ETH ticker
  - outputs: updated watchlist entry
- **Add Toncoin** (button, actor: user, callback: add_ticker:TON) — Add Toncoin to the user's watchlist
  - inputs: TON ticker
  - outputs: updated watchlist entry
- **Add custom ticker** (button, actor: user, callback: add_custom_ticker) — Prompt user to enter a custom ticker symbol to add to their watchlist
  - inputs: custom ticker symbol
  - outputs: updated watchlist entry
- **Configure quiet hours** (button, actor: user, callback: configure_quiet_hours) — Set or modify quiet hours for notification suppression
  - inputs: start time, end time
  - outputs: updated quiet hours settings
- **Set morning summary** (button, actor: user, callback: set_morning_summary) — Set the time for the daily morning price summary
  - inputs: local time
  - outputs: updated morning summary time
- **View watchlist** (button, actor: user, callback: view_watchlist) — Display the user's current watchlist with all set rules
  - inputs: none
  - outputs: watchlist display

## Flows

### onboarding
_Trigger:_ /start

1. Display main menu with quick actions for adding pre-set cryptocurrencies, configuring settings, and viewing the watchlist

_Data touched:_ User profile

### add_ticker
_Trigger:_ add_ticker:*

1. Add the specified ticker to the user's watchlist
2. Display confirmation and options for setting rules (threshold, percent change) or removing the ticker

_Data touched:_ Watchlist entry

### add_custom_ticker
_Trigger:_ add_custom_ticker

1. Prompt user to enter a custom ticker symbol
2. Validate the ticker and add it to the watchlist
3. Display confirmation and options for setting rules (threshold, percent change) or removing the ticker

_Data touched:_ Watchlist entry

### set_threshold_rule
_Trigger:_ set_threshold_rule

1. Ask user for direction (above/below) and target price
2. Confirm the rule and store it
3. Display confirmation and update the watchlist entry

_Data touched:_ Watchlist entry

### set_percent_change_rule
_Trigger:_ set_percent_change_rule

1. Ask user for percent, direction (up/down/both), and time window
2. Confirm the rule and store it
3. Display confirmation and update the watchlist entry

_Data touched:_ Watchlist entry

### price_check
_Trigger:_ /price

1. If no arguments, list all watchlist items with current prices and active rules
2. If a ticker is specified, show current price and 24h change percentage

_Data touched:_ Watchlist entry

### morning_summary
_Trigger:_ scheduled_morning_summary

1. Check if current time matches user's set morning summary time
2. If outside quiet hours, send summary of watchlist prices and notable changes
3. If during quiet hours, queue summary for delivery at quiet hours end

_Data touched:_ User profile, Watchlist entry

### alert_delivery
_Trigger:_ price_update

1. Check all active rules against current prices
2. For triggered rules, send alerts with details and apply cooldown
3. Log notification records

_Data touched:_ Watchlist entry, Notification record

### owner_stats
_Trigger:_ /stats

1. Aggregate and display user count, top 20 most-triggered tickers, and rule types
2. Ensure metrics are non-identifying

_Data touched:_ Owner metrics

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User profile** _(retention: persistent)_ — Stores user-specific settings and preferences
  - fields: Telegram ID, display name, timezone, quiet hours window, morning summary time, notification preferences
- **Watchlist entry** _(retention: persistent)_ — Represents a cryptocurrency in a user's watchlist with associated rules
  - fields: user_id, ticker, display name, threshold rules, percent-change rules, last-notified state, cooldown expiry
- **Notification record** _(retention: persistent)_ — Tracks alerts sent to users for audit and metrics
  - fields: user_id, ticker, rule_id, time, old_price, new_price, percent_change
- **Owner metrics** _(retention: persistent)_ — Aggregated statistics for the bot owner
  - fields: user count, per-ticker trigger counts, per-rule trigger counts

## Integrations

- **Telegram** (required) — Bot API messaging and notifications
- **Price data provider** (required) — Fetch current and historical cryptocurrency prices
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- /stats command to view aggregated metrics
- Configuration of default currency (USD) and alert cooldowns

## Notifications

- Price alerts with rule details
- Morning summary of watchlist prices
- Error notifications for price API failures
- Owner metrics updates via /stats command

## Permissions & privacy

- User data is stored privately and not shared with the bot owner
- Only aggregated metrics are available to the bot owner
- Users can manage their watchlists and notification preferences
- Quiet hours and morning summary settings are user-controlled

## Edge cases

- Handling of unknown or invalid ticker symbols
- Price API failures and retries
- Multiple rule triggers within cooldown periods
- Morning summary during quiet hours
- Timezone and local time calculations for notifications

## Required tests

- Verify that price alerts are triggered correctly based on user-defined rules
- Test morning summary delivery at scheduled times respecting quiet hours
- Validate error handling for price API failures and retries
- Ensure owner metrics are aggregated correctly without exposing user identities
- Test cooldown periods for alert de-duplication

## Assumptions

- Default currency is USD
- Percent-change window is 1 hour
- Quiet hours suppress all notifications but queue summaries
- Threshold alert cooldown is 3 hours
- Percent-change alert cooldown equals the window duration
- Morning summary is scheduled based on user's local time
- Unknown ticker handling includes fuzzy matching and suggestions
