# Create an agent skill for testing interactive TTY commands. Agents currently run

**Date:** 2026-02-20
**Author:** mnbiehl
**Status:** new

## Description

Create an agent skill for testing interactive TTY commands. Agents currently run shell commands without a TTY, making it difficult to fully test features that change behavior based on TTY status (like 'reins feedback'). Consider teaching agents to use the 'script' utility, 'expect', 'socat', or Python's 'pexpect'/'pty' modules to simulate an interactive terminal for end-to-end testing.

## Notes

_No additional notes yet._
