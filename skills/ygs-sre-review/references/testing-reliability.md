# Testing for Reliability

## Test Pyramid (Google SRE)
- **80% unit tests** — Fast, isolated, test public APIs not internals
- **15% integration tests** — Verify component interactions
- **5% end-to-end tests** — Full system validation

## Testing Principles
- Test behavior through public interfaces, not implementation details
- Minimize mocking — prefer state-based testing over interaction testing
- Hermetic testing in isolated environments for determinism
- Property-based testing for combinatorial explosion scenarios
- Contract testing between services (not mocks from docs)
- Beyonce Rule: "If you liked it, you shoulda put a test on it"

## Reliability-Specific Testing

### Chaos / Fault Injection
- Test service unavailability scenarios
- Test high-latency and timeout conditions
- Test partial failure (3 of 5 nodes down)
- Verify monitoring and alarms fire correctly
- Validate recovery procedures actually work
- Test cascading failure prevention (circuit breakers)

### Distributed Systems Testing
- **Metastable failure testing** — Model positive feedback loops in retry logic, test timeout thresholds against load
- **Cross-system interaction (CSI)** — Write-read verification across systems, round-trip serialization, version compatibility
- **Consistency tradeoff testing** — Measure actual latency cost of consistency choices under varying network conditions
- **CRDT convergence** — Observe replica divergence/convergence during partitions

### Simulation-Based Testing
- Interactive simulators for boundary conditions hard to reproduce in real environments
- Parameter tweaking with instant feedback (no cloud cost)
- Emergent behavior discovery beyond component-level testing

### Property-Based / Generative Testing
- Generate random inputs meeting preconditions, validate properties across iterations
- Fuzzing for edge cases and failure scenarios difficult to reproduce manually
- Mock services from OpenAPI specs with randomized responses
- Contract-based scenarios with custom assertions executed repeatedly

### Load & Performance
- Simulate real-world use cases with varied data sizes
- Test both primary services AND dependencies under load
- Validate autoscaling policies
- Test without caching during peak (what if cache fails?)
- Measure cold-start times

### Canary & Progressive Rollout
- Deploy to limited user subset with real-time monitoring
- Auto-rollback on test failure
- Include both common and edge-case scenarios
- Phased: edit/compile → pre-submit → post-submit → release-candidate → promotion

## Caching Test Scenarios
- Thundering herd after cache expiry (all clients hit backend simultaneously)
- Stale data serving during update failures
- Cache unavailability causing cascading failures
- Memory leaks from improper cache management
- Security: cache keys must respect authorization boundaries
- Performance without cache during peak traffic
