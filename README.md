# cytnx-deployment-analysis

Analysis of how [Cytnx](https://github.com/Cytnx-dev/Cytnx) (>= 1.1.0) is
deployed today, and a recommended deployment strategy for the maintainers.

The Cytnx source is pinned as a reference-only submodule under `external/Cytnx`
(v1.1.0). This repo proposes; it does not modify Cytnx.

## How to read this

1. [Executive summary](docs/00-executive-summary.md)
2. [Current-state audit](docs/01-current-state-audit.md)
3. [Dependency taxonomy](docs/02-dependency-taxonomy.md)
4. [Core problems](docs/03-core-problems.md)
5. [Reference projects](docs/04-reference-projects.md)
6. [Channel evaluation](docs/05-channel-evaluation.md)
7. [Recommended strategy & roadmap](docs/06-recommended-strategy.md)

## Status

Analysis phase. Prototype deployment configs are a marked follow-up (see
chapter 06).
