# TankSmart

Local analytics over German fuel price history.

TankSmart loads the complete historical fuel price dataset for all German filling
stations into a local DuckDB database and exposes it through a Streamlit dashboard,
to answer one question: **when is the best time to refuel at the stations I care
about?**

It is a batch, single-user, local-first analytics project — no server, no live API,
no accounts.

## Status

Pre-implementation. See [`docs/SCOPE.md`](docs/SCOPE.md) for goals, non-goals,
architecture, data model, roadmap and open questions.

## Data

Fuel price data is provided by [Tankerkönig](https://creativecommons.tankerkoenig.de/).
The historical dataset is licensed CC BY-NC-SA 4.0; verify the current terms and
include the required attribution before publishing anything derived from it.
