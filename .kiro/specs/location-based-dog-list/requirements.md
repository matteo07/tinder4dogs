# Requirements Document

## Project Description (Input)
Dog owners using tinder4dogs to find a playmate or mating partner for their dog currently get ranked suggestions that can be impractically far away: `GET /api/matches/{id}` returns **every** other scorable dog sorted best-first by compatibility score, because no location exists anywhere in the system today — neither in the dog model nor in the API.

That should change so that the ranked-candidates list becomes location-aware: each dog profile gains a stored location, and the list shows only candidates within a radius of *x* km around the subject dog's stored location, still ordered by match score from highest to lowest. Clarified with the requester:

- The search center is the location stored on the subject dog's profile (the system has no user accounts, only dogs).
- The radius *x* is supplied by the client as a query parameter, validated against a defined range, with a named-constant default.
- Candidates without a valid stored location are silently skipped from the list, mirroring the existing tolerance for unscorable candidates.
- Only the ranked-candidates endpoint changes; the pairwise lookup between two known dogs keeps today's behaviour.

Source: PRD `docs/PRD.md` feature F-05 "Location-based dog list". Geolocation consent (nLPD/GDPR) and the default/range value of *x* are flagged as open in the PRD.

## Requirements
<!-- Will be generated in /kiro-spec-requirements phase -->