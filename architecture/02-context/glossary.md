# Architecture Glossary

**Stability:** Evolving — grows as new architecture-specific terms are introduced.
**Status:** Skeleton — pending authoring.

## Purpose
Defines terms that belong to the *architecture* vocabulary (e.g., "module boundary," "source-of-truth module," "read model," "backpressure") — terms about how the system is built, not what the business means by them.

## Relationship to other documents
- **DDD:** This glossary must never redefine or duplicate the DDD's entity/data terminology. It links to the DDD where database-design terms are already defined. If a term means something different at the architecture layer than at the data-design layer, that distinction is called out explicitly here to prevent drift.
- **SRD / SDD / Implementation:** Any architecture term used in an SDD document or in code comments/ADR text should be traceable to an entry here.

## Sections to be authored
1. Term list (alphabetical), each with a one-line definition and a pointer to the document where it is used most
2. Explicit "do not confuse with" cross-references to SRD/DDD terms where naming could collide
