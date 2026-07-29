---
description: Pure-Lua type constructors for the yg.oo.define.class / yg.schema system.
tags: meta/library
name: Library/yggi/yg/type
files:
  - ui.md
---

# yg.type

Type constructor system. Provides `yg.type.String`, `yg.type.Ref`, `yg.type.Record`, etc.
used by `yg.oo.define.class` declarations and the `yg.schema` translation layer.

## Namespace

```#yg/object
tags: [luaref]
displayName: "yg.type"
luaref:
  name: yg.type
  type: table
  description: "namespace for type constructors used in yg.oo.define.class declarations"
```

```space-lua
-- priority: 94
-- Tier 94: the shared metatable must exist before any constructor's setmetatable() runs.
-- (Same-priority load order is unspecified in SilverBullet — see sb-scripting §Load Order —
--  so yg.type is tiered: 94 _mt → 93 leaf types → 92 composites/derived + all co-located tests.)
yg = yg or {}
yg.type = yg.type or {}

-- Shared callable metatable for all type constructor values.
-- Calling a type with a string → { help = string }.
-- Calling a type with a table → returns a new constrained copy.
-- The original type value is never mutated.
yg.type._mt = {}
yg.type._mt.__call = function(self, constraints)
  if type(constraints) == "string" then
    constraints = { help = constraints }
  end
  if type(constraints) ~= "table" then return self end
  local copy = {}
  for k, v in pairs(self) do copy[k] = v end
  for k, v in pairs(constraints) do copy[k] = v end
  return setmetatable(copy, yg.type._mt)
end
```

```space-lua
-- priority: 79
-- Self-test at the warm test tier (79), NOT co-located at the def's tier 94. At the early
-- def tiers (92-94) a boot-indexer quirk deterministically dropped _yg_type from __call's
-- pairs(self) copy of a freshly setmetatable'd literal; the identical code passes at tier 79
-- (where the schema/oo suites also run) and via the Runtime API. (See MEMORY §API quirks:
-- boot-vs-Runtime-API divergence — prefer running type tests at the warm test tier.)
yg.test.run("yg.type._mt", function(t)
  t.assert("_mt exists",     yg.type._mt ~= nil)
  t.assert("_mt is table",   type(yg.type._mt) == "table")
  t.assert("_mt has __call", type(yg.type._mt.__call) == "function")
  -- string shorthand
  local probe = setmetatable({ _yg_type = "Test" }, yg.type._mt)
  t.assert("string becomes help", function() return probe("hint").help end, "hint")
  -- table merge
  local c = probe { required = true }
  t.assert("constraint merged",   c.required, true)
  t.assert("_yg_type preserved",  function() return c._yg_type end, "Test")
  t.assert("original unchanged",  probe.required == nil)
  -- result is also callable
  t.assert("result callable",     type(getmetatable(c)) == "table")
end)
```

${yg.test.show("yg.type._mt")}

---

## String

yg.type.String: type value
Base string type. Callable with constraints: `yg.type.String { required = true }`.

```#yg/object
tags: [luaref]
displayName: "yg.type.String"
luaref:
  name: String
  scope: yg.type
  type: table
  description: "String type constructor. Call with a constraint table or help string."
  example: 'yg.type.String { required = true } -> constrained String type'
```

```space-lua
-- priority: 93
yg.type.String = setmetatable({ _yg_type = "String" }, yg.type._mt)

yg.test.run("yg.type.String", function(t)
  t.assert("_yg_type",         function() return yg.type.String._yg_type end, "String")
  t.assert("callable",         type(yg.type.String) == "table")
  t.assert("constraint copy",  function() return yg.type.String { required = true }.required end, true)
  t.assert("original clean",   yg.type.String.required == nil)
  t.assert("help shorthand",   function() return yg.type.String("hint").help end, "hint")
  t.assert("widget constraint", function() return yg.type.String { widget = "textarea" }.widget end, "textarea")
  t.assert("run constraint", function()
    local fn = function(self) return "x" end
    return yg.type.String { run = fn }.run == fn
  end)
end)
```

