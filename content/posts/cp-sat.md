---
title: "✨ Recipes for Google OR-Tools"
date: 2020-12-14
draft: false
weight: 1

aliases:
  - /cp-sat

tags: ["Python", "Operations research", "ortools", "CP-SAT"]
cover:
  image: "https://pbs.twimg.com/media/DuoN35ZXgAAKzC_.jpg"
---

Some tips and tricks for the CP-SAT solver.

<!--more-->

## Resources:

- https://github.com/or-tools/awesome_or-tools
- https://github.com/d-krupke/cpsat-primer
- https://stackoverflow.com/questions/tagged/or-tools
- https://or.stackexchange.com/questions/tagged/or-tools
- https://groups.google.com/g/or-tools-discuss
- https://github.com/google/or-tools/discussions
- https://discord.gg/ENkQrdf

## Solver Parameters

By default, OR-tools will try to use all available cores but you can set `num_search_workers` manually.

```python
solver.parameters.num_search_workers = 8
```

If the program silently crashes you can enable logging to get more info.

```python
solver.parameters.log_search_progress = True
```

You can also see which worker is better for your problem and improve the single core performance by setting the same parameters.

## Search for all optimal solutions

You can't enumerate all solutions if you have an objective, so you have to do it in two steps:

```python
# Get the optimal objective value
model.Maximize(objective)
solver.Solve(model)

# Set the objective to a fixed value
# use round() instead of int()
model.Add(objective == round(solver.ObjectiveValue()))
model.Proto().ClearField('objective')

# Search for all solutions
solver.parameters.enumerate_all_solutions = True
solver.Solve(model, cp_model.VarArraySolutionPrinter([x, y, z]))
```

## Non-contiguous Intvar

Alternatives to `model.NewIntVar(lb, ub, name)`, solving might be faster:

```python
# List of values
model.NewIntVarFromDomain(
    cp_model.Domain.FromValues([1, 3, 4, 6]), 'x'
)

# List of intervals
model.NewIntVarFromDomain(
    cp_model.Domain.FromIntervals([[1, 2], [4, 6]]), 'x'
)

# Exclude [-1, 1]
model.NewIntVarFromDomain(
    cp_model.Domain.FromIntervals([[cp_model.INT_MIN, -2], [2, cp_model.INT_MAX]]), 'x'
)

# Constant
model.NewConstant(154)
```

## Domains

Working with Domains gives a lot of flexibility and should be faster than creating multiple constraints.

```python
domain = cp_model.Domain(0, 10)
domain = cp_model.Domain.FromValues([1, 3, 4, 6])
domain = cp_model.Domain.FromIntervals([[1, 2], [4, 6]])
domain = cp_model.Domain.FromIntervals([
    [cp_model.INT_MIN, -2],
    [2, cp_model.INT_MAX]
])

model.AddLinearExpressionInDomain(sum(my_vars), domain).OnlyEnforceIf(b)
```

## Iff, equivalence, boolean product

`p <=> (x and y)`

```python
# (x and y) => p, rewrite as not(x and y) or p
model.AddBoolOr([x.Not(), y.Not(), p])

# p => (x and y)
model.AddImplication(p, x)
model.AddImplication(p, y)
```

## Boolean sum, boolean or

`p <=> (x or y)`

```python
# p => (x or y), rewrite as not(p) or x or y
model.AddBoolOr([x, y, p.Not()])

# (x or y) => p
model.AddImplication(x, p)
model.AddImplication(y, p)
```

