# Requirements Document

## Introduction

Today `GET /api/matches/{id}` returns every other scorable dog ordered by compatibility score, regardless of how far away they are, because no location exists anywhere in the system. This feature makes the ranked-candidates list location-aware: each dog profile gains a stored location (latitude and longitude), and the ranked list shows only candidates within a client-supplied radius around the subject dog's stored location, still ordered by match score from highest to lowest. Each returned candidate also shows its distance from the subject dog, rounded to the nearest kilometre so exact positions are never exposed. The pairwise lookup between two known dogs is unchanged. Source: PRD `docs/PRD.md` feature F-05.

## Boundary Context

- **In scope**:
  - Storing a location (latitude/longitude) on a dog profile at creation, and returning it with the dog's own profile.
  - Radius query parameter on the ranked-candidates list: default 25 km, validated against the range 0–1000 km.
  - Filtering the ranked list to candidates within the radius of the subject dog's stored location.
  - Showing each candidate's distance from the subject dog, rounded to the nearest kilometre.
  - Tolerating candidates with missing or invalid stored location by silently skipping them.
- **Out of scope**:
  - Any consent mechanism for geolocation (PRD NFR-01 / nLPD/GDPR) — deferred to a later spec.
  - Updating or deleting a dog's location (no update/delete endpoints exist today).
  - Geocoding: the system accepts coordinates, not addresses, and does not convert between them.
  - Filters beyond location (PRD F-06) and any client-side presentation.
- **Adjacent expectations**:
  - Existing ranked-list behaviour that is not location-related is preserved: unscorable candidates are skipped, an unscorable subject yields 422, a missing subject yields 404, and ordering remains by score from highest to lowest.
  - Existing dogs created before this feature have no stored location; reads of those profiles must not fail because of it.

## Requirements

### Requirement 1: Location on the dog profile

**Objective:** As a dog owner, I want my dog's profile to carry its location, so that nearby playmates or mating partners can be found.

#### Acceptance Criteria

1. When a client creates a dog profile with a location, the tinder4dogs service shall store that location with the profile and return it in the created profile response.
2. When a client reads an existing dog profile, the tinder4dogs service shall return the stored location of that profile, or its absence, exactly as stored.
3. If a client creates a dog profile with a latitude outside [-90, 90] or a longitude outside [-180, 180], the tinder4dogs service shall reject the request with a client error and store no dog.
4. When a client creates a dog profile without a location, the tinder4dogs service shall create the profile and store it as having no location.

### Requirement 2: Radius parameter on the ranked-candidates list

**Objective:** As a dog owner, I want to control how far away candidates may be, so that suggested matches are practical to meet.

#### Acceptance Criteria

1. When a client requests the ranked-candidates list without a radius parameter, the tinder4dogs service shall use a radius of 25 km.
2. If a client requests the ranked-candidates list with a radius outside the range 0–1000 km or a radius that is not a number, the tinder4dogs service shall reject the request with a 422 and return no candidates.
3. When a client requests the ranked-candidates list with a radius within 0–1000 km, the tinder4dogs service shall use that radius for this request only and shall not persist it.

### Requirement 3: Location-filtered ranked candidates

**Objective:** As a dog owner, I want to see only candidates within the chosen radius of my dog's location, still ranked by compatibility, so that I can act on the suggestions.

#### Acceptance Criteria

1. When a client requests the ranked-candidates list for a dog that exists, is scorable, and has a stored location, the tinder4dogs service shall return only candidates whose stored location lies within the request's radius of that dog's location, each with its compatibility score, ordered by score from highest to lowest.
2. While a candidate has no stored location or an invalid stored location, the tinder4dogs service shall silently skip that candidate from the list rather than failing the request.
3. If the subject dog of a ranked-candidates request does not exist, the tinder4dogs service shall return a 404.
4. If the subject dog of a ranked-candidates request is not scorable or has no stored location, the tinder4dogs service shall return a 422.
5. The tinder4dogs service shall never include the subject dog itself in its ranked-candidates list.

### Requirement 4: Candidate distance in the list

**Objective:** As a dog owner, I want to see how far away each suggested dog is, so that I can judge whether a meeting is practical.

#### Acceptance Criteria

1. When the tinder4dogs service returns a ranked-candidates list, it shall include for each candidate its distance from the subject dog in kilometres, rounded to the nearest whole kilometre.
2. The tinder4dogs service shall not include any candidate's exact stored coordinates in a ranked-candidates list response.

### Requirement 5: Pairwise lookup unchanged

**Objective:** As a dog owner checking two specific known dogs, I want their compatibility answer to keep today's behaviour, so that existing clients do not break.

#### Acceptance Criteria

1. When a client requests the compatibility between two known dogs, the tinder4dogs service shall respond exactly as it does today, independent of whether either dog has a stored location.