${yg.test.show("yg.type.String")}

---

## Number

yg.type.Number: type value
Base number type. Callable with constraints.

```#yg/object
tags: [luaref]
displayName: "yg.type.Number"
luaref:
  name: Number
  scope: yg.type
  type: table
  description: "Number type constructor."
  example: 'yg.type.Number { default = 0 } -> Number with default'
```

```space-lua
-- priority: 93
yg.type.Number = setmetatable({ _yg_type = "Number" }, yg.type._mt)

yg.test.run("yg.type.Number", function(t)
  t.assert("_yg_type",        function() return yg.type.Number._yg_type end, "Number")
  t.assert("default",         function() return yg.type.Number { default = 42 }.default end, 42)
  t.assert("nullable",        function() return yg.type.Number { nullable = true }.nullable end, true)
  t.assert("original clean",  yg.type.Number.default == nil)
end)
```

${yg.test.show("yg.type.Number")}

---

## Boolean

yg.type.Boolean: type value
Base boolean type. Callable with constraints.

```#yg/object
tags: [luaref]
displayName: "yg.type.Boolean"
luaref:
  name: Boolean
  scope: yg.type
  type: table
  description: "Boolean type constructor."
  example: 'yg.type.Boolean "should I yell?" -> Boolean with help text'
```

```space-lua
-- priority: 93
yg.type.Boolean = setmetatable({ _yg_type = "Boolean" }, yg.type._mt)

yg.test.run("yg.type.Boolean", function(t)
  t.assert("_yg_type",       function() return yg.type.Boolean._yg_type end, "Boolean")
  t.assert("help shorthand", function() return yg.type.Boolean("should I yell?").help end, "should I yell?")
  t.assert("original clean", yg.type.Boolean.help == nil)
end)
```

${yg.test.show("yg.type.Boolean")}

---

## Id

yg.type.Id: type value
Convenience alias: `yg.type.String { id = true }`. Marks the field as the instance identity key.

```#yg/object
tags: [luaref]
displayName: "yg.type.Id"
luaref:
  name: Id
  scope: yg.type
  type: table
  description: "Alias for yg.type.String { id = true }. Marks the instance identity field."
  example: 'yg.type.Id -> String type with id=true'
```

```space-lua
-- priority: 92
yg.type.Id = yg.type.String { id = true }

yg.test.run("yg.type.Id", function(t)
  t.assert("_yg_type is String", function() return yg.type.Id._yg_type end, "String")
  t.assert("id is true",         yg.type.Id.id, true)
  t.assert("callable",           function() return yg.type.Id { required = true }.required end, true)
  t.assert("id preserved after call", function() return yg.type.Id { required = true }.id end, true)
end)
```

${yg.test.show("yg.type.Id")}

---

## Ref(class_name)

yg.type.Ref(class_name): type value
Reference to another class. Stored as a `[[$ref]]` anchor link (indexed as a native
relation); resolved by casting the target to `class_name` (`class_name$$ref`). See the
"Identity & References" section of the type-system spec. The returned type value is callable
with constraints: `yg.type.Ref "ingredient" { required = true }`.

```#yg/object
tags: [luaref]
displayName: "Ref(class_name)"
luaref:
  name: Ref
  scope: yg.type
  type: function
  description: "Reference to another class. Stored as a [[$ref]] anchor link (native relation); resolved by casting the target to class_name at bootstrap."
  example: 'yg.type.Ref "ingredient" -> Ref type for the ingredient class'
  arguments:
    - name: class_name
      type: string
      description: "name of the referenced class"
```

