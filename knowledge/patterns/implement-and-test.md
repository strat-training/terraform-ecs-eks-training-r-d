# Pattern: Implement → Test Happy Path → Test Failure Path

Used throughout this course, not just in the capstone.

1. **Implement** the script/config for the task at hand.
2. **Test the happy path** — valid input, confirm the expected output.
3. **Test the failure path** — deliberately broken input (wrong column count, missing file, denied permission), confirm the script fails the way you intended (logged, routed to `rejected/`, non-zero exit code) rather than silently or with a crash.

Any module lab that produces a script (M9, M10, M11, Capstone) should be tested against both a valid and an invalid input before you consider it done — this is exactly what the capstone's required failure-injection run checks for.
