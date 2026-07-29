---
description: "yg.type.ui — per-type {view,input} tables + dispatchers with tm.ui override hook."
tags: ["meta"]
---

# yg.type.ui — per-type view / input

```space-lua
-- priority: 90
yg = yg or {}; yg.type = yg.type or {}; yg.type.ui = yg.type.ui or {}

-- Dispatchers — check tm.ui override first, then per-type table, then fallback.
yg.type.ui.view = function(tm, v, ctx)
  local r = (tm and tm.ui and tm.ui.view) or (yg.type.ui[tm and tm._yg_type] or {}).view
  return r and r(tm, v, ctx) or (v ~= nil and tostring(v) or "")
end
yg.type.ui.input = function(tm, v, ctx)
  local r = (tm and tm.ui and tm.ui.input) or (yg.type.ui[tm and tm._yg_type] or {}).input
  return r and r(tm, v, ctx) or { node = "", get = function() return v end }
end

-- row_editor(opts): shared DOM row-table builder for Array and Map composite editors.
-- Only call in widget mode (callers must guard with `if not widget then return ... end`).
-- opts.css          : CSS class for the <table>
-- opts.init         : list of {key, val} pairs (key may be nil for arrays)
-- opts.make_tds(k,v): returns {tds={<td> nodes}, key_fn=fn|nil, val_fn=fn}
-- opts.add_footer(insert_fn, get_rows_fn): returns an add-row <tr>
-- opts.assemble(rows): builds get() result from rows list
local function row_editor(opts)
  local rows = {}
  local add_tr

  local function insert_row(k, v)
    local td_info = opts.make_tds(k, v)
    local entry = { key_fn = td_info.key_fn, val_fn = td_info.val_fn }
    local rm_btn = dom.button {
      class = "yg-method-btn yg-map-remove",
      onclick = function()
        entry.tr.remove()
        for i, r in ipairs(rows) do if r == entry then table.remove(rows, i); break end end
      end,
      "✕"
    }
    local tr_spec = {}
    for _, td in ipairs(td_info.tds) do table.insert(tr_spec, td) end
    table.insert(tr_spec, dom.td { rm_btn })
    local tr = dom.tr(tr_spec)
    entry.tr = tr
    table.insert(rows, entry)
    return tr
  end

  local tspec = { class = opts.css or "yg-map-editor" }
  for _, pair in ipairs(opts.init or {}) do
    table.insert(tspec, insert_row(pair[1], pair[2]))
  end
  add_tr = opts.add_footer(
    function(k, v) add_tr.before(insert_row(k, v)) end,
    function() return rows end
  )
  table.insert(tspec, add_tr)

  return {
    node = dom.div { class = "yg-map-input", dom.table(tspec) },
    get  = function() return opts.assemble(rows) end,
  }
end

-- composite_seed(tm, v): serialise a composite value to its headless string form.
-- Used as the text-input seed for Map/Record/Array inputs.
local function composite_seed(tm, v)
  if not v then return "" end
  local k = tm and tm._yg_type
  if k == "Map" then
    local parts = {}
    for mk, mv in pairs(v) do table.insert(parts, tostring(mk) .. ": " .. tostring(mv)) end
    return table.concat(parts, ", ")
  elseif k == "Record" then
    local parts = {}
    for _, name in ipairs(tm._order or {}) do
      table.insert(parts, name .. "=" .. tostring(v[name] or ""))
    end
    return table.concat(parts, ", ")
  elseif k == "Array" then
    local parts = {}
    for _, it in ipairs(v) do table.insert(parts, tostring(it)) end
    return table.concat(parts, ", ")
  end
  return tostring(v)
end

-- ── Per-type tables ──────────────────────────────────────────────────────────

-- String: plain label display; text input.
yg.type.ui.String = {
  view = function(tm, v, ctx)
    if v == nil then return "" end
    return tostring(v)
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local inp = dom.input { type = "text", value = (v ~= nil and tostring(v)) or "" }
    return { node = inp, get = function() return inp.value end }
  end,
}

-- Number: plain label; numeric input (coerced via tonumber).
yg.type.ui.Number = {
  view = function(tm, v, ctx)
    if v == nil then return "" end
    return tostring(v)
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local inp = dom.input { type = "number", value = (v ~= nil and tostring(v)) or "" }
    return { node = inp, get = function() return tonumber(inp.value) end }
  end,
}

-- Boolean: checkmark/dash view; toggle-button input.
yg.type.ui.Boolean = {
  view = function(tm, v, ctx)
    return v and "✓" or "—"
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local field = ctx and ctx._field or ""
    local state = (v == true)
    local btn
    btn = dom.button {
      class = "yg-method-btn yg-toggle",
      onclick = function()
        state = not state
        btn.textContent = field .. ": " .. (state and "on" or "off")
      end,
      field .. ": " .. (state and "on" or "off")
    }
    return { node = btn, get = function() return state end }
  end,
}

-- Enum: plain label; <select> over tm.values.
yg.type.ui.Enum = {
  view = function(tm, v, ctx)
    if v == nil then return "" end
    return tostring(v)
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local spec = {}
    for _, opt in ipairs(tm.values or {}) do
      local o = { value = opt, opt }
      if opt == v then o.selected = true end
      table.insert(spec, dom.option(o))
    end
    local sel = dom.select(spec)
    return { node = sel, get = function() return sel.value end }
  end,
}

-- Id: same as String (identity string, no special display).
yg.type.ui.Id = {
  view = function(tm, v, ctx)
    if v == nil then return "" end
    return tostring(v)
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local inp = dom.input { type = "text", value = (v ~= nil and tostring(v)) or "" }
    return { node = inp, get = function() return inp.value end }
  end,
}

-- Ref: resolve via ctx.deref(ctx._field); display inline pill or broken marker.
-- Input: pill display + Change button opening yg.ui.picker.
yg.type.ui.Ref = {
  view = function(tm, v, ctx)
    local view = ctx.deref(ctx._field)
    if view then
      -- as="record": render the target's facet inline instead of a pill (Layer 3 callback).
      if tm.as == "record" and ctx.inline_record then return ctx.inline_record(view) end
      return ctx.inline(view)
    end
    return ctx.broken(ctx.get(ctx._field))
  end,
  input = function(tm, v, ctx)
    -- Seed: v is a sentinel {_ref, _class} from Pass 4, or nil.
    local display_name = "none"
    local initial_link = nil
    if v and v._ref then
      local ref = v._ref
      if string.find(ref, "@") == nil then
        initial_link = "[[$" .. ref .. "]]"
      end
      display_name = (ctx and ctx.display_name and ctx.display_name(ref)) or ref
    end
    if not widget then
      return { node = "", get = function() return initial_link end }
    end
    local link_state = initial_link
    local pill_span  = dom.span { class = "yg-ref-current", display_name }
    local btn = dom.button {
      class = "yg-method-btn",
      onclick = function()
        local sel = yg.ui.picker({
          label       = "Pick " .. (tm.ref_class or "object"),
          tags        = { tm.ref_class },
          placeholder = "Search...",
        })
        if not sel then return end
        link_state = sel.link
        pill_span.textContent = sel.name
      end,
      "Change..."
    }
    return {
      node = dom.span { class = "yg-ref-input", pill_span, " ", btn },
      get  = function() return link_state end,
    }
  end,
}

-- Array: joins items (Ref→inline pills, scalars→tostring).
-- Input: Ref item_type → picker-based row editor; scalar → typed row editor.
yg.type.ui.Array = {
  view = function(tm, v, ctx)
    local items = {}
    if tm.item_type and tm.item_type._yg_type == "Ref" then
      local views = ctx.deref(ctx._field) or {}
      for _, view in ipairs(views) do table.insert(items, ctx.inline(view)) end
    elseif v then
      for _, it in ipairs(v) do table.insert(items, tostring(it)) end
    end
    if widget then
      local spec = { class = "yg-array-list" }
      for _, it in ipairs(items) do
        table.insert(spec, dom.div { class = "yg-array-item", it })
      end
      return widget.html(dom.div(spec))
    end
    return table.concat(items, ", ")
  end,
  input = function(tm, v, ctx)
    local seed = composite_seed(tm, v)
    if not widget then return { node = "", get = function() return seed end } end
    local it = tm.item_type

    -- Array(Ref): picker-based add; rows display the object name (fixed once added).
    if it and it._yg_type == "Ref" then
      local ref_class = it.ref_class
      local init = {}
      for _, item in ipairs(v or {}) do
        local ref = (type(item) == "table" and item._ref)
                 or (yg.name and yg.name.normalize(tostring(item)))
                 or tostring(item)
        table.insert(init, { nil, ref })
      end
      return row_editor({
        css  = "yg-array-editor",
        init = init,
        make_tds = function(_, ref_key)
          local nm   = (ref_key and ctx and ctx.display_name and ctx.display_name(ref_key)) or ref_key or "?"
          local link = ref_key and ("[[$" .. ref_key .. "]]") or nil
          -- Build a pill DOM node without widget.html — safe to call from onclick contexts.
          local pill_node = dom.span { class = "yg-pill",
            dom.span { class = "yg-label", ref_class },
            dom.span { class = "yg-label", nm },
          }
          return {
            tds    = { dom.td { class = "yg-map-key", pill_node } },
            key_fn = nil,
            val_fn = function() return link end,
          }
        end,
        add_footer = function(insert_fn, _)
          return dom.tr { dom.td { colspan = "2",
            dom.button {
              class   = "yg-method-btn",
              onclick = function()
                local sel = yg.ui.picker({ label = "Add " .. ref_class, tags = { ref_class }, placeholder = "Search..." })
                if not sel or not sel.link then return end
                local ref_key = yg.name and yg.name.normalize(sel.link)
                if not ref_key then return end
                insert_fn(nil, ref_key)
              end,
              "+ Add " .. ref_class
            }
          } }
        end,
        assemble = function(rows)
          local result = {}
          for _, r in ipairs(rows) do
            local lnk = r.val_fn()
            if lnk then table.insert(result, lnk) end
          end
          return result
        end,
      })
    end

    -- Array(scalar): one typed input per row; blank-row add button.
    local init = {}
    for _, item in ipairs(v or {}) do table.insert(init, { nil, item }) end
    return row_editor({
      css  = "yg-array-editor",
      init = init,
      make_tds = function(_, val)
        local ctl = yg.type.ui.input(it or { _yg_type = "String" }, val, ctx)
        return {
          tds    = { dom.td { ctl.node } },
          key_fn = nil,
          val_fn = ctl.get,
        }
      end,
      add_footer = function(insert_fn, _)
        return dom.tr { dom.td { colspan = "2",
          dom.button {
            class   = "yg-method-btn",
            onclick = function() insert_fn(nil, nil) end,
            "+ Add"
          }
        } }
      end,
      assemble = function(rows)
        local result = {}
        for _, r in ipairs(rows) do table.insert(result, r.val_fn()) end
        return result
      end,
    })
  end,
}

-- Map: k:v display (Ref-key → table with pill keys; plain → span).
-- Input: Ref-key → structured row editor; plain → text seed.
yg.type.ui.Map = {
  view = function(tm, v, ctx)
    local vt = tm.value_type
    local kt = tm.key_type
    local use_ref_keys = kt and kt._yg_type == "Ref"
    if widget then
      if use_ref_keys then
        local tspec = { class = "yg-map-view" }
        if v then
          for mk, mv in pairs(v) do
            local obj = ctx and ctx.deref_ref and ctx.deref_ref(mk, kt.ref_class)
            local key_cell = obj and ctx.inline(obj) or ("⚠ " .. tostring(mk))
            local val_cell = tostring((vt and yg.type.ui.view(vt, mv, ctx)) or tostring(mv))
            table.insert(tspec, dom.tr {
              dom.td { class = "yg-map-view-key", key_cell },
              dom.td { class = "yg-map-view-val", val_cell },
            })
          end
        end
        return widget.html(dom.table(tspec))
      else
        local spec = { class = "yg-map" }
        local first = true
        if v then
          for mk, mv in pairs(v) do
            if not first then table.insert(spec, ", ") end
            first = false
            table.insert(spec, tostring(mk) .. ": ")
            local vd = (vt and yg.type.ui.view(vt, mv, ctx)) or tostring(mv)
            table.insert(spec, tostring(vd))
          end
        end
        return widget.html(dom.span(spec))
      end
    end
    -- headless
    local items = {}
    if v then
      for mk, mv in pairs(v) do
        local key_disp
        if use_ref_keys then
          local obj = ctx and ctx.deref_ref and ctx.deref_ref(mk, kt.ref_class)
          key_disp = obj and ctx.inline(obj) or ("⚠ " .. tostring(mk))
        else
          key_disp = tostring(mk)
        end
        local vd = (vt and yg.type.ui.view(vt, mv, ctx)) or tostring(mv)
        local sep = use_ref_keys and " → " or ": "
        table.insert(items, tostring(key_disp) .. sep .. tostring(vd))
      end
    end
    return table.concat(items, ", ")
  end,
  input = function(tm, v, ctx)
    -- Map(Ref-key): picker-based row editor via shared row_editor.
    if widget and tm.key_type and tm.key_type._yg_type == "Ref" then
      local ref_class = tm.key_type.ref_class
      local init = {}
      if v then for mk, mv in pairs(v) do table.insert(init, { mk, mv }) end end
      return row_editor({
        css  = "yg-map-editor",
        init = init,
        make_tds = function(mk, mv)
          -- mk is a bare ref (normalized by Pass 4 resolveRefs)
          local nm       = (ctx and ctx.display_name and ctx.display_name(mk)) or mk or "?"
          local obj      = ctx and ctx.deref_ref and ctx.deref_ref(mk, ref_class)
          local key_disp = obj and dom.span { class = "yg-map-key-label", nm }
                        or dom.span { class = "yg-map-key-label yg-method-err", "⚠ " .. tostring(mk) }
          local val_ctl  = yg.type.ui.input(tm.value_type, mv, ctx)
          return {
            tds    = { dom.td { class = "yg-map-key", key_disp }, dom.td { val_ctl.node } },
            key_fn = function() return mk end,
            val_fn = val_ctl.get,
          }
        end,
        add_footer = function(insert_fn, get_rows_fn)
          return dom.tr { dom.td { colspan = "3",
            dom.button {
              class   = "yg-method-btn",
              onclick = function()
                local sel = yg.ui.picker({ label = "Add " .. ref_class, tags = { ref_class }, placeholder = "Search..." })
                if not sel or not sel.link then return end
                local key = yg.name.normalize(sel.link)
                if not key then return end
                for _, r in ipairs(get_rows_fn()) do if r.key_fn() == key then return end end
                insert_fn(key, nil)
              end,
              "+ Add " .. ref_class
            }
          } }
        end,
        assemble = function(rows)
          local result = {}
          for _, r in ipairs(rows) do result["[[$" .. r.key_fn() .. "]]"] = r.val_fn() end
          return result
        end,
      })
    end

    -- Headless fallback for all plain Map variants.
    local seed = composite_seed(tm, v)
    if not widget then return { node = "", get = function() return seed end } end

    -- Map(plain): free-text key + typed value input per row.
    local vt   = tm.value_type
    local init = {}
    if v then for mk, mv in pairs(v) do table.insert(init, { mk, mv }) end end
    return row_editor({
      css  = "yg-map-editor",
      init = init,
      make_tds = function(k, val)
        local key_inp = dom.input { type = "text", value = tostring(k or "") }
        local val_ctl = yg.type.ui.input(vt or { _yg_type = "String" }, val, ctx)
        return {
          tds    = { dom.td { class = "yg-map-key", key_inp }, dom.td { val_ctl.node } },
          key_fn = function() return key_inp.value end,
          val_fn = val_ctl.get,
        }
      end,
      add_footer = function(insert_fn, _)
        return dom.tr { dom.td { colspan = "3",
          dom.button {
            class   = "yg-method-btn",
            onclick = function() insert_fn("", nil) end,
            "+ Add"
          }
        } }
      end,
      assemble = function(rows)
        local result = {}
        for _, r in ipairs(rows) do
          local k = r.key_fn()
          if k and k ~= "" then result[k] = r.val_fn() end
        end
        return result
      end,
    })
  end,
}

-- Record: joins fields in order with typed values.
-- Input: sub-table with one typed input per field (recursive via yg.type.ui.input).
yg.type.ui.Record = {
  view = function(tm, v, ctx)
    local items = {}
    for _, name in ipairs(tm._order or {}) do
      local disp = yg.type.ui.view(tm[name], v and v[name], ctx)
      table.insert(items, name .. "=" .. tostring(disp))
    end
    if widget then
      local spec = { class = "yg-record" }
      for i, it in ipairs(items) do
        if i > 1 then table.insert(spec, ", ") end
        table.insert(spec, it)
      end
      return widget.html(dom.span(spec))
    end
    return table.concat(items, ", ")
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local sub_gets = {}
    local trs = {}
    for _, name in ipairs(tm._order or {}) do
      local ftm = tm[name]
      if ftm then
        local fctx = setmetatable({ _field = name }, { __index = ctx })
        local ctl  = yg.type.ui.input(ftm, v and v[name], fctx)
        sub_gets[name] = ctl.get
        table.insert(trs, dom.tr {
          dom.td { class = "yg-label", name },
          dom.td { ctl.node },
        })
      end
    end
    local tspec = { class = "yg-record-editor" }
    for _, tr in ipairs(trs) do table.insert(tspec, tr) end
    return {
      node = dom.table(tspec),
      get  = function()
        local result = {}
        for name, gfn in pairs(sub_gets) do result[name] = gfn() end
        return result
      end,
    }
  end,
}

-- raw: plain fallback (same as String / Number / Enum scalar branch).
yg.type.ui.raw = {
  view = function(tm, v, ctx)
    if v == nil then return "" end
    return tostring(v)
  end,
  input = function(tm, v, ctx)
    if not widget then return { node = "", get = function() return v end } end
    local inp = dom.input { type = "text", value = (v ~= nil and tostring(v)) or "" }
    return { node = inp, get = function() return inp.value end }
  end,
}
```