```space-lua
-- priority: 93
yg.type.Ref = function(arg)
  -- Two authoring forms:
  --   Ref "Class"                          — string shorthand
  --   Ref{ "Class", init=, as=, fixed=, … } — dom-style table (owned-ref attributes)
  local class_name = arg
  local tm = setmetatable({ _yg_type = "Ref" }, yg.type._mt)
  if type(arg) == "table" then
    class_name = arg[1]
    for k, v in pairs(arg) do if k ~= 1 then tm[k] = v end end
  end
  tm.ref_class = class_name
  return tm
end

yg.test.run("yg.type.Ref()", function(t)
  local r = yg.type.Ref "ingredient"
  t.assert("_yg_type",    function() return r._yg_type end, "Ref")
  t.assert("ref_class",   function() return r.ref_class end, "ingredient")
  -- dom-style table form carries owned-ref attributes onto the descriptor
  local o = yg.type.Ref{ "ItemCollection", init = {}, as = "record", fixed = true }
  t.assert("table ref_class", function() return o.ref_class end, "ItemCollection")
  t.assert("table _yg_type",  function() return o._yg_type end, "Ref")
  t.assert("table as",        function() return o.as end, "record")
  t.assert("table fixed",     function() return o.fixed end, true)
  t.assert("table init set",  function() return type(o.init) end, "table")
  t.assert("callable adds constraint", function()
    return yg.type.Ref "person" { required = true } .required
  end, true)
  t.assert("callable preserves ref_class", function()
    return yg.type.Ref "person" { required = true } .ref_class
  end, "person")
  t.assert("callable preserves _yg_type", function()
    return yg.type.Ref "plan" { nullable = true } ._yg_type
  end, "Ref")
end)
```

${yg.test.show("yg.type.Ref()")}

---

## Enum(values)

yg.type.Enum(values): type value
Enumerated string type. `values` is an array of strings. The returned type value is callable
with constraints: `yg.type.Enum { "a", "b" } { required = true }`.

```#yg/object
tags: [luaref]
displayName: "Enum(values)"
luaref:
  name: Enum
  scope: yg.type
  type: function
  description: "Enumerated string type. Stores allowed values; used for JSON Schema enum and CRUD default."
  example: 'yg.type.Enum { "draft", "published" } -> Enum type'
  arguments:
    - name: values
      type: array
      description: "array of allowed string values"
```

```space-lua
-- priority: 93
yg.type.Enum = function(values)
  local vals = {}
  for _, v in ipairs(values) do table.insert(vals, v) end
  return setmetatable({ _yg_type = "Enum", values = vals }, yg.type._mt)
end

yg.test.run("yg.type.Enum()", function(t)
  local e = yg.type.Enum { "draft", "published", "archived" }
  t.assert("_yg_type",    function() return e._yg_type end, "Enum")
  t.assert("values[1]",   function() return e.values[1] end, "draft")
  t.assert("values[2]",   function() return e.values[2] end, "published")
  t.assert("values[3]",   function() return e.values[3] end, "archived")
  t.assert("value count", #e.values, 3)
  t.assert("callable adds required", function()
    return yg.type.Enum { "a", "b" } { required = true } .required
  end, true)
  t.assert("callable preserves values", function()
    return yg.type.Enum { "x", "y" } { required = true } .values[1]
  end, "x")
end)
```

${yg.test.show("yg.type.Enum()")}

---

## Array(item_type)

yg.type.Array(item_type): type value
Ordered integer-indexed array of a uniform type. Callable with constraints.

```#yg/object
tags: [luaref]
displayName: "Array(item_type)"
luaref:
  name: Array
  scope: yg.type
  type: function
  description: "Ordered array of a uniform element type. Maps to JSON Schema {type:array, items:...}."
  example: 'yg.type.Array(yg.type.String) -> Array of strings'
  arguments:
    - name: item_type
      type: table
      description: "any yg.type.* value"
```

