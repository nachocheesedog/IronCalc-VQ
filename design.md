# IronCalc-VQ Design Document

This document outlines the architectural design and decisions for the IronCalc-VQ project.

## Table of Contents

*   [Overview](#overview)
*   [Core Engine (`base` crate)](#core-engine-base-crate)
*   [Data Model](#data-model)
*   [API Design](#api-design)

## Overview

*(To be populated as more is understood about the project)*

## Core Engine (`base` crate)

The `base` crate appears to be the core calculation engine of IronCalc.

Key observations from `base/src/lib.rs` (2025-05-08):
*   **Entry Point:** `base/src/lib.rs` defines the public API for the crate.
*   **Public Modules:** Exposes modules like `cell`, `expressions`, `formatter`, `language`, `locale`, `new_empty`, `number_format`, `types`, `worksheet`.
*   **Core Structs:** The `Model` (from `model.rs`) and `UserModel` (from `user_model.rs`) structs are publicly exported and seem central to the engine's operation.
*   **Functionality:** Likely handles parsing formulas, calculations, cell management, workbook/worksheet structures, and data type management.

### `Model` Struct (`base/src/model.rs`)

The `Model` struct is a cornerstone of the engine (identified 2025-05-08). It encapsulates:
*   **Workbook Data:** Contains a `workbook: Workbook` field, holding all sheets, cells, styles, etc.
*   **Formula Engine:**
    *   `parsed_formulas: Vec<Vec<Node>>`: Stores Abstract Syntax Trees (ASTs) of formulas.
    *   `parser: Parser`: An instance of the formula parser.
    *   Manages defined names (`parsed_defined_names`).
*   **Evaluation State:** Tracks cell evaluation status (`cells: HashMap<..., CellState>`).
*   **Context:** Manages `Locale`, `Language`, and `TimeZone` for calculations.
*   **String Optimization:** Uses `shared_strings` for efficient string handling.
*   **Rich API:** Provides a comprehensive set of methods for:
    *   Evaluating formulas and cells.
    *   Getting/setting cell values, styles, and formats.
    *   Managing sheets (creation, deletion, properties).
    *   Handling rows/columns (insertion, deletion, styles).
    *   Working with defined names.
    *   Various utility functions for cell and sheet properties.

### Formula Parsing (`base/src/expressions/parser/mod.rs` - identified 2025-05-08)
The formula parser is responsible for converting raw formula strings into an Abstract Syntax Tree (AST), which is a structured, hierarchical representation of the formula.

*   **Grammar:** Adheres to a defined grammar specifying operator precedence (Comparison > Concatenation > Add/Subtract > Multiply/Divide > Power > Unary/Percent > Range > Implicit Intersection > Primary).
*   **`Node` Enum (AST):** Defines the various types of nodes in the AST:
    *   Literals: `NumberKind`, `StringKind`, `BooleanKind`.
    *   References: `ReferenceKind` (single cell), `RangeKind`.
    *   Operators: `OpSumKind`, `OpProductKind`, `OpPowerKind`, `OpConcatenateKind`, `OpCompareKind`, `OpRangeKind`, `UnaryKind`.
    *   Functions: `FunctionKind`, `InvalidFunctionKind`.
    *   Arrays: `ArrayKind` (for constants like `{1,2}`).
    *   Named Ranges: `DefinedNameKind`.
    *   Table References: `TableNameKind`.
    *   Error Nodes: `ErrorKind`, `ParseErrorKind`.
*   **`Parser` Struct:** Manages the parsing process.
    *   Contains a `lexer` to tokenize the formula.
    *   Maintains context like worksheet names (`worksheets_map`), defined names (`defined_names_map`), table structures (`tables`), and the reference of the cell being parsed (`current_context`).
*   **Parsing Strategy:** Uses recursive descent. Methods like `parse_expr`, `parse_term`, `parse_factor`, `parse_primary`, etc., correspond to grammar rules.
*   **Output:** The `parse` method returns the root `Node` of the AST.
*   **Error Handling:** Generates `ParseErrorKind` nodes for syntax errors.
*   **Sub-modules:** Includes `move_formula` (for adjusting references), `static_analysis` (for formula analysis), and `stringify` (AST to string).

### Formula Evaluation (`base/src/model.rs` and `base/src/functions/` - identified 2025-05-08)
Once a formula string is parsed into an AST (`Node`), the `Model` evaluates it.

*   **Evaluation Trigger:** The main `Model::evaluate()` method iterates through all cells and calls `Model::evaluate_cell()` for each.
*   **`Model::evaluate_cell(cell_ref)`:**
    *   If the cell has no formula, its direct value is retrieved.
    *   If it has a formula, it retrieves the corresponding AST `Node` (from `Model.parsed_formulas`).
    *   It then calls `Model::evaluate_node_in_context(node, cell_ref)`.
    *   Manages cell evaluation state (`CellState::Evaluating`, `CellState::Evaluated`) in `Model.cells` to handle dependencies and detect circular references.
    *   The result is stored back in the cell (e.g., `CellFormulaNumber{ v: ... }`).
*   **`Model::evaluate_node_in_context(node, cell_ref) -> CalcResult`:**
    *   This is the core recursive AST traversal function.
    *   It `match`es on the `Node` type:
        *   **Literals:** Return `CalcResult` with the value.
        *   **Operators:** Recursively evaluate operands, then perform the operation.
        *   **References:** Evaluate the referenced cell(s) (potentially calling `evaluate_cell` again) or return a range.
        *   **Functions (`FunctionKind`):** Calls `Model::evaluate_function(kind, args, cell_ref)`.
*   **`Model::evaluate_function(kind, args, cell_ref) -> CalcResult`:**
    *   Located in `base/src/functions/mod.rs` (and its sub-modules like `mathematical.rs`, `logical.rs`).
    *   `match`es on the `Function` enum variant.
    *   Dispatches to specific function implementations (e.g., `Model::fn_sum(args, cell_ref)`).
    *   These implementations evaluate their arguments (which are `Node`s) and compute the function's result.
*   **`CalcResult` Enum (`base/src/calc_result.rs` - identified 2025-05-08):**
    *   The primary return type for all evaluation functions.
    *   Variants: `String(String)`, `Number(f64)`, `Boolean(bool)`, `Error { error, origin, message }`, `Range { left, right }`, `EmptyCell`, `EmptyArg`, `Array(Vec<Vec<ArrayNode>>)`.
    *   Includes helper methods like `new_error()` and `is_error()`.
    *   Implements comparison traits (`Ord`, `PartialEq`) to define sorting and equality for different result types, which is crucial for logical operations and certain functions.
*   **Built-in Function Implementation (e.g., `base/src/functions/mathematical.rs` - identified 2025-05-08):**
    *   Functions are methods of the `Model` struct.
    *   They take unevaluated `Node` arguments and the `CellReferenceIndex` context.
    *   Commonly check argument counts and evaluate arguments using `model.evaluate_node_in_context()` or helper like `model.get_number()`.
    *   Handle different `CalcResult` types from argument evaluation (e.g., direct numbers, iterating over ranges, propagating errors).
    *   Utilize macros like `single_number_fn!` for boilerplate reduction in simple single-argument math functions.
*   **Dependency Management:** The evaluation process implicitly handles dependencies as `evaluate_cell` calls can trigger evaluations of other precedent cells.

## Data Model

The core data structures are primarily defined in `base/src/types.rs` (identified 2025-05-08). They are designed to represent a spreadsheet in memory and support serialization via `serde` and `bitcode`.

### `Workbook` Struct (`types.rs`)
Top-level container for all spreadsheet data.
*   `name: String`: Name of the workbook.
*   `settings: WorkbookSettings`: Contains `tz` (timezone) and `locale`.
*   `metadata: Metadata`: Application version, creator, modification dates.
*   `shared_strings: Vec<String>`: List of unique strings for optimization.
*   `defined_names: Vec<DefinedName>`: Named ranges or formulas.
*   `worksheets: Vec<Worksheet>`: Collection of individual sheets.
*   `styles: Styles`: Holds all styling information (fonts, fills, borders, number formats, cell styles).
*   `tables: HashMap<String, Table>`: Structured tables within sheets.
*   `views: HashMap<u32, WorkbookView>`: Workbook view configurations.

### `Worksheet` Struct (`types.rs`)
Represents a single sheet.
*   `name: String`: Sheet name.
*   `sheet_id: u32`: Unique identifier.
*   `dimension: String`: Range of used cells (e.g., "A1:Z100").
*   `sheet_data: SheetData` (alias for `HashMap<i32, HashMap<i32, Cell>>`): Stores cell data, indexed by row then column.
*   `cols: Vec<Col>`: Column properties (width, style).
*   `rows: Vec<Row>`: Row properties (height, style).
*   `state: SheetState`: Enum (`Visible`, `Hidden`, `VeryHidden`).
*   `merge_cells: Vec<String>`: List of merged cell ranges (e.g., "A1:B2").
*   `shared_formulas: Vec<String>`: List of shared formula strings.
*   `color: Option<String>`: Tab color.
*   `comments: Vec<Comment>`.
*   `frozen_rows: i32`, `frozen_columns: i32`: For frozen panes.
*   `views: HashMap<u32, WorksheetView>`: Sheet view configurations.
*   `show_grid_lines: bool`.

**Associated Methods (`base/src/worksheet.rs` - identified 2025-05-08):**
The `impl Worksheet` block provides extensive functionality:
*   **Cell Access & Modification:** `cell()`, `cell_mut()`, `update_cell()`, and specific setters like `set_cell_with_number()`, `set_cell_with_formula()`, etc.
*   **Styling:** Methods to get/set styles for the sheet, columns, rows, and individual cells (`get_style()`, `set_column_style()`, `set_cell_style()`).
*   **Content Clearing:** `cell_clear_contents()`, `cell_clear_contents_with_style()`.
*   **Layout & View:** Managing frozen panes (`set_frozen_rows()`), row heights (`set_row_height()`), column widths (`set_column_width()`), and merged cells (`set_merge_cell()`).
*   **Dimension Management:** Calculating and updating the sheet's data bounds (`get_worksheet_dimension()`, `expand_worksheet_dimension()`).
*   **Row/Column Operations:** `insert_row()`, `delete_row()`, `insert_column()`, `delete_column()`. These methods handle shifting cell data in the `sheet_data` HashMap.
*   **Utilities:** Checking for empty cells/rows, finding data extents, navigation helpers.

### `Cell` Enum (`types.rs`)
Represents the content and type of a single cell. Most variants include an `s: i32` field, which is an index to a style in `Workbook.styles.cell_xfs`.
*   `Empty`: No content.
*   `Boolean { v: bool, s: i32 }`
*   `Number { v: f64, s: i32 }`
*   `Error { ei: Error, s: i32 }` (where `Error` is an enum for formula errors like `#VALUE!`, `#REF!`)
*   `SharedString { si: i32, s: i32 }`: Index `si` into `Workbook.shared_strings`.
*   `String { v: String, s: i32 }`
*   `CellFormula { f: i32, s: i32 }`: `f` is an index to `Worksheet.shared_formulas` or a direct formula reference.
*   `CellFormulaBoolean { f: i32, v: bool, s: i32 }`: Formula with a cached boolean value.
*   `CellFormulaNumber { f: i32, v: f64, s: i32 }`: Formula with a cached numeric value.
*   `CellFormulaString { f: i32, v: String, s: i32 }`: Formula with a cached string value.
*   `CellFormulaError { f: i32, ei: Error, s: i32, o: String, m: String }`: Formula with a cached error value, origin `o`, and message `m`.

**Associated Methods (`base/src/cell.rs` - identified 2025-05-08):**
The `impl Cell` block provides:
*   **Constructors:** `new_string()`, `new_number()`, `new_formula()`, etc.
*   **Formula Access:** `get_formula()`, `has_formula()`.
*   **Style Management:** `get_style()`, `set_style()`.
*   **Type Information:** `get_type()` (returns `CellType` enum for Excel's `TYPE()` function).
*   **Value Retrieval:**
    *   `value(shared_strings, language) -> CellValue`: Resolves the `Cell` into a simpler `CellValue` enum. Unevaluated formulas return an error string; evaluated formula cells return their cached value.
    *   `get_text(shared_strings, language) -> String`: Basic string representation.
    *   `formatted_value(shared_strings, language, formatter_fn) -> String`: Produces a display string, using a provided function for number formatting.

### `CellValue` Enum (`base/src/cell.rs` - identified 2025-05-08)
A simpler representation of a resolved cell value, used internally after evaluation.
*   `None`
*   `String(String)`
*   `Number(f64)`
*   `Boolean(bool)`
Equipped with `From` trait implementations for easy conversion from primitive types.

### `Styles` Struct (`types.rs`)
Container for all styling information.
*   `num_fmts: Vec<NumberFormat>`: Custom number formats.
*   `fonts: Vec<Font>`.
*   `fills: Vec<Fill>`.
*   `borders: Vec<Border>`.
*   `cell_style_xfs: Vec<CellStyleXfs>`: Master formatting records for named styles (e.g., "Good", "Bad").
*   `cell_xfs: Vec<CellXfs>`: Master formatting records applied directly to cells. The `s` in `Cell` variants typically indexes this vector.
*   `cell_styles: Vec<CellStyles>`: Definitions of named cell styles.

### `DefinedName` Struct (`types.rs`)
*   `name: String`: The defined name (e.g., "SalesTaxRate").
*   `formula: String`: The formula or cell reference it points to (e.g., "0.075" or "Sheet1!A1").
*   `sheet_id: Option<u32>`: Scope of the defined name (None for global, Some(sheet_id) for local).

### Other Key Structures from `types.rs`
*   `Row`: Defines properties for a row (height, style index `s`, hidden, etc.).
*   `Col`: Defines properties for a column (min/max col index, width, style index `s`, hidden, etc.).
*   `Comment`: Represents a cell comment.
*   `Table`, `TableColumn`: For Excel-like structured tables.
*   Various enums for `BorderStyle`, `FillPatternType`, `HorizontalAlignment`, `VerticalAlignment`, etc.

### `Workbook` Struct (definition likely in `types.rs`, methods in `workbook.rs`)
The `Workbook` struct is the container for all spreadsheet data (identified 2025-05-08). Methods in `base/src/workbook.rs` provide access to:
*   **Worksheets:**
    *   Retrieving lists of worksheet names (`get_worksheet_names`) and IDs (`get_worksheet_ids`).
    *   Accessing individual `Worksheet` objects (immutable `worksheet` and mutable `worksheet_mut`).
*   **Defined Names:**
    *   `get_defined_names_with_scope`: Retrieves defined names along with their scope (global or sheet-specific).

It's assumed the `Workbook` struct itself (defined in `types.rs`) holds collections of `Worksheet` objects, `DefinedName` objects, `Style` objects, and other workbook-level information like locale and timezone settings.

## API Design

*(To be populated)*
