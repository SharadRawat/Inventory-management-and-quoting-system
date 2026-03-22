# Munder Difflin Multi-Agent System Project Reflection Report

This report reflects on the design, implementation, and evaluation of the multi-agent inventory management and quoting system built for Munder Difflin, a paper supply company.

## Agentic Workflow Diagram

### Architecture Overview

The system follows a **hierarchical orchestrator pattern** using the `smolagents` framework's `ToolCallingAgent` class. An **Orchestrator agent** sits at the top and delegates tasks to three specialized sub-agents, each responsible for a distinct business domain. The decision to use this architecture was driven by the principle of **separation of concerns** — each agent has a focused role with its own set of tools, which keeps prompts targeted and reduces the chance of a single monolithic agent making errors across unrelated domains.

### Agent Roles

1. **Orchestrator (`Orchestrator`)**: The top-level coordinator. It receives a customer request via `process_order()` and breaks it into a three-step workflow:
   - **Step 1 — Inventory Check**: Delegates to the Inventory Manager to verify current stock levels and trigger restocking if any item has fallen below its minimum threshold.
   - **Step 2 — Order Finalization**: Delegates to the Sales Closure Agent to verify item availability, estimate delivery timelines from the supplier, and record the sales transaction.
   - **Step 3 — Quotation Generation**: Delegates to the Quotation Manager to search historical quotes for similar orders and produce a customer-facing quotation document.

   The Orchestrator exposes four internal tools (`get_inventory_status`, `get_qutotation`, `finalize_order`, `generate_financial_report`) that wrap calls to the sub-agents. This design lets the LLM powering the Orchestrator decide the sequence and whether to invoke all steps or skip some based on context.

2. **Inventory Manager (`InventoryManagerAgent`)**: Responsible for monitoring stock levels. It uses two tools:
   - `get_all_inventory` — retrieves a snapshot of all item stock levels as of a given date by computing net quantities from the transactions ledger (stock orders minus sales).
   - `create_transaction` — records a `stock_orders` transaction to replenish inventory when an item's stock drops below its configured `min_stock_level`.

   This agent acts as a **proactive restocking guard**: before any customer order is processed, it ensures the warehouse is adequately stocked, preventing stockouts from cascading into failed deliveries.

3. **Quotation Manager (`QuotationManagerAgent`)**: Generates customer-facing price quotes. It uses one tool:
   - `search_quote_history` — performs a keyword-based search across the `quote_requests` and `quotes` database tables to find historically similar orders.

   The agent uses retrieved historical quotes (including `total_amount`, `quote_explanation`, `job_type`, `order_size`, and `event_type`) as reference templates. If no sufficiently similar historical quote exists, it constructs a new one following the same format. This ensures consistency in customer communications.

4. **Sales Closure Agent (`SalesClosureAgent`)**: Handles order finalization and delivery logistics. It uses three tools:
   - `get_stock_level` — checks the current stock of a specific item as of a date.
   - `get_supplier_delivery_date` — estimates delivery lead time based on order quantity (same-day for ≤10 units, up to 7 days for >1000 units).
   - `create_transaction` — records a `sales` transaction to debit inventory and credit revenue.

   This agent is responsible for the critical decision of whether a timely delivery is feasible. It compares available stock against the requested quantities and uses the supplier lead-time model to determine if the order can arrive by the customer's requested date.

### Decision-Making Rationale

- **Hierarchical delegation** was chosen over a flat multi-agent setup because the business workflow is inherently sequential (check stock → fulfill order → quote). The Orchestrator naturally models this pipeline.
- **Tool-based inter-agent communication**: Rather than having agents message each other directly, the Orchestrator wraps each sub-agent's `run()` method inside a `@tool`-decorated function. This keeps the `smolagents` framework's tool-calling mechanism as the single communication protocol, simplifying debugging and tracing.
- **Shared database as state**: All agents operate on the same SQLite database (`munder_difflin.db`) via the `db_engine`. The transactions table serves as a ledger-style single source of truth — stock levels are always computed dynamically from the transaction history rather than maintained as mutable state, which prevents inconsistencies.
- **Fuzzy item matching**: Both the Inventory Manager and Sales Closure Agent are instructed to find the closest matching item if the customer's request doesn't exactly match an inventory item name, adding robustness to natural-language order processing.

## Evaluation Results

The system was evaluated using the `run_test_scenarios()` function, which processed 20 customer requests from `quote_requests_sample.csv` sequentially between April 1 and April 17, 2025. After each order, the system's cash balance and inventory value were recorded and saved to `test_results.csv`. The analysis below is grounded in the actual outputs captured in that file.

### Strengths Identified

1. **100% request completion rate**: All 20 customer requests received a complete response — the system never crashed, timed out, or returned an empty result. Even complex multi-item requests (e.g., Request 7 with four distinct items, Request 17 with five items including biodegradable products) were parsed and processed end-to-end through the three-agent pipeline without manual intervention.

2. **Structured, itemized quotations**: The majority of responses contain professionally formatted, itemized quotations with per-item quantities, unit prices, line totals, and a grand total. For example, Request 5 breaks down 500 sheets of colored paper ($50.00), 300 sheets of cardstock ($60.00), and 200 rolls of washi tape ($500.00) into a clear $610.00 total. Request 14 similarly itemizes A4 paper ($250), poster paper ($1,500), and cardstock ($75) into a $1,825 total with per-item delivery dates. This level of detail is consistent with real-world customer-facing quotations.