```space-lua
-- priority: 92
yg.type.Array = function(item_type)
  return setmetatable({ _yg_type = "Array", item_type = item_type }, yg.type._mt)
end

yg.test.run("yg.type.Array()", function(t)
  local a = yg.type.Array(yg.type.String)
  t.assert("_yg_type",          function() return a._yg_type end, "Array")
  t.assert("item_type",         function() return a.item_type._yg_type end, "String")
  local a2 = yg.type.Array(yg.type.Ref "ingredient")
  t.assert("ref item_type",     function() return a2.item_type._yg_type end, "Ref")
  t.assert("ref item ref_class", function() return a2.item_type.ref_class end, "ingredient")
  t.assert("callable",          function()
    return yg.type.Array(yg.type.Number) { required = true } .required
  end, true)
end)
```

${yg.test.show("yg.type.Array()")}

---

## Map(spec)

yg.type.Map(spec): type value
String-keyed map. Three calling forms:
- `Map(value_type)` — open keys, uniform value type
- `Map { key = key_type, value = value_type }` — typed open keys
- `Map { keys = {"A","B"}, value = value_type }` — fixed slots (implies additionalProperties:false)

`keys = {...}` is sugar for `key = yg.type.Enum{...}` and sets `fixed = true`.

```#yg/object
tags: [luaref]
displayName: "Map(spec)"
luaref:
  name: Map
  scope: yg.type
  type: function
  description: "String-keyed map type. Three forms: shorthand, typed-open, fixed-slots."
  example: 'yg.type.Map(yg.type.Number) -> open-key number map'
  arguments:
    - name: spec
      type: table | type value
      description: "value type (shorthand) or {key, value} or {keys, value} spec table"
```

```space-lua
-- priority: 92
-- Map(value_type)             → shorthand: open string keys
-- Map { key=K, value=V }      → typed open keys
-- Map { keys={...}, value=V } → fixed slots (sugar for key=Enum{...}, sets fixed=true)
yg.type.Map = function(spec)
  -- Shorthand: spec is a yg.type.* value (has _yg_type)
  if type(spec) == "table" and spec._yg_type ~= nil then
    return setmetatable({ _yg_type = "Map", value_type = spec }, yg.type._mt)
  end
  -- Full spec table
  local t = { _yg_type = "Map" }
  if spec.keys then
    -- Fixed slots: sugar for key = Enum{...}
    t.key_type   = yg.type.Enum(spec.keys)
    t.value_type = spec.value
    t.fixed      = true
  else
    t.key_type   = spec.key
    t.value_type = spec.value
  end
  return setmetatable(t, yg.type._mt)
end

yg.test.run("yg.type.Map()", function(t)
  -- shorthand: Map(value_type)
  local m1 = yg.type.Map(yg.type.Number)
  t.assert("shorthand _yg_type",   function() return m1._yg_type end, "Map")
  t.assert("shorthand value_type", function() return m1.value_type._yg_type end, "Number")
  t.assert("shorthand no key_type", m1.key_type == nil)
  t.assert("shorthand no fixed",    m1.fixed ~= true)

  -- typed open keys: Map { key = K, value = V }
  local m2 = yg.type.Map { key = yg.type.Ref "ingredient", value = yg.type.Number }
  t.assert("open key_type",   function() return m2.key_type._yg_type end, "Ref")
  t.assert("open value_type", function() return m2.value_type._yg_type end, "Number")
  t.assert("open no fixed",   m2.fixed ~= true)

  -- fixed slots: Map { keys = {...}, value = V }
  local m3 = yg.type.Map { keys = {"Tank","Healer","DPS"}, value = yg.type.Ref "character" }
  t.assert("fixed key_type is Enum",       function() return m3.key_type._yg_type end, "Enum")
  t.assert("fixed key_type values[1]",     function() return m3.key_type.values[1] end, "Tank")
  t.assert("fixed value_type",             function() return m3.value_type._yg_type end, "Ref")
  t.assert("fixed flag",                   m3.fixed, true)

  -- callable
  t.assert("callable on shorthand", function()
    return yg.type.Map(yg.type.String) { required = true } .required
  end, true)
end)
```

${yg.test.show("yg.type.Map()")}

---

