# The Story of One Night and the Trading Day After

## The setting

Before the market opens, trading desks need current reference data: which securities exist, what changed overnight (splits, dividends), closing prices, trading calendars. The bank rebuilds this data every night from an external vendor's market data feed. If it is wrong, positions are mispriced, risk limits misfire, and in the worst case the desk trades on bad numbers. That is a risk and compliance problem, not an IT glitch.

## What happens overnight

Four teams share the night. The platform team deploys a new security certificate for the vendor connection and swaps it in during the maintenance window, because the old one is expiring. The feed team restarts the feed handler, the system that receives the vendor's data. The reference data team builds the next day's reference data from that feed. Market data support checks before the open that everything is ready in time.

## The fault

The certificate swap has the certificate's name hardcoded, so it points to the old, expired certificate. At the last rotation the name happened to match; this time it does not. The swap itself succeeds. The feed handler restarts successfully on that certificate but receives no data, and overnight a quiet feed looks normal. The reference data rebuild runs on whatever arrived, does not know the feed delivered nothing, and succeeds too. On a quiet day or a holiday there may genuinely be no new corporate actions, so zero records alone is not an error.

**Every step succeeds; none of them knows what the step before actually delivered.**

## When it surfaces

Only before the open, when market data support notices the reference data has not been refreshed. They know *that* something is broken, not *why*, and they do not know the feed or platform teams exist. The search spans four teams, four tools and four audit trails. By the time someone finds the wrong certificate name, the market is already open.

## The core point

No team made a mistake it could have seen on its own. The problem sits between the teams, where nobody is looking.

> Green means the task ran. It doesn't mean the outcome is right. That's why you watch the data, not just the jobs.

## The trading day after the night

Once the cause is fixed, the reference data is fresh again and the desks can trade. Orders do not arrive one at a time through a form. They arrive as events on Kafka, and Flink checks each one for fraud and against the limits in milliseconds. The fast execution path stays in the order management system. Only orders that need a documented process, above 250,000 USD, are published to a workflow topic. Kafka can deliver an event twice after a consumer restart, and a second execution of the same order would mean a second trade, so each order runs exactly once. A standard order is checked, recorded and goes back to the order management system without approval.

The first order that matters is a big one. A pension fund client wants to buy 50,000 shares of ACME at a limit of 50 USD, a notional of 2.5 million USD. An order this size is not a single API call. It is a long-running transaction that starts in the morning and only ends when the trade settles on the next business day, T+1.

Three teams own the steps, and a fourth owns the process. Core banking reserves the buying power, so the client cannot spend the same money twice. Risk analytics checks the order against the bank's limits, using the reference data from the night before. The trading platform routes the order to the exchange, where an execution algorithm slices it into smaller pieces over the day so it does not move the price. Trading operations owns the chain from the first check to the settlement.

## Where the trade can go wrong

The check finds that this order breaches a soft limit on portfolio concentration. A soft limit does not reject the order outright, it needs a risk officer to decide. That decision has a deadline, because the client expects the order in the market today, and an approval that sits unanswered for hours is a missed trade. If nobody decides within 15 minutes, the order fails and the reserved buying power is released.

What the process does when a step fails depends on how far the trade has come. Before execution, nothing has happened in the market yet, so the process releases the reserved buying power and the order is simply rejected. During execution, an open order sits at the exchange, so the process cancels it and releases the money. After execution, the trade is real, the shares are bought and the money is owed. There is nothing left to roll back. A settlement fail on T+1 goes to operations as an escalation case, and people resolve it.

Whatever the outcome, an audit entry records the order, the execution and the final state. In a regulated bank, a trade without a complete record is a problem in its own right.

## The second point

Each team owns its step, and the process owns the chain. Without someone owning the chain, the reserved money stays blocked after a rejection, the open order keeps running after a failure, and nobody sees which state the trade is in. The same lesson as the night applies here: the risk sits between the teams, so the orchestration has to cover the whole chain, not just the individual steps.

## The platform underneath

Flink and Fluss run underneath the trading day. Flink checks every order, and Fluss holds the reference data the check looks up. Both belong to the streaming platform team, and no workflow sits in their data path. That part is a stream and it stays a stream.

Changing that platform is a different matter. A new version of the pre-trade check during market hours is a risk of its own, so it needs a savepoint first, a second person to approve, a health check that looks at what the business sees, and a way back when the new version does not behave. Loading the nightly reference data into Fluss is a batch job with its own lineage: if the night went wrong, as in Demo 1, the stale data reaches every limit check of the day.

## The third point

The same orchestration that runs the trade also runs the changes to the platform underneath, in a different domain and for a different team. Real-time data beats slow data, and still not every use case is a stream.
