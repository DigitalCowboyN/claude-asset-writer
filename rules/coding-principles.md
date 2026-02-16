# Coding Principles

Apply universal coding principles to every solution, regardless of language or project type.

## When This Applies

Every session, every language, every codebase.

## Requirements

- [ ] **KISS** — Favor the simplest solution that meets requirements; avoid unnecessary complexity
- [ ] **DRY** — Avoid code duplication; extract reusable logic into functions, classes, or modules
- [ ] **YAGNI** — Don't implement features unless currently required; add complexity only when justified
- [ ] **SOLID** — Follow Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
- [ ] **Single Responsibility** — Keep functions and classes small, focused, with one well-defined purpose
- [ ] **Modularity** — Design systems as independent, reusable, testable components
- [ ] **Meaningful Returns** — Functions return predictable values with appropriate error indicators
- [ ] **Efficiency** — Avoid unnecessary computations, loops, or queries; write clear AND efficient code

## Examples

### Good
```
# Simple, focused function with single responsibility
def calculate_total(items):
    return sum(item.price for item in items)

# Reusable module with clear interface
class PaymentProcessor:
    def process(self, amount): ...
```

### Bad
```
# Overly complex, does too much, duplicated logic
def process_order_and_send_email_and_update_inventory(...):
    # 200 lines of mixed concerns
    ...
```