## Record(fields)

yg.type.Record(fields): type table
Fixed named-field composite type. Models a YAML subtree. Returns a plain table (not callable)
with `_yg_type = "Record"`, the field entries, and `_order` listing field names in
declaration order.

```#yg/object
tags: [luaref]
displayName: "Record(fields)"
luaref:
  name: Record
  scope: yg.type
  type: function
  description: "Fixed named-field record type. Fields and their declaration order are preserved in _order."
  example: 'yg.type.Record { calories = yg.type.Number } -> Record type with one field'
  arguments:
    - name: fields
      type: table
      description: "field name → yg.type.* value pairs in declaration order"
```

```space-lua
-- priority: 92
-- Record receives a plain fields table. Since pairs() preserves insertion order
-- in Space Lua (V8-backed), we capture _order via a single pass. We build a fresh
-- result table to avoid mutating the caller's table literal.
yg.type.Record = function(fields)
  local result  = { _yg_type = "Record" }
  local _order = {}
  for k, v in pairs(fields) do
    if type(k) == "string" then
      table.insert(_order, k)
      result[k] = v
    end
  end
  result._order = _order
  return result
end

yg.test.run("yg.type.Record()", function(t)
  local r = yg.type.Record {
    calories = yg.type.Number,
    protein  = yg.type.Number,
    fat      = yg.type.Number,
  }
  t.assert("_yg_type",              function() return r._yg_type end, "Record")
  t.assert("has _order",            r._order ~= nil)
  t.assert("order length",          #r._order, 3)
  t.assert("first field in order",  function() return r._order[1] end, "calories")
  t.assert("second field in order", function() return r._order[2] end, "protein")
  t.assert("third field in order",  function() return r._order[3] end, "fat")
  t.assert("fields accessible",     function() return r.calories._yg_type end, "Number")
  t.assert("_order not a field",    r._order[1] ~= "_order")
  t.assert("_yg_type not in order", function()
    for _, k in ipairs(r._order) do
      if k == "_yg_type" or k == "_order" then return false end
    end
    return true
  end)
  -- nested Record
  local nested = yg.type.Record {
    source = yg.type.Record { url = yg.type.String, year = yg.type.Number },
    title  = yg.type.String,
  }
  t.assert("nested _yg_type",          function() return nested._yg_type end, "Record")
  t.assert("nested field _yg_type",    function() return nested.source._yg_type end, "Record")
  t.assert("nested inner field",       function() return nested.source.url._yg_type end, "String")
end)
```

${yg.test.show("yg.type.Record()")}

---

## Function(spec)

yg.type.Function(spec): type table
Behavioral type — rendered as an interactive row (inputs + GO button) in a card. Returns a
plain table (not callable) with `_yg_type = "Function"`, `run` fn, and `args` table.
`args` has `_order` for argument positional order.

```#yg/object
tags: [luaref]
displayName: "Function(spec)"
luaref:
  name: Function
  scope: yg.type
  type: function
  description: "Interactive method type. Rendered as input row + button. run receives self + arg values."
  example: 'yg.type.Function { args={factor=yg.type.Number}, run=fn } -> Function type'
  arguments:
    - name: spec
      type: table
      description: "{args = {name → type}, run = function(self, ...) end}"
```