```space-lua
-- priority: 79
-- Tests retargeted from yg.ui.value / yg.ui.input → yg.type.ui.view / yg.type.ui.input.
-- ctx._field carries the field name (replaces old 4th positional arg).

yg.test.run("yg.type.ui.view", function(t)
  local ctx = {
    _field = "desc",
    get    = function(f) return ({ desc = "hi", cnt = 3, main = { _ref = "x", _class = "ingredient" } })[f] end,
    deref  = function(f)
      if f == "main"   then return { _render = function() return "Ingredient: Basil" end } end
      if f == "gone"   then return nil end
      if f == "extras" then return { { _render = function() return "Ingredient: Garlic" end } } end
    end,
    inline = function(view) return view:_render("pill") end,
    broken = function(s) return "⚠ " .. tostring(s._class) end,
  }
  t.assert("String",     function() return yg.type.ui.view({ _yg_type = "String" }, "hi", ctx) end, "hi")
  t.assert("Number",     function() return yg.type.ui.view({ _yg_type = "Number" }, 3, ctx) end, "3")
  t.assert("Bool true",  function() return yg.type.ui.view({ _yg_type = "Boolean" }, true, ctx) end, "✓")
  t.assert("Bool false", function() return yg.type.ui.view({ _yg_type = "Boolean" }, false, ctx) end, "—")
  t.assert("Enum",       function() return yg.type.ui.view({ _yg_type = "Enum", values = {"a","b"} }, "b", ctx) end, "b")
  t.assert("nil → empty",function() return yg.type.ui.view({ _yg_type = "String" }, nil, ctx) end, "")
  -- Ref resolves → pill via ctx.inline (ctx._field = "main")
  local rctx = setmetatable({ _field = "main" }, { __index = ctx })
  t.assert("Ref resolved", function() return yg.type.ui.view({ _yg_type = "Ref", ref_class = "ingredient" }, nil, rctx) end, "Ingredient: Basil")
  -- Ref dangling → broken marker
  local dctx = setmetatable({ _field = "gone" }, {
    __index = { get = function() return { _ref = "y", _class = "ingredient" } end, deref = function() return nil end, broken = ctx.broken, inline = ctx.inline }
  })
  t.assert("Ref dangling", function() return yg.type.ui.view({ _yg_type = "Ref", ref_class = "ingredient" }, nil, dctx) end, "⚠ ingredient")
  -- Array of Ref → joined pills
  local actx = setmetatable({ _field = "extras" }, { __index = ctx })
  t.assert("Array of Ref", function() return yg.type.ui.view({ _yg_type = "Array", item_type = { _yg_type = "Ref", ref_class = "ingredient" } }, nil, actx) end, "Ingredient: Garlic")
  -- Array of scalars
  t.assert("Array scalar", function() return yg.type.ui.view({ _yg_type = "Array", item_type = { _yg_type = "String" } }, { "x", "y" }, ctx) end, "x, y")
end)

yg.test.run("yg.type.ui.view composites", function(t)
  local ctx = {}
  local mapTm = { _yg_type = "Map", value_type = { _yg_type = "Number" } }
  t.assert("Map renders k: v (order-independent)", function()
    local s = yg.type.ui.view(mapTm, { sulphur = 2, salt = 1 }, ctx)
    return string.find(s, "sulphur: 2", 1, true) ~= nil and string.find(s, "salt: 1", 1, true) ~= nil
  end, true)
  t.assert("Map empty → empty string", function() return yg.type.ui.view(mapTm, {}, ctx) end, "")
  local recTm = {
    _yg_type = "Record", _order = { "potency", "element", "instant" },
    potency = { _yg_type = "Number" },
    element = { _yg_type = "Enum", values = { "fire" } },
    instant = { _yg_type = "Boolean" },
  }
  t.assert("Record renders fields in order with typed values", function()
    return yg.type.ui.view(recTm, { potency = 8, element = "fire", instant = false }, ctx)
  end, "potency=8, element=fire, instant=—")
end)

yg.test.run("yg.type.ui.view Map Ref key", function(t)
  local base = yg.oo.refs and yg.oo.refs["rec_vigor"]
  local r  = base and base.Recipe
  local cd = yg.oo.classes  and yg.oo.classes["Recipe"]
  if not r or not cd then t.assert("skipped — no recipe objects", true); return end
  local tm  = cd.schema["ingredients"]
  local ctx = {
    inline    = function(view) return view:_render("pill") end,
    deref     = function() return nil end,
    deref_ref = function(mk, class_name)
      local base = yg.oo.refs and yg.oo.refs[mk]
      return base and base[class_name] or nil
    end,
  }
  local out = yg.type.ui.view(tm, r.ingredients, ctx)
  t.assert("contains →",              function() return string.find(out, " → ", 1, true) ~= nil end, true)
  -- missing ref falls back to ⚠ (bare ref, as normalized by Pass 4)
  local out2 = yg.type.ui.view(tm, { ["nope-nope"] = 1 }, ctx)
  t.assert("missing ref shows ⚠",    function() return string.find(out2, "⚠", 1, true) ~= nil end, true)
end)

yg.test.run("yg.type.ui.input composite", function(t)
  local mi = yg.type.ui.input({ _yg_type = "Map", value_type = { _yg_type = "Number" } }, { a = 1, b = 2 }, {})
  t.assert("Map returns table",   function() return type(mi) end, "table")
  t.assert("Map seed has a: 1",   function() return string.find(mi.get(), "a: 1", 1, true) ~= nil end, true)
  t.assert("Map seed has b: 2",   function() return string.find(mi.get(), "b: 2", 1, true) ~= nil end, true)
  local rt = { _yg_type = "Record", _order = { "x", "y" }, x = { _yg_type = "Number" }, y = { _yg_type = "String" } }
  local ri = yg.type.ui.input(rt, { x = 3, y = "hi" }, {})
  t.assert("Record get() table",  function() return type(ri.get()) end, "table")
  t.assert("Record get() x=3",    function() return ri.get().x end, 3)
  t.assert("Record get() y=hi",   function() return ri.get().y end, "hi")
  local ai = yg.type.ui.input({ _yg_type = "Array", item_type = { _yg_type = "String" } }, { "p", "q" }, {})
  t.assert("Array seed",          function() return ai.get() end, "p, q")
  local ni = yg.type.ui.input({ _yg_type = "Map", value_type = { _yg_type = "Number" } }, nil, {})
  t.assert("nil value → empty",   function() return ni.get() end, "")
end)

yg.test.run("yg.type.ui.input", function(t)
  local i = yg.type.ui.input({ _yg_type = "String" }, "x", {})
  t.assert("returns {node,get} table",       function() return type(i) end, "table")
  t.assert("headless get() returns seed",     function() return i.get() end, "x")
  t.assert("headless Number get() is seed",   function() return yg.type.ui.input({ _yg_type = "Number" }, 7, {}).get() end, 7)
  t.assert("headless Boolean get() is seed",  yg.type.ui.input({ _yg_type = "Boolean" }, true, {}).get() == true)
  -- Ref branch (headless): get() returns the initial_link derived from the seed sentinel
  local ri1 = yg.type.ui.input({ _yg_type = "Ref", ref_class = "reagent" }, { _ref = "ing_quicksilver", _class = "reagent" }, {})
  t.assert("Ref stable seed get()",    function() return ri1.get() end, "[[$ing_quicksilver]]")
  local ri2 = yg.type.ui.input({ _yg_type = "Ref", ref_class = "reagent" }, nil, {})
  t.assert("Ref nil seed get() nil",   ri2.get() == nil)
  local ri3 = yg.type.ui.input({ _yg_type = "Ref", ref_class = "reagent" }, { _ref = "wiki/Foo@5", _class = "reagent" }, {})
  t.assert("Ref positional seed nil",  ri3.get() == nil)
end)
```