3. **Proactive inventory replenishment that sustained operations**: The inventory value grew from $6,444.10 after Request 1 to $13,890.35 by Request 20, demonstrating that the Inventory Manager consistently detected low-stock items and placed restocking orders. This is visible in the data: between Requests 9 and 13, inventory rose from $6,988.60 to $7,631.10, and again between Requests 14 and 15 it jumped from $9,428.85 to $11,044.85. Without this automatic restocking, later orders would have failed due to depleted stock.

4. **Correct handling of unfulfillable orders**: Request 20 (5,000 flyers, 2,000 posters, and 10,000 tickets for a concert) was correctly identified as unfulfillable — the system responded that "all requested items are out of stock" and that "delivery cannot be ensured by the requested date," while still providing a quotation total of $2,100. This shows the Sales Closure Agent properly evaluated stock levels and delivery feasibility rather than blindly accepting every order.

5. **Fuzzy item-name resolution**: Customers used varied item descriptions (e.g., "heavy cardstock (white)" in Request 1, "colorful poster paper" in Request 2, "8.5"x11" colored paper" in Request 5), which do not exactly match canonical inventory names like "Cardstock," "Poster paper," or "Colored paper." The system successfully mapped these to the closest inventory items in most cases, demonstrating the LLM's ability to bridge the gap between natural-language customer requests and the structured product catalog.

### Weaknesses Observed in the Data

1. **Cash balance went deeply negative**: The cash balance started at approximately $43,416 after Request 1 and declined steadily, crossing into negative territory by Request 13 (−$5,974) and reaching −$167,919 by Request 20. This indicates the system is spending heavily on stock replenishment orders without a guardrail that checks whether the company can afford the purchase. The Inventory Manager's restocking logic triggers based solely on whether stock is below `min_stock_level`, with no check against available cash. In a real business, this would represent unsustainable spending.

2. **Incorrect delivery dates in many responses**: A significant number of responses cite delivery dates in 2023 (e.g., "2023-10-02" in Request 1, "2023-10-05" in Requests 6 and 10, "2023-10-14" in Request 7) despite all requests being dated April 2025. This is a hallucination by the LLM — the `get_supplier_delivery_date` tool itself computes correct dates relative to the input, but the LLM sometimes fabricated dates in its final response text rather than using the tool's actual output. This occurred in at least 10 of the 20 responses and would be a critical issue in production.

3. **Placeholder values in Response 1**: The very first response contains "$XX.XX" placeholders instead of actual prices, suggesting the Quotation Manager failed to compute or retrieve pricing for that request. While subsequent requests did not repeat this issue, it indicates the system can occasionally produce incomplete outputs, particularly for the first request when the agent may not yet have established reliable tool-calling patterns.

4. **Inconsistent pricing across similar items**: The same or comparable items are priced differently across requests. For instance, A4 paper is quoted at $0.05/sheet in Request 14 (matching the catalog price) but at "$4.00 per 100 sheets" ($0.04/sheet) in Request 16. Cardstock appears at $0.50/sheet in Request 12 but $0.15/sheet in Request 14 and $0.20/sheet in Request 5. These inconsistencies arise because the Quotation Manager sometimes uses historical quote data and other times invents prices, without a single authoritative price source enforced in the prompt.

5. **Some orders not recorded as transactions**: Request 4 shows no change in cash balance ($16,365.74 before and after), suggesting the sales transaction was never actually created in the database even though the response claims the order was "successfully placed." This means the quotation was generated and returned to the customer, but the underlying ledger was not updated — a data integrity gap between what the customer sees and what the system records.

## Suggestions for Improvements

### 1. Add a Deterministic Item-Matching Layer

Currently, the system depends entirely on the LLM to map customer-requested item names (e.g., "glossy sheets") to canonical inventory names (e.g., "Glossy paper"). This is non-deterministic and could fail silently. A significant improvement would be to introduce a **deterministic fuzzy-matching preprocessing step** — for example, using `thefuzz` (formerly `fuzzywuzzy`) or a TF-IDF similarity search — that maps each requested item to the closest match in `PAPER_SUPPLIES` before passing it to the agents. This would serve as a reliable first pass, with the LLM handling only genuinely ambiguous cases. This change would reduce LLM token usage, increase reliability, and make the system's item-resolution behavior testable and auditable.

### 2. Implement a Feedback Loop with Order Validation Agent

The current architecture is a one-pass pipeline: inventory check → order → quote. There is no validation step that reviews the final output for correctness. Adding a **Validation Agent** (or a validation tool on the Orchestrator) that cross-checks the recorded transaction against the original customer request would catch errors such as:
- Wrong item names being ordered
- Quantity mismatches between the request and the recorded transaction
- Quotation amounts that don't align with the actual transaction price

This agent could run after `finalize_order` and before `get_quotation`, acting as a quality gate. If discrepancies are detected, the Orchestrator could retry the order step — introducing a self-correcting feedback loop that significantly improves reliability.

### 3. Support Concurrent Multi-Item Orders with Partial Fulfillment

Currently, orders with multiple items are processed as a single block. If one item is out of stock or has long lead times, the entire order's handling becomes ambiguous. The system should support **partial fulfillment** — explicitly splitting an order into items that can be delivered immediately versus those that require backorder. This would involve:
- The Sales Closure Agent returning a structured result per item (available/backordered/unavailable)
- The Quotation Manager generating a quote that itemizes delivery dates per product
- The Orchestrator presenting the customer with options (e.g., "Ship available items now, backorder the rest" vs. "Wait for full order")

This would make the system significantly more practical for real-world use where mixed-availability orders are common.