```space-lua
-- priority: 92
-- Function receives a spec table {args, run}. We build a fresh result table.
-- args (if present) gets _order via pairs() (declaration order preserved in Space Lua).
yg.type.Function = function(spec)
  local result = { _yg_type = "Function", run = spec.run }
  if spec.args then
    local ordered_args = {}
    local args_order   = {}
    for k, v in pairs(spec.args) do
      if type(k) == "string" then
        table.insert(args_order, k)
        ordered_args[k] = v
      end
    end
    ordered_args._order = args_order
    result.args = ordered_args
  end
  return result
end

yg.test.run("yg.type.Function()", function(t)
  local fn = function(self, factor) return (self.recipe.servings or 1) * factor end
  local f = yg.type.Function {
    args   = { factor = yg.type.Number "scale factor" },
    run    = fn,
  }
  t.assert("_yg_type",           function() return f._yg_type end, "Function")
  t.assert("has run",            type(f.run) == "function")
  t.assert("run is correct",     f.run == fn)
  t.assert("args exists",        f.args ~= nil)
  t.assert("args has factor",    f.args.factor ~= nil)
  t.assert("arg _yg_type",       function() return f.args.factor._yg_type end, "Number")
  t.assert("arg help",           function() return f.args.factor.help end, "scale factor")
  t.assert("args has _order",    f.args._order ~= nil)
  t.assert("args _order[1]",     function() return f.args._order[1] end, "factor")

  -- multi-arg ordering
  local f2 = yg.type.Function {
    args = {
      yell   = yg.type.Boolean "should I yell?",
      prefix = yg.type.String "optional prefix",
    },
    run = function(self, yell, prefix) return "" end,
  }
  t.assert("multi-arg _order[1]", function() return f2.args._order[1] end, "yell")
  t.assert("multi-arg _order[2]", function() return f2.args._order[2] end, "prefix")

  -- no-args Function
  local f3 = yg.type.Function { run = function(self) return "computed" end }
  t.assert("no-args run", type(f3.run) == "function")
  t.assert("no-args no args table", f3.args == nil)
end)
```

${yg.test.show("yg.type.Function()")}

---

## raw(json)

yg.type.raw(json): type table
Escape hatch — the given JSON Schema fragment is emitted **verbatim** by `yg.schema`. Carries
`_yg_type = "raw"` internally so renderers/CRUD can identify it, but no other `yg` metadata is
applied. Use for schema features `yg.type` does not model (`format`, `oneOf`, `minimum`, …).

```#yg/object
tags: [luaref]
displayName: "raw(json)"
luaref:
  name: raw
  scope: yg.type
  type: function
  description: "Escape hatch: JSON Schema fragment emitted verbatim. Not modelled by other yg.type constructors."
  example: 'yg.type.raw { type = "string", format = "date" } -> verbatim schema fragment'
  arguments:
    - name: json
      type: table
      description: "a JSON Schema fragment, passed through untouched"
```

```space-lua
-- priority: 92
yg.type.raw = function(json)
  return { _yg_type = "raw", json = json }
end

yg.test.run("yg.type.raw()", function(t)
  local r = yg.type.raw { type = "object", additionalProperties = true }
  t.assert("_yg_type",      function() return r._yg_type end, "raw")
  t.assert("json preserved", function() return r.json.type end, "object")
  t.assert("json verbatim",  function() return r.json.additionalProperties end, true)
end)
```

${yg.test.show("yg.type.raw()")}

---

## Backref(spec)

yg.type.Backref(spec): type value
Promotes one incoming relationship to a named, read-only field. Sugar over a **derived
`Array(Ref SourceClass)`** whose `derive` slices `self:_backrefs().structural[Class][field]`
into `{_ref,_class}` sentinels — so it rides existing `Array(Ref)` rendering and adds no
engine. Carries `backref = { class, kind }` metadata so the generic "Referenced by" panel can
dedup it (a promoted relationship shows once, as its field, not twice). Authoring forms:
`Backref{ "Recipe", "result" }` (dom-style) or `Backref("Recipe", "result")`.

```#yg/object
tags: [luaref]
displayName: "Backref(spec)"
luaref:
  name: Backref
  scope: yg.type
  type: function
  description: "Promote an incoming relationship to a read-only derived Array(Ref) field. Sugar over _backrefs()."
  example: 'yg.type.Backref{ "Recipe", "result" } -> read-only recipes that reference this object as result'
  arguments:
    - name: spec
      type: table | string
      description: '{ "SourceClass", "field" } (or two positional string args)'
```

