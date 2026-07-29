---
name: Library/yggi/yg/test
tags: meta/library
files: []
description: unified test registry — write once in code blocks, query inline with show()
---
---
---

# yg.test

Write-once test assertions that register into a global registry. Display inline with `${yg.test.show(...)}`. Query headlessly via the Runtime API.

## Namespace


```#yg/object
tags: [luaref]
displayName: "yg.test"
luaref:
  name: yg.test
  type: table
  description: "Test registry: run assertions in code blocks, display results inline."
```

---

## assert(key, actual, expected?, description?)

yg.test.assert(key, actual, expected?, description?): entry | widget
Register one test result in `yg.test._test_registry[key]`. Returns a widget when called from `${...}`, the result entry table from code blocks.

- `actual` as **string** → evaluated as Lua expression via `spacelua.parseExpression`/`evalExpression`, pcall-wrapped; source string stored for `show('full')`
- `actual` as **function** → pcall-called; no source stored
- `actual` as **value** → used as-is
- `expected` omitted → truthiness check
- Side-effect: emits `js.log("[PASS]"/"[FAIL]")` for headless runner

```#yg/object
tags: [luaref]
displayName: "assert(key, actual, expected?, description?)"
luaref:
  name: assert
  scope: yg.test
  type: function
  description: "Register a test result. String actual is evaluated as Lua. Returns widget in ${} context."
  example: 'yg.test.assert("yg.foo() : adds", "1+1", 2)'
  arguments:
    - name: key
      type: string
      description: "full registry key, e.g. 'yg.foo() : label'"
    - name: actual
      type: string | function | any
      description: "string → eval as Lua expr; function → pcall; other → use as-is"
    - name: expected
      type: any
      description: "optional — omit for truthiness check; compared via tostring()"
    - name: description
      type: string
      description: "optional prose shown in show('full') card"
```

---

## run(prefix, fn)

yg.test.run(prefix, fn): nil
Suite runner. Wraps each `t.assert()` call with pcall protection so one failure does not abort others. Prepends `prefix .. " : "` to each label. Calls `codeWidget.refreshAll()` at the end.

```#yg/object
tags: [luaref]
displayName: "run(prefix, fn)"
luaref:
  name: run
  scope: yg.test
  type: function
  description: "Suite runner: pcall-protected assertions with prefix prepended to each label."
  example: 'yg.test.run("yg.foo()", function(t) t.assert("adds", 1+1, 2) end)'
  arguments:
    - name: prefix
      type: string
      description: "suite key prefix, e.g. 'yg.oo.tools.wrapObject()'"
    - name: fn
      type: function
      description: "receives t with t.assert(label, actual, expected?, description?)"
```

---

## show(?prefix, ?style)

yg.test.show(?prefix, ?style): widget
Query `yg.test._test_registry` for keys starting with `prefix` (all if nil). Returns a rendered widget. Only call from `${...}` expressions.

- `style = 'mini'` (default) — compact table: label + ✅/❌/⚠️
- `style = 'full'` — one card per test with description, source expression, result detail

```#yg/object
tags: [luaref]
displayName: "show(?prefix, ?style)"
luaref:
  name: show
  scope: yg.test
  type: function
  description: "Render test results from the registry as a widget. Call from ${} only."
  example: '${yg.test.show("yg.oo.tools.wrapObject()")}'
  arguments:
    - name: prefix
      type: string
      description: "optional key prefix filter; nil shows all results"
    - name: style
      type: string
      description: "'mini' (default) compact table, 'full' detailed cards"
```

---

## Implementation

