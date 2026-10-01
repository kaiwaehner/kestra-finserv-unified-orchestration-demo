# The Story: Pre-Market Readiness

## The setting

Before the market opens, trading desks need current reference data: which securities exist, what changed overnight (splits, dividends), closing prices, trading calendars. The bank rebuilds this data every night from an external vendor's market data feed. If it is wrong, positions are mispriced, risk limits misfire, and in the worst case the desk trades on bad numbers. That is a risk and compliance problem, not an IT glitch.

## What happens overnight

- **Platform team:** deploys a new security certificate for the vendor connection and swaps it in during the maintenance window. The old one is expiring.
- **Feed team:** restarts the feed handler, the system that receives the vendor's data.
- **Reference data team:** builds the next day's reference data from that feed.
- **Market data support:** checks before the open that everything is ready in time.

## The fault

The certificate swap has the certificate's name hardcoded, so it points to the old, expired certificate. At the last rotation the name happened to match; this time it does not. The swap itself succeeds. The feed handler restarts successfully on that certificate but receives no data, and overnight a quiet feed looks normal. The reference data rebuild runs on whatever arrived, does not know the feed delivered nothing, and succeeds too. On a quiet day or a holiday there may genuinely be no new corporate actions, so zero records alone is not an error.

**Every step succeeds; none of them knows what the step before actually delivered.**

## When it surfaces

Only before the open, when market data support notices the reference data has not been refreshed. They know *that* something is broken, not *why*, and they do not know the feed or platform teams exist. The search spans four teams, four tools and four audit trails. By the time someone finds the wrong certificate name, the market is already open.

## The core point

No team made a mistake it could have seen on its own. The problem sits between the teams, where nobody is looking.

> Green means the task ran. It doesn't mean the outcome is right. That's why you watch the data, not just the jobs.