```space-lua
-- priority: 92
-- Backref{ "SourceClass", "field" }  or  Backref("SourceClass", "field")
-- A function (refs Array/Ref at call time, not load time → no same-tier load-order issue).
yg.type.Backref = function(spec, field2)
  local source_class, kind
  if type(spec) == "table" then
    source_class, kind = spec[1], spec[2]
  else
    source_class, kind = spec, field2
  end
  return yg.type.Array(yg.type.Ref(source_class)) {
    backref = { class = source_class, kind = kind },
    derive  = function(self)
      local br = self._backrefs and self:_backrefs()
      local by = br and br.structural and br.structural[source_class]
      local refs = (by and by[kind]) or {}
      local out = {}
      for _, ref in ipairs(refs) do
        table.insert(out, { _ref = ref, _class = source_class })
      end
      return out
    end,
  }
end
```

```space-lua
-- priority: 79
-- Self-test at the warm tier (79): depends on yg.type.isComputed (priority 91) and exercises
-- the derive — neither is safe co-located at the def's tier 92 (see the boot-indexer caveat).
yg.test.run("yg.type.Backref()", function(t)
  local b = yg.type.Backref{ "Recipe", "result" }
  t.assert("is Array",           function() return b._yg_type end, "Array")
  t.assert("item is Ref",        function() return b.item_type._yg_type end, "Ref")
  t.assert("item ref_class",     function() return b.item_type.ref_class end, "Recipe")
  t.assert("is computed",        yg.type.isComputed(b), true)
  t.assert("backref class",      function() return b.backref.class end, "Recipe")
  t.assert("backref kind",       function() return b.backref.kind end, "result")
  -- positional form
  local b2 = yg.type.Backref("Dish", "main")
  t.assert("positional kind",    function() return b2.backref.kind end, "main")
  -- derive slices an injected _backrefs
  local fake = { _backrefs = function() return { structural = { Recipe = { result = { "rec_vigor", "rec_stable" } } } } end }
  local got = b.derive(fake)
  t.assert("derive count",       #got, 2)
  t.assert("derive sentinel ref", function() return got[1]._ref end, "rec_vigor")
  t.assert("derive sentinel class", function() return got[1]._class end, "Recipe")
  -- empty when nothing matches
  local empty = b.derive({ _backrefs = function() return { structural = {} } end })
  t.assert("empty derive",       #empty, 0)
end)
```

${yg.test.show("yg.type.Backref()")}

---

## isComputed(td)

yg.type.isComputed(td): boolean
True iff `td` is a **value** type carrying a `derive` function — i.e. a computed/derived
property, not a stored one. `Function` types (behavioral, `run`) are never "computed" in this
sense. Used by `yg.schema` to exclude derive-fields from stored JSON Schema, and by
`object:_get` to evaluate the derived value on read.

```#yg/object
tags: [luaref]
displayName: "isComputed(td)"
luaref:
  name: isComputed
  scope: yg.type
  type: function
  description: "True iff td is a value type with a derive function (a computed field, not stored)."
  example: 'yg.type.isComputed(yg.type.String{ derive = f }) -> true'
  arguments:
    - name: td
      type: table
      description: "a yg.type.* type value"
```

```space-lua
-- priority: 91
yg.type.isComputed = function(td)
  return type(td) == "table" and td._yg_type ~= "Function" and type(td.derive) == "function"
end
```

```space-lua
-- priority: 79
yg.test.run("yg.type.derive", function(t)
  local f = function(self) return "x" end
  t.assert("String preserves derive",      yg.type.String{ derive = f }.derive == f)
  t.assert("isComputed on derive String",  yg.type.isComputed(yg.type.String{ derive = f }) == true)
  t.assert("isComputed false on plain",    yg.type.isComputed(yg.type.String) == false)
  t.assert("isComputed false on Function", yg.type.isComputed(yg.type.Function{ run = f }) == false)
end)
```

${yg.test.show("yg.type.derive")}

---

## Tests

${yg.test.show("yg.type")}