```space-lua
-- priority: 97
-- Single block: all definitions and self-tests in document order.
yg = yg or {}
yg.test = yg.test or {}
yg.test._test_registry = yg.test._test_registry or {}
-- Registered suites, keyed by prefix (re-registration overwrites — no dupes).
-- Enables eval-phase re-run via runAll(): see execution-phases spec.
yg.test._suites = yg.test._suites or {}

-- assert(key, actual, expected?, description?)
yg.test.assert = function(key, actual, expected, description)
  local actual_val, source_str, err_msg

  if type(actual) == "string" then
    source_str = actual
    local ok, result = pcall(function()
      return spacelua.evalExpression(spacelua.parseExpression(actual))
    end)
    if ok then actual_val = result
    else actual_val = nil; err_msg = result end
  elseif type(actual) == "function" then
    local ok, result = pcall(actual)
    if ok then actual_val = result
    else actual_val = nil; err_msg = result end
  else
    actual_val = actual
  end

  local status
  if err_msg then
    status = "error"
  elseif expected == nil then
    status = actual_val and "pass" or "fail"
  else
    status = (tostring(actual_val) == tostring(expected)) and "pass" or "fail"
  end

  local entry = {
    key = key, status = status, actual = actual_val,
    expected = expected, source = source_str,
    description = description, error = err_msg,
  }
  yg.test._test_registry[key] = entry

  if status == "pass" then
    js.log("[PASS] " .. key)
  elseif status == "error" then
    js.log("[FAIL] " .. key .. " — error: " .. tostring(err_msg))
  else
    js.log("[FAIL] " .. key .. " — expected " .. tostring(expected) .. ", got " .. tostring(actual_val))
  end

  if widget then
    local icon = status == "pass" and "✅" or (status == "error" and "⚠️" or "❌")
    local html = icon .. " <code>" .. key .. "</code>"
    if status == "fail" then
      html = html .. " — expected <code>" .. tostring(expected) .. "</code>, got <code>" .. tostring(actual_val) .. "</code>"
    elseif status == "error" then
      html = html .. " — <code>" .. tostring(err_msg) .. "</code>"
    end
    return widget.html(html)
  end
  return entry
end

-- _exec(prefix, fn): build the suite's `t` and run fn(t) with per-assert pcall
-- protection. The execution primitive shared by run() and runAll().
yg.test._exec = function(prefix, fn)
  local t = {}
  t.assert = function(label, actual, expected, description)
    local ok, err = pcall(function()
      yg.test.assert(prefix .. " : " .. label, actual, expected, description)
    end)
    if not ok then
      yg.test._test_registry[prefix .. " : " .. label] = {
        key = prefix .. " : " .. label, status = "error",
        error = err, actual = nil, expected = expected,
      }
      js.log("[FAIL] " .. prefix .. " : " .. label .. " — runner error: " .. tostring(err))
    end
  end
  fn(t)
end

-- run(prefix, fn): register the suite for eval-phase re-run AND execute it now.
-- Boot execution is preserved transitionally so `node bin/sb-test.mjs` (boot
-- registry, widget-absent) stays green while the eval-phase lane comes online.
yg.test.run = function(prefix, fn)
  yg.test._suites[prefix] = fn
  yg.test._exec(prefix, fn)
end

-- runAll(): re-execute every registered suite in the CURRENT context. Invoked
-- from the runtime API (eval phase → widget present) by `sb-test.mjs --run`, so
-- dual-mode render code takes its DOM branch. Returns a status tally.
yg.test.runAll = function()
  -- Re-entrancy guard: a suite that itself calls runAll() must not recurse
  -- through the whole batch (that hangs). Nested calls short-circuit.
  if yg.test._running then return end
  yg.test._running = true
  for prefix, fn in pairs(yg.test._suites) do
    pcall(yg.test._exec, prefix, fn)
  end
  yg.test._running = false
  local pass, fail, err = 0, 0, 0
  for _, e in pairs(yg.test._test_registry) do
    if e.status == "pass" then pass = pass + 1
    elseif e.status == "error" then err = err + 1
    else fail = fail + 1 end
  end
  return { pass = pass, fail = fail, error = err, total = pass + fail + err }
end

-- show(?prefix, ?style)
yg.test.show = function(prefix, style)
  style = style or "mini"
  local matches = {}
  for key, entry in pairs(yg.test._test_registry) do
    if prefix == nil or string.sub(key, 1, #prefix) == prefix then
      table.insert(matches, entry)
    end
  end
  table.sort(matches, function(a, b) return a.key < b.key end)

  local pass_count, fail_count, err_count = 0, 0, 0
  for _, e in ipairs(matches) do
    if e.status == "pass" then pass_count = pass_count + 1
    elseif e.status == "error" then err_count = err_count + 1
    else fail_count = fail_count + 1 end
  end

  if #matches == 0 then
    local label = prefix and (" for <code>" .. prefix .. "</code>") or ""
    local msg = "<em>No tests registered" .. label .. "</em>"
    if widget then return widget.html(msg) end
    return msg
  end

  local summary = "<p><small>✅ " .. pass_count .. "  ❌ " .. fail_count .. "  ⚠️ " .. err_count .. "</small></p>"

  if style == "mini" then
    local rows = {}
    for _, e in ipairs(matches) do
      local icon = e.status == "pass" and "✅" or (e.status == "error" and "⚠️" or "❌")
      table.insert(rows, "<tr><td><code>" .. e.key .. "</code></td><td>" .. icon .. "</td></tr>")
    end
    local html = "<table>" .. table.concat(rows, "") .. "</table>" .. summary
    if widget then return widget.html(html) end
    return html
  end

  -- style == "full"
  local cards = {}
  for _, e in ipairs(matches) do
    local icon = e.status == "pass" and "✅" or (e.status == "error" and "⚠️" or "❌")
    local border = e.status == "pass" and "#4caf50" or "#f44336"
    local card = "<div style='margin:0.5em 0;padding:0.75em;border-left:3px solid " .. border .. "'>"
    card = card .. "<strong>" .. icon .. " " .. e.key .. "</strong>"
    if e.description then card = card .. "<br><em>" .. e.description .. "</em>" end
    if e.source      then card = card .. "<br>Expr: <code>" .. e.source .. "</code>" end
    if e.status == "error" then
      card = card .. "<br>Error: <code>" .. tostring(e.error) .. "</code>"
    elseif e.status == "fail" then
      card = card .. "<br>Expected: <code>" .. tostring(e.expected)
        .. "</code>  Got: <code>" .. tostring(e.actual) .. "</code>"
    end
    card = card .. "</div>"
    table.insert(cards, card)
  end
  local html = table.concat(cards, "") .. summary
  if widget then return widget.html(html) end
  return html
end

-- Self-tests (run after all three functions are defined above)
yg.test.assert("yg.test.assert() : _selftest_pass", 1 + 1, 2)

yg.test.run("yg.test.assert()", function(t)
  t.assert("pass case registered",  yg.test._test_registry["yg.test.assert() : _selftest_pass"] ~= nil)
  t.assert("pass case status", function()
    return yg.test._test_registry["yg.test.assert() : _selftest_pass"].status
  end, "pass")
  t.assert("expr form evaluates",   "1 + 1", 2)
  t.assert("expr source stored", function()
    return yg.test._test_registry["yg.test.assert() : expr form evaluates"].source
  end, "1 + 1")
  t.assert("truthiness pass",       true)
  t.assert("fail detection", function()
    yg.test.assert("yg.test.assert() : _tmp_fail_check", false)
    local s = yg.test._test_registry["yg.test.assert() : _tmp_fail_check"].status
    yg.test._test_registry["yg.test.assert() : _tmp_fail_check"] = nil
    return s
  end, "fail")
  t.assert("error detection", function()
    yg.test.assert("yg.test.assert() : _tmp_err_check", "nil.boom")
    local s = yg.test._test_registry["yg.test.assert() : _tmp_err_check"].status
    yg.test._test_registry["yg.test.assert() : _tmp_err_check"] = nil
    return s
  end, "error")
end)

yg.test.run("yg.test.run()", function(t)
  yg.test.run("_run_prefix_check", function(t2) t2.assert("marker", true) end)
  t.assert("prefix prepended to key", function()
    return yg.test._test_registry["_run_prefix_check : marker"] ~= nil
  end)
end)

yg.test.run("yg.test.show()", function(t)
  t.assert("returns non-nil for existing prefix", yg.test.show("yg.test.assert()") ~= nil)
  t.assert("returns non-nil for nil prefix",      yg.test.show() ~= nil)
  t.assert("returns non-nil for unknown prefix",  yg.test.show("no.such.prefix.xyz") ~= nil)
end)

yg.test.run("yg.test.runAll()", function(t)
  t.assert("_suites registry is a table", type(yg.test._suites) == "table")
  t.assert("run registers the suite by prefix", function()
    yg.test.run("_ra_probe", function(tt) tt.assert("hit", true) end)
    return yg.test._suites["_ra_probe"] ~= nil
  end)
  t.assert("_exec re-executes a registered suite", function()
    yg.test._test_registry["_ra_probe : hit"] = nil
    yg.test._exec("_ra_probe", yg.test._suites["_ra_probe"])
    return yg.test._test_registry["_ra_probe : hit"] ~= nil
  end)
  t.assert("runAll is callable", type(yg.test.runAll) == "function")
  t.assert("runAll is re-entrancy guarded", function()
    local saved = yg.test._suites
    local marker = false
    yg.test._suites = { _guard_probe = function(tt) marker = true end }
    yg.test._running = true            -- pretend a batch is in progress
    yg.test.runAll()                   -- must short-circuit, not iterate
    yg.test._running = false
    yg.test._suites = saved
    return marker == false
  end)
  -- cleanup the probe so it does not leak into the real registry / runAll batch
  yg.test._suites["_ra_probe"] = nil
  yg.test._test_registry["_ra_probe : hit"] = nil
end)
```

${yg.test.show("yg.test")}

```space-lua
-- priority: 1
-- Final refresh: all startup scripts have completed, re-render widgets once.
if codeWidget then codeWidget.refreshAll() end
```
