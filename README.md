import json
from turtle import clear
from ollama import Client

client = Client()
import json
import re


# -----------------------------
# Load Data
# -----------------------------

with open("ApprovalRule.json") as f:
    data = json.load(f)

with open("Quote.json") as f:
    quote = json.load(f)

with open("QuoteLine.json") as f:
    quote_lines = json.load(f)


# Fix export structure
if isinstance(quote, list):
    quote = quote[0]

if not isinstance(quote_lines, list):
    quote_lines = [quote_lines]


rules = data["rules"]


# -----------------------------
# Helpers
# -----------------------------
def normalize(v):

    if v is None:
        return None

    if isinstance(v, bool):
        return v

    v = str(v).strip().lower()

    if v in ["true", "1"]:
        return True

    if v in ["false", "0"]:
        return False

    return v



def evaluate_operator(actual, operator, expected):

    actual = normalize(actual)
    expected = normalize(expected)

    if operator == "equals":
        return actual == expected

    if operator == "not equals":
        return actual != expected

    if operator == "contains":
        if actual is None:
            return False
        return str(expected).lower() in str(actual).lower()

    if operator == "does not contain":
        if actual is None:
            return True
        return str(expected).lower() not in str(actual).lower()

    return False



def safe_float(v):
    try:
        return float(v)
    except:
        return 0


# -----------------------------
# Operator evaluation
# -----------------------------

def normalize_bool(v):
    if isinstance(v, bool):
        return v
    if v is None:
        return None
    v = str(v).lower().strip()
    if v in ["true", "1"]:
        return True
    if v in ["false", "0"]:
        return False
    return v


def evaluate_operator(actual, operator, expected):

    actual = normalize_bool(actual)
    expected = normalize_bool(expected)

    if operator == "equals":
        return actual == expected

    if operator == "not equals":
        return actual != expected

    if operator == "contains":
        if actual is None:
            return False
        return str(expected).lower() in str(actual).lower()

    if operator == "does not contain":
        if actual is None:
            return True
        return str(expected).lower() not in str(actual).lower()

    try:
        actual_f = float(actual)
        expected_f = float(expected)
    except:
        return False

    if operator == "greater than":
        return actual_f > expected_f

    if operator == "less than":
        return actual_f < expected_f

    if operator == "greater or equals":
        return actual_f >= expected_f

    if operator == "less or equals":
        return actual_f <= expected_f

    return False
# -----------------------------
# Filter evaluation
# -----------------------------
def evaluate_filter(line, cond):

    field = cond.get("filter_field")
    operator = cond.get("filter_operator")
    value = cond.get("filter_value")

    if not field:
        return True

    actual = line.get(field)
    print(actual)

    if actual is None:
        return False

    actual = str(actual).lower().strip()
    value = str(value).lower().strip()
    print('oper,', operator)

    if operator == "contains":
        
        return value in actual

    if operator == "equals":
        return actual == value

    if operator == "not equals":
        return actual != value
   

    return False
# -----------------------------
# Approval variable evaluator
# -----------------------------
def evaluate_variable(cond):

    filtered_lines = []

    for line in quote_lines:

        if evaluate_filter(line, cond):
            filtered_lines.append(line)

    print("Filtered lines:", len(filtered_lines))

    values = [
        safe_float(l.get(cond["target_field"]))
        for l in filtered_lines
    ]

    agg = cond.get("aggregation", "").lower()

    if agg == "sum":
        result = sum(values)

    elif agg == "count":
        result = len(filtered_lines)

    elif agg == "max":
        result = max(values) if values else 0

    elif agg == "min":
        result = min(values) if values else 0

    else:
        result = 0

    print("Aggregation Result:", result)

    return evaluate_operator(result, cond["operator"], cond["value"])
# -----------------------------
# Field condition evaluator
# -----------------------------

def evaluate_field(cond):

    field = cond.get("field")
    operator = cond.get("operator")
    value = cond.get("value")

    actual = quote.get(field)
    print("Eval:", actual, operator, value, "->", evaluate_operator(actual, operator, value))

    return evaluate_operator(actual, operator, value)


# -----------------------------
# Condition evaluator
# -----------------------------

def evaluate_condition(cond):
    

    if cond["type"] == "field_condition":
        return evaluate_field(cond)

    if cond["type"] in ["approval_variable", "summary_variable"]:
        return evaluate_variable(cond)

    return False


# -----------------------------
# Advanced condition evaluator
# -----------------------------

import re

import re

def evaluate_advanced(condition_results, expression):

    if not expression:
        return all(condition_results.values())

    expr = expression
    print("Original Expression:", expr)

    # normalize operators
    expr = expr.replace("AND", "and").replace("OR", "or")

    # replace condition numbers safely
    for num, result in condition_results.items():

        expr = re.sub(
            r'\b{}\b'.format(num),
            str(result),
            expr
        )

        print(mappigDic)

    print("Evaluating:", expr)

    return eval(expr)


# -----------------------------
# Rule evaluation
# -----------------------------
triggered_rules = []
not_triggered_rules = []
ruleId =''

for rule in rules:
    ruleId = rule.get("rule_id")

    if len(rule["conditions"]) == 0:
        not_triggered_rules.append(rule)
        continue

    print("Rule:", rule["rule_name"])

    # -----------------------------
    # Remove duplicate conditions
    # -----------------------------
    unique_conditions = []
    seen = set()

    for cond in rule["conditions"]:

        key = (
            cond.get("field"),
            cond.get("operator"),
            cond.get("value"),
            cond.get("variable_name")
        )

        if key not in seen:
            seen.add(key)
            unique_conditions.append(cond)

    # -----------------------------
    # Evaluate conditions
    # -----------------------------
    condition_results = {}
    mappigDic = {}
    count = 0

    for cond in unique_conditions:
        count += 1

        print("Field:", cond.get("field"))
        print("Actual:", quote.get(cond.get("field")))
        print("Expected:", cond.get("value"))
        condition_number = ruleId + str(count) if rule["conditions_met"] == 'All' else cond.get("condition_number")
        num = cond.get("condition_number")

        if num is None:
            num = f"cond_{len(condition_results)+1}"

        result = evaluate_condition(cond)

    
        condition_results[condition_number] = result
        print("Condition Results:", condition_results)

    # -----------------------------
    # Evaluate rule logic
    # -----------------------------
    if rule["conditions_met"] == "All":
        print('f')

        rule_triggered = all(condition_results.values())

    elif rule["conditions_met"] == "Any":

        rule_triggered = any(condition_results.values())

    elif rule["conditions_met"] == "Custom":

        rule_triggered = evaluate_advanced(
            condition_results,
            rule["advanced_condition"]
        )
        print('r',rule_triggered)

    else:
        rule_triggered = all(condition_results.values())

    # -----------------------------
    # Store results
    # -----------------------------
    if rule_triggered:
        triggered_rules.append(rule)
    else:
        not_triggered_rules.append(rule)

# -----------------------------
# Output
# -----------------------------

print("\nQuote ID:", quote.get("Id"))
print("Quote Lines:", len(quote_lines))

print("\nTriggered Rules:", len(triggered_rules))

for r in triggered_rules:
    print("-", r["rule_name"])

print("\nNot Triggered Rules:", len(not_triggered_rules))
