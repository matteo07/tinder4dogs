# Research & Design Decisions

## Summary
- **Feature**: `location-based-dog-list`
- **Discovery Scope**: Extension
- **Key Findings**:
  - Dog profiles are owned by the `dog` package, while ranked selection and pairwise scoring are owned by the `match` package.
  - The current ranked endpoint performs candidate filtering, scoring, and ordering in `MatchController`; location filtering should move into a focused match service to preserve controller boundaries.
  - Liquibase is the schema authority and existing dogs may lack locations, so the migration and runtime model must preserve legacy-row tolerance while requiring locations for new profiles.

## Research Log

### Existing dog and match integration points
- **Context**: The feature extends profile data and the ranked-candidate endpoint.
- **Sources Consulted**: `Dog.kt`, `DogController.kt`, `DogRepository.kt`, `MatchController.kt`, `MatchScoreService.kt`, `product.md`, `structure.md`.
- **Findings**:
  - `Dog` is a mutable JPA entity with profile fields and eager preferences.
  - `DogController` owns profile creation and reads; there are currently no update endpoints.
  - `MatchController` owns both ranked candidate orchestration and pairwise lookup, although steering identifies ranked orchestration as domain logic that belongs in a service.
  - `MatchScoreService.canScore` allows invalid candidate rows to be skipped without failing a list request.
- **Implications**:
  - Add location request/response data to the dog vertical slice.
  - Add a location update endpoint without introducing a generic profile update API.
  - Introduce a focused match-list service for radius validation, location filtering, distance calculation, score ordering, and legacy-data handling.

### Persistence and migration constraints
- **Context**: Location is new persisted dog data.
- **Sources Consulted**: `db.changelog-master.yaml`, existing SQL changesets, `tech.md`.
- **Findings**:
  - Schema changes use append-only plain SQL changesets under `src/main/resources/db/changelog/changes/`.
  - Hibernate validates the schema; it does not update it.
  - Existing seed and legacy rows predate location and must remain readable.
- **Implications**:
  - Add a new numbered changeset and append it to the master index.
  - Keep persisted location nullable for existing rows; enforce mandatory location at create and update HTTP boundaries rather than making the migration fail on legacy rows.
  - Invalid legacy coordinates are treated as unusable candidate data and do not fail the whole ranked request.

### Geospatial calculation and privacy
- **Context**: The list must filter by kilometres and return rounded distance without exposing coordinates.
- **Sources Consulted**: Product requirements, `docs/PRD.md` F-05 and NFR-02, project dependency list.
- **Findings**:
  - No geospatial dependency, PostGIS integration, or location implementation exists.
  - The required scale is bounded to 0–1000 km and the current application computes matches in memory.
  - Responses need a rounded distance field and must omit exact coordinates from candidate list responses.
- **Implications**:
  - Use a small typed domain value for coordinates and a standard great-circle distance calculation in the match-list service; no new dependency or database extension is required.
  - Validate finite latitude/longitude values and coordinate ranges at request boundaries and again before using persisted values.
  - Use kilometres consistently; radius 0 includes only candidates at zero calculated distance.

### Compatibility and runtime
- **Context**: The feature changes existing HTTP payloads and adds a query parameter.
- **Sources Consulted**: `tech.md`, `structure.md`, existing controllers and build conventions.
- **Findings**:
  - Kotlin official style, constructor injection, Spring MVC, JPA, Jakarta validation, and unit-level JUnit 5 tests are mandatory conventions.
  - There is no central exception handler; controllers use `ResponseEntity` for non-200 outcomes.
  - Existing endpoint pairwise lookup must remain location-independent.
- **Implications**:
  - Keep the existing endpoint path and add an optional `radiusKm` query parameter with a named default constant.
  - Map invalid radius and missing subject location to 422 through the controller/service result boundary without changing pairwise behavior.
  - Add focused unit tests for geospatial and list-selection behavior; retain direct service construction without a Spring context.

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations | Notes |
|--------|-------------|-----------|---------------------|-------|
| Existing vertical slices with focused match service | Keep profile persistence/API in `dog`; move ranked-list rules into `match` | Matches repository structure, testable, small change | Adds one service to an existing controller flow | Selected |
| Database-side geospatial query | Filter candidates in PostgreSQL | Could reduce application-side rows at scale | Requires spatial expressions or PostGIS, complicates legacy invalid data and current unit-test model | Rejected for current scope |
| New geospatial dependency | Adopt a third-party distance library | Less custom math | New dependency is unnecessary for bounded point distance and increases compatibility surface | Rejected |

