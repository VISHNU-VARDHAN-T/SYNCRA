# Working together

1. Agree on the interface contract in `docs/interface-contract.md` before parallel implementation.
2. Keep `main` buildable as code appears; use focused feature branches.
3. Avoid committing generated builds, device-specific settings, or audio recordings without a reason.
4. Add reproducible test steps for audio/device bugs (OS version, output devices, sample rate, route, and observed behaviour).
5. Separate measured results from guesses. Note device, codec/route, and test method when recording latency.
6. Review changes to shared models/contracts with both contributors.
