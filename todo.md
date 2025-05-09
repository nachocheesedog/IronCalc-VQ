# Project Todo List

This file tracks tasks, bug fixes, feature requests, and refactoring items for the IronCalc-VQ project.

## Table of Contents

*   [Tasks](#tasks)
*   [Bug Fixes](#bug-fixes)
*   [Feature Requests](#feature-requests)
*   [Refactoring](#refactoring)
*   [Security Checklist](#security-checklist)

## Tasks

*   [X] (2025-05-08) Familiarize with the entire codebase.
    *   [X] (2025-05-08) Explored top-level directory structure.
    *   [X] (2025-05-08) Explored `base/` directory structure.
    *   [X] (2025-05-08) Examined `base/src/lib.rs`.
    *   [X] (2025-05-08) Examine `base/src/model.rs`.
    *   [X] (2025-05-08) Examine `base/src/workbook.rs`.
    *   [X] (2025-05-08) Examine `base/src/types.rs`.
    *   [X] (2025-05-08) Examine `base/src/worksheet.rs`.
    *   [X] (2025-05-08) Examine `base/src/cell.rs`.
    *   [X] (2025-05-08) Examine `base/src/expressions/parser/mod.rs`.
    *   [X] (2025-05-08) Investigate formula evaluation initiation and flow in `base/src/model.rs` (via `evaluate`, `evaluate_cell`, `evaluate_node_in_context`).
    *   [X] (2025-05-08) Examine `base/src/functions/mathematical.rs` for built-in function implementation.
    *   [X] (2025-05-08) Examine `base/src/calc_result.rs` for the `CalcResult` enum definition.

## Bug Fixes

*(No bug fixes yet)*

## Feature Requests

*(No feature requests yet)*

## Refactoring

*(No refactoring items yet)*

## Security Checklist

*   [ ] Input Validation: Ensure all user inputs are validated and sanitized.
*   [ ] Output Encoding: Ensure data is properly encoded before display to prevent XSS.
*   [ ] Authentication & Authorization: Implement robust mechanisms if applicable.
*   [ ] Session Management: Secure session handling if applicable.
*   [ ] Error Handling & Logging: Implement proper error handling and secure logging.
*   [ ] Dependency Management: Regularly update and audit dependencies for vulnerabilities.
*   [ ] Data Protection: Secure sensitive data at rest and in transit.
*   [ ] API Security: Follow best practices for API design and security (e.g., rate limiting, authentication).
*   [ ] Infrastructure Security: Secure deployment environments.