## Design Decisions

### Decision: Store coordinates as nullable profile fields
- **Context**: New profiles require location, but existing rows have none.
- **Alternatives Considered**:
  1. Make database columns immediately non-null — incompatible with existing rows.
  2. Keep columns nullable and validate new writes — preserves compatibility while enforcing the product rule at the API boundary.
- **Selected Approach**: Add nullable latitude and longitude persistence fields; require a complete valid pair for creation and location updates.
- **Rationale**: Existing data remains readable, and the ranked list can apply the specified legacy tolerance.
- **Trade-offs**: Database-level non-null enforcement is unavailable until legacy data is migrated; application validation is authoritative for new writes.
- **Follow-up**: Verify Hibernate validation against the new migration and test partial/null legacy data.

### Decision: Add a dedicated ranked-list service
- **Context**: Location filtering, distance output, score filtering, and ordering are domain rules currently in a controller.
- **Alternatives Considered**:
  1. Extend `MatchController` — smallest file count but violates the existing boundary guidance.
  2. Add a dedicated service — one new component with a clear contract and direct unit tests.
- **Selected Approach**: `MatchListService` owns ranked candidate selection and returns typed list results; `MatchController` maps service outcomes to HTTP.
- **Rationale**: Keeps controllers focused on transport and allows geospatial behavior to be tested without Spring.
- **Trade-offs**: Pairwise lookup remains in the controller until a separate refactor; two match flows coexist temporarily.
- **Follow-up**: Ensure no pairwise endpoint behavior is changed.

### Decision: Use a dedicated location update endpoint
- **Context**: Location is mandatory but only location updates are in scope; a generic profile update endpoint is not required.
- **Alternatives Considered**:
  1. Add `PUT /api/dogs/{id}` for the full profile — expands scope and changes unrelated fields.
  2. Add `PUT /api/dogs/{id}/location` — focused, idempotent replacement of the location pair.
- **Selected Approach**: `PUT /api/dogs/{id}/location` accepts a complete coordinate pair and returns the updated dog profile.
- **Rationale**: Makes removal impossible by contract and limits write ownership to location fields.
- **Trade-offs**: Clients cannot atomically update other profile fields through this endpoint.
- **Follow-up**: Test missing/partial request fields as client errors and verify unrelated profile fields remain unchanged.

### Decision: Keep exact coordinates out of ranked candidate responses
- **Context**: NFR-02 requires privacy-preserving distance display.
- **Alternatives Considered**:
  1. Return exact candidate coordinates — violates the requirement.
  2. Return only whole-kilometre distance — satisfies the current requirement without exposing coordinates.
- **Selected Approach**: Dog profile responses expose the subject's own stored coordinates; ranked candidate responses expose only `distanceKm` rounded to the nearest whole kilometre.
- **Rationale**: Separates profile ownership from candidate discovery privacy.
- **Trade-offs**: Rounded distances lose precision intentionally.
- **Follow-up**: Add response-shape tests that fail if candidate coordinates are serialized.

## Synthesis Outcomes

- **Generalization**: Coordinate validation is one reusable domain rule applied to create, update, and persisted candidate data; the list service should distinguish valid points from unusable legacy points without duplicating range logic.
- **Build vs. adopt**: Existing Kotlin/JVM math and typed value objects are sufficient; no geospatial library or database extension is justified by the current bounded, in-memory workload.
- **Simplification**: The feature needs one location value concept and one ranked-list service, not a general geospatial subsystem, search abstraction, or generic dog profile update service.

## Risks & Mitigations
- Nullable legacy location fields may be encountered during candidate selection — validate persisted pairs and skip unusable candidates.
- Distance calculations near the international date line or poles may be incorrect if implemented as planar distance — use a great-circle calculation with normalized longitude difference.
- Existing HTTP clients may expect the old candidate response shape — retain existing fields and add only `distanceKm`; do not alter pairwise responses.
- A failed location update could partially mutate an entity before persistence — validate the complete request before changing the entity.

## References
- `src/main/kotlin/com/ai4dev/tinder4dogs/dog/Dog.kt` — current dog persistence model.
- `src/main/kotlin/com/ai4dev/tinder4dogs/dog/DogController.kt` — current dog HTTP contracts.
- `src/main/kotlin/com/ai4dev/tinder4dogs/match/MatchController.kt` — current ranked and pairwise endpoints.
- `src/main/resources/db/changelog/db.changelog-master.yaml` — migration index and append-only rule.
- `docs/PRD.md` F-05 and NFR-02 — product location and privacy constraints.
