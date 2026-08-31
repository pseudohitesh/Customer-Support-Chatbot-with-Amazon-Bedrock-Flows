# AI Support Chatbot with AWS Bedrock

A customer support chatbot built on Amazon Bedrock that classifies incoming customer messages and routes them to the correct handling path: **Bug Report**, **Platform Question**, or **Other Request**. Includes an automated evaluation harness and results from AWS Bedrock's LLM-as-a-judge model evaluation.

## Overview

The chatbot acts as a front-line support agent for an online shop. On every customer message, it decides which of three categories the message belongs to and follows a fixed set of rules for that category:

1. **Bug Report** – Something on the website/app is broken. The bot collects `description`, `stepsToReproduce`, and `environment` one missing field at a time, then calls a tool to create a ticket.
2. **Platform Question** – A question answerable from the shop's FAQ (orders, shipping, returns, refunds, payments, products, accounts, privacy). Answered strictly from the FAQ, never from general knowledge.
3. **Other Request** – Anything outside the above. The bot declines and refers the customer to the human support line (1-800-555-0199, Mon–Fri).

The full routing logic and constraints live in `system_prompt.txt`.

## Architecture

Built as an Amazon Bedrock **Flow**:

- **Classifier prompt node** – reads the user message and outputs a category (`BUG_REPORT`, `PLATFORM_QUESTION`, or a fallback).
- **Condition node** – branches to one of three downstream prompt nodes based on the classifier's output.
- **Bug_report / Platform_request / Other_request prompt nodes** – each handles its category according to the system prompt rules.
- **Lambda function node** – invoked from the bug report path to persist tickets.
- **Flow output nodes** – return the final response for each branch.

Bug report tickets are stored in a **DynamoDB** table (`bug-report-tool-stack-bug-reports`) with fields `ticketId`, `createdAt`, `description`, `stepsToReproduce`, `environment`, and `status` (defaults to `OPEN`).

## Project Structure

```
.
├── chat.py                      # CLI entry point for chatting with the bot
├── setup_gateway.py             # AgentCore gateway setup
├── cleanup_agentcore.py         # Teardown/cleanup script for AgentCore resources
├── create_harness.py            # Runs the evaluation harness against test cases
├── create_bug_report.py         # Tool used by the flow to create bug report tickets
├── generate-eval-dataset.py     # Builds the evaluation dataset (output_eval_dataset.jsonl)
├── agentcore_config.json        # AgentCore configuration
├── cloudformation-tool.yaml     # CloudFormation template for the bug-report tool stack
├── cloudformation-testing.yaml  # CloudFormation template for testing infrastructure
├── harness-tests.json           # Ground-truth test cases used for evaluation
├── harness-tests-template.json  # Template for adding new harness tests
├── system_prompt.txt            # Full routing logic and rules for the chatbot
├── online_shop_faq.md           # FAQ knowledge source for Platform Questions
├── output_eval_dataset.jsonl    # Generated evaluation dataset
├── eval-results-backup/         # Backup of Bedrock model evaluation run results
└── requirements.txt             # Python dependencies
```

## Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/project1.git
   cd project1
   ```

2. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   venv\Scripts\activate   # Windows
   source venv/bin/activate  # macOS/Linux
   ```

3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

4. Configure AWS credentials with access to Amazon Bedrock, DynamoDB, and Lambda.

5. Deploy the bug-report tool stack:
   ```bash
   aws cloudformation deploy --template-file cloudformation-tool.yaml --stack-name bug-report-tool-stack
   ```

6. Update `agentcore_config.json` with your Bedrock Flow / AgentCore identifiers.

## Usage

**Chat with the bot:**
```bash
python chat.py
```

**Generate an evaluation dataset:**
```bash
python generate-eval-dataset.py
```

**Run the evaluation harness against `harness-tests.json`:**
```bash
python create_harness.py
```

**Clean up AgentCore/Flow resources:**
```bash
python cleanup_agentcore.py
```

**Check created bug report tickets directly in DynamoDB:**
```bash
aws dynamodb scan --table-name bug-report-tool-stack-bug-reports --region us-east-1 \
  --query 'Items[].[ticketId.S,status.S]' --output table
```

## Evaluation

The bot was evaluated using **Amazon Bedrock Model Evaluation (LLM-as-a-judge)** against 10 test cases covering all three routing categories, plus multi-turn scenarios where required bug-report fields arrive across several messages.

- **Overall correctness: 0.85** (average across 10 prompts)
- 8 of 10 test cases scored a perfect 1.0
- Two lower-scoring cases involved sequencing: one where the bot created a bug report before confirming to the user, and one where it asked a follow-up question rather than fully completing the ticket in one turn

Test cases live in `harness-tests.json`, each with a `prompt` and an `expected` (ground truth) description of the correct routing decision and behavior. Example categories tested:

- Bug reports with all required fields present vs. partially missing
- Platform questions answerable directly from the FAQ
- Requests outside the FAQ's scope, correctly deferred to human support

## Notes

- This project was built as part of an AWS Agent Engineer course.
- Ensure sensitive configuration (AWS account IDs, credentials) is not committed to version control — see `.gitignore`.