**Note**: see [De Morgan's laws](https://en.wikipedia.org/wiki/De_Morgan%27s_laws)

## Boolean Implications

```python
# a => b (both booleans)
model.AddImplication(a, b)

# a <=> b (use only one of them if you can)
model.Add(a == b)

# a and b and c => d
model.Add(d == 1).OnlyEnforceIf([a, b, c])
or
model.AddBoolOr([a.Not(), b.Not(), c.Not(), d])
```

## If-Then-Else

Using intermediate boolean variables.

```python
b = model.NewBoolVar('b')

# Implement b == (x >= 5).
model.Add(x >= 5).OnlyEnforceIf(b)
model.Add(x < 5).OnlyEnforceIf(b.Not())
```

## Solution hint / Warm start

It may speed up the search.

```python
num_vals = 3
x = model.NewIntVar(0, num_vals - 1, 'x')
y = model.NewIntVar(0, num_vals - 1, 'y')
z = model.NewIntVar(0, num_vals - 1, 'z')

model.Add(x != y)

model.Maximize(x + 2 * y + 3 * z)

# Solution hinting: x <- 1, y <- 2
model.AddHint(x, 1)
model.AddHint(y, 2)
```

## Storing Multi-index variables

I recommend using dictionary comprehensions:

```python
employee_works_day = {
    (e, day): model.NewBoolVar(f"{e} works {day}")
    for e in employees
    for day in days
}
```

## Variable product (Non linear)

You have to create an intermediate variable:

```python
x = model.NewIntvar(0, 8, 'x')
y = model.NewIntvar(0, 5, 'y')
xy = model.NewIntVar(0, 8*5, 'xy')

model.AddMultiplicationEquality(xy, [x, y])
```

## Black box function (Non linear)

As your last resort you can precalculate all the posible values and use an element constraint.

```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()
x = model.NewIntVar(0, 10, "")
two_to_the_x = model.NewIntVar(1, 2 ** 10, "")

precalculated = [2 ** i for i in range(11)]
model.AddElement(x, precalculated, two_to_the_x)

# test
model.Add(x == 3)

solver = cp_model.CpSolver()
solver.Solve(model)

print("x", solver.Value(x))
print("2**x", solver.Value(two_to_the_x))
```

## Load model Proto

This allows you to use different machines.

```python
from google.protobuf import text_format
from ortools.sat.python import cp_model

model = cp_model.CpModel()
a = model.NewIntVar(0, 10, "a")
b = model.NewIntVar(0, 10, "b")
model.Maximize(a + b)

new_model = cp_model.CpModel()
text_format.Parse(str(model), new_model.Proto())

solver = cp_model.CpSolver()
status = solver.Solve(new_model)

print(solver.StatusName(status))
print(solver.ObjectiveValue())
```

## Circuit constraint (ordering)

Ordering the numbers from 1 to 10 so that we maximize the distance between between numbers:

```python
from itertools import permutations
from ortools.sat.python import cp_model

model = cp_model.CpModel()
solver = cp_model.CpSolver()

literals = {}
# An arc is just a (int, int, BoolVar) tuple
all_arcs = []
nodes = range(1, 11)

for i in nodes:
    # We use 0 as a dummy nodes as we don't have an actual circuit
    literals[0, i] = model.NewBoolVar(f"0 -> {i}")  # start arc
    literals[i, 0] = model.NewBoolVar(f"{i} -> 0")  # end arc
    all_arcs.append([0, i, literals[0, i]])
    all_arcs.append([i, 0, literals[i, 0]])

for i, j in permutations(nodes, 2):
    # this booleans will be true if the arc is present
    literals[i, j] = model.NewBoolVar(f"{i} -> {j}")
    all_arcs.append([i, j, literals[i, j]])
# to make an arc optional, add the [i, i, True] loop

model.AddCircuit(all_arcs)
model.Maximize(sum(literals[i, j] * abs(i - j) for i, j in permutations(nodes, 2)))
solver.Solve(model)

node = 0
print(node, end="")
while True:
    for i in nodes:
        if i != node and solver.Value(literals[node, i]):
            print(f" -> {i}", end="")
            node = i
            break
    else:
        break
print(" -> 0")
```

## Fairness, distribute items evenly

- Maximize the minimum value

```python
model = cp_model.CpModel()
n_tasks = 100
n_employees = 3

min_tasks = model.NewIntVar(0, n_tasks, "")
employee_tasks = [model.NewIntVar(0, n_tasks, "") for _ in range(n_employees)]
model.Add(sum(employee_tasks) == n_tasks)

model.AddMinEquality(min_tasks, employee_tasks)

model.Maximize(min_tasks)
solver = cp_model.CpSolver()
status = solver.Solve(model)

print([solver.Value(n) for n in employee_tasks])
```

- Minimize delta to the average value

```python
model = cp_model.CpModel()
n_tasks = 100
n_employees = 3
avg = n_tasks // n_employees

delta = model.NewIntVar(0, n_tasks, "")
employee_tasks = [model.NewIntVar(0, n_tasks, "") for _ in range(n_employees)]
model.Add(sum(employee_tasks) == n_tasks)

for i in range(n_employees):
    model.Add(employee_tasks[i] <= avg + delta)
    model.Add(employee_tasks[i] >= avg - delta)

model.Minimize(delta)
solver = cp_model.CpSolver()
status = solver.Solve(model)

print([solver.Value(n) for n in employee_tasks])
```

## Multiobjective optimization

Two ways to achieve that:

- Add a weight to each objective
- Solve with the first objective, constraint the objective with the solution, hint and solve with the new objective.

```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()
solver = cp_model.CpSolver()
x = model.NewIntVar(0, 10, "x")
y = model.NewIntVar(0, 10, "y")

# Maximize x
model.Maximize(x)
solver.Solve(model)
print("x", solver.Value(x))
print("y", solver.Value(y))
print()

# Hint (speed up solving)
model.AddHint(x, solver.Value(x))
model.AddHint(y, solver.Value(y))

# Maximize y (and constraint prev objective)
model.Add(x == round(solver.ObjectiveValue()))  # use <= or >= if not optimal
model.Maximize(y)

solver.Solve(model)
print("x", solver.Value(x))
print("y", solver.Value(y))
```

## Soft constraints

https://stackoverflow.com/a/66377562

## Early stopping

Stop if objective does not improve.

```python
from threading import Timer

from ortools.sat.python import cp_model


class ObjectiveEarlyStopping(cp_model.CpSolverSolutionCallback):
    def __init__(self, timer_limit: int):
        super(ObjectiveEarlyStopping, self).__init__()
        self._timer_limit = timer_limit
        self._timer = None
        self._reset_timer() # Remove to guarantee a solution

    def on_solution_callback(self):
        self._reset_timer()

    def _reset_timer(self):
        if self._timer:
            self._timer.cancel()
        self._timer = Timer(self._timer_limit, self.StopSearch)
        self._timer.start()

    def StopSearch(self):
        print(f"{self._timer_limit} seconds without improvement")
        super().StopSearch()
```
