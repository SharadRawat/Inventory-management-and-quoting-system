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

The system was evaluated using the `run_test_scenarios()` function, which processed 6 customer requests from `quote_requests_sample.csv`. After each order, the system's cash balance and inventory value were recorded and saved to `test_results.csv`. The following summarizes the observed strengths and weaknesses based on the new results:

### Strengths

1. **Consistent and clear communication:** Every customer request received a structured, readable response. The system reliably informed customers when items were out of stock and provided quotations even when orders could not be fulfilled.
2. **Itemized, professional quotations:** Quotations were broken down by item, with quantities and prices clearly listed (e.g., Request 4). This matches real-world business standards.
3. **Partial fulfillment transparency:** When only some items could be fulfilled (e.g., Request 6), the system clearly indicated which items were available and which were not.
4. **No negative cash or inventory:** The cash balance and inventory value remained stable, showing that the system did not process invalid transactions or allow negative balances.

### Weaknesses

1. **No successful full order fulfillment:** All requests resulted in either full stock-outs or only partial fulfillment. There were no cases of a complete customer order being processed and delivered.
2. **No financial movement:** Cash balance and inventory value did not change across requests, indicating that no actual sales or stock orders were processed, even when partial fulfillment was claimed.
3. **Delivery date inconsistencies:** Some responses (e.g., Requests 5 and 6) included delivery dates in 2023, which do not match the 2025 scenario, suggesting a logic or prompt grounding issue.
4. **Non-catalog pricing and vague explanations:** Some quotations referenced "market prices" or "special rates" without clear catalog justification, which could confuse customers.
5. **Occasional internal or unnecessary details:** Some responses included transaction IDs or terms and conditions that may not be needed in customer-facing communication.

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