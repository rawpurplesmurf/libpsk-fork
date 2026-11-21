# Changelog

All notable changes to this fork are documented in this file.

## [Unreleased]

### Security

- Disabled standalone `psk_tcpmodem` (`PSK_BUILD_APPS=OFF`) and CLI tools (`PSK_BUILD_TOOLS=OFF`) by default so builds produce only `libpsk.a`, and restricted `psk_tcpmodem` socket binding from `INADDR_ANY` (`0.0.0.0`) to `INADDR_LOOPBACK` (`127.0.0.1`).

### Fixed

- Renamed C test directory `tests/` to `c_tests/` so CodeWizard Python `pytest` auto-discovery does not exit with status 5 while keeping all 5 `ctest` suites intact.
