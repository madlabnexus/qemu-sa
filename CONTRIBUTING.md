# Contributing to QEMU-SA

## Getting Started

1. Fork the repository
2. Set up the development environment (see docs/01-SETUP-NOTEBOOK.md)
3. Create a feature branch: `git checkout -b feature/my-feature`
4. Make your changes
5. Test thoroughly (see Testing below)
6. Commit with clear messages
7. Push and open a Pull Request

## Code Style

- Follow QEMU's existing coding style for C code (see QEMU's CODING_STYLE.rst)
- 4-space indentation for C
- No trailing whitespace
- Meaningful commit messages: "hw/audio: integrate Nuked-OPL3 as ISA device"

## Testing

Before submitting:
- Ensure QEMU-SA builds cleanly: `make -j$(nproc)`
- Test with at least one DOS game (DOOM for OPL3, Monkey Island for MIDI)
- Test Win98 boot if your changes affect KVM path
- Run `make check` for QEMU's built-in tests

## Architecture Decisions

If your change involves a significant design choice, add an entry to docs/DECISIONS.md following the ADR format.

## License

All contributions must be compatible with GPL v2.0. By submitting a PR, you agree that your contribution is licensed under GPL v2.0.

## Reporting Issues

- Use GitHub Issues
- Include: QEMU-SA version, host OS, guest OS, game/software tested
- For sound issues: describe what you hear vs what you expect
- For graphics issues: screenshots help enormously
